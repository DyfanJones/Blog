---
title: R and Athena update
author: Dyfan Jones
date: '2021-10-28'
slug: [r-and-athena-update]
categories:
  - RBloggers
tags:
  - Athena
  - R
description: 'Latest updates and feature for R packages RAthena and noctua'
thumbnail: '/post/2021-10-28-r-and-athena-update/update.jpg'
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](https://feeds.feedburner.com/RBloggers)

## Intro:

As it has been an while since `RAthena` and `noctua` updates have been announce, I thought I would try and get them all out of the way now. This blog will cover, key new features that has been made from version 1.9.0 to 2.3.0.

## New Features:

* **Big integers**:  Big integers from `AWS Athena` can be return to R in the following supported data types [`integer64`, `integer`, `numeric`, `character`]
* **Extra `AWS Athena` data types**: Added support to `AWS Athena` data types [`array`, `row`, `map`, `json`, `binary`, `ipaddress`]

```R
library(DBI)
library(RAthena)

# default conversion methods
con <- dbConnect(RAthena::athena())

# change json conversion method
RAthena_options(json = "character")
RAthena:::athena_option_env$json
# [1] "character"

# change json conversion to custom method
RAthena_options(json = jsonify::from_json)
RAthena:::athena_option_env$json
# function (json, simplify = TRUE, fill_na = FALSE, buffer_size = 1024) 
# {
#   json_to_r(json, simplify, fill_na, buffer_size)
# }
# <bytecode: 0x7f823b9f6830>
#   <environment: namespace:jsonify>

# change bigint conversion without affecting custom json conversion methods
RAthena_options(bigint = "numeric")
RAthena:::athena_option_env$json
# function (json, simplify = TRUE, fill_na = FALSE, buffer_size = 1024) 
# {
#   json_to_r(json, simplify, fill_na, buffer_size)
# }
# <bytecode: 0x7f823b9f6830>
#   <environment: namespace:jsonify>

RAthena:::athena_option_env$bigint
# [1] "numeric"

# change binary conversion without affect, bigint or json methods
RAthena_options(binary = "character")
RAthena:::athena_option_env$json
# function (json, simplify = TRUE, fill_na = FALSE, buffer_size = 1024) 
# {
#   json_to_r(json, simplify, fill_na, buffer_size)
# }
# <bytecode: 0x7f823b9f6830>
#   <environment: namespace:jsonify>

RAthena:::athena_option_env$bigint
# [1] "numeric"

RAthena:::athena_option_env$binary
# [1] "character"

# no conversion for json objects
con2 <- dbConnect(RAthena::athena(), json = "character")

# use custom json parser
con <- dbConnect(RAthena::athena(), json = jsonify::from_json)
```

* **RStudio connection tab**: Allowed RStudio connection tab to be optional, this is to speed up connection when users are connecting to large Data Lakes.
* **Time zone**: Added support to `AWS Athena` `timestamp` with time zone data type.
* **R list**: Properly support data type `list` when converting data to `AWS Athena` `SQL` format.

```r
library(data.table)
library(DBI)

x = 5

dt = data.table(
  var1 = sample(LETTERS, size = x, T),
  var2 = rep(list(list("var3"= 1:3, "var4" = list("var5"= letters[1:5]))), x)
)

con <- dbConnect(noctua::athena())

#> Version: 2.2.0
sqlData(con, dt)

# Registered S3 method overwritten by 'jsonify':
#   method     from    
#   print.json jsonlite
# Info: Special characters "\t" has been converted to " " to help with Athena reading file format tsv
#    var1                                                   var2
# 1:    1 {"var3":[1,2,3],"var4":{"var5":["a","b","c","d","e"]}}
# 2:    2 {"var3":[1,2,3],"var4":{"var5":["a","b","c","d","e"]}}
# 3:    3 {"var3":[1,2,3],"var4":{"var5":["a","b","c","d","e"]}}
# 4:    4 {"var3":[1,2,3],"var4":{"var5":["a","b","c","d","e"]}}
# 5:    5 {"var3":[1,2,3],"var4":{"var5":["a","b","c","d","e"]}}

#> Version: 2.1.0
sqlData(con, dt)

# Info: Special characters "\t" has been converted to " " to help with Athena reading file format tsv
#    var1                                        var2
# 1:    1 1:3|list(var5 = c("a", "b", "c", "d", "e"))
# 2:    2 1:3|list(var5 = c("a", "b", "c", "d", "e"))
# 3:    3 1:3|list(var5 = c("a", "b", "c", "d", "e"))
# 4:    4 1:3|list(var5 = c("a", "b", "c", "d", "e"))
# 5:    5 1:3|list(var5 = c("a", "b", "c", "d", "e"))
```

* **`AWS Athena UNLOAD`**: Add support to [`AWS Athena UNLOAD`](https://docs.aws.amazon.com/athena/latest/ug/unload.html/) . This is to take advantage of read/write speed `parquet`.

> Set up AWS Athena table (example taken from AWS Data Wrangler: [Amazon Athena Tutorial](https://aws-data-wrangler.readthedocs.io/en/stable/tutorials/006%20-%20Amazon%20Athena.html)):
```python
import awswrangler as wr

import getpass
bucket = getpass.getpass()
path = f"s3://{bucket}/data/"


if "awswrangler_test" not in wr.catalog.databases().values:
    wr.catalog.create_database("awswrangler_test")

cols = ["id", "dt", "element", "value", "m_flag", "q_flag", "s_flag", "obs_time"]

df = wr.s3.read_csv(
    path="s3://noaa-ghcn-pds/csv/189",
    names=cols,
    parse_dates=["dt", "obs_time"])  # Read 10 files from the 1890 decade (~1GB)

wr.s3.to_parquet(
    df=df,
    path=path,
    dataset=True,
    mode="overwrite",
    database="awswrangler_test",
    table="noaa"
);

wr.catalog.table(database="awswrangler_test", table="noaa")
```
> Benchmark unload method using `noctua`.
```R

library(DBI)

con <- dbConnect(noctua::athena())

# Query ran using CSV output
system.time({
  df = dbGetQuery(con, "SELECT * FROM awswrangler_test.noaa")
})
# Info: (Data scanned: 80.88 MB)
#    user  system elapsed
#  57.004   8.430 160.567 

noctua::noctua_options(cache_size = 1)

# Query ran using UNLOAD Parquet output
system.time({
  df = dbGetQuery(con, "SELECT * FROM awswrangler_test.noaa", unload = T)
})
# Info: (Data scanned: 80.88 MB)
#    user  system elapsed 
#  21.622   2.350  39.232 

# Query ran using cache
system.time({
  df = dbGetQuery(con, "SELECT * FROM awswrangler_test.noaa", unload = T)
})
# Info: (Data scanned: 80.88 MB)
#    user  system elapsed 
#  13.738   1.886  11.029 
```
> **Note:** Benchmark ran on `AWS Sagemaker` ml.t3.xlarge instance.

## Finally:

If there is any new features or bug fixes, please raise a ticket on: [noctua](https://github.com/DyfanJones/noctua/issues) & [RAthena](https://github.com/DyfanJones/RAthena/issues)

