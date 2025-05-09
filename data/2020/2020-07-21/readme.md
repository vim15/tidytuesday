![Pets](https://images.unsplash.com/photo-1547623542-de3ff5941ddb?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1350&q=80)

# Australian Pets

The data this week comes from the [RSPCA](https://www.rspca.org.au/what-we-do/our-role-caring-animals/annual-statistics), [Townsville Animal Complaints](https://data.gov.au/data/dataset/animal-complaints) and [Brisbane Open Data - Animal Complaints](https://www.data.brisbane.qld.gov.au/data/dataset/96bec69c-6170-4ef0-93f1-eda279149b97).

h/t to [Georgios Karamanis](https://github.com/rfordatascience/tidytuesday/issues/56) for suggesting part of this data.

> The RSPCA is Australia's oldest, largest and most trusted animal welfare organisation. With this
privileged position comes great responsibility. This year we received1 124,146 animals into our
animal shelters and adoption centres across the country.  
>
> With a great deal of effort from RSPCAs all over the country, adoption and reclaiming rates nationally
have been increasing over time and significant improvements in the outcomes for cats and dogs
(including kittens and puppies) have been achieved. This can be attributed to the introduction of
new approaches and programs to increase the number of animals adopted and reunited with their
owners.

For general data on the states/regions of Australian with population as of Dec 2019 - [Wikipedia](https://en.wikipedia.org/wiki/States_and_territories_of_Australia).

There's a *remarkable* amount of possible data cleaning/aggregation here. The `animal_outcomes` dataset is pretty much ready to go, although you could pivot it longer to stack the regions. If you want to go really far down the rabbit hole, check out the cleaning script and see if you can improve upon it or approach things a different way. Note the data came from PDFs and I had to do a lot of manual spot-checks. Some fun `dplyr::update_rows()` which saved me a LOT of time.

The `brisbane_complaints` dataset has an interesting structure. No dates were reported within a dataset, so we had to add them from the file names within a `purrr` call. This cleaning step is pretty realistic! I left the data_range messy so you can further clean it up.

Note that there is state-level data from the RSPCA, but only two cities for the animal complaints. Up to you if you want to extrapolate between the data, but I'm not confident it will be meaningful. 

The RSPCA Report (lots of graphs to recreate) - [RSPCA Report](https://www.rspca.org.au/sites/default/files/RSPCA%20Report%20on%20animal%20outcomes%202018-2019.pdf).

Journal article - [A Retrospective Analysis of Complaints to RSPCA Queensland, Australia, about Dog Welfare](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6562769/).

### Get the data here

```r
# Get the Data

# Read in with tidytuesdayR package 
# Install from CRAN via: install.packages("tidytuesdayR")
# This loads the readme and all the datasets for the week of interest

# Either ISO-8601 date or year/week works!

tuesdata <- tidytuesdayR::tt_load('2020-07-21')
tuesdata <- tidytuesdayR::tt_load(2020, week = 30)

animal_outcomes <- tuesdata$animal_outcomes

# Or read in the data manually

animal_outcomes <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2020/2020-07-21/animal_outcomes.csv')
animal_complaints <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2020/2020-07-21/animal_complaints.csv')
brisbane_complaints <- readr::read_csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/main/data/2020/2020-07-21/brisbane_complaints.csv')

```
### Data Dictionary

# `animal_outcomes.csv`

|variable    |class     |description |
|:-----------|:---------|:-----------|
|year        |double    | Full year |
|animal_type |character | Animal type (horse, wildlife, dog, cat, etc) |
|outcome     |character | Animal outcome - euthanized, released, rehomed, etc |
|ACT         |double    | ACT - Australian Capital Territory |
|NSW         |double    | New South Wales |
|NT          |double    | Northern Territory |
|QLD         |double    | Queensland |
|SA          |double    | South Australia |
|TAS         |double    | Tasmania |
|VIC         |double    | Victoria |
|WA          |double    | Western Australian |
|Total       |double    | Australian Total|

# `animal_complaints.csv`

|variable           |class     |description |
|:------------------|:---------|:-----------|
|Animal Type        |character | Animal Type |
|Complaint Type     |character | Complaint type |
|Date Received      |character | Date received (Month - year) |
|Suburb             |character | Suburb/region |
|Electoral Division |character | Electoral Division |

# `brisbane_complaints.csv`

|variable           |class     |description |
|:------------------|:---------|:-----------|
|nature             |character | Nature of complaints (animal) |
|animal_type        |character | Animal type |
|category           |character | Category of complaint |
|suburb             |character | Suburb where reported |
|date_range         |character | Date range (typically 1 quarter + year) |
|responsible_office |character | Responsible office for the complaint |
|city               |character | City (Brisbane) |

### Cleaning Script

```r

library(tidyverse)
library(pdftools)
library(rvest)
library(glue)

# Brisbane data for complaints:

# Source: https://www.data.brisbane.qld.gov.au/data/dataset/96bec69c-6170-4ef0-93f1-eda279149b97

all_complaints <- list.files("2020/2020-07-21/raw_csv") %>% 
  map_dfr(.f = function(x){
    read_csv(file = paste0("2020/2020-07-21/raw_csv/", x)) %>% 
      mutate(date_range = x)
  })

brisbane_complaints <- all_complaints %>% 
  mutate(date_range = str_remove(date_range, "animal-compliance-|cars-bis-open-data-animal-related-complaints-")) %>% 
  set_names(nm= c("nature", "animal_type", "category", "suburb", "date_range", "responsible_office")) %>% 
  mutate(city = "Brisbane")

brisbane_complaints %>% write_csv("2020/2020-07-21/brisbane_complaints.csv")

library(tidyverse)
library(pdftools)
library(rvest)
library(glue)

# Get all urls for PDFs ---------------------------------------------------

raw_pca <- "https://www.rspca.org.au/what-we-do/our-role-caring-animals/annual-statistics" %>%
  read_html()

all_urls <- raw_pca %>%
  html_node("body > div:nth-child(5) > div.node.node-full-page > div.paragraphs-items.paragraphs-items-field-shared-sections.paragraphs-items-field-shared-sections-full.paragraphs-items-full > div > div > div > div > div > div:nth-child(6)") %>%
  html_nodes("p > a") %>%
  html_attr("href") %>%
  enframe() %>%
  arrange(desc(name)) %>%
  mutate(name = c(1999:2015, 2017)) %>%
  add_row(name = 2018, value = "https://www.rspca.org.au/sites/default/files/RSPCA%20Australia%20Annual%20Statistics%20final%202018-2019.pdf") %>%
  set_names(nm = c("year", "url"))

# Download all the files
save_rspca <- function(year, url){
  download.file(url, destfile = glue::glue("rspca_{year}.pdf"))
}

# Iterate across the files and save
all_urls %>% 
  pwalk(save_rspca)

### Function to get data
get_animals <- function(year, url) {
  url_rspca <- url
  
  # Read PDF as character strings
  raw_pdf <- pdftools::pdf_text(url_rspca)
  
  # Get table number by page
  page_num_df <- raw_pdf %>%
    enframe() %>%
    mutate(value = str_trim(value) %>%
             word(1, 2))
  
  # Add logic for getting the right table number
  page_num_pull <- page_num_df %>%
    filter(str_detect(value, if_else(year < 2007, "Table 3", "Table 4"))) %>%
    pull(name)
  
  # More logic because someone in 2011 decided to further wrap tables :facepalm:
  page_num <- if_else(year == 2011, list(5), list(page_num_pull))[[1]]
  
  # Tables can span up to 2 additional pages
  # Combine these pages and split into rows by new lines
  raw_text <- raw_pdf[c(page_num:(page_num + 2))] %>%
    str_split("\n") %>%
    flatten_chr()
  
  # Find and limit to start/end of table
  table_start <- stringr::str_which(raw_text, "Dogs")
  table_end <- stringr::str_which(raw_text, "Total animals|Total received|Total Animals")
  
  table_end <- table_end[[length(table_end)]]
  
  # Trim table of extra whitespace and limit to right rows
  table_trimmed <- raw_text[c(table_start:table_end)] %>% str_trim()
  
  # squish table together and replace excess whitespace with a '|'
  squished_table <- str_replace_all(table_trimmed, "\\s{2,}", "|") %>%
    str_remove_all(",")
  
  raw_df <- enframe(squished_table)
  
  # separate the columns
  animal_df <- raw_df %>%
    separate(
      value,
      into = c(
        "outcome",
        "ACT",
        "NSW",
        "NT",
        "QLD",
        "SA",
        "TAS",
        "VIC",
        "WA",
        "Total"
      ),
      sep = "\\|"
    ) %>%
    # convert columns to double
    mutate(across(c(ACT:Total), as.double)) %>%
    mutate(
      year = year,
      # Clean up the outcome column
      outcome = str_to_title(outcome) %>% str_remove("[:digit:]+")
    ) %>%
    mutate(
      # create animal_type column
      animal_type = if_else(
        str_detect(outcome, paste(c("Dogs", "Cats", "Horses", "Livestock", "Other animals", "Other Animals", "Wildlife"), collapse = "|")),
        outcome,
        NA_character_
      ),
      animal_type = str_to_title(animal_type)
    ) %>%
    # fill down the animal type column
    fill(animal_type)
  
  
  animal_df
}

# Get all the data --------------------------------------------------------

# Note you could do some of this in bulk with purrr, but since I had to
# do so much extra cleaning outside of the core function I decided to 
# just do it manually and merge at the end

data_1999 <- get_animals(1999, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics1999-2000.pdf")
data_2000 <- get_animals(2000, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202000%20-%202001.pdf")
data_2001 <- get_animals(2001, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202001%20-%202002.pdf")
data_2002 <- get_animals(2002, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202002-2003.pdf")
data_2003 <- get_animals(2003, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202003%20-%202004.pdf") %>%
  filter(!is.na(Total))
data_2004 <- get_animals(2004, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202004%20-%202005.pdf") %>%
  filter(!is.na(Total))
data_2005 <- get_animals(2005, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202005%20-%202006.pdf") %>%
  rows_update(
    tibble(name = 5, NT = NA, QLD = 456, SA = 97, TAS = 79, VIC = 472, WA = 133, Total = 1951)
  ) %>%
  rows_update(
    tibble(name = 10, SA = NA, TAS = 8, VIC = 100, WA = 0, Total = 1730)
  ) %>%
  rows_update(
    tibble(name = 7, NT = NA, QLD = 47, SA = 451, TAS = 127, VIC = 0, WA = 0, Total = 1015)
  ) %>%
  rows_update(
    tibble(
      name = c(18, 19), NT = c(NA, NA), QLD = c(474, 21), SA = c(96, 0),
      TAS = c(52, 300), VIC = c(218, 829), WA = c(52, 0), Total = c(1336, 1168)
    )
  ) %>%
  rows_update(
    tibble(
      name = c(60:66), NT = c(rep(NA, 7)),
      QLD = c(54, 694, 80, 0, 645, 97, 1570),
      SA = c(10, 108, 2, 0, 222, NA, 342),
      TAS = c(5, 95, 1, 0, 11, 2, 114),
      VIC = c(47, 329, 76, 0, 373, 20, 845),
      WA = c(0, 6, 0, 0, 1, 0, 7),
      Total = c(162, 1594, 224, 260, 2507, 481, 5228)
    )
  ) %>%
  filter(!is.na(Total))

data_2006 <- get_animals(2006, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202006%20-%202007.pdf") %>%
  rows_update(
    tibble(name = 7, TAS = NA, VIC = 153, WA = 5, Total = 2087)
  ) %>%
  rows_update(
    tibble(name = 32, NT = NA, QLD = 30, SA = 5, TAS = 7, VIC = 42, WA = 0, Total = 95)
  )

# Note decided to use data_2006_upd instead of overwriting
# One of the columns got dropped in parsing step, so I used it to
# update rows in place

data_2006_upd <- data_2006 %>%
  rows_update(
    data_2006 %>%
      filter(name %in% c(44:50, 53:60)) %>%
      select(-ACT) %>%
      set_names(nm = c(
        "name", "outcome", "ACT", "NSW", "NT", "QLD",
        "SA", "TAS", "VIC", "WA", "year", "animal_type"
      )) %>%
      mutate(Total = c(
        1893, 142, 1502, 6035, 1318, 10890, 11051,
        167, 1631, 169, 186, 2291, 227, 4671, 5228
      ), .after = WA) %>%
      mutate(
        outcome = c(
          "Released", "In Stock", "Transferred", "Euthanased", "Other", "Total", "Last Year's Total",
          "Reclaimed", "Rehomed", "In Stock", "Transferred", "Euthanased", "Other", "Total", "Last Year's Total"
        ),
        animal_type = c(rep("Wildlife", 7), rep("Other Animals", 8))
      )
  ) %>%
  filter(!is.na(Total))

data_2007 <- get_animals(2007, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20national%20Statistics%202007-2008.pdf") %>%
  filter(!is.na(Total))

data_2008 <- get_animals(2008, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202008%20-%202009.pdf") %>%
  rows_update(
    tibble(
      name = c(2, 3, 8, 16),
      WA = c(621, 1045, 248, 958),
      Total = c(22896, 19236, 22085, 19666)
    )
  ) %>%
  rows_update(
    tibble(
      name = 19,
      NT = 212, QLD = 11045, SA = 2712, TAS = 1974, VIC = 9801, WA = 99, Total = 39495
    )
  ) %>%
  filter(!is.na(Total))

data_2009 <- get_animals(2009, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202009%20-%202010.pdf") %>%
  filter(!is.na(Total))

data_2010 <- get_animals(2010, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202010%20-%202011.pdf") %>%
  filter(!is.na(Total))

data_2011 <- get_animals(2011, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202011-2012.pdf") %>%
  filter(!is.na(Total))

data_2012 <- get_animals(2012, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA%20Australia%20National%20Statistics%202012%20-%202013.pdf") %>%
  filter(!is.na(Total))

data_2013 <- get_animals(2013, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA_Australia-Annual_Statistics_2013-2014.pdf") %>%
  filter(!is.na(Total))

data_2014 <- get_animals(2014, "https://www.rspca.org.au/sites/default/files/website/The-facts/Statistics/RSPCA_Australia-Annual_Statistics_2014-2015.pdf") %>%
  filter(!is.na(Total))

data_2015 <- get_animals(2015, "https://www.rspca.org.au/sites/default/files/RSPCA%20Australia%20Annual%20Statistics%202015-2016%20.pdf") %>%
  rows_update(
    tibble(
      name = 62, ACT = 581, NSW = 1361, NT = 34, QLD = 720, SA = 449, TAS = 191, VIC = 716, WA = 38, Total = 4090
    )
  ) %>%
  filter(!is.na(Total))

data_2016 <- get_animals(2016, "https://www.rspca.org.au/sites/default/files/RSPCA%20Australia%20Annual%20Statistics%20final%202016-2017.pdf") %>% 
  filter(!is.na(Total))

data_2017 <- get_animals(2017, "https://www.rspca.org.au/sites/default/files/RSPCA%20Australia%20Annual%20Statistics%202017-2018.pdf") %>%
  filter(!is.na(Total))

data_2018 <- get_animals(2018, "https://www.rspca.org.au/sites/default/files/RSPCA%20Australia%20Annual%20Statistics%20final%202018-2019.pdf") %>%
  filter(!is.na(Total))

# Combine all the cleaned data by year!
# note that 2016 was not reported. :sad:
# I would recommend imputing if you want

comb_data <- bind_rows(
  list(
    data_1999, data_2000, data_2001, data_2002, data_2003, data_2004, data_2005,
    data_2006_upd, data_2007, data_2008, data_2009, data_2010, data_2011, 
    data_2012, data_2013, data_2014, data_2015, data_2016, data_2017, data_2018
  )
)

# clean the final data
clean_data <- comb_data %>%
  filter(outcome %in% c(
    "Euthanased", "Other", "Rehomed", "Transferred",
    "Reclaimed", "Currently In Care", "In Stock",
    "Released", "In Our Care"
  )) %>%
  mutate(
    outcome = case_when(
      outcome == "In Our Care" ~ "Currently In Care",
      outcome == "Euthanased" ~ "Euthanized",
      TRUE ~ outcome
    )
  ) %>%
  select(year, animal_type, outcome, ACT:Total)

# Sanity checking
clean_data %>% ggplot(aes(x = year, y = Total, color = outcome)) +
  geom_line() +
  facet_wrap(~animal_type, scales = "free_y")

# Sanity checking
clean_data %>%
  filter(outcome %in% c("Euthanized", "Rehomed"), animal_type != "Wildlife") %>%
  ggplot(aes(x = year, y = Total, fill = outcome)) +
  geom_col(position = "fill") +
  facet_wrap(~animal_type)

# Sanity checking
clean_data %>%
  count(animal_type, sort = T)

# Sanity checking Wildlife being an odd number
# It's because first 3 years only had 3 outcomes

clean_data %>%
  filter(animal_type == "Wildlife") %>%
  count(year)

clean_data %>% 
  write_csv("2020/2020-07-21/animal_outcomes.csv")


```
