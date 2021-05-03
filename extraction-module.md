# EXTRACTION module

NOTES: 1. default behavior of write status to db, 2. comparison between previous setting and new setting regarding the db data

## Overview

Extraction module is part of the LEA ecosystem, which is committed to oversee the modules status of parse, release commands, extract points, and data pushing.

Redis is massively used in this package and almost all of modules interact with Redis to gather the config information. However, instead of using one centralized Redis that served as Gateway Redis, we also implemented the Redis for each **Project/Building.** For each, all the information, say, updates, errors, discovery\_points, etc, will be transferred from the Gateway Redis to Project Redis once we receive from driver, via project CLIENT program, and all the operations will be conducted there. By doing this, transparency, distribution and maintainability are the vectors we seriously considered and improved.

In addition, we accomplished the parallel reading, parsing and pushing [updates ](extraction-module.md#updates)by duplicating the extraction modules and creating services for each of them. In specific, say we have two extraction modules for project A, each will take half of updates, parse and push to database. The philosophy behind this design is that usually once a updates data coming, we parse them by one extraction module, but it will risk our virtual machine to go to a resource consumption peak and result in some consequential effects to other program if the updates are enormous. Using of parallelization technique directly help us eliminate this kind of possible negative influences by turning one huge peak to two smaller wave.

In this upgrading, we not only covered one single extraction mode, but two simultaneously. The previous version of extraction only serve one extraction mode without possible switching, we instead make our package adaptive to the mode changes. In other words, it's expected that when there is a need to switch extraction mode from haystack \(read extraction list from hs\_points and push to hs\_measure\) to convention \(read extraction list from file and push to TL\_measures\) or vice versa could be managed automatically without human intervention. Even for both mode at the same time, the package is also capable for doing it. This will give us a huge amount of conveniences when a switching is required.

To give a general picture of how entire architecture looks like:

![](.gitbook/assets/microsoftteams-image-3-.png)

NOTE: Any terms that causes confusion, please refer to [Terminologies Section](extraction-module.md#terminologies-definition)

### LEA

![](.gitbook/assets/image.png)

## Environments

This project is massively applied with Redis database, so it's mandatory to have it installed, both redis server and python redis packages.

A list of required external packages:

1. redis
2. psutil
3. environ
4. pandas
5. pymysql
6. sqlalchemy

## Modules

The Extraction package includes two modules: Init\_Extraction and Extraction, both are able to run independently but need small modifications. The following description will speak from the perspective of a whole.

The **Init\_Extraction** module is the first of first module to execute in Extraction package, it takes the full responsibilities of **reading from config file and updating of every modules to Redis database**, and other modules, such as Extraction will read config from Redis rather than from file. Considering it as root, all other program takes information from here.

The **Extraction** module takes care of the duties of reading, parsing and pushing the updates that we received from Brainbox AI driver. The updates data originally sit on our [**Gateway** Redis](extraction-module.md#gateway-redis) for its corresponding controller id, and it will be taken by our **project** CLIENT program and push to our [**Project** Redis](extraction-module.md#project-redis). Starting from Project Redis with all piece of information related to this project, the extraction module is able to perform the parsing operation.

### Init\_Extraction

In general, Init Extraction is the modules that responsible for [initializing ](extraction-module.md#initialization)the extraction.

#### Flowchart

The init extraction module in Extraction package is designed in the below way.

![](.gitbook/assets/microsoftteams-image-2-1-.png)

#### Highlight Points:

1. If the extraction list changes, which indicates we would like to subscribe/tell our driver there is some different points we would like to have visuals, we have to first send a command "unsubscribe\_all" to our driver via gateway redis, which informs our driver "we don't want you to send any points value back to us", then follows the steps mentioned [here ](extraction-module.md#initialization)to re-initialize the extraction
2. Another circumstance that a re-initialization is required is at the time the extraction mode is switched \(convention, haystack, dual mode\), this has been automated in the extraction module. 
3. This module is run as an independent service at tgw VM with infinite loop

### Extraction

This module will handle the scenario of needs of reading, parsing and pushing updates to our database.

#### Flowchart

![](.gitbook/assets/image%20%287%29.png)

#### Highlight Points:

1. As stated in [Overview](extraction-module.md#overview), Extraction supports mode switching and dual mode
2. As stated in [Overview](extraction-module.md#overview), Extraction supports parallel reading, parsing and pushing
3. All extractions modules, say, extraction\_1 and extraction2, are run as independent service in tgw VM, the [wave\_counter ](extraction-module.md#wave-counter)will be different but close enough. 
4. [Extraction cycle](extraction-module.md#extraction-cycle) table will log all the information related to this extraction loop, either success or fail. One crucial point is the wave counter in extraction cycle table match the data in measures.

## Quick Start

### Install modules

The installation follows the general steps:

1. Download the code: git clone [https://git.brainboxai.net/DataStreams/LEA\_IO\_modules\_Tridium](https://git.brainboxai.net/DataStreams/LEA_IO_modules_Tridium)    
   1. enter your git username and password
2. Change the configuration file adaptively 
   1. list of configuration files:
      1. **InitExtraction/extraction\_config.json**: all necessary attributes for running the extraction modules
      2. **DataSmith\_\#/DS\_config.json:** majorly use redis channel name, the preloaded message redis name, and the size of each pushing to database
      3. **utils/.env**: the password and username of the database
      4. **utils/config.json**: the configuration required for connecting to database, such as table names, database name and server name
      5. **CLIENT\_CONFIGURATION.json**: the extraction mode is controller here; also the controller ids in this project are included here \(a must; otherwise the program won't executed properly\). 
      6. **service\_creator/service**_**\_**_**CONFIGURATION.json**: the configurations for creating the system service for running the modules on background. Support multiple services creation.
3. Set up the Redis socket server:
   1. No need to change anything in this step
   2. Simply run _redisConfiguration/prepare\_redis\_conf.py_ 
   3. \(when there is a need to stop such server, run _redisConfiguration/steop\_redis\_service.py_, this will not affect the data that already stored in redis database\)
4. Once you feel confident about the configurations setting, feel free to use sudoer permission to run _service\_creator/service\_creator.py_
5. After creation of service, it's necessary to do extra quality assurance, which is use command line of `sudo systemctl status yourservicename.service` to check if the service is up and run, in case of failure, run `sudo systemctl restart yourservicename.service`, and recheck the status. If still failed, double check your configurations file.

### Frequent scenarios and its measures

This section assumes the related service is running on background.

#### Reinit extraction

In _InitExtraction/extraction\_config.json_, swith attribute of "init" from false to **true,** save and exit. The service of _InitEXTRACTION_ will take of the changes and proceed to reinit the extraction.

#### Change of extraction mode

Under CLIENT\_CONFIGURATION.json file, there are two attributes: 

1. haystack\_ext\_mode: boolean; controls if the extraction is on [haystack extraction mode](extraction-module.md#single-mode)
2. dual\_ext\_mode: boolean; controls if the extraction is on [dual extraction mode](extraction-module.md#dual-mode)

Please note that, the program will first check if we are in dual extraction mode, if yes, it will omit if haystack extraction mode is on or off. Likewise, if dual is off, then the program will check which single mode we are directing, haystack or conventional.

#### Debug mode

The debug mode is available for modules:

1. Client
2. Init Extraction/ Extraction
3. Data Smith

Once this mode is on, the verbose will be maximized and print every details on each turning points.

#### Change of extraction module duties

By default, each extraction package is equipped with two extraction modules, and their responsibilities was split by the _extraction\_config.json,_ for example, 

1. **"extraction1\_duties":\["f6b9c110-83da-30ed-91ab-9c3dbd6b108e\_updates\_300\_\[0-4\]"\]**
2. **extraction2\_duties":\["f6b9c110-83da-30ed-91ab-9c3dbd6b108e\_updates\_300\_\[4-9\]"\],**

#### Change of pushing to database size

#### Extraction only for locally, stop pushing to cloud database

## Functionalities

### Dual mode

The extraction package supports the dual mode of extraction. In particular, it allows itself to subscribe the points to driver based on a joint extraction list from conventional and haystack approach, in which, conventional mode push updates to TL measures table, while haystack mode push to hs measures.

One thing that requires extra concern is the joint extraction list. Under the circumstance of lack of a joint extraction list that prepared by Data Mapping team, we shall merge the individual extraction list together by the key of "point address", e.g. _/Drivers/BacnetNetwork/$31$2d10/points/CLG$2dMAXFLOW_. Among them, the conventional extraction list is preserved in the file format of csv that provided by Data Mapping team and haystack extraction list could be found at hs points of each project in AWS database.

In another case, the joint extraction list is prepared by Data Mapping team and could be directly read as data frame and do the initialization of extraction.

### Single mode

As its mode name implies, this is only for extraction for one single mode, either Haystack or Conventional approach. The difference is they may have different extraction list, and push to different tables \(haystack: hs\_measures; conventional: TL\_measures\). 

### Rich report

In this module, we are committed to log program running details as much as possible. It would be one of the ways to collect data for the future integration of Machine Learning application and Software Engineer, such as Smart Operation project.

The examples that covered are:

1. program running time in secs
2. whether or not the extraction is success
3. the number of normal/status updates
4. which extraction mode is currently on
5. The offset between creation of updates and creation of database row
6. CPU/memory consumption
7. updates with status, if any
8. current cycle configuration

### Memory garbage collection

The execution of program is in the format of hierarchy. By this, it means it was implemented with multiple levels of calling modules. Particularly, there is a main script \(e.g. EXTRACTION\_1.py\) that takes care of running the script runner \(e.g. extraction.py\). In this design, the main script will be embedded with a **while True** loop, and script runner will contain a runner\_main function that serves as the caller of main business logic of the present script, which equips a **for loop** and maximum iteration number in the configuration file. 

By doing so, after running out of the for loop of script runner, it will exit itself to clean up all the allocated memory in order to achieve the goal of the garbage collection operation. In the next cycle, the program will go back to main script while loop, and start another script runner.  

### Massive use of Redis

As you may notice, this is a project that massively applied Redis, which is a memory-based database with extremely fast read/write ability.

Two major applications:

1. Use of redis database to store temporary data from our driver, extraction list, module status, etc
2. Use of redis channel to listen and collect data from another endpoint, in specific, the parsed data will be pushed to redis channel and [Data Smith](extraction-module.md#data-smith) module will take this data and push to its requested database

### Immediate effect of configuration change

This applies to every modules involved in this project, extraction, init\_extration, client, etc. 

The extraction configuration file changes would be immediately effect to the production, for example, while one of the attribute modified in configuration file, the [init\_extraction module](extraction-module.md#modules) will read it and write to redis, which will later be consumed by other program, same for client module.  

### Debug mode

This enables the maintainer debug the module with print almost all information in the pipeline, so that it's time-saving to manually print the message of where the error occurred.

With the debug mode on, the module will output the data in every major turning points and give a heads-up for every details for debugging. 

### Push to extraction cycle table even if extraction failed

Even if the extraction of points failed, the module is able to push the error message to the database, along with the extraction mode, 

### Rich options 

#### Write to database

#### Delete after read

#### Points with status

1. Whether or not write points with status to database, default at NO, in other words, all points come with status \(abnormal\) will collected into the **detailed report** column in **extraction cycle** table by default
2. Options to choose which status write to database and rest of them will be kept in the **detailed report** column in **extraction cycle** table 
3. Options to choose which columns of status points kept into detailed report: default at \[point value, status, URI, point id\]

### Extraction module duties

### Heart beat/modules status

code to reboot

### Data smith pushing by configurable size

### Auto check the changes in configuration file





## Output

## Use cases/Examples

## Terminologies Definition

### Gateway/Project Redis

#### Gateway Redis

This is the primary, centralized Redis database for storing all communication information related to the specific controller id, such as raw request, raw response, controller id, session time, current time, etc, this is the transfer hub of our brainbox gateway and client driver.

#### Project Redis

This is designed for each of project/building, which means that for each of project, one Redis object will be available for each to place any related data with a controller id as prefix, such as controllerID\_current\_time. It is beneficial for maintenance of each project/building as well as modules.

### Data Smith

Part of LEA, an application of redis channel, listen to the specific channel and then push the received data to desired database table.

### Initialization

By meaning of _Initialization,_ it follows these steps:

1. Based on the extraction list \(the points we desired to acquire from driver\)
   1. split the list to several smaller chunks according to the configurable chunk size
   2. assign those chunks name, such as _"controllerID\_1", "controllerID\_COV\_2",_ varied by the type of extraction \(ordinary or COV\) 
   3. put everything together and send to our gateway redis and wait for driver to take and execute
   4. Another name for the above procedures is **Subscription -** in short, tell our driver we want those points value to be sent to us in a certain time interval.
2. Reinit means we are refreshing the extraction list/subscription list and let the driver be aware of the latest ones. 

### Updates

Very straightforward, after we initialized extraction, the driver will then send the points values that we have in the extraction list to us, in a timely manner. Those bunch of points values, we name'em Updates.

### Wave counter

The unix time of a timestamp

### Extraction cycle

The table that sits in database that responsible to log all the information related to this specific extraction cycle

The wave counter will be used as a reference to match the updates data back to measures table

### Heart beat

## Configurations

## Issues

## Collaborators

