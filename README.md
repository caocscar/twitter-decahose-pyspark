# Using Twitter Decahose with Cavium
Twitter data already resides in a directory on Cavium. Log in to Cavium to get started.

## UM Hadoop Cavium Cluster
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

## Exit Interactive Shell
Type `exit()` or press Ctrl-D

# Example Code
Generic PySpark data wrangling commands can be found at https://github.com/caocscar/workshops/blob/master/pyspark/pyspark.md

## Read in twitter file
The twitter data is stored in JSONLINES format and compressed using bz2. PySpark has a `sqlContext.read.json` function that can handle this for us (including the decompression).
```
import os
wdir = '/var/twitter/decahose/raw'
df = sqlContext.read.json(os.path.join(wdir,'decahose.2018-03-02.p2.bz2'))
```
This reads the JSONLINES data into a PySpark DataFrame. We can see the structure of the JSON data using the `printSchema` method.

`df.printSchema()`

The schema shows the "root-level" attributes as columns of the dataframe. Any nested data is squashed into arrays of values (no keys included).

#### Reference
 - PySpark JSON Files Guide https://spark.apache.org/docs/latest/sql-data-sources-json.html

 - Twitter Tweet Objects https://developer.twitter.com/en/docs/tweets/data-dictionary/overview/tweet-object.html

## Selecting Data
For example, if we wanted to see what the tweet text is and when it was created, we could do the following.
```
tweet = df.select('created_at','text')
tweet.printSchema()
tweet.show(5)
```
The output is truncated by default. We can override this using the truncate argument.

`tweet.show(5, truncate=False)`

### Getting Nested Data
What if we wanted to get at data that was nested? Like in `user`.

```
user = df.select('user')
abc.printSchema()
abc.show(1, truncate=False)
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

### Getting Nested Data II
What if we wanted to get at data in a list? Like the indices in `user_mentions`.
```
idx = mentions.select('user_mentions.indices')
idx.printSchema()
idx.show(5)
```
The schema shows that the data is in an `array` type. For some reason, `explode` will put each element in its own row. Instead, we can use the `withColumn` method to index the list elements.
```
idx2 = idx.withColumn('first', idx['indices'][0]).withColumn('second', idx['indices'][1])
```
Why the difference?  Because the underlying element is not a `struct` data type but a `long` instead.

## Summary
So if you access JSON data in Python like this:

`(tweet['created_at'], tweet['user']['name'], tweet['user']['screen_name'], tweet['text'])`

The equivalent of a PySpark Dataframe would be like this:
`df.select('created_at','user.name','user.screen_name','text')`

## Saving Data
Once you have constructed your PySpark DataFrame of interest, you should save it (append or overwrite) as a parquet file as so.
```
folder = 'twitterExtract'
df.write.mode('append').parquet(folder)
```
## Complete Script
Here is a sample script which combines everything we just covered. It extracts a four column DataFrame.
```
import os
from pyspark.sql.functions import explode

wdir = '/var/twitter/decahose/raw'
file = 'decahose.2018-03-02.p2.bz2'
df = sqlContext.read.json(os.path.join(wdir,file))
four = df.select('created_at','user.name','user.screen_name','text')
folder = 'twitterDemo'
four.write.mode('overwrite').parquet(folder)
```

## Example: Finding text in a Tweet
Read in parquet file.
```
folder = 'twitterDemo'
df = sqlContext.read.parquet(folder)
```
Below are several ways to match text
---

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
regex = df.filter(df['text'].rlike('[i ]king'))
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