---
title: RAthena 1.3.0 has arrived
author: Dyfan Jones
date: '2019-10-26'
slug: rathena-1-3-0-has-arrived
categories: [RBloggers]
tags: [R, Athena, Boto3, Python]
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](http://feeds.feedburner.com/RBloggers )

# Recap:

[`RAthena`](https://github.com/DyfanJones/RAthena) is a R package that interfaces into Amazon Athena. However, it doesn't use the standard `ODBC` and `JDBC` drivers like `AWR.Athena` and `metis`. Instead `RAthena` utilises Python's SDK (software development kit) into Amazon, [`Boto3`](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html). It does this by using the [`reticulate`](https://rstudio.github.io/reticulate/) package that provides an interface into Python. What this means is that `RAthena` doesn't require any driver installation or setup. That can be particularly difficult when you are considering setting up the ODBC drivers and you are not familiar with how ODBC works on your current operating system. If you wish to use ODBC, RStudio has provided a good user guide [Setting up ODBC Drivers](https://db.rstudio.com/best-practices/drivers/) to help set up ODBC drivers on your system. However if you do not wish to go down that route `RAthena` might be a good option for you.

# New Features in `RAthena`:

Anyway, getting back to `RAthena` and what does the new update provide. One of the key changes in `RAthena` is the method of transferring data to and from AWS Athena. `RAthena` now utilising [`data.table`](https://rdatatable.gitlab.io/data.table/) for this process. The reason for this change is the raw speed `data.table`. When transferring data to and from AWS Athena the last thing you want is a bottle neck in R just preparing the data before it even transfers it to AWS Athena. This bottle neck can easily be 50 - 100x longer without the use of `data.table`. 

The next change is `bigint`, and how it is converted from AWS Athena to R. In the past `RAthena` would just convert `integer64` to `bigint` when writing to AWS Athena, however it would then convert `bigint` back into R as a normal `integer`. Which means it is constrained to 32-bit integers. This has now been fixed. When reading `bigint` from AWS Athena `RAthena` will now convert it into `integer64`. 

# Sum Up:

`RAthena` now provides a faster method in reading and writing data from AWS Athena (thanks `data.table`). With the correct handling of AWS Athena `bigint`. So please give `RAthena` a try and let me know what you think of the package. Suggestions/Bugs/Enhancements are always welcome and they will help the package to improve: https://github.com/DyfanJones/RAthena/issues.

## Installation methods:

Just in case you are not aware [`Rathena`](https://cran.r-project.org/web/packages/RAthena/index.html) is available on the CRAN and GitHub.

CRAN:
```
install.packages("RAthena")
```

GitHub development version:

```
remotes::install_github("dyfanjones/RAthena")
```
