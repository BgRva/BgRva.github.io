---
layout: post
title: "Deploy Crawler to EC2 With Scrapyd"
date: 2014-04-13 21:21:01 -0400
comments: true
categories: [python, scrapy, MongoDB, scrapyd]
---

This post will walk through using [scrapyd](http://scrapyd.readthedocs.org/en/latest/) to deploy and manage a [scrapy](http://scrapy.org/) spider from a local linux instance to a remote EC2 linux instance. The steps below were run on a local Ubuntu 12.04 instance any one of the free AWS linux Ubuntu 12.04 tiers. This is another step in the development of the scraping project described in the first post for which the indented goal is to have multiple instances of scrapy spiders crawling for extended periods of time and saving the items to a common database for further processing.

This post assumes you have scrapy and scrapyd installed on a local (linux) system, a working scrapy crawler, an AWS account, and have some knowledge of basic web security practices. Note: If you have not yet installed scrapy on your local system, when installing pip for scrapy, install the ‘setup tools’ and not the ‘distro tools’, as the setup tools will be needed by scrapyd for ‘egg-i-fying’ crawlers for deployment.

If you don’t have an AWS then get one set up first and spend some time reading about [IAM best practices](http://docs.aws.amazon.com/IAM/latest/UserGuide/IAMBestPractices.html). In particular, don’t use your main account credentials for development. Create a [new user](http://docs.aws.amazon.com/IAM/latest/UserGuide/IAMBestPractices.html#create-iam-users) in your AWS account and assign that user admin rights and do all development with that user. This separates you development credentials from your master account credentials (which have your credit card).

###Setting up scrapy on a EC2 Ubuntu instance

These are the broad steps I took to get an Ubuntu Server 12.04 LTS (free tier) instance up and running with scrapy and scrapyd installed. They do not cover all the details. If this is your first time with AWS, there are plenty of docs and quality videos available – invest some time in them.

1) Log into your AWS account

2) Go to your EC2 Dashboard

3) Create a Security Group with the following inbound and outbound rules:

Inbound (for inbound ssh and scrapyd communication):
{% img /images/ec2-inbound-rules.jpg 'inbound rules' 'image of ec2 inbound rules' %}

Outbound (for outbound scrapyd and crawler http gets):
{% img /images/ec2-outbound-rules.jpg 'outbound rules' 'image of ec2 outbound rules' %}

4) Create a key value pair for the ‘dev’ user and download it so you have access to it on your local system

5) Launch a linux instance and associate the security group and key value pair with this instance, (we will use my-ec2.amazonaws.com as the public dns of the instance)

6) When the instance is running, connect to the EC2 instance using the key-value pair, [connect to your instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

8) Install scrapy and scrapyd on the EC2 instance

###Prepare the crawler for deployment

Scrapyd deployment [targets](http://scrapyd.readthedocs.org/en/latest/deploy.html#show-and-define-targets) are specified in a crawler’s *scrapy.cfg* file. The scrapyd commands to deploy a crawler need to be run in the root directory of your crawler project (or use fully qualified paths).

1) On your local system, go to the root folder of your crawler project

2) Edit *scrapy.cfg* by replacing any *deploy* sections so that the file looks like the following:

{% codeblock lang:python scrapy.cfg  %}
# Automatically created by: scrapy startproject
#
# For more information about the [deploy] section see:
# http://doc.scrapy.org/en/latest/topics/scrapyd.html

[settings]
default = projectX.settings

[deploy:local-target]
url = http://localhost:6800/
project = projectX

[deploy:aws-target]
url = http://my-ec2.amazonaws.com:6800/
project = projectX
{% endcodeblock %}

Here we have added two scrapyd deployment targets, one to localhost and one to the EC2 instance.

3) Verify the targets with the command:

{% codeblock  %}
scrapyd-deploy -l
{% endcodeblock %}

you should see

{% codeblock  %}
local-target           http://localhost:6800/
aws-target             http://my-ec2.amazonaws.com:6800/
{% endcodeblock %}

###Deploy the crawler to EC2

Make sure scrapyd is running on the target instance. For our EC2 instance we can check this by using one of the scrapy web service commands or in a browser navigate to the scrapyd web page (format: my-ec2.amazonaws.com:6800). Note: after I had scrapyd installed on EC2, if I stopped and restarted the instance, scrapyd would be run on startup. When I tried to run scrapyd I got an error “port 6800 in use”. If you get a similar error check if scrapyd is already running.

1) To deploy the spider to the EC2 target use the scrapyd command:

{% codeblock  %}
scrapyd-deploy aws-target -p projectX
{% endcodeblock %}

you should see:

{% codeblock  %}
Packing version 1396800856    
    Deploying to project "projectX" in http://my-ec2.amazonaws.com:6800/addversion.json
    Server response (200):
    {"status": "ok", "project": "projectX", "version": "1396800856", "spiders":1}
{% endcodeblock %}

2) Verify the deployment using the scrapyd web service:

{% codeblock scrapyd command  %}
$> curl http://my-ec2.amazonaws.com:6800/listprojects.json
{% endcodeblock %}

you should see:

{% codeblock  %}
{"status": ok, "projects": ["projectX"]}
{% endcodeblock %}

or you can refresh the scrapyd webpage on your remote instance and Available projects

###Schedule and instance of the crawler

1) Use the scrapyd web service to schedule to the spider. Note: for the spiders I needed to run, each had multiple constructor parameters. To schedule a spider with constructor parameters, each parameter must be preceded by -d and they must be in the order as they appear in the spider constructor. For this example, the constructor parameters are *session_id*, *seed_id*, and *seed_url*. The particular spider to be used is *spider2b*:

{% codeblock  scrapyd command %}
curl http://my-ec2.amazonaws.com:6800/schedule.json -d project=projectX -d spider=spider2b -d session_id=33 -d seed_id=54 -d seed_url=http://www.blah.com/
{% endcodeblock %}

you should see a response similar to:

{% codeblock  %}
{"status": "ok", "jobid": "c77c633ac28011e3ac0a0a32be087ede"}
{% endcodeblock %}

2) Verify the job:

{% codeblock scrapyd command %}
curl http://my-ec2.amazonaws.com:6800/listjobs.json?project=projectX
{% endcodeblock %}

will show:

{% codeblock  %}
{"status": "ok", "running": [{"id": "c77c633ac28011e3ac0a0a32be087ede", "spider": "spider2b"}], "finished": [], "pending": []}
{% endcodeblock %}

or you can refresh the scrapyd webpage on your remote instance and check Jobs

3) To stop a job, use the following

{% codeblock  scrapyd command %}
curl http://my-ec2.amazonaws.com:6800/cancel.json -d project=simple -d job=c77c633ac28011e3ac0a0a32be087ede
{% endcodeblock %}

you should see:

{% codeblock  %}
{"status": "ok", "prevstate": "running"}
{% endcodeblock %}

Note that the *schedule* and *cancel* commands of the scrapyd webservice are not immediate, so give them a bit to take affect. When getting started, the scrapyd web page on the EC2 instance is the easiest way of viewing the logs and output items.

