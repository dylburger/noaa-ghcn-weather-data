# Processing Daily Weather Data

The NOAA (National Oceanic and Atmospheric Adminstration) aggregates world-wide, daily weather data and [exposes it](https://gis.ncdc.noaa.gov/geoportal/catalog/search/resource/details.page?id=gov.noaa.ncdc:C00861) for researchers on an [FTP server](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/). 

Here, I run through an example of how to download, process, and ingest data for a full year (2015). I also join the weather data with station metadata in a [PostgreSQL](https://www.postgresql.org/) database.

## Audience

You're learning data science techniques and want to familiarize yourself with some of the basic tools for fetching, processing, and ingesting data. I'll review simple Unix command line tools, the pandas data analysis Python library, Jupyter notebooks, and PostgreSQL.

## Understanding the data, getting the data we need

The FTP server provides a [high-level README](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/readme.txt) that shows us where to get the data we need (all data from 2015), and points us to the metadata files we'll need to understand where the observations were recorded. 

To get a global picture of day-by-day weather data in 2015, with the (lat, long) of all observations, we'll need to download the following:

* The [2015 data](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/2015.csv.gz), which you can find in the `by_year` subdirectory.
* The [metadata for weather stations](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/ghcnd-stations.txt), which includes the (lat, long) of the station.

The high-level README above describes the fields in the station metadata. Separately, [this README](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/readme.txt) and [this README](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/ghcn-daily-by_year-format.rtf) describe the fields included in the daily-level 2015 data.

## Downloading the data

The rest of this guide will walk you through downloading and processing the data from the command line (on Mac OS X). Most of the code here will work on any Unix-like system (Linux, FreeBSD, etc.). The instructions for installing some of what you'll need (e.g. PostgreSQL) will vary based on your system.

You can download the data from the FTP server with cURL:

    curl -O ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/2015.csv.gz
    curl -O ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/ghcnd-stations.txt

`curl -O` will save the file with the filename as it reads on the server (e.g. _2015.csv.gz_).

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
    weather=# grant select on all tables in schema weather to weather_read_only;
    GRANT

Record the password for the weather\_read\_only user somewhere secure for later use.

## Processing the data for ingestion into PostgreSQL

Both the weather data and station metadata require a bit of processing before ingesting them into PostgreSQL, namely:

* The date of measurement needs to be read into the database as a date, not a string.
* We only want to ingest a subset of the measurements, keeping precipitation, snowfall, and temperature readings, but removing more esoteric measures.
* The _ghcnd-stations.txt_ file delimits columns by multiple spaces, which can be difficult to parse correctly. Missing data is recorded with custom values (e.g. missing elevation is recorded as -999.9), but we want to correctly note those values as missing.

Python provides better tools than Unix utilities for processing data like this. Specifically, we'll use the [pandas](http://pandas.pydata.org/) library to do most of the heavy lifting.

You can find the code in [this Jupyter notebook](Ingest_Data_Into_PostgreSQL.ipynb). All the requirements for running this code are in the _requirements.txt_ file. It's recommended you create a new Python [virtual environment](http://docs.python-guide.org/en/latest/dev/virtualenvs/), and run

    pip install -r requirements.txt

to install the necessary modules. Then open the notebook:

    jupyter notebook Ingest_Data_Into_PostgreSQL.ipynb

and run the code from there.

At this point, you can start writing SQL queries in Postgres to ask interesting questions. You can even play around more with the code in the Jupyter notebook, slicing and dicing the data with Python. But if you want to enable normal users to derive insights, you'll need other tools.

## Configuring Superset

[Superset](https://github.com/airbnb/superset) is a general "ad hoc analysis" tool that enables an average user to ask complex questions of a dataset without writing a line of code. Typically, users might do this type of analysis in Excel, or Tableau, but Superset has benefits those tools don't:

* It's free and open source.
* Its data modeling layer centralizes the definitions for measures and dimensions. Metric are defined in one place, with one formula. When individuals keep this logic in Excel, they might define metrics inconsistently. This causes confusion for the business.
* Superset effectively combines the best of tools like Tableau and Excel. Tableau enables you to create beautiful visualizations from data in centralized databases, but fails to give you the rich power of a pivot table. Excel lets you ask and answer more abstract business questions, but lacks powerful visualizations, and the results are typically saved locally, not shared with the whole business. Superset provides a central platform for data exploration, enabling people to ask general questions, visualize the results, and share that with the business in an easy way.
* In addition to the simpler, business-friendly interface for asking questions, Superset also provides a [SQL IDE](http://airbnb.io/superset/sqllab.html) for running more complex SQL queries, making it easy to turn the results into a visualization or dashboard.

To setup a test Superset instance, see the [installation instructions](http://airbnb.io/superset/installation.html) and [tutorial](http://airbnb.io/superset/tutorial.html).
