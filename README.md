# greenplum-predicate-pushdown

This tutorial performs predicate pushdown from Greenplum into an Oracle database.

To start, create/ log into an Oracle database, create a table and insert a record:
 
`Create table test (id number, name varchar2(20));`

`Insert into test (id, name) values (1,'druid');`

`Select * from test;`
```
ID  NAME                
----- -----
1 druid           	
```

Now run the following steps/commands on the Greenplum MASTER node

1.

Install cx-oracle to connect from python to Oracle
https://cx-oracle.readthedocs.io/en/latest/installation.html#quick-start-cx-oracle-installation 

2.
 
Clone this repo.
 
3. Update oracele username, password, hostname and service name of the database you are connecting to
 
`$ vi pushpredicate.py `

```
import os
import cx_Oracle
from os import environ
connection = cx_Oracle.connect("<orausername>", "<orapassword>", "<orahost>/<ora service name>")
cursor = connection.cursor()
if environ.get('QUERYFILTER') is not None:
	querystring= "select * from TEST where " + environ.get('QUERYFILTER')
else:
	querystring = "select * from TEST"
cursor.execute(querystring)
result=cursor.fetchall()
for row in result:
	if row is None:
    		print ''
	print row[0],'|',row[1]
connection.close()
```

4. executepush.sh contains the path of your Oracle client libraries, and the predicate that you want pushed to Oracle.
Update LD_LIBRARY_PATH and point to Oracle client libraries on your greenplum master, and QUERYFILTER to the predicate you want.
<br><br/>
Here we are pushing the Name field and the value we want. <br/>
Note: LD_LIBRARY_PATH (or corresponding variable) in the script below needs to be customized for OS which runs on your greenplum master instance. The greenplum node used to test this was CentOS and the Oracle client libraries were installed in the path below:

## Case 1: When predicate exists in Oracle DB

$ cat executepush.sh 

```
#!/bin/bash
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/oracle/12.1/client64/lib/
export QUERYFILTER="NAME='druid'"
python /home/gpadmin/pushpredicate.py
 ```
5.
```
$ psql
psql (8.3.23)
Type "help" for help.
 
gpadmin=#
``` 
6.
```
gpadmin=# create external web table test1(ID int, NAME varchar) EXECUTE '/home/gpadmin/executepush.sh' on MASTER FORMAT 'TEXT' ( DELIMITER '|' );
 
CREATE EXTERNAL TABLE
 ```
 7.
``` 
gpadmin=# select * from test1;
 id |  name 
----+--------
  1 |  druid
(1 row)
 ```
##  Case 2: When predicate value DOES NOT EXIST in Oracle 

8.
Update LD_LIBRARY_PATH and point to Oracle client libraries, and QUERYFILTER to the predicate you want.

`$ vi executepush.sh `
``` 
#!/bin/bash
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/oracle/12.1/client64/lib/
export QUERYFILTER="NAME='groot'"
python /home/gpadmin/pushpredicate.py
 ```
9.
```
gpadmin=# create external web table test1(ID int, NAME varchar) EXECUTE '/home/gpadmin/executepush.sh' on MASTER FORMAT 'TEXT' ( DELIMITER '|' );
 
CREATE EXTERNAL TABLE
``` 
10..
``` 
gpadmin=# select * from test1;
 id | name
----+------
(0 rows)
```
