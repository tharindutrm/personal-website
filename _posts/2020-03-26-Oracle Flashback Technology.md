---
title: "Oracle Flashback Technology"
---

# 1. ORACLE FLASHBACK TECHNOLOGY
Use to recover data from Logical corruptions. Most of the Flashback technologies depend on the UNDO data to retrieve older data

1.1 Database Parameters

  - DB_FLASHBACK_RETENTION_TARGET: Time limit for the deleted data to be retained
    
    Alter System Set DB_FLASHBACK_RETENTION_TARGET=4320 

  - DB_RECOVERY_FILE_DEST_SIZE: Size limit for the maximum data that can be retained.
  
    Alter System Set DB_RECOVERY_FILE_DEST_SIZE=536870912

  - DB_RECOVERY_FILE_DEST: Location where the data needs to be retained.
  
    Alter System Set DB_RECOVERY_FILE_DEST='/Source/File/ '

## 1.2 Check Parameters

SQL> show parameter DB_FLASHBACK_RETENTION_TARGET

SQL> show parameter DB_RECOVERY_FILE_DEST_SIZE

SQL> show parameter db_recovery_file_dest

SQL> show parameter undo_retention
