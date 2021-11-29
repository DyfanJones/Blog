---
title: AWS Athena dplyr unload
author: R package build
date: '2021-11-29'
slug: [aws-athena-dplyr-unload]
categories:
  - RBloggers
tags:
  - Athena
  - R
description: 'Latest 2.4.0 update including latest features.'
thumbnail: ''
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](https://feeds.feedburner.com/RBloggers)

## Intro:

This is a quick update on the latest features for `RAthena` and `noctua` 2.4.0.

## Latest features:

### `dbplyr`:

`RAthena` and `noctua` now fully supports [`dbplyr` backend api 2+](https://dbplyr.tidyverse.org/articles/backend-2.html). `dplyr` database generics will be deprecated in later versions of the `dbplyr` package development. This is to future proof `RAthena` and `noctua`, while keeping the same functionality developed for `dbplyr` backend api version 1.

### `dplyr` and unload:

`RAthena` and `noctua` can now set `AWS Athena Unload` on a session level. This enables dplyr syntax to leverage [`AWS Athena unload`](https://docs.aws.amazon.com/athena/latest/ug/unload.html) without any extra code.

**Note:** Set up AWS Athena example table using [AWS Athena awswrangler example:](https://aws-data-wrangler.readthedocs.io/en/stable/tutorials/006%20-%20Amazon%20Athena.html)

```python
# Python
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

Set up connection to `AWS Athena`.

```r
library(DBI)
library(RAthena) # or library(noctua)
library(dplyr, warn.conflicts = F)

con <- dbConnect(athena())
```

Query `AWS Athena` table without using `AWS Athena unload` method.

```r
noaa_tbl = tbl(con, dbplyr::in_schema("awswrangler_test","noaa"))

system.time({noaa = collect(noaa_tbl)})
#> Info: (Data scanned: 80.86 MB)
#>   user  system elapsed 
#> 67.144   7.353 126.914 

dim(noaa)
#> 29554220 8
```

Query `AWS Athena` table using `AWS Athena unload` method.

```r
RAthena_options(unload = TRUE) # or noctua_options(unload = TRUE) for noctua

system.time({noaa = collect(noaa_tbl)})
#> Info: (Data scanned: 80.86 MB)
#>   user  system elapsed 
#> 11.062   2.317  32.078 

dim(noaa)
#> 29554220 8
```

Query `AWS Athena` table using `AWS Athena unload` method, while caching.

```r
RAthena_options(cache_size = 10, unload = TRUE) # or noctua_options(cache_size = 10, unload = TRUE) for noctua
```

```r
system.time({noaa = collect(noaa_tbl)})
#> Info: (Data scanned: 80.86 MB)
#>   user  system elapsed 
#> 10.768   2.541  12.305 

dim(noaa)
#> 29554220 8
```
> Benchmark ran on AWS Sagemaker

**NOTE:** Cache speeds will only benefit repeat queries!

`AWS Athena Unload` method doesn't work on every `sql` query type. As a result, several `dplyr` methods won't work when unload is set to `True`, for example:

```r
noaa_tbl %>% filter(element == "TMAX") %>% compute()

#> Error: Unable to create table when `RAthena_options(unload = TRUE)`. Please run `RAthena_options(unload = FALSE)` and try again.
```

In general `AWS Athena unload` method is a nice way to speed up some queries when working with `RAthena` and `noctua` when the data returning is large.

## Finally

If there is any new features or bug fixes please raise them at [`RAthena` issues](https://github.com/DyfanJones/RAthena/issues) or [`noctua` issues](https://github.com/DyfanJones/noctua/issues).
