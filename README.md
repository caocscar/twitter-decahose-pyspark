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

```
import os
wdir = '/var/twitter/decahose/raw'
df = sqlContext.read.json(os.path.join(wdir,'decahose.2018-03-02.p1.bz2'))

df.printSchema()
```
Assumes the json file is in jsonlines format (decahose data is in this format).
The schema shows that `.read.json` puts the top level keys into columns of the dataframe. Any nested json is put into an array of values (no keys included).

PySpark JSON Files Guide
https://spark.apache.org/docs/latest/sql-data-sources-json.html

```
abc = df.select('user')
abc.show(1, truncate=False)

from pyspark.sql.functions import udf, explode, array
from pyspark.sql.types import Row

get_date = udf(lambda x: x[1], 'string')
newdf = abc.withColumn('date', get_date('user'))
newdf.show(5)

user_description = udf(lambda x: x[4], 'string')
newdf = abc.withColumn('other', user_description('user'))
newdf.show(5)

get_column = udf(lambda x: x[17], 'string')
getc = udf(lambda x: x[10], 'string')
newdf = abc.withColumn('other', get_column('user')).withColumn('geo', getc('user'))
newdf.show(10)

get_column = udf(lambda x: [x[17], x[10] 'string')
getc = udf(lambda x: x[10], 'string')
newdf = abc.withColumn('other', get_column('user')).withColumn('geo', getc('user'))
newdf.show(10)

def extract(x):
  return [x[17], x[10]]

schema = StructType([
    StructField("Out1", StringType(), True),
    StructField("Out2", StringType(), True)])

get_column = udf(extract)
newdf = abc.withColumn('other', array(get_column('user')))
newdf.show(10)

```
user list index|column
---|---
1|created_at
4|description
5|favourites_count
7|followers_count
9|friends_count
10|geo_enabled
11|id
12|id_str
14|lang
15|listed_count
16|location
17|name
32|screen_name
33|statuses_count

