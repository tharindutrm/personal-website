---
title: "Oracle Flashback Technology"
---

# ORACLE FLASHBACK TECHNOLOGY
Use to recover data from Logical corruptions. Most of the Flashback technologies depend on the **UNDO** data to retrieve older data

**1. Set Database Parameters**

  - DB_FLASHBACK_RETENTION_TARGET: Time limit for the deleted data to be retained
    
    SQL> Alter System Set DB_FLASHBACK_RETENTION_TARGET=4320;

  - DB_RECOVERY_FILE_DEST_SIZE: Size limit for the maximum data that can be retained.
  
    SQL> Alter System Set DB_RECOVERY_FILE_DEST_SIZE=536870912;

  - DB_RECOVERY_FILE_DEST: Location where the data needs to be retained.
  
    SQL> Alter System Set DB_RECOVERY_FILE_DEST='/Source/File/';

**2.Check Parameters**

SQL> show parameter DB_FLASHBACK_RETENTION_TARGET;

SQL> show parameter DB_RECOVERY_FILE_DEST_SIZE;

SQL> show parameter db_recovery_file_dest;

SQL> show parameter undo_retention;

SQL> select TABLESPACE_NAME,RETENTION from dba_tablespaces where tablespace_name like 'UNDO%';

SQL> select log_mode,flashback_on from v$database;

**3.SCENARIOS**

