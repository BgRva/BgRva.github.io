---
layout: post
title: "Scrapy to MongoDB"
date: 2014-03-23 20:11:01 -0400
comments: true
categories: [python, scrapy, MongoDB]
---


Now that we can crawl some sites (based on the previous posts), we need to persist the data somewhere where it can be retrieved and processed for analysis. Recall that the goal of the scraper was to harvest connections between domains in order to generate a model of a specific industry’s on-line presence. In Scrapy, [Item Pipelines](http://doc.scrapy.org/en/latest/topics/item-pipeline.html) are the prescribed mechanism by which scraped items can be persisted to a data store. In this post, we will be using a custom pipeline extension for [MongoDB](http://www.mongodb.org/) which we will further customize to check for duplicates. The code examples in this post build upon those in the previous examples. It is also assumed that you have some familiarity with MongoDB and have a working (basic) crawl spider. It does not cover installing Scrapy, MongoDB, or any dependencies, (there is plenty of good documentation on these subjects). Before continuing, you will need to install the following (in addition to having Scrapy working):

– [MongoDB](http://www.mongodb.org/)<br/>
– [Scrapy-MongoDB](https://github.com/sebdah/scrapy-mongodb), an item pipeline extension written by Sebastian Dahlgren<br/>

Once installed, the first step will be to get Scrapy-MongoDB working and saving to a collection. Once you’ve got MongoDB installed, create a database named scrapy and within it, a collection named items. You can use the crawl spider from the previous posts and update the settings.py file to use Scrapy-MongoDB. A quick note about MongoDB ids: mongo will automatically add a unique id object with each item if and id field is not specified. We will use default id and benefit from it because the default MongoDB id object contains an embedded [date-time stamp](http://docs.mongodb.org/manual/reference/method/ObjectId.getTimestamp/#ObjectId.getTimestamp). Once you get Scrapy-MongoDB working and you are saving to a collection, we need to extend the [MongoDBPipeline](https://github.com/sebdah/scrapy-mongodb/blob/master/scrapy_mongodb.py) class to include behavior to check for duplicates. A duplicate of the example item is one in which the following fields match:

– Session_id<br/>
– Current_url<br/>
– Referring_url<br/>

The *session_id* is used to group items from different harvests. We want to persist a single connection between two given urls for each harvest session. The *current_url* and *referring_url* will be used to represent the connection between two domains and will be used to generate a directed graph for the model, (a connection between two valid business backed domains can be used to infer a relationship which in turn can be used in social network analysis, more on this later …).

The new class will be called *CustomMongoDBPipeline* and should be placed within the *pipelines.py* file in the scrapy project folder. You can keep the default pipeline initialized with the project in the same file and switch back by changing the settings file.

{% codeblock lang:python pipelines.py %}
from datetime import datetime
from scrapy.exceptions import DropItem
from scrapy_mongodb import MongoDBPipeline

class CustomMongoDBPipeline(MongoDBPipeline):

    def process_item(self, item, spider):
        # the following lines are a duplication of MongoDBPipeline.process_item()
        if self.config['buffer']:
            self.current_item += 1
            item = dict(item)

            if self.config['append_timestamp']:
                item['scrapy-mongodb'] = { 'ts': datetime.datetime.utcnow() }

            self.item_buffer.append(item)

            if self.current_item == self.config['buffer']:
                self.current_item = 0
                return self.insert_item(self.item_buffer, spider)
            else:
                return item

        # if the connection exists, don't save it
        matching_item = self.collection.find_one(
            {'session_id': item['session_id'],
             'referring_url': item['referring_url'],
             'current_url': item['current_url']}
        )
        if matching_item is not None:
            raise DropItem(
                "Duplicate found for %s, %s" %
                (item['referring_url'], item['current_url'])
            )
        else:
            return self.insert_item(item, spider)
{% endcodeblock %}

Lines 9-22 are a duplication of the parent class MongoDBPipeline.process_item() method to maintain compatability with the Scrapy-MongoDB configuration options. Line 25 is where we retrieve an entry that matches the specified fields. If an entry is found, then the current item is dropped and we return to crawling. The settings are:

{% codeblock lang:python settings.py %}
BOT_NAME = 'farm2'

SPIDER_MODULES = ['farm2.spiders']
NEWSPIDER_MODULE = 'farm2.spiders'

#set to 0 for no depth limit
DEPTH_LIMIT = 3
DOWNLOAD_DELAY = 2
CONCURRENT_REQUESTS_PER_DOMAIN = 4

ITEM_PIPELINES = {
    'farm2.pipelines.CustomMongoDBPipeline': 100
}

# 'scrapy_mongodb.MongoDBPipeline' connection
MONGODB_URI = 'mongodb://localhost:27017'
MONGODB_DATABASE = 'scrapy'
MONGODB_COLLECTION = 'items'
{% endcodeblock %}

Recall you can set the session_id from the command line

{% codeblock  %}
scrapy crawl farm2 -a session_id=337
{% endcodeblock %}

