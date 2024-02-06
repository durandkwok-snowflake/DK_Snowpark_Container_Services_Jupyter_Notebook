# DK_Snowpark_Container_Services_Jupyter_Notebook
## Common Setup
### Run the following commands in Snowsight:
```SQL
USE ROLE ACCOUNTADMIN;

CREATE ROLE test_role;

CREATE DATABASE IF NOT EXISTS tutorial_db;

GRANT OWNERSHIP ON DATABASE tutorial_db TO ROLE test_role;

CREATE OR REPLACE WAREHOUSE tutorial_warehouse WITH
  WAREHOUSE_SIZE='X-SMALL';
  
GRANT USAGE ON WAREHOUSE tutorial_warehouse TO ROLE test_role;

CREATE SECURITY INTEGRATION IF NOT EXISTS snowservices_ingress_oauth
  TYPE=oauth
  OAUTH_CLIENT=snowservices_ingress
  ENABLED=true;

GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE test_role;

CREATE COMPUTE POOL tutorial_compute_pool
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS;
  
GRANT USAGE, MONITOR ON COMPUTE POOL tutorial_compute_pool TO ROLE test_role;

GRANT ROLE test_role TO USER mike;

-- Using the test_role role, execute the following script to create database-scoped objects common to all the tutorials.

USE ROLE test_role;
USE DATABASE tutorial_db;
USE WAREHOUSE tutorial_warehouse;

CREATE SCHEMA IF NOT EXISTS data_schema;

CREATE IMAGE REPOSITORY IF NOT EXISTS tutorial_repository;

CREATE STAGE IF NOT EXISTS tutorial_stage
  DIRECTORY = ( ENABLE = true );


-- Verify that you are ready to continue
SHOW COMPUTE POOLS; 

