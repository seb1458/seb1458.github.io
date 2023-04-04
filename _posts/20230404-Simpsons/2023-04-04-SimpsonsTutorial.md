---
layout: post
title: "Scrape IMDB with R"
author: Seba
categories: [ R, Visualization, Scraping ]
---

Lately I've been on Reddit, more specifically r/dataisbeautiful, more often. I found an [older post](https://www.reddit.com/r/dataisbeautiful/comments/uc6d3w/oc_the_simpsons_episode_and_season_data_episode/) visualizing the IMDB ratings of *The Simpsons*, one of my favorite shows when I was younger. This looks nice, but in order to get all the ratings from each episode we have to do a lot of clicking and scribbling. Or maybe not? There are several ways of getting data into R and one way which can be handy is scraping data from web pages. In this tutorial I am going to show you how to do that and afterwards we are going to reproduce the plot from the Reddit post.

# Scraping the data from IMDB

We use **rvest** for scraping as it can be included nicely in a **tidyverse** workflow. We start by scraping the ratings from the first season and then build towards scraping the ratings of all season. Using the functions of **rvest**, we can access the HTML code of an URL and extract the information via so called *nodes*. What is a node you ask? Why don't we ask a total greenhorn in HTML coding? HTML is basically the code behind a webpage and the textual representation of said page. This code consists of so called *tags*, e.g. something like ```<title>...</title>```, which is the title of a homepage. We can access this information by using ```html_node()``` from the **rvest** package. Accessing the information is quiet simple, but cleaning the extracted information can be tricky. For example, a webpage has one title but there are several text bodies (```<body>...</body>```), so you have to clean you data carefully after the extraction. Okay, but how do I find the tags? That is actually easy. Go to the desired homepage, look at the source code and search the textfile for your desired information and the respective tags (e.g., "rating", "8.1", ...). For example, we can find the ratings for each episode of the Simpsons in the tag ".ipl-rating-star__rating" or the name of the epsiode in the tag "strong". Let's look at some code.

```r
# Packages
library(rvest)
library(tidyverse)

# Define URL for The Simpsons season 1
url <- "https://www.imdb.com/title/tt0096697/episodes?season=1"

# Read the HTML file
imdb <- read_html(url)

# Extract information at desired nodes
imdb_epRate <- html_nodes(imdb, ".ipl-rating-star__rating")
imdb_title <- html_nodes(imdb, "strong")

# Convert to character vectors
rating_raw <- html_text(imdb_epRate)
title_raw <- html_text(imdb_title)
```

As you can see, we have a lot of undesired information. We can clean the scraped ratings, by just keeping all entries with a decimal point (the actual ratings), and then using the number of ratings to subset the vector containing the titles.

```r
# Subset both vectors
rating_ep <- rating_raw[grepl("\\.", rating_raw)]
title_ep <- title_raw[1:length(rating_ep)]
```

At the end we have the titles and ratings for each episode of the first season of the Simpsons. We can use a for-loop to extract the information in the same way for all 34 seasons of the simpsons. We can add a few more lines to store this information in a table.

```r
# Create empty tibble
tbl <- tibble()

# For loop
for (i in 1:34) {
    
    # Define URL
    url <- paste0("https://www.imdb.com/title/tt0096697/episodes?season=", i)
    
    # Read HTML from defined URL
    imdb <- read_html(url)
    
    # Extract information at desired nodes
    imdb_epRate <- html_nodes(imdb, ".ipl-rating-star__rating")
    imdb_title <- html_nodes(imdb, "strong")
    
    # Episode ratings: convert to text and keep entries with a decimal point
    rating_raw <- html_text(imdb_epRate)
    rating_ep <- rating_raw[grepl("\\.", rating_raw)]
    
    # Episode titles: convert to text and subset using the number of episodes
    title_raw <- html_text(imdb_title)
    title_ep <- title_raw[1:length(rating_ep)]
    
    # Build table
    temp_tbl <- tibble(season = as.factor(i),
                       ep_no = 1:length(rating_ep),
                       ep_title = title_ep,
                       ep_rating = as.numeric(rating_ep))
    
    # Bind into final tibble
    tbl <- bind_rows(tbl, temp_tbl)
}
```

# Cleaning the data

