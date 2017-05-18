
# OpenStreetMap Data Case Study

### Author: Jianru Shi 
### Last edit: April 2017
### Map area: San Francisco Bay Area

## Introduction 

This report is one of projects for the Data Analyst Nano Degree at Udacity. 

In this project, I choose the San Francisco Bay Area in [OpenStreetMap](https://www.openstreetmap.org) and use data munging techniques to assess the quality of the data for validity, accuracy, completeness, consistency and uniformity. I will also show my approach to clean the OpenStreetMap data programmatically. At the end I will conduct some SQL quries to give some overviews about the SF Bay Area. 

![map area](map_area.png)

The outline of the this report is: 

* Introduction
* Data Sampling
* Data Auditing 
* Data Cleanning 
* Create Database and Tables
* Data Exploring with SQL
* Additional Ideas



```python
# Useful packages
import xml.etree.cElementTree as ET
import csv
import re
from collections import defaultdict
import codecs
import cerberus
import schema
import sqlite3 
import pandas as pd
```

## Data Sampling

The data is downloaded from [OpenStreetMap](https://www.openstreetmap.org). The full dataset is 3.03GB. I first use [sample_data](funcs/sample_func.py) function to take subsets of the data. I use a small subset (about 3MB) to test my code and then use an intermedian subset of data (about 300MB) to do the data auditing. The sampling process takes about **10 minutes** since the original file is large. The function returns two new osm files: [sample.osm](funcs/sample.osm) and [test.osm](funcs/test.osm).  



## Data Auditing

To audit the data I use the function [audit_address](funcs/audit_func.py) to check if there is any problem with data associated with the address, for example the street name, city and county name, and the postcode.

By running the test data in auditing functions I found following problems: 

* Inconsistency of street types (Street vs. ST, Avenue vs. AVE)


* Typoes and invailid street types ('Avenies', '88', 'CA-88')


* Invalid postcode (6digits postcode, postcodes are not in CA)


* Using full address as postcode (41 Westside Blvd, Hollister, CA 95023)


* Inconsistency of city names ('Vallejo' vs. ''Vallejo, California', 'Oakland' vs. 'oakland')


* Inconsistency of county names ('Santa Clara County' vs. 'Santa Clara')


* Inconsistency of state names ('CA', 'Ca' vs. 'California')


* Incorrect state names ('WA', 'AZ')



## Data Cleanning 

After auditing the test data file, I found some problems as stated above. In this sections I use the [process_map](funcs/process_func.py) to clean problematical data. The main function process_map converts the processed .osm file to 5 .csv files and will be used to build the SQL database for further analysis. The csv files are: 


* nodes.csv
* nodes_tags.csv
* ways.csv
* ways_nodes.csv
* ways_tags.csv

With calling some [helper functions](funcs/helper_func.py) in the main function, the address information in the dataset will be revised by:


* clean postcode: remove prefix 'CA' from some data or extract the 5 digits postcode from the full address
* clean street: change abbrivated street type to full street suffix 
* clean state: change abbrivated state names to "CA"




## Create Database and Tables


In this section, I first created a database named 'sfbay.db', and import the five csv files into the database. I created tables using the to_sql function in pandas package. The full code is in [create_db.py](funcs/create_db.py)



```python
# create database sfbay.db and tables from csv files. 

conn=sqlite3.connect('sfbay.db')
cur = conn.cursor() 
cur.execute("CREATE TABLE nodes ( id INTEGER PRIMARY KEY NOT NULL, lat REAL, lon REAL,\
    user TEXT, uid INTEGER, version INTEGER, changeset INTEGER, timestamp TEXT )")
conn.commit()
node_df = pd.read_csv('nodes.csv', dtype=object)
node_df.to_sql('nodes', conn, if_exists='append', index=False)
```

## Data Exploration with SQL

### Database overview- files sizes


```python
# Database overview 
# list file sizes in the data folder.  
# ref http://stackoverflow.com/questions/10565435/iterating-through-directories-and-checking-the-files-size

import os
cwd = os.getcwd()
for root, dirs, files in os.walk(cwd+'/OSM_data', topdown=False):
    for name in files:
        f = os.path.join(root, name)
        print (name.ljust(20), round(os.path.getsize(f)/1000000, 2), 'MB')
```

    .DS_Store            0.01 MB
    nodes.csv            1182.59 MB
    nodes_tags.csv       27.62 MB
    sfbay.db             1625.4 MB
    SFBay.osm            3031.44 MB
    ways.csv             92.49 MB
    ways_nodes.csv       394.77 MB
    ways_tags.csv        174.52 MB


### Database overview - Numbers of nodes and ways

There are 14149895 nodes and 1557085 ways. 


```python
query='''select count(id) from nodes; '''

result=cur.execute(query)
for row in result:
    print (row)
```

    (14149895,)



```python
query='''select count(id) from ways; '''

result=cur.execute(query)
for row in result:
    print (row)
```

    (1557085,)


### Database overview - Top 10 contributors


```python
# Top 10 contributor and the number of nodes they craeted

query='select user, count(*) from nodes group by user order by count(*) desc limit 10;'
for row in cur.execute(query):
    print (row)
```

    ('nmixter', 1582191)
    ('andygol', 1167899)
    ('ediyes', 1019023)
    ('Eureka gold', 709818)
    ('Luis36995', 706698)
    ('woodpeck_fixbot', 636960)
    ('RichRico', 547427)
    ('dannykath', 532098)
    ('mk408', 403888)
    ('Rub21', 391532)


### Database overview - Count of distinct users 

There are 5480 distinct users.


```python
query='select count(distinct(user)) from \
(select user from nodes union all select user from ways);'

result=cur.execute(query)
for row in result:
    print (row)
```

    (5480,)


### Database overview - Count of shops

There are 9907 nodes tags have 'shop' as key. 


```python
query='''select count(*) from nodes_tags where key='shop';'''

result=cur.execute(query)
for row in result:
    print (row)
```

    (9907,)



### How many Starbucks store in Bay Area? 

There are 355 Starbucks stores in Bay Area. As a fan I would say 'Good job Starbucks :)'



```python
query='''select count (*) from nodes_tags where upper(value)='STARBUCKS';'''

result=cur.execute(query)
for row in result:
    print (row)
```

    (355,)


### Most popular cuisines

The most popular cuisines in Bay Area is Mexican food, Chinese and Pizza are also on the top of the list. 


```python
query='''SELECT nodes_tags.value, COUNT(*) as num
        FROM nodes_tags
            JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
            ON nodes_tags.id=i.id
        WHERE nodes_tags.key='cuisine'
        GROUP BY nodes_tags.value
        ORDER BY num DESC
        LIMIT 10;'''

result=cur.execute(query)
for row in result:
    print (row)
```

    ('mexican', 394)
    ('chinese', 301)
    ('pizza', 293)
    ('japanese', 229)
    ('italian', 217)
    ('american', 184)
    ('thai', 177)
    ('vietnamese', 144)
    ('indian', 125)
    ('burger', 102)


## Additional Ideas

Openstreetmap is maintained by thousands of users from all over the world. It’s impossible to be perfect. For example, from auditing on addresses we found different formats, typos, and incorrect informations. Although there are suggested standards on openstreetmap.org, contributors may neglect it or prefer use their own formats. 

Another problem is the incompleteness of tag information. In this project, I put lots effort to clean the data and import the data to the database using the suggested schema. The main purpose to do so is to extract tags information and put them into a easier access form. However, when I use the database to query ‘how many Starbucks stores in San Francisco?’ , I got a number of 15, which is way less than I expected. So I went to the openstreetmap.org, and checked the number. It turned out to has more than 40 stores in San Francisco. The reason is the tags with city name were not added to the nodes. 

### How many Starbucks store in San Francisco?  


```python
query='''select count (*) 
         from (SELECT * 
               from (select * from ways_tags union all select * from nodes_tags)
               where id in (select id 
                            from (select * from ways_tags union all select * from nodes_tags)
                            where upper(value)='STARBUCKS')) 
         where value like "%San Francisco%"
                        
      ;'''


result=cur.execute(query)
for row in result:
    print (row)
```

    (15,)

