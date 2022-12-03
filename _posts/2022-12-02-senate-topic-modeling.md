---
title: "Visualizing Topics of 74th Congress Senate Bills and Joint Resolutions"
layout: post
date: "2022-12-02"
---

With this post, I'm taking a break from my [metadata workflow series](https://blackerby.github.io/workflow-series/) to write about something that might come at the end of such a workflow: using [topic modeling](https://en.wikipedia.org/wiki/Topic_model) to analyze a text corpus.

In my digital libraries class this semester, we have been studying the application of machine learning tools to text analysis problems. For practice, I decided to apply one of those tools, topic modeling, to the corpus of 74th Congress Senate bills and join resolutions I processed during my remote metadata internship with the Law Library of Congress.

# The Workflow

## Setting Up

### The Necessary Libraries

Below, we load the libraries this project uses into my work environment. `tidyverse`, `tidytext`, and `ggplot2` are pretty well know inhabitants of the R Tidyverse, so I won't explain them here. [stm](https://www.structuraltopicmodel.com/) is the package used in this project for topic modeling. [LDAvis](https://github.com/cpsievert/LDAvis) is the tool we will use to visualize the [LDA](https://en.wikipedia.org/wiki/Latent_Dirichlet_allocation) model applied to Senate bill corpus. [quanteda](http://quanteda.io/) is a text analysis library, and [spacyr](https://spacyr.quanteda.io/index.html) is an R wrapper for the Python natural language processing library [spaCy](https://spacy.io/). [gistr](https://github.com/ropensci/gistr) is necessary for publishing the visualization to the web.

```r
library(tidyverse)
library(tidytext)
library(ggplot2)
library(stm)
library(LDAvis)
library(quanteda)
library(spacyr)
library(gistr)
```

### Initializing spaCy

Below, we initialize spaCy with its built-in English language model.

```r
spacy_initialize(model = "en_core_web_sm")
```

### Load dataset

The data we will use is available for download as a CSV file from a simple API created using Datasette. I have crafted a SQL query to get just the data that we need. This query will return a table containing each bill summary's unique ID in the database and its text.

```r
data <- read.csv("https://llc.herokuapp.com/summaries.csv?sql=select%0D%0A++rowid%2C%0D%0A++content_text%0D%0Afrom%0D%0A++files%0D%0Awhere%0D%0A++%22path%22+like+%2274_2_s%25%22%0D%0Aorder+by%0D%0A++path&_size=max")
head(data)
```

## Preprocessing

With the environment set up and the data acquired, we can start massaging the data into the shape the topic modeling algorithm needs.

### spaCy

First, we put the text of all the bills we downloaded into one big string which we then use spaCy to parse.

```r
text <- data$content_text %>% str_c()
parsed_text <- spacy_parse(text, lemma = TRUE, entity = TRUE, nounphrase = TRUE)
head(parsed_text)
```

### Getting the words we want

Next, we reduce the corpus to tokens that have been [part-of-speech tagged](https://en.wikipedia.org/wiki/Part-of-speech_tagging) as proper nouns, verbs, nouns, adjectives, and adverbs. The part-of-speech tagging isn't perfect, so we remove tokens tagged as those parts of speech that we know aren't, actually. Finally, make all of the lemmas lowercase and create the `word` column.

```r
tokens <- parsed_text %>% 
  filter(pos == "PROPN" | pos == "VERB" | pos == "NOUN" | pos == "ADJ" | pos == "ADV") %>%
  filter(lemma != "." & lemma != "Mr." & str_detect(lemma, "^\\w\\.?$", negate = TRUE)) %>% 
  filter(lemma != "§") %>% 
  mutate(word = tolower(lemma))
```

Next we get rid of [stop words](https://en.wikipedia.org/wiki/Stop_word)

```{r}
data("stop_words")
tokens <- tokens %>% anti_join(stop_words)
```

And now we can take a quick look at the tokens that appear more than 100 times in the bill summaries.

```r
tokens %>%
  count(word, sort = TRUE) %>% 
  filter(n > 100) %>% 
  mutate(word = reorder(word, n)) %>% 
  ggplot(aes(n, word)) +
  geom_col() +
  labs(y = NULL)
```

The names of months show up a lot, but they won't contribute meaningfully to the topic model. Let's get rid of them...

```r
month_tokens <- tibble(month.name) %>% 
  unnest_tokens(word, month.name)

tokens <- tokens %>% anti_join(month_tokens)
```

...and see what our chart looks like now

```{r}
tokens %>%
  count(word, sort = TRUE) %>% 
  filter(n > 100) %>% 
  mutate(word = reorder(word, n)) %>% 
  ggplot(aes(n, word)) +
  geom_col() +
  labs(y = NULL)
```

This is better, but there's more we can do. First, let's remove some more words that might clutter our model.

These calls bring the text of bill types and the names of sponsors identified in the original document into the environment.

```r
bill_types <- read.csv("https://llc.herokuapp.com/summaries.csv?sql=select+distinct+bill_type+from+actions&_size=max")
sponsors <- read.csv("https://llc.herokuapp.com/summaries.csv?sql=select+distinct+sponsor+from+actions+where+sponsor+is+not+null+and+sponsor+%21%3D+%22%22&_size=max")
```

We can now make that text tokens

```r
bill_type_tokens <- bill_types %>% 
  unnest_tokens(word, bill_type)
sponsor_tokens <- sponsors %>% 
  unnest_tokens(word, sponsor)
```

and remove them from our corpus.

```r
tokens <- tokens %>% 
  anti_join(bill_type_tokens) %>%
  anti_join(sponsor_tokens)
```

How does our chart look now?

```r
tokens %>%
  count(word, sort = TRUE) %>% 
  filter(n > 100) %>% 
  mutate(word = reorder(word, n)) %>% 
  ggplot(aes(n, word)) +
  geom_col() +
  labs(y = NULL)
```

It doesn't look like any change, but we have reduced the number of tokens in our corpus. I think we're in good enough shape to do our topic modeling.

## Topic Modeling

### A document feature matrix

The `stm()` function exposed by the `stm` package can't take our data frame as input. Instead, it needs a [document-term matrix](https://en.wikipedia.org/wiki/Document-term_matrix), in this case in the form of a document-feature matrix, which we create with the `tidytext` `cast_dfm()` function.

```r
dfm <- tokens %>% 
  count(doc_id, word, sort = TRUE) %>% 
  cast_dfm(doc_id, word, n)
```

We're ready to apply the model! I've set k, the number of topics, as 48 because there were 48 Senate committees in 74th Congress. The following line of code takes a few minutes to finish executing.

```r
model <- stm(dfm, K = 48, init.type = "LDA", seed = 1234, verbose = FALSE)
```

### Visualization

Before we can visualize our model with `LDAvis`, we need to put our document-term matrix into a form that `toLDAvis`, an `stm`-provided wrapper around `LDAvis`, can understand. We do that with the call to `convert()` below. The call to `toLDAvis` creates the visualization as a static website and publishes it online.

```r
lda <- convert(x = dfm, to = "lda")

ldavis <- toLDAvis(model, lda$documents, R = 48,
		   open.browser = interactive(),
		   as.gist = TRUE)
```

You can see the final product [here](https://bl.ocks.org/blackerby/raw/def5c205228616e8e0e6cc0945bd9f4f/#topic=0&lambda=1&term=).
