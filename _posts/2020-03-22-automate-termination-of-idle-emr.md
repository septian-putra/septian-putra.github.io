---
layout: post
title: "Automate Termination of Idle EMR"
tags: [ engineering, analysis, aws ]
categories: engineering
published: true
---

It’s important to regularly monitor the cost of the services and analyze cost patterns for big data workloads. Otherwise, the costs can explode as our usage grows due to unnecessary costs being unnoticed. One of example is when users forget terminating EMR and keep the idle clusters running. We propose a solution which can help identify unused-idle EMR and shut them down, which do not require big efforts but can lead to substantial savings.  


### Introduction

Amazon CloudWatch has `IsIdle` metric, an Amazon EMR native metric which determines the idleness of the cluster by checking whether there’s a YARN job running. However, it does not consider other conditions, such as SSH users connected or Presto jobs running, to determine whether the cluster is idle.
Also, when you execute any Spark jobs in Jupyter Notebook, it remains active (1) for long hours, even after the job is finished. In such cases, the metric is not ideal in deciding the inactivity of a cluster and we need to combine those conditions to create our own custom metric.

We implemented a bash script to be installed in the master node of the EMR cluster, and the script is scheduled to run every 5 minutes. The script monitors the clusters and sends a custom metric,`isEMRUsed` (0=inactive; 1=active), to CloudWatch every 5 minutes. If CloudWatch receives 0 (inactive) for some predefined set of data points, it triggers an alarm, which in turn executes an AWS Lambda function that terminates the cluster.

### Technical Implementation
<p align="center"><img src="/assets/images/posts/emr-idle-shutdown.png"></p>

The architecture of the solution we implemented is shown by the diagram above. First, we set up the EMR cluster to create aggregated metric called `isEMRUsed` and an alarm in the Amazon CloudWatch. Regularly (every 5 minutes), the driver will send some metrics about EMR usage to the CloudWatch. The alarm is raised when it gets 9 consecutive 0 signals of isEMRUsed and then a message is sent to the specific topic on Amazon SQS. Subsequently, an AWS Lambda that subscribes to the topics will be triggered to shut down the EMR cluster.

To achieve the scenario above, 3 scripts below are implemented. The implementation of those scripts and the lambda handler can also be seen in the following [repository](https://github.com/septian-putra/emr-monitoring).

#### JupyterHub API token script
The script, [`jupyterhub_addAdminToken.sh`](https://github.com/septian-putra/emr-monitoring/blob/master/jupyterhub_addAdminToken.sh), is shell script used to add a token for monitoring via REST APIs provided by JupyterHub. In our design, we use it to check whether the JupyterHub application is in use.
<pre><code class="language-bash">#!/bin/sh
set -x
if [ -f /etc/jupyter/conf/jupyterhub_config.py ]; then
   if grep -q 'c.JupyterHub.api_tokens' /etc/jupyter/conf/jupyterhub_config.py ; then
      echo "Api Token already exists for the Admin user"
   else
      if grep -q 'c.Authenticator.admin_users' /etc/jupyter/conf/jupyterhub_config.py ; then
        admin_token=`openssl rand -hex 32`
        admin_token="'${admin_token}'"
        echo $admin_token
        admin_users=`cat /etc/jupyter/conf/jupyterhub_config.py | grep 'c.Authenticator.admin_users' | awk '{print $3}'`
        users=`echo $admin_users | awk '{print substr($0, 2, length($0) - 2)}'`
        lstusers=`echo $users | tr "," "\n"`
        for usr in $lstusers
        do
          #admin_user=`echo  $usr | awk '{print substr($0, 2, length($0) - 2)}'`
          admin_user=$(echo $usr)
          break
        done
        echo $admin_user
        sudo bash -c 'echo "c.JupyterHub.api_tokens = {" >> /etc/jupyter/conf/jupyterhub_config.py'
        sudo bash -c 'echo "    '$admin_token' : '$admin_user'," >> /etc/jupyter/conf/jupyterhub_config.py'
        sudo bash -c 'echo "}" >> /etc/jupyter/conf/jupyterhub_config.py'
        sudo docker stop jupyterhub
        sudo docker start jupyterhub
      else
        echo "No admin users - API Token cannot be configured"
      fi
   fi
fi</code></pre>

#### Scheduler script
The [`schedule_script.sh`](https://github.com/septian-putra/emr-monitoring/blob/master/schedule_script.sh) is the shell script used to copy the monitoring script from the Amazon S3 artifacts folder and schedules the monitoring script to run every 5 minutes.
<pre><code class="language-bash">#!/bin/sh
set -x
# check if bucket and R53 domain name is passed
if [[ "$#" -eq 1 ]]; then
    ARTIFACTS_BUCKET=$1
else
    echo "value for ARTIFACTS_BUCKET is required"
    exit 1;
fi
# check for master node
IS_MASTERNODE=false
if grep isMaster /mnt/var/lib/info/instance.json | grep true;
then
    IS_MASTERNODE=true
fi
if [ "$IS_MASTERNODE" = true ]
then
    cd ~
    mkdir EMRShutdown
    cd EMRShutdown
    aws s3 cp ${ARTIFACTS_BUCKET} .
    chmod 700 pushShutDownMetrin.sh
    /home/hadoop/EMRShutdown/pushShutDownMetrin.sh --isEMRUsed
    #sudo bash -c 'echo "" >> /etc/crontab'
	sudo bash -c 'echo "*/5 * * * * root /home/hadoop/EMRShutdown/pushShutDownMetrin.sh --isEMRUsed" >> /etc/crontab'
fi</code></pre>
#### Monitoring script
The [`pushShutDownMetrin.sh`](https://github.com/septian-putra/emr-monitoring/blob/master/pushShutDownMetrin.sh) is the monitoring script which is scheduled to be run on the master node every 5 minutes and send some metrics to CloudWatch, including `isEMRUsed` metric.

### How to Use It?

To use this solution, we just need to execute two shell scripts as an EMR step before any other steps running in our cluster using the following [JAR location](s3://eu-west-1.elasticmapreduce/libs/script-runner/script-runner.jar).

1. Step to add JupyterHub API token
    <p align="center"><img src="/assets/images/posts/terminate-emr-step1.png"></p>

2. Step to schedule pushing metrics to CloudWatch.
    <p align="center"><img src="/assets/images/posts/terminate-emr-step2.png"></p>


#### Reference
1. [Optimize Amazon EMR costs with idle checks and automatic resource termination using advanced Amazon CloudWatch metrics and AWS Lambda](https://aws.amazon.com/blogs/big-data/optimize-amazon-emr-costs-with-idle-checks-and-automatic-resource-termination-using-advanced-amazon-cloudwatch-metrics-and-aws-lambda/)