We already could use our scraped data to produce plots. But I wanted to add a category based on the rating values. First, we use ```case_when()``` from **tidyverse** and note that the categories for this example were chosen arbitrarily. There are many methods to derive classes (e.g., using a distribution of the ratings). Next, we use ```fct_relevel()``` from **forcats** to order the categories in a logical order. The last step is not necessary but it is easier to understand the plot if e.g., the categories "Good", "Medium", and "Bad" are plotted in a logical and not by default in an alphabetical order.

```r
# Packages
library(forcast)

# Setup desired factor order
category_order <- c("Great", "Good", "Okay", "Weak", "Bad")

# Prepare scraped data
tbl <- tbl %>%
    
    # Add category based on rating
    mutate(ep_ratingCat = case_when(
        ep_rating < 4.5 ~ "Bad",
        ep_rating >= 4.5 & ep_rating < 6.0 ~ "Weak",
        ep_rating >= 6.0 & ep_rating < 7.5 ~ "Okay",
        ep_rating >= 7.5 & ep_rating < 9.0 ~ "Good",
        ep_rating >= 9.0 ~ "Great",
        
    )) %>%
    
    # Order categories according to our definition
    mutate(ep_ratingCat = fct_relevel(ep_ratingCat, category_order))
```

# Working with the data

There are many great tutorials on how to build plots using **ggplot2**. And the strength comes from their versatility and the near endless customizations. If you are looking for inspiration and hacks, check out this [page](https://github.com/teunbrand/ggplot_tricks) with some helpful tricks to make plotting more convenient. We will build the plot step by step, and we start by setting up the theme, fonts, labels at the beginnig. This way we can focus on the plotting of the data afterwards.

```r
# Setup theme, background color, text elements, ...
theme_set(theme_minimal())

my_theme <- theme(

    # Add a title with a Simpson style (needs to be downloaded manually)
    plot.title = element_text(family = "Simpsonfont", size = 25),
    
    # Add a background in Simpsons yellow to remain on theme
    plot.background = element_rect(fill = "#ffd90f"),
    
    # Bold and black font for all axis and legend elements. I did not use the Simpsons font as it becomes hard to read at lower font sizes.
    axis.text = element_text(face = "bold", color = "black"),
    axis.title = element_text(face = "bold", vjust = ),
    legend.text = element_text(face = "bold"),
    legend.title = element_text(face = "bold"),
    
    # Color the grid lines to make the plot less crowded
    panel.grid.major = element_line(color = "gray35"),
    panel.grid.minor = element_blank())

# Setup color palette using RColorBrewer
library(RColorBrewer)
my_pal <- rev(brewer.pal(5, "RdYlGn"))
```

After the setup we can finally plot our data! We will use ```geom_tile()``` to build a heatmap-like plot in addition to some niche functions to reproduce the plot mentioned in the beginning.

```r
# Basic plot. We use coord_fixed() to yield squares instead of rectangles with geom_tile()
ggplot(tbl, aes(x = season, y = ep_no, fill = ep_ratingCat)) +
    geom_tile(color = "white") +
    coord_fixed() +

    # We want to have the x-axis on top with individual breaks. Therefore, we also have to adjust ( reverse) the y-axis
    scale_x_continuous(breaks = c(1, 5, 10, 15, 20, 25, 30, 34),
                       position = "top") +
    scale_y_reverse(breaks = c(1, 5, 10, 15, 20, 25)) +
    
    # We add the rating value inside the squares by using geom_text()
    geom_text(aes(label = ep_rating), color = "black", size = 2) +
    
    # Here, we add the defined color palette and also modify the title of the legend
    scale_fill_manual(name = "Rating Category", values = pal) +
    
    # We add a title, a caption with our data source, and axis labels
    labs(title = "Rating of The Simpsons", 
         caption = "\nData: Internet Movie Database",
         x = "Season\n",
         y = "Episode\n")
    
    # And finally we add our defined theme
    my_theme
```

<figure>
<img src="https://seb1458.github.io/assets/img/230404-Simpsons/simpsonsRating.png" style="display: block; margin: auto;" />
<figcaption>Fig 1. Evolution of The Simpsons episodes ranking.</figcaption>
</figure>

An we are done! We scraped data from IMDB and plotted it in a cool way. We can save the plot using ```ggsave()``` or the GUI of RStudio.

# Final thoughts

We scraped data from IMDB, cleaned the data and produced a nice visualization. The basics are not that difficult but remember that scraping also has its limits and sometimes it is easier to access data in another way. Usually, a work around for complex scrapings can be found (such as putting in a user names and password with **RSelenium** package) but plan ahead and consider if it makes sense to scrape data.
