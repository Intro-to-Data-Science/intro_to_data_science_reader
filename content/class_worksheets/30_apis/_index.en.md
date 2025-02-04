---
pre: <b>11/16. </b>
title: "APIs"
weight: 30
summary: "Ask for your data the right way"
format:
    hugo:
      toc: true
      output-file: "_index.en.md"
      reference-links: true
      code-link: true
      
---



-   [Overview][]
-   [The Data][]
-   [Figuring out the Rules][]
    -   [API Structure][]
-   [Getting the Data][]
    -   [Via API][]
    -   [Web Interface][]

## Overview

APIs allow us access to large amounts of data that we wouldn't have access to otherwise. While setting one up can take some time, it is almost always preferable to the alternative of downloading and dealing with with a pile of very large, very slow files. One situation in which this applies is the [Open Payments][] data we are using for our project!

Today we'll work through how you create API calls for the Open Payments data. This will hopefully make working with the large data sets more approachable. I encourage you to read through this process *completely* with me first, and then customize it to fit your specific project after.

## The Data

While we are working with Open Payments data today, we're going to do so in a completely new way. Rather than loading all the data into R first, then sub-setting what we want, we will essentially do the sub-setting in our API call. This will save us the hassle fo loading a *massive* file into R, only to throw a bunch of it away later. It will also make getting data from multiple years much easier, as we can re-use our API calls to get the same data across years.

For the purpose of this worksheet, we're going to be looking at the [2021 General Payment Data][], but the process is the same for any of the Open Payments data. Open up the data page in a browser, as we'll need to reference the API documentation a lot; it's at the bottom of the page.

## Figuring out the Rules

The first step of using any API is figuring out how it works. Every API is different, so reading the docs is super important. The Open Payments docs could be better.

At the bottom of the [2021 General Payment Data][] page, you will see several colored boxes like the following:

![][1]

We want to use `GET` to "query" (subset) data from this dataset, so click on the arrow in the box corresponding to this single dataset.

![][2]

Once you do that, the box will open up to give us more information about getting data from this dataset. The top part of the box defines the *parameters*, or what you can actually ask for in your API call. The bottom portion explains the *responses*, or what we can expect back given our parameters; a response of 200 means things worked, anything else was an error of some kind.

Beyond that, this documentation is... lacking. Unfortunately, that isn't particularly uncommon. Either through lack of expertise on the part of the provider, or the confidence that if you are data-nerd-y enough to be using APIs you can figure it out yourself, sometimes you don't have much else to work from. Yet, I wanted you to learn how to use this API for your project, so I figured it out. I'll walk you through it now.

### API Structure

Recall from lecture that all APIs have a few components in common:

-   The Endpoint - What site we are getting data from
-   The Data Source - What data we want from that site
-   The Data Type - What format are we getting the data in
-   The Query - What specifically we are asking for

The same is true of the Open Payments data, even if they don't really tell us. In the API documentation box, there is a button in the upper right that says "Try it out." If you click that, it will let you start modifying the parameters, and unlock the option to execute a test API call. If you do so, it will give you an example API call, and try to run it in your browser. *This will take a fairly long time, and potentially crash your browser.* Instead, I'll provide the example API call here:

    https://openpaymentsdata.cms.gov/api/1/datastore/query/0380bbeb-aea1-58b6-b708-829f92a48202/0?offset=0&count=true&results=true&schema=true&keys=true&format=json&rowIds=false

I figured out that their API call breaks down as follows:

Endpoint
:   https://openpaymentsdata.cms.gov/

Data Source
:   api/1/datastore/.../0380bbeb-aea1-58b6-b708-829f92a48202/

Data Type
:   format=json

Query
:   Everything else.

This call will get you the whole 2021 General Payment Data as one large JSON file. From here, we can start to modify it to get only what we want. In the example below, I've tweaked the same call to give me only the rows from Massachusetts. I added:

`&conditions[0][property]=recipient_state`
:   This specified the column I wanted to subset by

`&conditions[0][value]=MA`
:   This was the value in that column I wanted to get

`&conditions[0][operator]==`
:   I wanted the value in the column to equal the value I provided (as opposed to not equal, etc.)

{{% notice info %}}
If any part of your API call has a space, you will need to replace it with `%20`. Computers typically don't like spaces in paths, and it will break things. `%20` means the same thing as space to a computer without that danger.
{{% /notice %}}

