
1. 场景描述
Amazon Redshift是一种完全托管的PB级云中数据仓库服务。借助 Redshift，您可以使用标准 SQL 在数据仓库、运营数据库和数据湖中查询和合并结构化和半结构化数据。


Amazon Redshift设计表的 最佳实践

在规划数据库时，某些关键表设计决策对整体查询性能影响很大。这些设计选择可以减少 I/O 操作数和尽量减少处理查询所需的内存，因而对存储需求以至查询性能也有很大影响。

在本节中，您可以找到最重要的设计决策的总结和优化查询性能的最佳实践。使用自动表优化提供更为详细的表设计选项说明和示例。

主题

选择最佳的排序键
选择最佳的分配方式
让 COPY 选择压缩编码
定义主键和外键约束
使用尽可能小的列大小
在日期列中使用日期/时间数据类型




首先，在Redshift中创建一个存储过程proc_replicate_table_with_resize_columns。
该存储过程包含一下4个参数：
1. var_schema_name
2. var_table_name
3. var_postfix
4. var_ratio


create or replace procedure proc_replicate_table_with_resize_columns
(
  var_schema_name in varchar(50)      
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
