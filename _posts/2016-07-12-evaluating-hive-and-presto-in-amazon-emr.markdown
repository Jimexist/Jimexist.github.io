---
layout: post
title:  "Evaluating Hive and Presto in Amazon EMR"
categories: hacking
tags:
- Hive
- Presto
- AWS EMR
---

These days I've been playing with [Hive](http://hive.apache.org/) and [Presto](https://prestodb.io/), to find out their speed and usability.

Setting up a Hadoop cluster on my two year old MacBook can be challenging, because more than two Virtual box processes can literally kill my 4G memory. Also, all the configs (e.g. HDFS, Yarn, MySQL Metastore, site configs, etc.) can be painful to setup. AWS EMR has out-of-box support for both Hive and Presto. So in this post I'm going to share my experience playing with them in AWS EMR.

## A word about Presto

I used to setup some [*Docker*-ing repos](https://github.com/Jimexist/docker-presto) just to try out Presto, knowing that it's a nice bridge between SQL (Postgres, MySQL, Hive, Cassandra) and NoSQL (MongoDB) data stores (and also amongst themselves). It stroke me as a pleasant surprise, that you can simply do:

```sql
create table mongo.local.users as
select * from postgresql.some_database.users;
-- mongo.local.users is <catalog>.<schema>.<table> in Presto's hierarchy
-- just as in the case of postgresql.some_database.users
```

and you'll have a MongoDB (NoSQL) collection with same data and schema from a Postgres (SQL) table.

In my understanding, Facebook [set out to develop Presto](https://prestodb.io/docs/current/overview/use-cases.html), as a bridge between their clusters of MySQL servers and in-house Hive data warehouse, for OLAP purposes. Hive, being a pleasant improvement of the good old *MapReduce* way to talk to Hadoop, is much more useful for non-programmers (after all, SQL is the *lingua franca*). But Presto is set out to be a [faster](https://code.facebook.com/posts/370832626374903/even-faster-data-at-the-speed-of-presto-orc/) and more standard-compliant replacement for Hive in the OLAP space.

## Amazon EMR with Hive and Presto

Yes, Amazon has Redshift, but it doesn't stop them from also [shipping Presto with EMR](https://aws.amazon.com/elasticmapreduce/details/presto/). To make my life much easier by skip setting up all of them by myself, I set out to try the most recent EMR (i.e. release 4.7.1) with Presto-Sandbox.

In [this excellent post](http://tech.marksblogg.com/billion-nyc-taxi-rides-presto-emr.html) the author explained in great details on how one can setup EMR with Presto to analyze NYC taxi data. So if you haven't read it you can go to that link first. My post from now will only focus on parts that are different.

Unlike that post, I used the AWS Web Console to create my cluster, which looks like this:

![AWS EMR Console](/assets/aws_emr_console.png)

Unfortunately, this is setup did not work out for me. Presto-Sandbox only includes *Hive Metastore* which is enough for Presto to work, but in order to also get Hive working, you'll have to go to the advanced options,

![AWS EMR Console Advanced](/assets/aws_emr_console_advanced.png)

and check the *Hive* application. (I've got `ClassNotFoundException` because of missing `hadoop-mapreduce` jar files).

In retrospect, this should have been the command to create my setup of cluster:

```sh
aws emr create-cluster \
  --applications Name=Hadoop Name=Hive Name=Presto-Sandbox \
  --tags 'Type=Dev' \
  --ec2-attributes '{
    "KeyName":"my-key",
    "InstanceProfile":"EMR_EC2_DefaultRole",
    "SubnetId":"subnet-<id>",
    "EmrManagedSlaveSecurityGroup":"sg-<slave-id>",
    "EmrManagedMasterSecurityGroup":"sg-<master-id>"
  }' \
  --service-role EMR_DefaultRole \
  --enable-debugging \
  --release-label emr-4.7.1 \
  --log-uri 's3n://aws-logs-<id>-cn-north-1/elasticmapreduce/' \
  --name 'Evaluate Presto' \
  --instance-groups '[{
      "InstanceCount":1,
      "InstanceGroupType":"MASTER",
      "InstanceType":"m3.xlarge",
      "Name":"Master instance group"
    },{
      "InstanceCount":2,
      "InstanceGroupType":"CORE",
      "InstanceType":"m3.xlarge",
      "Name":"Core instance group"
    }]' \
  --region cn-north-1
```

*N.B.* the slaves are of type `CORE`, but you can also optionally specify some slave hosts to be in `TASK` group. The latter does not have HDFS daemon running locally.

## Connecting to cluster and importing data

After the cluster is ready (i.e. in *Waiting* status), you can connect to the master using SSH. Make sure you modify the security group for Master (i.e. the `EmrManagedMasterSecurityGroup` as shown above) to allow port `22` (and optionally also `8889` for local presto-cli client) from your IP address.

After login, you'll see something like:

```
λ ~/Downloads/keys/ ssh -i my-key.pem hadoop@master
Last login: Wed Jul 13 05:15:50 2016

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2016.03-release-notes/
1 package(s) needed for security, out of 3 available
Run "sudo yum update" to apply all updates.

EEEEEEEEEEEEEEEEEEEE MMMMMMMM           MMMMMMMM RRRRRRRRRRRRRRR    
E::::::::::::::::::E M:::::::M         M:::::::M R::::::::::::::R   
EE:::::EEEEEEEEE:::E M::::::::M       M::::::::M R:::::RRRRRR:::::R
  E::::E       EEEEE M:::::::::M     M:::::::::M RR::::R      R::::R
  E::::E             M::::::M:::M   M:::M::::::M   R:::R      R::::R
  E:::::EEEEEEEEEE   M:::::M M:::M M:::M M:::::M   R:::RRRRRR:::::R
  E::::::::::::::E   M:::::M  M:::M:::M  M:::::M   R:::::::::::RR   
  E:::::EEEEEEEEEE   M:::::M   M:::::M   M:::::M   R:::RRRRRR::::R  
  E::::E             M:::::M    M:::M    M:::::M   R:::R      R::::R
  E::::E       EEEEE M:::::M     MMM     M:::::M   R:::R      R::::R
EE:::::EEEEEEEE::::E M:::::M             M:::::M   R:::R      R::::R
E::::::::::::::::::E M:::::M             M:::::M RR::::R      R::::R
EEEEEEEEEEEEEEEEEEEE MMMMMMM             MMMMMMM RRRRRRR      RRRRRR

[hadoop@ip-xxx-xx-0-xxx ~]$

```

### Dataset: Last.fm 1K

To evaluate Hive, you'll need a dataset. I used [Last.fm Dataset - 1K users](http://www.dtic.upf.edu/~ocelma/MusicRecommendationDataset/lastfm-1K.html) which includes a file `userid-timestamp-artid-artname-traid-traname.tsv` containing their streaming history of ~1000 users. Then I gzipped this file and putting it in my S3 bucket, say `s3://lastfm-dataset-1k/timestamp-tsv/data.tsv.gz`.

Hive has support for external data (i.e. it doesn't own) and can create table from it. AWS EMR extended this with S3 support, so importing the data is as simple as defining an external table.

```sql
create external table if not exists user_timestamps_tsv (
  user_id string,
  ts string,
  artist_uuid string,
  artist_name string,
  track_uuid string,
  track_name string
)
comment 'lastfm user activities'
row format delimited
fields terminated by '\t' -- it's tsv, use ',' for csv
lines terminated by '\n'
location 's3://lastfm-dataset-1k/timestamp-tsv/';
```

Note this table is created with `LOCATION` that points to an S3 bucket directory (not file), allowing you to append more data to the same folder easily.

Let's do a simple count:

```
hive> select count(*) from user_timestamps_tsv;
Query ID = hadoop_20160713052929_802fcfde-8525-4dd1-a6fa-f8f27f18a148
Total jobs = 1
Launching Job 1 out of 1
[ omitted for brevity ]
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1
2016-07-13 05:29:19,475 Stage-1 map = 0%,  reduce = 0%
[ omitted for brevity ]
2016-07-13 05:30:07,054 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 41.71 sec
MapReduce Total cumulative CPU time: 41 seconds 710 msec
Ended Job = job_1468377358342_0012
MapReduce Jobs Launched:
Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 41.71 sec   HDFS Read: 243 HDFS Write: 9 SUCCESS
Total MapReduce CPU Time Spent: 41 seconds 710 msec
OK
19150868
Time taken: 59.304 seconds, Fetched: 1 row(s)
```

That took nearly 1 minute for counting 20 million rows, which is somewhat slow.

### HDFS + ORC v.s. S3 + TSV

Defining an external table on S3 gives you flexibility and reliability (in case of cluster failure or shutdown), but it'll definitely help to store data with better locality, i.e. in HDFS.

We can create an ORC table with same schema like this:

```sql
create table if not exists user_timestamps_orc (
  user_id string,
  ts string,
  artist_uuid string,
  artist_name string,
  track_uuid string,
  track_name string
)
comment 'lastfm user activities'
stored as orc;
```

and then load the data from `user_timestamps_tsv` to `user_timestamps_orc`:

```
hive> insert into table user_timestamps_orc select * from user_timestamps_tsv;
Query ID = hadoop_20160713040101_ffee4f0f-a255-46ad-a308-9233d432b791
Total jobs = 1
Launching Job 1 out of 1
[ omitted for brevity ]
2016-07-13 04:01:44,641 Stage-1 map = 0%,  reduce = 0%
2016-07-13 04:01:55,999 Stage-1 map = 1%,  reduce = 0%, Cumulative CPU 8.6 sec
[ omitted for brevity ]
2016-07-13 04:05:16,057 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 213.23 sec
MapReduce Total cumulative CPU time: 3 minutes 33 seconds 230 msec
Ended Job = job_1468377358342_0007
Stage-4 is selected by condition resolver.
Stage-3 is filtered out by condition resolver.
Stage-5 is filtered out by condition resolver.
Moving data to: hdfs://<internal-ip>.cn-north-1.compute.internal:8020/tmp/hive/hadoop/4743bfde-7042-4ae3-add9-59328e017234/hive_2016-07-13_04-01-37_861_3135991179230807872-1/-ext-10000
Loading data to table default.user_timestamps_orc
Table default.user_timestamps_orc stats: [numFiles=1, numRows=19150868, totalSize=273765263, rawDataSize=12007594236]
MapReduce Jobs Launched:
Stage-Stage-1: Map: 1   Cumulative CPU: 213.23 sec   HDFS Read: 243 HDFS Write: 273765361 SUCCESS
Total MapReduce CPU Time Spent: 3 minutes 33 seconds 230 msec
OK
Time taken: 219.41 seconds
```

That took ~220 seconds. But now a count would be much faster:

```
hive> select count(*) from user_timestamps_orc;
Query ID = hadoop_20160713053030_08caaa4e-f66e-403d-b96d-9a10e86e66dd
Total jobs = 1
Launching Job 1 out of 1
[ omitted for brevity ]
Hadoop job information for Stage-1: number of mappers: 2; number of reducers: 1
2016-07-13 05:30:48,097 Stage-1 map = 0%,  reduce = 0%
[ omitted for brevity ]
2016-07-13 05:31:02,588 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 10.88 sec
MapReduce Total cumulative CPU time: 10 seconds 880 msec
Ended Job = job_1468377358342_0013
MapReduce Jobs Launched:
Stage-Stage-1: Map: 2  Reduce: 1   Cumulative CPU: 10.88 sec   HDFS Read: 52512 HDFS Write: 9 SUCCESS
Total MapReduce CPU Time Spent: 10 seconds 880 msec
OK
19150868
Time taken: 23.152 seconds, Fetched: 1 row(s)
```

A speed up from 59 seconds to 23 seconds, thanks for data locality and possibly ORC format with compression.

## Some more meaningful queries

Time for some more meaningful queries. Say we want to know which artists' songs got listened to most:

```
hive> select artist_uuid, artist_name, count(*) as counts
    > from user_timestamps_orc group by artist_uuid, artist_name
    > order by counts desc
    > limit 10;
Query ID = hadoop_20160713054545_7661c88f-d710-42ea-9a44-e839856baf87
Total jobs = 2
Launching Job 1 out of 2
[ omitted for brevity ]
Hadoop job information for Stage-1: number of mappers: 2; number of reducers: 2
2016-07-13 05:45:45,492 Stage-1 map = 0%,  reduce = 0%
[ omitted for brevity ]
2016-07-13 05:46:13,332 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 34.29 sec
MapReduce Total cumulative CPU time: 34 seconds 290 msec
Ended Job = job_1468377358342_0016
Launching Job 2 out of 2
[ omitted for brevity ]
2016-07-13 05:46:21,902 Stage-2 map = 0%,  reduce = 0%
2016-07-13 05:46:29,116 Stage-2 map = 100%,  reduce = 0%, Cumulative CPU 7.98 sec
2016-07-13 05:46:36,330 Stage-2 map = 100%,  reduce = 100%, Cumulative CPU 10.63 sec
MapReduce Total cumulative CPU time: 10 seconds 630 msec
Ended Job = job_1468377358342_0017
MapReduce Jobs Launched:
Stage-Stage-1: Map: 2  Reduce: 2   Cumulative CPU: 34.29 sec   HDFS Read: 45007988 HDFS Write: 10454998 SUCCESS
Stage-Stage-2: Map: 2  Reduce: 1   Cumulative CPU: 10.63 sec   HDFS Read: 10455868 HDFS Write: 550 SUCCESS
Total MapReduce CPU Time Spent: 44 seconds 920 msec
OK
a74b1b7f-71a5-4011-9441-d0b5e4122711	Radiohead	115209
b10bbbfc-cf9e-42e0-be17-e2c3e1d2600d	The Beatles	100338
b7ffd2af-418f-4be2-bdd1-22f8b48613da	Nine Inch Nails	84421
9c9f1380-2516-4fc9-a3e6-f9f61941d090	Muse	63346
cc197bad-dc9c-440d-a5b5-d52ba2e14234	Coldplay	62251
8538e728-ca0b-4321-b7e5-cff6565dd4c0	Depeche Mode	59910
83d91898-7763-47d7-b03b-b92132375c47	Pink Floyd	58561
0039c7ae-e1a7-4a7d-9b49-0cbc716821a6	Death Cab For Cutie	58083
847e8284-8582-4b0e-9c26-b042a4f49e57	Placebo	53518
03ad1736-b7c9-412a-b442-82536d63a5c4	Elliott Smith	50278
Time taken: 58.461 seconds, Fetched: 10 row(s)
```

What about most popular tracks:

```
hive> select track_uuid, track_name, count(*) as counts
    > from user_timestamps_orc
    > group by track_uuid, track_name
    > order by counts desc
    > limit 10;
Query ID = hadoop_20160713054848_6cf239c5-d0f9-4ee0-b8a9-8a0e72081616
Total jobs = 2
Launching Job 1 out of 2
[ omitted for brevity ]
Hadoop job information for Stage-1: number of mappers: 2; number of reducers: 2
2016-07-13 05:48:57,880 Stage-1 map = 0%,  reduce = 0%
[ omitted for brevity ]
2016-07-13 05:49:45,404 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 66.59 sec
MapReduce Total cumulative CPU time: 1 minutes 6 seconds 590 msec
Ended Job = job_1468377358342_0018
Launching Job 2 out of 2
[ omitted for brevity ]
Hadoop job information for Stage-2: number of mappers: 2; number of reducers: 1
2016-07-13 05:49:52,932 Stage-2 map = 0%,  reduce = 0%
2016-07-13 05:50:03,214 Stage-2 map = 100%,  reduce = 0%, Cumulative CPU 14.24 sec
2016-07-13 05:50:11,449 Stage-2 map = 100%,  reduce = 100%, Cumulative CPU 18.21 sec
MapReduce Total cumulative CPU time: 18 seconds 210 msec
Ended Job = job_1468377358342_0019
MapReduce Jobs Launched:
Stage-Stage-1: Map: 2  Reduce: 2   Cumulative CPU: 66.59 sec   HDFS Read: 135304834 HDFS Write: 91183905 SUCCESS
Stage-Stage-2: Map: 2  Reduce: 1   Cumulative CPU: 18.21 sec   HDFS Read: 91184779 HDFS Write: 585 SUCCESS
Total MapReduce CPU Time Spent: 1 minutes 24 seconds 800 msec
OK
db16d0b3-b8ce-4aa8-a11a-e4d53cc7f8a6	Such Great Heights	3992
7f1f45c0-0101-49e9-8d69-23951d271163	Love Will Tear Us Apart	3663
9e2ad5bc-c6f9-40d2-a36f-3122ee2072a3	Karma Police	3534
ff1e3e1a-f6e8-4692-b426-355880383bb6	Supermassive Black Hole	3483
4e17b118-70a6-4c1f-b326-b4ce91fd3fad	Soul Meets Body	3479
db4c9220-df76-4b42-b6f5-8bf52cc80f77	Heartbeats	3156
60e94685-0481-4d3d-bd84-11c389d9b2a5	Starlight	3060
f874c752-65bc-4d50-ac7e-932243ae9f02	Rebellion (Lies)	3048
bd782340-6fa5-4b52-aa5a-ceafb9bc0340	Gimme More	3004
4ad08552-6c35-49ed-bcc6-6822c8f9dfd8	When You Were Young	2998
Time taken: 81.195 seconds, Fetched: 10 row(s)
```

To wrap it up:

- I didn't realize Radiohead was this popular 😎, at least amongst the 993 users' streaming history in this dataset
- a simple count on 20 million rows on external text file based Hive table took `59` seconds, and on a managed ORC based Hive table it took `23` seconds
- a more meaningful query for top 10 most played artists took `58` seconds on the ORC based table

## Entering Presto

Presto can read the underlying ORC format file for a Hive table, but it does not use Hive's execution engine. Instead, it is more memory-bound and thus expected to be much faster.

```
presto> use hive.default;
presto:default> select count(*) from user_timestamps_orc;
  _col0   
----------
 19150868
(1 row)

Query 20160713_070503_00235_yfcme, FINISHED, 2 nodes
Splits: 10 total, 9 done (90.00%)
0:00 [19.2M rows, 5.22MB] [40M rows/s, 10.9MB/s]
```
As you can see, the above query returns almost instantly.

The top ten artists query:

```
presto:default> select artist_uuid, artist_name, count(*) as counts
             -> from user_timestamps_orc group by artist_uuid, artist_name
             -> order by counts desc
             -> limit 10;
             artist_uuid              |     artist_name     | counts
--------------------------------------+---------------------+--------
 a74b1b7f-71a5-4011-9441-d0b5e4122711 | Radiohead           | 115209
 b10bbbfc-cf9e-42e0-be17-e2c3e1d2600d | The Beatles         | 100338
 b7ffd2af-418f-4be2-bdd1-22f8b48613da | Nine Inch Nails     |  84421
 9c9f1380-2516-4fc9-a3e6-f9f61941d090 | Muse                |  63346
 cc197bad-dc9c-440d-a5b5-d52ba2e14234 | Coldplay            |  62251
 8538e728-ca0b-4321-b7e5-cff6565dd4c0 | Depeche Mode        |  59910
 83d91898-7763-47d7-b03b-b92132375c47 | Pink Floyd          |  58561
 0039c7ae-e1a7-4a7d-9b49-0cbc716821a6 | Death Cab For Cutie |  58083
 847e8284-8582-4b0e-9c26-b042a4f49e57 | Placebo             |  53518
 03ad1736-b7c9-412a-b442-82536d63a5c4 | Elliott Smith       |  50278
(10 rows)

Query 20160713_070621_00240_yfcme, FINISHED, 2 nodes
Splits: 12 total, 6 done (50.00%)
0:05 [19.2M rows, 47.4MB] [3.52M rows/s, 8.73MB/s]
```

and the top ten tracks query:

```
presto:default> select track_uuid, track_name, count(*) as counts
             -> from user_timestamps_orc
             -> group by track_uuid, track_name
             -> order by counts desc
             -> limit 10;
              track_uuid              |       track_name        | counts
--------------------------------------+-------------------------+--------
 db16d0b3-b8ce-4aa8-a11a-e4d53cc7f8a6 | Such Great Heights      |   3992
 7f1f45c0-0101-49e9-8d69-23951d271163 | Love Will Tear Us Apart |   3663
 9e2ad5bc-c6f9-40d2-a36f-3122ee2072a3 | Karma Police            |   3534
 ff1e3e1a-f6e8-4692-b426-355880383bb6 | Supermassive Black Hole |   3483
 4e17b118-70a6-4c1f-b326-b4ce91fd3fad | Soul Meets Body         |   3479
 db4c9220-df76-4b42-b6f5-8bf52cc80f77 | Heartbeats              |   3156
 60e94685-0481-4d3d-bd84-11c389d9b2a5 | Starlight               |   3060
 f874c752-65bc-4d50-ac7e-932243ae9f02 | Rebellion (Lies)        |   3048
 bd782340-6fa5-4b52-aa5a-ceafb9bc0340 | Gimme More              |   3004
 4ad08552-6c35-49ed-bcc6-6822c8f9dfd8 | When You Were Young     |   2998
(10 rows)

Query 20160713_070732_00243_yfcme, FINISHED, 2 nodes
Splits: 12 total, 9 done (75.00%)
0:11 [19.2M rows, 132MB] [1.77M rows/s, 12.2MB/s]
```

### Comparison

So the comparison here is:

- `count(*)`
  - Hive: 23 seconds
  - Presto: less than 1 second
- top ten artists
  - Hive: 58 seconds
  - Presto: 5 seconds
- top ten tracks
  - Hive: 81 seconds
  - Presto: 11 seconds

The time-run data collected here is not comprehensive nor systematic, so please don't take them for serious benchmarks. But they do give a hint about how much faster Presto can be compared to Hive.

## Conclusion

The description on Presto's front page is:

> Presto is an open source distributed SQL query engine for running interactive analytic queries against data sources of all sizes ranging from gigabytes to petabytes.

Given the size of the data and response time tested in this post, Presto is in a much closer place than Hive
to the magic word *interactive*. Currently it is being actively developed by Facebook as well as the community, also there's much less vendor lock-in as it speaks ANSI-SQL. So if you haven't yet, try it out next time when you want to gain quick insight from large and small datasets.
