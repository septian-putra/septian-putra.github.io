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

### Logging Experiment Data
MLFlow Tracking provided client API for logging the experiment runs to our server. I created a wrapper library along with the example usage that can be check in [this repository](). 