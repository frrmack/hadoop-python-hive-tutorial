### Hadoop installation and setup on an Ubuntu server

If you have a cloud server (through a service such as AWS, Rackspace, Digital Ocean, etc.), ssh to it.
If you're doing this on your local server, just open up a terminal and follow the white rabbit below.

#### Install TextBlob

We're going to use it in our text processing, so make sure you have textblob in there.

    sudo pip install textblob

#### Install Java 7

    sudo apt-get install python-software-properties
    sudo add-apt-repository ppa:webupd8team/java
    sudo apt-get update
    sudo apt-get install oracle-jdk7-installer

The java install will ask a few straightforward questions, just answer
them.

####Check that java version is 1.7

    java -version


####Create a Hadoop user

    sudo addgroup hadoop
    sudo adduser --ingroup hadoop hduser

This will ask for a password, give it one. Each user in unix has a
password. You will use that when you switch to that user.

Make an ssh key so hadoop can connect to machines with ssh without entering a password every time.

    su hduser
    ssh-keygen  -t rsa -P ""

(hit enter when asked where to save the key)

Add the key to recognized keys in target computers (same as localhost
in this tutorial case)

    cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys

Add key to recognized keys list and test that it works without a
password

    ssh localhost

(type yes and enter when it asks about adding the key print to the
known keys list)


####Download and Install Hadoop

    wget http://psg.mtu.edu/pub/apache/hadoop/common/stable/hadoop-2.6.0.tar.gz

    su irmak

(switch to your own user, you’ll need some sudo here)

    sudo tar xvzf hadoop-2.6.0.tar.gz

    sudo mv hadoop-2.6.0 /usr/local/hadoop

    cd /usr/local

    sudo chown -R hduser:hadoop hadoop

#### Change bashrc settings

    su hduser

    emacs ~/.bashrc

You can use another editor, too, of course. (If you don't have emacs,
and do not want to use another editor you can install emacs with
*apt-get install emacs*). Add the following lines:

    # Environment variable for Hadoop location, include bin in the path
    export HADOOP_HOME=/usr/local/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin
    
    # Environment varibale for Java location
    export JAVA_HOME=/usr/lib/jvm/java-7-oracle
    
    # Hadoop related aliases
    unalias fs &> /dev/null
    alias fs="hadoop fs"
    unalias hls &> /dev/null
    alias hls="fs -ls"

Exit emacs (`Ctrl-x Ctrl-s` to save, `Ctrl-x Ctrl-c` to exit). Great, now these will run every time you connect to the
server, but let's also make sure they apply now. Type this in your
terminal.:

    source ~/.bashrc
    
#### Create the place to put HDFS on and tell Hadoop where it is

    su irmak

(We need some more sudo stuff so switch back to yourself for now)

    sudo mkdir -p /app/hadoop/tmp

    sudo chown -R hduser:hadoop /app/hadoop/tmp

    su hduser

(back to hduser to edit the configuration files)

    emacs /usr/local/hadoop/etc/hadoop/core-site.xml

Between `< configuration >`  and `< /configuration >` put this in:

```xml
<property>
<name>hadoop.tmp.dir</name>
<value>/app/hadoop/tmp</value>
      <description>A base for other temporary directories.</description>
</property>


<property>
<name>fs.default.name</name>
<value>hdfs://localhost:54310</value>
<description>The name of the default file system.  A URI whose
scheme and authority determine the FileSystem implementation.
The uri's scheme determines the config property (fs.SCHEME.impl)
naming the FileSystem implementation class. The uri's authority
is used to determine the host, port, etc. for a filesystem.</description>
</property>
```

Ok now another one.

    emacs /usr/local/hadoop/etc/hadoop/mapred-site.xml.template

Between `< configuration >`  and `< /configuration >`  put this in:
```xml
<property>
<name>mapred.job.tracker</name>
<value>localhost:54311</value>
      <description>The host and port that the MapReduce job tracker
	  runs
	        at.  If "local", then jobs are run in-process as a single
			map
			and reduce task.
			</description>
			</property>
```
And the last one

    emacs /usr/local/hadoop/etc/hadoop/hdfs-site.xml

Between `< configuration >`  and `< /configuration >` put this in:
```xml
<property>
<name>dfs.replication</name>
<value>1</value>
<description>Default block replication. The actual number of replications
can be specified when the file is created. The default is used if replication
is not specified in create time.</description>
</property>
```

Also tell hadoop where java 7 is

    emacs /usr/local/hadoop/etc/hadoop/hadoop-env.sh 
    
And at the very end, append this line:
    
    export JAVA_HOME=/usr/lib/jvm/java-7-oracle

Save, quit, and we're good.

#### Format the HDFS (hadoop filesystem)

    hdfs namenode -format

## STOP HERE.

Take a breath. Your setup is complete. The rest is actually using hadoop.

