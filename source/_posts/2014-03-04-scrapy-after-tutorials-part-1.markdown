---
layout: post
title: "Scrapy After the Tutorials Part 1"
date: 2014-03-04 10:22:33 -0400
comments: true
categories: [python, scrapy]
---
I was given the task of building a scalable web scraper to harvest connections between domains of a specific industry in order to generate a network model of that industry’s online presence. The scraper will need to run for a period of time, (say a week), and the resulting harvest would be the raw data from which the model would be generated. This harvesting process will need to be repeatable to create ‘snapshots’ of the network for future longitudinal analysis. The implementation will use [scrapy](http://www.scrapy.org/) with a [MongoDB](http://www.mongodb.org) back end on a linux platform to be run (ultimately) in AWS.


This post (and subsequent ones) will provide some code samples and documentation of issues faced during the process of getting the scraper operational. The intent is to help bridge the gap between the initial scrapy tutorials and real-world code. The examples assume you have scrapy installed and running, and have at least worked through the basic tutorials. I did find many of the [tutorials](https://github.com/scrapy/scrapy/wiki) on the wiki very helpful and worked through several of them, (multiple times). Get a basic crawl spider up and running and then pick up here.


In this example, I hope to demonstrate the the following scrapy features:

   -Adding a spider parameter and using it from the command line<br/>
   -Getting current crawl depth and the referring url<br/>
   -Setting crawl depth limits<br/>

The code samples below were from a scrapy project named **farm1**

The item:

{% codeblock lang:python item  %}
from scrapy.item import Item, Field

class Farm1Item(Item):
    session_id = Field()
    depth = Field()
    current_url = Field()
    referring_url = Field()
    title = Field()

{% endcodeblock %}

*session_id*: a unique session id for each scrapy run or harvest<br/>
*depth*: the depth of the current page with respect to the start url<br/>
*current_url*: the url of the current page being processed<br/>
*referring_url*: the url of the site which was linked to the current page<br/>

The crawl spider:

{% codeblock lang:python spider  %}
from scrapy.contrib.spiders import CrawlSpider, Rule
from scrapy.contrib.linkextractors.sgml import SgmlLinkExtractor
from scrapy.selector import Selector
from farm1.items import Farm1Item

class Harvester1(CrawlSpider):
    name = 'Harvester1'
    session_id = -1
    start_urls = ["http://www.example.com"]
    rules = ( Rule (SgmlLinkExtractor(allow=("", ),),
                callback="parse_items",  follow= True),
    )

    def __init__(self, session_id=-1, *args, **kwargs):
        super(Harvester1, self).__init__(*args, **kwargs)
        self.session_id = session_id

    def parse_items(self, response):
        sel = Selector(response)
        items = []
        item = Farm1Item()
        item["session_id"] = self.session_id
        item["depth"] = response.meta["depth"]
        item["current_url"] = response.url
        referring_url = response.request.headers.get('Referer', None)
        item["referring_url"] = referring_url
        item["title"] = sel.xpath('//title/text()').extract()
        items.append(item)
        return items

{% endcodeblock %}

The spider uses the [SgmlLinkExtractor](http://doc.scrapy.org/en/latest/topics/link-extractors.html#sgmllinkextractor) and follows every link, (a later post will cover filtering which links to follow).

####Adding a spider parameter and using it from the command line

Lines 14-16 in the spider shows the constructor which has a session_id parameter with a defaul assignment. The session id is assigned to the spider and persisted with each item processed and will be used to identify items from different harvest sessions. Defining the parameter in the constructor allows using it from the command line:

{% codeblock %}
x:~$ scrapy crawl Harvester7 -a session_id=337
{% endcodeblock %}

####Getting current crawl depth and the referring url

Line 23 is where the current depth of the crawl is retrieved. The ‘depth’ key word is one of several [predefined keys](http://doc.scrapy.org/en/latest/topics/request-response.html?highlight=meta#request-meta-special-keys) in the meta dictionary for the request and response objects. The depth value will be used in the analysis phase.

Line 25 shows how to retrieve the referring url. The goal of this project was harvesting connections between domains, not just data from individual pages. For each item processed, the connection between two links: *referring_url –> current_url* is stored. Implied with each connection is the directionality of the link which will help build a directed graph for analysis. This is enabled by default in the [default middleware settings](https://scrapy.readthedocs.org/en/latest/topics/settings.html#std:setting-SPIDER_MIDDLEWARES_BASE), (and note that ‘Referer’ is the correct usage).

####Setting crawl depth limits

Rather than use ctrl-c to kill the crawling spider, you can set a depth limit at which the spider will not go beyond. I found this helpful during testing. DEPTH_LIMIT is a predefined setting that can be assigned in your settings file. This was the only additional setting used in this example other than the defaults created with the project.

{% codeblock %}
DEPTH_LIMIT = 3
{% endcodeblock %}

The depth limit can also be set in the [command line](http://doc.scrapy.org/en/latest/topics/settings.html?highlight=depth#global-overrides), (as can all pre-defined settings):

{% codeblock %}
x:~$ scrapy crawl Harvester1 –s DEPTH_LIMIT=2
{% endcodeblock %}

The command line assignment will take priority of the settings file. Note that the depth limit will have little affect if you are running a broad crawl. Ultimately, when this scraper is released into the wild, it will probably be set to run a [broad crawl](http://doc.scrapy.org/en/latest/topics/broad-crawls.html?highlight=broad#broad-crawls). But it will still need to troll deep to really pull out much of a domain’s public connections. As to how this will be handled, (a mix of broad + deep crawls), I am not sure but it will be documented here once it is figured out.





