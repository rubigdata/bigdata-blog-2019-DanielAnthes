# Final Assignment

In this blog post I will document my journey as I learn how to analyze data collected in the [Common Crawl](https://commoncrawl.org/) using Spark. I will perform all the steps for a basic analysis. Starting with getting relevant data from the common crawl, analyzing it interactively using Spark Notebooks and finally creating a self contained Spark Application.

## What Data?

For my project I want to look into data crawled from [Twitter](https://twitter.com). To start off I will try to parse out all usernames in the collected data. If this is successful I want to extend the analysis and look into basic statistics on these tweets such as most used hashtags.

## Getting Data

The crawls in the CommonCrawl data base are stored in WARC format. Since for my analysis only a very small part of the data is relevant - only data on Twitter's domain - it is useful to use an indexing service to find out which WARC files contain relevant data. For this blog post I use data from the May 2019 crawl, which can be searched [here](http://index.commoncrawl.org/CC-MAIN-2019-22). To get data from subdomains of twitter.com, I used the search term twitter.com/\*. This returns a json file with useful information on relevant WARC files including the location of the WARC files.
To access the WARC files with the actual data I first download and parse the JSON file in Spark:


```scala
new URL("http://index.commoncrawl.org/CC-MAIN-2019-22-index?url=twitter.com%2F*&output=json") #> new File("FinalAssignmentData/twitteridx.json") !!  
val twitteridx = spark.read.json("FinalAssignmentData/twitteridx.json")
```
The first line saves the search engine output as a JSON File and the second parses its contents and creates a new DataFrame with the file contents. Since Spark creates a structured DataFrame and supports SQL queries we can conveniently have a first quick look at the data.

When doing so it immediately becomes clear that not all entries will be interesting as many of them have a status code '301' or '404'. Since the only interesting WARC files are the ones that correspond to a valid URL I filtered the list so that only these entries remain. Additionally, it seems that there were some parsing errors when reading in the JSON file and as a result some of the URL's include 'robots.txt' as part of the URL. While this may not actually be a problem and the corresponding WARC files may still be valid I exclude these entries for now. (There is more data than I know what to do with anyways)

```scala
spark.sql("SELECT length, status, url FROM tidx WHERE status == '200' AND NOT url LIKE '%robots.txt%'"))
```

After the above steps 573 WARC files remain. To access the WARC files in my program the only field that I need is the one containing the name and location of the WARC file. Therefore I create an RDD containing only this information and the URL and subsequently use the RDD to import the data. To create a valid link to the WARC file the address of CommonCrawl needs to be prepended to the location of the WARC file on their server:

```scala
val WARC_locs = spark.sql("SELECT filename, url FROM tidx WHERE status == '200' AND NOT url LIKE '%robots.txt%'").rdd
val warc_s3_locs = WARC_locs.map(x => ("https://commoncrawl.s3.amazonaws.com/" + x.getString(0), x.getString(1)))
```

The first line creates an RDD of the shape (filename, url) and the second line prepends the address to each filename.

If the analysis were to be performed on an actual cluster I would assume that the crawl data would already be available on the cluster. Since this is not the case for my laptop, I had to find a different way to get all the relevant data and put it on my 'cluster' by hand. I first created an RDD out of all URLs pointing to the relevant WARC files. Subsequently I write the list to a local file and use xargs and wget to download all the WARC files (and copy them back to the 'cluster' in my docker container):

```scala
val addrlist = warc_s3_locs.map(x => x._1)
addrlist.saveAsTextFile("FinalAssignmentData/WARClocs")
```
The above writes the RDD to a file and puts every URL on a new line.

```bash
xargs -a WARCaddresses wget
```
This bash command executes wget for every line in the WARCaddresses file and thus downloads all 573 WARC files.

Lastly, now that all WARC files of interest are available they need to be loaded into the Spark notebook. Happily, `newAPIHadoopFile` accepts folders as input. Loading all 500+ files into a single RDD is as simple as running a single command, the same way as would be done for a single file:

```scala
val WARC_objects= sc.newAPIHadoopFile(
              "/opt/docker/FinalAssignmentData/WARCs",
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )
```

Now that all available data is neatly collected in a single RDD we can start with the actual analysis.

(Note: for the rest of this assignment I will be working only with a small subset of WARC files collected in the way described above, since executing queries on the full dataset are impractically slow)

## Cleaning the Data

Even though I searched the CommonCrawl database exclusively for WARC files related to the twitter domain, a quick look at the targetURIs of the WARC objects revealed that a large number of WARC objects are associated with different URLs and are not relevant for the analysis. Further, I am only interested in WARC entries for responses. These two filter operations reduced the total number of WARC objects in my analysis sample from 175037 to 46.

```scala
// responses only
val twitter_responses = all_warc.filter(x => x._2.header.warcTypeIdx == 2)
//get twitter domain warc files only
val twitter = twitter_responses.filter(x => x._2.header.warcTargetUriStr.contains("twitter.com"))
```
## Getting Usernames inlcuded in the Sample

Conveniently, URLs on Twitter's domain seem to have a fairly consistent format that includes the username the current page is associated with. URLs in the sample I collected had the shape: www.twitter.com/USERNAME (or /@USERNAME). Sometimes the username is followed by more text, for example if the link points to a list (e.g.: https://twitter.com/obamauni/lists/damohub-com). To extract the username only, I wrote a helper function to first remove the leading part of the URL and then extract the username from the rest by removing the '@' and trailing part of the URL if present:

```scala
def getUsername(url : String):String = {
  //remove first part of URL
  val ending = url.substring(20)
  //split on slashes
  val split_ending = ending.split("/")
  //if username is preceded by @, remove it
  if(split_ending(0).charAt(0) == '@')
    return split_ending(0).substring(1)
  else
    return split_ending(0)
}
```

I then mappend this function onto all twitter URLs in my sample:

```scala
twitter.map(x => getUsername(x._2.header.warcTargetUriStr))
```

The result is not perfect, but seems to successfully extract most usernames in my sample. The function I wrote to parse the URL is very simple and does not deal with some cases, such as in the following examples: 'chris_bavin?lang=en' or cases where the url starts with 'pic.twitter.com'.

## Creating a Standalone Application

To perform the analysis described above on an actual cluster it needs to be packaged into a JAR that can be submitted to and executed on a cluster. As a starting point for this I used the example code provided on the course's [GitHub](https://github.com/rubigdata/cc-2019-DanielAnthes). Initially, building the app using `docker build --rm=true -t rubigdata/spark-app .` failed due to missing dependencies - the warcutils dependency was not found - but after modifying the corresponding dependency in the app's build file everything worked as intended (I replaced -SNAPSHOT by master):

```scala

libraryDependencies += "com.github.sara-nl" % "warcutils" % "master"
```



```scala
package org.rubigdata
import nl.surfsara.warcutils.WarcInputFormat
import org.jwat.warc.{WarcConstants, WarcRecord}
import org.apache.hadoop.io.LongWritable;
import org.apache.commons.lang.StringUtils;
import org.apache.spark.sql.SparkSession

object TwitterUsernameApp {
  def main(args: Array[String]) {
    val data_path = "file:///app/sample"
    val spark = SparkSession.builder.appName("TwitterUsernameApp").getOrCreate()
    val sc = spark.sparkContext

    val warc_objs = sc.newAPIHadoopFile(
              data_path,
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )

  val num_warcs = warc_objs.count()
  println(num_warcs + " warc objects created")

  val all_warc = warc_objs.map{wr => wr}.cache()

  // responses only
  val twitter_responses = all_warc.filter(x => x._2.header.warcTypeIdx == 2)
  //get twitter domain warc files only
  val twitter = twitter_responses.filter(x => x._2.header.warcTargetUriStr.contains("twitter.com"))

  println("number of twitter domain WARC files: " + twitter.count())

  // cache twitter RDD
  twitter.cache()

  val usernames = twitter.map(x => getUsername(x._2.header.warcTargetUriStr)).collect()

  println("Number of user names: " + usernames.length)
  println("*** USER NAMES ***")
  for(uname <- usernames){
    println(uname)
  }
  spark.stop()
  }

  def getUsername(url : String):String = {
    //remove first part of URL
    val ending = url.substring(20)
    //split on slashes
    val split_ending = ending.split("/")
    //if username is preceded by @, remove it
    if(split_ending(0).charAt(0) == '@')
      return split_ending(0).substring(1)
    else
      return split_ending(0)
  }
}
```

Running this app gave the following output:

```
Number of user names: 46
*** USER NAMES ***
jeroenvaninkel
om
Hexxeh
iciradio87
gerardekdom
Wnicholasgomes
dnupdate
photomarathonuk
...
```
## Extracting hashtags

Next, I tried to build on this basic program by extracting all hashtags used in my twitter data sample. To do so I built the following app:

```scala
package org.rubigdata
import nl.surfsara.warcutils.WarcInputFormat
import org.jwat.warc.{WarcConstants, WarcRecord}
import org.apache.hadoop.io.LongWritable;
import org.apache.commons.lang.StringUtils;
import org.apache.spark.sql.SparkSession
import java.io.InputStreamReader;
import java.io.IOException;
import org.jsoup.Jsoup;
import scalaz._
import Scalaz._

object TwitterHastagApp {
  def main(args: Array[String]) {
    val data_path = "file:///app/sample"
    val spark = SparkSession.builder.appName("TwitterHastagApp").getOrCreate()
    val sc = spark.sparkContext

    val warc_objs = sc.newAPIHadoopFile(
              data_path,
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )

    val twitter_contents = warc_objs.filter(_._2.hasPayload()).
                                        filter(x => x._2.header.warcTypeIdx == 2).
                                        map{wr => (wr._2.header.warcTargetUriStr,getContent(wr._2))}

    val twittertxt = twitter_contents
                      .filter(x => x._1 contains "twitter")
                      .filter(x => !(x._1 contains "robots.txt"))
                      .map(x => (x._1, HTML2Txt(x._2)))  

    val twitter_tkns = twittertxt.map(x => (x._1, getUsername(x._1), x._2.split(" ")))

    val hashtags = twitter_tkns.map(x => (x._1, x._2, x._3.filter(_.startsWith("#"))))

    val counts = hashtags.map(x => x._3.groupBy(identity).mapValues(_.length))

    val hashtag_counts = counts.reduce((a,b) => a |+| b)
    val output = hashtag_counts.toSeq.sortWith(_._2 > _._2)

    for(e <- output){ println(e._1 + ": " + e._2) }

    spark.stop()
  }
  def HTML2Txt(content: String) = {
    try {
      Jsoup.parse(content).body().text()
    }
    catch {
      case e: Exception => throw new IOException("Caught exception processing input row ", e)
    }
  }

  def getContent(record: WarcRecord):String = {
    val cLen = record.header.contentLength.toInt
    //val cStream = record.getPayload.getInputStreamComplete()
    val cStream = record.getPayload.getInputStream()
    val content = new java.io.ByteArrayOutputStream();

    val buf = new Array[Byte](cLen)

    var nRead = cStream.read(buf)
    while (nRead != -1) {
      content.write(buf, 0, nRead)
      nRead = cStream.read(buf)
    }

    cStream.close()

    val contentString = content.toString("UTF-8")
    return contentString
  }

  def getUsername(url : String):String = {
    //remove first part of URL
    val ending = url.substring(20)
    //split on slashes
    val split_ending = ending.split("/")
    //if username is preceded by @, remove it
    if(split_ending(0).charAt(0) == '@')
      return split_ending(0).substring(1)
    else
      return split_ending(0)
  }

}
```

Unfortunately, this app did not run on the provided cluster, but instead gave the following error message that I was not able to fix:

```
19/06/30 17:10:12 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
log4j:WARN No appenders could be found for logger (org.apache.spark.deploy.SparkSubmit$$anon$2).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.

```

However, the code did run fine in my Spark notebook, which can be found [here](https://github.com/rubigdata/cc-2019-DanielAnthes/blob/master/Twitter%20Analysis.snb.ipynb)

Below are the 10 most common hashtags along with their counts that were found in the sample:

```scala
(#per?nurkka??@teamperanurkka,20),
 (#success,14),
 (#entrepreneur,11),
 (#T20Japan,7),
 (#LloydsBankAcademy,5),
 (#unboxing,5),
 (#digitalskills,5),
 (#GoHoos,4),
 (#art,4),
 (#artwork,4)
```

## What I Learned

### How Big is Big Data

Even though the WARC files that were returned by my query to the CommonCrawl database were only a very small subset of the entire Crawl (573 files totalling ~ 1.9Gb) performing an analysis on the data became prohibitively slow very quickly. I did not expect that a dataset of less than 2GB would already be too large to be analyzed on my local machine. Even when working with a small sample taken from this dataset analysis was still slow at times. This was surprising to me and led to me apprechiating just how much information is stored in the CommonCrawl - the most recent crawl contains 220TiB of data, more than 100000 times as large as the sample I took.

### Unstructured Data is full of Surprises

Even a simple task such as parsing a URL to find usernames can be difficult if the exact format of the data is not known. Originally I thought it would be easy to correctly parse all URLs, but there were many unexpected edge cases that lead to parsing errors.

### Unstructured Data can lead to Unexpected Problems

As a follow up to parsing all usernames out of my Twitter sample I tried to extract the html contents. Even though this had worked before on a small test crawl I created myself using wget, I could not get it to work on the Twitter dataset extracted from the CommonCrawl at first. This may have been an easy to diagnose problem if I had only been working on a small amount of data in a simple program, but without a straightforward way to look at a single object in Spark for debugging, I spent quite some time trying to find out where the issue lies.

### Working with 'BigData' is difficult

Even though the programs presented in this blog post are very simple it took a long time to get them to work (and some parts still don't...). Most steps along the way took much longer than expected and problems occurred in unexpected places. It took a long time to even get data to work on out of the CommonCrawl and most of the time working on this assignment was spent trying to fix seemingly random exceptions that started appearing as soon as I started working on a larger sample from the CommonCrawl instead of a very small crawl of the BBC twitter account I created myself for testing.  
Further, I expected that it would be straightforward to create a standalone app out of my analysis in a spark notebook. This was not the case and I ended up spending quite some time on trying to compile my programs.
