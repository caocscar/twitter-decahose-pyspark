# Using Twitter Decahose with Great Lakes <!-- omit in toc -->

This tutorial covers the process for obtaining access to twitter data, filtering to a subset of tweets based on multiple keywords, and conducting a simple analysis using Jupyter Notebook and PySpark on Great Lakes HPC.

## Table of Contents <!-- omit in toc -->
- [Prerequisites](#prerequisites)
  * [Principal Investigator (PI) Setup](#principal-investigator-pi-setup)
  * [Team Members' Setup](#team-members-setup)
  * [Connecting to Great Lakes OnDemand](#connecting-to-great-lakes-ondemand)
- [Data Acquisition](#data-acquisition)
  * [Example: Parsing JSON](#example-parsing-json)
  * [Read in twitter file](#read-in-twitter-file)
  * [Selecting Data](#selecting-data)
  * [Getting Nested Data](#getting-nested-data)
  * [Getting Nested Data II](#getting-nested-data-ii)
  * [Summary](#summary)
  * [Saving Data](#saving-data)
  * [Complete Script](#complete-script)
  * [Example: Finding text in a Tweet](#example-finding-text-in-a-tweet)
  * [Example: Filtering Tweets by Location](#example-filtering-tweets-by-location)
    + [Coordinates](#coordinates)
    + [Place](#place)
    + [Place Types](#place-types)
    + [Country](#country)
    + [Admin (US examples)](#admin--us-examples-)
    + [City](#city)
    + [Neighborhood (US examples)](#neighborhood--us-examples-)
    + [POI (US examples)](#poi-us-examples-)
  * [Example: Filtering Tweets by Language](#example-filtering-tweets-by-language)
- [Natural Language Toolkit and H2O Analysis](#natural-language-toolkit-and-h2o-analysis)

## Prerequisites

### Principal Investigator (PI) Setup
1. PIs must [Request a Twitter Decahose project](https://forms.office.com/Pages/ResponsePage.aspx?id=tHdu5iRX10SHIQbfFgRQzv7PMSYa-JRDlzQs8rgy5WJUMjdNSlNGUzgySDMyM0hOSUEzMFFHR1VSNS4u)
2. Sign up for the [Research Computing Package](https://arc.umich.edu/umrcp/) if you don't already have one, once approved add team members to Great Lakes Slurm account [ARC portal](https://portal.arc.umich.edu/)
3. Sign up for an [ARC login](https://arc.umich.edu/login-request)

### Team Members' Setup
1. When a PI submits a project request form and the project is approved, members identified on the form will receive a link to sign-up for access to the decahose data
2. Sign up for an [ARC login](https://arc.umich.edu/login-request)
3. Once a member has an ARC login individuals can be added by the PI on the [ARC portal](https://portal.arc.umich.edu/)

### Connecting to Great Lakes OnDemand
1. Navigate to [Great Lakes](https://greatlakes.arc-ts.umich.edu/)
2. Click on `My Interactive Sessions`
3. Choose `Jupyter + Spark Advanced`
4. Enter slurm account code. This is usually your PIs uniqname followed by a number and can be found on the [ARC portal](https://portal.arc.umich.edu/)
5. Enter number of hours: (e.g. 4), nodes: (e.g. 1), cores: (e.g. 32), memory: (e.g. 180gb)

## Data Acquisition

### Example: Parsing JSON
[Generic PySpark data wrangling commands](https://github.com/caocscar/workshops/blob/master/pyspark/pyspark.md)

### Read in twitter file from Turbo Drive
The Turbo Drive contains approximately 18 months of the most recent data. For historical data see the next section.

The twitter data is stored in JSONLINES format and compressed using bz2. PySpark has a `sqlContext.read.json` function that can handle this for us (including the decompression). You can pass a file, directory, or a combination using a wildcard at the end of the path `*`.
```python
import os
wdir = '/nfs/turbo/twitter-decahose/decahose/raw'
df = sqlContext.read.json(os.path.join(wdir,'decahose.2022-03-02.p2.bz2'))
```

### (Coming Soon) Read historical data from locker 
Historical data on Locker is only mounted to the login node so it is recommended to use Globus `umich#greatlakes` to transfer data from locker `/nfs/locker/twitter-decahose-locker/` to `/scratch/<root_account/`

### Print data structure

This reads the JSONLINES data into a PySpark DataFrame. We can see the structure of the JSON data using the `printSchema` method.

`df.printSchema()`

The schema shows the "root-level" attributes as columns of the dataframe. Any nested data is squashed into arrays of values (no keys included).

**Reference**
 - [PySpark JSON Files Guide](https://spark.apache.org/docs/latest/sql-data-sources-json.html)

 - [Twitter Tweet Objects](https://developer.twitter.com/en/docs/twitter-api/enterprise/data-dictionary/native-enriched-objects/tweet)

### Selecting Data
For example, if we wanted to see what the tweet text is and when it was created, we could do the following.
```python
tweet = df.select('created_at','extended_tweet.full_text') OR tweet = df.select('created_at','text') for data prior to November 8th, 2017
tweet.printSchema()
tweet.show(5)
```
The output is truncated by default. We can override this using the truncate argument.

`tweet.show(5, truncate=False)`

### Getting Nested Data
What if we wanted to get at data that was nested? Like in `user`.

```python
user = df.select('user')
user.printSchema()
user.show(1, truncate=False)
```
This returns a single column `user` with the nested data in a list (technically a `struct`).

We can select nested data using the `.` notation.
```python
names = df.select('user.name','user.screen_name')
names.printSchema()
names.show(5)
```
To expand ALL the data into individual columns, you can use the `.*` notation.
```python
allcolumns = df.select('user.*')
allcolumns.printSchema()
allcolumns.show(4)
```

Some nested data is stored in an `array` instead of `struct`.
```python
arr = df.select('entities.user_mentions.name')
arr.printSchema()
arr.show(5)
```
The data is stored in an `array` similar as before. We can use the `explode` function to extract the data from an `array`.
```python
from pyspark.sql.functions import explode

arr2 = df.select(explode('entities.user_mentions.name'))
arr2.printSchema()
arr2.show(5)
```

If we wanted multiple columns under user_mentions, we'd be tempted to use multiple `explode` statements as so.

`df.select(explode('entities.user_mentions.name'), explode('entities.user_mentions.screen_name'))`

This generates an error: *Only one generator allowed per select clause but found 2:*

We can get around this by using `explode` on the top most key with an `alias` and then selecting the columns of interest.
```python
mentions = df.select(explode('entities.user_mentions').alias('user_mentions'))
mentions.printSchema()
mentions2 = mentions.select('user_mentions.name','user_mentions.screen_name')
mentions2.show(5)
```

### Getting Nested Data II
What if we wanted to get at data in a list? Like the indices in `user_mentions`.
```python
idx = mentions.select('user_mentions.indices')
idx.printSchema()
idx.show(5)
```
The schema shows that the data is in an `array` type. For some reason, `explode` will put each element in its own row. Instead, we can use the `withColumn` method to index the list elements.
```python
idx2 = idx.withColumn('first', idx['indices'][0]).withColumn('second', idx['indices'][1])
idx2.show(5)
```
Why the difference?  Because the underlying element is not a `struct` data type but a `long` instead.

### Summary
So if you access JSON data in Python like this:

`(tweet['created_at'], tweet['user']['name'], tweet['user']['screen_name'], tweet['extended_tweet']['full_text'])`

The equivalent of a PySpark Dataframe would be like this:

`df.select('created_at','user.name','user.screen_name','extended_tweet.full_text')`

### Saving Data
Once you have constructed your PySpark DataFrame of interest, you should save it (append or overwrite) as a parquet file as so.
```python
folder = 'twitterExtract'
df.write.mode('append').parquet(folder)
```
### Complete Script
Here is a sample script which combines everything we just covered. It extracts a six column DataFrame.
```python
import os

wdir = '/var/twitter/decahose/raw'
file = 'decahose.2018-03-02.p2.bz2'
df = sqlContext.read.json(os.path.join(wdir,file))
six = df.select('created_at','user.name','user.screen_name','extended_tweet.full_text','coordinates','place')
folder = 'twitterExtract'
six.write.mode('overwrite').parquet(folder)
```

### Example: Finding text in a Tweet 
Read in parquet file.
```python
folder = 'twitterExtract'
df = sqlContext.read.parquet(folder)
```
Below are several ways to match text 
***

Exact match `==`
```python
hello = df.filter(df.extended_tweet.full_text == 'hello world')
hello.show(10)
```

`contains` method
```python
food = df.filter(df['extended_tweet']['full_text'].contains(' food'))
food = food.select('full_text')
food.show(10, truncate=False)
```

`startswith` method
```python
once = df.filter(df.extended_tweet.full_text.startswith('Once'))
once = once.select('full_text')
once.show(10, truncate=False)
```

`endswith` method
```python
ming = df.filter(df['extended_tweet']['full_text'].endswith('ming'))
ming = ming.select('full_text')
ming.show(10, truncate=False)
```

`like` method using SQL wildcards
```python
mom = df.filter(df.extended_tweet.full_text.like('%mom_'))
mom = mom.select('full_text')
mom.show(10, truncate=False)
```

regular expressions ([workshop material](https://github.com/caocscar/workshops/tree/master/regex))
```python
regex = df.filter(df['extended_tweet']['full_text'].rlike('[ia ]king'))
regex = regex.select('full_text')
regex.show(10, truncate=False)
```

Applying more than one condition. When building DataFrame boolean expressions, use
- `&` for `and`
- `|` for `or`
- `~` for `not`  
```python
resta = df.filter(df.extended_tweet.full_text.contains('resta') & df.extended_tweet.full_text.endswith('ing'))
resta = resta.select('full_text')
resta.show(10, truncate=False)
```

Using a list of keywords
```python
import pyspark.sql.functions as f

li=['ketchup', 'mustard']

tweets_filtered = df.filter(f.lower(df['extended_tweet.full_text']).rlike('|'.join(li)))
```

**Reference**: http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.Column

### Example: Filtering Tweets by Location

Read in parquet file.
```python
folder = 'twitterExtract'
df = sqlContext.read.parquet(folder)
```
From the [Twitter Geo-Objects documentation](https://developer.twitter.com/en/docs/tweets/data-dictionary/overview/geo-objects):

> There are two "root-level" JSON objects used to  describe the location associated with a Tweet: `coordinates` and `place`.

> The `place` object is always present when a Tweet is geo-tagged, while the `coordinates` object is only present (non-null) when the Tweet is assigned an exact location. If an exact location is provided, the `coordinates` object will provide a [long, lat] array with the geographical coordinates, and a Twitter Place that corresponds to that location will be assigned.

#### Coordinates
Select Tweets that have gps coordinates. In this sample dataset, **157,954 / 173,398,330 tweets (~0.1%)** have `coordinates`.
```python
coords = df.filter(df['coordinates'].isNotNull())
```

Construct a longitude and latitude column
```python
coords = coords.withColumn('lng', coords['coordinates.coordinates'][0])
coords = coords.withColumn('lat', coords['coordinates.coordinates'][1])
coords.printSchema()
coords.show(5, truncate=False)
```

Apply a bounding box to tweets and count number of matching tweets
```python
A2 = coords.filter(coords['lng'].between(-84,-83) & coords['lat'].between(42,43))
A2.show(5, truncate=False)
A2.count()
```

Suppose you have a different bounding box you want to apply to each row instead of a constant. Let's set up a fake dataset where we have a bounding box specified first.
```python
from pyspark.sql.functions import rand
A2 = A2.withColumn('bbox_x1', rand()-84.5)
A2 = A2.withColumn('bbox_x2', rand()-83.5)
A2 = A2.withColumn('bbox_y1', rand()+41.5)
A2 = A2.withColumn('bbox_y2', rand()+42.5)
```

Now we can apply a filter as before. But now, we use the `col` function to return a `Column` type instead of a constant.
```python
from pyspark.sql.functions import col
A2_bbox = A2.filter(coords['lng'].between(col('bbox_x1'),col('bbox_x2')) & coords['lat'].between(col('bbox_y1'),col('bbox_y2')))
A2_bbox.show(5)
A2_bbox.count()
```

**Done!**

#### Place
Search for places by name. In this sample dataset, **2,067,918 / 173,398,330 tweets (~1.2%)** have `place`.

Create separate columns from `place` object
```python
place = df.filter(df['place'].isNotNull())
place = place.select('place.country', 'place.country_code', 'place.place_type','place.name', 'place.full_name')
place.printSchema()
place.show(10, truncate=False)
```

Apply place filter
```python
MI = place.filter(place['full_name'].contains(' MI'))
MI.show(10, truncate=False)
MI.count()
```
**Tip**: Refer to section ["Finding text in a Tweet"](#example-finding-text-in-a-tweet) for other search methods

#### Place Types
There are five kinds of `place_type` in the twitter dataset in approximately descending geographic area:
1. country
2. admin
3. city
4. neighborhood
5. poi (point of interest)

Here's a breakdown of the relative frequency for this dataset. 
```python
import pyspark.sql.functions as f
from pyspark.sql.window import Window

place.registerTempTable('Places')
place_type_ct = sqlContext.sql('SELECT place_type, COUNT(*) as ct FROM Places GROUP BY place_type ORDER BY ct DESC')
place_type_ct = place_type_ct.withColumn('pct', f.format_number(f.lit(100) * f.col('ct') / f.sum('ct').over(Window.partitionBy()),1))
place_type_ct = place_type_ct.orderBy('ct', ascending=False)
place_type_ct.show()
```
|place_type|count|pct|
|:---:|---:|---:|
|city|1738893|84.1|
|admin|221170|10.7|
|country|79811|3.9|
|poi|24701|1.2|
|neighborhood|3343|0.2|

Here are some examples of each `place_type`:
#### Country
```python
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
```python
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
```python
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
```python
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
```python
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

### Example: Filtering Tweets by Language
Read in parquet file
```python
wdir = 'twitterExtract'
df = spark.read.parquet(os.path.join(wdir)
```

From the [Twitter documentation about language](https://developer.twitter.com/en/docs/twitter-api/v1/tweets/filter-realtime/guides/premium-operators):

> Matches Tweets that have been classified by Twitter as being of a particular language (if, and only if, the tweet has been classified). It is important to note that each Tweet is currently only classified as being of one language, so AND’ing together multiple languages will yield no results.

> **Note**: if no language classification can be made the provided result is ‘und’ (for undefined).

Let's look at the first few rows of tweets.
```python
tweets = df.select('lang','text')
tweets.show(20, truncate=False)
```
lang|text
---|---
en|One thing I love as much as traveling to see my favorite bands, is seeing my friends/mutuals travel to see their favorite bands. 🥰
en|RT @calumstruly: ashton: the truth luke: https://t.co/XbFOKBPd6B
en|Best me to JA!
ko|RT @BTSW_official: [#BTSWORLD_OST] "다시 널 찾을거야, 운명처럼💜" 드디어! #방탄소년단 이 열심히 부른 BTS WORLD OST 타이틀곡! &lt;Heartbeat&gt;가 나왔습니다! (👏🏻) 매니저님을 위한 특별한 선물…
ja|いやwwww逆に運良すぎかwwww三枚目wwww https://t.co/7WgmYTrFWu
en|RT @kookpics: cr. _FE_JK0901 - #JUNGKOOK #정국 @BTS_twt https://t.co/gFOMHUN5f2
en|RT @ughhhsierra: it’s been a couple months since i’ve felt like i’m home
tr|@gulsumm_c Of ne güzel hava attın esti buralar  skkdkd
ar|RT @nj1la: ضيفني+لآيكك+رتويت+سوي فولو من الرتويت واللآيكات. وآضمن لك آكثر من ٥٠٠ متابع في ساعة. يلا ضيفوا بعض. وتابعوني.🖤🖤 July 01, 2019 at…
und|RT @carluxokuster: https://t.co/7W3k6FrFK4
ar|RT @Jurgen3ziz: كانت مُجرد نظرة إلى الأرض لمُدة 17 ثانية ..كانت مُجرد درس قصير تم فهمه في ثواني ..كانت مُجرد تساؤل: إلى أين سنصل ياتُرى ؟…
es|Soñé que conocía a Tom Holland y me daba un besito y ahora estoy triste porque no pasó
tr|ölüşüm ama sanki böyle ölmemişim gibiyim
ja|【絶対1位】高橋あゆみのどんどん動画 現正廣播中！！https://t.co/xOic40JFch
th|@MESa131_ ขนาดเดินวนรอบเขาแล้วเรียกฟุคุซาวะซัง ฟุคุซาวะซังงงงง
ja|RT @BLUESOLVALOU: 『父にサビを歌われて剣を抜く娘』の詳細を調べてたら日が暮れたでござるの巻 https://t.co/azYUKq2BTx
pt|@theskindoctor13 @ANI @humasqureshi Huma mam real life Mai Laila vaale gunde yahi h
ja|RT @katsuse_m: 梅雨の雰囲気ばかりなタイムラインに、花火してるギャルのツイートが流れてきた。自分の知らないところで夏が始まってた。その次のツイートで知らないOLが「彼氏と別れた」とフォロワーに報告してた。いいねで回ってきてた。ちっともよくなさそうだった。自分の知ら…
ja|RT @aikanium: 祇園のカラーコーンの隠し方が私は好きで。カラーコーンなんてあったら確実にvisual pollutionになるのに、これだと逆にあった方がアクセントになってかわいいかも？と思えるくらいの見た目。そこまで持っていけるのすごい。 https://t.co…
in|RT @YourAverageOta3: Banger✊ https://t.co/eyJptcI31z

Let's look at the distribution of languages.
```python
languages = df.groupBy('lang').count().orderBy('count', ascending=False)
languages = languages.withColumn('pct', f.format_number(f.lit(100) * f.col('count') / f.sum('count').over(Window.partitionBy()),1))
languages.orderBy('count', ascending=False).show(20)
```                                                            
language|lang|count|pct
---|:---:|---:|---:
English|en|10858344|31.2
Japanese|ja|6475431|18.6
Spanish|es|2828876|8.1
undefined|und|2495456|7.2
Portuguese|pt|2356523|6.8
Arabic|ar|1994858|5.7
Thai|th|1639563|4.7
Korean|ko|1505357|4.3
Indonesian|in|987514|2.8
French|fr|765138|2.2
Turkish|tr|685058|2.0
Tagalog|tl|521548|1.5
Italian|it|198574|0.6
Russian|ru|193590|0.6
Hindi|hi|167066|0.5
German|de|142481|0.4
Urdu|ur|104673|0.3
Persian|fa|96038|0.3
Polish|pl|93276|0.3
Dutch|nl|78095|0.2

**Note**: I've tacked on the `language` column for clarification.

To filter out only spanish tweets, we can use the `filter` method.
```python
espanol = tweets.filter(df['lang'] == 'es')
espanol.show(10, truncate=False)
```
lang|text
---|---
es|Soñé que conocía a Tom Holland y me daba un besito y ahora estoy triste porque no pasó
es|@Sus1216 @Kaykas90 @NetflixES Graciassss
es|Que va xd
es|RT @pixelatedboat: Primary debate https://t.co/soyz8tiUft
es|RT @elcapitansur: @BMarmoldeLeon También Se mata con diálogos que le dan tiempo a la tiranía de seguir subyugando al venezolano, se mata co…
es|RT @hernanflash: @matiaseperezz Nada del otro mundo como para no superarlo 🤷‍♂️
es|@adnradiochile @SandraZeballos #EclipseSolar #CiudadanoADN #EclipseCiudadano #Radionautas Este dato de Radio Futur… https://t.co/z2EXUNNGKC
es|@INFOnews La prepaga es más cara que un alquiler si sigue así tendremos que evaluar si no conviene ir a vivir https://t.co/fb8hHAtVa4
es|@Pykare La verdas calles y plazas en casi todo el país estan un desastre no da ni para sentarte a tomat terere, los… https://t.co/uzEWM8sy2R
es|RT @dherranzba: Me traigo un gran proyecto competitivo entre manos, gracias a uno de los clubes competitivos más importantes en España. Es…

We can see that there was one language misclassification of a tweet in the fourth row. This should have been classified as English.


## Natural Language Toolkit and H2O Analysis
```python
%pip install h2o
%pip install h2o_pysparkling_2.1
```

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("SparklingWaterApp").getOrCreate()
```

Install and start pysparkling and the H2o cluster
```python
from pysparkling import *
hc = H2OContext.getOrCreate()
```

View columns
```python
df.columns
```

Start H2o
```python
import h2o

from h2o.estimators.word2vec import H2OWord2vecEstimator  
from h2o.estimators.gbm import H2OGradientBoostingEstimator  
from h2o.estimators.deeplearning import H2OAutoEncoderEstimator, H2ODeepLearningEstimator 
from pysparkling import *  
from nltk.corpus import stopwords
```

Tokenize words
```python
from pyspark.ml.feature import RegexTokenizer, StopWordsRemover
import pyspark.sql.functions as f
tokenizer = RegexTokenizer(inputCol='text', outputCol = 'tokenized_words', pattern="\\W+", minTokenLength = 3)
#filter_star_rating = filtered_data_year_category.filter(filtered_data_year_category.star_rating == stars)
tokenized_words = tokenizer.transform(df)
remover = StopWordsRemover(inputCol='tokenized_words', outputCol = 'word_tokens')
clean_tokens = remover.transform(tokenized_words)
word_counts = clean_tokens.withColumn('word', f.explode(f.col('word_tokens'))).groupBy('word').count().sort('count', ascending=False)
word_counts.show()
```

Start natural language toolkit and download the lexicon
```python
import nltk
nltk.download('vader_lexicon')
```

Calculate sample polarity score
```python
from nltk.sentiment.vader import SentimentIntensityAnalyzer
vds = SentimentIntensityAnalyzer()

text = 'I like turtles.'
vds.polarity_scores(text)
```


```python
df_pd = df.toPandas()

analyzer = SentimentIntensityAnalyzer()
df_pd['compound'] = [analyzer.polarity_scores(x)['compound'] for x in df_pd['full_text']]
df_pd['neg'] = [analyzer.polarity_scores(x)['neg'] for x in df_pd['full_text']]
df_pd['neu'] = [analyzer.polarity_scores(x)['neu'] for x in df_pd['full_text']]
df_pd['pos'] = [analyzer.polarity_scores(x)['pos'] for x in df_pd['full_text']]
df_pd.columns
```

View the data
```python
df_pd.head()
```

Write data to an excel spreadsheet
```python
import openpyxl
df_pd.to_excel(r'sentiment_data.xlsx', index = False)
```
