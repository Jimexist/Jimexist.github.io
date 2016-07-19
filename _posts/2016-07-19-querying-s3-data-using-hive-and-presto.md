---
layout: post
title:  "Querying S3 data using Hive and Presto"
categories: tutorial
tags:
- Hive
- Presto
- AWS EMR
- S3
---

*Context Warning: this was written as an internal tutorial for new hires.*

## Goal

This tutorial demonstrates how you can:

- store CSV data as gzipped file to an S3 bucket
- use Hive external table to model and query the data in the S3 bucket
- use Presto to query the same data using Hive’s metadata

This tutorial assumes you have a basic understanding of [Hadoop](https://hadoop.apache.org), [Hive](http://hive.apache.org), [AWS](http://aws.amazon.com/) [EMR](https://aws.amazon.com/elasticmapreduce/), [S3](https://aws.amazon.com/s3/), and [Presto](http://prestodb.io).

## Preparation

- set up an EMR cluster
  - ask sysadmin if you don’t know how to do it, or you don’t have access to one
  - make sure the EMR cluster has both Hive and Presto installed on it. you can do it using AWS console or command line, e.g. argument like `aws emr create-cluster --applications Name=Hadoop Name=Hive Name=Presto-Sandbox ...`
- setup `passwordless ssh` login to the EMR master host
  - if you have a key-pair of choice set up during the cluster creation time then you are all set
  - otherwise you’ll need your own public key which most probably is `~/.ssh/id_rsa.pub`
  - you might also want to check for the rules for SSH (i.e. port `22`) in the cluster master’s security group
- set up AWS CLI client locally
  - you’ll need to do `brew install awscli` or `pip install awscli` if you don’t have the client
  - you’ll need to do `aws configure` to setup the credential
- make sure you have access to S3
  - try `aws s3 ls` to list all buckets, for testing purpose
  - if you can’t access S3, check IAM or ask sysadmin

## Uploading data to S3

- create an S3 bucket if you don’t have one, e.g.:

  `aws s3 mb s3://bucket-name`

- create a directory that will sync to the bucket, e.g.:

  `mkdir ~/Desktop/bucket-folder`

- create a folder under the directory that represents a table (a collection of data), e.g.:

  `mkdir ~/Desktop/bucket-folder/employees`

- adding a sample csv data to the folder, e.g.:

  ```sh
  echo 'hilary,female
  trump,male' | gzip > ~/Desktop/bucket-folder/employees/1.data.csv.gz
  ```

- sync the folder with S3, e.g.:

  `aws s3 sync ~/Desktop/bucket-folder s3://bucket-name`

- make sure you have the data successfully uploaded, e.g.:

  `aws s3 ls s3://bucket-name/employees`

## Defining and querying Hive table

- ssh to the Hadoop master machine:

  `ssh -i key.pem hadoop@your-hadoop-master-machine`

- enter Hive CLI:

  `hive`

- create an external table matching your data schema:

  ```sql
  create external table employees (
    name varchar(100),
    gender varchar(10)
  )
  row format delimited
  fields terminated by ','
  lines terminated by '\n'
  location 's3://bucket-name/employees/';
  ```

- now you can try to query the Hive table:

  `select * from employees;`
- your SQL should return 2 rows

## Appending more data

- keep the Hive shell open
- now you can open up another local shell, and append more data to the folder:

  ```sh
  echo 'superman,unknown
  batman,male' | gzip > ~/Desktop/bucket-folder/employees/2.data.csv.gz
  ```

- sync the new data to S3:

  `aws s3 sync ~/Desktop/bucket-folder s3://bucket-name`

- go back the hive shell and do another query:

  `select * from employees;`

- your SQL should return 4 rows

## Presto against the same table

Once you have the table defined in Hive, you can also use Presto to query it. Using presto is as simple as:

- enter Presto client by typing `presto-cli` on the cluster’s master machine
- switch to Hive catalog and default schema: `use hive.default;`
- `show tables;`
- `select * from employees;`

Play along, try some other things like `select count(*) from employees;` which incurs map-reduce job in Hive’s case but not in Presto’s. You’ll find the speed difference between two query engines.

## Conclusion

We’ve seen how you can:

- define an external table pointing to an S3 folder from Hive, and query data stored in there (i.e. on the fly, remembering Hive is a schema-on-read system)
- append data to a Hive table by simply adding more files (of the same structure) to the same S3 folder
- use Presto to query the same table (since Presto uses to Hive Metastore)

If you are curious, try some more things with this setting, e.g.:

- delete a file from that S3 folder, what will happen to the Hive table?
- what about replacing a file with different content?
- what about adding some more files with wrong schema/format?
- upload lines of JSON data and define a schema on it
  - you might want to search for *Json Serde*
  - Hive does support `map`, `array`, and `struct` which you might find handy
- upload a larger file, and benchmark the speed of Hive and Presto with more realistic dataset and queries
