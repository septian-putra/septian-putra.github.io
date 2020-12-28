---
layout: post
title: "Jupyter Notebook in AWS EMR: Part 1"
tags: [ analytics, aws, spark ]
categories: analytics
published: true
---

Sometimes we need to do an exploratory data analysis or write an ETL script on a huge dataset which cannot be handled by small machine.
With the Spark and Jupyter Notebook functionality in Amazon EMR, it is easy to set up a cluster, test your script and get the result in super fast and efficient way.
Data analysts and data scientists frequently use these types of clusters, known as analytics EMR clusters.

<!--more-->
There are 2 different ways to use Spark and Jupyter Notebook on EMR, first is using the EMR Notebook and the second is using JupyterHub from the master node.
Both approaches required a running EMR Cluster with Spark, Hadoop, Ganglia, JupyterHub, and Livy installed. 

### Setup EMR Cluster
1. Click create cluster button in EMR Console, and then go to advanced options.
2. In Step Software and Steps, choose the lates EMR release and checked  **Spark**, **Hadoop**, **Ganglia**, **JupyterHub**, and **Livy**.
3. Select Enter configuration. Fill with the following JSON
    <pre><code class="language-json">[
        {
            "classification": "yarn-site",
            "properties": {
                "yarn.nodemanager.pmem-check-enabled": "false",
                "yarn.nodemanager.vmem-check-enabled": "false"
            }
        },
        {
            "classification": "spark",
            "properties": {
                "maximizeResourceAllocation": "false"
            }
        },
        {
            "classification": "spark-defaults",
            "properties": {
                "spark.executor.instances": "8",
                "spark.executor.cores": "2",
                "spark.executor.memory": "17G",
                "spark.executor.memoryOverhead": "2G",
                "spark.driver.cores": "2",
                "spark.driver.memory": "17G",
                "spark.driver.memoryOverhead": "2G",
                "spark.executor.heartbeatInterval": "60s",
                "spark.rdd.compress": "true",
                "spark.network.timeout": "800s",
                "spark.memory.storageFraction": "0.30",
                "spark.sql.shuffle.partitions": "50",
                "spark.yarn.scheduler.reporterThread.maxFailures": "1",
                "spark.shuffle.spill.compress": "true",
                "spark.shuffle.compress": "true",
                "spark.storage.level": "MEMORY_AND_DISK_SER",
                "spark.default.parallelism": "50",
                "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
                "spark.memory.fraction": "0.80",
                "spark.executor.extraJavaOptions": "-XX:+UseG1GC -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=75 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:OnOutOfMemoryError='kill -9 %p'",
                "spark.dynamicAllocation.enabled": "false",
                "spark.driver.extraJavaOptions": "-XX:+UseG1GC -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=75 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:OnOutOfMemoryError='kill -9 %p'"
            }
        },
        {
            "classification": "livy-conf",
            "properties": {
                "livy.server.session.timeout": "4h"
            }
        },
        {
            "classification": "jupyter-s3-conf",
            "properties": {
                "s3.persistence.bucket": "aws-emr-resources-&lt;your_account&gt;-&lt;your_region&gt;",
                "s3.persistence.enabled": "true"
            }
        }
    ]
</code></pre>

4. In Step Hardware, select your public EC2 subnet and then select m5.large as Master with On Demand option, and `r4.2xlarge` or `r5.2xlarge` as Slave with Spot option. You can use [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/) to check which instance type has less frequent interruption.
Start with the 1 master and 3 slaves if you don't know how big the cluster you need.
5. In Step General Cluster Settings, provide the cluster name.
6. Select Custom action for Bootstrap Actions and point JAR location in S3. Below is some example of a bootstrap script.
    <pre><code class="language-bash">#!/usr/bin/env bash
    INSTALL_COMMAND="sudo pip install"
    dependencies="pandas numpy statsmodels pyarrow==0.12.1 boto3 botocore py4j"
    sudo apt-get install -y python-pip gcc
    for dep in $dependencies; do
        $INSTALL_COMMAND $dep
    done;</code></pre>*There is a compatibility issues regarding the pyarrow package at the time this post is created. Check yourself which pyarrow version is currently working.*

7. In Step Security, provide Additional security groups for Master and for Slave, which allow access to some ports for [EMR application interface](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-web-interfaces.html).
8. Click create cluster button.

#### Using EMR Notebook (Method 1)
1.  Go to Notebook tabs in the EMR console page.
2.  Create Notebook, fill Notebook Name, Description and running cluster to attach.
3.  Select Create Notebook. When it's ready, click Open.
4.  Your jupyter notebook is available.

#### Using JupyterHub on Master Node (Method 2)
1.  In the EMR console page, select your cluster. Click the JupyterHub hyperlink.
2.  If Warning: Potential Security Risk Ahead pop up, click advanced, accept the risk and continue.
3.  Login using `jovyan` as user name and `jupyter` as password
4.  Create your own folder with your name, launch a Jupyter Notebook inside the folder.
5.  Your jupyter notebook is available 

### Managing Your Pyspark Session
Apache Livy is a service that enables interaction with a Spark cluster over a REST interface, including submission of Spark code and Spark session management. By default, when we run a Jupyter cell, Livy will convert it into JSON and then send to the EMR Cluster. However, we can also run the script locally by specifying a magic `%%local` on top of a Jupyter cell.

In Method 1, we use serverless features in which the Jupyter Notebook is launched in an instance that internally handled by AWS.
While in Method 2, the Jupyter Notebook is launched in a container inside the master node with port forwarding access.
That's why some dependencies that we installed during bootstrapping cannot be found when we executed the command as local (`%%local`).

A Pyspark session is automatically created with Spark default configuration when you run your first PySpark cell. The duration this session will be active depends on `livy.server.session.timeout` that we previously set. We can also check which session is currently active, delete session or modify Spark configuration using sparkmagic commands. To check available sparkmagic and their descriptions, we can run `%help` magic.
{% include image-caption.html imageurl="/assets/images/posts/sparkmagic.png#center" title="Table of Possible SparkMagic" %}

<!-- ## Configure PySpark Session


From Jupyter cell, we can also modify spark configuration with the following local command in a Juptyter Notebook cell:

```
%%configure -f
{"executorMemory": "36G", 
"executorCores": 5,
"driverMemory": "36G", 
"driverCores": 5,
"numExecutors": 8}
```

If you decide to modify the number of slave instances, you should also modify the "numExecutors" in the script above to `(#slave_instances x 3) - 1`. -->

#### I hope your not feeling overwhelmed

It may seem like there is a lot to do to get started, but really it shouldn't take very long to set up and running. All the options are there just in case you want to further configure the cluster to be more optimal to your dataset, but you can just use the basic settings to get yourself up and running.