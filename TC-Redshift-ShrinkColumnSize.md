# Amazon Redshift系统概述
Amazon Redshift数据仓库是种快速且完全托管的数据仓库服务，让您可以使用标准SQL和现有的商业智能工具经济高效地分析您的所有数据，提供优质的数据仓库解决方案。

# Amazon Redshift表设计的最佳实践

在规划Amazon Redshift数据库时，某些关键表的设计对整体查询性能影响很大。这些设计的优化可以减少I/O操作数和尽量减少处理查询所需的内存，因而对存储需求以至查询性能也有很大的影响。

对Redshift表设计的最佳实践包括了以下几点：

- 选择最佳的排序键
- 选择最佳的分配方式
- 让 COPY 选择压缩编码
- 定义主键和外键约束
- **使用尽可能小的列大小**
- 在日期列中使用日期/时间数据类型

本文中，我们会聚焦于“使用尽可能小的列大小”这一点，介绍如何通过SQL脚本的方式半自动化地完成对Redshift数据表中列大小的优化。


# 使用尽可能小的列大小

### 优化的必要性
根据官方文档上所描述的，虽然Redshift在数据压缩方面非常出色，定义过大的列长度对数据表的大小不会造成很大影响。但是，在运行一些复杂的查询时，由于中间过程数据会被临时存储，而这时创建的临时表不会被指定压缩格式，这样就会造成查询占用过多的内存或者临时磁盘空间的现象，从而导致查询性能的降低。

[https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/c_best-practices-smallest-column-size.html](url)

### 测试环境准备

首先，运行如下SQL脚本在你的Redshift数据库中新建一张数据表test_schema.customer用于测试。

```sql
CREATE SCHEMA test_schema;

CREATE TABLE test_schema.customer
(
  c_custkey      INTEGER NOT NULL encode delta,
  c_name         VARCHAR(65535) NOT NULL encode zstd,
  c_address      VARCHAR(65535) NOT NULL encode zstd,
  c_city         VARCHAR(65535) NOT NULL encode zstd,
  c_nation       VARCHAR(65535) NOT NULL encode zstd,
  c_region       VARCHAR(65535) NOT NULL encode zstd,
  c_phone        VARCHAR(65535) NOT NULL encode zstd,
  c_mktsegment   VARCHAR(65535) NOT NULL encode zstd
) diststyle even;
```

其次，执行如下SQL脚本将测试数据导入test_schema.customer表。测试数据位于[LoadingDataSampleFiles.zip](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/samples/LoadingDataSampleFiles.zip)压缩包中。
```sql
copy test_schema.customer
from 's3://<your-bucket-name>/load/customer-fw.tbl'
credentials 'aws_access_key_id=<Your-Access-Key-ID>;aws_secret_access_key=<Your-Secret-Access-Key>' 
fixedwidth 'c_custkey:10, c_name:25, c_address:25, c_city:10, c_nation:15, c_region :12, c_phone:15,c_mktsegment:10';
```

现在，数据表test_schema.customer中的varchar字段的长度都被定义为了65535。运行以下SQL脚本，可以看到实际数据中的字段长度远远小于65535.

