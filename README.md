# Processing Daily Weather Data

The NOAA aggregates world-wide, daily weather data and [exposes it](https://gis.ncdc.noaa.gov/geoportal/catalog/search/resource/details.page?id=gov.noaa.ncdc:C00861) for researchers on an [FTP server](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/). 

Here, we run through an example of how to download, process, and ingest data for a full year (2015). We also join the weather data with station metadata in a [PostgreSQL](https://www.postgresql.org/) database, making the values of some fields more human-readable in the process.

## Understanding the data, getting the data we need

The FTP server provides a [high-level README](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/readme.txt) that shows us where to get the data we need (all data from 2015), and points us to the metadata files we'll need to understand where the observations were recorded. 

To get a global picture of day-by-day weather data in 2015, with the (lat, long) of all observations, we'll need to download the following:

* The [2015 data](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/2015.csv.gz), which you can find in the `by_year` subdirectory
* The [metadata for weather stations](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/ghcnd-stations.txt), which includes the (lat, long) of the station

The high-level README above describes the fields in the station metadata. Separately, [this README](ftp://ftp.ncdc.noaa.gov/pub/data/ghcn/daily/by_year/readme.txt) describes the fields included in the daily-level 2015 data.

## Downloading the data

The rest of this guide will walk you through downloading and processing the data from the command line (from Mac OS X). Most of the code here will work on any Unix-like system (Linux, FreeBSD, etc.). The instructions for installing some of what you'll need (e.g. PostgreSQL) will vary based on your system.

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

* `cut -d, -f1 2015.csv` : the `cut` command takes a file as an argument, "cutting" specific columns for selective processing. Here, since we have a CSV, `-d,` tells `cut` to delimit columns by commas. We grab the first field (the field with the station identifier) by passing the `-f1` option
* `sort` : sorts the records of the column alphabetically (if we had numeric data, we could sort numerically. See `man sort` for more options)
* `uniq` : given sorted data, `uniq` selects and outputs only the unique records in the set
* `wc -l` : here, again, we use `wc` to count the number of lines in the data passed to it. Before, we passed the filename as an argument to the `wc` command, but like most UNIX utilities, `wc` will also accept data passed to it through a pipeline

For files of this size, **this is not the most efficient way to get the count of records we want**. But these commands are terse, and with certain data sets, it's much easier to use these utilities than it is to process the data in Python, or in a database.

## Processing the data for ingestion into PostgreSQL

The weather data in _2015.csv_ is provided to us in a well-structured CSV, so it's easy to use the command-line to work with. The _ghcnd-stations.txt_ data is a bit more messy:

* Columns are delimited by multiple spaces, but some field names (the name of the station) also contain spaces
* Certain missing data are recorded with custom values (e.g. missing elevation for the station is recorded as -999.99), but we want to correctly note those values as missing
* And more

Python provides better tools for processing data like this. Specifically, we'll use the [pandas](http://pandas.pydata.org/) library to do most of the heavy lifting.

We've created an [iPython notebook](Parse_Station_Metadata.ipynb) with all the code used to read and process the _ghcnd-stations.txt_ file. The data are stored in _station_metadata.csv_.
