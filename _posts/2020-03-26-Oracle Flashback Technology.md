---
layout: default
title: "Oracle Flashback Technology"
comments: true
---

Use to recover data from Logical corruptions. Most of the Flashback technologies depend on the **UNDO** data to retrieve older data

## 1. Set Database Parameters ##

**1.1 `DB_FLASHBACK_RETENTION_TARGET`** Setting

`DB_FLASHBACK_RETENTION_TARGET`


Time limit (in minutes) for the deleted data to be retained.

{% highlight SQL %}
SQL> Alter System Set DB_FLASHBACK_RETENTION_TARGET=4320;
{% endhighlight %}
 
**1.2 DB_RECOVERY_FILE_DEST_SIZE**

Size limit for the maximum data that can be retained.
   
{% highlight SQL %}

SQL> Alter System Set DB_RECOVERY_FILE_DEST_SIZE=2G;

{% endhighlight %}

**1.3 DB_RECOVERY_FILE_DEST**

Location where the data needs to be retained.
    
{% highlight SQL %}

SQL> Alter System Set DB_RECOVERY_FILE_DEST='/Source/File/';

{% endhighlight %}

## 2. Check Database Parameters

{% highlight SQL %}

SQL> show parameter DB_FLASHBACK_RETENTION_TARGET;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_flashback_retention_target	     integer	 4320

SQL> show parameter DB_RECOVERY_FILE_DEST_SIZE;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest_size	     big integer 2G

SQL> show parameter db_recovery_file_dest;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest		     string	 /ora01/app/oracle/fast_recovery_area
db_recovery_file_dest_size	     big integer 2G

SQL> show parameter undo_retention;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
undo_retention			     integer	 900

SQL> select TABLESPACE_NAME,RETENTION from dba_tablespaces where tablespace_name like 'UNDO%';

TABLESPACE_NAME 	       RETENTION
------------------------------ -----------
UNDOTBS1		       NOGUARANTEE

{% endhighlight %}

**Switching from  NOGUARANTEE to GUARANTEE**

{% highlight SQL %}

SQL> ALTER TABLESPACE UNDOTBS1 RETENTION GUARANTEE;

SQL> select TABLESPACE_NAME,RETENTION from dba_tablespaces where tablespace_name like 'UNDO%';

TABLESPACE_NAME 	       RETENTION
------------------------------ -----------
UNDOTBS1		       GUARANTEE

SQL> select log_mode,flashback_on from v$database;

LOG_MODE     FLASHBACK_ON
------------ ------------------
ARCHIVELOG NO

{% endhighlight %}

## 3. Flashback Scenarios


### 3.1 DROP TABLE


Flashback drop is used to restore accidentally dropped tables and depended objects. After restoring the table will be renamed as its original whereas the indexes will have system generated names. 

**Before doing flashback, confirm that the dropped object has not been purged**

{% highlight SQL %}

SQL> FLASHBACK TABLE EMP2 TO BEFORE DROP;

{% endhighlight %}

### CASE 1. Tabaes Without Indexes

{% highlight SQL %}

SQL> select * from emp2;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> drop table emp2;

Table dropped.

SQL> select * from emp2;
select * from emp2
              *
ERROR at line 1:
ORA-00942: table or view does not exist


SQL> select object_name,original_name from recyclebin;

OBJECT_NAME		       ORIGINAL_NAME
------------------------------ --------------------------------
BIN$oeQS8wUwDSvgUwEAAH9fzA==$0 EMP2

SQL> flashback table emp2 to before drop;

Flashback complete.

SQL> select * from emp2;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

{% endhighlight %}

**CASE 2. Tables With Indexes**

{% highlight SQL %}

SQL> create index ind3 on emp3(id);

Index created.

SQL> select table_name,index_name from user_indexes where index_name='IND3';

TABLE_NAME		       INDEX_NAME
------------------------------ ------------------------------
EMP3			       IND3

SQL> drop table emp3;

Table dropped.

SQL> flashback table emp3 to before drop;

Flashback complete.

SQL> select table_name,index_name from user_indexes where index_name='IND3';

no rows selected

SQL> select table_name,index_name from user_indexes where table_name='EMP3';   

TABLE_NAME		       INDEX_NAME
------------------------------ ------------------------------
EMP3			       BIN$oeQS8wUxDSvgUwEAAH9fzA==$0

SQL> alter index "BIN$oeQS8wUxDSvgUwEAAH9fzA==$0" rename to IND3;

Index altered.

SQL> select table_name,index_name from user_indexes where index_name='IND3';

TABLE_NAME		       INDEX_NAME
------------------------------ ------------------------------
EMP3			       IND3

{% endhighlight %}

### 3.2 Oracle Flashback Version Query ###


