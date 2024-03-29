Scrape Games
================
Luke Welsh
07-26-2022

In this Document, I will:

1.  Create functions to scrape MLB pitch-by-pitch data from
    baseball-savant.com, summarize games, and join all game summaries
    together.

2.  Use these functions together to produce one large table with average
    exit velocity for each game.

3.  Add additional statistic which will correct for certain teams simply
    having better batters and thus higher exit velocities on average.

4.  Join MLB data with TV broadcast data to add column with which
    station each game was broadcast to.

Example of Pitch-by-Pitch Game Data.

``` r
pitches = get_pbp_mlb(663046) %>% 
  drop_na(hitData.launchSpeed) %>%
  select(matchup.batter.fullName, batting_team, hitData.launchSpeed)
head(pitches)
```

    ## # A tibble: 6 x 3
    ##   matchup.batter.fullName batting_team    hitData.launchSpeed
    ##   <chr>                   <chr>                         <dbl>
    ## 1 Luke Williams           Miami Marlins                  87.4
    ## 2 Willians Astudillo      Miami Marlins                  84.4
    ## 3 Bryan De La Cruz        Miami Marlins                  93.5
    ## 4 Donovan Solano          Cincinnati Reds                81.1
    ## 5 Kyle Farmer             Cincinnati Reds                88.1
    ## 6 JJ Bleday               Miami Marlins                  88.5

Fuction to break down full game stats into average exit velo by team.

``` r
sum_game = function(game_pk){
  sum_table = get_pbp_mlb(game_pk) %>% 
  drop_na(hitData.launchSpeed) %>% 
  filter(hitData.trajectory != 'bunt_grounder') %>% 
  select(matchup.batter.fullName, batting_team, hitData.launchSpeed, game_date) %>% 
  group_by(batting_team) %>% 
  summarize(velo = mean(hitData.launchSpeed), game = game_pk, date = first(game_date))
  sum_table
}
sum_game(663046)
```

    ## # A tibble: 2 x 4
    ##   batting_team     velo   game date      
    ##   <chr>           <dbl>  <dbl> <chr>     
    ## 1 Cincinnati Reds  88.3 663046 2022-07-25
    ## 2 Miami Marlins    89.2 663046 2022-07-25

Function to combine each team’s average exit velo from each game into
one large tibble.

``` r
join_games = function(game_ids){
  all_games = sum_game(game_ids[1])
  for (i in 2:length(game_ids)){
    all_games = rbind(all_games, sum_game(game_ids[i]))
    if(i %% 10 == 0){
      print(i)
    }
  }
  all_games
}
```

Getting each game’s game\_pk (id number).

Filters are to eliminate preseason games, games going on while the code
runs, and ‘DR’.

‘DR’ means a game that was postponed.

``` r
schedule = mlb_schedule(2022) %>% 
  filter(series_description == "Regular Season") %>% 
  filter(status_abstract_game_state == "Final") %>% 
  filter(status_status_code != 'DR')
completed_games_pk = schedule %>% 
  pull(game_pk)
```

**TAKES A LONG TIME (\~1 hour)** Using previously written function to
join information from every game.

Save resulting tibble as a csv so I don’t have to re-run the big
function between sessions.

``` r
# all_games = join_games(completed_games_pk)
# write.csv(all_games, "all_game_sum.csv")

all_games = read.csv("all_game_sum.csv")
```

Adding 2 additional columns.

First, team\_avg is the corresponding team’s average game exit velocity
over the season.

Secondly, game\_diff is a statistic that finds the difference the team
had from their team\_avg in a specific game. This is because different
teams are, on average, better than others in terms of average exit
velocity. By finding the difference in a single game from their season
average, the column game\_diff is independent of the batting team and,
assuming balls do not travel faster when games are on various
broadcasts, the variation of this statistic should be just due to
randomness.

``` r
all_games = all_games %>% 
  group_by(batting_team) %>% 
  mutate(team_avg = mean(velo)) %>% 
  ungroup() %>% 
  mutate(game_diff = velo - team_avg)
```

Importing independently gathered National TV Schedule

``` r
tv_schedule = read_excel("../juicy/national_tv_list.xlsx") %>% 
  rename(game = game_pk) %>% 
  select('Team 1', 'Team 2', 'TV Provider', 'game')
```

Joining together MLB games with TV schedule.

In essence, adding a column with where each game was broadcast.

``` r
all_games_tv = all_games %>% left_join(tv_schedule, by = 'game', keep = FALSE) %>% 
  select(-'Team 1', -'Team 2')
all_games_tv[is.na(all_games_tv)] = 'Local'
all_games_tv = all_games_tv %>%
  rename(tv_provider = `TV Provider`)
```

Saving data, this time with TV broadcast information added in.

``` r
write.csv(all_games_tv, "all_game_tv.csv")
```