```sql
  select
      max(len(c_name)) as c_name,
      max(len(c_address)) as c_address,
      max(len(c_city)) as c_city,
      max(len(c_nation)) as c_nation,
      max(len(c_region)) as c_region,
      max(len(c_phone)) as c_phone,
      max(len(c_mktsegment)) as c_mktsegment
  from test_schema.customer
```
![Xnip2021-05-05_17-01-12](https://user-images.githubusercontent.com/19648269/117145037-4021b900-ade5-11eb-82ad-cb3e8499f9ce.jpg)


### 优化现有数据表中列大小的操作过程

接下来，我们会通过以下八个步骤，来完成对数据表test_schema.customer的优化工作，将数据表中所有varchar数据类型的列大小进行优化，根据已有数据的最大长度来进行相应的压缩。下面示例脚本所操作的数据表为test_schema.test_table，请根据实际情况进行替换。

第一步，在Redshift数据库中创建存储过程 proc_replicate_table_with_resized_columns。
这个存储过程提供了4个参数，分别是：

- var_schema_name varchar(50)，该参数用于指定您需要进行列大小的优化的数据表的schema。
- var_table_name varchar(50)，该参数用于指定您需要进行列大小的优化的数据表的表名。
- var_postfix varchar(50)，该参数用于在原表名后附加后缀名，作为新创建数据表的表名。
- var_ratio decimal(19,2)，该参数用于指定一个系数，将列大小调整为该列最大长度乘以该系数。

首先，该存储过程会创建一个和指定数据表一样表结构的新数据表，该新表的名称会在原先表的名称后附加一个您指定的后缀；其次，该存储过程会检查指定数据表中所有varchar数据类型的列，如果改列在现有数据中的最大长度乘以一个系数之后，仍然小于表定义中原先设定的长度，则会生成一个SQL脚本来调整新创建表中该列的长度，长度被调整为round(column_actual_len * var_ratio)。

```sql
create or replace procedure proc_replicate_table_with_resized_columns
	(
        var_schema_name in varchar(50),
        var_table_name in varchar(50),
        var_postfix in varchar(50),
        var_ratio in decimal(19,2)
    )
    language plpgsql
as $$
DECLARE
    sql_text varchar(65535);
    table_full_name varchar (100) ;
    table_full_name_new varchar (100) ;
    rec RECORD;
BEGIN
    select into table_full_name var_schema_name || '.' || var_table_name;
    select into table_full_name_new  var_schema_name || '.' || var_table_name || var_postfix;

    --create a new table with the same schema
    EXECUTE 'create table '  || table_full_name_new || ' (like ' || table_full_name || ')';

    -- Get the temp table for all columns for a table, temp_table_column_scripts
    drop table if exists temp_table_column_scripts;
    create temp table temp_table_column_scripts(script varchar);

    insert into temp_table_column_scripts
    select  'select ''' || column_name || ''' as column_name, ' ||
       'max(len(' || column_name ||')) as column_actual_len, ' ||
       cast(character_maximum_length as varchar) || ' as column_def_len' ||
       ' from ' || table_schema || '.' || table_name as column_script
    from svv_columns
    where table_schema = var_schema_name and table_name = var_table_name and data_type = 'character varying';

    --loop to insert the column info into temp_table_column_info
    drop table if exists temp_table_column_info;
    create temp table temp_table_column_info(column_name varchar, column_actual_len int, column_def_len int);

    FOR rec IN SELECT script FROM temp_table_column_scripts
      LOOP
        RAISE INFO 'insert into temp_table_column_info %', rec.script;
        EXECUTE 'insert into temp_table_column_info ' || rec.script;
      END LOOP;

    --Generate temp table for alter scripts
    drop table if exists temp_table_alter_scripts;
    create temp table temp_table_alter_scripts as
    SELECT 'alter table ' || table_full_name_new || ' alter column ' ||
           column_name || ' type varchar(' || cast(round(column_actual_len * var_ratio) as varchar) || ');'
    FROM temp_table_column_info
    where round(column_actual_len * var_ratio) < column_def_len;

END;
$$;
```

第二步，执行上一步中创建出来的存储过程，并从结果表中获得用于优化列大小的SQL脚本。

具体操作可以参考以下SQL脚本，请将脚本中{s3-bucket-name}、{ACCESS_KEY_ID}、{SECRET_ACCESS_KEY}根据实际情况进行替换。

```sql
call proc_replicate_table_with_resize_columns('test_schema', 'test_table', '_resize_columns', '1.15');

select * from temp_table_alter_scripts;
```

第三步，运行上一步中从temp_table_alter_scripts表中返回的SQL脚本。

第四步，运行以下SQL脚本用于检查数据表中的列大小是否已经调整。
```sql
select  a.table_name, a.column_name, a.data_type,
       a.character_maximum_length as as_is_length, b.character_maximum_length as to_be_length
from svv_columns as a
inner join svv_columns as b
    on a.table_name + '_resize_columns' = b.table_name
    and a.column_name = b.column_name
where a.table_schema = 'test_schema'
  and a.table_name = 'test_table'
  and b.table_name = 'test_table_resize_columns';
```

第五步，将原数据表中的数据UNLOAD到指定的S3路径中。
具体操作可以参考以下SQL脚本，请将脚本中{s3-bucket-name}、{ACCESS_KEY_ID}、{SECRET_ACCESS_KEY}根据实际情况进行替换。
```sql
unload ('select * from test_schema.test_table')
to 's3://{s3-bucket-name}/resize-redshift-columns/test_table/'
ACCESS_KEY_ID '{ACCESS_KEY_ID}'
SECRET_ACCESS_KEY '{SECRET_ACCESS_KEY}'
GZIP;
```

第六步，将S3中的数据COPY到新建数据表中。

具体操作可以参考以下SQL脚本，请将脚本中{s3-bucket-name}、{ACCESS_KEY_ID}、{SECRET_ACCESS_KEY}根据实际情况进行替换。
```sql
copy test_schema.test_table_resize_columns
from 's3://{s3-bucket-name}/resize-redshift-columns/test_table/'
ACCESS_KEY_ID '{ACCESS_KEY_ID}'
SECRET_ACCESS_KEY '{SECRET_ACCESS_KEY}'
GZIP;
```

第七步，检查新建数据表中的数据是否与原数据表中的数据完全一致。

运行以下SQL脚本，如果返回结果为空，则代表新建数据表中的数据与原数据表完全一致。如果返回有数据，则需要检查之前的步骤是否操作有误，导致数据不一致。

```sql
select * from test_schema.test_table
except
select * from test_schema.test_table_resize_columns
union all
select * from test_schema.test_table_resize_columns
except
select * from test_schema.test_table
```

第八步，重命名数据表，使得新建数据表替换原数据表，并在替换之后删除原数据表。

在该步骤中，首先将原数据表重命名，例如：添加_original后缀；然后，将新建数据表的名称重命名为原数据表的名称；最后，删除原数据表（注意：删除的数据表是带_original后缀的数据表）。

具体操作可以参考以下SQL脚本：
```sql
alter table test_schema.test_table
rename to test_table_original;

alter table test_schema.test_table_resize_columns
rename to test_table;

drop table test_schema.test_table_original;
```
通过以上八个步骤，您就完成了对数据表test_schema.test_table内所有varchar数据列大小的优化工作。

