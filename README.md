# Using Twitter Decahose with Cavium <!-- omit in toc -->

Go directly to Jupyter Notebook viewer version  
 https://nbviewer.jupyter.org/github/caocscar/twitter-decahose-pyspark/blob/master/twitterCavium.ipynb

## Table of Contents <!-- omit in toc -->
- [UM Hadoop Cavium Cluster](#um-hadoop-cavium-cluster)
  - [Setting Python Version](#setting-python-version)
- [PySpark Interactive Shell](#pyspark-interactive-shell)
  - [Exit Interactive Shell](#exit-interactive-shell)
- [Using Jupyter Notebook with PySpark](#using-jupyter-notebook-with-pyspark)
- [Example: Parsing JSON](#example-parsing-json)
  - [Read in twitter file](#read-in-twitter-file)
  - [Selecting Data](#selecting-data)
    - [Getting Nested Data](#getting-nested-data)
    - [Getting Nested Data II](#getting-nested-data-ii)
  - [Summary](#summary)
  - [Saving Data](#saving-data)
  - [Complete Script](#complete-script)
- [Example: Finding text in a Tweet](#example-finding-text-in-a-tweet)
- [Example: Filtering Tweets by Location](#example-filtering-tweets-by-location)
  - [Coordinates](#coordinates)
  - [Place](#place)
  - [Place Types](#place-types)
    - [Country](#country)
    - [Admin (US examples)](#admin-us-examples)
    - [City](#city)
    - [Neighborhood (US examples)](#neighborhood-us-examples)
    - [POI (US examples)](#poi-us-examples)

## UM Hadoop Cavium Cluster
Twitter data already resides in a directory on Cavium. Log in to Cavium to get started.

SSH to `cavium-thunderx.arc-ts.umich.edu` `Port 22` using a SSH client (e.g. PuTTY on Windows) and login using your Cavium account and two-factor authentication.

**Note:** ARC-TS has a [Getting Started with Hadoop User Guide](http://arc-ts.umich.edu/new-hadoop-user-guide/)

### Setting Python Version 
Change Python version for PySpark to Python 3.X (instead of default Python 2.7) 

```
export PYSPARK_PYTHON=/bin/python3  
export PYSPARK_DRIVER_PYTHON=/bin/python3
```

## PySpark Interactive Shell
The interactive shell is analogous to a python console. The following command starts up the interactive shell for PySpark with default settings in the `workshop` queue.  
`pyspark --master yarn --queue workshop`

The following line adds some custom settings.  The 'XXXX' should be a number between 4050 and 4099.  
`pyspark --master yarn --queue workshop --num-executors 500 --executor-memory 5g --conf spark.ui.port=XXXX`

**Note:** You might get a warning message that looks like `WARN Utils: Service 'SparkUI' could not bind on port 40XX. Attempting port 40YY.` This usually resolves itself after a few seconds. If not, try again at a later time.

The interactive shell does not start with a clean slate. It already has several objects defined for you. 
- `sc` is a SparkContext
- `sqlContext` is a SQLContext object
- `spark` is a SparkSession object

You can check this by typing the variable names.
```
sc
sqlContext
spark
```

### Exit Interactive Shell
Type `exit()` or press Ctrl-D

## Using Jupyter Notebook with PySpark
Currently, the Cavium configuration only supports Python 2.7 on Jupyter.

1. Open a command prompt/terminal in Windows/Mac. You should have putty in your PATH (for Windows).  Port 8889 is arbitrarily chosen.  
`putty.exe -ssh -L localhost:8889:localhost:8889 cavium-thunderx.arc-ts.umich.edu` (Windows)  
`ssh -L localhost:8889:localhost:8889 cavium-thunderx.arc-ts.umich.edu` (Mac/Linux)
2. This should open a ssh client for Cavium. Log in as usual.
3. From the Cavium terminal, type the following (replace XXXX with number between 4050 and 4099):
```
export PYSPARK_PYTHON=/bin/python3  # not functional code
export PYSPARK_DRIVER_PYTHON=jupyter  
export PYSPARK_DRIVER_PYTHON_OPTS='notebook --no-browser --port=8889'  
pyspark --master yarn --queue workshop --num-executors 500 --executor-memory 5g --conf spark.ui.port=XXXX
```
4. Copy/paste the URL (from your terminal where you launched jupyter notebook) into your browser. The URL should look something like this but with a different token.
http://localhost:8889/?token=745f8234f6d0cf3b362404ba32ec7026cb6e5ea7cc960856
5. You should be connected.

## Example: Parsing JSON
Generic PySpark data wrangling commands can be found at https://github.com/caocscar/workshops/blob/master/pyspark/pyspark.md

### Read in twitter file
The twitter data is stored in JSONLINES format and compressed using bz2. PySpark has a `sqlContext.read.json` function that can handle this for us (including the decompression).
```
import os
wdir = '/var/twitter/decahose/raw'
df = sqlContext.read.json(os.path.join(wdir,'decahose.2018-03-02.p2.bz2'))
```
This reads the JSONLINES data into a PySpark DataFrame. We can see the structure of the JSON data using the `printSchema` method.

`df.printSchema()`

The schema shows the "root-level" attributes as columns of the dataframe. Any nested data is squashed into arrays of values (no keys included).

**Reference**
 - PySpark JSON Files Guide https://spark.apache.org/docs/latest/sql-data-sources-json.html

 - Twitter Tweet Objects https://developer.twitter.com/en/docs/tweets/data-dictionary/overview/tweet-object.html

### Selecting Data
For example, if we wanted to see what the tweet text is and when it was created, we could do the following.
```
tweet = df.select('created_at','text')
tweet.printSchema()
tweet.show(5)
```
The output is truncated by default. We can override this using the truncate argument.

`tweet.show(5, truncate=False)`

#### Getting Nested Data
What if we wanted to get at data that was nested? Like in `user`.

```
user = df.select('user')
user.printSchema()
user.show(1, truncate=False)
```
This returns a single column `user` with the nested data in a list (technically a `struct`).

We can select nested data using the `.` notation.
```
names = df.select('user.name','user.screen_name')
names.printSchema()
names.show(5)
```
To expand ALL the data into individual columns, you can use the `.*` notation.
```
allcolumns = df.select('user.*')
allcolumns.printSchema()
allcolumns.show(4)
```

Some nested data is stored in an `array` instead of `struct`.
```
arr = df.select('entities.user_mentions.name')
arr.printSchema()
arr.show(5)
```
The data is stored in an `array` similar as before. We can use the `explode` function to extract the data from an `array`.
```
from pyspark.sql.functions import explode

arr2 = df.select(explode('entities.user_mentions.name'))
arr2.printSchema()
arr2.show(5)
```

If we wanted multiple columns under user_mentions, we'd be tempted to use multiple `explode` statements as so.

`df.select(explode('entities.user_mentions.name'), explode('entities.user_mentions.screen_name'))`

This generates an error: *Only one generator allowed per select clause but found 2:*

We can get around this by using `explode` on the top most key with an `alias` and then selecting the columns of interest.
```
mentions = df.select(explode('entities.user_mentions').alias('user_mentions'))
mentions.printSchema()
mentions2 = mentions.select('user_mentions.name','user_mentions.screen_name')
mentions2.show(5)
```

#### Getting Nested Data II
What if we wanted to get at data in a list? Like the indices in `user_mentions`.
```
idx = mentions.select('user_mentions.indices')
idx.printSchema()
idx.show(5)
```
The schema shows that the data is in an `array` type. For some reason, `explode` will put each element in its own row. Instead, we can use the `withColumn` method to index the list elements.
```
idx2 = idx.withColumn('first', idx['indices'][0]).withColumn('second', idx['indices'][1])
idx2.show(5)
```
Why the difference?  Because the underlying element is not a `struct` data type but a `long` instead.

### Summary
So if you access JSON data in Python like this:

`(tweet['created_at'], tweet['user']['name'], tweet['user']['screen_name'], tweet['text'])`

The equivalent of a PySpark Dataframe would be like this:

`df.select('created_at','user.name','user.screen_name','text')`

### Saving Data
Once you have constructed your PySpark DataFrame of interest, you should save it (append or overwrite) as a parquet file as so.
```
folder = 'twitterExtract'
df.write.mode('append').parquet(folder)
```
### Complete Script
Here is a sample script which combines everything we just covered. It extracts a four column DataFrame.
```
import os

wdir = '/var/twitter/decahose/raw'
file = 'decahose.2018-03-02.p2.bz2'
df = sqlContext.read.json(os.path.join(wdir,file))
six = df.select('created_at','user.name','user.screen_name','text','coordinates','place')
folder = 'twitterExtract'
six.write.mode('overwrite').parquet(folder)
```

## Example: Finding text in a Tweet 
Read in parquet file.
```
folder = 'twitterDemo'
df = sqlContext.read.parquet(folder)
```
Below are several ways to match text 
***

Exact match `==`
```
hello = df.filter(df.text == 'hello world')
hello.show(10)
```

`contains` method
```
food = df.filter(df['text'].contains(' food'))
food = food.select('text')
food.show(10, truncate=False)
```

`startswith` method
```
once = df.filter(df.text.startswith('Once'))
once = once.select('text')
once.show(10, truncate=False)
```

`endswith` method
```
ming = df.filter(df['text'].endswith('ming'))
ming = ming.select('text')
ming.show(10, truncate=False)
```

`like` method using SQL wildcards
```
mom = df.filter(df.text.like('%mom_'))
mom = mom.select('text')
mom.show(10, truncate=False)
```

regular expressions ([workshop material](https://github.com/caocscar/workshops/tree/master/regex))
```
regex = df.filter(df['text'].rlike('[ia ]king'))
regex = regex.select('text')
regex.show(10, truncate=False)
```

Applying more than one condition. When building DataFrame boolean expressions, use
- `&` for `and`
- `|` for `or`
- `~` for `not`  
```
resta = df.filter(df.text.contains('resta') & df.text.endswith('ing'))
resta = resta.select('text')
resta.show(10, truncate=False)
```

**Reference**: http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.Column

## Example: Filtering Tweets by Location

Read in parquet file.
```
folder = 'twitterDemo'
df = sqlContext.read.parquet(folder)
```
From the [Twitter Geo-Objects documentation](https://developer.twitter.com/en/docs/tweets/data-dictionary/overview/geo-objects):

> There are two "root-level" JSON objects used to  describe the location associated with a Tweet: `coordinates` and `place`.

> The `place` object is always present when a Tweet is geo-tagged, while the `coordinates` object is only present (non-null) when the Tweet is assigned an exact location. If an exact location is provided, the `coordinates` object will provide a [long, lat] array with the geographical coordinates, and a Twitter Place that corresponds to that location will be assigned.

### Coordinates
Select Tweets that have gps coordinates
```
coords = df.filter(df['coordinates'].isNotNull())
```

Construct a longitude and latitude column
```
coords = coords.withColumn('lng', coords['coordinates.coordinates'][0])
coords = coords.withColumn('lat', coords['coordinates.coordinates'][1])
coords.printSchema()
coords.show(5, truncate=False)
```

Apply a bounding box to tweets and count number of matching tweets
```
A2 = coords.filter(coords['lng'].between(-84,-83) & coords['lat'].between(42,43))
A2.show(5, truncate=False)
A2.count()
```
**Done!**

### Place
Search for places by name 

Create separate columns from `place` object
```
place = df.filter(df['place'].isNotNull())
place = place.select('place.country', 'place.country_code', 'place.place_type','place.name', 'place.full_name')
place.printSchema()
place.show(10, truncate=False)
```

Apply place filter
```
MI = place.filter(place['full_name'].contains(' MI'))
MI.show(10, truncate=False)
MI.count()
```
**Tip**: Refer to section ["Finding text in a Tweet"](#example-finding-text-in-a-tweet) for other search methods

### Place Types
There are five kinds of `place_type` in the twitter dataset in approximately descending geographic area:
1. country
2. admin
3. city
4. neighborhood
5. poi (point of interest)

Here's a breakdown of the relative frequency for this dataset (Note: Percentages have been added post-hoc and won't show up in the SQL result)
```
place.registerTempTable('Places')
place_type_ct = sqlContext.sql('SELECT place_type, COUNT(*) as ct FROM Places GROUP BY place_type ORDER BY ct DESC')
place_type_ct.show()
```
|place_type|count|pct|
|:---:|---:|---:|
|city|1738893|84.1%|
|admin|221170|10.7%|
|country|79811|3.9%|
|poi|24701|1.2%|
|neighborhood|3343|0.2%|

Here are some examples of each `place_type`:
#### Country
```
country = sqlContext.sql("SELECT * FROM Places WHERE place_type = 'country'")
country.show(5, truncate=False)
```
|country|country_code|place_type|name|full_name|
|---|:---:|:---:|---|---|
|Uzbekistan|UZ|country|Uzbekistan|Uzbekistan|
|Bosnia and Herzegovina|BA|country|Bosnia and Herzegovina|Bosnia and Herzegovina|
|United States|US|country|United States|United States|
|Ukraine|UA|country|Ukraine|Ukraine|
|República de Moçambique|MZ|country|República de Moçambique|República de Moçambique|

#### Admin (US examples)
```
admin = sqlContext.sql("SELECT * FROM Places WHERE place_type = 'admin' AND country_code = 'US'")
admin.show(10, truncate=False)
```
|country|country_code|place_type|name|full_name|
|---|:---:|:---:|---|---|
|United States|US|admin|Louisiana|Louisiana, USA|
|United States|US|admin|New York|New York, USA|
|United States|US|admin|California|California, USA|
|United States|US|admin|Michigan|Michigan, USA|
|United States|US|admin|South Carolina|South Carolina, USA|
|United States|US|admin|Virginia|Virginia, USA|
|United States|US|admin|South Dakota|South Dakota, USA|
|United States|US|admin|Louisiana|Louisiana, USA|
|United States|US|admin|Florida|Florida, USA|
|United States|US|admin|Indiana|Indiana, USA|

#### City
```
city = sqlContext.sql("SELECT * FROM Places WHERE place_type = 'city'")
city.show(5, truncate=False)
```
|country|country_code|place_type|name|full_name|
|---|:---:|:---:|---|---|
|Portugal|PT|city|Barcelos|Barcelos, Portugal|
|Brasil|BR|city|São Luís|São Luís, Brasil|
|Malaysia|MY|city|Petaling Jaya|Petaling Jaya, Selangor|
|Germany|DE|city|Illmensee|Illmensee, Deutschland|
|Ireland|IE|city|Kildare|Kildare, Ireland|
#### Neighborhood (US examples)
```
neighborhood = sqlContext.sql("SELECT * FROM Places WHERE place_type = 'neighborhood' AND country_code = 'US'")
neighborhood.show(10, truncate=False)
```
|country|country_code|place_type|name|full_name|
|---|:---:|:---:|---|---|
|United States|US|neighborhood|Duboce Triangle|Duboce Triangle, San Francisco|
|United States|US|neighborhood|Downtown|Downtown, Houston|
|United States|US|neighborhood|South Los Angeles|South Los Angeles, Los Angeles|
|United States|US|neighborhood|Cabbagetown|Cabbagetown, Atlanta|
|United States|US|neighborhood|Downtown|Downtown, Memphis|
|United States|US|neighborhood|Downtown|Downtown, Houston|
|United States|US|neighborhood|Hollywood|Hollywood, Los Angeles|
|United States|US|neighborhood|Clinton|Clinton, Manhattan|
|United States|US|neighborhood|Noe Valley|Noe Valley, San Francisco|
|United States|US|neighborhood|The Las Vegas Strip|The Las Vegas Strip, Paradise|
#### POI (US examples)
```
poi = sqlContext.sql("SELECT * FROM Places WHERE place_type = 'poi' AND country_code = 'US'")
poi.show(10, truncate=False)
```
|country|country_code|place_type|name|full_name|
|---|:---:|:---:|---|---|
|United States|US|poi|Bice Cucina Miami|Bice Cucina Miami|
|United States|US|poi|Ala Moana Beach Park|Ala Moana Beach Park|
|United States|US|poi|Los Angeles Convention Center|Los Angeles Convention Center|
|United States|US|poi|Cleveland Hopkins International Airport (CLE)|Cleveland Hopkins International Airport (CLE)|
|United States|US|poi|Indianapolis Marriott Downtown|Indianapolis Marriott Downtown|
|United States|US|poi|Round 1 - Coronado Center|Round 1 - Coronado Center|
|United States|US|poi|Golds Gym - Lake Mead|Golds Gym - Lake Mead|
|United States|US|poi|Lower Keys Medical Center|Lower Keys Medical Center|
|United States|US|poi|Mockingbird Vista|Mockingbird Vista|
|United States|US|poi|Starbucks|Starbucks|
