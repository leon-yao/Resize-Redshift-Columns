# Resize-Redshift-Columns



## Step 1: Create the stored procedure in Redshift

```sql
create or replace procedure proc_replicate_table_with_resize_columns
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

    /*
    --loop to resize the column length to actual size in new table

    FOR rec IN SELECT column_name,column_actual_len,column_def_len FROM temp_table_column_info
      LOOP
        --RAISE INFO 'column name: %; actual len: %; def len: %', rec.column_name,rec.column_actual_len,rec.column_def_len;
        IF rec.column_actual_len <> rec.column_def_len
            THEN
                RAISE INFO 'column name: %; actual len: %; def len: %', rec.column_name,rec.column_actual_len,rec.column_def_len;
                EXECUTE 'alter table ' || table_full_name_new || ' alter column ' || rec.column_name || ' type varchar(' || cast(rec.column_actual_len as varchar) || ');';
            END IF;
      END LOOP;
    */

END;
$$;

```


## Step 2: Execute the stored procedure and then get the alter scripts 
The procedure has the 4 parameters:
1. var_schema_name varchar(50), the schema's name of the table where you want to resize the columns 
2. var_table_name varchar(50), the table's name where you want to resize the columns 
3. var_postfix varchar(50), the postfix to be appended in the table's name as the temporary table for resizing process  
4. var_ratio decimal(19,2), the ratio used to get the to-be column length based on the as-is column length 

```sql
call proc_replicate_table_with_resize_columns('ad_dw', 'd301_dwm_customer', '_resize_columns', '1.15');

select from temp_table_alter_scripts;
```

## Step 3: Copy the alter scripts from Step 2, and run the scripts

## Step 4: Confirm whether the column size has been changed as expected 

```sql
select  a.table_name, a.column_name, a.data_type,
       a.character_maximum_length as as_is_length, b.character_maximum_length as to_be_length
from svv_columns as a
inner join svv_columns as b
    on a.table_name + '_resize_columns' = b.table_name
    and a.column_name = b.column_name
where a.table_schema = 'ad_dw'
  and a.table_name = 'd301_dwm_customer'
  and b.table_name = 'd301_dwm_customer_resize_columns';
```

## Step 5: Unload the data from the original table to S3

```sql

unload ('select * from ad_dw.d301_dwm_customer')
to 's3://s3-bucket-name/resize-redshift-columns/d301_dwm_customer/'
ACCESS_KEY_ID '{ACCESS_KEY_ID}'
SECRET_ACCESS_KEY '{SECRET_ACCESS_KEY}'
GZIP;

```

## Step 6: Copy the data from S3 to the new table 

```sql
copy ad_dw.d301_dwm_customer_resize_columns
from 's3://s3-bucket-name/resize-redshift-columns/d301_dwm_customer/'
ACCESS_KEY_ID '{ACCESS_KEY_ID}'
SECRET_ACCESS_KEY '{SECRET_ACCESS_KEY}'
GZIP;
```

## Step 7: Check if the new table is identical to the original table 

It passes if the below script returns the empty result. 

```sql
select * from ad_dw.d301_dwm_customer
except
select * from ad_dw.d301_dwm_customer_resize_columns

union all

select * from ad_dw.d301_dwm_customer_resize_columns
except
select * from ad_dw.d301_dwm_customer
```

## Step 8: Rename the tables and drop the original table 

```sql
alter table ad_dw.d301_dwm_customer
rename to d301_dwm_customer_original;

alter table ad_dw.d301_dwm_customer_resize_columns
rename to d301_dwm_customer;

drop table ad_dw.d301_dwm_customer_original;
```
