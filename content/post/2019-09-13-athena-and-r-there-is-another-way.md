---
title: Athena and R ... there is another way!?
author: ~
date: '2019-09-13'
slug: athena-and-r-there-is-another-way
categories: []
tags: [R, Athena, Boto3, Python]
---
# Intro

Currently there are two key ways in connecting to Athena from R, using the [ODBC](https://docs.aws.amazon.com/athena/latest/ug/connect-with-odbc.html) and [JDBC](https://docs.aws.amazon.com/athena/latest/ug/connect-with-jdbc.html) drivers. To access ODBC driver R users can use the excellent [odbc package](https://github.com/r-dbi/odbc) supported by Rstudio. To access the JDBC driver R users can either use [RJDBC](https://cran.r-project.org/web/packages/RJDBC/index.html) or helpful wrapper package [AWR.Athena](https://github.com/nfultz/AWR.Athena) which wraps the RJDBC package to make the connection to Athena through the JDBC driver simpler. These methods are an excellent way for R to connect to Athena, however is there another way?

Well glad you asked...yes there is! Ever since the [reticulate pacakge](https://rstudio.github.io/reticulate/) was developed (by Rstudio) the interface into Python from R has never been simpler. This makes another route into Athena possible! Amazon has developed a Python software developement kit (SDK) called [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html?id=docs_gateway). By using `boto3` in combination with the R package `reticulate` a new method into accessing Athena can be made possible. Introducing the R package [RAthena](https://dyfanjones.github.io/RAthena/).

# RAthena

* **What is RAthena?**

>`Rathena` is a R package that creates a DBI (Database Interface) for the R package [DBI](https://dbi.r-dbi.org/) by using `Boto3` as the backend.

* **Why was RAthena created when there are already methods for connecting to Athena?**

> `RAthena` was created to provide an extra method to connect to Athena for R users. Nothing more, nothing less.

* **Why is RAthena call RAthena?**

> Isn't it obvious? Most R packages that interfaces with databases are called `"R<database>"` for example `RSQLite`, `RPostgreSQL`, etc... Plus this package is "roughly" the R equivalent to the superb `Python` package [PyAthena](https://github.com/laughingman7743/PyAthena). So calling this pacakge `RAthena` seems like the best fit.

# Getting Started

Now lets get into how to actually use `RAthena`. I am going to skip over the part were you have to set up an [Amazon Web Services Account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) (AWS Acount) and get straight into the good stuff.

Before working with `RAthena`, [Python 3+](https://www.python.org/downloads/) is require. Please install it directly or use the [Anaconda Distribution](https://www.anaconda.com/distribution/).

Now we have `Python 3+` we now need to install `Boto3`. If you installed `Python 3+` with the Anaconda Distribution I believe `Boto3` comes as standard, if so then you can skip this step, but for everyone else you can install `Boto3` either by the `pip` command or the inbuilt installation function in `RAthena`.

**pip command:**
```
pip install boto3
```

**`RAthena` installation function:**
```r
RAthena::install_boto()
```

# Usage

## Connecting to Athena

Now we have everything that is required we are now ready to connect to `Athena`. `RAthena` provides several method to connect to `Athena` ranging from hard-coding credentials to using Amazon Role Name Roles (ARN roles).

### Hard-Coding Method:

This method isn't recommended as your credentials are hard-coded.

```r
library(DBI)
con <- dbConnect(RAthena::athena(),
                 aws_access_key_id = "<YOUR AWS KEY ID>",
                 aws_secret_access_key = "<YOUR SECRET ACCESS KEY>",
                 s3_staging_dir = "<LOCATION FOR ATHENA QUERY OUTPUT>")
```
**Note:** *`s3_staging_dir` requires the be the format of `s3 uri` for example "s3://path/to/query/bucket/"*

### Environment Variable Method:

`RAthena` supports setting [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) into environment variables to avoid hard-coding. From what I have found out an easy way to set up environment variables (that persists) in R is to use the `file.edit` function like so:
```r
file.edit("~/.Renviron")
```
And now you can simply add in your variables to the file for example:
```
AWS_ACCESS_KEY_ID = <YOUR AWS KEY ID>
```

Once you have set your environment variables you can connect to Athena in the following method:

```r
library(DBI)
con <- dbConnect(RAthena::athena(),
                 s3_staging_dir = "<LOCATION FOR ATHENA QUERY OUTPUT>")
```

### AWS Profile Names:

Another method is to use AWS Profile Names. AWS profile names can be set either manually in the `~/.aws` directory or by using the [AWS Command Line Interface (AWS CLI)](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html). Once you have set up your profile name you can connect to `Athena`:

**Using Default Profile Name:**
```r
library(DBI)
con <- dbConnect(RAthena::athena(),
                 s3_staging_dir = "<LOCATION FOR ATHENA QUERY OUTPUT>")
```
**Using Non-Default Profile Name:**
```r
library(DBI)
con <- dbConnect(RAthena::athena(),
                 profile_name = "rathena",
                 s3_staging_dir = "<LOCATION FOR ATHENA QUERY OUTPUT>")
```
### ARN Roles:

ARN roles are fairly useful if you need to assume a role that can connect to another AWS account and use the `Athena` in that account. Or whether you want to create a temporary connection that will expire (there are other reasons why to use ARN roles please check them out in the [AWS documentation](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html)).

**Creating ARN role credentials before connecting to Athena:**
```r
library(RAthena)
library(DBI)
assume_role(profile_name = "YOUR_PROFILE_NAME",
            role_arn = "arn:aws:sts::123456789012:assumed-role/role_name/role_session_name",
            set_env = TRUE)

# Connect to Athena using ARN Role
con <- dbConnect(athena(),
                s3_staging_dir = "s3://path/to/query/bucket/")
```
**Connect to Athena directly using ARN role:**
```r
library(DBI)
con <- dbConnect(athena(),
                  profile_name = "YOUR_PROFILE_NAME",
                  role_arn = "arn:aws:sts::123456789012:assumed-role/role_name/role_session_name",
                  s3_staging_dir = 's3://path/to/query/bucket/')
```

**Note:** *ARN Roles have a duration timer before they will expirer. To change the default you can increase the `duration_seconds` parameter from the default 3600 seconds (1 hour).*

### Temporary Sessions:

Finally you can create temporary credentials before connecting to `Athena`.
```r
library(RAthena)
library(DBI)

# Create Temporary Credentials duration 1 hour
get_session_token("YOUR_PROFILE_NAME",
                  serial_number='arn:aws:iam::123456789012:mfa/user',
                  token_code = "531602",
                  set_env = TRUE)

# Connect to Athena using temporary credentials
con <- dbConnect(athena(),
                s3_staging_dir = "s3://path/to/query/bucket/")
```

**Note:** *This method will work for users who have set up [Multi-Factor Authentication](https://aws.amazon.com/iam/details/mfa/) (MFA).*

## Basic Usage

Now we have created a connection to `Athena` we can ulitise `DBI` methods to query `Athena` for example:

**All avialable tables in Athena:**
```r
dbListTables(con)
```

**Send Query to Athena**
```r
res <- dbSendQuery(con, "SELECT * FROM INFORMATION_SCHEMA.COLUMNS")
dbFetch(res)
dbClearResult(res)
```
Or ...
```r
res <- dbExecute(con, "SELECT * FROM INFORMATION_SCHEMA.COLUMNS")
dbFetch(res)
dbClearResult(res)
```
Or ...

`dbGetQuery` wraps sending, fetching and clearing results in one easy step.
```r
dbGetQuery(con, "SELECT * FROM INFORMATION_SCHEMA.COLUMNS")
```

**Note:** *You might of noticed that if you have `data.table` installed `RAthena` will attempt to return the data as a `data.table`. This is to improve speed when larger queries are returned from `Athena`. If you don't have `data.table`, `RAthena` will return the output as a `data.frame`.*

**Get Column information**
```r
res <- dbSendQuery(con, "SELECT * FROM INFORMATION_SCHEMA.COLUMNS")
dbColumnInfo(res)
dbClearResult(res)
```

To learn about what `DBI` methods have been implemented in `RAthena` please refer to: [link](https://dyfanjones.github.io/RAthena/).

## Intermediate Usage:

It is all very well querying data from `Athena` but what is more useful is to upload data as well. `RAthena` has addressed this and implemented a method in `dbWriteTable`

```r
dbWriteTable(con, "mtcars", mtcars,
             partition=c("TIMESTAMP" = format(Sys.Date(), "%Y%m%d")),
             s3.location = "s3://mybucket/data/")
```

Here are all variable parameters for the `dbWriteTable` method:

>**conn:** An AthenaConnection object, produced by dbConnect()

>**name:** A character string specifying a table name. Names will be automatically quoted so you can use any sequence of characters, not just any valid bare table name.

> **value:** A data.frame to write to the database.

> **overwrite:** Allow overwriting the destination table. Cannot be 'TRUE' if 'append' is also 'TRUE'.

> **append:** Allow appending to the destination table. Cannot be 'TRUE' if 'overwrite' is also 'TRUE'.

> **row.names:** Either TRUE, FALSE, NA or a string. If TRUE, always translate row names to a column called "row_names". If FALSE, never translate row names. If NA, translate rownames only if they're a character vector. A string is equivalent to TRUE, but allows you to override the default name. For backward compatibility, NULL is equivalent to FALSE.

> **field.types:** Additional field types used to override derived types.

> **partition:** Partition Athena table (needs to be a named list or vector) for example: c(var1 = "2019-20-13")

> **s3.location** s3 bucket to store Athena table, must be set as a s3 uri for example ("s3://mybucket/data/")

> **file.type:** What file type to store data.frame on s3, RAthena currently supports ["csv", "tsv", "parquet"]. **Note:** *file.type "parquet" is supported by R package [`arrow`](https://github.com/apache/arrow/tree/master/r) and will need to be installed seperately if you wish to upload data.frames in "parquet" format.*

> **...:** Other arguments used by individual methods.

## Tidyverse Usage:

`RAthena` can integrate with the famous R package [dplyr](https://github.com/tidyverse/dplyr).

```r
library(DBI)
library(dplyr)

con <- dbConnect(RAthena::athena(),
                 profile_name = "rathena",
                 s3_staging_dir = "<LOCATION FOR ATHENA QUERY OUTPUT>")

tbl(con, sql("SELECT * FROM INFORMATION_SCHEMA.COLUMNS"))
```

Or if you have already uploaded a table into `Athena`

```r
tbl(con, "mtcars")
```

# Conclusion

So hopefully this has given you an insight into the up coming package `RAthena` and it's usefulness. This package is not meant to replace any of the other packages that connect into `Athena` but give another route into `Athena` for R users.

**Final note:** `RAthena` offers alot more functionality please check it out at [Github](https://github.com/DyfanJones/RAthena). If you have any suggestions please raise and issue at [Github Issues](https://github.com/DyfanJones/RAthena/issues).