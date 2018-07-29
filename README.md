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

Follow this [link](https://dev.maxmind.com/geoip/legacy/csv/#Integer_IPv4_Representation) for a description

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

And create the table below

```sql
CREATE TABLE geo_ip_country_whois (
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
LOAD DATA INPATH '/user/admin/workshop/GeoIPCountryWhois.csv' OVERWRITE INTO TABLE geo_ip_country_whois;
```

**Exercise:** Explore the table

```sql
SELECT * FROM geo_ip_country_whois;
```

Create a table to access the semi-structure logs format

```sql
CREATE TABLE raw_logs (
log STRING
);
```

Then laod the data

```sql
LOAD DATA INPATH '/user/admin/workshop/access.log' OVERWRITE INTO TABLE raw_logs;
```

**Exercise:** Explore the table

```sql
SELECT * FROM raw_logs LIMIT 10;
```

You should notice that web logs need to be parsed following a [specific pattern](https://httpd.apache.org/docs/1.3/logs.html#combined)

In order to increase the read performance we are going to extract the valuable information and insert the results in an optimized table

SELECT
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1) ip,
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",1) as bigint) * 16777216 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",2) as bigint) * 65536 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",3) as bigint) * 256 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",4) as bigint) as ip_to_int,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 4) time,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 5) request
FROM raw_logs
limit 1;

CREATE TABLE web_logs (
ip VARCHAR(15),
ip_to_int INT,
time STRING,
request STRING
)
STORED AS ORC
TBLPROPERTIES ("orc.compress"="SNAPPY");

INSERT OVERWRITE TABLE web_logs
SELECT
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1) ip,
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",1) as bigint) * 16777216 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",2) as bigint) * 65536 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",3) as bigint) * 256 +
  cast(regexp_extract(regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",4) as bigint) as ip_to_int,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 4) time,
  regexp_extract(log, "([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?", 5) request
FROM raw_logs


SELECT count(*) FROM web_logs;

https://dev.maxmind.com/geoip/legacy/csv/#Integer_IPv4_Representation


select ip, country_name
from (
select
regexp_extract(log, '^(?:([^ ]*),?){1}', 1) ip,
cast(regexp_extract(regexp_extract(log, '^(?:([^ ]*),?){1}', 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",1) as bigint) * 16777216 +
cast(regexp_extract(regexp_extract(log, '^(?:([^ ]*),?){1}', 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",2) as bigint) * 65536 +
cast(regexp_extract(regexp_extract(log, '^(?:([^ ]*),?){1}', 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",3) as bigint) * 256 +
cast(regexp_extract(regexp_extract(log, '^(?:([^ ]*),?){1}', 1),"(\\d+)\\.(\\d+)\\.(\\d+)\\.(\\d+)",4) as bigint) as ip_to_int
FROM raw_logs
LIMIT 10
) accesses
JOIN geo_ip_country_whois
WHERE ip_to_int between start_ip_int and end_ip_int;

SELECT ip, country_name
FROM (
SELECT * FROM web_logs LIMIT 10
) accesses
LEFT JOIN geo_ip_country_whois
WHERE ip_to_int between start_ip_int and end_ip_int;


























