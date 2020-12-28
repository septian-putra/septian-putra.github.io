---
layout: post
title: "Managing EC2 Spot Request"
tags: [ aws, python, tips ]
categories: engineering
published: true
---

We can save about **65-75%** of the EC2 cost with Spot Instance. Using Spot Instance is recommended during the development of ML model, especially if we need to use GPU instances. However, managing Spot Instance is tricky and more complicated compared to On Demand instance. Here, I would like to share how I use Spot Instance when developing ML model. 

<!--more-->
### What is EC2 Spot Instance?
Amazon EC2 Spot instances are spare compute capacity in the AWS cloud available at steep discounts compared to On-Demand prices.
The price we pay for the instance is not fixed, but fluctuate according to market price.
We need to specify max bidding price in our request in addition to the instance type. Otherwise, the On-Demand price will be automatically used as our maximum bidding price.
There is a risk that our Spot Instance is interrupted by AWS, which happens for example when our bidding price is too low compared to market price, or no spare-compute capacity available for instance type we choose. The interuption will change the state of our instance into Stop, Hibernate or Terminated depends on the instance type and also the configuration that we choose when requesting the instances. 
<table>
<caption>Comparison Between On-Demand and Spot Request Price</caption>

<thead>
<tr>
<th>Instance Type</th>
<th>On-Demand</th>
<th>Spot-Price</th>
<th>% Money Saved</th>
</tr>
</thead>
<tbody>
<tr>
<td>m4.2xlarge</td>
<td>0.444</td>
<td>0.1375</td>
<td>69.03</td>
</tr>
<tr>
<td>c4.2xlarge</td>
<td>0.478</td>
<td>0.1280</td>
<td>73.22</td>
</tr>
<tr>
<td>g3.4xlarge (GPU)</td>
<td>1.210</td>
<td>0.3726</td>
<td>69.21</td>
</tr>
<tr>
<td>g3.8xlarge (GPU)</td>
<td>2.420</td>
<td>0.7260</td>
<td>70.00</td>
</tr>
<tr>
<td>p2.xlarge (GPU)</td>
<td>0.972</td>
<td>0.2916</td>
<td>70.00</td>
</tr>
<tr>
<td>p3.2xlarge (GPU)</td>
<td>3.305</td>
<td>0.9915</td>
<td>70.00</td>
</tr>
</tbody>
</table>  

### Choosing Instance Type and Availability Zone (AZ)
We can use [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/) to help choosing the best instance type for your task by providing a region and the minimum specifications.
It will provide the available instance types with their estimated chance of getting interruption and Saving Cost over On-Demand.
However, the values are just rough estimation over the last 30 days. The actual value of interruption chance and market price depends also on which AZ we select for our instance.
We can improve our task’s fault tolerance by selecting the instance type with lower frequency of being interrupted.
In case the instance type you choose is not available in one AZ, you can try selecting the other AZs as they might be available there.
<p align="center"><img src="/assets/images/posts/spot-instance-advisor.png"></p>

### Persistent Request and Spot Block
Spot Instance request can be configured as persistent, which remains active until it expires or is canceled You can only set this configuration when creating your spot instance request.
This setting enable you to select `Stop ` or `Hibernate` as interruption behaviour instead of the default, `Terminate`.
 If the request is persistent and you stop your Spot Instance, the request only opens after you start your Spot Instance.
 Also, the request is opened again after your Spot Instance is interrupted by AWS, which can be really powerful in combination with the User data and checkpoint mechanism.
 Another alternative in case a interruption happens is creating an AMI based on interrupted instance and create a new Spot Instance request with different instance type.

 Spot Instances with a defined duration (also known as Spot blocks) is a new features released by the AWS. Spot Blocks allow you to request Amazon EC2 Spot instances for 1 to 6 hours at a time to avoid being interrupted while your job completes, which come at slightly higher price point compared to regular Spot Instance. This makes them ideal for jobs that take a finite time to complete and has no checkpoints. In rare situations, Spot blocks may be interrupted due to Amazon EC2 capacity needs.

### Automate Request and Termination
The step to create spot instance and terminating it via console is time consuming and boring. I created a script to ease those processes, including specifying the instance parameters, choosing the region, bidding the price, and tagging the instance. The script is implemented using the boto3 library and currently available in [this repository](https://github.com/septian-putra/get-spot-instance). The script cover following functionality:    
* Find the cheapest Availability Zones in a region for selected instance type.
* Bid the instance 105% higher from the cheapest price found.
* Configure EC2 user data.
* Request the spot instance and give back the IP address.
* Tagged the spot instance created with Name, Project, and Owner.
* Notify the user if the instance capacity is not available for selected instance.
* Cancel the spot request and terminated the instance.
<p align="center"><img src="/assets/images/posts/spot-instace-manager.png"></p>
