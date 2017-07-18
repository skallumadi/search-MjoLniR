Terms
=====

A few terms used in this document:

* yarn - This is the hadoop cluster manager. Anything that wants to request
  resources from the hadoop cluster must request it from yarn. Yarn has a webui
  at https://yarn.wikimedia.org which requires LDAP credentials to view.

* stage - At a high level spark jobs are described as a DAG of stages that need
  to be executed on data. Each stage represents one more more operations to
  perform on some data.

* task - Spark stages are broken up into many tasks. Each task represents the work
  to do for a single partition of a single stage of computation.

* driver - This is the JVM that is in control of the spark job and orchestrates
  everything. This instance does not generally get assigned any tasks, instead only
  being responsible for orchestrating the delivery of tasks to executors. For
  the instructions in this document this is always on stat1004, but it is
  possible for the driver to be created inside the hadoop cluster instead.

* executor - Also known as workers. These instances do the heavy lifting of executing
  spark tasks. When asked for resources yarn will spin up containers on nodes
  inside the hadoop cluster and startup JVM's running spark. Those JVM's will
  contact the driver and be assigned tasks to perform.

* RDD - This is the basic building block of spark parallelism. It stands for
  resilient distributed dataset. Underneath every parallel operation in spark
  there is eventually an rdd.

Caveats
=======

As MjoLniR is still in early days there is not a defined process to deploy to the analytics cluster.
This will be worked out, but for now it is all manual. Some caveats:
>
* MjoLniR must be run from a debian jessie based host. The is because the hadoop workers
  are also running debian jessie and the python binary on ubuntu will not run properly on them.

* The only debian jessie based host available as a spark driver is stat1004.eqiad.wmnet.

* MjoLniR requires spark 2.1.0, but the default installed version in the hadoop
  cluster is 1.6.0.  You will need to fetch spark from apache and uncompress
  the archive in your home directory on stat1004.

* MjoLniR requires a python virtualenv containing all the appropriate library
  dependencies. Unfortunately it is not possible yet to build the virtualenv on
  stat1004 due to some missing compile time dependencies. See https://gerrit.wikimedia.org/r/348669

* MjoLniR requires a jar built in the /jvm directory of this repository. This is a fat jar
  containing other dependencies, such as xgboost and kafka streaming.

Spark gotchas
=============

* Spark will only write out new datasets to non-existent directories. It is possible if some command
  (i.e. data_pipeline.py) that was supposed to write out to directory fails mid run that the directory
  will be created but unpopulated. It needs to be removed with `hdfs dfs -rm -r -f hdfs://analytics-hadoop/...`
  before re-running the command. You can check if the directory exists in `/mnt/hdfs`.

Setting everything up
=====================

Taking the above into account, the process to get started is basically:

Build a virtualenv inside the vagrant box configured with this repository::

	cd /vagrant
	virtualenv mjolnir_venv
	mjolnir_venv/bin/pip install .
	cd mjolnir_venv
	zip -qr ../mjolnir_venv.zip .
	cd ..
	rm -rf mjolnir_venv

That zip file needs to be copied over to stat1004, along with the appropriate jar::

	scp mjolnir_venv.zip stat1004.eqiad.wmnet:~/
	scp jvm/target/mjolnir-0.1-jar-with-dependencies.jar stat1004.eqiad.wmnet:~/

SSH into stat1004::

	ssh stat1004.eqiad.wmnet

Make a directory just for mjolnir stuff::

	mkdir ~/mjolnir

Move the files we copied to the machine into it::

	mv mjolnir_venv.zip mjolnir/
	mv mjolnir-0.1-jar-with-dependencies.jar mjolnir/

Pull down spark 2.1 and decompress that there too::

	cd mjolnir/
	https_proxy=http://webproxy.eqiad.wmnet:8080 wget https://archive.apache.org/dist/spark/spark-2.1.0/spark-2.1.0-bin-hadoop2.6.tgz
	tar -zxf spark-2.1.0-bin-hadoop2.6.tgz

We need to uncompress mjolnir_venv.zip so that it is accessible both in the
driver and the hadoop workers::

	mkdir venv
	cd venv
	unzip -q ../mjolnir_venv.zip

Running an interactive shell
============================

Finally we should be ready to run things. Lets start first with the pyspark
REPL to see things are working::

	ssh stat1004.eqiad.wmnet
	cd mjolnir/
	PYSPARK_PYTHON=venv/bin/python SPARK_CONF_DIR=/etc/spark/conf spark-2.1.0-bin-hadoop2.6/bin/pyspark \
	  --jars ${HOME}/mjolnir/mjolnir-0.1-jar-with-dependencies.jar \
	  --driver-class-path ${HOME}/mjolnir/mjolnir-0.1-ja[0/1818]ependencies.jar \
	  --master yarn \
	  --files /usr/lib/libhdfs.so.0.0.0 \
	  --archives 'mjolnir_venv.zip#venv'

