---
layout: project
title:  "Unsupervised Classification of NBA Positions"
date:   2025-03-04 00:00:00 +0300
categories: jekyll update
image: "/assets/img/post_images/nba-position-classifier.png"
---

Can an NBA player's position be predicted solely from their stats? If so, how accurately?

# Background
The NBA is the world's premier basketball league, and as such, it's always on the forefront of evolution in the sport. Coaches and analysts often say that the game of basketball is becoming - or has become - 'positionless', but is that really true? Is a player's position really as ambiguous as these experts suggest?

# The Data
The dataset for this project is comprised of NBA player statistics for the 2023-2024 NBA basketball season, sourced from [Basketball Reference](https://www.basketball-reference.com/)

Lots of cleaning was required to make the dataset usable, since it was web scraped:

1. Tables with multidimensional data were reduced.
2. Columns not including the player name or target variable were converted into numeric.
3. Duplicate entries for players who switched teams midseason were averaged.
4. Tables were joined.
5. Unneeded columns (rank, team, awards, etc.) were removed.
6. Any column names that included characters that the RF model complained about were modified.
7. A new boolean ‘Starter’ column was created, based on whether or not a player started more games than they came off the bench. This is the same metric that the NBA uses to decide who is eligible for the “Sixth Man of the Year” award, which goes to the best non-starter in the league.

If you're *really* interested in how that works for some reason, the code is below.

<details>
<summary>&#9658; Show data cleaning code (R)</summary>
<pre><code class="language-r">extract_and_process_player_data &lt;- function(year = 2025) {
  library(rvest)
  library(dplyr)
  library(stringr)

  # URLs for Basketball Reference data
  base_url &lt;- &quot;https://www.basketball-reference.com/leagues/&quot;
  urls &lt;- list(
    per_game = paste0(base_url, &quot;NBA_&quot;, year, &quot;_per_game.html&quot;),
    per_poss = paste0(base_url, &quot;NBA_&quot;, year, &quot;_per_poss.html&quot;),
    advanced = paste0(base_url, &quot;NBA_&quot;, year, &quot;_advanced.html&quot;),
    play_by_play = paste0(base_url, &quot;NBA_&quot;, year, &quot;_play-by-play.html&quot;),
    shooting = paste0(base_url, &quot;NBA_&quot;, year, &quot;_shooting.html&quot;),
    adj_shooting = paste0(base_url, &quot;NBA_&quot;, year, &quot;_adj_shooting.html&quot;)
  )

  # Reads tables from URL links
  pergame_table &lt;- read_html(urls$per_game) |&gt; html_element(&quot;table&quot;) |&gt; html_table()
  per100_table &lt;- read_html(urls$per_poss) |&gt; html_element(&quot;table&quot;) |&gt; html_table()
  advanced_table &lt;- read_html(urls$advanced) |&gt; html_element(&quot;table&quot;) |&gt; html_table()
  playbyplay_table &lt;- read_html(urls$play_by_play) |&gt; html_element(&quot;table&quot;) |&gt; html_table()
  shooting_table &lt;- read_html(urls$shooting) |&gt; html_element(&quot;table&quot;) |&gt; html_table()
  adjustedshooting_table &lt;- read_html(urls$adj_shooting) |&gt; html_element(&quot;table&quot;) |&gt; html_table()

  # Fixes tables with multidimensional data
  tidy_dimensions &lt;- function(table) {
    table[1, ] &lt;- as.list(paste0(colnames(table), &quot;_&quot;, table[1, ]))
    colnames(table) &lt;- table[1, ]
    table &lt;- table[-1, ]
    table &lt;- table %&gt;% rename(Player = `_Player`)
    colnames(table) &lt;- str_remove(colnames(table), &quot;^_&quot;)
    return(table)
  }

  playbyplay_table &lt;- tidy_dimensions(playbyplay_table)
  shooting_table &lt;- tidy_dimensions(shooting_table)
  adjustedshooting_table &lt;- tidy_dimensions(adjustedshooting_table)

  # Function to convert every column except 'Player' and 'Pos' into numeric
  convert_except_player_pos &lt;- function(df) {
    df %&gt;%
      mutate(across(
        !any_of(c(&quot;Player&quot;, &quot;Pos&quot;)),
        ~ as.numeric(str_replace_all(., &quot;[^0-9.-]&quot;, &quot;&quot;))
      ))
  }

  # Function to average data for each player across their teams
  # On basketball-reference.com, players who play on multiple teams in the same
  # season will sometimes be represented by multiple rows, depending on the table.
  average_aggregate_data &lt;- function(df) {
    df %&gt;%
      group_by(Player) %&gt;%
      summarize(
        Pos = first(Pos),
        across(where(is.numeric), mean, na.rm = TRUE),
        .groups = &quot;drop&quot;
      )
  }

  pergame_table &lt;- convert_except_player_pos(pergame_table)
  advanced_table &lt;- convert_except_player_pos(advanced_table)
  playbyplay_table &lt;- convert_except_player_pos(playbyplay_table)
  shooting_table &lt;- convert_except_player_pos(shooting_table)
  adjustedshooting_table &lt;- convert_except_player_pos(adjustedshooting_table)

  pergame_table &lt;- average_aggregate_data(pergame_table)
  advanced_table &lt;- average_aggregate_data(advanced_table)
  playbyplay_table &lt;- average_aggregate_data(playbyplay_table)
  shooting_table &lt;- average_aggregate_data(shooting_table)
  adjustedshooting_table &lt;- average_aggregate_data(adjustedshooting_table)

  # Joins tables
  players &lt;- pergame_table %&gt;%
    left_join(advanced_table, by = &quot;Player&quot;) %&gt;%
    left_join(playbyplay_table, by = &quot;Player&quot;) %&gt;%
    left_join(shooting_table, by = &quot;Player&quot;) %&gt;%
    left_join(adjustedshooting_table, by = &quot;Player&quot;) %&gt;%
    select(-ends_with(&quot;.y&quot;)) %&gt;%
    select(-ends_with(&quot;.x.x&quot;)) %&gt;%
    select(-ends_with(&quot;.x.y&quot;))

  # Gets rid of duplicate and/or irrelevant columns
  players &lt;- players %&gt;%
    select(-starts_with(&quot;Rk&quot;),
           -starts_with(&quot;Age&quot;),
           -starts_with(&quot;Team&quot;),
           -starts_with(&quot;Awards&quot;),
           -MP,
           -starts_with(&quot;Position&quot;))

  colnames(players) &lt;- str_remove(colnames(players), &quot;\.x$&quot;)
  players &lt;- players[, !duplicated(names(players))]

  # Removes any characters that the RF model errors on
  colnames(players) &lt;- colnames(players) %&gt;%
    str_replace(&quot;^\\+/-&quot;, &quot;PlusMinus_&quot;) %&gt;%
    str_replace(&quot;^2&quot;, &quot;Two_&quot;) %&gt;%
    str_replace(&quot;^3&quot;, &quot;Three_&quot;) %&gt;%
    str_replace(&quot;%$&quot;, &quot;_pct&quot;) %&gt;%
    str_replace_all(&quot;%&quot;, &quot;pct&quot;) %&gt;%
    str_replace_all(&quot;[/]&quot;, &quot;_&quot;) %&gt;%
    str_replace_all(&quot;\\.&quot;, &quot;_&quot;) %&gt;%
    str_replace_all(&quot;-&quot;, &quot;_&quot;) %&gt;%
    str_replace_all(&quot;\\+&quot;, &quot;_plus&quot;)

  players &lt;- players %&gt;% filter(Player != &quot;League Average&quot;)
  players &lt;- players %&gt;% rename_with(~ str_replace_all(.x, &quot; &quot;, &quot;_&quot;))

  # Creates new variable &quot;Starter&quot;
  players &lt;- players %&gt;%
    mutate(Starter = ifelse(is.na(G), &quot;NA&quot;,
                            ifelse(is.na(GS), &quot;NA&quot;,
                                   ifelse(G - GS &lt; GS, &quot;Starter&quot;, &quot;Non-Starter&quot;))))

  return(players)
}
</code></pre>
</details>

# Exploratory Data Analysis (EDA)
In basketball, the point guard's role is usually to initiate the offense, start plays, and pass well. We see this in the distribution of assists per 100 possessions:

![Violin shart of assists per game, separated by position](/assets/img/post_images/nba-position-eda-1.png)

One thing to note: That singular shooting guard averaging 24 assists per 100 possessions is Marquis Nowell, who checked into 1 game for 4 minutes and had 2 assists. The dataset includes some more outliers like that, including Adam Flagler (14 total minutes played, 13.7 assists per 100 possessions) and JD Davison (39 minutes played, 12.7 assists per 100 possessions). Handling players with low minutes is discussed in [#reflection](#reflection)

*(Also, when I submitted this assignment for class, I said that chart was per-game rather than per 100 possessions, which doesn't make a lot of sense given how many point guards in the distribution were averaging over 10 assists. My professor didn't notice. Sorry Dr. Kleffner, if you see this.)*

This bar schart shows the percentage of field goal attempts taken from beyond the three point-line as opposed to within the arc. It makes sense that the shooting guard position would take the highest percentage of their shots from behind the line, but it's interesting to see that the three makes up about the same percentage of a point guard's shot diet as a power forward. That is probably something that's changed from 20 years ago.

![Three-point range shot distribution, separated by position](/assets/img/post_images/nba-position-eda-2.png)

# Modeling
The model used in this project is called a **random forest**. A **random forest** is made up of **decision trees**, which are pretty simple to understand. A decision tree is a sort of flowchart that makes a classification decision by asking a series of yes/no questions. A decision tree for basketball statistics might look  like:

```
Is assists per 100 possessions > 7?
├── Yes → Is three-point attempt rate > 45%?
│         ├── Yes → Shooting Guard
│         └── No  → Point Guard
└── No  → Is blocks per 100 possessions > 2?
          ├── Yes → Center
          └── No  → Power Forward / Small Forward
```

Decision trees are intuitive, but they have a well-known flaw: they tend to **overfit**. A single tree memorizes the training data too well and struggles to generalize. It loses nuance - Josh Hart is a small forward who grabs a *ton* of rebounds, but if a decision tree set a strict cutoff like "if a player averages 8 rebounds, they're a center or power forward" it would miss Hart.

A Random Forest fixes this issue by building hundreds of trees, each one trained on a random sample of the data and allowed to look at only a random subset of the available stats. Each tree votes on what position a player is, and the whatever position wins the majority of votes wins.

The "random" in Random Forest refers to two sources of randomness:
1. **Bootstrapping** — each tree trains on a random sample (with replacement) of the players in the dataset.
2. **Feature sampling** — at each split, the tree can only consider a random subset of stats, rather than all of them at once.

This randomness forces the trees to be different from each other, which is what makes the forest powerful.

# Evaluation
I was a bit surprised by the model underperforming compared to my expectations. I figured that with so many statistics to go off of, the model would be able to classify players at a very high rate.

However, while the model wasn't the strongest, it certianly wasn't weak either. The Confusion Matrix below bears a sharper orange up the diagonal .45 than to the corners, but there's a good amount of error.

![Confusion matrix showing aggregated predictions vs actual predictions for 2023-2024 season](/assets/img/post_images/nba-position-results-1.png)

Aside from just accuracy (whether the model got a pick right or wrong), we can also measure how *far off* the model was on average. Treating basketball positions as ordinal categorical data points, we can use the Quadratic Weighted Kappa (QWK) to measure how far off the model was on average.

- Mean Model Accuracy: 46.69%
- Mean Quadratic Weighted Kappa: 0.7057

We can also look at which variables the model decided was most important to guessing which position a player played. There are some interesting results - rebound-related statistics make up 5/10 of the most important variables, whereas shooting-related statistics only make up 4/10.

The data certainly seems to suggest a somewhat "positionless" game, but there's an even better way to check.

# 20-Year Comparison

When the exact same analysis is run on data from 2003-2004, the new model actually performs better - meaning it's better at predicting which position a player plays based on their stats.

![Confusion matrix showing aggregated predictions vs actual predictions for 2003-2004 season](/assets/img/post_images/nba-position-results-3.png)

# Conclusion & App
This analysis actually lends a lot of credit to the idea that basketball *has* become more positionless over the years - if it's more difficult to guess a player's position now than 20 years ago, it means that more players are being asked to play roles typically filled by other positions.

I could have (probably should have) added a minutes requirement, since some players with few minutes but high per-36 and per-100-possessions stats probably skew the model.

The analysis above is also available as a Shiny app, which you can access at this link to perform the analysis yourself: [NBA Player Analysis and Bootstrapping](https://93q1q1-james0brainard.shinyapps.io/nba-position-classifier/)