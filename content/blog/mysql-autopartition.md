---
path: awesome-images
date: 2021-06-27 18:06:28
title: Mysql auto partition
description: How to automatically create partition in mysql
---

Notes of Mysql auto partition which mimicks the Cookbooks from O'reilly.

## Problem

You need to create a new partition for a mysql table every month, automatically.

## Solution

Create a stored procedure to add a new partion for a month, and create a scheduled event to call the partition creation procedure before every month starts. 

Make sure at least one partition created before partitions can be added with 'ADD PARTITION'.

```SQL
DELIMITER $$

USE `your_database_name`$$

DROP PROCEDURE IF EXISTS `create_partition`$$
CREATE PROCEDURE `create_partition`()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION ROLLBACK;
    START TRANSACTION;

    SET @partition_end=TIMESTAMP(DATE_FORMAT(DATE_ADD(CURRENT_DATE(),INTERVAL 2 MONTH),'%Y-%m-01'), "00:00:00");
    SET @partition_name=DATE_FORMAT(DATE_ADD(CURRENT_DATE(),INTERVAL 1 MONTH),"p%Y%m");

    SET @s=CONCAT('ALTER TABLE your_table_name ADD PARTITION (PARTITION ',@partition_name,' VALUES LESS THAN (''',@partition_end,'''))');

    SELECT @s;
    PREPARE stmt FROM @s;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    COMMIT ;
END$$

DROP PROCEDURE IF EXISTS `init_partition`$$
CREATE PROCEDURE init_partition()
BEGIN
    DECLARE partition_count INT DEFAULT 0;

    SELECT count(PARTITION_NAME)
		INTO partition_count
    FROM information_schema.`PARTITIONS`
    WHERE TABLE_NAME = 'your_table_name';

    --  Create partition for the current month
    IF partition_count = 0 THEN
        SET @partition_end=TIMESTAMP(DATE_FORMAT(DATE_ADD(CURRENT_DATE(),INTERVAL 1 MONTH),'%Y-%m-01'), "00:00:00");
        SET @partition_name=DATE_FORMAT(CURRENT_DATE(),"p%Y%m");

        SET @s=CONCAT('ALTER TABLE your_table_name PARTITION BY RANGE COLUMNS ( time ) (PARTITION ',@partition_name,' VALUES LESS THAN (''',@partition_end,'''))');

        PREPARE stmt FROM @s;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END IF;
END$$

-- create initial partition if needed
call init_partition()$$

-- Schedule at 20:00 last day of each month
CREATE EVENT add_partition_event
ON SCHEDULE EVERY 1 MONTH
STARTS DATE_ADD(LAST_DAY(CURRENT_DATE()), INTERVAL 20 HOUR)
DO
    BEGIN
        CALL your_database_name.`create_partition`;
	END$$
DELIMITER ;

```

## Discussion

`create_partition` Creates a partition for the next month. For example, It creates a partition `p202107` if it's called in this month(202106). To create this partition, we need a SQL statement like 
```SQL
ALTER TABLE your_table_name ADD PARTITION (PARTITION p202107 VALUES LESS THAN ('2021-08-01 00:00:00')
```

`DATE_FORMAT(DATE_ADD(CURRENT_DATE(),INTERVAL 2 MONTH),'%Y-%m-01')` gives the first day of two month after the current month, ie. 20210801, and `TIMESTAMP` is used to append the "00:00:00". The partition name `p202107` is created with `DATE_FORMAT(DATE_ADD(CURRENT_DATE(),INTERVAL 1 MONTH),"p%Y%m")`.  


The above `ALTER TABLE` doesn't specify which column to partition on, and for it to add a new partition successfully, at least one partition should exist. `init_partition` uses the information in `information_schema` to determine if a partition already exists, and create one for the current month if not. The partitions are created on `time` column in this example.

`CREATE EVENT` creates an event that triggers at 20:00 on last day of each month. So this script needs to be run before this time. Usually `CREATE EVENT` need a future time after `STARTS`. If a past time is passed, the next schedule time will be calculated from that past time.
