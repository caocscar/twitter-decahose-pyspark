# twitter-decahose

## UM Hadoop Cavium Cluster
SSH to `cavium-thunderx.arc-ts.umich.edu` `Port 22` using a SSH client (e.g. PuTTY on Windows) and login using your Cavium account and two-factor authentication.

**Note:** ARC-TS has a [Getting Started with Hadoop User Guide](http://arc-ts.umich.edu/new-hadoop-user-guide/)

### Setting Python Version 
Change Python version for PySpark to Python 3.X (instead of default Python 2.7) 

`export PYSPARK_PYTHON=/bin/python3`  
`export PYSPARK_DRIVER_PYTHON=/bin/python3`  
`export SPARK_YARN_USER_ENV=PYTHONHASHSEED=0`

# PySpark Interactive Shell
The interactive shell is analogous to a python console. The following command starts up the interactive shell for PySpark with default settings in the `workshop` queue.  
`pyspark --master yarn --queue workshop`

The following line adds some custom settings.  The 'XXXX' should be a number between 4050 and 4099.  
`pyspark --master yarn --queue workshop --num-executors 900 --executor-memory 5g --executor-cores 4 --conf spark.ui.port=XXXX`

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

## Example Code
```
import os
wdir = '/var/twitter/decahose/raw'
df = sqlContext.read.json(os.path.join(wdir,'decahose.2018-03-02.p2.bz2'))

df.printSchema()
```
Assumes the json file is in jsonlines format (decahose data is in this format).
The schema shows that `.read.json` puts the top level keys into columns of the dataframe. Any nested json is put into an array of values (no keys included).

PySpark JSON Files Guide
https://spark.apache.org/docs/latest/sql-data-sources-json.html

```
from pyspark.sql.functions import explode

abc = df.select('user')
abc.show(1, truncate=False)
abc.printSchema()

fgh = df.select('user.*') # selects all child of userfgh.colum
fgh.printSchema()

ijk = df.select('user.created_at','user.name','user.screen_name')
ijk.printSchema()
ijk.show(5)
ijk

aa = df.select('entities.user_mentions.name')
aa.printSchema()
aa.show(5)

bb = df.select(explode('entities.user_mentions.name'))
bb.printSchema()
bb.show(5)

bb = df.select(explode('entities.user_mentions.name'), explode('entities.user_mentions.screen_name')
'Only one generator allowed per select clause but found 2:'

cc = df.select(explode('entities.user_mentions').alias('user_mentions'))
cc.printSchema()
dd = cc.select('user_mentions.*')
dd.show(5)
```
Suppose I wanted a four column dataframe.
In Python, I would access the JSON file as follows and save into a tuple:
`(tweet['created_at'], tweet['user']['name'], tweet['user']['screen_name'], tweet['text'])`

In PySpark
`df.select('created_at','user.name','user.screen_name','text')`

