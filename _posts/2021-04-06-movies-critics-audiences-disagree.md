---
layout: post
title: "What movies do critics and audiences disagree on? - An analysis in R"
tags:
    - R
---

*Rotten Tomatoes* is a film aggregator site that deems movies to be
"fresh" or "rotten" based on the reviews of critics. However, critics
and audiences aren't always in agreement about what makes a good movie
and this has been the source of many articles over the years.

In this short post, I'll be analysing the data for myself using data
from the *Rotten Tomatoes* site. The focus will be on data
pre-processing and visualisation. I typically perform data analysis in
Python/pandas. However, I'm now trying to build up skills in R and will
be using R for this project.

Let's get started!

Data
====

I'll be using [this Rotten Tomatoes dataset from
Kaggle](https://www.kaggle.com/stefanoleone992/rotten-tomatoes-movies-and-critic-reviews-dataset).

    rt_movies <- read_csv('rotten_tomatoes_movies.csv')

    nrow(rt_movies)

    ## [1] 17712

There's less than 18k rows in this dataset but there's over 22k movies
on the *Rotten Tomatoes* site, so we seem to be missing some entries.
Unfortunately the Kaggle webpage for this dataset doesn't provide too
much detail in this regard - we just know that the data was scraped in
October 2020. It's not clear whether there are just random movies
missing or whether some other piece of logic has resulted in movies
being missing.

Usually I would spend a large chunk of my time investigating
discrepancies like this. However, as this piece of data analysis is just
for fun, I won't worry about it too much. I'll just have to accept that
we haven't got an exhaustive list.

Data pre-processing
===================

This dataset includes both feature films and documentaries. I want to
focus on feature films - there are a lot of political documentaries that
have polarising reviews on Rotten Tomatoes, but that's not the focus of
this post.

So I will **remove movies where "Documentary" is the primary genre**.
The dataset has a column called `genres` which contains a list of
comma-separated genres, with the first genre being the primary one. I
use this to create a new column called `primary_genre` and filter
appropriately.

    rt_movies <- rt_movies %>%
      mutate(primary_genre = as.character(map(str_split(genres, ','), 1)))  %>%
      filter(primary_genre != 'Documentary') %>%
      drop_na(primary_genre)

I've also decided to **take the top 5,000 movies** according to
`audience_count`, the number of audience members that reviewed the
movie. This means that a few ratings won't sway the score, and we should
be able to recognise most of the movies. I save this to a new dataframe
called `top_rt_movies`.

    top_rt_movies <- rt_movies %>%
      drop_na(audience_count, tomatometer_count) %>%
      arrange(desc(audience_count, tomatometer_count)) %>%
      select(rotten_tomatoes_link, movie_title, content_rating, genres, original_release_date, tomatometer_rating, tomatometer_count, audience_rating, audience_count, movie_info, primary_genre) %>%
      head(5000)

    summary(top_rt_movies$audience_count)

    ##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
    ##    17017    35285    64772   492367   207166 35797635

    summary(top_rt_movies$tomatometer_count)

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##     5.0    46.0    99.0   113.6   162.0   574.0

All of the movies in `top_rt_movies` have over 17,000 audience reviews
and at least 5 critic reviews.

Exploratory data analysis
=========================

Now that I've prepared the dataset, I can do some exploratory data
analysis.

Decade distribution
-------------------

Let's quickly look at movie decades. Note that there are 35 rows missing
a value for `original_release_date`, and these will be omitted from the
following plot.

    top_rt_movies <- top_rt_movies %>%
      mutate(original_release_year = year(original_release_date)) %>%
      # Create a new column that gives us the decade the movie was released
      mutate(original_release_decade = original_release_year - (original_release_year %% 10))

    ggplot(data = top_rt_movies %>% drop_na(original_release_decade)) +
      geom_bar(mapping = aes(x = as.character(original_release_decade))) +
      labs(x = "Decade Movie Released", y = "Number of Movies", title = "Decade Distribution of Movies")

![Decade distribution of movies]({{ site.url }}{{ site.baseurl }}/images/movies-decade-distribution.png)

The majority of movies in our dataset are from the 21st century,
however, we do also have older movies, particularly from the 80s and
90s.

Score distribution
------------------

Let's also look at the distribution of scores. The `tomatometer_rating`
variable gives the average critic score and the `audience_rating` gives
the average audience score. Both scores are on a scale from 0 to 100.

    scores <- top_rt_movies %>%
      select(tomatometer_rating, audience_rating) %>%
      rename(critic = tomatometer_rating, audience = audience_rating)

    scores <- melt(scores)

    ## No id variables; using all as measure variables

    ggplot(data = scores, aes(x=value, fill=variable)) + geom_density(alpha=0.25) + 
      labs(x = 'Score', y = 'Density', title = 'Density plots of scores', fill = 'Type of score')

![Density plot of scores]({{ site.url }}{{ site.baseurl }}/images/movies-score-density.png)

    summary(top_rt_movies$tomatometer_rating)

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##     0.0    32.0    60.0    56.7    82.0   100.0

    summary(top_rt_movies$audience_rating)

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##   10.00   49.00   65.00   63.64   80.00   99.00

    ggplot(data = scores, aes(variable, value)) +
      geom_boxplot() +
      labs(x = "", y = 'Score', title = "Boxplots of scores")

![Boxplots of scores]({{ site.url }}{{ site.baseurl }}/images/movies-score-boxplots.png)

The density plots and boxplots show that the critic and audiences scores
follow different distributions.

The critic scores have a much wider distribution. It also looks like
critic scores take on more extreme values than audience scores.

On the other hand, audience scores have a narrower distribution. Very
few films achieve an audience score below 25%. Similarly, not many
movies have an average audience score above 90%. In fact, out of the
5,000 movies considered, only 317 achieved an audience score above 90%,
but 644 achieved a critic score above 90%.

It's important to keep these differences in mind as we continue our
analysis.

What movies do critics and audiences disagree on?
=================================================

Now that we've got our data in a good state, let's look at the
difference between critics and audiences.

We'll define a new variable called **critical disconnect** as
critical\_disconnect = tomatometer\_rating - audience\_rating
. A positive value means the critics rated the movie higher than
audiences.

    # Define column critical_disconnect
    # Also create a column movie_title_and_year which will be used to make labels on plots
    top_rt_movies <- top_rt_movies %>%
      mutate(critical_disconnect = tomatometer_rating - audience_rating) %>%
      mutate(movie_title_and_year = paste(top_rt_movies$movie_title, " (", top_rt_movies$original_release_year, ")", sep=""))

    ggplot(data=top_rt_movies) +
      geom_bar(aes(x=critical_disconnect), width=1) +
      labs(x = 'Critical Disconnect', y = 'Number of Movies', title = 'Distribution of critical disconnect scores')

![Distribution of critical disconnect scores]({{ site.url }}{{ site.baseurl }}/images/movies-critical-disconnect-scores.png)

    summary(top_rt_movies$critical_disconnect)

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##  -76.00  -19.00   -4.00   -6.94    6.00   47.00

We see that the critical disconnect scores follow a left-skewed
distribution. The median is negative, meaning that the majority of
movies have higher audience scores.

### What movies do critics rate much higher than audiences?

Now that we've defined our measure of *critical disconnect*, we can look
at the highest positive values. This will give us the movies that
critics rated much higher than audiences. Let's plot the top 10 movies
that critics rated higher than audiences, in decreasing order of
*critical disconnect*.

    critics_love_audiences_hate <- top_rt_movies %>%
      arrange(desc(critical_disconnect), original_release_year) %>%
      head(10)

    critics_love_audiences_hate %>% 
      arrange(critical_disconnect, desc(original_release_year)) %>%
      mutate(movie_title_and_year=factor(movie_title_and_year, levels=movie_title_and_year)) %>%
      ggplot(aes(x=movie_title_and_year, y = critical_disconnect)) + 
      geom_col() +
      labs(y = 'Critical Disconnect', title='Movies that Critics Rated Higher than Audiences') +
      xlab(NULL) +
      theme(axis.ticks.y=element_blank(), axis.text.y = element_blank()) +
      geom_text(aes(label=critical_disconnect), hjust=-0.3, vjust=0.5, size=4) +
      geom_text(aes(label=movie_title_and_year, y=2), hjust=0, size=4, color='white') +
      coord_flip()

![Movies that Critics Rated Higher than Audiences]({{ site.url }}{{ site.baseurl }}/images/movies-critics-higher.png)

In case we don't recognise all of these movies, we can print a table of
the movie synopses.

    kable(critics_love_audiences_hate %>% 
      arrange(desc(critical_disconnect)) %>% 
      select(movie_title_and_year, primary_genre, movie_info, tomatometer_rating, audience_rating))

<table>
<colgroup>
<col width="5%" />
<col width="3%" />
<col width="84%" />
<col width="3%" />
<col width="2%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">movie_title_and_year</th>
<th align="left">primary_genre</th>
<th align="left">movie_info</th>
<th align="right">tomatometer_rating</th>
<th align="right">audience_rating</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">Spy Kids (2001)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">Two young kids become spies in attempt to save their parents, who are ex-spies, from an evil mastermind. Armed with a bag of high tech gadgets and out-of-this world transportation, Carmen (Alexa Vega) and Juni (Daryl Sabara) will bravely jet through the air, dive under the seas and crisscross the globe in a series of thrilling adventures on a mission to save their parents ... and maybe even the world.</td>
<td align="right">93</td>
<td align="right">46</td>
</tr>
<tr class="even">
<td align="left">Star Wars: The Last Jedi (2017)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">Luke Skywalker's peaceful and solitary existence gets upended when he encounters Rey, a young woman who shows strong signs of the Force. Her desire to learn the ways of the Jedi forces Luke to make a decision that changes their lives forever. Meanwhile, Kylo Ren and General Hux lead the First Order in an all-out assault against Leia and the Resistance for supremacy of the galaxy.</td>
<td align="right">90</td>
<td align="right">43</td>
</tr>
<tr class="odd">
<td align="left">It Comes At Night (2017)</td>
<td align="left">Drama</td>
<td align="left">After a mysterious apocalypse leaves the world with few survivors, two families are forced to share a home in an uneasy alliance to keep the outside evil at bay -- only to learn that the true horror may come from within.</td>
<td align="right">87</td>
<td align="right">44</td>
</tr>
<tr class="even">
<td align="left">Hail, Caesar! (2016)</td>
<td align="left">Comedy</td>
<td align="left">In the early 1950s, Eddie Mannix is busy at work trying to solve all the problems of the actors and filmmakers at Capitol Pictures. His latest assignments involve a disgruntled director, a singing cowboy, a beautiful swimmer and a handsome dancer. As if all this wasn't enough, Mannix faces his biggest challenge when Baird Whitlock gets kidnapped while in costume for the swords-and-sandals epic &quot;Hail, Caesar!&quot; If the studio doesn't pay $100,000, it's the end of the line for the movie star.</td>
<td align="right">85</td>
<td align="right">44</td>
</tr>
<tr class="odd">
<td align="left">Antz (1998)</td>
<td align="left">Animation</td>
<td align="left">Z the worker ant (Woody Allen) strives to reconcile his own individuality with the communal work-ethic of the ant colony. He falls in love with ant-Princess Bala (Sharon Stone), Z strives to make social inroads, and then must save the ant colony from the treacherous scheming of the evil General Mandible (Gene Hackman) that threaten to wipe out the entire worker population.</td>
<td align="right">92</td>
<td align="right">52</td>
</tr>
<tr class="even">
<td align="left">Stuart Little 2 (2002)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">Plucky, pint-sized hero Stuart Little (Michael J. Fox) returns in &quot;Stuart Little 2,&quot; delighting audiences with his big heart and even more action-packed adventure. This time, Stuart must journey through the city with a reluctant Snowbell (Nathan Lane) to rescue a new friend, Margalo (Melanie Griffith), from a villainous Falcon (James Woods).</td>
<td align="right">81</td>
<td align="right">41</td>
</tr>
<tr class="odd">
<td align="left">Haywire (2012)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">Mallory Kane (Gina Carano) is a highly trained operative for a government security contractor. Her missions take her to the world's most dangerous areas. After Mallory successfully frees a hostage journalist, she's betrayed and left for dead by someone in her own agency. Knowing her survival depends on learning the truth behind the double-cross, Mallory uses her black-ops training to set a trap. But when things go awry, Mallory knows she'll die unless she can turn the tables on her adversary.</td>
<td align="right">80</td>
<td align="right">41</td>
</tr>
<tr class="even">
<td align="left">Arachnophobia (1990)</td>
<td align="left">Comedy</td>
<td align="left">After a nature photographer (Mark L. Taylor) dies on assignment in Venezuela, a poisonous spider hitches a ride in his coffin to his hometown in rural California, where arachnophobe Dr. Ross Jennings (Jeff Daniels) has just moved in with his wife, Molly (Harley Jane Kozak), and young son. As town residents start turning up dead, Jennings begins to suspect spiders, and must face his fears as he and no-nonsense exterminator Delbert McClintock (John Goodman) fight to stop a deadly infestation.</td>
<td align="right">92</td>
<td align="right">54</td>
</tr>
<tr class="odd">
<td align="left">Nurse Betty (2000)</td>
<td align="left">Comedy</td>
<td align="left">What happens when a person decides that life is merely a state of mind? If you're Betty, a small-town waitress and soap opera fan from Fair Oaks, Kansas, you refuse to believe that you can't be with the love of your life just because he doesn't really exist. After all, life is no excuse for not living. Traumatized by a savage event, Betty enters into a fugue state that allows -- even encourages -- her to keep functioning... in a kind of alternate reality.</td>
<td align="right">83</td>
<td align="right">45</td>
</tr>
<tr class="even">
<td align="left">About a Boy (2002)</td>
<td align="left">Comedy</td>
<td align="left">A comedy-drama starring Hugh Grant as Will, a rich, child-free and irresponsible Londoner in his thirties who, in search of available women, invents an imaginary son and starts attending single parent meetings. As a result of one of his liaisons, he meets Marcus, an odd 12-year-old boy with problems at school. Gradually, Will and Marcus become friends, and as Will teaches Marcus how to be a cool kid, Marcus helps Will to finally grow up.</td>
<td align="right">93</td>
<td align="right">55</td>
</tr>
</tbody>
</table>

-   A few of the films - *Spy Kids*, *Antz* and *Stuart Little 2* - are
    aimed at kids. It's interesting to see that there's so much
    disagreement on these between critics and audiences.
-   *Star Wars: The Last Jedi* is tied as the top movie critics rated
    higher than audiences (along with *Spy Kids*) and this was a
    divisive film among fans of *Star Wars*. However, it is also
    possible that [this movie fell victim to a "review-bombing campaign"
    on *Rotten Tomatoes*, with some users giving a negative rating
    without even having seen the
    movie](https://www.theverge.com/2019/3/7/18254548/film-review-sites-captain-marvel-bombing-changes-rotten-tomatoes-letterboxd).
-   *Hail Caesar!* is a movie by the Coen brothers about the Hollywood
    film industry and stars big names such as George Clooney. Despite
    being loved by the critics, it was received poorly by audiences.
-   Both *Arachnophobia* and *Nurse Betty* are described as "black
    comedy" films on Wikipedia - it might be the case that audiences
    find such movies disappointing, as they don't provide the same
    laughs that one might expect from a comedy movie.
-   *About a Boy*, a romcom starring Hugh Grant, also achieved a much
    higher score from critics than audiences. Interestingly, the IMDb
    audience score for this movie was high and the majority of top user
    reviews there are highly favourable. This raises the question of
    whether *Rotten Tomatoes* users are somehow different to users on
    *IMDb*.

### What movies do audiences rate much higher than critics?

We can also look at the lowest negative values of *critical disconnect*.
This will give us the movies that audiences rated much higher than
critics.

    audiences_love_critics_hate <- top_rt_movies %>%
      arrange(critical_disconnect) %>%
      head(10)

    audiences_love_critics_hate %>% 
      arrange(desc(critical_disconnect), desc(original_release_year)) %>%
      mutate(movie_title_and_year=factor(movie_title_and_year, levels=movie_title_and_year)) %>%
      ggplot(aes(x=movie_title_and_year, y = critical_disconnect)) + 
      geom_col() +
      labs(y = 'Critical Disconnect', title='Movies that Audiences Rated Higher than Critics') +
      xlab(NULL) +
      theme(axis.ticks.y=element_blank(), axis.text.y = element_blank()) +
      geom_text(aes(label=critical_disconnect), hjust=1.2, vjust=0.5, size=4) +
      geom_text(aes(label=movie_title_and_year, y=2), hjust=1, nudge_y = -5, size=4, color='white') +
      coord_flip()

![Movies that Audiences Rated Higher than Critics]({{ site.url }}{{ site.baseurl }}/images/movies-audiences-higher.png)

    kable(audiences_love_critics_hate %>% 
      arrange(critical_disconnect, original_release_year) %>% 
      select(movie_title_and_year, primary_genre, movie_info, tomatometer_rating, audience_rating))

<table>
<colgroup>
<col width="5%" />
<col width="3%" />
<col width="84%" />
<col width="3%" />
<col width="2%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">movie_title_and_year</th>
<th align="left">primary_genre</th>
<th align="left">movie_info</th>
<th align="right">tomatometer_rating</th>
<th align="right">audience_rating</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">Out Cold (2001)</td>
<td align="left">Comedy</td>
<td align="left">Combining outrageous, sexy comedy with the hottest snowboarding action around, snowboarding buddies Rick (Jason London), Luke (Zach Galifianakis), Anthony (Flex Alexander) and Pigpen live to board on Alaska's Bull Mountain. Catching air, partying hard, and plotting ways to attract women is what life on Bull Mountain is all about.</td>
<td align="right">8</td>
<td align="right">84</td>
</tr>
<tr class="even">
<td align="left">A Low Down Dirty Shame (1994)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">After hitting a wall in his case against drug kingpin Ernesto Mendoza (Andrew Divoff), private eye Andre Shame (Keenen Ivory Wayans) quits the police department in disgrace. The wisecracking investigator thinks he's left his old life behind him, but now everything changes when an old chum from the force informs him of a break in the case. With the help of his motor-mouthed secretary, Peaches (Jada Pinkett), Andre leaps back into the action to solve the case and clear his name.</td>
<td align="right">0</td>
<td align="right">71</td>
</tr>
<tr class="odd">
<td align="left">Belly (1998)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">Ever since they were kids, Sincere (Nas) and Buns (DMX) have lived life close to the edge, doing whatever it takes to survive. As adults, they build up their kingdom of crime on drug dealing and robbery. But Sincere grows weary of the criminal lifestyle and joins a black Muslim religious group. Buns, on the other hand, sinks deeper into criminality and faces serious prison time. The cops offer him a deal, however -- assassinate the head of the Muslim group, and he will go free.</td>
<td align="right">16</td>
<td align="right">87</td>
</tr>
<tr class="even">
<td align="left">Grind (2003)</td>
<td align="left">Action &amp; Adventure</td>
<td align="left">A Chicago teenager (Mike Vogel) and his friends (Vince Vieluf, Adam Brody) travel across America to watch a legendary skateboarder compete.</td>
<td align="right">8</td>
<td align="right">79</td>
</tr>
<tr class="odd">
<td align="left">Diary of a Mad Black Woman (2005)</td>
<td align="left">Comedy</td>
<td align="left">After 18 years of marriage to lawyer Charles (Steve Harris), Helen (Kimberly Elise) is shocked when he announces he's ending their marriage and shacking up with Brenda (Lisa Marcos). Helen retreats to the house of her grandmother Madea (Tyler Perry), who helps her destroy much of Charles' property, earning her house arrest. While Charles prepares for the trial of a corrupt client, Helen is courted by Orlando (Shemar Moore), an affectionate moving man with strong Christian values.</td>
<td align="right">16</td>
<td align="right">87</td>
</tr>
<tr class="even">
<td align="left">Grandma's Boy (2006)</td>
<td align="left">Comedy</td>
<td align="left">When he and his roommate can't pay their rent, video game creator Alex (Allen Covert) finds himself homeless and moves in with Lilly (Doris Roberts), his wacky grandmother. Lilly and her elderly pals like to hang out in front of the television all day, but their constant presence puts a damper on Alex's social life and pot smoking. Alex wants to court co-worker Samantha (Linda Cardellini), but he's preoccupied by a rivalry with another game designer, so the would-be relationship is in limbo.</td>
<td align="right">16</td>
<td align="right">85</td>
</tr>
<tr class="odd">
<td align="left">Facing the Giants (2006)</td>
<td align="left">Drama</td>
<td align="left">Grant Taylor, a Christian high-school football coach (Alex Kendrick), gets some very bad news. Besides his and his wife's (Shannen Fields) infertility problems, he faces the attempt of local parents to force the school to replace him. His team, the Shiloh Eagles, has never had a winning season in the six years that he has coached the boys. Following a visitor's message, Grant tries to inspire his team to use faith to conquer fear and opposing teams.</td>
<td align="right">16</td>
<td align="right">85</td>
</tr>
<tr class="even">
<td align="left">Madea's Family Reunion (2006)</td>
<td align="left">Comedy</td>
<td align="left">Southern matriarch Madea (Tyler Perry) has a lot on her plate. Her nieces have relationship troubles, and Madea has just been ordered by the court to become the guardian of a rebellious teenager named Nikki. Madea must keep the peace and keep her family together while simultaneously planning her clan's reunion.</td>
<td align="right">26</td>
<td align="right">94</td>
</tr>
<tr class="odd">
<td align="left">Madea Goes to Jail (2009)</td>
<td align="left">Comedy</td>
<td align="left">After a high-speed car chase, Madea (Tyler Perry) winds up behind bars because her quick temper gets the best of her. Meanwhile, Assistant District Attorney Josh Hardaway (Derek Luke) lands a case that's too personal to handle: that of a young prostitute and former drug addict named Candace (Keshia Knight Pulliam). When Candace winds up in jail, Madea takes the young woman under her protective wing.</td>
<td align="right">29</td>
<td align="right">96</td>
</tr>
<tr class="even">
<td align="left">Drop Dead Fred (1991)</td>
<td align="left">Comedy</td>
<td align="left">An unhappy housewife (Phoebe Cates) gets a lift from the return of her imaginary childhood friend, Drop Dead Fred (Rik Mayall).</td>
<td align="right">11</td>
<td align="right">77</td>
</tr>
</tbody>
</table>

We make the following observations:

-   The majority of these movies are comedies.
-   *A Low Down Dirty Shame* is [one of very few films with a 0%
    tomatometer
    rating](https://en.wikipedia.org/wiki/List_of_films_with_a_0%25_rating_on_Rotten_Tomatoes).
-   Both *Belly* and *Drop Dead Fred* have achieved somewhat of a cult
    following.
-   Three of the movies on the list are by
    actor/director/producer/screenwriter Tyler Perry, suggesting that
    his work appeals more to audiences than to critics.

Critical disconnect by movie genres
===================================

In this section I'll look at the average *critical disconnect* for each
genre, to see if the *critical disconnect* varies depending on genre. We
are still using the sample of top 5,000 movies.

    movies_by_genre <- top_rt_movies %>%
      group_by(primary_genre) %>%
      summarize(avg_critics_score = mean(tomatometer_rating), 
                avg_audience_score = mean(audience_rating),
                count = n(),
                avg_critical_disconnect = mean(critical_disconnect)
                ) %>%  
      filter(count >= 100)

    kable(movies_by_genre %>%
      arrange(desc(avg_critical_disconnect)))

<table>
<colgroup>
<col width="27%" />
<col width="19%" />
<col width="20%" />
<col width="7%" />
<col width="25%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">primary_genre</th>
<th align="right">avg_critics_score</th>
<th align="right">avg_audience_score</th>
<th align="right">count</th>
<th align="right">avg_critical_disconnect</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">Classics</td>
<td align="right">86.93605</td>
<td align="right">83.98837</td>
<td align="right">172</td>
<td align="right">2.9476744</td>
</tr>
<tr class="even">
<td align="left">Art House &amp; International</td>
<td align="right">77.29858</td>
<td align="right">77.40284</td>
<td align="right">211</td>
<td align="right">-0.1042654</td>
</tr>
<tr class="odd">
<td align="left">Animation</td>
<td align="right">65.02632</td>
<td align="right">67.18947</td>
<td align="right">190</td>
<td align="right">-2.1631579</td>
</tr>
<tr class="even">
<td align="left">Horror</td>
<td align="right">42.75439</td>
<td align="right">48.57544</td>
<td align="right">285</td>
<td align="right">-5.8210526</td>
</tr>
<tr class="odd">
<td align="left">Drama</td>
<td align="right">62.92663</td>
<td align="right">69.92663</td>
<td align="right">995</td>
<td align="right">-7.0000000</td>
</tr>
<tr class="even">
<td align="left">Action &amp; Adventure</td>
<td align="right">53.56750</td>
<td align="right">61.03000</td>
<td align="right">1600</td>
<td align="right">-7.4625000</td>
</tr>
<tr class="odd">
<td align="left">Comedy</td>
<td align="right">51.00492</td>
<td align="right">60.54213</td>
<td align="right">1424</td>
<td align="right">-9.5372191</td>
</tr>
</tbody>
</table>

The plot below visualises this information. Each point represents the
average critics and audience score for movies of that genre. The dotted
line is the line where average audience score equals the average
critic’s score.

    ggplot(data=movies_by_genre, aes(x=avg_critics_score, y=avg_audience_score, color=primary_genre)) + 
      geom_jitter() +
      geom_text(aes(label=primary_genre), vjust=-1, size = 3) + 
      theme(legend.position = "none") + 
      geom_abline(intercept = 0, slope = 1, linetype = "dotted") +
      labs(x = 'Average Critic\'s Score', y = 'Average Audience Score',  title = 'Critical Disconnect by Movie Genre') +
      xlim(35,90) +
      ylim(35,90)

![Critical Disconnect by Movie Genre]({{ site.url }}{{ site.baseurl }}/images/movies-critical-disconnect-genre.png)

The graph suggests that the magnitude of the *critical disconnect* does
vary by genre somewhat.

The genres with the greatest *critical disconnect* are “Comedy”, “Action
& Adventure”, and Drama”. Movies in these genres received on average
9.5, 7.4, and 7 more points from audiences compared to critics,
respectively.

In fact, every genre had a higher average audience score, with the
exception of the small “Classics” genre, which includes older movies
from a variety of genres. When I took the secondary genre as the primary
genre for these, the results were similar. So on average, audiences are
more generous than critics for every genre.

Critical disconnect by decade
=============================

We can do a similar analysis looking at the decade the movie was
released. Remember, the majority of the movies in our dataset are from
the 21st century.

    movies_by_decade <- top_rt_movies %>%
      group_by(original_release_decade) %>%
      summarize(avg_critics_score = mean(tomatometer_rating), 
                avg_audience_score = mean(audience_rating),
                count = n(),
                avg_critical_disconnect = mean(critical_disconnect)
                ) %>%  
      filter(count >= 100)

    kable(movies_by_decade %>%
      arrange(desc(avg_critical_disconnect)))

<table style="width:100%;">
<colgroup>
<col width="26%" />
<col width="19%" />
<col width="20%" />
<col width="7%" />
<col width="26%" />
</colgroup>
<thead>
<tr class="header">
<th align="right">original_release_decade</th>
<th align="right">avg_critics_score</th>
<th align="right">avg_audience_score</th>
<th align="right">count</th>
<th align="right">avg_critical_disconnect</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="right">1960</td>
<td align="right">86.19626</td>
<td align="right">82.61682</td>
<td align="right">107</td>
<td align="right">3.579439</td>
</tr>
<tr class="even">
<td align="right">1970</td>
<td align="right">78.98684</td>
<td align="right">76.65132</td>
<td align="right">152</td>
<td align="right">2.335526</td>
</tr>
<tr class="odd">
<td align="right">1980</td>
<td align="right">65.48724</td>
<td align="right">68.89559</td>
<td align="right">431</td>
<td align="right">-3.408353</td>
</tr>
<tr class="even">
<td align="right">2010</td>
<td align="right">57.92946</td>
<td align="right">62.04066</td>
<td align="right">1205</td>
<td align="right">-4.111203</td>
</tr>
<tr class="odd">
<td align="right">1990</td>
<td align="right">53.32335</td>
<td align="right">62.98967</td>
<td align="right">968</td>
<td align="right">-9.666322</td>
</tr>
<tr class="even">
<td align="right">2000</td>
<td align="right">50.25089</td>
<td align="right">60.39251</td>
<td align="right">1977</td>
<td align="right">-10.141629</td>
</tr>
</tbody>
</table>

    ggplot(data=movies_by_decade, aes(x=original_release_decade, y=avg_critical_disconnect)) +
      geom_col() +
      labs(x= 'Original Release Decade', y = 'Average Critical Disconnect', title='Critical Disconnect by Movie Decade') 

![Critical Disconnect by Movie Decade]({{ site.url }}{{ site.baseurl }}/images/movies-critical-disconnect-decade.png)

This plot shows the average *critical disconnect* for each decade of
movie release. We see that on average, movies from the 60s and 70s were
rated more favourably by the critics than by audiences (as indicated by
the positive average *critical disconnect*). On the other hand, movies
from the 90s and 00s received on average much higher scores from
audiences than from critics.

Conclusion
==========

In this post we used data from Rotten Tomatoes to identify which movies
have the highest disagreement between critics and audiences. We did this
by defining a metric we refer to as *critical disconnect*.

The findings suggests that critics do not always understand what will
appeal to audiences, particularly in the case of the movies we listed
above. For instance, it looks like the audiences disagreed with the
critics on movies such as *Spy Kids*, *Star Wars: The Last Jedi*,
*Antz*, *Hail Caesar!* and *About a Boy*.

Furthermore, the differences in average critical disconnect among
different genres and movie release decades suggests that there are
certain categories of movies where critics are more out of touch.

However, a lot of these observations may actually be explained by
**selection bias**, as suggested in [this *New York Times* article by
Catherine
Rampell](economix.blogs.nytimes.com/2013/08/14/reviewing-the-movies-audiences-vs-critics/).
Unlike critics, audiences *choose* which movies they will watch, and are
therefore more likely to go watch movies that they know they will
probably enjoy.

In fact, this selection bias explains the difference in critic and
audience score distributions. Critic scores take on a wider range of
values because they are paid to watch all sorts of movies, including
ones they wouldn't choose to watch otherwise. This is why some critic
scores are particularly low. On the other hand, audiences will generally
only watch movies they think they will enjoy, meaning most scores will
be fairly positive. This feels particularly relevant for the movies that
audiences rated much higher than the critics - the people watching those
movies are probably already likely to enjoy those kinds of movies.

This illustrates the need to *always think carefully about any biases in
our data*.

Next steps
==========

As always, this piece of analysis raises more questions, such as:

-   How do critics scores compare to IMDb user scores?
-   How can we build an interactive visualisation to present these
    results?
-   What movies do *critics* disagree on?
-   What movies do *audience members* disagree on?
-   Which critic has the lowest "critical disconnect", or in other
    words, most reflects audiences' opinions?
-   Can we build a model to predict critic scores?