``` r
library(httr)
library(jsonlite)

# get the data from API
op_ma_2021_1 = GET("https://openpaymentsdata.cms.gov/api/1/datastore/query/0380bbeb-aea1-58b6-b708-829f92a48202/0?limit=500&offset=0&count=true&results=true&schema=true&keys=true&format=json&rowIds=false&conditions[0][property]=recipient_state&conditions[0][value]=MA&conditions[0][operator]==")

# convert to DF
op_ma_2021_1 = fromJSON(rawToChar(op_ma_2021_1$content))$results
```

## Getting the Data

### Via API

If you look at this dataframe and compare it to the MA data we got on the project page, you will notice some key differences however. One is that everything is a character; that can be fixed. Another is that we only got 500 rows. What gives?! Turns out, you can only get 500 rows at a time with the API; if you try to get any more it will only return an error.

So how do we get our data? We iterate 500 rows at a time. In our API call, we need to pull the first 500 rows, then use the `offset=` parameter to start at row 501, then 1001, then 1501, etc. until we run out of rows to get. I do so below:

{{% notice info %}}
Do not run this code. It takes forever. Just read it and try to understand how it works.
{{% /notice %}}

``` r
# Turn off scientific notation so it does not break query format
options(scipen=999)

# set the default value of GET return to 200
status_return = 200

# start with 0 offset (so row 1)
starting_row = 0

# defualt for check if new rows are still coming
still_data = TRUE

# make empty list for each dataframe of 500 rows
out_list = list()

# until the API tells me the it has no more rows to give me (a non-200 response), keep offsetting by 500 and GET again
while(status_return == 200 & still_data){
  
  # print current starting row
  print(starting_row)
  
  # get data starting at starting_row, put it in list with element named starting_row + 1 (API starts at 0, R starts at 1)
  api_return = GET(paste0("https://openpaymentsdata.cms.gov/api/1/datastore/query/0380bbeb-aea1-58b6-b708-829f92a48202/0?limit=500&offset=", starting_row, "&count=true&results=true&schema=true&keys=true&format=json&rowIds=false&conditions[0][property]=recipient_state&conditions[0][value]=MA&conditions[0][operator]=="))
  
  # puts results into list as df
  out_list[[as.character(starting_row + 1)]] = fromJSON(rawToChar(api_return$content))$results
  
  # get status of call (will != 200 if failed)
  status_return = api_return$status_code
  
  # add 500 to starting row
  starting_row = starting_row + 500
  
  # test if results are empty
  still_data = length(out_list[[as.character(starting_row + 1)]]) > 0
  
}

# combine all the mini-dataframes into final dataframe
api_dfs = lapply(out_list, FUN=function(x){fromJSON(rawToChar(x$content))$results})
api_dfs = do.call(rbind, api_dfs)
```

My `api_dfs` is now identical to if I filtered the whole dataset by `recipient_state==MA` in R. The bonus is I could do that *before* loading the whole dataset into R. This is practically required once the dataset gets big enough.

### Web Interface

The Open Payments API isn't particularly easy to use. Rather than using that for your projects, I will recommend another system. Go the the page for the dataset you are interested in, and then click the "View and Filter Data" button.

![][3]

You can then add and apply filters to the dataset on the website itself (make sure you actually press the "Apply" button).

![][4]

Once you have done that, you can right-click on the "Download filtered data (CSV)" button, copy the link, then paste it into R with a `read.csv()` function to accomplish the same thing as the API in a much less painful way.

![][5]

``` r
# get the general payment data for MA 2021
filter_return = read.csv("https://openpaymentsdata.cms.gov/api/1/datastore/query/0380bbeb-aea1-58b6-b708-829f92a48202/0/download?conditions%5B0%5D%5Bproperty%5D=recipient_state&conditions%5B0%5D%5Bvalue%5D=MA&conditions%5B0%5D%5Boperator%5D=%3D&format=csv")
```

  [Overview]: #overview
  [The Data]: #the-data
  [Figuring out the Rules]: #figuring-out-the-rules
  [API Structure]: #api-structure
  [Getting the Data]: #getting-the-data
  [Via API]: #via-api
  [Web Interface]: #web-interface
  [Open Payments]: https://openpaymentsdata.cms.gov/
  [2021 General Payment Data]: https://openpaymentsdata.cms.gov/dataset/0380bbeb-aea1-58b6-b708-829f92a48202
  [1]: img/api_docs_1.png
  [2]: img/api_docs_2.png
  [3]: img/api_filter.png
  [4]: img/filter_2.png
  [5]: img/filter_3.png
