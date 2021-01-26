# Resize-Redshift-Columns



## Step 1: Create the stored procedure in Redshift



## Step 2: Execute the stored procedure, and then get the alter scripts to resize the columns by querying the temp table 

The procedure has the 4 parameters:
1. var_schema_name varchar(50), the schema's name of the table where you want to resize the columns 
2. var_table_name varchar(50), the table's name where you want to resize the columns 
3. var_postfix varchar(50), the postfix to be appended in the table's name as the temporary table for resizing process  
4. var_ratio decimal(19,2), the ratio used to get the to-be column length based on the as-is column length 

```sql
call proc_replicate_table_with_resize_columns('ad_dw', 'd301_dwm_customer', '_resize_columns', '1.15');
```

## Step 3: 


