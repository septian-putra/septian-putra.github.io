---
layout: post
title: "Using Sagemaker Processing Job for Vectorization"
categories: engineering
description: "Controlling Access to Invoke AWS API Gateway"
image: 'https://i.imgur.com/TZPQtBH.jpg'
published: false
author: Septian 
---


Recently, I’ve been working on deployment of machine learning model as endpoint in AWS.
One of the challenge is to make sure that it is securely configured. In my case, the endpoint will be invoked from different AWS account, which is a common problem.
So, a controll access mechanism need to be implemented to only allow specific AWS account invoking the endpoint and I will share how I implement it in this post.

<!--more-->
## Permission Model

To allow an API caller (from another AWS account) to invoke the API or refresh its caching, we need to create IAM policies that permit a specified API caller to invoke the API method, for which the IAM user authentication is enabled.

We also need to create an IAM role representing the API caller, in which we specify a list of IAM users we trust and attach the previous policy on it. The API caller, which is in another AWS account, need to execute AssumeRole action before invoking the endpoint.

We also need to set the method's authorizationType property in AWS API Gateway to AWS_IAM. It will require the caller to submit their IAM user's access keys to be authenticated before they can access the endpoint.

Below is the JSON configuration attached to the IAM policy, `<name>-api-invoker`, with X, Y and Z is depends on your endpoint. You can also set X, Y, Z to `*` to allow access to all endpoint, but it is not recommended to do so. 

{% highlight javascript linenos %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "execute-api:Invoke",
                "execute-api:ManageConnections"
            ],
            "Resource": "arn:aws:execute-api:<X>:<Y>:<Z>"
        }
    ]
}
{% endhighlight %}

After creating the invoker policy for specific endpoint, we create an IAM role for the invoker,`<name>-api-invoker`, and attach the previous policy to it. Below is the example of Trust Relationship setting attached to the IAM role.
You can set user to `root` to give access to all user from the account.

{% highlight javascript linenos %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "apigateway.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<api_account>:<user>"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<another_account>:<user>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
{% endhighlight %}


## Authentication Steps

To call a deployed API or refresh the API caching, API callers need to have permission which requires them to perform IAM authentication actions, including AssumeRole to the InvokerRole we defined in the previous section.

Here is an example of how to call the endpoint from another account:
{% highlight python linenos %}

import requests, boto3, json
from boto3.session import Session
from aws_requests_auth.aws_auth import AWSRequestsAuth
 
role_arn = 'arn:aws:iam::<endpoint_account>:role/<name>-api-invoker'
session_name = 'invoke-test'
 
# Assume Role
sts = boto3.client('sts')
assumed_role_obj = sts.assume_role(RoleArn=role_arn , RoleSessionName=session_name)
session = Session(aws_access_key_id=assumed_role_obj['Credentials']['AccessKeyId'],
                  aws_secret_access_key=assumed_role_obj['Credentials']['SecretAccessKey'],
                  aws_session_token=assumed_role_obj['Credentials']['SessionToken'])
assumed_sts = session.client('sts')
 
# Invoke API
api_endpoint = 'xxxxxxxxxx.execute-api.eu-west-1.amazonaws.com'
endpoint_url = 'https://{}/prod/predict'.format(api_endpoint)
auth = AWSRequestsAuth(aws_access_key=assumed_role_obj['Credentials']['AccessKeyId'],
                       aws_secret_access_key=assumed_role_obj['Credentials']['SecretAccessKey'],
                       aws_token=assumed_role_obj['Credentials']['SessionToken'],
                       aws_host=api_endpoint,
                       aws_region='eu-west-1',
                       aws_service='execute-api')
 
payload = <json_object>
s_data=json.dumps(payload)
response = requests.post(endpoint_url, data=s_data, auth=auth)
response.text
{% endhighlight %}

## What do you think?

If you decide to give the theme a go, it would be great to see how you are using it and if you have any ideas of how it can be developed further.
