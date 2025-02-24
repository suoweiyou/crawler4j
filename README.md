https://www.zhihu.com/question/31427895  GitHub 上有哪些优秀的 Java 爬虫项目？

# crawler4j
[![Build Status](https://travis-ci.org/yasserg/crawler4j.svg?branch=master)](https://travis-ci.org/yasserg/crawler4j)
[![Maven Central](https://img.shields.io/maven-central/v/edu.uci.ics/crawler4j.svg?style=flat-square)](https://search.maven.org/search?q=g:edu.uci.ics%20a:crawler4j)
[![Gitter Chat](http://img.shields.io/badge/chat-online-brightgreen.svg)](https://gitter.im/crawler4j/Lobby)

crawler4j is an open source web crawler for Java which provides a simple interface for
crawling the Web. Using it, you can setup a multi-threaded web crawler in few minutes.

## Table of content

- [Installation](#installation)
- [Quickstart](#quickstart)   
- [More Examples](#more-examples)
- [Configuration Details](#configuration-details)
- [License](#license)

## Installation

### Using Maven

Add the following dependency to your pom.xml:

```xml
    <dependency>
        <groupId>edu.uci.ics</groupId>
        <artifactId>crawler4j</artifactId>
        <version>4.4.0</version>
    </dependency>
```

### Using Gradle

Add the following dependency to your build.gradle file:

    compile group: 'edu.uci.ics', name: 'crawler4j', version: '4.4.0'

## Quickstart
You need to create a crawler class that extends WebCrawler. This class decides which URLs
should be crawled and handles the downloaded page. The following is a sample
implementation:
```java
public class MyCrawler extends WebCrawler {

    private final static Pattern FILTERS = Pattern.compile(".*(\\.(css|js|gif|jpg"
                                                           + "|png|mp3|mp4|zip|gz))$");

    /**
     * This method receives two parameters. The first parameter is the page
     * in which we have discovered this new url and the second parameter is
     * the new url. You should implement this function to specify whether
     * the given url should be crawled or not (based on your crawling logic).
     * In this example, we are instructing the crawler to ignore urls that
     * have css, js, git, ... extensions and to only accept urls that start
     * with "https://www.ics.uci.edu/". In this case, we didn't need the
     * referringPage parameter to make the decision.
     */
     @Override
     public boolean shouldVisit(Page referringPage, WebURL url) {
         String href = url.getURL().toLowerCase();
         return !FILTERS.matcher(href).matches()
                && href.startsWith("https://www.ics.uci.edu/");
     }

     /**
      * This function is called when a page is fetched and ready
      * to be processed by your program.
      */
     @Override
     public void visit(Page page) {
         String url = page.getWebURL().getURL();
         System.out.println("URL: " + url);

         if (page.getParseData() instanceof HtmlParseData) {
             HtmlParseData htmlParseData = (HtmlParseData) page.getParseData();
             String text = htmlParseData.getText();
             String html = htmlParseData.getHtml();
             Set<WebURL> links = htmlParseData.getOutgoingUrls();

             System.out.println("Text length: " + text.length());
             System.out.println("Html length: " + html.length());
             System.out.println("Number of outgoing links: " + links.size());
         }
    }
}
```
As can be seen in the above code, there are two main functions that should be overridden:

- shouldVisit: This function decides whether the given URL should be crawled or not. In
the above example, this example is not allowing .css, .js and media files and only allows
 pages within 'www.ics.uci.edu' domain.
- visit: This function is called after the content of a URL is downloaded successfully.
 You can easily get the url, text, links, html, and unique id of the downloaded page.

You should also implement a controller class which specifies the seeds of the crawl,
the folder in which intermediate crawl data should be stored and the number of concurrent
 threads:

```java
public class Controller {
    public static void main(String[] args) throws Exception {
        String crawlStorageFolder = "/data/crawl/root";
        int numberOfCrawlers = 7;

        CrawlConfig config = new CrawlConfig();
        config.setCrawlStorageFolder(crawlStorageFolder);

        // Instantiate the controller for this crawl.
        PageFetcher pageFetcher = new PageFetcher(config);
        RobotstxtConfig robotstxtConfig = new RobotstxtConfig();
        RobotstxtServer robotstxtServer = new RobotstxtServer(robotstxtConfig, pageFetcher);
        CrawlController controller = new CrawlController(config, pageFetcher, robotstxtServer);

        // For each crawl, you need to add some seed urls. These are the first
        // URLs that are fetched and then the crawler starts following links
        // which are found in these pages
        controller.addSeed("https://www.ics.uci.edu/~lopes/");
        controller.addSeed("https://www.ics.uci.edu/~welling/");
    	controller.addSeed("https://www.ics.uci.edu/");
    	
    	// The factory which creates instances of crawlers.
        CrawlController.WebCrawlerFactory<BasicCrawler> factory = MyCrawler::new;
        
        // Start the crawl. This is a blocking operation, meaning that your code
        // will reach the line after this only when crawling is finished.
        controller.start(factory, numberOfCrawlers);
    }
}
```
## More Examples
- [Basic crawler](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/basic/): the full source code of the above example with more details.
- [Image crawler](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/imagecrawler/): a simple image crawler that downloads image content from the crawling domain and stores them in a folder. This example demonstrates how binary content can be fetched using crawler4j.
- [Collecting data from threads](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/localdata/): this example demonstrates how the controller can collect data/statistics from crawling threads.
- [Multiple crawlers](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/multiple/): this is a sample that shows how two distinct crawlers can run concurrently. For example, you might want to split your crawling into different domains and then take different crawling policies for each group. Each crawling controller can have its own configurations.
- [Shutdown crawling](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/shutdown/): this example shows have crawling can be terminated gracefully by sending the 'shutdown' command to the controller.
- [Postgres/JDBC integration](crawler4j-examples/crawler4j-examples-postgres/): this shows how to save the crawled content into a Postgres database (or any other JDBC repository), thanks [rzo1](https://github.com/rzo1/).

## Configuration Details
The controller class has a mandatory parameter of type [CrawlConfig](crawler4j/src/main/java/edu/uci/ics/crawler4j/crawler/CrawlConfig.java).
 Instances of this class can be used for configuring crawler4j. The following sections
describe some details of configurations.

### Crawl depth
By default there is no limit on the depth of crawling. But you can limit the depth of crawling. For example, assume that you have a seed page "A", which links to "B", which links to "C", which links to "D". So, we have the following link structure:

A -> B -> C -> D

Since, "A" is a seed page, it will have a depth of 0. "B" will have depth of 1 and so on. You can set a limit on the depth of pages that crawler4j crawls. For example, if you set this limit to 2, it won't crawl page "D". To set the maximum depth you can use:
```java
crawlConfig.setMaxDepthOfCrawling(maxDepthOfCrawling);
```
### Enable SSL
To enable SSL simply:

```java
CrawlConfig config = new CrawlConfig();

config.setIncludeHttpsPages(true);
```

### Maximum number of pages to crawl
Although by default there is no limit on the number of pages to crawl, you can set a limit
on this:

```java
crawlConfig.setMaxPagesToFetch(maxPagesToFetch);
```

### Enable Binary Content Crawling
By default crawling binary content (i.e. images, audio etc.) is turned off. To enable crawling these files:

```java
crawlConfig.setIncludeBinaryContentInCrawling(true);
```

See an example [here](crawler4j-examples/crawler4j-examples-base/src/test/java/edu/uci/ics/crawler4j/examples/imagecrawler/) for more details.

### Politeness
crawler4j is designed very efficiently and has the ability to crawl domains very fast
(e.g., it has been able to crawl 200 Wikipedia pages per second). However, since this
is against crawling policies and puts huge load on servers (and they might block you!),
since version 1.3, by default crawler4j waits at least 200 milliseconds between requests.
However, this parameter can be tuned:

```java
crawlConfig.setPolitenessDelay(politenessDelay);
```

### Proxy
Should your crawl run behind a proxy? If so, you can use:

```java
crawlConfig.setProxyHost("proxyserver.example.com");
crawlConfig.setProxyPort(8080);
```
If your proxy also needs authentication:
```java
crawlConfig.setProxyUsername(username);
crawlConfig.setProxyPassword(password);
```

### Resumable Crawling
Sometimes you need to run a crawler for a long time. It is possible that the crawler
terminates unexpectedly. In such cases, it might be desirable to resume the crawling.
You would be able to resume a previously stopped/crashed crawl using the following
settings:
```java
crawlConfig.setResumableCrawling(true);
```
However, you should note that it might make the crawling slightly slower.

### User agent string
User-agent string is used for representing your crawler to web servers. See [here](http://en.wikipedia.org/wiki/User_agent)
for more details. By default crawler4j uses the following user agent string:

```
"crawler4j (https://github.com/yasserg/crawler4j/)"
```
However, you can overwrite it:
```java
crawlConfig.setUserAgentString(userAgentString);
```

## License

Copyright (c) 2010-2018 Yasser Ganjisaffar

Published under [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0), see LICENSE
