---
title:  "Tweet Ingestion with NiFi and Elasticsarch"
date:   2019-05-25 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#eadcb5">On This Page</font></a>
header:
  image: /assets/images/Tweet_Ingestion_with_NiFi_and_Elasticsearch.png
  teaser: /assets/images/Tweet_Ingestion_with_NiFi_and_Elasticsearch.png
categories:
  - Programming
tags: 
  - Elasticsearch 
  - NiFi
  - Network
  - Twitter
---

[Apache NiFi](https://nifi.apache.org/){:target="_blank"} is an open source data ingestion system that offers up a rich variety of components you can assemble to ingest, enrich, and store data.  NiFi can grab data from any number of data sources, including logs and live streams. This article will exlplore the latter, ingesting tweets from the Twitter public feed and indexing them in Elasticsearch. 

## Install NiFi

You can get NiFi by building from [source](https://github.com/apache/nifi){:target="_blank"} or just grabbing working [binaries](https://nifi.apache.org/download.html){:target="_blank"}. I suggest using the binary version to get *NiFi* up and running as quickly as possible. 

After downloading and unpacking the *NiFi* binary package, go to the NiFi directory and run `bin/nifi.sh start`.  It will take a coulple of miniutes for *NiFi* to start. When is running open the NiFi console at *http://localhost:8080/nifi* in a browser. The default listening port 8080 can be changed by setting the `nifi.web.http.port` field in the *conf/nifi.properties* configuration file. You should get a blank console canvas that looks like this:

![Nifi_blank_canvas](/assets/images/Nifi_blank_canvas.png)

## Install Elasticsearch

If you don't already have Elasticsearch, you can install it on your local system with these commands:

{% highlight bash %}
mkdir elasticsearch-example
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.1.0-linux-x86_64.tar.gz
tar -xzf elasticsearch-7.1.0.tar.gz
{% endhighlight %}

For the Mac use this *wget* command instead:

{% highlight bash %}
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.1.0-linux-x86_64.tar.gz
{% endhighlight %}

You can run Elasticsearch from the command line like this:

{% highlight bash %}

./elasticsearch-7.1.0/bin/elasticsearch

{% endhighlight %}

By default, Elasticsearch listens for requests on port 9200.

## Twitter Data Pipeline

NiFi data pipelines consist of one or more *processors* that are connected together to ingest, filter/enrich data, and output data to some sort of storage medium. At the very least there must be an ingestion processor that pipes sends data to an output processor. Filtering and enriching processors are optional, but more often than not you need these intermediate processors to clean and map the data you want to end up with.

The Twitter data pipeline built in this article will use the following five processors in order, one connected to the other:

- `GetTwitter` - Makes a connection with and receives tweet data from the Twitter Public Feed. 
- `EvaluateJsonPath` - Extracts specified JSON fields to acted upon by downstream processors.
- `RouteOnAttribute` - Take actions on fields that come from an upstream processor.
- `JoltTransformJson` - Map and transform JSON fields.
- `PutElasticsearchHttp` - Index output messages in Elasticsearch using the RESTful API.

Processors are added to the NiFi console by dragging the processor button to the empty canvas.

![Nifi_processor](/assets/images/Nifi_processor.png){:width="60%" .align-center}

When you release the processor button, a dialog box will be displayed in which you specify the processor you want to add.

![Nifi_add_processor](/assets/images/Nifi_add_processor.png){:width="80%" .align-center}

As you type in the *Filter* box, the list of processors will be whittled down until you type the one you want or select the one you want from the list.  For this pipeline project, you will want to create the processors in the list, so you can just copy the text for each one into the *Filter* box and click on the *Add* button to create the processor in question.  When you are done you can postion the processors to your liking, but in the end you will have a canvas that looks something like this:

![Nifi_unconfigured_processors](/assets/images/Nifi_unconfigured_processors.png)

The processors show the yellow triangular indicator until they are configured. To configure a processor, right click anywhere on the proccessor panel then select *Configure*.

## Configure Data Pipeline Processors

### GetTwitter

Before configuring the *GetTwitter* processor, you have to create a Twitter application on the [Twitter Developers](https://developer.twitter.com/en/apps){: target="_blank"} site. After doing that you can click on the *Details* button for your app then the *Keys and tokens* tab to get consumer keys and access tokens you need to configure the processor.  The  *GetTwitter* configuration dialog box looks like this:

![Nifi_configure_get_twitter](/assets/images/Nifi_configure_get_twitter.png){:width="80%" .align-center} 

To enter consumer keys and access tokens, click on each property then enter the values for your Twitter app.  Here is an example of how to add the `Consumer key` property:

![Nifi_configure_get_twitter_consumer_key](/assets/images/Nifi_configure_get_twitter_consumer_key.png){:width="80%" .align-center} 

It is a good idea to give the processors a descriptive name that summarizes what each one does.  Select the  *Settings* tab and enter the name `Ingest Tweets from Public Feed`.

When you are done configuring the processor, click on *Apply* to finish.

### EvaluateJsonPath

The *EvaluateJsonPath* is used to check whether there is a `text` field present in each tweet.  If there is it will set  property with the `text` field string then pass it on to the *RouteOnAttribute* processor. Open the *EvaluateJsonPath* processor configuration dialog then set the `Destination` property to `flowfile-attribute` . Click on the plus icon to add the property `twitter.text` and set it to `$.text` as shown below:

![Nifi_configure_evaluate_json_path_props](/assets/images/Nifi_configure_evaluate_json_path_props.png){:width="80%" .align-center} 

Set the processor name to `Get Tweet Text Field` in the *Settings* tab.  Check the `failure` and `unmatched` boxes in the `Automatically Terminate Relationships` section to drop messages that are not properly formatted or do not have a text field.

![Nifi_configure_evaluate_json_path_terminate](/assets/images/Nifi_configure_evaluate_json_path_terminate.png){:width="80%" .align-center} 

When you are done configuring the processor, click on *Apply* to finish.

### RouteOnAttribute

The *RouteOnAttribute* processor forwards tweets from *EvaluateJsonPath* that have `text` fields with a lneght or 1 or more characters.  To do that add a property called `tweet` then set it to `${twitter.text:isEmpty():not()}`.

![Nifi_configure_route_on_attr_props](/assets/images/Nifi_configure_route_on_attr_props.png){:width="80%" .align-center} 

Then set the name of the processor to `Drop Invalid Tweets` and automatically terminate unmatched relationships.

![Nifi_configure_route_on_attr_terminate](/assets/images/Nifi_configure_route_on_attr_terminate.png){:width="80%" .align-center}

When you are done configuring the processor, click on *Apply* to finish.

### JoltTransformJson

Tweets contain **a lot** of fields like tweet text, whether the tweet was favorited or retweeted, tweet time stamp, many user information fields, and so on.  Elasticsearch limits the numebr of fields that can be indexed to 1000. This may seem a like number that would be tough to exceed, but well travelled tweets that have numerous retweets, favorites, hashtags, and the like can exceed this limit.

That's where the *JoltTransformJson* processor comes in, by only forwarding a specific subset of JSON tweet fields.  *JoltTransformJson* can also rename and map fields to your liking. *JoltTransformJson* maps fields with *specifications* that are comprised of one or more JSON objects that specify field name mappings.  For tweets in this example,  the processor specification retains most of the top level fields, with their original names. Several `user` object fields are flattened into the top level and the names are prepended with the term `user` to emphasize their meaning in a tweet. 

To set the  *JoltTransformJson* specification, open the processor configuration panel then add then copy and paste the following JSON into the `Specification` property:

{% highlight bash %}

[
  {
    "operation": "shift",
    "spec": {
      "created_at": "created_at",
      "time_zone": "time_zone",
      "utc_offset": "utc_offset",
      "timestamp_ms": "timestamp_ms",
      "id_str": "id_str",
      "text": "text",
      "source": "source",
      "favorited": "favorited",
      "retweeted": "retweeted",
      "possibly_sensitive": "possibly_sensitive",
      "lang": "lang",
      "user": {
        "id_str": "user_id_str",
        "name": "user_name",
        "screen_name": "user_screen_name",
        "description": "user_description",
        "verified": "user_verified",
        "followers_count": "user_followers_count",
        "friends_count": "user_friends_count",
        "listed_count": "user_listed_count",
        "favourites_count": "user_favourites_count",
        "created_at": "user_created_at",
        "lang": "user_lang"
      }
    }
  }
]

{% endhighlight %}

Click to the *Settings* tab set the processor name to `Map Tweet Fields` and check the `failure` automatic terminate relationship. 

![Nifi_configure_jolt_transform_json_terminate](/assets/images/Nifi_configure_jolt_transform_json_terminate.png){:width="80%" .align-center}

When you are done configuring the processor, click on *Apply* to finish.

### PutElasticsearchHttp

*PutElasticsearchHttp* is the last processor in the data pipeline. It will take the mapped tweets from the *JoltTransformJson* process and index them in an Elasticsearch instance. To configure  *PutElasticsearchHttp*, open the processor configuration panel then set these properties:

- `Elasticsearch URL` - `localhost:9200` or another IP/host if Elasticsearch runs on a remote system.
- `Index` - `tweets-$now():format(yyyy-MM-dd)}`  This will create a new index every day.
- `Type` - `_doc` Default type for Elasticsearch 7.x.  Starting with this version you can only have one type per index.
- `Index Operation` - `Index` Specifies indexing tweets in Elasticsearch, as opposed to querying.

While the configuration dialog is open, click on the *Settings* tab then set the processor name to `Index Tweets in Elasticsearch` . Select the `failure` and `retry`automatic terminate relationships.

When you are done configuring the processor, click on *Apply* to finish.

## Connect Data Pipeline Processors

The tweet data piepline processors can now be connected to define how tweets will move through the pipeline. To make the first connection, move your mouse over the *GetTwitter* processor until an arrow in a dark circle appears, as shown below:

![Nifi_connect_processors](/assets/images/Nifi_connect_processors.png){:width="80%" .align-center}

Drag the cursor outside of the processor box over to the *GetTwitter* processor box then release. the *Create Connection* dialog box will be produced:

 ![Nifi_create_connection](/assets/images/Nifi_create_connection.png){:width="80%" .align-center}

Click the *Add* button to complete the connection. Repeat this process to connect *EvaluateJsonPath* to *RouteOnAttribute*, *RouteOnAttribute* to *JoltTransformJson*, and *JoltTransformJson* to *PutElasticsearchHttp*.    

When you get to *PutElasticsearchHttp*, click and drag the cursor without leaving the processor border then release.  This connects the process to itself and produces another *Create Connection* dialog box. Check the `failure`, `retry`, and `success` boxes then click on the *Add* button to complete the connection. After all the connections are completed the data pipeline canvas should look like this:

![Nifi_full_canvas](/assets/images/Nifi_full_canvas.png)

To make it easier to start and stop the data pipeline, you can put the processors in a group.  Select all the processors and connetions on the NiFi canvas by typing Command-A on a Mac keyboard or Ctrl-A on any other system keyboard, then click on the *Group* button in the *Operate* box to the left of the canvas. 

![Nifi_group_processors](/assets/images/Nifi_group_processors.png)

Enter `tweet-nifi` in the *Process Group Name* text box, then click on *Add* to finish.  The NiFi canvas should now look like this:

![Nifi_tweet_nifi_group](/assets/images/Nifi_tweet_nifi_group.png)

The final step in building the pipeline is to save it as a template.  Click on *Create Template* button in the *Originate* panel, then enter template name `tweet-nifi` , and click on the *Create* button. If at a later time you want to load the twiiter data pipeline tempalte and run it again, drag the *Template* button from the top toolbar and select the `tweet-nifi` template.

![Nifi_add_template](/assets/images/Nifi_add_template.png){:width="80%" .align-center}

## Tweet Index Mapping

When it comes to tweet indexing in Elasticsearch, the heavy lifting as far as field selection is concerned is done by the *JoltTransformJson* processor. Elasticsearch does a pretty good job of dynamically mapping fields, but it is a good practice to explicitly map fields to make sure Elasticsearch gets the field types right. This is particlarly true of date fields that come in various formats.  One example is the `timestamp_ms` tweet field. You want to make sure Elasticsearch maps this field as a date so Elasticsearch doesn't treat this field as a string preventing you from using date aritmetic in your tweet seaches. The following script creates a mapping template for the time series tweet indexes:

{% highlight bash linenos %}

#!/usr/bin/env bash

if [[ $# -ne 1 ]] ; then
    echo "usage: tweets_template.sh node"
    exit
fi

curl -H 'Content-Type: application/json' -XPUT 'http://'$1'/_template/tweets' -d '
{
  "index_patterns": ["tweets-*"],
  "mappings": {
      "properties": {
        "timestamp_ms": {
          "type": "date",
          "format": "epoch_millis"
        },
        "created_at": {
          "type": "text",
          "fields": {
            "keyword": { 
              "type":  "keyword"
            }
          }
        },
        "time_zone": {
          "type": "keyword" 
        },
        "utc_offset": {
          "type": "keyword"
        },
        "id_str": {
          "type": "keyword"
        },
        "text": {
          "type": "text",
          "fields": {
            "keyword": { 
              "type":  "keyword"
            }
          }
        },
        "favorited": {
          "type": "boolean"
        },
        "retweeted": {
          "type": "boolean"
        },
        "possibly_sensitive": {
          "type": "boolean"
        },
        "lang": {
          "type": "keyword"
        },
        "user_id_str": {
          "type": "keyword"
        },
        "user_name": {
          "type": "keyword"
        },
        "user_screen_name": {
          "type": "keyword",
          "fields": {
            "keyword": { 
              "type":  "keyword"
            }
          }
        },
        "user_description": {
          "type": "text"
        },
        "user_verified": {
          "type": "boolean"
        },
        "users_followers_count": {
          "type": "integer"
        },
        "users_friends_count": {
          "type": "integer"
        },
        "users_listed_count": {
          "type": "integer"
        },
        "user_favourites_count": {
          "type": "integer"
        },
        "user_created_at": {
          "type": "text",
          "fields": {
            "keyword": { 
              "type":  "keyword"
            }
          }
        },
        "user_lang": {
          "type": "keyword"
        }
      }
    }
 }'

echo

{% endhighlight %}

The `node` argument for this script refers to the Elasticsearch IP and port, e.g. *localhost:9200*. 

The index pattern is set to `tweets-*` in lines 13 - 15,  meaning that the mapping will be applied to any tweet index that has a name the begins with `tweet-`.  Numerical and boolean tweet fields are mapped accordingly.  All other text and date fields are mapped as text and include an additional keyword field for sorting.

Run this script for the Elasticsearch instance you are using, then the mapping template will be set to go.

{% highlight bash  %}

./tweets_template.sh localhost:9200

{% endhighlight %}

## Run the Data Pipeline

If you want to just get started with tweet ingestion, you can get the source code on Github in the [tweet-nifi](https://github.com/vichargrave/tweet-nifi){:target="_blank"} repo. After unpacking the tarball, you can load the tweet-nifi facility by following these steps:

1. Click on the *Upload Template* button in the *Operate* pane. 
2. Click on the maginfying glass icon next to the text *Select Template* in the *Upload Template* pane.
3. Navigate to the directory where you unpacked the tweet-nifi tarball.
4. Select the *tweet-nifi.xml* file.
5. Click *Open*.
6. Click on the *UPLOAD* button in the *Upload Template* pane.
7. Click on the *Template* button on the top of the NiFi window, then drag it to the open canvas.
8. Select the *tweet-nifi* template from the the *Choose Template* dropdown menu.
9. Click the *Add* button.

Now double click on the *Ingest Tweets from Public Feed* process and set the Twitter Consumer and Access Token fields as described earlier in this article.  

To start the tweet ingestion and indexing, click on the play button in the *Operate* box. After running the piepline for several minutes, you can run a query to check that tweets were indexed. Run a *curl* command like this:

{% highlight bash  %}

curl http://localhost:9200/tweets\*/_search\?pretty

{% endhighlight %}

The output of this command should look like this:

{% highlight bash  %}

{
  "took" : 4565,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "tweets-2019-05-25",
        "_type" : "_doc",
        "_id" : "xS_P8WoBB9fmJNnwbj2e",
        "_score" : 1.0,
        "_source" : {
          "created_at" : "Sun May 26 01:44:12 +0000 2019",
          "timestamp_ms" : "1558835052659",
          "id_str" : "1132462407165632512",
          "text" : "RT @startrekcbs: 25 years after #StarTrek #TNG ended, a new chapter begins. #StarTrekPicard starring @SirPatStew is coming soon to @CBSAllA…",
          "source" : "<a href=\"http://twitter.com/download/android\" rel=\"nofollow\">Twitter for Android</a>",
          "favorited" : false,
          "retweeted" : false,
          "lang" : "en",
          "user_id_str" : "15274698",
          "user_name" : "Colby Ringeisen",
          "user_screen_name" : "colbyringeisen",
          "user_description" : "Software Engineer in #ATx, #Longhorns fan, dog owner, mountain biker #mtb, part-time hermit",
          "user_verified" : false,
          "users_followers_count" : 120,
          "users_friends_count" : 361,
          "users_listed_count" : 9,
          "user_favourites_count" : 49,
          "user_created_at" : "Mon Jun 30 00:43:06 +0000 2008",
          "user_lang" : null
        }
      },

...

{% endhighlight %}

## Summing Up

Now that you have a basic working NiFi data pipeline, there are several things you can explreo to improve it. If you want to expand the number of tweet fields you ingest, you can add the field maps to the *JoltTransformJson* processor and the *tweets_template.sh* .  

Running NiFI and Elasticsearch on a single system may cause the NiFi queues to hit a critical point and bog down considerably. It's possible to run NiFi on a cluster of systems to get better throughput. It might also be a good idea to run Elasticsearch on a separate cluster. 
