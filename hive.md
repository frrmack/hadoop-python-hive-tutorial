## Using Hive

ssh to your cloud computer and switch to the hduser. Go to the hduser's home.

```bash
$ su hduser
$ cd
```
Let's download Hive.

```bash
$ wget http://mirror.cogentco.com/pub/apache/hive/hive-0.14.0/apache-hive-0.14.0-bin.tar.gz
```
This is a zipped file. Extract it.

```bash
$ tar -xzvf apache-hive-0.14.0-bin.tar.gz 
```
Now, in hduser's home directory, you have a directory called `apache-hive-0.14.0-bin`.
We are going to edit your `.bashrc` file to make an environment varible called HIVE_HOME
so that Hadoop and Hive know where it lives (like we did with the Hadoop setup).

```bash
$ emacs ~/.bashrc
```
To the end of the `.bashrc` file that we are editing, add the following lines

```bash
export HIVE_HOME=/home/hduser/apache-hive-0.14.0-bin
export PATH=$PATH:$HIVE_HOME/bin
```
Save and close. Now we need to run the `.bashrc` to make sure HIVE_HOME is defined.

```bash
$ source ~/.bashrc
```
That's it. Now we installed Hive. We have one last setup step left.
Hive basically behaves like a SQL database that lives on the hdfs.
To be able to do that, it needs a temporary files folder and a place to store the underlying data (in the hdfs).
We need to create these directories in the hdfs and give the necessary permissions.
> Obviously, to be able to make changes to the hdfs, make sure your hadoop cluster is up and running.
> Do `jps` to check if the namenode, secondary namenode and the datanode are up.
> If not, you need to start them with the `start-dfs.sh` first, you can check to hadoop tutorial to see how we did that.

```bash
$ hdfs dfs -mkdir -p /tmp
$ hdfs dfs -mkdir -p /user/hive/warehouse
$ hdfs dfs -chmod g+w /tmp
$ hdfs dfs -chmod g+w /user/hive/warehouse
```
Aaand, your Hive is ready to rock your world. You can run it by typing

```
$ hive
```
You should get a hive prompt, like this:

```
hive> _
```
Hive's syntax is (almost) identical to SQL. So let's load up some data and use it. First, exit:

```
hive> exit;
```

You should be back at your regular prompt now. Let's download some baseball data.
```bash
$ wget http://seanlahman.com/files/database/lahman-csv_2014-02-14.zip
```
This is a zipped file with a bunch of csv files, each is a sql table.
These tables are full of baseball statistics from 2013. Let's unzip this.
First we need to switch back to a user with sudo powers so we can install unzip, 
then switch back to hduser and unzip it.
(Of course, switch `irmak` below with your own username)

```bash
$ su irmak
$ sudo apt-get install unzip
$ su hduser
$ mkdir baseballdata
$ unzip lahman-csv_2014-02-14.zip -d baseballdata
```

Now you have a bunch of csv files in the `baseballdata` directory.
You can think of each csv as a table in a baseball database.
Let's create one Hive table and read a csv into that table.
This is the exact analog of loading a csv into a sql table.
Let's do this with the `baseballdata/Master.csv`.
First, take a look at that csv file.

```bash
$ head baseballdata/Master.csv
```

You should see this:

