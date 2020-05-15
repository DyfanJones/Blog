---
title: R doesn't need to throttle AWS Athena anymore
author: Dyfan Jones
date: '2020-02-22'
slug: r-doesnt-need-to-throttle-aws-athena-anymore
categories: [RBloggers]
tags:
  - R
  - Athena
  - Boto3
  - paws1150
  - Python
description: ''
thumbnail: ''
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](https://feeds.feedburner.com/RBloggers)

I am happy to announce that `RAthena-1.9.0` and `noctua-1.7.0` have been released onto the cran. They both bring two key features:

* More stability when working with `AWS Athena`, focusing on `AWS` `Rate Exceeded` throttling errors
* New helper function to convert `AWS S3` backend files to save cost

**NOTE:** `RAthena` and `noctua` features correspond to each other, as a result I will refer to them interchangeability.

# Stability
## Throttling AWS

One of the main problems when working with `AWS` API is stumbling into `Rate Exceeded` throttling error. With the latest update to the packages, the connection between `AWS Athena` and `R` has been made more robust through retry functionality. This allows `R` to automatically retry its request using an exponential backoff with jitter ([Best practices for API packages](https://httr.r-lib.org/articles/api-packages.html#turn-api-errors-into-r-errors))

![](/post/2020-05-15-r-doesnt-need-to-throttle-aws-athena-anymore_files/aws_retry.png)

1. `R` sends a call to `AWS Athena`, let's say `dbListTables(con)`. However `AWS` is over run with requests, and returns an error back to `R` saying it is overwhelmed (this is a `rate exceeded` throttling error). As `RAthena` and `noctua` retry noisely, the error will be printed to the console letting you know `AWS` is busy (`{expection message} + "Request failed. Retrying in " + {wait time}  + " seconds..."`).
2. `R` will then wait for a given time (please see error format above) and retry the request again. `AWS` replies it is still busy and can't do the request.
3. This time `R` will back off for a long period of time, this gives `AWS` some breathing room. Now when `R` sends the request over to `AWS`, `AWS` is able to complete the call and return out desired results.

This feature is a great step in the right direction for making `R` and `AWS Athena` work together seamlessly. For anyone who wishes to create their own retry method both packages have enabled this through their `..._options()` function. For more information please refer to [link](https://dyfanjones.github.io/noctua/articles/how_to_retry.html).

# Save the pennies
## Converting `AWS S3` files

`AWS Athena` costs by the amount of data it scans. This makes it very important to have your `AWS S3` backend files in the suitable format to reduce the cost of using `AWS Athena`. This is where the next key feature comes in. This feature basically creates a simple wrapper to allow you to convert `AWS S3` files into a more suitable format.

```
library(DBI)
library(RAthena)

con <- dbConnect(athena())

# Upload iris data.frame to AWS Athena as a delimited file
dbWriteTable(con, "iris_delim", iris)

# Convert to parquet using AWS Athena
dbConvertTable(con,
               obj = "iris_delim",
               name = "iris_parquet",
               file.type = "parquet")
```

In this example simply uploaded iris `data.frame` to `AWS Athena` in a default delimited file format (please see [link](https://dyfanjones.github.io/RAthena/reference/AthenaWriteTables.html) for more information around how to upload data to `AWS Athena`). Then it is converted into [`parquet`](https://parquet.apache.org/) file format using `AWS Athena`. This wrapper isn't limited to converting just AWS Athena tables, it can also convert [`SQL DML`](https://docs.aws.amazon.com/athena/latest/ug/select.html) queries. Please refer to [`dbConvertTable`](https://dyfanjones.github.io/RAthena/reference/dbConvertTable.html) for more documentation or to the `dbConvertTable` vignette [link](https://dyfanjones.github.io/RAthena/articles/convert_and_save_cost.html).

Finally for more informations around best practises with `AWS Athena` please look at [Top 10 Perfromace Tuning Tips for Amazon Athena](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)

# Sum Up

These two new features bring `R` and `AWS Athena` that little bit closer together. As always if you have any new features or identify any bugs please feel free to raise a pull request or ticket on the corresponding package github pages ([`RAthena`](https://github.com/DyfanJones/RAthena/) and [`nocuta`](https://github.com/DyfanJones/noctua))

