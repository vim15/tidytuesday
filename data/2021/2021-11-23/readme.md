### Please add alt text to your posts

Please add alt text (alternative text) to all of your posted graphics for `#TidyTuesday`. 

Twitter provides [guidelines](https://help.twitter.com/en/using-twitter/picture-descriptions) for how to add alt text to your images.

The DataViz Society/Nightingale by way of Amy Cesal has an [article](https://medium.com/nightingale/writing-alt-text-for-data-visualization-2a218ef43f81) on writing _good_ alt text for plots/graphs.

> Here's a simple formula for writing alt text for data visualization:
> ### Chart type
> It's helpful for people with partial sight to know what chart type it is and gives context for understanding the rest of the visual.
> Example: Line graph
> ### Type of data
> What data is included in the chart? The x and y axis labels may help you figure this out.
> Example: number of bananas sold per day in the last year
> ### Reason for including the chart
> Think about why you're including this visual. What does it show that's meaningful. There should be a point to every visual and you should tell people what to look for.
> Example: the winter months have more banana sales
> ### Link to data or source
> Don't include this in your alt text, but it should be included somewhere in the surrounding text. People should be able to click on a link to view the source data or dig further into the visual. This provides transparency about your source and lets people explore the data.
> Example: Data from the USDA

Penn State has an [article](https://accessibility.psu.edu/images/charts/) on writing alt text descriptions for charts and tables.

> Charts, graphs and maps use visuals to convey complex images to users. But since they are images, these media provide serious accessibility issues to colorblind users and users of screen readers. See the [examples on this page](https://accessibility.psu.edu/images/charts/) for details on how to make charts more accessible.

The `{rtweet}` package includes the [ability to post tweets](https://docs.ropensci.org/rtweet/reference/post_tweet.html) with alt text programatically.

Need a **reminder**? There are [extensions](https://chrome.google.com/webstore/detail/twitter-required-alt-text/fpjlpckbikddocimpfcgaldjghimjiik/related) that force you to remember to add Alt Text to Tweets with media.

# Dr. Who

The data this week comes from the [`datardis` package](https://github.com/KittJonathan/datardis/tree/main/data) by way of Jonathan Kitt.

They have a short blogpost on the package at: https://randomics.netlify.app/posts/2021-11-16-datardis/ (post no longer available as of 2024-05-06).

Additional brief articlet from the [Independent.ie](https://www.independent.ie/entertainment/doctor-who-suffers-lowest-ratings-since-2005-revival-39028919.html)

### Get the data here

```r
# Get the Data

# Read in with tidytuesdayR package 
# Install from CRAN via: install.packages("tidytuesdayR")
# This loads the readme and all the datasets for the week of interest

# Either ISO-8601 date or year/week works!

tuesdata <- tidytuesdayR::tt_load('2021-11-23')
tuesdata <- tidytuesdayR::tt_load(2021, week = 48)

directors <- tuesdata$directors

# Or read in the data manually

directors <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2021/2021-11-23/directors.csv')
episodes <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2021/2021-11-23/episodes.csv')
writers <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2021/2021-11-23/writers.csv')
imdb <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2021/2021-11-23/imdb.csv')



```
### Data Dictionary

# `directors.csv`

|variable     |class     |description |
|:------------|:---------|:-----------|
|story_number |character | Story number |
|director     |character | Director |

# `episodes.csv`

|variable        |class     |description |
|:---------------|:---------|:-----------|
|era             |character |Era = Classic or revived |
|season_number   |double    | Season number |
|serial_title    |character | Serial title |
|story_number    |character | Storu number |
|episode_number  |integer   | Episode number |
|episode_title   |character | Episode title |
|type            |character | Type |
|first_aired     |double    |First aired date |
|production_code |character |Production code |
|uk_viewers      |double    | UK Viewership in Millions |
|rating          |double    | Rating |
|duration        |double    | Duration in minutes|

# `writers.csv`

|variable     |class     |description |
|:------------|:---------|:-----------|
|story_number |character | Story number |
|writer       |character | Writer |

# `imdb.csv`

|variable |class     |description |
|:--------|:---------|:-----------|
|season   |integer   | Season number |
|ep_num   |double    |Episode number |
|air_date |character | Air date |
|rating   |double    | Rating|
|rating_n |double    | Number of ratings |
|desc     |character | Episode description |

### Cleaning Script

``` r
library(tidyverse)
library(rvest)

season <- 1

get_imdb <- function(season){
  url <- glue::glue("https://www.imdb.com/title/tt0436992/episodes?season={season}")
  
  raw_html <- read_html(url)
  
  raw_div <- raw_html %>% 
    html_elements("div.list.detail.eplist") %>% 
    html_elements("div.info")
  
  ep_num <- raw_div %>% 
    html_elements("meta") %>% 
    html_attr("content")
  
  air_date <- raw_div %>% 
    html_elements("div.airdate") %>% 
    html_text() %>% 
    str_squish()
  
  ratings <- raw_div %>% 
    html_elements("div.ipl-rating-star.small > span.ipl-rating-star__rating") %>% 
    html_text()
  rate_ct <- raw_div %>% 
    html_elements("div.ipl-rating-star.small > span.ipl-rating-star__total-votes")%>% 
    html_text() %>% 
    str_remove_all("\\(|\\)|,")
  
  descrip <- raw_div %>% 
    html_elements("div.item_description") %>% 
    html_text() %>% 
    str_squish()
  
  tibble(
    season = season,
    ep_num = ep_num,
    air_date = air_date,
    rating = ratings, rating_n = rate_ct, desc = descrip)
  
}

all_season <- 1:12 %>% 
  map_dfr(get_imdb)

clean_season <- all_season %>% 
  type_convert()

clean_season %>% 
  write_csv("2021/2021-11-23/imdb.csv")
```