```Text
playerID,birthYear,birthMonth,birthDay,birthCountry,birthState,birthCity,deathYear,deathMonth,deathDay,deathCountry,deathState,deathCity,nameFirst,nameLast,nameGiven,weight,height,bats,throws,debut,finalGame,retroID,bbrefID
aardsda01,1981,12,27,USA,CO,Denver,,,,,,,David,Aardsma,David Allan,205,75,R,R,2004-04-06,2013-09-28,aardd001,aardsda01
aaronha01,1934,2,5,USA,AL,Mobile,,,,,,,Hank,Aaron,Henry Louis,180,72,R,R,1954-04-13,1976-10-03,aaroh101,aaronha01
aaronto01,1939,8,5,USA,AL,Mobile,1984,8,16,USA,GA,Atlanta,Tommie,Aaron,Tommie Lee,190,75,R,R,1962-04-10,1971-09-26,aarot101,aaronto01
aasedo01,1954,9,8,USA,CA,Orange,,,,,,,Don,Aase,Donald William,190,75,R,R,1977-07-26,1990-10-03,aased001,aasedo01
abadan01,1972,8,25,USA,FL,Palm Beach,,,,,,,Andy,Abad,Fausto Andres,184,73,L,L,2001-09-10,2006-04-13,abada001,abadan01
abadfe01,1985,12,17,D.R.,La Romana,La Romana,,,,,,,Fernando,Abad,Fernando Antonio,220,73,L,L,2010-07-28,2013-09-27,abadf001,abadfe01
abadijo01,1854,11,4,USA,PA,Philadelphia,1905,5,17,USA,NJ,Pemberton,John,Abadie,John W.,192,72,R,R,1875-04-26,1875-06-10,abadj101,abadijo01
abbated01,1877,4,15,USA,PA,Latrobe,1957,1,6,USA,FL,Fort Lauderdale,Ed,Abbaticchio,Edward James,170,71,R,R,1897-09-04,1910-09-15,abbae101,abbated01
abbeybe01,1869,11,11,USA,VT,Essex,1962,6,11,USA,VT,Colchester,Bert,Abbey,Bert Wood,175,71,R,R,1892-06-14,1896-09-23,abbeb101,abbeybe01
```

Ok. We need to create at table with these following column headers.
To be able to read it better, let's print each column name in a line.
```bash
$ head -n 1 baseballdata/Master.csv | tr ',' '\n'
```
You should see

```Text
playerID
birthYear
birthMonth
birthDay
birthCountry
birthState
birthCity
deathYear
deathMonth
deathDay
deathCountry
deathState
deathCity
nameFirst
nameLast
nameGiven
weight
height
bats
throws
debut
finalGame
retroID
bbrefID
```
What did we do up there? `head` shows us only several lines at the beginning of a file.
The option `-n 1` tells it to show only the first line (`-n 5` would have shown the first five).
The output of this is a line of column headers separated by commas.
We pipe this output into `tr ',' '\n'`, which converts (or *tr*anslates) every `,` character into a newline character (`\n`).
That way, we get a new line every time there was a comma.
Ok, great. This will help us construct the table.
One last thing we need to do is to remove the first line from the file, though, to make it easier to upload it to Hive.
We can use `tail` for this, which, just like head, only shows several lines of a file, but the **last** lines instead of the first.
`tail -n 4` shows the last 4 lines, for example. `tail -n +8` shows all lines including and after the 8th line.
So, to get rid of the first line, we want `tail -n +2`. Let's pipe the output into head to check if it will indeed work:

