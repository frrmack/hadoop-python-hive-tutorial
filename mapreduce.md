## MapReduce with Python

#### Start the cluster!

    /usr/local/hadoop/sbin/start-dfs.sh

Yes!!! It’s running. You can check the report on the cluster at this
address on your web browser:

    http://<YOUR_CLOUD_SERVER_IP>:50070

(Replace <YOUR_CLOUD_SERVER_ID> with the actual ip, like 167.214.312.54)    
On the terminal,

    jps

will show you that `DataNode`, `NameNode` and `SecondaryNameNode` are running.

#### Let’s stop it

    /usr/local/hadoop/sbin/stop-dfs.sh

Now go check

    http://<YOUR_CLOUD_SERVER_IP>:50070

It shouldn’t be there anymore!

#### Get data

Alright, let’s put some data in.

Let’s make a directory for these

    mkdir -p /home/hduser/textdata

First we’ll start with putting the data into our normal data system.
If you have some text files, you can use them for this.
If not, here are three ebooks (plain text `utf-8` encoding) you can
`wget`:

Ulyses by James Joyce    
[http://www.gutenberg.org/cache/epub/4300/pg4300.txt][1]   

Notebooks of Leonardo Da Vinci    
[http://www.gutenberg.org/cache/epub/5000/pg5000.txt][2]   

The Outline of Science by J Arthur Thomson    
[http://www.gutenberg.org/cache/epub/20417/pg20417.txt][3]   

(For example, to get these, you can do this:

    cd /home/hduser/textdata
    wget http://www.gutenberg.org/cache/epub/4300/pg4300.txt
    wget http://www.gutenberg.org/cache/epub/5000/pg5000.txt
    wget http://www.gutenberg.org/cache/epub/20417/pg20417.txt
    
)

#### Put data in hdfs

First, let’s start the cluster again!

    /usr/local/hadoop/sbin/start-dfs.sh

make some directories **in the hadoop distributed file system!**

    hdfs dfs -mkdir /user/
    hdfs dfs -mkdir /user/irmak/

Of course replace `irmak` with your own username.
Let’s check that they exist

    hdfs dfs -ls /
    hdfs dfs -ls /user/

Yay!

Ok, put some data in

    hdfs dfs -put /home/hduser/textdata/* /user/irmak

Check and make sure it is in the hdfs

    hdfs dfs -ls /user/irmak

Yay!

####Our mapper and reducer

Our mapper `count_mapper.py` includes the following code:
```python
#!/usr/bin/env python

import sys
from textblob import TextBlob

for line in sys.stdin:
    line = line.decode('utf-8')
    words = TextBlob(line).words
    for word in words:
        word = word.encode('utf-8')
        print "%s\t%i" % (word, 1)
```

And our reducer `count_reducer.py` looks like this:
```python
#!/usr/bin/env python

import sys

current_word = None
current_count = 0
word = None

for line in sys.stdin:
    word, count = line.split('\t')
    count = int(count)
    if word == current_word:
        current_count += count
	else:
        if current_word:
	        print '%s\t%i' % (current_word, current_count)
		current_word = word
		current_count = count

if current_word == word:
    print '%s\t%i' % (current_word, current_count)
```

Before running these codes, we need to make sure that textblob has its nltk corpora downloaded, so that it can work without an error. To do that, execute this on the command line (as the hduser):

    python -m textblob.download_corpora

####Let's run it!

Before giving the following command, don't forget to replace the `/user/irmak` path (in the hdfs) with your own version, and the paths to `count_mapper.py` and `count_reducer.py` (in your droplet's local filesystem) with your own versions.

    hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-2.6.0.jar -file /home/hduser/count_mapper.py -mapper /home/hduser/count_mapper.py -file /home/hduser/count_reducer.py -reducer /home/hduser/count_reducer.py -input /user/irmak/* -output /user/irmak/book-output

Booom ! It's running.

#### Looking at the output
Once it's done,

    hdfs dfs -ls /user/irmak/book-output

should show that there is a `_SUCCESS` file (showing we did it!) and
another file called `part-00000`

This `part-00000` is our output. To look in:

    hdfs dfs -cat /user/irmak/book-output/part-00000

or just

    hdfs dfs -cat /user/irmak/book-output/*

will show the output of our job!

If you want to see the most common words, run:

    hdfs dfs -cat /user/irmak/book-output/* | sort -rnk2 | less

########Note:
If something went wrong when you ran your mapreduce job, you fix something and want to run it again, it will throw a different error, saying that the book-output directory already exists in hdfs. This error is thrown to avoid overwriting previous results. If you want to just rerun it anyway, you need to delete the output first, so it can be created again:

    hdfs dfs -rm -r /user/irmak/book-output
    
    

[1]: http://www.gutenberg.org/cache/epub/4300/pg4300.txt
[2]: http://www.gutenberg.org/cache/epub/5000/pg5000.txt
[3]: http://www.gutenberg.org/cache/epub/20417/pg20417.txt