This feature helps to view all the versions of all the rows that ever existed in one or more tables in between two points in time or system change numbers (SCN).This feature also depends on **UNDO** data.

(Update a table row several times and flashback)

{% highlight SQL %}

Select .. Versions Between SCN | Timestamp And SCN | Timestamp

Select Version_Xid,Dept_Id
From Dept
Version Between SCN (Referencing A Start And End SCN);

Select Version_Xid,Dept_Id
From Dept
Version Between TIMESTAMP (Referencing A Start And End Timestamp);

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> update emp3 set salary=19900 where id=3;

1 row updated.

SQL> commit;

Commit complete.

SQL> update emp3 set salary=27700 where id=3;

1 row updated.

SQL> commit;

Commit complete.

SQL> col versions_starttime for a27
SQL> col versions_endtime for a27
SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	27700
	 4	20000

SQL> select versions_starttime,versions_endtime,id,salary from emp3 versions between scn minvalue and maxvalue where id=3;

VERSIONS_STARTTIME	    VERSIONS_ENDTIME			ID     SALARY
--------------------------- --------------------------- ---------- ----------
28-MAR-20 11.05.19 AM						 3	27700
28-MAR-20 11.05.07 AM	    28-MAR-20 11.05.19 AM		 3	19900
			    28-MAR-20 11.05.07 AM		 3	20000

SQL> select versions_startscn,versions_endscn,id,salary from emp3 versions between scn minvalue and maxvalue where id=3;

VERSIONS_STARTSCN VERSIONS_ENDSCN	  ID	 SALARY
----------------- --------------- ---------- ----------
	  1243556			   3	  27700
	  1243543	  1243556	   3	  19900
			  1243543	   3	  20000

SQL> select * from emp3 as of timestamp TO_TIMESTAMP('28-03-2020 11:05:07','DD-MM-YYYY HH24:MI:SS');

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> select id,salary from emp3 as of scn 1243543 where id=3;

	ID     SALARY
---------- ----------
	 3	19900

SQL> select id,salary from emp3 as of scn 1243542 where id=3;

	ID     SALARY
---------- ----------
	 3	20000

SQL> flashback table emp3 to scn 1243542;
flashback table emp3 to scn 1243542
                *
ERROR at line 1:
ORA-08189: cannot flashback the table because row movement is not enabled

SQL> alter table emp3 enable row movement;

Table altered.

SQL> flashback table emp3 to scn 1243542;

Flashback complete.

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> select id,salary from emp3 as of timestamp sysdate-interval '10' minute where id=3;

	ID     SALARY
---------- ----------
	 3	27700

SQL> select id,salary from emp3 as of timestamp sysdate-interval '1' minute where id=3;

	ID     SALARY
---------- ----------
	 3	20000

{% endhighlight %}

#### Update multiple rows ####

{% highlight SQL %}

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> update emp3 set salary=1999;

4 rows updated.

SQL> commit;

Commit complete.

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	 1999
	 2	 1999
	 3	 1999
	 4	 1999

SQL> select versions_startscn,versions_endscn,id,salary from emp3 versions between scn minvalue and maxvalue where id=3;

VERSIONS_STARTSCN VERSIONS_ENDSCN	  ID	 SALARY
----------------- --------------- ---------- ----------
	  1244311			   3	   1999
	  1243820	  1244311	   3	  20000
	  1243820			   3	  27700
	  1243556	  1243820	   3	  27700
	  1243543	  1243556	   3	  19900
			  1243543	   3	  20000

6 rows selected.

SQL> select * from emp3 as of scn 1244311;

	ID     SALARY
---------- ----------
	 1	 1999
	 2	 1999
	 3	 1999
	 4	 1999

SQL> select * from emp3 as of scn 1244310;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> flashback table emp3 to scn 1244310;

Flashback complete.

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

{% endhighlight %}

### 3.3 Flashback Transcation Query ###


Flashback Transaction query allows viewing changes made by single transaction or all transactions during a period of time.

Transaction Query Feature uses the FLASHBACK_TRANSACTION_QUERY view for retrieving transaction information.

It enables you to get the undo-sql statement for a transaction. You can use that undo sql statement to revert back the changes made by the transaction. 
**To get undo-sql statement, supplement logging should be enabled.**

**Undo-sql**

Insert >> delete 

Update>>update

Delete>> insert

{% highlight SQL %}

SELECT Logon_User, Operation, Start_Timestamp, Undo_Sql
FROM Flashback_Transaction_Query
WHERE Xid In (
SELECT  Versions_Xid
FROM  Emp BETWEEN TIMESTAMP (Systimestamp - Interval '6' Minute) AND Systimestamp);

{% endhighlight %}

{% highlight SQL %}

SQL> select  SUPPLEMENTAL_LOG_DATA_ALL from v$database;

