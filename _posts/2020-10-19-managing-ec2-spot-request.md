---
layout: post
title: "Managing EC2 Spot Request"
tags: [ aws, python, tips ]
categories: engineering
published: true
---

Using Spot Instance during when developing ML model is recommended, especially when we need to use GPU instances, as we can save about **65-75%** of the EC2 cost. However, managing it is not as easy as the On Demand instance. I would like to share how I use Spot Instance for developing ML model. 

<!--more-->
### What is EC2 Spot Instance?
Amazon EC2 Spot instances are spare compute capacity in the AWS cloud available at steep discounts compared to On-Demand prices.
The price we pay for the instance is not fixed, but fluctuate according to market price.
To get the instance, we need to specify max bidding price in our request in addition to the instance type. Otherwise, the On-Demand price will be used as maximum bidding price.
There is a risk that our Spot Instance is automatically terminated by AWS. It happens, for example when our bidding price is too low compared to market price, or no spare-compute capacity available for instance type we choose. 
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

### Spot Instance Advisor
The Spot Instance advisor helps you determine pools with the least chance of interruption and provides the savings you get over on-demand rates.
We can minimize the risk of getting interruption by choosing the instace type which has lower terminatio 
AWS keep track the rate of each Spot Instance

### Persistent Spot Instance Request
A persistent Spot Instance request remains active until it expires or you cancel it, even if the request is fulfilled.

b

### Automate Spot Instance request and termination
The step to create spot instance and terminating it via console is time consuming and boring. I created a script to ease those processes, including specifying the instance parameters, choosing the region, bidding the price, and tagging the instance. The script is implemented using the boto3 library and currently available in the GitHub repository. To use it, users need to prepare the requirements as mentioned in the repository.