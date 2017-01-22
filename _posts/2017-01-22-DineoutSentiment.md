---
layout: post
title: Top 10 2017 Dineout Vancouver Restaurants by Twitter Sentiment Analysis
---

I'm a huge fan of food. I like cooking it, eating it, taking pictures of it and talking about it. January is an exciting month in Vancouver for food lovers, because we have [Dineout Vancouver](http://www.dineoutvancouver.com/), an annual event where hundreds of local restaurants create fixed priced menus showcasing their cuisine, for very reasonable prices.

But, with over 280 restaurants participating according to the [Georgia Straight](http://www.straight.com/food/853156/over-280-restaurants-participating-dine-out-vancouver-2017), and fierce competition for reservations, how are you supposed to choose where to spend your money?

Inspired by David Robinson's post [here](http://varianceexplained.org/r/yelp-sentiment/), I decided to combine my love of food with my love of data science, and conduct a sentiment analysis of tweets about Dineout Vancouver and rank restaurants based on their average sentiment.

Getting The Tweets
------------------

My first task was getting all the tweets. For this I used Python and the [tweepy library](http://www.tweepy.org/). I took a quick google search for Dineout Vancouver, to find relevant hashtags and user accounts. I found that most posts were using \#DineOutVancouver, \#DOVF, \#dinearound2017 and @DineOutVanFest, the organization's official Twitter handle. Next I connected to the twitter API using my credentials. If you're following along here, I'm assuming that you already know how to register a new twitter application, and grab your credentials. If you don't know how to do that read [this](https://iag.me/socialmedia/how-to-create-a-twitter-app-in-8-easy-steps/).

I started by opening a connection using tweepy, and searching for the relevant words. I eliminated the tweets containing "RT" as these tend to be the restaurants themselves retweeting positive tweets. For each tweet, I grab the text of the tweet, the sender and the unique id. Then, I turn everything into a [pandas](http://pandas.pydata.org/) dataframe and append the the output to a CSV file, as I'm going to do the rest of the analysis in R. Note that I'm appending here instead of overwriting, because the twitter API limits your tweets to the last week. I'd like to re-run this a number of times to get all the tweets during dineout.

    # load packages
    import tweepy
    import json
    import pandas as pd

    # put in your credentials
    consumer_key = "your credentials here"
    consumer_secret = "your credentials here"
    access_token = "your credentials here"
    access_secret = "your credentials here"

    # authorize
    authorization = tweepy.OAuthHandler(consumer_key=consumer_key, consumer_secret=consumer_secret)
    authorization.set_access_token(access_token, access_secret)

    # create the client
    client = tweepy.API(authorization)

    # initialize empty lists to store tweet info
    tweets = list()
    from_user = list()
    tweet_id = list()

    # for each tweet matching the search criteria
    for tweet in tweepy.Cursor(client.search,
                               q='@DineOutVanFest OR #DineOutVancouver OR #DOVF OR #dinearound2017 exclude:retweets').items():
        
        # only get tweets that aren't retweets
        if "RT" not in tweet.text:
            tweets.append(tweet.text) # add the tweet text
            from_user.append(tweet.user.screen_name) # add the sender
            tweet_id.append(tweet.id_str) # add the tweet id

    # turn everything into a dataframe
    dineout_tweets = pd.DataFrame({'sender': from_user,
                                    'tweet_text': tweets,
                                   'tweet_id': tweet_id}, columns=['sender', 'tweet_text', 'tweet_id'])

    # print the number of tweets grabbed
    print("Number of Tweets Grabbed: %f" % (len(tweets)))

    # append the tweets to the current file instead of overwriting to get cumulative tweets
    # handle duplicates later
    dineout_tweets.to_csv("../data/tweets.csv", mode='a', sep=',', index=False)

Necessary But Painful Data Entry
--------------------------------

I'm not going to lie, this part wasn't fun. I'm sure with enough effort I could have mostly automated all of this by writing a little program in [scrapy](https://scrapy.org/) to crawl search results and look for twitter handles, but I felt like that would take longer than doing it manually, so I just bit the bullet and manually collected all of the twitter names of participating restaurants I could find. I used Google Sheets to compile this into a list and then save it as a CSV. I think it's important to note here, that not every restaurant has a Twitter handle, and some restaurant chains such as Cactus Club or Glowbal Group, have multiple restaurants under one name. This isn't ideal, as in the case of Glowbal Group, they have a bunch of different restaurants serving different food. So, a review of one restaurant shouldn't effect the others, but it's the best I could do.

Data Wrangling and Analysis
---------------------------

First, I loaded all of the packages I needed for the analysis, including [tidyverse](http://tidyverse.org/). If you don't have this installed, using `install.packages("tidyverse")` will install everything you need for data wrangling, and then you'll need to install [tidytext](https://github.com/juliasilge/tidytext) with `install.packages("tidytext")` which is a package in R for doing text analysis.

``` r
library(tidyverse)
library(stringr)
library(jsonlite)
library(tidytext)
library(knitr)
```

Next, I read in my list of participating restaurants, and eliminated the duplicates mentioned above. We're going to use this later to filter out tweets to users who aren't participating restaurants.

``` r
# load in the list of dineout restaurants
restaurants <- read_csv("../data/dineout_restaurants.csv")

# filter out the ones that don't have a twitter username
restaurants <- restaurants %>% 
  filter(!is.na(twitter_username))

# unique list of participating restaurant twitter names
restaurant_twitter_names <- unique(restaurants$twitter_username)
```

Here, I load in the list of tweets, and eliminate any duplicates. I'm doing this because every time I pull new tweets I'm appending instead of overwriting, so there will be duplicates. Then I write the clean dataset back to the file. I'm also converting the id to a character string rather than a number.

``` r
# load in the list of tweets with either #dineoutvancouver or @dineoutvanfest
tweets <- read_csv(file = "../data/tweets.csv")

# eliminate duplicates
tweets <- distinct(tweets)

# write the eliminated duplicates back to file
write_csv(x = tweets, path = "../data/tweets.csv", col_names = TRUE)

# convert the tweets_ids to a character
tweets <- tweets %>% 
  mutate(tweet_id=as.character(tweet_id))
```

Now I have a list of unique tweets, but the users I'm looking for are within the text of the tweet, and some tweets mention multiple users. So, I start by using `map` to create an extra column only containing the users mentioned in the tweet. Then, for each tweet I create a separate row containing the same tweet information for each user mentioned. This is important, because where multiple users are mentioned, I want the sentiment of each tweet to contribute to each of the mentioned user's scores.

``` r
# pull the list of users sent the message from the text of the tweet and create a new column
users <- unlist(map(map(map(tweets$tweet_text, .f = str_extract_all, pattern="@\\w+"), unlist),paste, collapse = " "))
tweets$users <- users

# get one row per user
tweets <- tweets %>% 
  unnest_tokens(username, users)
```

Here's where the magic of [tidytext](https://github.com/juliasilge/tidytext) comes in. First, the `unnest_tokens()` method allows me to easily split each tweet into a separate row per word. The tidytext package also contains a list of pre-compiled stop words to help you filter out words not relevant to the sentiment analysis. So, my next step is removing these from the words in each tweet. Next, I'm using the AFINN lexicon, also included in the tidytext package that includes a positivity score from -5 to positive 5 for each word, and using an inner join with the list of tweeted words, to get a sentiment score for each word. Then, I calculating the mean sentiment score for each tweet.

``` r
# remove stop words from the tweets
tweet_words <- tweets %>%
  select(username, tweet_text, tweet_id) %>%
  unnest_tokens(word, tweet_text) %>%
  filter(!word %in% stop_words$word,
         str_detect(word, "^[a-z']+$"))

# get a list of words with sentiment scores from the tidytext package
AFINN <- sentiments %>%
  filter(lexicon == "AFINN") %>%
  select(word, afinn_score = score)

# get the average sentiment of each tweet
tweet_sentiment <- tweet_words %>%
  inner_join(AFINN, by = "word") %>%
  group_by(username, tweet_id) %>% 
  summarize(sentiment = mean(afinn_score))

print(head(tweet_sentiment,10))
```

    ## Source: local data frame [10 x 3]
    ## Groups: username [7]
    ## 
    ##           username           tweet_id sentiment
    ##              <chr>              <chr>     <dbl>
    ## 1     aliencheftim 822521576134299649  3.000000
    ## 2  alportoitaliano 822621287864532992  1.000000
    ## 3        alumniubc 819705180627279872  2.000000
    ## 4     ancoradining 822680393455267840  4.000000
    ## 5      annalenayvr 821520226495799296  1.666667
    ## 6      annalenayvr 821949762035298305  2.000000
    ## 7      annalenayvr 822604362082029570  4.000000
    ## 8      annalenayvr 822637253591703552  3.000000
    ## 9      askforluigi 820358644185333760 -2.000000
    ## 10     basilmintbc 822248572951392256  2.000000

``` r
print(sprintf("Number of tweets with a sentiment score: %d", length(tweet_sentiment$tweet_id)))
```

    ## [1] "Number of tweets with a sentiment score: 453"

``` r
print(sprintf("Number of total tweets: %d", length(tweets$tweet_id)))
```

    ## [1] "Number of total tweets: 1434"

Now we have a list of tweets with an average sentiment score, based on the sentiment score of the individual words in each tweet. But, we have a bit of problem. There's only 453 out of 1434 tweets that contained any of the words in the AFINN lexicon. What this means for my analysis, is that there's a lot of tweets that don't contain negative or positive words. At this point, I need to make a choice. I'm going to count tweets with no negative or positive words as neutral by assigning them a zero value. To do this, I'm combining the tweets with sentiment scores with the original tweets again, and then I'm assigning 0 to the sentiment value of any tweet with `NA`.

``` r
# combine the sentiment with original tweet list
tweet_with_sentiment <- tweets %>% 
  left_join(tweet_sentiment)

# fill the tweets with no sentiment with zeros so they can be neutral
tweet_with_sentiment$sentiment[is.na(tweet_with_sentiment$sentiment)] <- 0
```

Now we have a list of all our tweets again with sentiment scores for all of them. Next, we need to determine a sentiment score for each user. Finally, we need to filter out users mentioned that aren't participating restaurants.

``` r
# get the average sentiment of each user
user_sentiment <- tweet_with_sentiment %>%
  group_by(username) %>%
  summarize(sentiment = mean(sentiment), n_tweets=n())

# filter only for participating restaurants and not other users.
participant_sentiment <- user_sentiment %>% 
  filter(username %in% restaurant_twitter_names) %>% 
  arrange(-sentiment)

# get the top ten users
top_ten_sentiment <- head(participant_sentiment, 10)

# format column names more descriptively
top_ten_sentiment <- top_ten_sentiment %>% 
  select(`Twitter Name`=username, Sentiment=sentiment, Tweets=n_tweets)

# print a nice table of the top restaurants
top_ten_table <- kable(x = top_ten_sentiment, align = 'c', caption = "Top Ten Dineout 2017 Restaurants by Twitter Sentiment")
```

Now I'll print the top ten restaurants in a table along with their sentiment score and number of tweets. I'll also produce a bar chart with the same information. I'm also going to create a scatter plot of the number of tweets versus the sentiment score to visualize any relationship between the two.

``` r
# reorder the restaurants by decending sentiment
plot_sentiment <- participant_sentiment %>%
  mutate(username=reorder(username, -sentiment)) %>%
  select(username, sentiment, n_tweets) %>%
  head(10) %>% 
  gather(key = measure, value = value, sentiment, n_tweets)

# plot the sentiment and number of tweets
top_ten_bar <- ggplot(plot_sentiment, aes(x=username, y=value, fill=measure)) + geom_bar(stat = "identity", position = "dodge") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  xlab("Twitter Username") +
  ylab("Sentiment / Number of Tweets") + 
  ggtitle(label = "Top Ten Dineout Vancouver Restaurants", subtitle = "by Twiiter Sentiment Score")

# plot to compare number of tweets v sentiment
tweets_v_sentiment <- ggplot(participant_sentiment, aes(x=n_tweets, y=sentiment, label = username)) + 
  geom_point() +
  geom_text(size = 3, check_overlap = TRUE, hjust = 0, nudge_x = 0.5) +
  xlab("Number of Tweets") +
  ylab("Sentiment") + 
  ggtitle(label = "Top Ten Dineout Vancouver Restaurants", subtitle = "by Twiiter Sentiment Score")
```

Results
-------

**Drumroll please. Here are the top ten Dineout Vancouver 2017 restaurants by sentiment score.**

|   Twitter Name  | Sentiment | Tweets |
|:---------------:|:---------:|:------:|
|   ancoradining  |  4.000000 |    1   |
|     catch122    |  4.000000 |    1   |
|    diva\_met    |  3.000000 |    1   |
| pidginvancouver |  3.000000 |    2   |
|  marketjg\_van  |  2.166667 |    3   |
|  chewiesoysters |  2.000000 |    2   |
|   fsvancouver   |  2.000000 |    5   |
|  westrestaurant |  2.000000 |    2   |
| housespecialvan |  1.666667 |    6   |
|   annalenayvr   |  1.523810 |    7   |

![](https://kitsonswann.github.io/kitsonswann.github.io/images/sentiment_bar.png)
![](https://kitsonswann.github.io/kitsonswann.github.io/images/sentiment_v_tweets.png)

Discussion
----------

This is pretty exciting to me. In a few hours worth of work using the Twitter API and packages such as `tweepy` for Python and `tidytext` for R, I was able to conduct a real life sentiment analysis of something I care about, to help me make a purchase decision. But, there's definitely some limitations of this analysis and room for improvement. Firstly, there isn't a lot of data to go on here, as Dineout Vancouver 2017 only began on Friday, January 20th, so there's very few tweets in general. One positive or negative tweet, can really sway the sentiment score. Also, looking at some of the tweets that are negative, the algorithm definitley isn't perfect. Here's a few examples:

``` r
# get the ten worst tweets
worst_tweets <- tweet_with_sentiment %>%
  filter(username %in% restaurant_twitter_names & sentiment < 0) %>% 
  arrange(sentiment) %>%
  select(`Tweet Text`=tweet_text, Sentiment=sentiment)

# print them in a nice table
kable(format = 'html', x = worst_tweets, caption = "Worst Tweets by Sentiment")
```

<table>
<caption>
Worst Tweets by Sentiment
</caption>
<thead>
<tr>
<th style="text-align:left;">
Tweet Text
</th>
<th style="text-align:right;">
Sentiment
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
@Glowbal\_Group tonight's \#dineoutvancouver at coast was a total shit show. waited 2 hours for food to arrive cold and undercooked. wtf? 😡👎🏻
</td>
<td style="text-align:right;">
-4.0
</td>
</tr>
<tr>
<td style="text-align:left;">
Newfoundland Seal on menu?? You bring SHAME AND DISGUST to Canada! @ediblecanada @dineoutvanfest @JustinTrudeau… <https://t.co/hkoEgpyoTl>
</td>
<td style="text-align:right;">
-2.5
</td>
</tr>
<tr>
<td style="text-align:left;">
@ediblecanada @DineOutVanFest SSUK is appalled to hear you serve seal on your menu, we thought Vancouver was beyond… <https://t.co/GijVYKF6ls>
</td>
<td style="text-align:right;">
-2.0
</td>
</tr>
<tr>
<td style="text-align:left;">
@cbcnewsbc @EdibleCanada @DineOutVanFest Boo!!! \#Canada \#seals \#shame \#vancouver \#vanpoli <https://t.co/6wTQ7UcNUJ>
</td>
<td style="text-align:right;">
-2.0
</td>
</tr>
<tr>
<td style="text-align:left;">
@thisispopulist your post&amp;article plugs @DineOutVanFest @EdibleCanada seal menu. all these animals want 2 live &amp; aren't ours 2 exploit.
</td>
<td style="text-align:right;">
-2.0
</td>
</tr>
<tr>
<td style="text-align:left;">
@candymaptones @EdibleCanada @DineOutVanFest WHEN WILL CANADA LEARN?? \#shameful \#canadasshame \#banthesealhunt
</td>
<td style="text-align:right;">
-2.0
</td>
</tr>
<tr>
<td style="text-align:left;">
went to @La\_Pentola food was delicious, service was pretty bad. wonder if it was because we were there for \#dineoutvancouver?
</td>
<td style="text-align:right;">
-1.0
</td>
</tr>
<tr>
<td style="text-align:left;">
\#DineOut alert! Head over to @TableauBistro &amp; @LABATTOIR\_VAN for delicious, local lingcod cooked by the best chefs… <https://t.co/l5CMIeupkr>
</td>
<td style="text-align:right;">
-1.0
</td>
</tr>
<tr>
<td style="text-align:left;">
\#DineOut alert! Head over to @TableauBistro &amp; @LABATTOIR\_VAN for delicious, local lingcod cooked by the best chefs… <https://t.co/l5CMIeupkr>
</td>
<td style="text-align:right;">
-1.0
</td>
</tr>
<tr>
<td style="text-align:left;">
Don't forget that @dineoutvanfest starts tomorrow! @La\_Pentola is one of the participating… <https://t.co/pNLP0CDmt0>
</td>
<td style="text-align:right;">
-1.0
</td>
</tr>
</tbody>
</table>

For example most of these tweets are negative, but there are a couple that aren't. Also, there's a lot of negative tweets about @EdibleCanada, but they are all related to outrage about them serving seal meat, rather than how good the food is or how good the service is. I think another improvement that needs to be made here, would be to create or find a list of positive/negative words specifically related to food, that may not be in the list I used from tidytext.
