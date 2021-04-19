# Data Proxy project

Under the requests of AI team, we are committed to help them to prepare data in their desired data format. 

## Introduction

### What?

The data proxy for control kit is designed to provide pivot tables-type of points values for control kit in a manner of time series. It will be served in AWS MongoDB as a format of documents/collections for each project/extraction name/extraction interval. 

This is the first application built over a NoSQL database at BrainBoxAI.

![All current active collections in MongoDB, is in the pattern of projectName;tableName;monitoryName;interval](https://lh4.googleusercontent.com/qW4b_Sg6Hf-Vv7l7njIQIF2TrHWd1L4CcGMiFlCje6KpTjqu5XYeKUxNw4IOvyvFwKyAzBmeOssaa7hfqlSZdOICmgFZj_q6kr7qGSUzafFwuHhYCKoR8lH-1citLMmH34V7EWI6)

![](https://lh3.googleusercontent.com/Hvb9FKsachGzrjlLhG_D-M_iv0rd0IWadlsg4wZNHw9abTXYQO68tPHDzLp0xkao1lD4YYLs0eaeJoHTPL2-GT0opGoaL4MS_QKm9Sqkfo0I4AH3WIW9ccJuA_nd3bTCpajiJZhB)

![A document in MongoDB in pivot table row format](https://lh6.googleusercontent.com/HctWe9tppNBd2zVycX4azpzzqYBbv4sjfdl3FUedlXF5neaX5aRPB26F0aUpop0MN9D9W1B19CPB2-pv9O8PGOa1ChyvbLL49daryMH_9VFLo454uouYtUSpOMfBn-LWygMH6Eq3)

### Why?

Accessing raw aggregated information from MySQL cloud databases and creating pivot tables is a very heavy and repetitive process.Therefore, in order to decrease and moderate this load of process, this new flexible cashing mechanism is built upon NoSQL database \(AWS MongoDB\) service. In addition to reducing Load from infrastructure, it removes latency in pivot table creation and provides pivot tables on demand.

### Who?

AI team \(control kit\) or all other applications that require pivot tables.

### What it can do?

1. Transform structured data from MySQL databases to documents-like AWS Mongo collections per wave counter values.
2. Provide the latest point values from extraction in the format of time-series. 
3. Maintain the fresh data and remove the outdated point values from NoSQL collections.

## Usage/Examples

The module is embedded with one of the most important features - create service for new project/building automatically. In order to let the program perform its duties, it is strictly required that the configurations are set properly. 

The 

#### Configuration Example

`"1":{   
    "service_name":"TOR-BGO-150KingW",   
    "wave_counter_threshold": 1617753611, "delete_old_value": 7, "monitory_name_kept": ["Extraction_1_dev_a","Extraction_2_dev_a","Extraction_3_dev_a"], "monitory_dbName": "script_n", "interval_dbName": "interval", "TABLES": { "ext_cycle": "extraction_cycle", "measures": "TL_measures"}, "database": "TOR-BGO-150KingW", "path":"/home/brainbox/DATA_PROXY/TOR-BGO-150KingW/src", "server":"awsdb.brainboxai.net", "sleep_time":5, "mongo_to_local":false }`

## Functionalities

## Environments

## Issues

## Collaborators

