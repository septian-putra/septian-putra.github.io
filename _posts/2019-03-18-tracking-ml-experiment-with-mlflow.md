---
layout: post
title: "Managing ML Experiment with MLFlow"
tags: [ development, machine-learning, logging ]
categories: engineering
published: true
---

How can we manage ML lifecycle? Keeping track only the source code in a repository, like in the software lifecycle management, is not enough as we cannot reproduce exactly same result. To reproduce a version in ML lifecycle, the deliverable must contains the source code, model artifacts, and the performance result. 

<!--more-->
### What is MLFlow?
Based on my experience, it is quite normal to have multiple person working on the same modelling task.
In that case, sharing and comparing the experiments result becomes a complicated problem. Initially, my team used MS Excel Online to keep track our experiment result and to compare each modelling approach. However, it lack the functionality to pair the experiment results with the code and model artifacts to reproduce the result.

MLFlow is an open source platform for managing the end-to-end machine learning lifecycle.
We decide to use one of its functionality, [MLFlow Tracking](https://mlflow.org/docs/latest/tracking.html), to record the source code and compare parameters and results of our experiments. Now, multiple person in our team can run experiments, share the model artifacts and source code, and compare the metric that our previously agreed on in MLFlow Tracking server easily.

### Setup MLFlow Tracking Server
MLFlow Tracking Server is dedicated server which handle metadata storage, artifacts storage and interactive web interface. Setting up MLFlow Tracking Server in AWS environment is easy:
* Choose the instance type based on the amount of expected requests. I use an `t2.micro` as our team is small. 
* Place your EC2 in public Subnet and create a security group to allow the access on port 5000.
* Assign an EC2 role that allow full S3 Access from EC2 instance
* Install mlflow and gitpython package on your python environment.
* Start the MLFLow server by executing the following command:
    <pre><code class="language-bash">nohup mlflow server --default-artifact-root s3://&lt;artifacts-store-bucket&gt;/mlflow/ --host 0.0.0.0 &</code></pre>

### Setup MLflow Client
MLFlow Tracking provided client API for logging the experiments to our server. I created a wrapper library along with the example usage that can be check in [this repository](https://github.com/septian-putra/mlflow-util).

Here are the steps to configure the MLFlow client (EC2 instance or your local machine):
* Install same mlflow and gitpython package as your MLFlow server.
* Setup the AWS CLI with a proper EC2 role for sending the artifact to AWS S3.
* On EC2:
   - Configure your EC2 to use security group allowed to the access to port 5000 of MLFlow server.
   - During the initialization of Experiment object, configure the `tracking_uri` with MLFlow server's private ipv4 address.
* On machine inside Exact IP address range:
   - During the initialization of Experiment object, configure the `tracking_uri` with MLFlow server's public ipv4 address.
* Add this repo as a submodule:
    <pre><code class="language-bash">git submodule add -b master git@github.com:septian-putra/mlflow-util.git mlflow_util</code></pre>


### Logging Experiment Data
Here is example how to use the wrapper library.
<pre><code class="language-py">import os
from experiment import Experiment
config = {
            'experiment_name': 'mlflow_test',
            'user_id': 'Septian',
            'tracking_uri': 'http://10.1.3.49:5000',
            'artifact_location': 's3://cig-ds-dev/mlflow',
            'use_git_version': True
         }
model_path = os.path.expanduser('~/model')
hyperparam = {}
metric = {}
ex_log = Experiment(config)
ex_log.create_run('sample_run')

# Save the hyperparam & metric
ex_log.log_params(hyperparam)
ex_log.log_metrics(metric)

# Delete or clean model directory
shutil.rmtree(model_path)
if not os.path.isdir(model_path): os.makedirs(model_path)

# Save the model/artifact
joblib.dump(model, os.path.join(model_path, 'model.pkl'))
ex_log.log_artifacts(model_path,'model')

# Terminate run
ex_log.terminate_run()</code></pre>

### Additional link
* More methods for this Experiment object based on [**mlflow.tracking.MlflowClient**](https://mlflow.org/docs/latest/python_api/mlflow.tracking.html)
* [Scaling Ride-Hailing with Machine Learning on MLflow](https://databricks.com/session/scaling-ride-hailing-with-machine-learning-on-mlflow)
