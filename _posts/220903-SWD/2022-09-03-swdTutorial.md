---
layout: post
title:  "Known to novel"
author: Seba
categories: [ R, Visualization, Maps ]
---

# Known to novel

A new month means a new [challenge](https://community.storytellingwithdata.com/challenges/sep-2022-known-to-novel) over at the blog of *Storytelling with Data*.
The task was to create a data visualization by going from known to unknown graphs. But in addition explain the process in a way an audience could follow along.
Some interesting data comes with the *crimedata* pacakge and I though about different possibilities to visualize the data. I did enjoy the this process a lot
and decided to write a quick tutorial since I encountered some difficulties in producing the final plots. 

# Setup

As mentioned, I used data from the *crimedata* package, as well as the packages *tidyverse* and *lubridate* for data wrangling. For producing the maps and plots
I used *ggplot*, *ggpubr*, *ggmap*, *sf*, and *scatterpie*. Load the packages individually or use the awesome package *pacman* to install and load packages simultaneously.

```r
library(crimedata)
library(tidyverse)
library(lubridate)
library(ggpubr)
library(ggmap)
library(sf)
library(scatterpie)

# Or with the function p_load() {pacman}
pacman::p_load(crimedata, tidyverse, lubridate, ggpubr, ggmap, sf, scatterplot)
```

# Data preparation

After loading all packages we can begin by preparing our data. I used the **homicides14** dataset, which contains homicide records from the year 2015 of nine
large US cities. I decided to use the data from New York City and to investigate the distribution of homicides between day and night. I defined daytime
as the time between 8 AM and 10 PM. A little bit arbitrary and there is an uneven split between day (= 14 h) and night (= 8 h), but check for yourself how
the final visualization changes if different splits or more classes (e.g., afternoon, evening, ...) are used! All the steps can easily be done using the
pipe operator.

```r
dt <- homicides15 %>%
    
    # Filter rows
    filter(grepl("New York", city_name)) %>%
    
    # Select columns
    select(city_name, offense_type, date_single, longitude, latitude) %>%
    
    # Add category for daytime
    mutate(daytime = case_when(
        hour(date_single) >= 8 & hour(date_single) < 22 ~ "Day",
        TRUE ~ "Night"
    ))
```

