# Resize-Redshift-Columns



## Step 1: Create the stored procedure in Redshift



## Step 2: Execute the stored procedure, and then get the alter scripts to resize the columns by querying the temp table 

```sql
call proc_replicate_table_with_resize_columns('ad_dw', 'd301_dwm_customer', '_resize_columns', '1.15');
```

## Step 3: 