SUP
---
NO

SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database add  SUPPLEMENTAL log data;

Database altered.
SQL> alter database open;

SQL> grant update on hr.emp3 to scott;

SQL> conn hr/hr
Connected.
SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

SQL> conn scott/scott
Connected.
SQL> update hr.emp3 set salary=4999 where id=2;

1 row updated.

SQL> commit;

SQL> conn hr/hr
Connected.

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	 4999
	 3	20000
	 4	20000
	 
SQL> show user;
USER is "SYS"
SQL> desc flashback_transaction_query;
 Name					   Null?    Type
 ----------------------------------------- -------- ----------------------------
 XID						    RAW(8)
 START_SCN					    NUMBER
 START_TIMESTAMP				    DATE
 COMMIT_SCN					    NUMBER
 COMMIT_TIMESTAMP				    DATE
 LOGON_USER					    VARCHAR2(30)
 UNDO_CHANGE#					    NUMBER
 OPERATION					    VARCHAR2(32)
 TABLE_NAME					    VARCHAR2(256)
 TABLE_OWNER					    VARCHAR2(32)
 ROW_ID 					    VARCHAR2(19)
 UNDO_SQL					    VARCHAR2(4000)

SQL> select UNDO_SQL from flashback_transaction_query where LOGON_USER ='SCOTT';

UNDO_SQL
--------------------------------------------------------------------------------
delete from "SYS"."AUD$" where ROWID = 'AAAVRLAABAAAXHHAAO';

update "HR"."EMP3" set "SALARY" = '20000' where ROWID = 'AAAVphAAEAAAAIrAAB';


SQL> update "HR"."EMP3" set "SALARY" = '20000' where ROWID = 'AAAVphAAEAAAAIrAAB';

1 row updated.

SQL> commit;

Commit complete.

SQL> select * from emp3;

	ID     SALARY
---------- ----------
	 1	20000
	 2	20000
	 3	20000
	 4	20000

{% endhighlight %}

### 3.4 Flashback Database ###

This feature is used to recover the database from logical corruptions.
- It is used to perform PITR(point in time recovery) of database. 

- Faster than the traditional way of performing PITR 

- It requires flashback log files which are generated when we enabled flashback database. 

- Only for Logical errors .not for physical errors

Check estimated size

{% highlight SQL %}

SQL> SELECT ESTIMATED_FLASHBACK_SIZE FROM V$FLASHBACK_DATABASE_LOG;

{% endhighlight %}

{% highlight SQL %}

SQL> select flashback_on from v$database;

FLASHBACK_ON
------------------
NO

SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database flashback on; 
alter database flashback on
*
ERROR at line 1:
ORA-38706: Cannot turn on FLASHBACK DATABASE logging.
ORA-38707: Media recovery is not enabled.


SQL> alter database archivelog;

Database altered.

SQL> alter database flashback on;

Database altered.

SQL> alter database open;

Database altered.

SQL> select flashback_on from v$database; 

FLASHBACK_ON
------------------
YES

SQL> select NAME from v$flashback_database_logfile ;

NAME
--------------------------------------------------------------------------------
/ora01/app/oracle/fast_recovery_area/ORCL/flashback/o1_mf_h7xvy1by_.flb
/ora01/app/oracle/fast_recovery_area/ORCL/flashback/o1_mf_h7xvy3yp_.flb

{% endhighlight %}

**Drop SCOTT schema and flashback database**

{% highlight SQL %}

SQL> select username from dba_users where username='SCOTT';

USERNAME
------------------------------
SCOTT

SQL> drop user SCOTT cascade;

User dropped.

SQL> select username from dba_users where username='SCOTT';

no rows selected

SQL>  select timestamp_to_scn(to_date('03/28/2020 12:05:00','mm/dd/yyyy hh24:mi:ss')) from dual;

TIMESTAMP_TO_SCN(TO_DATE('03/28/202012:05:00','MM/DD/YYYYHH24:MI:SS'))
----------------------------------------------------------------------
							       1247017
							       
SQL> create user u2 identified by u2;

User created.

SQL> select username from dba_users where username='U2';

USERNAME
------------------------------
U2

SQL> shutdown immediate;
SQL> startup mount;

SQL> flashback database to scn 1247017;

Flashback complete.

SQL> alter database open resetlogs;

Database altered.

SQL> select username from dba_users where username='U2';

no rows selected

SQL> select username from dba_users where username='SCOTT';

USERNAME
------------------------------
SCOTT

SQL> conn SCOTT/scott
Connected.
SQL> show user;
USER is "SCOTT"
SQL> 

{% endhighlight %}

SCOTT user exists.but U2 is not available.
Database succesfully completed the flashback operation to given SCN.








