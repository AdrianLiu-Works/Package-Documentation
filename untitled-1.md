# Comprehensive Guide for LEA Deployment

## What is LEA?

LEA serves as a management layer to oversee the status of all services under this project. If one of the service failed and went down, the LEA will order this project to turn on the safe mode, which is directly off all services except Extraction and Data Smith. By doing so, LEA offers an additional safety layer to command the building.

## What is LEA Environment for TRIDIUM?

The LEA environment is the terminology that refer to the package/modules that takes care of the data flow and command handling all TRIDIUM projects. It includes the modules of:

* **Client**
  * This is responsible for:
    * transferring the data from GATEWAY redis to Project redis
    * push the latest client configuration to redis
    * consume all commands or writes or sessions time changes and write to GATEWAY redis for driver to take
    * [detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/LEA.md#the-overall-design)
    * [another detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/LEA.md#gatewayproject-redis)
* **Init\_Extraction**
  * This is responsible for:
    * initialize the extraction based on the given extraction mode and list
    * populate every configuration file to redis, keep them as fresh as possible
    * [detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Extraction.md#init_extraction)
    * [another detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Extraction.md#initialization)
* **Extraction**
  * This is responsible for:
    * extract the updated points from redis object once we receive them and populate to database
    * record the resource consumptions and populate to database
    * record the details of each extraction cycle and populate to database
    * [detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Extraction.md#extraction-module)
    * [another detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Extraction.md#extraction)
* **Data Smith**
  * This is responsible for:
    * listen to the specific channel and then push the received data to desired database table
    * majorly used in extraction data pushing
    * [detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Extraction.md#data-smith%5D)
* **Chart Smith**
  * This is responsible for:
    * loop to fetch new commands from DB
    * preprocessing captured commands
    * push different kind of commands to different channel \([Report Written Smith](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/README.md/#report-written-smith) and [Report Smith](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/README.md/#report-smith)\)
    * [!!! important detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Chart_Command_Smith.md)
* **Command Smith**
  * This is responsible for:
    * filter non-writable commands \(expired or executed\) that captured from Chart Smith
    * write written commands to redis and wait for [Report Written Smith](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/README.md/#report-written-smith)
    * [!!! important detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Chart_Command_Smith.md)
* **Code Gease**
  * This is responsible for:
    * handle the future commands and safety loop commands
    * [!!! important detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Chart_Command_Smith.md)
* **Report Smith**
  * This is responsible for:
    * report any non-writable and non-translatable commands to database
    * [!!! important detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Chart_Command_Smith.md)
* **Report Written Smith**
  * This is responsible for:
    * report all writable commands to database
    * check if the _write_ success or not
    * [!!! important detailed description](https://git.brainboxai.net/DataStreams/Lea_Tridium/src/branch/development/docs/Chart_Command_Smith.md)

## How to Use it?

### Prerequisites

Before deploying the LEA environment to the production, it's required to check the prerequisites/checklists to make sure this project is applicable for deploying LEA environment. The order of the following lists not matters:

1. Database tables
   1. for each of schema/project, it's mandatory to have the following tables ready
      1. **audit\_trail** \(for audit of execution of commands\)
      2. **commands** \(for records of historical requested commands\)
      3. **extraction\_cycle** \(for records/information of each extraction cycle\)
      4. **TL\_measures**/**hs**_**\_**_**measures** \(optional; for haystack mode\)
      5. **hs\_poins** \(optional; for haystack mode\)
   2. if any of missing, please refer to [this appendix](untitled-1.md#create-statement-for-database-table) section for the _create statement._ 
   3. if all exists, please use the statement below as reference to correct the project database table to make it standardized. Please note that for table of _**hs points**,_ the order of columns matters.  
2. LEO environment
   1. The LEA environment will interact with LEO via the cloud heartbeat of each other. In other words, they will check the heartbeat for each other to see if they are active within a fixed time interval, if not, a safe mode will be triggered and as a result, only the safe mode accepted modules will be allowed to continue running and the rest will be stopped immediately. 
   2. [Deploy LEO environment ](untitled-1.md#deploy-leo-environment)
3. Redis server
   1. check the redis server connection:
      1. ```text
         redis-cli ping
         ```

         it's supposed to return `PONG`

      2. otherwise, please refer to the online tutorial to install redis server to the new VM or direct this request to ECO team.
4. Extraction list column names

   This is for the standardization/scalability purpose

   1. Convention
      1. while the extraction list has to be standardized already, it's worthy to mention, the extraction list csv file columns have to follow this order

         ```text
         [
               "URI",
               "DB_name",
               "value",
               "obj_type",
               "controller_id",
               "system",
               "point_name",
               "units",
               "extraction_frequency",
               "writable",
               "COV",
               "internal_index",
               "c_id"
            ]
         ```

         adjust either extraction configuration _convention\_ext\_list\_colName_ attribute or the extraction list csv file. 
   2. Haystack

      1. make sure the order and spell of the column names from _hs\_points_ table are strictly obey: 

      ```text
      [
            "id",
            "DB_name",
            "equipId",
            "value",
            "kind",
            "units",
            "writable",
            "controller_id",
            "obj_type",
            "URI",
            "addressType",
            "COV",
            "extraction_frequency",
            "maxExtractionFrequency",
            "createdBy",
            "createdAt",
            "updatedBy",
            "updatedAt"
         ]
      ```

      adjust the database table or  _haystack\_ext\_list\_colName_  attribute in the configuration file, accordingly. 

### Prepare

NOTE: All the steps conducted in this section is inside `./prototype` directory

We have to collect the related building information to be able to push the modules running. Here is the list of all you need:

1. building host name
2. device id
3. database name
4. database server location
5. relevant onboarding email list
6. the database user
7. the database password
8. all controller ids in a list format
9. extraction mode \(dual, convention, or haystack?\)
10. the extraction list file location \(if convention/dual extraction mode\)
11. number of points that will be processed by one extraction module \(parallel computing\)

After acquiring, fill the acquired information to _BuildingSpecs.json,_ follow the indication of the column names. Then  do `sudo python3 PrepareDeployment.py`, this will create a directory with the host name, which contains all the configurations and source codes. 

### Deploy

NOTE: All the steps conducted in this section is inside `./the_building_host_name` directory

After preparing for the source code and configurations, we shall proceed to the next step.

1. change the directory to the host name directory
2. `sudo ./TRIGGER.py deploy`
3. all services that are specified in the _SystemConfig.json_ will be created a system service, all information in _BuildingSpecs.json_ and extraction list directory will be emptied, as well as the database credentials that sit in the _specs.json_
4. under the same directory, do `sudo ./TRIGGER.py freshstart LEA` to do a freshstart over LEA, a message of "redis\_custom.conf is created!" will pop up if success, which indicates the success running of both LEA layer and the project redis
5. LEA will start the required/rest modules on its own, such as extraction, data smith, chart smith, etc. 

## Understand the Report in Extraction Cycle

### Flag

This will indicate the success of the extraction

### Num Updates to DB

This will indicate the number of updates that are pushed to database

### Num Updates Status

This will indicate the number of updates with status that are **NOT** pushed to database. They will be stored in the detailed report blob object. By default, all status points value will be pushed to database

This could be enabled/disabled by switching the attribute _write status to db_ from true to false, then all status will be directed to detailed report blob object instead of pushing to db. 

### Num Updates Null

This will indicate the number of updates with null values. This is by default filtering all null values to the detailed report blob object. So when querying the specific point that with null values, it won't be in the database. 

### Script N

This will indicate which extraction module runs this cycle.

### Ext Mode

This will indicate which extraction mode are we in, options: conventional, haystack, dual

### Error Message

For each of extraction cycle, if no errors occurred, it will indicate no error message; Otherwise, it will print the shorten format of the traceback message

### Offset

This indicates the offset time between our current unix time and the updates unix time. This will explain the freshness of the updates. The number means how long does the driver take to send us the updates. Negative number or incredible number implies the clocking sync issue on Client side. If necessary, please advice the Client to reset their system clock. 

### Logging

This offers a more detailed error message \(if applicable\) of **each steps** that the extraction pipeline/cycle went through. This is also the good reference to check which step cause the issue and what the error message is.

### CPU Consumption

This indicates the CPU consumption of this specific extraction cycle

### Memory Consumption

This indicates the Memory consumption of this specific extraction cycle

### Current Overall CPU percentage

This indicates the CPU consumption of the entire VM. Note that the higher overall CPU percentage will result in lower processing time and longer time in each of extraction cycle

### Current Overall Memory percentage

This indicates the Memory consumption of the entire VM

### Updates with Status

The updates that come with status. This will be intentionally empty if the _write status to db_ is **False.** Otherwise, all the updates with status will transferring here. 

### Updates with Null

The updates that come with value Null. This will by default stored all the null updates.

### Extraction Config

The specific configuration for this extraction cycle

### Data Smith Config

The specific configuration for the data smith of this extraction cycle

## Understand the Message in LEA Safe Mode Email Notification 

LEA will go to safe mode if one of the following circumstances are met:

1. one of the running modules is down \(by checking the status of the module service\)
2. heartbeat from server/LEO is missing within a certain time interval
3. connection to database cannot be established

In such case, the LEA module will notify the people in the mailing list about the triggering of safe mode. The body text will also include the tail of the LEA log message so that the receivers are able to refer to it and debug the modules, this is a preventive measure that the LEA log are cleared.  

## Use of Offered Tools

All following tools are independent python scripts

### TRIGGER

This tool is the main one that covers all the possible operations to the **modules service** in our use cases. 

#### it offers 

1. '**start**': start the modules
2. '**freshstart**': start the module that is first time running; exclusive for LEA
3. '**stop**': stop the modules
4. '**restart**':  restart the modules
5. '**deploy**': deploy \(create the system service\) the modules
6. '**view**': print the tail of the service log
7. '**status**': print the status of the service
8. '**safe**': convert the module to safe mode; exclusive for LEA

#### how to use

`sudo ./TRIGGER.py <options> <modules_name>`

for example:  `sudo ./TRIGGER.py freshstart LEA`

### redisOverview

This tool mainly covers all the possible operations to the **project redis server** in our use cases. This is also a debugging tool to check the status of each attribute and make sure they are updated timely or not lost or not never there. 

#### it offers

1. '-k' or '--key':  show the content of the key
2. '-s' or '--status': show status of given argument
3. '-a' or '--arg': specify the argument for a key, fetched in the format of hashmap
4. '-n' or '--keyarg': return all keys that is being fetched from a hashmap
5. '-ak' or '--allkey': print all keys in redis
6. '-m' or '--memory': print the memory for the given key
7. '-ms' or '--memoryStats': display the memory statistics 
8. '-del' or '--deletekey': delete a given key

#### how to use

Make sure you are in the `brainbox` permission

`python3 redisOverview.py <options> <key(optional)>`

for example: 

with key

`python3 redisOverview.py -k 234fv2-32f2-wrbtywee-234v4_current`_`_`_`time -a 234fv2-32f2-wrbtywee-234v4_creation_time_utc` 

OR without key/argument

`python3 redisOverview.py -ms`

#### frequent use case

1. check if the controller id current time is changing \(means CLIENT and driver are working properly\)
2. check if the heart beat of each module is updating \(means the modules services are running\)
3. check if all keys are available \(means CLIENT is working fine\)
4. check the data
5. remove a no longer needed key 

### RELEASE

This tool will release all the written point values on the driver side by changing the session time \(to be specific, eco session time\). By referring to _release_, it means that all the points that we wrote to the driver will be erased and the controller will be given back to the BMS system. 

#### how to use

Make sure you are in the `brainbox` permission

`python3 RELEASE.py`

The tool will automatically ensure the success of the release. There is a waiting time for confirming. All the tracked written points will be populated to the database audit trail table with LEA\_release tag if release is success, otherwise, an error will be raised. 

 ==============================================================

**NOTE: ALL THE FOLLOWING TOOLS ARE AVAILABLE INSIDE freqUseCases DIRECTORY**

**ALL PERMISSIONS ARE IN brainbox**

**ALL TOOLS WILL PRINT OKAY! AFTER DONE; OTHERWISE AN ERROR**

**USE CASES:**

1. do the additional operations other than the offered functionalities of available running modules
2. debug the project
3. other special operations that fits users' needs

### discover

This tool is used to discover all the possible points that available in the TRIDIUM network. It will be saved as  discovered\_points\_&lt;date of time&gt;.json file under the main directory `./` 

The file will be useful for mapping/creating extraction list purpose

#### how to use it

`python3 discover_points.py`

The entire process may take up to 10 - 15 mins, depends on the size of the buildings

### set command

Based on the BrainboxAI Gateway protocol, you can select the commands that you would like to send to the driver and ask it to do the corresponding operations. 

#### how to use it 

`nano set_commands.py`

modify based on your need, make sure the controller id is correct

`python3 set_commands.py`

### unsubscribe all

This will clear the subscription list on the driver side, as a result, updates will no longer send to our gateway unless the next subscription. 

#### how to use it

`python3 unsubscribe_all.py`

### get config

This will return the available driver configurations for all controllers, such as version of driver etc. The message will be printed.

#### how to use it

`python3 get_config.py`

### get session time

This will return the session time for all controllers. The message will be printed

#### how to use it

`python3 get_session_time.py`

### get subscription

This will return the current subscription list for all controllers, include valid and invalid ones. The message will be printed

#### how to use it

`python3 get_subscription.py`

### get write

This will return the write points value

#### how to use it

`python3 get_write.py`

### set config

This will set the different configurations to the driver, such as the max message interval, details could be referred to BrainboxAI Protocol

#### how to use it 

`nano set_config.py`

modify based on your need, note that this will apply changes to all controllers

`python3 set_config.py`

### set session time

This will set the new cloud/eco session time to the driver. Consequently, all the written points will be released. 

#### how to use it

`python3 set_session_time.py`

### set subscribe

This will allow the user to subscribe the specific points on the specific controller. 

#### how to use it 

`nano set_subscribe.py`

modify based on your need. Example is available in the script.

`python3 set_subscribe.py`

## Frequent Asked Questions / Debugging Guide



## Appendix

### Create statement for database table

#### audit\_trail

```text
CREATE TABLE `audit_trail` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `command_id` bigint(20) DEFAULT NULL,
  `outcome` varchar(35) NOT NULL,
  `algo_name` varchar(30) DEFAULT NULL,
  `point_id` varchar(50) DEFAULT NULL,
  `net_id` varchar(70) DEFAULT NULL,
  `controller_id` varchar(50) DEFAULT NULL,
  `obj_type` varchar(20) DEFAULT NULL,
  `priority` varchar(5) DEFAULT NULL,
  `point_value` varchar(20) DEFAULT NULL,
  `cpriority` varchar(10) DEFAULT NULL,
  `creation_date_utc` datetime DEFAULT NULL,
  `expiration_date_utc` datetime DEFAULT NULL,
  `execution_date_utc` datetime NOT NULL,
  `write_status` tinyint(1) NOT NULL,
  `wave_counter` bigint(20) NOT NULL,
  `verification` varchar(9) DEFAULT NULL,
  `in_repeat` varchar(9) DEFAULT NULL,
  `row_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `system_ref` varchar(40) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `Index_date` (`execution_date_utc`),
  KEY `Index_PtID` (`point_id`),
  KEY `my_index` (`point_id`,`creation_date_utc`)
) ENGINE=InnoDB AUTO_INCREMENT=1324 DEFAULT CHARSET=latin1;
```

#### commands

```text
CREATE TABLE `commands` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `algo_name` varchar(30) DEFAULT NULL,
  `point_id` varchar(90) DEFAULT NULL,
  `priority` varchar(1) DEFAULT NULL,
  `point_value` varchar(20) DEFAULT NULL,
  `creation_date_utc` datetime DEFAULT NULL,
  `expiration_date_utc` datetime DEFAULT NULL,
  `verification` varchar(9) DEFAULT NULL,
  `in_repeat` varchar(9) DEFAULT NULL,
  `system_ref` varchar(45) DEFAULT NULL,
  `row_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `Index_date` (`creation_date_utc`),
  KEY `Index_PtID` (`point_id`),
  KEY `index_PtID_date` (`point_id`,`creation_date_utc`)
) ENGINE=InnoDB AUTO_INCREMENT=104 DEFAULT CHARSET=latin1;
```

#### extraction\_cycle

```text
CREATE TABLE `extraction_cycle` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `cycle_duration` int(11) NOT NULL,
  `wave_counter` int(11) DEFAULT NULL,
  `report` varchar(150) DEFAULT NULL,
  `detailed_report` mediumblob,
  `row_created` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `id_UNIQUE` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=14313 DEFAULT CHARSET=latin1;
```

#### TL\_measures

```text
CREATE TABLE `TL_measures` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `point_id` varchar(90) NOT NULL,
  `controller_id` varchar(45) NOT NULL DEFAULT '',
  `system_id` varchar(45) NOT NULL DEFAULT '',
  `point_name` varchar(45) NOT NULL,
  `point_type` varchar(45) NOT NULL,
  `unit` varchar(5) NOT NULL DEFAULT 'C',
  `point_value` float DEFAULT NULL,
  `creation_date_utc` datetime NOT NULL,
  `wave_counter` bigint(20) unsigned NOT NULL,
  `row_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `Uniquekey` (`controller_id`,`system_id`,`point_name`,`point_type`,`unit`,`wave_counter`),
  KEY `Index_date` (`creation_date_utc`),
  KEY `point_id` (`point_id`),
  KEY `kaizen` (`point_id`,`creation_date_utc`),
  KEY `wave` (`wave_counter`)
) ENGINE=InnoDB AUTO_INCREMENT=26447608 DEFAULT CHARSET=utf8;
```

#### hs\_measures

```text
CREATE TABLE `hs_measures` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `point_id` int(11) NOT NULL,
  `point_value` varchar(45) NOT NULL,
  `wave_counter` int(11) NOT NULL,
  `created_by` varchar(45) NOT NULL,
  `row_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `creation_date_utc` timestamp NOT NULL,
  `source_ts_utc` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `ix_hs_measures_creation_date_utc` (`creation_date_utc`),
  KEY `ix_hs_measures_wave_counter` (`wave_counter`),
  KEY `ix_hs_measures_point_id` (`point_id`),
  CONSTRAINT `hs_measures_ibfk_1` FOREIGN KEY (`point_id`) REFERENCES `hs_points` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=26617892 DEFAULT CHARSET=latin1;
```

#### hs\_points

```text
CREATE TABLE `hs_points` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `dis` varchar(45) NOT NULL,
  `equipId` int(11) NOT NULL,
  `curVal` varchar(45) NOT NULL,
  `kind` varchar(45) NOT NULL,
  `unit` varchar(45) DEFAULT NULL,
  `writable` tinyint(4) NOT NULL,
  `controllerId` varchar(90) NOT NULL,
  `objectType` varchar(45) NOT NULL,
  `pointAddress` varchar(90) NOT NULL,
  `addressType` varchar(45) NOT NULL,
  `cov` tinyint(4) NOT NULL,
  `extractionFrequency` int(11) NOT NULL,
  `maxExtractionFrequency` int(11) NOT NULL,
  `createdBy` varchar(45) NOT NULL,
  `createdAt` timestamp NOT NULL,
  `updatedBy` varchar(45) NOT NULL,
  `updatedAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=15561 DEFAULT CHARSET=latin1;
```

### Deploy LEO environment

1. In order to activate LEO environment, which is majorly deployed in LEA/Calvin VM, it's required to have the related building information handy:
   1. building id
   2. device id
   3. building code
   4. host name
   5. database name
   6. weather station
   7. database server location
2. Follow the same procedure of [LEA](untitled-1.md#prepare)

