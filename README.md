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
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/ebb2c1f6-b5ef-47d6-beff-6a1647db5538)
```SQL
SHOW WAREHOUSES;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/7a37258e-874c-491c-9433-00ad9a9c3ac8)
```SQL
SHOW IMAGE REPOSITORIES;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/c92efe51-ef87-4e28-b3d0-126c0a72d558)
```SQL
SHOW STAGES;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/16748175-2842-4c33-bef8-85917e760f7b)

**###Create Folder For Jupyter Tutorial:**
```SQL
(base) dkwok@D43HJ72NVN SnowparkContainerServices-Tutorials % ls
Tutorial-1	Tutorial-2	Tutorial-3
(base) dkwok@D43HJ72NVN SnowparkContainerServices-Tutorials % mkdir Tutorial-DK
(base) dkwok@D43HJ72NVN SnowparkContainerServices-Tutorials % cd Tutorial-DK
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/a6eedaee-ca3e-4bcd-86f2-609097236a00)
**###Copy Docker and my_jupyter.yaml file to folder:**
Dockers Build the Image:
```SQL
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % docker build --rm --platform linux/amd64 -t sfsenorthamerica-dkdemo3.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_jupyter_image:latest .
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/be3e34d2-bb52-4450-831d-fe4b9b6cc21c)
**###Login to Dockers**
```SQL
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % docker login sfsenorthamerica-dkdemo3.registry.snowflakecomputing.com -u mike
Password: 
Login Succeeded
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/e6fbc398-b6fa-4e7e-86d3-55daf7127f78)
**###Dockers Push the Image**
```SQL
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % docker push sfsenorthamerica-dkdemo3.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_jupyter_image:latest
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/eb5772ce-c7fe-4d54-8b87-08666e8ff38a)
**###Dockers and My_jupyter.yaml:**
**###Dockers**
```SQL
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % cat Dockerfile 
FROM rapidsai/rapidsai:22.12-cuda11.5-base-ubuntu20.04-py3.8
RUN apt-get update && apt-get install -y --no-install-recommends
RUN conda --version
RUN conda install -n rapids -c https://repo.anaconda.com/pkgs/snowflake snowflake-snowpark-python pandas jupyterlab
# OPTIONAL: Copy Notebook and other files into the container at /app
# COPY MyNotebook.ipynb .
# COPY data.csv .
# Make port 8888 available to the world outside this container
EXPOSE 8888
# Run Jupyter Lab on port 8888 when the container launches
CMD ["/opt/conda/envs/rapids/bin/jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root", "--NotebookApp.token=''", "--NotebookApp.password=''"]
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % 
```
**###My_jupyter.yaml**
```SQL
(base) dkwok@D43HJ72NVN Tutorial-Jupyter % cat my_jupyter.yaml 
spec:
  container:  
  - name: snowpark
    image:  sfsenorthamerica-dkdemo3.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_jupyter_image
    volumeMounts: 
    - name: workspace
      mountPath: /home/snowpark/notebooks/workspace
  endpoint:
  - name: snowpark
    port: 8888
    public: true
  volume:
  - name: workspace
    source: local
    uid: 1000
    gid: 1000
```
**###Upload my_jupyter.yaml to TUTORIAL_STAGE**
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/842cc6c3-9f45-40b1-bb5b-ed76314a4b40)

**###Set Context, upload yaml file to stage and create service for jupyter**
```SQL
USE ROLE test_role;
USE DATABASE tutorial_db;
USE SCHEMA data_schema;
USE WAREHOUSE tutorial_warehouse;
```
**###Create jupyter_service pointing to image from my_jupyter.yaml**
```SQL
CREATE SERVICE jupyter_service
  IN COMPUTE POOL tutorial_compute_pool
  FROM SPECIFICATION $$
spec:
  container:  
  - name: snowpark
    image:  sfsenorthamerica-dkdemo3.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_jupyter_image
    volumeMounts: 
    - name: workspace
      mountPath: /home/snowpark/notebooks/workspace
  endpoint:
  - name: snowpark
    port: 8888
    public: true
  volume:
  - name: workspace
    source: local
    uid: 1000
    gid: 1000
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;
```
```SQL
DESCRIBE SERVICE JUPYTER_SERVICE;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/b9807d9a-32b0-4340-8947-99de971aaaeb)
```SQL
SELECT SYSTEM$GET_SERVICE_STATUS('JUPYTER_SERVICE');
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/b98ec21a-1d34-4665-9f1e-8be0309ca545)
```SQL
SHOW ENDPOINTS IN SERVICE JUPYTER_SERVICE;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/fcb88265-2044-4401-b942-f996a976a174)
```SQL
SHOW COMPUTE POOLS;
```
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/7bfba67c-4132-48db-8f41-9fabc39d1563)
**###Test running Jupyter:**
Run the following link on a browser tab
https://kzhbyi-sfsenorthamerica-dkdemo3.snowflakecomputing.app/lab
![image](https://github.com/durandkwok-snowflake/DK_Snowpark_Container_Services_Jupyter_Notebook/assets/109616231/3e296857-9735-4b2b-acf0-f2eb6fb1496f)
