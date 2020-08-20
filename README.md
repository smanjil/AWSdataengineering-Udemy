# Data-Engineering AWS

## Preparation
```
$ sudo apt install pipenv
$ pipenv shell --python 3.8  (Creates a virtual environment with Python3.8)
```

## Install dependencies
```
$ pipenv install  (Installs the dependencies from Pipfile)
$ pip freeze
```

## Login to ur aws console (I chose free-tier, as I am a student ;))

## First: Setting up RDS
```
- Select RDS service.
- Create database (Mysql 5.7.26)
- Free tier
- dbname: <dbname>
- username: <username>
- password: <password>
- db instance, storage (default)
- Most of the settings default which are not mentioned here..
- Connectivity --> Additional connectivity --> Publicly accesible (Yes)
- Create database (and u will be redirected to a database list page)
```
After successful creation of database, click on the database identifier, and
 select the endpoint from "Connectivity & Security" which will be used
  further to connect to RDS instance remotely.
  
### Install dbeaver in local system; to connect to remote RDS
```
$ https://computingforgeeks.com/install-and-configure-dbeaver-on-ubuntu-debian/
```

### Install mysql in local system
```
$ https://support.rackspace.com/how-to/install-mysql-server-on-the-ubuntu-operating-system/
```

### Create security group inside VPC in aws services
```
- Select the group with group name default.
- Goto inbound rules.
- Edit inbound rules.
- Then, Add rule, and select 'mysql' in type and 'my ip' in source type.
```

### Connect RDS with dbeaver with the endpoint, username and password you have while creating RDS.

### Restore the mysql dump to the RDS using cmd line:
```
- Crate a new database named ecommerce_db inside ur RDS instance in dbeaver.
$ mysql -h dedb.cn5u6kuqn09d.eu-west-1.rds.amazonaws.com -u smanjil -p ecommerce_db < datasets/mysql_files/full_db_dump/ecommerce_db.sql

- Assuming ur dump is already there in ur project folder 
```

### Create Redshift cluster
```
- Select Redshift from services
- Create cluster
- Give a cluster identifier (any name)
- Chosse dc2.large for free-tier
- Num nodes: 2
- Leave database configuration as is.
- Select a master username
- Select a master password
- Cluster permissions
    - Create a IAM role (IAM services)
    - Click on roles
    - Create a role (Choose Redshift then)
    - Click on Redshift customizable, then click Next
    - Permission
        - Select AmazonRedshiftFullAccess (click Next)
        - Skip tags
        - Give a role name, and click Next.
    - Role has now been created.
    - Refresh on the cluster permissions and select the IAM role u created
 earlier.
- Additional configurations
    - Unselect Use defaults
    - Network and security
        - Publicly accessible to Yes
- Create cluster.
```

### Connect the cluster to dbeaver using the Endpoint in Connection details after u click on the cluster, and use ur username and password for the database of the cluster.

### Create AWS Data Pipeline
```
- Select AWS Data Pipeline from services
- Click on Create new pipeline
    - Name
    - Source -> Build Using a template
        - RDS templates
            - Full copy of RDS MySQL table to S3
    - RDS MySQL password
    - output s3 folder (the one you create beforehand)
    - RDS MySQL username
    - RDS MySQL table name (the one table name to copy to s3)
    - EC2 instance type: t2.micro (free tier)
    - RDS instance ID (from RDS instance page: db identifier)
    - Schedule -> Run -> on pipeline activation
    - Logging disabled
- Click on Edit in Architect
    - U will see a flowchart on left, and properties on right
    - U want to update query in DataNodes by:
        - select
            a.order_id,
            a.order_status,
            a.customer_id,
            a.order_approved_at,
            a.order_delivered_carrier_date,
            a.order_delivered_customer_date,
            a.order_estimated_delivery_date,
            a.order_purchase_timestamp,
            b.customer_city,
            b.customer_state,
            b.customer_zip_code_prefix
        from orders a
        left join customers b
        on a.customer_id  = b.customer_id
        where date(a.order_purchase_timestamp)<='2018-09-31'
    - Delete the directory path in S3output location.
        - Choose file path from Add an optional field and:
            - enter #{myOutputS3Loc}/order.csv
    - Save and Activate
```
This thing will roughly take about 5 to 10 minutes to finish.