An expanation of the options used:
* PYSPARK_PYTHON - Tells spark where to find the python executable. This path
  must be a relative path to work both locally and on the worker nodes where
  mjolnir_venv.zip is decompressed.

* SPARK_CONF_DIR - Tells spark where to find it's configuration. This is
  required because we are using spark 2.1.0, but spark 1.6.0 is installed on
  the machines

* spark-2.1.0-bin-hadoop2.6/bin/pyspark - The executable that stands up the
  jvm, talks to yarn, etc. The pyspark executable specifically stands up an
  interactive python REPL.

* --jars ... - Tells spark about extra jars we want it to provide to the executor nodes

* --driver-class-path ... - Tells spark about extra jars we want in the driver application

* --master yarn - Tells spark we will be distributing the work across a cluster.
  Without this option all spark workers will be local within the same JVM

* --files ... - Additional files spark should ship to the executors. For some
  reason libhdfs isn't always found so this ensures it is available.

* --archives ... - Files that spark should decompress into the working
  directory. The part before # is the path to the file locally, and the part
  after the # is the directory to decompress to.

After a bunch of output, some warnings, perhaps a few exceptions printed out
(normal, they are usually related to trying to find a port to run the web ui
on), you will be greated with a prompt. It should look something like::

	Welcome to
	      ____              __
	     / __/__  ___ _____/ /__
	    _\ \/ _ \/ _ `/ __/  '_/
	   /__ / .__/\_,_/_/ /_/\_\   version 2.1.0
	      /_/
	
	Using Python version 2.7.9 (default, Jun 29 2016 13:08:31)
	SparkSession available as 'spark'.
	>>>

From here you can do anything you could do when programming mjolnir. This can be quite
useful for one-off tasks such as evaluating a previously trained model against a new
dataset, or splitting up an existing dataset into smaller pieces.

Running data_pipeline.py
========================

The commandline for kicking off the data pipeline looks like::

	cd ~/mjolnir
	PYSPARK_PYTHON=venv/bin/python SPARK_CONF_DIR=/etc/spark/conf spark-2.1.0-bin-hadoop2.6/bin/spark-submit \
		--jars "${HOME}/mjolnir/mjolnir-0.1-jar-with-dependencies.jar" \
		--driver-class-path "${HOME}/mjolnir/mjolnir-0.1-jar-with-dependencies.jar" \
		--master yarn \
		--files /usr/lib/libhdfs.so.0.0.0 \
		--archives 'mjolnir_venv.zip#venv' \
		venv/lib/python2.7/site-packages/mjolnir/cli/data_pipeline.py \
		-i 'hdfs://analytics-hadoop/wmf/data/discovery/query_clicks/daily/year=*/month=*/day=*' \
		-o hdfs://analytics-hadoop/user/${USER}/mjolnir/training_data \
		-c codfw \
		enwiki

This uses all the same basic spark options as before, but changes the binary
run from `pyspark`, the interactive REPL, to `spark-submit` which runs a
predefined script. This script takes a few options, but for simplicity here we
pass only a few of the available parameters:

* -i The input directory containing the query click data. It is unlikely you
  will ever need to use a different value than shown here.

* -o The output directory. This is where the training data will be stored. This
  must be on HDFS. This may vary as you generate different sizes of training data

* -c The search cluster to use. It is very important that this is pointed at
  the *hot*spare* search cluster.  Pointing this at the currently active cluster
  could cause increased latency for our users.

Running training_pipeline.py
============================

The commandline for kicking off training looks like::

	PYSPARK_PYTHON=venv/bin/python SPARK_CONF_DIR=/etc/spark/conf ~/spark-2.1.0-bin-hadoop2.6/bin/spark-submit \
		--jars /home/ebernhardson/mjolnir-0.1-jar-with-dependencies.jar \
		--driver-class-path /home/ebernhardson/mjolnir-0.1-jar-with-dependencies.jar \
		--master yarn --files /usr/lib/libhdfs.so.0.0.0 \
		--archives 'mjolnir_venv.zip#venv' \
		--conf spark.dynamicAllocation.maxExecutors=105 \
		--conf spark.sql.autoBroadcastJoinThreshold=-1 \
		--conf spark.task.cpus=4 \
		--conf spark.yarn.executor.memoryOverhead=1536 \
		--executor-memory 2G \
		--executor-cores 4 \
		venv/lib/python2.7/site-packages/mjolnir/cli/training_pipeline.py \
		-i hdfs://analytics-hadoop/user/ebernhardson/mjolnir/1193k_with_one_hot_wp10 \
		-o ~/training_size/1193k_with_one_hot_wp10 \
		-w 1 -c 100 -f 5 enwiki

This includes a few more arguments than the interactive shell did. These are:

* --conf spark.dynamicAllocation.maxExecutors=105 - The training process can
  use an incredible amount of resources on the cluster if allowed to. Generally
  we want to prevent mjolnir from taking up more than half the cluster for short
  runs, and probably less than 1/3 of the cluster for jobs that will run for many
  hours. Further below is some discussion on spark resource usage.

* --conf spark.sql.autoBroadcastJoinThreshold=-1 - Spark can do a join using an
  expensive distributed algorithm, or it can broadcast a small table to all
  executors and let them do a cheaper join directly against that broadcasted
  table. This configuration isn't strictly required, but if spark executors start
  getting killed for running over their memory limits on small to mid sized
  datasets this can help.

* --conf spark.task.cpus=4 - This sets the number of cpus in an executor to assign
  to an individual task. The default value of 1 means that if we spin up executors
  with 4 cores, 4 tasks will be assigned. When training with xgboost we want a single
  task to have access to all the cores, so we set this to the same value as the
  number of cores assigned to each executor.

* --conf spark.yarn.executor.memoryOverhead=1536 - This sets the amount of memory
  that will be requested from yarn (the cluster manager) but not provided to the
  JVM heap. When training with XGBoost all the training data is held off-heap in
  C++ so this needs to be large enough for general overhead and the off-heap
  training data.

* --executor-memory 2G - This is, approximately, the size of the java heap. Roughly
  60% of this will be reserved for spark block storage (local copies of dataframes
  held in memory, such as the cross-validation folds). The other 40% is available
  for execution overhead. A reasonably large amount of memory is needed for loading
  the training data and shipping it over to xgboost via JNI. See spark docs at
  https://spark.apache.org/docs/2.1.0/tuning.html#memory-management-overview

* --executor-cores 4 - This is the number of cores that will be requested from yarn
  for each executor. With the current cluster configuration 4 is the maximum that
  can be requested. Must be the same as spark.task.cpus above when training

* venv/lib/python2.7/site-packages/mjolnir/cli/training_pipeline.py - This is the
  script to run on the driver to actually run the spark job. Reaching into venv
  like this is perhaps undesirable but gets the job done

* -i ... - Tells the training pipeline where to find the training data. This must be
  on HDFS and should be the output of the `data_pipeline.py` script.

* -o ... - Tells the training pipeline where to write out various information about
  the results of training. This must be a local path.

* -w 1 - Tells the training pipeline how many executors should be used to train a single
 model. When doing feature engineering with small-ish (~1M sample) training sets the most
 efficient use of resources is to train many models in parallel with a single worker per
 model.

* -c 100 - This is the number of models to train in parallel. The total number of executors
 required is this times the number of workers per model. In this example that is 100 * 1.

* -f 5 - The number of folds to use for cross-validation. This can be a bit of a complicated
  decision, but generally 5 is an acceptable, it not amazing, tradeoff of training time
  vs. training accuracy. Basically for every set of training parameters attempted this many
  models will be trained and the results averaged between them. If training is showing high
  variance increasing this to 11 will make the training take longer but might have more accurate
  statistics.

* enwiki - Finally we take a list of wikis to train models for. Each wiki is trained on its own,
  and a training dataset can contain features for multiple wikis.

Resource usage in the hadoop cluster
====================================


Help! There are exceptions eveywhere!
=====================================

Unfortunately spark is pretty spammy around worker shutdown. Spark executors
will, by default, shut down after being idle for 60 seconds unless they contain
cached RDD's. Often enough the nodes shutdown before the driver has completely
cleared it's internal state about the node and you get exceptions about a socket
timing out, or a broadcast variable not being able to be cleaned up.  These
exceptions are basically OK and don't indicate anything wrong. There are on the
other hand exceptions that do need to be paid attention to. Task failures are
always important. Executors killed by yarn for overrunning their memory limits
are also worth paying attention to, although if the rate is very low it is
sometimes acceptable.

An example of when to expect node shutdowns is during model training when a
mjolnir.training.hyperopt.minimize run is completing. We may spin up 100 or so
containers to run the minimization, but at the end we are waiting for a few
stragglers to finish up. The first executors to finish may side idle for more than
60 seconds waiting for the last executors to finish and shut themselves down.

