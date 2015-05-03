---
layout: post
title: "Scrapy After the Tutorials Part 2"
date: 2014-03-08 11:22:33 -0400
comments: true
categories: [python, scrapy]
---

As described in the previous post, the scraper will be crawling over extended periods of time. Filtering out links which do not need to be crawled is one way of improving performance: less links to crawl = less items to process. In this example, I hope to demonstrate how to filter the links to be crawled. So how does the scraper determine which links to follow and how can we modify this behavior? Using step #7 in the [Data Flow](http://doc.scrapy.org/en/latest/topics/architecture.html#data-flow) as reference:

> The Spider processes the Response and returns scraped Items and new Requests (to follow) to the Engine.

So when a response is sent to the spider for processing, the spider will first extract all links in the response and send them to the engine to be scheduled for crawling. The spider [rules](http://doc.scrapy.org/en/latest/topics/spiders.html?highlight=rule#crawling-rules) control which links will be passed to the engine for crawling. Each rule uses a [link extractor](http://doc.scrapy.org/en/latest/topics/link-extractors.html) with which you can specify the links to follow or drop using the available parameters. Many of the basic [examples](http://doc.scrapy.org/en/latest/topics/spiders.html?highlight=rule#crawlspider-example) of crawl spiders demonstrate the ‘allow’ and ‘deny’ parameters for the rules. But for this example, we need a bit more functionality.

By default, for each response processed by the spider, the set of urls send to the engine for crawling will include every href tag in the response. To limit the amount of links that will be followed, we can remove the following types of links:

– any urls to the same domain as the response.url<br/>
– any relative urls<br/>

To remove urls to the same domain, we will implement a filter_links() for the crawling Rule. To remove relative urls, we will add a regex to the link extractor to limit all urls returned to fully qualified urls, (e.g. start with http: or https:). The Item has not changed from the last post.

{% codeblock lang:python Spider %}
from scrapy.contrib.spiders import CrawlSpider, Rule
from scrapy.contrib.linkextractors.sgml import SgmlLinkExtractor
from scrapy.selector import Selector
from urlparse import urlparse
from farm1.items import Farm1Item

class Harvester2(CrawlSpider):
    name = 'Harvester2'
    session_id = -1
    response_url = ""
    start_urls = ["http://www.mmorpg.com"]
    rules = (
        Rule (
            SgmlLinkExtractor(
                allow=("((mailto\:|(news|(ht|f)tp(s?))\://){1}\S+)", ),),
            callback="parse_items",
            process_links="filter_links",
            follow= True),
    )

    def __init__(self, session_id=-1, *args, **kwargs):
        super(Harvester2, self).__init__(*args, **kwargs)
        self.session_id = session_id

    def parse_start_url(self, response):
        self.response_url = response.url
        return self.parse_items(response)

    def parse_items(self, response):
        self.response_url = response.url
        sel = Selector(response)
        items = []
        item = Farm1Item()
        item["session_id"] = self.session_id
        item["depth"] = response.meta["depth"]
        item["title"] = sel.xpath('//title/text()').extract()
        item["current_url"] = response.url
        referring_url = response.request.headers.get('Referer', None)
        item["referring_url"] = referring_url
        items.append(item)
        return items

    def filter_links(self, links):
        baseDomain = self.get_base_domain( self.response_url)
        filteredLinks = []
        for link in links:
            if link.url.find(baseDomain) < 0:
                filteredLinks.append(link)
        return filteredLinks

    def get_base_domain(self, url):
        base = urlparse(url).netloc
        if base.upper().startswith("WWW."):
            base = base[4:]
        elif base.upper().startswith("FTP."):
            base = base[4:]
        # drop any ports
        base = base.split(':')[0]
return base
{% endcodeblock %}

Note: you can add this spider to the previous project by dropping it in the same folder as the other spider. As long as the file name and spider name are different, it will run with the same item and settings.

The response_url class variable (line 10) will maintain the current response.url and will be needed for filtering urls.

The [parse_start_url()](http://doc.scrapy.org/en/latest/topics/spiders.html?highlight=rule#scrapy.contrib.spiders.CrawlSpider.parse_start_url) method (line 25) overrides the base definition and is called only for the defined start_urls and processes each start url as an item. Most importantly, it initializes the response_url variable before any crawling.

The parse_item() method has not changed from the last example.

The get_base_domain() method (line 51) returns the base domain for a url. For example, if ‘http ://www.b.com:334/x/y.htm?z’ is passed in, ‘b.com’ will be returned.

The filter_links() method will filter all links passed in and only return links to external domains. The base domain of the response_url is retrieved by get_base_domain() method (line 44). Any links in the input list of links which contain this base domain are filtered out. Only links to different (external) domains are returned. This method is assigned to the process_links parameter in the Rule (line line 40).

The allow parameter for the link extractor (line 15) is assigned the regular expression “((mailto\:|(news|(ht|f)tp(s?))\://){1}\S+)” which precludes any relative urls by only allowing urls that start with a [URI Scheme](http://en.wikipedia.org/wiki/URI_scheme) (e.g. ftp, http, etc.).

When a response is sent to a spider for processing, the filter_links() method is called before the process_item() method. In this example, if the response_url variable is not set, it will fail. This is the reason we needed to use the parse_start_url() method, which is called for each defined start_url and allows us to set the response_url variable before any links are processed by filter_links(). This only allows fully qualified links to external urls to be sent to the engine for scheduled crawling.

