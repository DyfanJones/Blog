---
title: RAthena and noctua update 2.6.0
author: Package Build
date: '2022-05-23'
slug: []
categories:
  - RBloggers
tags:
  - Athena
  - Boto3
  - paws
  - Python
  - R
description: 'Latest 2.6.0 update including latest features.'
thumbnail: ''
---

[RBloggers](https://www.r-bloggers.com)|[RBloggers-feedburner](https://feeds.feedburner.com/RBloggers)

## Intro:

This is a quick update on the latest features for `RAthena` and `noctua` 2.6.0.

## Latest Features:

### Endpoint Override:

Introducing a new parameter within the function `dbConnect`, `endpoint_override`. This allows `RAthena`/`noctua` to override each AWS service endpoint they connect to. `RAthena`/`noctua` connect to the following AWS services:

  - `AWS Athena` ( https://aws.amazon.com/athena/ ): main service to manipulate `AWS Athena`.
  - `AWS Glue` ( https://aws.amazon.com/glue/ ): service to the get `AWS Glue Catalogue` for `AWS Athena`.
  - `AWS S3` ( https://aws.amazon.com/s3/ ): to get `AWS Athena`'s results.

```r
# RAthena implementation
library(DBI)

# Change AWS Athena endpoint only
con <- dbConnect(RAthena::athena(),
  endpoint_override = "https://athena.us-east-1.amazonaws.com"
)

# Change a different AWS service
con <- dbConnect(RAthena::athena(), endpoint_override = list(
    glue = "https://glue.us-east-1.amazonaws.com"
  )
)

# Change multiple services
con <- dbConnect(RAthena::athena(), endpoint_override = list(
    athena = "https://athena.us-east-1.amazonaws.com",
    glue = "https://glue.us-east-1.amazonaws.com",
    s3 = "https://s3.us-east-1.amazonaws.com"
  )
)
```
**Note** `noctua` has the same implementation when using:
```r
con <- dbConnect(noctua::athena(), ...)
```

### Clear `AWS S3` resource:

Minor update to allow users to turn off the clear down of `AWS s3` resource created by `AWS Athena`.

```r
# RAthena implementation
RAthena::RAthena_options(clear_s3_resource = FALSE)
```
```r
# noctua implementation
noctua::noctua_options(clear_s3_resource = FALSE)
```

## Finally

To get the latest changes please update from cran:

```r
# Download and install RAthena in R from the cran
install.packages("RAthena")

# Download and install noctua in R from the cran
install.packages("noctua")
```

If you prefer to install the latest dev version please use `r-universe`.

```r
# Enable repository from dyfanjones
options(repos = c(
  ropensci = 'https://dyfanjones.r-universe.dev',
  CRAN = 'https://cloud.r-project.org')
)
  
# Download and install RAthena in R
install.packages('RAthena')

# Download and install noctua in R
install.packages('noctua')
```

If there is any new features or bug fixes please raise them at [`RAthena` issues](https://github.com/DyfanJones/RAthena/issues) or [`noctua` issues](https://github.com/DyfanJones/noctua/issues).