```bash
$ tail -n +2 baseballdata/Master.csv | head
```
should show
```Text
aardsda01,1981,12,27,USA,CO,Denver,,,,,,,David,Aardsma,David Allan,205,75,R,R,2004-04-06,2013-09-28,aardd001,aardsda01
aaronha01,1934,2,5,USA,AL,Mobile,,,,,,,Hank,Aaron,Henry Louis,180,72,R,R,1954-04-13,1976-10-03,aaroh101,aaronha01
aaronto01,1939,8,5,USA,AL,Mobile,1984,8,16,USA,GA,Atlanta,Tommie,Aaron,Tommie Lee,190,75,R,R,1962-04-10,1971-09-26,aarot101,aaronto01
aasedo01,1954,9,8,USA,CA,Orange,,,,,,,Don,Aase,Donald William,190,75,R,R,1977-07-26,1990-10-03,aased001,aasedo01
abadan01,1972,8,25,USA,FL,Palm Beach,,,,,,,Andy,Abad,Fausto Andres,184,73,L,L,2001-09-10,2006-04-13,abada001,abadan01
abadfe01,1985,12,17,D.R.,La Romana,La Romana,,,,,,,Fernando,Abad,Fernando Antonio,220,73,L,L,2010-07-28,2013-09-27,abadf001,abadfe01
abadijo01,1854,11,4,USA,PA,Philadelphia,1905,5,17,USA,NJ,Pemberton,John,Abadie,John W.,192,72,R,R,1875-04-26,1875-06-10,abadj101,abadijo01
abbated01,1877,4,15,USA,PA,Latrobe,1957,1,6,USA,FL,Fort Lauderdale,Ed,Abbaticchio,Edward James,170,71,R,R,1897-09-04,1910-09-15,abbae101,abbated01
abbeybe01,1869,11,11,USA,VT,Essex,1962,6,11,USA,VT,Colchester,Bert,Abbey,Bert Wood,175,71,R,R,1892-06-14,1896-09-23,abbeb101,abbeybe01
abbeych01,1866,10,14,USA,NE,Falls City,1926,4,27,USA,CA,San Francisco,Charlie,Abbey,Charles S.,169,68,L,L,1893-08-16,1897-08-19,abbec101,abbeych01
```
Looks like it's working. So let's write this into a temporary file and then overwrite the original with this new temp file so Master.csv no longer has the header line.
```bash
$ tail -n +2 baseballdata/Master.csv > tmp && mv tmp baseballdata/Master.csv
```
The `&&` means do the first part first, and when it finished, do what follows the `&&`.

Ok. we removed the header. Let's make sure we did
```bash
$ head baseballdata/Master.csv
aardsda01,1981,12,27,USA,CO,Denver,,,,,,,David,Aardsma,David Allan,205,75,R,R,2004-04-06,2013-09-28,aardd001,aardsda01
aaronha01,1934,2,5,USA,AL,Mobile,,,,,,,Hank,Aaron,Henry Louis,180,72,R,R,1954-04-13,1976-10-03,aaroh101,aaronha01
aaronto01,1939,8,5,USA,AL,Mobile,1984,8,16,USA,GA,Atlanta,Tommie,Aaron,Tommie Lee,190,75,R,R,1962-04-10,1971-09-26,aarot101,aaronto01
aasedo01,1954,9,8,USA,CA,Orange,,,,,,,Don,Aase,Donald William,190,75,R,R,1977-07-26,1990-10-03,aased001,aasedo01
abadan01,1972,8,25,USA,FL,Palm Beach,,,,,,,Andy,Abad,Fausto Andres,184,73,L,L,2001-09-10,2006-04-13,abada001,abadan01
abadfe01,1985,12,17,D.R.,La Romana,La Romana,,,,,,,Fernando,Abad,Fernando Antonio,220,73,L,L,2010-07-28,2013-09-27,abadf001,abadfe01
abadijo01,1854,11,4,USA,PA,Philadelphia,1905,5,17,USA,NJ,Pemberton,John,Abadie,John W.,192,72,R,R,1875-04-26,1875-06-10,abadj101,abadijo01
abbated01,1877,4,15,USA,PA,Latrobe,1957,1,6,USA,FL,Fort Lauderdale,Ed,Abbaticchio,Edward James,170,71,R,R,1897-09-04,1910-09-15,abbae101,abbated01
abbeybe01,1869,11,11,USA,VT,Essex,1962,6,11,USA,VT,Colchester,Bert,Abbey,Bert Wood,175,71,R,R,1892-06-14,1896-09-23,abbeb101,abbeybe01
abbeych01,1866,10,14,USA,NE,Falls City,1926,4,27,USA,CA,San Francisco,Charlie,Abbey,Charles S.,169,68,L,L,1893-08-16,1897-08-19,abbec101,abbeych01
```

