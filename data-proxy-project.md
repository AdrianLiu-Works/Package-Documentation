# Data Proxy project

Under the requests of AI team, we are committed to design and implement the data proxy for control kit, which helps to create pivot table of wave counter and point ID with the value of point values. 

_**Note: For a quick start of using this module, please refer to**_ [_**Quick start**_](data-proxy-project.md#quick-start) _**section**_

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

## General Workflow

### Workflow

![Flow of this module](.gitbook/assets/image%20%283%29.png)

![Flow from bigger picture](.gitbook/assets/image%20%286%29.png)

### Details

1. Users [provided configuration](data-proxy-project.md#usage-examples) must be written into ._/prototype/submit\_request/request.json_
2. After completing _request.json_, it's mandatory to run _submit\_requst.py_
3. The destination of copying source code files are the ["path"](data-proxy-project.md#public-configuration-example) in the configuration, this will create a new folder and will merge the user-provided configuration file \(public\) with the default configuration in the program \(private\).
4. The package is implementing .env technique to securely retrieve the database passwords. Since it wouldn't be changed for a long period of time, it will be set as default \(in private configuration\)

## Usage/Examples

### Quick start

#### How to initiate DB proxy for one project: <a id="how-to-initiate-db-proxy-for-one-project"></a>

The program has an active running service named _DB\_PROXY\_Service\_Creator.service_ that checks new request for projects. A dictionary based request is needed to be created similar to example availble in this repo.

[Example of initiation request](https://git.brainboxai.net/DataStreams/DATA_PROXY_FOR_AI_MODELS/src/branch/master/submit_request/request.json)

After the submissions please run the **submit\_request.py** file with **brainbox permission** at the same directory. [Source code for submit\_request.py](https://git.brainboxai.net/DataStreams/DATA_PROXY_FOR_AI_MODELS/src/branch/master/submit_request/submit_request.py)

Required fields of _request.json -_ the configuration file of user:

* “wave\_counter\_threshold”: 
  * {unix value from the time data is supposed to be gathered from mysql},
* “delete\_old\_value”:               
  * {number of the days data will be kept in collections},
* “monitory\_name\_kept”:       
  * {list of extraction packages that correspond to one projects result in creation of multiple collections per one projects}

 ![](.gitbook/assets/image%20%284%29.png) 

* “measures”: 
  * {measures table name could be TL\_measures and hs\_measures}
* “database”: 
  * {name of the database},
* “server”:
  * {database server address},
* “sleep\_time”:
  * {secs data will be sync between mysql and Mongo DB}

After running the program, _request.json_ will be reinitiated to default values: [default request.json](https://git.brainboxai.net/DataStreams/DATA_PROXY_FOR_AI_MODELS/src/branch/master/submit_request/request_default.json)

**Don’t forget to remove \_ from the keys to use it next time ;\)**

_DB\_PROXY\_service\_creator_ program will take the new request and creates a new project at **/home/brainbox/DB\_PROXY/{database}** and run a background service with a name _DB\_PROXY\_{database}.service._

* you could check the service to insure project runs smoothly :\) \(systemctl status _DB\_PROXY\_{database}.service_\)

Now the db proxy for project starts working and gets information from MySQL and pushes the value into collections as pivot tables in MongoDB.

#### How to access Mongo DB data: <a id="how-to-access-mongo-db-data"></a>

The Handler folder supports neccessary functionality to fetch information from MongoDB.

[Mongo DB Handler](https://git.brainboxai.net/DataStreams/DATA_PROXY_FOR_AI_MODELS/src/branch/master/handler)

Mongo db handler requires environ file that includes credentials for connection to Mongo DB. This piece of information will be shared through LastPass. The .env file format is as follow: SERVER=“{mongo db server address}” USERNAME=“{mongo db username}” PASSWORD=“{mongo db password}” PORT= {mongo db port number}

For unit test: there are some examples some commented some not ! in the [mongo\_handler\_cloud.py](https://git.brainboxai.net/DataStreams/DATA_PROXY_FOR_AI_MODELS/src/branch/master/handler/mongo_handler_cloud.py) so that by running the code it should provide similar following results: ![](.gitbook/assets/image%20%285%29.png) 



### A bit details

The module is embedded with one of the most important features - create service for new project/building automatically. In order to let the program perform its duties, it is strictly required that the configurations are set properly. 

There are two types of configurations, one private \(proxy\_config.json\) and another for public \(configurable to the program\). The private one majorly covered some default settings, such as the service creator name and debug mode \(only for developer of this module\), in other words, those are not supposed to be a concern for the end-user. Thus, in this section, we will focus on the public configuration \(request.json\). 

#### Public Configuration Example

It is more intuitional to explain with an example. In the one below, which is a typical example of how the public configuration structure look like. 

```text
{"wave_counter_threshold": 1617753611,
"delete_old_value": 7,
"monitory_name_kept": ["Extraction_1_dev_a","Extraction_2_dev_a","Extraction_3_dev_a"],
"measures": "TL_measures",
"database": "TOR-BGO-150KingW",
"server":"awsdb.brainboxai.net",
"sleep_time":5}
```

#### General steps:

1. Open _request.json_
2. Modify any necessary attributes that mentioned in the above example
3. Organize them to a well-formatted json file and write back to request.json
4. Run _submit\_request.py_
5. Check the message printed, 
   1. if okay, check systemctl status _DB\_PROXY\_{database}.service_
   2. if not okay, it's most likely due to the configuration format error, recheck your submitted request in the json file
6. The service creator system service will take care of it and establish the new system service. 

## Functionalities

This section will cover all the functionalities that this module offers, the order of appearance is also the order of execution. 

### Automatically create system service

### Fetch data from Extraction Cycle table

### Match the wave counter from Extraction Cycle to Measures table 

### Prepare the point values 

### Push matched point values to AWS/Local Mongo DB

## Environments

There are several packages needed to perform duties:

1. [pymongo](https://pypi.org/project/pymongo/)
2. [sqlalchemy](https://pypi.org/project/SQLAlchemy/)
3. [environs](https://pypi.org/project/environs/)
4. [pymysql](https://pypi.org/project/PyMySQL/)

All available in the python pip install library

## Passed test cases

Due to the consideration of quality assurance, we made several tests to ensure the performance as intended. The following are the list of test cases that this modules passed:

1. Automatically generate a new system service for new project/building according to the provided configuration file
2. Fetch point values from Measures table based on the the wave counter we captured from Extraction Cycle table
3. Write fetched point values to AWS Mongo DB
4. Write fetched point values to Local Mongo DB
5. Transfer user provided configuration files to its desired destination

## Issues

The execution of program may encounter with the potential issues:

1. After creation of the system service, the service is still **Inactive/disabled** 
   1. **Solution:** sudo systemctl restart your\_service_\__name.service

Any other issues occurred, please use this [form ](https://app.asana.com/0/1188271036483121/overview)to submit a ticket

## Collaborators

[Farzam](https://git.brainboxai.net/Farzam)

[Adrian](https://git.brainboxai.net/adrian)

