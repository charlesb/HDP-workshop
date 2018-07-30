# HDP Labs

Content

* [Lab 1 - Accessing the sandbox](#accessing-the-sandbox)
* [Lab 2 - Load the sample datasets to HDFS](#load-the-sample-datasets-to-hdfs)
* [Lab 3 - Create Hive tables](#create-hive-tables)
* [Lab 4 - Analyze the datasets using SQL](#analyze-the-datasets-using-sql)

## Accessing the sandbox

Open a web browser and go to the following url

```http://<provided_url>:8080/```

Username: admin
Password: admin

![Image of Amabari Login Page](images/ambari_login_page.png)

Once logged in, start all services.

It can take up to 6 minutes...

![Image of Amabari Start Services](images/ambari_start_services.png)

## Load the sample datasets to HDFS

Download [GeoIPCountryCSV.zip](https://github.com/charlesb/HDP-workshop/raw/master/samples/GeoIPCountryCSV.zip) and [WebLogs.zip](https://github.com/charlesb/HDP-workshop/raw/master/samples/WebLogs.zip) from the samples folder

Follow this [link](https://dev.maxmind.com/geoip/legacy/csv/#GeoIP_Legacy_Country_CSV_Database_Fields) for a description of the GeoIP database.

Different formats can also be [downloaded](http://geolite.maxmind.com/download/geoip/database/GeoIPCountryCSV.zip)

Unzip the 2 files and go to Ambari's **Files View**

![Image of Amabari files view](images/ambari_files_view_1.png)

Then browse to /user/admin and create a new folder workshop.

Unzip GeoIPCountryCSV.zip and WebLogs.zip and upload the content to the new folder.

![Image of Amabari files view 2](images/ambari_files_view_2.png)

**Exercise:** Explore the content on both files on HDFS

## Create Hive tables

Go to Ambaris's **Hive View 2.0**

![Image of Amabari hive view](images/ambari_hive_view.png)

First create a new database

```sql
CREATE DATABASE workshop;
```

And create the table below

```sql
CREATE TABLE workshop.geo_ip_country_whois_csv (
start_ip_address STRING,
end_ip_address STRING,
start_ip_int INT,
end_ip_int INT,
country_code VARCHAR(2),
country_name VARCHAR(50)
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
   "separatorChar" = ",",
   "quoteChar"     = "\""
);
```
Then import the data from HDFS into Hive

```sql
LOAD DATA INPATH '/user/admin/workshop/GeoIPCountryWhois.csv' OVERWRITE INTO TABLE workshop.geo_ip_country_whois_csv;
```

Create an optimized table

```sql
CREATE TABLE workshop.geo_ip_country_whois (
start_ip_address STRING,
end_ip_address STRING,
start_ip_int INT,
end_ip_int INT,
country_code VARCHAR(2),
country_name VARCHAR(50)
)
STORED AS ORC
TBLPROPERTIES ("orc.compress"="SNAPPY");
```

Import the CSV data

```sql
INSERT INTO workshop.geo_ip_country_whois SELECT * FROM workshop.geo_ip_country_whois_csv;
```

**Exercise:** Explore the table

```sql
SELECT * FROM geo_ip_country_whois;
```

Create a table to access the semi-structured logs format

```sql
CREATE TABLE workshop.raw_logs (
log STRING
);
```

Then laod the data

```sql
LOAD DATA INPATH '/user/admin/workshop/access.log' OVERWRITE INTO TABLE workshop.raw_logs;
```

**Exercise:** Explore the table

```sql
SELECT * FROM workshop.raw_logs LIMIT 10;
```

Here is an example of Apache web server log:

```109.169.248.247 - - [12/Dec/2015:18:25:11 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"```

Notice that web logs need to be parsed following a [specific pattern](https://httpd.apache.org/docs/1.3/logs.html#combined)

Here is an example of a regular expression to extract each field:

```(\S+) (\S+) (\S+) \[([^\]]+)\] "([A-Z]+) ([^ "]+)? HTTP\/[0-9.]+" ([0-9]{3}) ([0-9]+|-) "([^"]*)" "([^"]*)" "(\S+)"```

Click [here](https://regexr.com/3t58q) for regex description.

## Analyze the datasets using SQL

In order to increase the read performance we are going to extract the valuable information and insert the results in an optimized table


First we create a target table

```sql
CREATE TABLE workshop.web_logs (
ip VARCHAR(15),
ip_to_int INT,
time STRING,
request STRING
)
STORED AS ORC
TBLPROPERTIES ("orc.compress"="SNAPPY");
```
We have added a new column ```ip_to_int``` to calculate the [integer representation of an IP (v4)](https://dev.maxmind.com/geoip/legacy/csv/#Integer_IPv4_Representation). It will be useful later to find out the origin of the connections.

The formula is:

```
first byte x 256^3 +
second byte x 256^2 +
third byte x 256^1 +
last byte
```

As an example, the integer representation of IP 10.0.5.9 is 167773449 (10×256^3 + 0×256^2 + 5×256^1 + 9)

Nowadays it's more commom to use [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) for the IP range representation.

Then extract valuable pieces of information from raw data map them to previously created table definition.

```sql
INSERT OVERWRITE TABLE workshop.web_logs
SELECT
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1) ip,
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",1) as bigint) * 16777216 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",2) as bigint) * 65536 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",3) as bigint) * 256 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",4) as bigint) as ip_to_int,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 4) time,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 5) request
FROM workshop.raw_logs
```

**Exercise:** How many logs do we have in the table?

We should have 10k logs!

**Exercise:** Retrieve the number of connections per origin

*tip:* Limit the connections to 100 as the cluster is a small sandbox

```sql
SELECT country_name, count(ip) as connections
FROM (
SELECT * FROM workshop.web_logs limit 100
) accesses, workshop.geo_ip_country_whois
WHERE ip_to_int between start_ip_int and end_ip_int
GROUP BY country_name
ORDER BY connections DESC;
```

**Exercise:** Explain the query using Ambari UI

## TODO

Create a Hive UDF to use CIDR notation instead

https://github.com/maxmind/geoip2-csv-converter


























