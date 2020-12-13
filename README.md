# snowflake-tutorials
Snowflake Database tutorial

# Snowflake hands-on
## Pre-requisites
- Snowflake account
- user with AccountAdmin or SecurityAdmin role
- snowsql installation
- Sample data files

# Install Snow SQL ( goto help and download ) below instructions

~/.snowsql/config file
```
accountname = <account_name>
username = <account_name>
password = <password> 
```
On MacOS run

```
/Applications/SnowSQL.app/Contents/MacOS/snowsql -a <username> -u <username>
```

## Hands-on
### Step-1:  
login to SnowSQL
```
snowsql -a <account_name> -u <user_name>
```

If this is the first time yyou are doing this, it'll take some time to istall some dependencies before it prompts for the password.

### Step-2:  
Create Snowflake objects
- create database
```
create or replace database sf_tuts;
```
- check the schema
```
select current_database(), current_schema();
```

- create table
```

create or replace table emp_basic (
  first_name string ,
  last_name string ,
  email string ,
  streetaddress string ,
  city string ,
  start_date date
  );
```
- create virtual warehouse
```
create or replace warehouse sf_tuts_wh with
  warehouse_size='X-SMALL'
  auto_suspend = 180
  auto_resume = true
  initially_suspended=true;
```

### Step-3:  
Stage data files  
Linux
```
put file:///Users/darek/workspace/kata/snowflake/getting-started/employees0*.csv @sf_tuts.public.%emp_basic;
```

list files
```
list @sf_tuts.public.%emp_basic;
```

### Step-4:  
Copy data to target tables
```
copy into emp_basic
  from @%emp_basic
  file_format = (type = csv field_optionally_enclosed_by='"')
  pattern = '.*employees0[1-5].csv.gz'
  on_error = 'skip_file';
```

## Querying stage files
```
select t.$1, t.$2 from @sf_tuts.public.%emp_basic  t

create or replace file format myformat
type = 'csv'  error_on_column_count_mismatch=false;

select t.$1, t.$2 from @sf_tuts.public.%emp_basic (file_format => myformat) t;
```

## Querying metadata
```
select metadata$filename, metadata$file_row_number, t.$1, t.$2 from @sf_tuts.public.%emp_basic (file_format => myformat) t;
```

### Step-5:  
Query loaded data
```
select * from emp_basic;

insert into emp_basic values
  ('Clementine','Adamou','cadamou@sf_tuts.com','10510 Sachs Road','Klenak','2017-9-22') ,
  ('Marlowe','De Anesy','madamouc@sf_tuts.co.uk','36768 Northfield Plaza','Fangshan','2017-1-26');

  select email from emp_basic where email like '%.uk';

  select first_name, last_name, dateadd('day',90,start_date) from emp_basic where start_date <= '2017-01-01';
```

### Step-6:  
Clean-up

```
drop database if exists sf_tuts;

drop warehouse if exists sf_tuts_wh;
```

## Onloading data

https://docs.snowflake.net/manuals/user-guide/data-unload-s3.html

Credits: https://docs.snowflake.net/manuals/user-guide-getting-started.html

## Joins

https://docs.snowflake.com/en/user-guide/querying.html
```
use SNOWFLAKE_SAMPLE_DATA.TPCH_SF1;

#inner joins
select * from CUSTOMER C JOIN ORDERS O ON O.O_CUSTKEY = C.C_CUSTKEY LIMIT 10; 

#subquery
SELECT * FROM CUSTOMER C WHERE C_CUSTKEY = (SELECT MAX(O.O_CUSTKEY) FroM ORDERS O);

#cte - common table expression
with
   topCust (okey, ckey, oprice, odate) as (
     select O_ORDERKEY, O_CUSTKEY, O_TOTALPRICE, O_ORDERDATE from  ORDERS LIMIT 10
   )

   select * from topCust;
  
  
#hierarchical (connectby)

use SF_TUTS;

create or replace table managers  (title varchar, employee_id integer);

create or replace table employees (title varchar, employee_id integer, manager_id integer);

insert into managers (title, employee_id) values
    ('President', 1);
    
insert into employees (title, employee_id, manager_id) values
    ('President', 1, null),  -- The President has no manager.
        ('Vice President Engineering', 10, 1),
            ('Programmer', 100, 10),
            ('QA Engineer', 101, 10),
        ('Vice President HR', 20, 1),
            ('Health Insurance Analyst', 200, 20);
            
            
select level || '->' || employee_id, manager_id, title
   from employees                       
   start with title = 'President'
   connect by manager_id = prior employee_id;
   
   
```

## Semi structured

```
create or replace table car_sales
(
  src variant
)
as
select parse_json(column1) as src
from values
('{
    "date" : "2017-04-28",
    "dealership" : "Valley View Auto Sales",
    "salesperson" : {
      "id": "55",
      "name": "Frank Beasley"
    },
    "customer" : [
      {"name": "Joyce Ridgely", "phone": "16504378889", "address": "San Francisco, CA"}
    ],
    "vehicle" : [
      {"make": "Honda", "model": "Civic", "year": "2017", "price": "20275", "extras":["ext warranty", "paint protection"]}
    ]
}'),
('{
    "date" : "2017-04-28",
    "dealership" : "Tindel Toyota",
    "salesperson" : {
      "id": "274",
      "name": "Greg Northrup"
    },
    "customer" : [
      {"name": "Bradley Greenbloom", "phone": "12127593751", "address": "New York, NY"}
    ],
    "vehicle" : [
      {"make": "Toyota", "model": "Camry", "year": "2017", "price": "23500", "extras":["ext warranty", "rust proofing", "fabric protection"]}
    ]
}') v;

select * from car_sales;

select src:dealership from car_sales;


select src:vehicle[0] from car_sales;


#flatten
select
   value:name as "Customer Name",
   value:address as "Address"
   from
     car_sales
   , lateral flatten(input => src:customer);

select get_path(src, 'vehicle[0]:make') from car_sales;
```
