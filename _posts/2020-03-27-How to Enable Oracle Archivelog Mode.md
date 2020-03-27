---
layout: default
title: "Oracle Flashback Technology"
comments: true
---

# How to Enable Oracle Archivelog Mode

## 1. Check Current Configuration

`[oracle@centos7 ~]$ sqlplus / as sysdba`

```SQL
SQL> archive log list
Database log mode	       No Archive Mode
Automatic archival	       Disabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     12
Current log sequence	       14
```

## 2.Shutdown database and Mount database

```SQL
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database archivelog;
SQL> alter database open;
```
## 3.Verify Configuration

```SQL
SQL> archive log list
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     12
Next log sequence to archive   14
Current log sequence	       14
```