# Hadoop MapReduce with Python and Hive

A tutorial for writing a MapReduce program for Hadoop in python, and using Hive to do MapReduce with SQL-like queries.

This uses the Hadoop Streaming API with python to teach the basics of using the MapReduce framework.
The main idea and structure is based on [Michael G. Noll's great tutorial](http://www.michael-noll.com/tutorials/writing-an-hadoop-mapreduce-program-in-python/). However, that tutorial is outdated and quite few of the steps do not work anymore, both in setting up and running Hadoop. This is an updated and expanded tutorial, combined with a Hive tutorial.

You can write map and reduce functions in python, and use them with Hadoop's streaming API as shown here. This gives you a lot of flexibility.

In many cases, however, the information you are trying to get from the data distributed on the cluster can be expressed in terms of a SQL query. Hive is a program that takes a SQL query like this, automatically builds map and reduce jobs, runs them and returns the results. It makes using MapReduce really simple, all you need is some familiarity with basic SQL queries.

This tutorial shows both of these ways of using Hadoop.

## Let's Begin

[Setup your Hadoop cluster](setup.md)

[Write your first Python MapReduce program](mapreduce.md)

[Use Hive to make MapReduce queries](hive.md)





