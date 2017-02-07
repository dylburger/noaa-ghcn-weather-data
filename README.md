# Processing Daily Weather Data

The NOAA (National Oceanic and Atmospheric Adminstration) aggregates world-wide, daily weather data and [exposes it](https://gis.ncdc.noaa.gov/geoportal/catalog/search/resource/details.page?id=gov.noaa.ncdc:C00861) for researchers on an [FTP server](http://bit.ly/2hKP5Kw). They refer to the network of weather stations as the Global Historical Climatology Network.

Below, I run through how to:

* Download the 2015 weather data and metadata from the NOAA FTP server
* Examine the data using Unix command line utilities
* Process the data using the Pandas data analysis Python library in a Jupyter notebook
* Ingest the data into a PostgreSQL database using Pandas
* Denormalize the data in PostgreSQL, preparing the data for analysis in Superset

Much of the processing and analysis of the data could be done with a single tool, e.g. Pandas. The goal here is to highlight the diversity of tools data scientists can use to manipulate and analyze this dataset.

## Audience

You're learning data science techniques and want to familiarize yourself with some of the basic tools for fetching, processing, and ingesting data. If you're unfamiliar with any of the above steps or tools, this guide is for you!

## Understanding the data, getting the data we need

The FTP server provides a [high-level README](http://bit.ly/2hKOKry) that shows us where to get the data we need (all data from 2015), and points us to the metadata files we'll need to understand where the observations were recorded.

To get a global picture of day-by-day weather data in 2015, with the (lat, long) of all observations, we'll need to download the following:

* The [2015 data](http://bit.ly/2j0CEMO), which you can find in the `by_year` subdirectory.
* The [metadata for weather stations](http://bit.ly/2j0LkTb), which includes the (lat, long) of the station.

The high-level README above describes the fields in the station metadata. Separately, [this README](http://bit.ly/2ivSmMU) and [this README](http://bit.ly/2iBswJL) describe the fields included in the daily-level 2015 data.

## Downloading the data

The rest of this guide will walk you through downloading and processing the data from the command line (on Mac OS X). Most of the code here will work on any Unix-like system (Linux, FreeBSD, etc.). The instructions for installing some of what you'll need (e.g. PostgreSQL) will vary based on your system.

You can download the data from the FTP server with cURL:

    curl -O ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/2015.csv.gz
    curl -O ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/ghcnd-stations.txt

`curl -O` will save the file with the filename as it reads on the server (e.g. _2015.csv.gz_).

After downloading the data, you'll want to `gunzip` it:

    gunzip 2015.csv.gz

## Examining the data

We saw roughly 34 million weather observations in 2015:

    $ wc -l 2015.csv
     34326749 2015.csv

We can use `wc` to count the number of records in the station metadata file, too, but that will list the count of stations that have recorded weather _for all time_. To get the count of stations that recorded data _in 2015_, we can [pipe](http://unix.stackexchange.com/questions/30759/whats-a-good-example-of-piping-commands-together) a handful of commands together:

    $ cut -d, -f1 2015.csv | sort | uniq | wc -l
       39912

Let's examine this step-by-step:

* `cut -d, -f1 2015.csv` : the `cut` command takes a file as an argument, "cutting" specific columns for selective processing. Here, since we have a CSV, `-d,` tells `cut` to delimit columns by commas. We grab the first field (the field with the station identifier) by passing the `-f1` option.
* `sort` : sorts the records of the column alphabetically (if we had numeric data, we could sort numerically. See `man sort` for more options).
* `uniq` : given sorted data, `uniq` selects and outputs only the unique records in the set.
* `wc -l` : here, again, we use `wc` to count the number of lines in the data passed to it. Before, we passed the filename as an argument to the `wc` command, but like most Unix utilities, `wc` will also accept data passed to it through a pipeline.

For files of this size, **this is not the most efficient way to get the count of records we want**. But these commands are terse, and for certain data sets, it's much easier to use these utilities than it is to process the data in Python, or in a database.

## Setting up the PostgreSQL database

Ingesting these data into a database will enable us to ask and answer basic questions of the data, and explore the data with tools like [Superset](https://github.com/airbnb/superset), which we discuss more below.

How you install PostgreSQL will vary depending on your system. Please consult the PostgreSQL documentation, and Google. Here's the set of steps I used to install PostgreSQL on Mac OS X with [Homebrew](http://brew.sh/), creating a weather database, schema, and read-only user to safely access the database from other tools:

    # Install PostgreSQL and related tools
    brew install postgres 

    # Start the Postgres server
    pg_ctl -D /usr/local/var/postgres start

    # Initialize the DB, enable your user to login to Postgres by default
    initdb /usr/local/var/postgres

    # Create the weather database
    createdb weather

    # Login
    $ psql -d weather
    psql (9.6.1)
    Type "help" for help.

    # Even though you'll only be running Postgres locally, choose a strong, random password here
    weather=# create user weather_read_only with password 'pass';
    CREATE ROLE
    weather=# create schema weather;
    CREATE SCHEMA

Record the password for the weather\_read\_only user somewhere secure for later use.

## Loading the data into PostgreSQL

Both the weather data and station metadata require a bit of processing before ingesting them into PostgreSQL, namely:

* The date of measurement needs to be read into the database as a date, not a string.
* We only want to ingest a subset of the measurements, keeping precipitation, snowfall, and temperature readings, but removing more esoteric measures.
* The _ghcnd-stations.txt_ file delimits columns by multiple spaces, which can be difficult to parse correctly. Missing data is recorded with custom values (e.g. missing elevation is recorded as -999.9), but we want to correctly note those values as missing.
* The weather types in the data file are difficult for humans to interpret (a new user might not know what 'PRCP' refers to). We want to attach a human-readable description to these measurements.

Python provides better tools than Unix utilities for processing data like this. Specifically, we'll use the [Pandas](http://pandas.pydata.org/) library to do most of the heavy lifting.

You can find the code in [this Jupyter notebook](Ingest_Data_Into_PostgreSQL.ipynb). All the requirements for running this code are in the _requirements.txt_ file. It's recommended you create a new Python [virtual environment](http://docs.python-guide.org/en/latest/dev/virtualenvs/), and run

    pip install -r requirements.txt

to install the necessary modules. Then open the notebook:

    jupyter notebook Ingest_Data_Into_PostgreSQL.ipynb

and run the code from there.

## Denormalizing the data and metadata, creating database indexes, separating data for specific weather types as views

To ask meaningful questions, we'll want to join data from all three of our data tables. If we join these tables consistently, it might make sense to [denormalize](https://en.wikipedia.org/wiki/Denormalization) our table design, keeping the data and metadata in a _single table_. Moreover, some tools - like Superset - cannot perform database joins, and so require that any data we want to analyze at the same time be in a single table.

Specifically, our new, "denormalized" table might have the following fields:

    station_identifier
    measurement_date
    measurement_type
    weather_description
    measurement_flag
    latitude
    longitude
    elevation

To create this unified table from our three existing tables, we can issue this query from `psql`:

    CREATE TABLE weather_data_denormalized AS 
        SELECT wd.station_identifier, 
               wd.measurement_date, 
               wd.measurement_type, 
               wt.weather_description, 
               wd.measurement_flag, 
               sm.latitude, 
               sm.longitude, 
               sm.elevation 
        FROM weather_data wd 
        JOIN station_metadata sm 
            ON wd.station_identifier = sm.station_id 
        JOIN weather_types wt 
            ON wd.measurement_type = wt.weather_type;

We now have data in a single table, but we can still improve the performance of our queries. Let's create some [database indexes](https://www.postgresql.org/docs/9.1/static/sql-createindex.html) on some of the fields in this table. Indexes will help PostgreSQL optimize our queries by querying only the set of records with a field that holds a specific value. For instance, let's say we only want to review the data recorded on 2015-01-01. We'd attach a filter to our query like so:

    WHERE measurement_date = '2015-01-01'

Without an index, PostgreSQL has to scan all the records of our table, _then_ filter on the ones tied to that date, returning the result. _With_ an index, Postgres will know exactly where to find the records we're interested in, immediately returning the set we want **without having to execute a full table scan**. Let's review how to create an index on the `measurement_date` field and see how it speeds up performance.

Without the index, here's how quickly we can retrieve the count of records with data on 2015-01-01:

    weather=# \timing
    Timing is on.
    weather=# SELECT COUNT(*) FROM weather_data_denormalized WHERE measurement_date = '2015-01-01';
    count
    -------
    73355
    (1 row)

    Time: 2463.825 ms

This takes roughly 2.5 seconds. Now, let's create an index:

    weather=# CREATE INDEX date_index ON weather_data_denormalized (measurement_date);
    CREATE INDEX

... and see how quickly we can retrieve the same count as above:
    
    weather=# SELECT COUNT(*) FROM weather_data_denormalized WHERE measurement_date = '2015-01-01';
    count
    -------
    73355
    (1 row)

    Time: 22.933 ms
    
It's magic!

On what fields should we create indexes? **If you'd filter on the field in a `WHERE` clause, you should create an index on the field**. Here, we'll likely be filtering on _measurement\_date_, _measurement\_type_, _weather\_description_, _measurement\_flag_, and _elevation_. Let's create indexes for all of the remaining fields:

    weather=# CREATE INDEX type_index ON weather_data_denormalized (measurement_type);
    CREATE INDEX
    weather=# CREATE INDEX description_index ON weather_data_denormalized (weather_description);
    CREATE INDEX
    weather=# CREATE INDEX flag_index ON weather_data_denormalized (measurement_flag);
    CREATE INDEX
    weather=# CREATE INDEX elevation_index ON weather_data_denormalized (elevation);
    CREATE INDEX

This will take some time to complete.

It's possible we might want to logically separate rainfall, snow, and temperate data, since the measurements are on different scales, and we're likely to ask different questions of each. We can use [database views](https://www.postgresql.org/docs/9.2/static/sql-createview.html) to separate these data. This gives the illusion that these data live in separate tables, where queries against the views really all run against the weather\_data\_denormalized_ table. Let's review how to create and query these views:

    weather=# CREATE VIEW precipitation_data AS SELECT * FROM weather_data_denormalized WHERE measurement_type = 'PRCP';
    CREATE VIEW
    weather=# SELECT COUNT(*) FROM precipitation_data;
    count
    ----------
    10192164

Finally, after we've loaded and created the new tables and views, we'll need to grant the weather\_read\_only user permission to execute SELECT statements on the data:

    weather=# GRANT SELECT ON ALL TABLES IN SCHEMA public TO weather_read_only
    GRANT

At this point, you can start writing SQL queries to ask interesting questions. You can even play around more with the code in the Jupyter notebook, slicing and dicing the data with Python. But if you want to enable "normal" users (people who don't know how to write code) to derive insights, you'll need other tools.

## Configuring Superset

[Superset](https://github.com/airbnb/superset) is a general "ad hoc analysis" tool that enables an average user to ask complex questions of a dataset without writing a line of code. Typically, users might do this type of analysis in Excel, or Tableau, but Superset has benefits those tools don't:

* It's free and open source.
* Its data modeling layer centralizes the definitions for measures and dimensions. Metric are defined in one place, with one formula. When individuals keep this logic in Excel, they might define metrics inconsistently. This causes confusion for the business.
* Superset effectively combines the best of tools like Tableau and Excel. Tableau enables you to create beautiful visualizations from data in centralized databases, but fails to give you the rich power of a pivot table. Excel lets you ask and answer more abstract business questions, but lacks powerful visualizations, and the results are typically saved locally, not shared with the whole business. Superset provides a central platform for data exploration, enabling people to ask general questions, visualize the results, and share that with the business in an easy way.
* In addition to the simpler, business-friendly interface for asking questions, Superset also provides a [SQL IDE](http://airbnb.io/superset/sqllab.html) for running more complex SQL queries, making it easy to turn the results into a visualization or dashboard.

Data scientists also benefit from ad hoc analysis tools in two key ways:

* Tools like pandas and SQL help us ask complex questions. The code also gets complicated, and hard to debug. Analysis errors increase, and answering questions starts to take a long time. Abstracting this code as reusable measures and dimensions in an ad hoc analysis tool reduces error and shortens the analysis process.
* If the business is asking you questions you can use SQL to answer, you should consider an ad hoc analysis tool. The benefits of "democratizing" data clearly outweigh the risks if implemented correctly. Domain experts ask the best questions, and you want to enable them with simple tools to do it. The more questions others answer themselves, the more time you have to apply more rigorous techniques to larger problems.

It's also important to note that Superset is immature, and actively being changed and developed. Other tools provide similar interfaces to business users, or friendly SQL IDEs from which you can create dashboards and visualizations. Please see [this comparison of tools](https://docs.google.com/spreadsheets/d/1j9zX0xF9QB79HMCCrM3PKyMR8qQmZ2IpMa_iKLaY_MA/) to help assess whether Superset is ideal for you.

To setup a test Superset instance, see the [installation instructions](http://airbnb.io/superset/installation.html) and [tutorial](http://airbnb.io/superset/tutorial.html).

## Sources

Menne, Matthew J., Imke Durre, Bryant Korzeniewski, Shelley McNeal, Kristy Thomas, Xungang Yin, Steven Anthony, Ron Ray, Russell S. Vose, Byron E.Gleason, and Tamara G. Houston (2012): Global Historical Climatology Network - Daily (GHCN-Daily), Version 3. 2015 Daily Data. NOAA National Climatic Data Center. doi:10.7289/V5D21VHZ, Accessed on 2016-12-30.
