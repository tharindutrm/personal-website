---
layout: default
title: "Oracle Flashback Technology"
comments: true
---

Use to recover data from Logical corruptions. Most of the Flashback technologies depend on the **UNDO** data to retrieve older data

## 1. Set Database Parameters 


1.1 DB_FLASHBACK_RETENTION_TARGET: Time limit (in minutes) for the deleted data to be retained.

{% highlight SQL %}  

SQL> Alter System Set DB_FLASHBACK_RETENTION_TARGET=4320;

{% endhighlight %}
 
1.2 DB_RECOVERY_FILE_DEST_SIZE: Size limit for the maximum data that can be retained.
   
{% highlight SQL %}

SQL> Alter System Set DB_RECOVERY_FILE_DEST_SIZE=2G;

{% endhighlight %}

1.3 DB_RECOVERY_FILE_DEST: Location where the data needs to be retained.
    
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

ALTER TABLESPACE UNDOTBS1 RETENTION GUARANTEE;

SQL> select TABLESPACE_NAME,RETENTION from dba_tablespaces where tablespace_name like 'UNDO%';

TABLESPACE_NAME 	       RETENTION
------------------------------ -----------
UNDOTBS1		       GUARANTEE

SQL> select log_mode,flashback_on from v$database;

LOG_MODE     FLASHBACK_ON
------------ ------------------
NOARCHIVELOG NO

{% endhighlight %}

## 3. Flashback Scenarios

### 3.1 DROP TABLE

Flashback drop is used to restore accidentally dropped tables and depended objects. After restoring the table will be renamed as its original whereas the indexes will have system generated names. 

**Before doing flashback, confirm that the dropped object has not been purged**

{% highlight SQL %}

SQL>FLASHBACK TABLE EMP2 TO BEFORE DROP;

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

###3.2 Oracle Flashback Version Query###

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

{% endhighlight %}








