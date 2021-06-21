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
      1. audit\_trail
      2. commands
      3. extraction\_cycle
      4. TL\_measures/hs_\__measures \(optional; for haystack mode\)
      5. hs\_poins \(optional; for haystack mode\)
   2. if any of missing, please refer to [this appendix](untitled-1.md#create-statement-for-database-table) section for the _create statement._ 
   3. if all exists, please use the statement below as reference to correct the project database table to make it standardized. Please note that for table of _**hs points**,_ the order of column names matters.  
   4. 

## Understand the Detailed Report in Extraction Cycle

## Frequent Asked Questions

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









