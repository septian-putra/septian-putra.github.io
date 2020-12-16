---
layout: post
title: "A/B Testing with AWS Sagemaker"
tags: [ production, machine-learning, aws ]
categories: engineering
image: 'https://www.csrhymes.com//img/static-site-generator.jpg'
published: false
# featured_image_thumbnail: /assets/uploads/coins.jpg
# featured_image: /assets/uploads/coins.jpg
---

I've been working on some data science projects which most of them have an endpoint as the deliverable.
Those endpoints must be able to handle the load in the production environment.
How can we check it? 
<!--more-->

What we can do is test the endpoint by giving the expected maximum load and observe whether it still runs properly. This process usually called load testing. Some measures that can be used to evaluate the endpoint performance, such as:
* **Latency** – The total time between the request sent and the response received by the client. We can calculate the average or the percentiles to see how good is our endpoint. typically < 300ms is good enough.
* **Response** Time_ – Amount of time system takes to process a request after it has received one
* **Throughput** – The number of requests the endpoint can handle simultaneously before it deviates from acceptable performance, usually measured by requests/second
* **Resource Consumption** – We can also observe the resource consumption on our endpoint when handling the expected maximum load, such as the CPU consumption and Memory consumption.
    
In AWS, the Response Time and Resource Consumption metrics can be retrieved from the Cloudwatch Log during the load testing. As for the Latency and Throughput metrics. they usually can be retrieved from the load testing tools that we use. Here are some open-source tools that available for load testing:

## [Bees with Machine Guns!](https://github.com/newsapps/beeswithmachineguns)
Bees with Machine Guns! is a utility for arming (creating) many bees (micro EC2 instances) to attack (load test) targets (web applications). It works on the AWS environment and has the capability to automatically handle the creation and termination of multiple EC2 instances. It is good for testing the endpoint which exposed to multiple clients in multiple regions and availabilities zones and often used to check whether the load balancer is working properly as it can also perform spike testing. 

## [Locust](https://github.com/locustio/locust)
Locust is a Python-based load testing tool with a real-time web UI for monitoring results.
Locust can also be run in distributed mode, where you can run a cluster of Locust servers and have them produce load in a coordinated fashion, but we need to configure it ourselves.
I use this often in my projects and in my opinion it is really good to do a simple load testing. [Here](https://github.com/septian-putra/locust-loadtesting), you can find some example code to use Locust for load testing.

Source:
* [Introduction to Load Testing](https://www.digitalocean.com/community/tutorials/an-introduction-to-load-testing)