Indeed it's gone. Alright. Let's upload this to hive. First, we need to upload it to hdfs.
(of course, change `irmak` to whichever directory you have in hdfs) 
```bash
$ hdfs dfs -mkdir -p /user/irmak/baseballdata
$ hdfs dfs -put baseballdata/Master.csv /user/irmak/baseballdata
```
We created a new directory in hsfs and uploaded the csv to it.
Let's make sure it's there.
```bash
$ hdfs dfs -ls /user/irmak/baseballdata
Found 1 items
-rw-r--r--   1 hduser supergroup    2422684 2015-03-11 22:12 /user/irmak/baseballdata/Master.csv
```
It is. Awesome. Time to run hive
```bash
$ hive

Logging initialized using configuration in jar:file:/home/hduser/apache-hive-0.14.0-bin/lib/hive-common-0.14.0.jar!/hive-log4j.properties
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hduser/apache-hive-0.14.0-bin/lib/hive-jdbc-0.14.0-standalone.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]  
hive> 
```
We now have the Hive prompt. (Ignore the SLF4J warning, it's an unimportant logging thing).
Let's create the table

```sql
hive> CREATE TABLE IF NOT EXISTS Master
      (playerID STRING,
      birthYear INT,
      birthMonth INT,
      birthDay INT,
      birthCountry STRING,
      birthState STRING,
      birthCity STRING,
      deathYear INT,
      deathMonth INT,
      deathDay INT,
      deathCountry STRING,
      deathState STRING,
      deathCity STRING,
      nameFirst STRING,
      nameLast STRING,
      nameGiven STRING,
      weight INT,
      height INT,
      bats STRING,
      throws STRING,
      debut STRING,
      finalGame STRING,
      retroID STRING,
      bbrefID STRING)
      COMMENT 'Master Player Table'
      ROW FORMAT DELIMITED
      FIELDS TERMINATED BY ','
      STORED AS TEXTFILE;
OK
Time taken: 1.752 seconds
```
And let's load the data
```sql
hive> LOAD DATA INPATH '/user/irmak/baseballdata/Master.csv' OVERWRITE INTO TABLE Master;
Loading data to table default.master
Table default.master stats: [numFiles=1, numRows=0, totalSize=2422684, rawDataSize=0]
OK
Time taken: 1.166 seconds
```
And it's in!
We now have a Hive table. The best part of hive is, when you make a query (that most of the time looks **exactly** like a sql query), Hive automatically creates the map and reduce tasks, runs them over the hadoop cluster, and gives you the answer, without you having to worry about any of it. If your question is easily represented in the form of a sql query, Hive will take care of all the dirty work for you. The table might be spread over thousands of computers, but you don't need to think hard about that at all.

Let's start easy. Let's find out how many players we have in this table.
```sql
hive> SELECT COUNT(playerid) FROM Master;
Query ID = hduser_20150311224646_00211363-82a3-49f0-aac2-b0d6abb4caf9
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Job running in-process (local Hadoop)
Hadoop job information for Stage-1: number of mappers: 0; number of reducers: 0
2015-03-11 22:46:52,488 Stage-1 map = 100%,  reduce = 100%
Ended Job = job_local747935831_0002
MapReduce Jobs Launched: 
Stage-Stage-1:  HDFS Read: 9690750 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
18354
Time taken: 2.424 seconds, Fetched: 1 row(s)
```
As you can see, hive reports on the job it is setting up and the map reduce tasks that job entails, reports on the progress, and finally gives the result: There are 18354 players.

Let's do something that would require more involved mapper and reducer functions, but is pretty straightforward with Hive. Let's get the weight distribution of the players in this table.
```sql
hive> SELECT weight, count(playerID) FROM Master GROUP BY weight;
Query ID = hduser_20150311223636_6eda794b-8400-4054-9fce-b1080af16f99
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks not specified. Estimated from input data size: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Job running in-process (local Hadoop)
Hadoop job information for Stage-1: number of mappers: 0; number of reducers: 0
2015-03-11 22:36:44,510 Stage-1 map = 0%,  reduce = 0%
2015-03-11 22:36:45,753 Stage-1 map = 100%,  reduce = 0%
2015-03-11 22:36:46,797 Stage-1 map = 100%,  reduce = 100%
Ended Job = job_local1584830750_0001
MapReduce Jobs Launched: 
Stage-Stage-1:  HDFS Read: 4845382 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
NULL	877
65	1
120	3
125	4
126	2
127	1
128	1
129	2
130	11
132	1
133	1
134	1
135	16
136	4
137	2
138	9
139	2
140	59
141	3
142	15
143	9
144	7
145	94
146	6
147	10
148	30
149	8
150	279
151	8
152	33
153	16
154	41
155	317
156	34
157	40
158	78
159	14
160	794
161	18
162	61
163	52
164	53
165	996
166	27
167	60
168	191
169	29
170	1273
171	16
172	113
173	54
174	69
175	1532
176	66
177	25
178	188
179	28
180	1604
181	21
182	75
183	61
184	45
185	1587
186	71
187	108
188	75
189	24
190	1444
191	12
192	62
193	49
194	33
195	1082
196	38
197	36
198	52
199	2
200	1012
201	7
202	18
203	19
204	19
205	634
206	5
207	18
208	22
209	10
210	620
211	4
212	15
213	6
214	5
215	463
216	5
217	9
218	12
219	3
220	412
221	3
222	4
223	4
225	243
226	5
227	2
228	6
230	189
233	2
234	3
235	112
237	3
240	105
241	1
242	1
243	1
244	2
245	48
250	52
254	1
255	22
257	1
260	21
265	8
269	1
270	8
275	8
280	5
283	1
285	3
290	2
295	2
310	1
320	1
Time taken: 9.265 seconds, Fetched: 132 row(s)
```
As you can see, a simple GROUP BY statement takes care of everything. Easier than writing and executing specific mapreduce functions.

In this manner, you can do sql-like queries over tons of data that live in the hdfs in a distributed state. Since hdfs and mapreduce have overheads, it will not be as fast as a sql query on data that fits a single machine, but you now get the answers in parallel, and are able to do sql queries over hundreds of terabytes of data.

Let's upload another table and see how joins work. Salaries.csv has four columns: year, team, league, player, salary. It only has salary information for after 1984, but it's pretty extensive.

Let's remove the header and upload it to hdfs
```bash
hive> exit;
$ tail -n +2 baseballdata/Salaries.csv > tmp && mv tmp baseballdata/Salaries.csv
$ hdfs dfs -put baseballdata/Salaries.csv /user/irmak/baseballdata
```
Switch to hive, create the table and load the data.
```sql
$ hive

Logging initialized using configuration in jar:file:/home/hduser/apache-hive-0.14.0-bin/lib/hive-common-0.14.0.jar!/hive-log4j.properties
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hduser/apache-hive-0.14.0-bin/lib/hive-jdbc-0.14.0-standalone.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
hive> CREATE TABLE IF NOT EXISTS Salaries                                                
      (yearID INT, teamID STRING, lgID STRING, playerID STRING, salary INT)              
      COMMENT 'Salary Table for Players'                                                 
      ROW FORMAT DELIMITED                                                               
      FIELDS TERMINATED BY ','                                                           
      STORED AS TEXTFILE;                                                                
OK
Time taken: 2.502 seconds
hive> LOAD DATA INPATH '/user/irmak/baseballdata/Salaries.csv' OVERWRITE INTO TABLE Salaries;
Loading data to table default.salaries
Table default.salaries stats: [numFiles=1, numRows=0, totalSize=724918, rawDataSize=0]
OK
Time taken: 2.237 seconds
hive> SHOW TABLES;
OK
master
salaries
Time taken: 0.203 seconds, Fetched: 2 row(s)
```
Mission accomplished. We have two tables now: `Master` and `Salaries`. (By the way, you should have noticed by now that nothing in Hive is case sensitive).

Let's do a somewhat more complicated query that involves two tables. Let's take a look at the upper end of the weight distribution among the players and their salaries. Here is the breakdown of the query:
>
> ```sql
> SELECT Salaries.yearID, Master.nameFirst, Master.nameLast, Master.weight, Salaries.salary
> ```
> This is what we want to read: The first & last name of the player, their weight, and their salary at a specific year. Salary and year comes from the salary table and the rest from the master table.
>
> ```sql
> FROM Master JOIN Salaries ON (Master.playerID = Salaries.playerID)
> ```
> This is how we combine the information from both tables. We want the row for a player to connect with the salary rows for that player. Note that there are multiple rows for the same player in the Salaries table (for multiple years).
>
> ```sql
> WHERE Master.weight > 270;
> ```
> Only show the players who weigh more than 270 pounds. Also note that we don't have yearly weights, but a single weight statistic for each player (that is reported in the Master table).
>

So, let's put this query together and execute it.
```sql
hive> SELECT Salaries.yearID, Master.nameFirst, Master.nameLast, Master.weight, Salaries.salary FROM Master JOIN Salaries ON (Master.playerID = Salaries.playerID) WHERE Master.weight > 270;                 
Query ID = hduser_20150312000707_f2a73817-d862-4080-9d23-8c0e77960e65
Total jobs = 1
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hduser/apache-hive-0.14.0-bin/lib/hive-jdbc-0.14.0-standalone.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
Execution log at: /tmp/hduser/hduser_20150312000707_f2a73817-d862-4080-9d23-8c0e77960e65.log
2015-03-12 12:08:02	Starting to launch local task to process map join;	maximum memory = 518979584
2015-03-12 12:08:05	Dump the side-table for tag: 1 with group count: 4668 into file: file:/tmp/hduser/734bcc7b-78b9-46d9-a7cc-868c9ded365d/hive_2015-03-12_00-07-53_654_5923233654914816047-1/-local-10003/HashTable-Stage-3/MapJoin-mapfile41--.hashtable
2015-03-12 12:08:06	Uploaded 1 File to: file:/tmp/hduser/734bcc7b-78b9-46d9-a7cc-868c9ded365d/hive_2015-03-12_00-07-53_654_5923233654914816047-1/-local-10003/HashTable-Stage-3/MapJoin-mapfile41--.hashtable (396345 bytes)
2015-03-12 12:08:06	End of local task; Time Taken: 3.41 sec.
Execution completed successfully
MapredLocal task succeeded
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Job running in-process (local Hadoop)
Hadoop job information for Stage-3: number of mappers: 0; number of reducers: 0
2015-03-12 00:08:09,069 Stage-3 map = 100%,  reduce = 0%
Ended Job = job_local414824063_0005
MapReduce Jobs Launched: 
Stage-Stage-3:  HDFS Read: 12113427 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
2007	Jonathan	Broxton	310	390000
2008	Jonathan	Broxton	310	454000
2009	Jonathan	Broxton	310	1825000
2010	Jonathan	Broxton	310	4000000
2011	Jonathan	Broxton	310	7000000
2012	Jonathan	Broxton	310	4000000
2013	Jonathan	Broxton	310	4000000
2012	Jose	Ceda	280	480000
2002	Adam	Dunn	285	250000
2003	Adam	Dunn	285	400000
2004	Adam	Dunn	285	445000
2005	Adam	Dunn	285	4600000
2006	Adam	Dunn	285	7500000
2007	Adam	Dunn	285	10500000
2008	Adam	Dunn	285	13000000
2009	Adam	Dunn	285	8000000
2010	Adam	Dunn	285	12000000
2011	Adam	Dunn	285	12000000
2012	Adam	Dunn	285	14000000
2013	Adam	Dunn	285	15000000
2006	Prince	Fielder	275	329500
2007	Prince	Fielder	275	415000
2008	Prince	Fielder	275	670000
2009	Prince	Fielder	275	7000000
2010	Prince	Fielder	275	11000000
2011	Prince	Fielder	275	15500000
2012	Prince	Fielder	275	23000000
2013	Prince	Fielder	275	23000000
2006	Bobby	Jenks	275	340000
2007	Bobby	Jenks	275	400000
2008	Bobby	Jenks	275	550000
2009	Bobby	Jenks	275	5600000
2010	Bobby	Jenks	275	7500000
2011	Bobby	Jenks	275	6000000
2012	Bobby	Jenks	275	6000000
2003	Seth	McClung	280	300000
2004	Seth	McClung	280	302500
2005	Seth	McClung	280	320000
2006	Seth	McClung	280	343000
2008	Seth	McClung	280	750000
2009	Seth	McClung	280	1662500
2009	Jeff	Niemann	285	1290000
2010	Jeff	Niemann	285	1032000
2011	Jeff	Niemann	285	903000
2012	Jeff	Niemann	285	2750000
2007	Chad	Paronto	285	420000
2002	Calvin	Pickering	283	200000
2005	Calvin	Pickering	283	323500
2007	Renyel	Pinto	280	380000
2008	Renyel	Pinto	280	391500
2009	Renyel	Pinto	280	404000
2010	Renyel	Pinto	280	1075000
2002	Jon	Rauch	290	200000
2006	Jon	Rauch	290	335000
2007	Jon	Rauch	290	455000
2008	Jon	Rauch	290	1200000
2009	Jon	Rauch	290	2525000
2010	Jon	Rauch	290	2900000
2011	Jon	Rauch	290	3500000
2012	Jon	Rauch	290	3500000
2013	Jon	Rauch	290	1000000
2002	CC	Sabathia	290	700000
2003	CC	Sabathia	290	1100000
2004	CC	Sabathia	290	2700000
2005	CC	Sabathia	290	5250000
2006	CC	Sabathia	290	7000000
2007	CC	Sabathia	290	8750000
2008	CC	Sabathia	290	11000000
2009	CC	Sabathia	290	15285714
2010	CC	Sabathia	290	24285714
2011	CC	Sabathia	290	24285714
2012	CC	Sabathia	290	23000000
2013	CC	Sabathia	290	24285714
2002	Carlos	Silva	280	200000
2003	Carlos	Silva	280	310000
2004	Carlos	Silva	280	340000
2005	Carlos	Silva	280	1750000
2006	Carlos	Silva	280	3200000
2007	Carlos	Silva	280	4325000
2008	Carlos	Silva	280	8250000
2009	Carlos	Silva	280	12250000
2010	Carlos	Silva	280	12750000
1996	Dmitri	Young	295	109000
1997	Dmitri	Young	295	155000
1998	Dmitri	Young	295	215000
1999	Dmitri	Young	295	375000
2000	Dmitri	Young	295	1950000
2001	Dmitri	Young	295	3500000
2002	Dmitri	Young	295	5500000
2003	Dmitri	Young	295	6750000
2004	Dmitri	Young	295	7750000
2005	Dmitri	Young	295	8000000
2006	Dmitri	Young	295	8000000
2007	Dmitri	Young	295	500000
2008	Dmitri	Young	295	5000000
2009	Dmitri	Young	295	5000000
2003	Carlos	Zambrano	275	340000
2004	Carlos	Zambrano	275	450000
2005	Carlos	Zambrano	275	3760000
2006	Carlos	Zambrano	275	6500000
2007	Carlos	Zambrano	275	12400000
2008	Carlos	Zambrano	275	16000000
2009	Carlos	Zambrano	275	18750000
2010	Carlos	Zambrano	275	18875000
2011	Carlos	Zambrano	275	18875000
2012	Carlos	Zambrano	275	19000000
Time taken: 15.451 seconds, Fetched: 106 row(s)
```
Done. By joining tables, you can build some pretty complicated queries, which Hive will automatically execute with MapReduce.

[You can find the documentation for Hive commands here](https://cwiki.apache.org/confluence/display/Hive/LanguageManual).

[And here is another tutorial with more examples](https://cwiki.apache.org/confluence/display/Hive/Tutorial)
