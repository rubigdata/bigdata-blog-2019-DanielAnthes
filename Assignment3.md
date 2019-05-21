# Assignment 3

In this blog post I describe my first experience with using Spark SQL. In the following I will describe the steps I took to analyze a dataset published by the city council of Nijmegen  [here](https://www.nijmegen.nl/opendata/BAG_ADRES.csv). To start off I will answer some of the questions posed in the notebook for this assignment and subsequently will look into the data further. In the process I will comment on how to use Spark to perform the analysis.

## Reading in a dataset

The data used is provided in .csv format. To import it as a dataset in Spark we use

```spark.read.format("csv").option("header", true).load("path/to/data")```

There are multiple ways to visualize the dataset. A few basic methods are listed below:

* `data.printSchema()` shows the structure of the dataset
* `data.show(n)` shows the first n entries in the dataset
* `data.describe()` provides statistics on the data, such as counts, mean, min, max values and standard deviations

These operations can be performed on Dataset objects, as created when importing data from a csv file in the way shown above.

## Performing operations on datasets

We can use a number of built in operations to work with the dataset that was created from the .csv file. All of these operations are part of Spark's 'Dataset API' (documentation [here](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset))
Alternatively, it is possible to work with the data using SQL queries. Both of these approaches lead to the same underlying operations being performed so for the outcome of the analysis it does not matter whether we use SQL queries or the dataset API.
To query the data using SQL in Spark a Spark session is needed:

```val spark = SparkSession.builder().appName("A3b-spark-df").getOrCreate()```

Then, this session can be used for sql queries:

```spark.sql("your query")```

## Artless quarters of Nijmegen

One of the questions in the assignment is to find all quarters of Nijmegen for which no entries in the art database exist.

After importing the data as described above we can first get all quarters of Nijmegen:

`val quarter_names = bagdata.select("WIJK_OMS").distinct().withColumnRenamed("WIJK_OMS", "quarter")`  
`quarter_names.show(10)`

These commands take the 'WIJK_OMS' column in the dataset and store all unique values in the variable quarter_names. The column 'WIJK_OMS' is also renamed into the more readable 'quarter'. The next line shows ten entries.

In the assignment spark notebook a dataframe has already been created containing each quarter together with the year in which its oldest artwork was created (kosquarter). To find all quarters that do not have any artwork in the database we can simply take the list of all quarters as created above and select from it all entries that are not in the kosquarter dataframe.

We will do this using an SQL query. However, to perform queries on the quarter_names data we first need to create a view:

`quarter_names.createOrReplaceTempView("quarter_names")`  

To list all views:

`spark.catalog.listTables.show(false)`

Now we can perform SQL queries on the two views:

`spark.sql("FROM quarter_names SELECT quarter WHERE quarter NOT IN (SELECT quarter FROM kosquarter)")`

This query results in a list of quarters with no associated art pieces:  
"Malvert"  
"Kwakkenberg"  
"Aldenhof"  
"Bijsterhuizen"  
"Oosterhout"  
"Grootstal"  
"Ressen"  
"Neerbosch-West"  
"Tolhuis"  
"Zwanenveld"  
"Haven- en industrieterrein"  
"Vogelzang"  
"Kerkenbos"  
"Brakkenstein"  
"'t Broek"  
"St. Anna"  
"Westkanaaldijk"  
"Staddijk"  
"Hatertse Hei"  
"Ooyse Schependom"  
"Groenewoud"  
"Heseveld"  
"'t Acker"  
"Lankforst"  
"Weezenhof"  


## Age of art pieces not associated with a quarter

Similarly, we can find out how old art pieces are that have no match in the address data base. To do so, we select the name and year of construction for entries in the 'kunst' dataset that do not have a matching name in the 'kosquarter' dataset.

`spark.sql("SELECT naam as name, bouwjaar as year from kunst WHERE naam NOT IN (SELECT naam FROM kosquarter)")`

Once we look at the results it becomes clear why the 'art pieces' do not appear in the list of artworks together with their locations. Most of the entries returned by this query do not contain names and dates at all, but instead what seems like pieces of descriptions or simply empty entries. It seems that this is an artifact of importing the data.

## Density of art per inhabitants in Nijmegen

To further explore the open data I set out to determine which quarter of Nijmegen has the highest density of artworks per inhabitant - the most "artsy" quarters of Nijmegen. To do so I added another dataset to the analysis (as recommended in the assignment notebook). The [dataset](http://www.nijmegen.nl/opendata/opendata_stadsgetallen.accdb) contains statistics about the population of the Netherlands. I started by reading in the dataset: (note that the dataset is distributed in the .accdb format and needs to be converted to .csv format first)

`val population_stats = spark.read.format("csv").option("header", false).load("/opt/docker/data/bigdata/tbl_OPENDATA.csv").cache()`  
`population_stats.printSchema()`  
`population_stats.show(10)`  

On first inspection the data seems quite messy. There are no column headers, some columns include multiple values and others contain different variable names. After looking at some rows of data it seems that the 4th column indicates what kind of information is stored in the row. We can list all of the possible values of that column using:

`population_stats.select("_c3").distinct()`  

"Bevolking"  
"Economie"  
"Sociaal-economisch profiel"  
"Woningmarkt"  

Since I am interested in the population numbers for the quarters of Nijmegen I selected only those entries from the dataset where _c3 is 'Bevolking' and create a new dataset that includes only data that may be relevant to my analysis - adding column headers in the process:

`val population = population_stats.withColumnRenamed("_c1", "count")
  .withColumnRenamed("_c2", "measure")
  .withColumnRenamed("_c6", "gender_age")
  .withColumnRenamed("_c8", "quarter")
  .where("_c3 == 'Bevolking'")
  .select("count", "measure", "gender_age", "quarter")`

This yields a new table with 4 columns:

* 'count' with what I think should be the number of inhabitants
* 'measure' the type of measurement (either absolute or estimated numbers)
* 'gender_age' a column containing an age range and gender
* 'quarter'

Now that all irrelevant information is removed the data still needs to be cleaned up to be usable. For example the counts are still saved as strings and the column gender_age contains two values that should best be split into two separate columns. This can be fixed by running the following command:

`val pop_cleaned = population.withColumn("gender", split(col("gender_age"), " ").getItem(0))
  .withColumn("age", split(col("gender_age"), " ").getItem(1))
  .drop(col("gender_age"))
  .withColumn("count", col("count").cast("float"))`

To inspect the cleaned dataset:

`pop_cleaned.describe()`

This query yields statistics about the dataset. Looking at the results shows that the dataset still has some problems. The last entry for 'quarter' is Zwolle - indicating that the data contains information that is not relevant to the analysis of inhabitants in Nijmegen only. Further it seems that the column gender contains an entry "woonwagens" and the last entry for age is "westerse".

Further problems are revealed when looking at the counts per quarter. To add up all counts of inhabitants per quarter I used the following SQL query:

pop_cleaned.createOrReplaceTempView("pop_stats")
spark.sql("SELECT SUM(count) AS count, quarter FROM pop_stats GROUP BY quarter")

According to this query 'Nijmegen-Midden' has 5311989 inhabitants and there are entries for Groningen, Oss and Eindhoven - clearly not relevant to the analysis.

I fixed the second problem first by revisting the quarter_names dataframe introduced earlier in this blog post. Since it contains all relevant quarters of Nijmegen it can be used to filter the new dataset:

`spark.sql("SELECT SUM(count),quarter FROM pop_stats WHERE quarter IN (SELECT quarter FROM quarter_names) GROUP BY quarter")`

To verify whether entries exist for all quarters listed in 'quarter_names' I decided to count the number of quarters in both datasets:

`spark.sql("SELECT COUNT(quarter) FROM quarter_names")`

There are 44 quarters in the quarter_names dataframe

`spark.sql("SELECT COUNT(DISTINCT quarter) from pop_stats WHERE quarter IN (SELECT quarter FROM quarter_names)")`

... and only 43 of them are represented in the population statistics.

To see which quarter is missing I ran the following query:

`spark.sql("SELECT quarter FROM quarter_names WHERE quarter NOT IN (SELECT DISTINCT quarter from pop_stats WHERE quarter IN (SELECT quarter FROM quarter_names))")`

Turns out there is a good reason for the missing quarter: The 44th quarter that was missing is called 'Haven- en industrieterrein' explaining why there is no population data.

To look into fixing the population counts I first created a new dataset only containing those quarters present in quarter_names:

`val pc_nijmegen = pop_cleaned.where("quarter IN (SELECT quarter FROM quarter_names)")`

Next I turned my attention to the other problem discovered: both the gender and the age columns contain values that seem to be artifacts. After listing all values for both columns with the two queries below I decided to try and get approximately correct data by restricting the values in the 'gender' column to 'man' and 'vrouw' and to restrict the values in the age column to those that start with a number.

`spark.sql("SELECT DISTINCT gender FROM pc_nij")`  
`spark.sql("SELECT DISTINCT age FROM pc_nij")`

To do so I used a spark's filter function on my dataframe:

`val pc_nij_cleaned = pc_nijmegen.filter($"age" rlike "[0-9].*").filter($"gender" isin ("man", "vrouw"))`

However, adding up all counts in this dataset still implies that Nijmegen has over 7 million inhabitants - so clearly there are still entries in the dataset that need to be cleaned up:

`pc_nij_cleaned.createOrReplaceTempView("inhabitants")`  
`spark.sql("SELECT SUM(count) FROM inhabitants")`

Running another query revealed that the data contained overlapping age ranges and therefore duplicate counts:

`spark.sql("SELECT count, gender, age FROM inhabitants WHERE quarter == 'Tolhuis' ORDER BY gender")`

The above query returns rows with age ranges such as "25-49" but als "25-29". There also exist multiple entries for the same age range.