### Create your own schema inside you redshift cluster (not the instance), and load the data from s3 to table.
```
- ignore the public schema.
- Open a sql editor in your cluster.
    - Create schema mysql_dwh;   (Run this, it will create a new schema
 named mysql_dwh)
    - Run CREATE TABLE mysql_dwh.orders (
            order_id VARCHAR(50) , 
            order_status VARCHAR(35) , 
            customer_id VARCHAR(100) , 
            order_approved_at TIMESTAMP, 
            order_delivered_carrier_date TIMESTAMP, 
            order_delivered_customer_date TIMESTAMP, 
            order_estimated_delivery_date TIMESTAMP, 
            order_purchase_timestamp TIMESTAMP, 
            customer_city VARCHAR(50) , 
            customer_state VARCHAR(10) , 
            customer_zip_code_prefix DECIMAL 
        );  to create a orders table inside that schema.
    - When executing queries, do not forget to specify the name of schema.
- Copy 'mysql_dwh.orders' query from 'copy_cmd.sql'.
- Before that give 'AmazonS3FullAccess' to the role used for redshift.
- Run the copy_cmd statement, this will populate the mysql_dwh.orders table
 from the data in s3.
- Verify if the data sync between RDS and redshift cluster happened
 correctly:
    - In cluster, run:
        - select count(*) from mysql_dwh.orders;  (should give, 99437)
    - In RDS, run:
        - select count(*) from ecommerce_db.orders where date
(order_purchase_timestamp) <= '2018-09-31';  (should also give, 99437)
- If this same, its good, else recheck the pipeline query.  (There could be
 some 'timestamp' mismatch in the pipeline query.
```

### Starting with AWS GLUE
```
- Create a role for glue access first
    - Got to IAM -> roles -> Create role
    - AWS Service -> Glue -> Click Next
    - Choose AmazonRedshiftFullAccess and AWSGlueConsoleFullAccess and
 AWSGlueServiceRole and AmazonS3FullAccess -> Click Next -> Next
   - Give a name for your role and click Next
```

#### Create a new Glue connection (Glue - Databases - Connections (on left side))
```
- Update your default security group to allow Glue connection.
    - VPC -> Security groups
    - Choose ur security group and add the following to inbound and outbound
 rule.
        - Type (All TCP) - Source type(Custom) - Source (ur security group
 name (starting with sg-*))

- Create a VPC endpoint for s3 connections in GLUE
    - VPC -> Endpoints -> Create Endpoint
    - Service name should be selected for s3.
    - Select route table with 3 subnets.
    - Click create with all defaults.

- Click on Add connection
    - Fill a connection name
    - Connection type (amazon redshift) (Click next)
    - Choose Cluster from the dropdown.
    - Fill dbname, username, password for the redshift db, and click Next.

- Test connection with the role u created earlier. It should be connected now.
```

### AWSDataPipeline setup first hourly jobs for incremental data loads
```
- Create new data pipeline with same setting as before.
- New output s3 folder: s3://mysql-de-dwh/orders/current
- Schedule (on a schedule)
- run every 1 hour
- starting (on pipeline activation)
- Use query from import_orders_hourly.sql in datasets dir.
```

### Python shell job for incremental data loads into redshift
```
- First create secrets in Services/Secrets Manager in AWS
- Click on Store a new secret
    - credentials for redshift cluster as a secret type
    - username and password, choose database and Next.
    - Pass a secret name
- Get python files from resource section.
- And, use glue_import_orders.py and use ur secret name and db name.
```

### Create a staging table in redshift cluster (dev) db in dbeaver:
```
-- create schema mysql_dwh_staging;
--create table mysql_dwh_staging.orders as select * from mysql_dwh.orders limit 1;
--truncate mysql_dwh_staging.orders;
```

### Create new GLUE job
```
- Aws services GLUE
- Add job
    - name: glue_import_orders_hourly
    - IAM role that you created for glue redshift.
    - type: python shell
    - A new script to be authored by you. (or else the script should be present in s3) ; click Next

    - Select Connection (Add connection)
    - a name, connection type amazon redshift
    - choose cluster, db name, user and password and next
    - Click select, it should show up in required connection.
    - Save job and edit script
        - Paste the python script from glue_import_orders.py.
    - Click on Save.
    - Go to IAM role and attach a new policy 'SecretsManagerReadWrite' to the IAM role for glue redshift connection created above.
```

-- Go back and run the job.