---
title: "Oracle Flashback Technology"
---

# How to Enable/Disable Oracle Archivelog Mode

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

## 2.Shutdown Database and Mount Database

```SQL
SQL> shutdown immediate;
SQL> startup mount;
```
## 3.Change the Parameter
```SQL
SQL> alter database archivelog;
SQL> alter database open;
```
## 4.Verify Configuration

```SQL
SQL> archive log list
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     12
Next log sequence to archive   14
Current log sequence	       14
```
## 5.Disable Archivelog Mode

```SQL
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database noarchivelog;
SQL> alter database open;

SQL> archive log list;
Database log mode	       No Archive Mode
Automatic archival	       Disabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     12
Current log sequence	       14
```
