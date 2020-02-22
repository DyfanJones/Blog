---
title: The next package release into AWS Athena
author: ~
date: '2020-02-22'
slug: the-next-package-release-into-aws-athena
categories: [RBloggers]
tags:
  - R
  - Athena
  - Boto3
  - paws
  - Python
description: ''
thumbnail: '/post/2020-02-22-the-next-package-release-into-aws-athena_files/r-athena-new.png'
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](https://feeds.feedburner.com/RBloggers)

`RAthena 1.7.1` and `noctua 1.5.1` package versions have now been released to the CRAN. They both bring along several improvements with the connection to AWS Athena noticeably the performance speed and several creature comforts.

These packages have both been designed to reflect one another,even down to how they connect to `AWS Athena`. This means that all features going forward will exist in both packages. I will refer to these packages as one as they basically work the same.

# Performance improvements:

Initially the packages mainly utilised `AWS Athena` SQL queries to achieve all the necessary requirements to create the `DBI` functionality. This worked reasonably well however the package would always send a SQL query to `AWS Athena` which intern would have to lift a flat file from `AWS S3` to before returning the final result to `R`. This means the performance of the packages would always be limited and fairly slow compared to other data base back ends.

![](/post/2020-02-22-the-next-package-release-into-aws-athena_files/r-athena-old.png)

The biggest change is to adopt more functionality of the SDKs (software development kit) into AWS. The key component that has been adopted is `AWS Glue`. `AWS Glue` contains all of `AWS Athena` table DDL's. This means instead of going to `AWS Athena` for this information `AWS Glue` can be used instead.

![](/post/2020-02-22-the-next-package-release-into-aws-athena_files/r-athena-new.png)

By utilising `AWS Glue`, table meta data for example column names, column types, schema hierarchy etc... Can easily be retrieved at a fraction of the time it would of taken to query `AWS Athena`. For example, the `DBI` function `dbListTables` would send a query to `AWS Athena` to retrieve all the tables listed in all `AWS Athena's` schemas this would of table over 3 seconds. Now it retrieves the same data in under 0.5 of a second.

## dplyr

Now that `AWS Glue` is used to collect metadata around a table in `AWS Athena`, a performance in `dplyr::tbl` can be done. I would like to say thanks to @OssiLehtinen for developing the initial implementation as this improvement would of gone over looked.

`dplyr::tbl` has too key methods when creating the initial object. The first is called SQL identifiers and this is the method that benefits from the new `AWS Glue` functionality. To use SQL identifiers is fairly straight forward.

```
library(DBI)
library(dplyr)
library(RAthena) #Or library(noctua)

con = dbConnect(athena())

dbWriteTable(con, "iris", iris)

ident_iris = tbl(con, "iris")
```

`dplyr` can identify the `iris` table within the connected schema. When a user, uses the SQL identifier method in tbl `AWS Glue` is called to retrieve all the meta data from `dplyr`. This has an increase in performance from 3.66 to 0.29 seconds. The second method is called SQL sub query. This sadly won't benefit from the new feature and will run in slower at 3.66 seconds.

```
subquery_iris = tbl(con, sql("select * from iris"))
```

Due to this I recommend to use the SQL identifier method as much as possible when using `dplyr's` interface.

# Creature Comforts

## AWS Athena Metadata

Due to feature requests the packages now return more meta data around each query sent to `AWS Athena`. By default the basic level of meta data returns in the amount of data that was scanned by `AWS Athena`. This is formatted into a more readable format depending on the amount of data that was scanned.

```
library(DBI)
library(RAthena) #Or library(noctua)

con = dbConnect(athena())

dbWriteTable(con, "iris", iris)

dbGetQuery(con, "select * from iris")
```

```
Info: (Data scanned: 860 Bytes)
     sepal_length sepal_width petal_length petal_width   species
  1:          5.1         3.5          1.4         0.2    setosa
  2:          4.9         3.0          1.4         0.2    setosa
  3:          4.7         3.2          1.3         0.2    setosa
  4:          4.6         3.1          1.5         0.2    setosa
  5:          5.0         3.6          1.4         0.2    setosa
 ---                                                            
146:          6.7         3.0          5.2         2.3 virginica
147:          6.3         2.5          5.0         1.9 virginica
148:          6.5         3.0          5.2         2.0 virginica
149:          6.2         3.4          5.4         2.3 virginica
150:          5.9         3.0          5.1         1.8 virginica
```
However if you set the new parameter `statistics` to `TRUE` then all the metadata around that query is printed out like so.

```
dbGetQuery(con, "select * from iris", statistics = TRUE)
```

```
$EngineExecutionTimeInMillis
[1] 1568

$DataScannedInBytes
[1] 860

$DataManifestLocation
character(0)

$TotalExecutionTimeInMillis
[1] 1794

$QueryQueueTimeInMillis
[1] 209

$QueryPlanningTimeInMillis
[1] 877

$ServiceProcessingTimeInMillis
[1] 17

Info: (Data scanned: 860 Bytes)
     sepal_length sepal_width petal_length petal_width   species
  1:          5.1         3.5          1.4         0.2    setosa
  2:          4.9         3.0          1.4         0.2    setosa
  3:          4.7         3.2          1.3         0.2    setosa
  4:          4.6         3.1          1.5         0.2    setosa
  5:          5.0         3.6          1.4         0.2    setosa
 ---                                                            
146:          6.7         3.0          5.2         2.3 virginica
147:          6.3         2.5          5.0         1.9 virginica
148:          6.5         3.0          5.2         2.0 virginica
149:          6.2         3.4          5.4         2.3 virginica
150:          5.9         3.0          5.1         1.8 virginica
```

This can also be retreived by using `dbStatistics`:

```
res = dbExecute(con, "select * from iris")

# return query statistic
query_stats = dbStatistics(res)

# return query results
dbFetch(res)

# Free all resources
dbClearResult(res)
```

## `RJDBC` inspired function

I have to give full credit to `RJDBC` for inspiring me to create this function. `DBI` has got a nice function called `dbListTables` that will list all the tables that are in `AWS Athena`, however it won't return what schema it is related to. To over come this `RJDBC` has a lovely function called `dbGetTables`. This function returns all the tables from `AWS Athena` as well, but this time it returns them in a data.frame. The advantage of this is that it can attach more information around what schema that table was from and what table type is. With the new integration into `AWS Glue` this can be return back quickly as well.

```
dbGetTables(con)
```

```
      Schema             TableName      TableType
 1:  default             df_bigint EXTERNAL_TABLE
 2:  default                  iris EXTERNAL_TABLE
 3:  default               mtcars2 EXTERNAL_TABLE
 4:  default         nyc_taxi_2018 EXTERNAL_TABLE
```

This just makes it a little bit easier when working in different IDE's for example `Jupyter`.

## Backend option changes

This is not really a creature comfort but it still fairly interesting and useful. Both packages are highlight dependent on `data.table` to read data into `R`. This is down to the amazing speed `data.table` offers when reading files into `R`. However a new package with equally impressive read speeds has come onto the scene [`vroom`](https://github.com/r-lib/vroom). As `vroom` has been designed to only read data into `R` similarily to `readr`, `data.table` is still used for all of the heavy lifting. However if a user wishes to use `vroom` as the file parser an `*_options` function has been created to enable this:

```
nocuta_options(file_parser = c("data.table", "vroom"))

# Or 

RAthena__options(file_parser = c("data.table", "vroom"))
```

By setting the file_parser to `vroom` then the backend will change to allow `vroom's` file parser to be used instead of `data.table`. 

If you aren't sure why you would opt in using `vroom` over `data.table`. `vroom` boasts a whopping 1.40GB/sec throughput

> *Stats Taken from vroom's github readme* 

package |	version |	time (sec) |	speedup |	throughput
---|---|---|---|---
vroom |	1.1.0 |	1.14 |	58.44 |	1.40 GB/sec
data.table |	1.12.8 |	11.88 |	5.62 |	134.13 MB/sec
readr |	1.3.1 |	29.02 |	2.30 |	54.92 MB/sec
read.delim |	3.6.2 |	66.74 |	1.00 |	23.88 MB/sec

## RStudio Interface!

Due to the nature of `AWS Glue` being used to retrieve metadata for `AWS Athena` quickly, it has now been possible to add the interface into RStudio's connection tab. When a connection is established:

```
library(DBI)
library(RAthena) #Or library(noctua)

con = dbConnect(athena())
```

The connection icon will as follows.

![](/post/2020-02-22-the-next-package-release-into-aws-athena_files/rstudio-con-tab.png)

A key note is that the AWS region you are connecting to will be reflected in the connection (highlighted above in the red square). This is to help users that are able to connect to multiple different `AWS Athena` over different regions.

Once you have connected, `AWS Athena` schema hierarchy will be displayed. In my example chase you can see some of the test tables I have created when testing these packages.

![](/post/2020-02-22-the-next-package-release-into-aws-athena_files/rstudio-con-schema.png)

For more information around RStudio's connection tab please check out [RStudio preview connections](https://blog.rstudio.com/2017/08/16/rstudio-preview-connections/).

# Summary

To sum up, `Rathena` and `noctua` latest versions have been released to cran with all the new goodies they bring. As these packages are based on AWS SDK's they are highly customisable. Meaning that features can easily be added to improve life when connecting to `AWS Athena`. So please raise any feature requests / bug issues to: https://github.com/DyfanJones/RAthena and https://github.com/DyfanJones/noctua
