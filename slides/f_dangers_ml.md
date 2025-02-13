---
title: "The dangers of machine learning"
author: "Alexander Murray-Watters"
date: "12 April 2019"
output: 
  slidy_presentation: 
    keep_md: yes
---



## Models are often uninterpretable
As the models produced by, e.g., a neural network are often
uninterpretable, it is difficult to work out where a model went wrong.

Google Flu trends is a prime example of this. For many years, Google
was able to accurately describe where the flu had spread to (and how
many people had it) faster than the CDC (Centers for Disease Control
and Prevention). Google wrote tracked the spread of flu by looking for
spikes in the number of searches for the flu (or flu related
symptoms). There was some discussion of using Google Flu trends as
part of the US government's response to flu outbreaks. The downside is
that when other events caused people to search about the flu on
Google, such as the Swine flu outbreak in 2009, the model broke down,
as it couldn't distinguish searches about the news story from searches
about the flu. Changes in how the news coverge has also led to
problems with Google's method. Since the individual components of a
neural network are difficult to interpret, it can be hard to tell if
the problem is with the variables included (or excluded) from the
model, or if the problem is with the model parameters, functional
relationship between variables, or some other
cause. https://www.nature.com/news/when-google-got-flu-wrong-1.12413

TODO: Insert image.

## Biased data

Garbage in => Garbage out.

TODO: Insert bit on racist Google image search 

https://www.theguardian.com/technology/2018/jan/12/google-racism-ban-gorilla-black-people
https://www.theverge.com/2018/1/12/16882408/google-racist-gorillas-photo-recognition-algorithm-ai

TODO: Insert image.

## Example: How to make a racist AI


(Based on: https://blog.conceptnet.io/posts/2017/how-to-make-a-racist-ai-without-really-trying/)

We grab the GloVe pre-trained word vector first: https://nlp.stanford.edu/projects/glove/



A word vector is a representation of a word within what is called a
"word vector space". These vector spaces attempt to model the semantic
and syntatical context of words, and are usually constructed by
feeding a large body of text into a neural network. The neural network
is usually constructed so that given an input such as context (e.g.,
the other words in a sentence surounding the word to be predict),
predict what word should "fit". The other direction is also possible,
that is, predict "context" given a particular word.

This particular GloVe word vector dataset has a 1.9 million word
vocabulary is 1.75 Gigabytes large (compressed), 5 Gigabytes
uncompressed.

- Method for constructing dataset: https://nlp.stanford.edu/pubs/glove.pdf

- Dataset: http://nlp.stanford.edu/data/glove.42B.300d.zip








```r
glove.df <- read.csv("~/Downloads/glove.42B.300d.txt",
                 header=TRUE, sep=" ",quote="",
                 col.names=c("word",paste0("e",1:300)))
```




```r
## Did the file read in correctly?
head(glove.df$word, n=100)

dim(glove.df)

head(glove.df$e1)

# We use the scan function to read in the list of words as it has an
# easy way to distinguish comments from data (that way we don't have
# to bother with grep or regular expressions).
negative.words <- scan("../data/negative-words.txt", comment.char=";", what="", blank.lines.skip=T)

positive.words <- scan("../data/positive-words.txt", comment.char=";", what="", blank.lines.skip=T)

length(positive.words)
length(negative.words)

head(positive.words)
head(negative.words)
```


```r
## Locating postive and negative words in our dataset.
pos.vectors <- which(glove.df$word %in% positive.words)
neg.vectors <- which(glove.df$word %in% negative.words)

## Limiting dataset to only those words in our lists of pos/neg words.
glove.df.reduc <- glove.df[c(pos.vectors, neg.vectors),]

## Need to reassign posiions now (since we have a smaller dataset).
pos.vectors <- which(glove.df.reduc$word %in% positive.words)
neg.vectors <- which(glove.df.reduc$word %in% negative.words)


## Binary indicator for positive or negative words.
sentiment.vec <- ifelse(1:length(glove.df.reduc$word)%in%pos.vectors,
                        1, ifelse(1:length(glove.df.reduc$word)%in%neg.vectors, -1, 0))

## backing up word vector
word.backup <- glove.df.reduc$word

## Converting to matrix for glmnet.
glove.df.reduc <- as.matrix(glove.df.reduc[,-1] )

rownames(glove.df.reduc) <- word.backup

## garbage collecting to recover RAM.
gc(); gc(); gc();


## Testing/traing datasets. 80% for training, 20% for testing.
train.df <- sample(1:length(word.backup), size=length(word.backup)*.8)

test.df <-  which(!(1:length(word.backup))%in%train.df)

## If this isn't 0 then we've messed up our spliting procedure.
sum(test.df%in%train.df)


## Fitting a model -- lasso regression.
fit.cv <- cv.glmnet(glove.df.reduc[train.df,], sentiment.vec[train.df],family="binomial" )

pred.fit <- predict(fit.cv$glmnet.fit, glove.df.reduc[test.df,], s=fit.cv$lambda.min)

pred.fit[sample(1:nrow(pred.fit), size=10),]
```



```r
library(ggplot2)
library(dplyr)


## Making sure things make sense.
ggplot(tibble(pred.fit,
              pos.neg = as.character(sentiment.vec[c(pos.vectors, neg.vectors)[test.df]]))) +
  aes(x = pos.neg, y = pred.fit) +
  geom_boxplot() +
  labs(title = "Boxplot of model Coefficents by word type",
       x="Positive or Negative ", y = "Coefficent")
```



```r
## Calculating the mean sentiment of a sentence.
calc.sentiment <- function(input.text="This is great!", reg.model = fit.cv,
                           known.word.list = word.backup, glove.df){

  ## Simple pattern that matches whitespace or puncuation. Note:
  ## screws-up apostrophes and hyphens!
  simple.regex <- "[[:blank:]|[:punct:]]"

  ## Splits text using our regex into a vector of words and empty spaces.
  word.vec <- tolower(unlist(strsplit(input.text, simple.regex)))

  ## Deleting the empty spaces. 
  word.vec <- word.vec[nchar(word.vec)>0]


  n.unknown.words <- sum(!(word.vec %in% known.word.list))

  ## If no words match, return 0.  
  if(n.unknown.words == length(unique(word.vec))){return(0)}

  ## If only 1 word matches, we have to transpose the matrix so that
  ## it matches what glmnet needs.
  ## else if((length(unique(word.vec)) - n.unknown.words) == 1 ){
  ##   new.data <- t(as.matrix(glove.df[which(glove.df$word %in% word.vec),]))
  ## }
  else{
    new.data <- as.matrix(glove.df[which(known.word.list %in% word.vec),])
  }


  pred.val <- predict(reg.model$glmnet.fit,
                      new.data,
                      s=reg.model$lambda.min)

  return(mean(c(pred.val)))
}

## Backup list of words
word.backup <- glove.df$word

#convert to matrix and drop list of words (otherwise we end up with
#matrix of strings, not numbers)
glove.df.final <- as.matrix(glove.df[, -1])

gc()
```

## Now for the Racism



```r
calc.sentiment("Let's go out for Italian food.", glove.df=glove.df.final)

calc.sentiment("Let's go out for Chinese food.", glove.df=glove.df.final)

calc.sentiment("Let's go out for Mexican food.", glove.df=glove.df.final)
```
So "Mexican" is rated lower than the other two.


## What about names?


```r
calc.sentiment("My name is Emily", glove.df=glove.df.final)

calc.sentiment("My name is Heather", glove.df=glove.df.final)

calc.sentiment("My name is Yvette", glove.df=glove.df.final)

calc.sentiment("My name is Shaniqua", glove.df=glove.df.final)
```

## Exercise: calc.sentiment
Try giving `calc.sentiment` some other sentences to check for other
kinds of prejudices or bias contained in the glove neural network.

## Exercise: calc.sentiment
Try to find some words or sentences that break the regular expression
that extracts words from sentences.

A stereotypical black name (Shaniqua) is rated far more negatively
than a stereotypical white name (such as Emily or Heather).



Even when a method appears to be unbiased on the surface, if there is
bias in how data are collected, the *model* will also be biased.

## Other examples

- The COMPAS algorithm (sometimes used to decide who gets parole),
resulted in [racist parole decisions](https://www.propublica.org/article/how-we-analyzed-the-compas-recidivism-algorithm)
	- Code and data used in the writer's analysis is available on [github](https://github.com/propublica/compas-analysis)

- If your model's predictions effect what new data is gathered (e.g.,
predictive policing) there is a risk of creating distorting feedback
loops. [One example](https://www.themarshallproject.org/2016/02/03/policing-the-future) involves using a model of where arrests for a
particular crime occurred to guide police patrol assignments, resulting
in more arrests in those areas (due to the increased patrol presence),
leading to more patrol assignments in that area, etc.
	- A notorious example of a feedback loop resulting in junk data
      and horrific consequences is the "[body
      count](https://en.wikipedia.org/wiki/Vietnam_War_body_count_controversy#Body_count_inflation)",
      which invovled using the number of "enemies" killed as a measure
      for how well the war was progressing. As commanders were assesed
      based on the their unit's count, there was a strong incentive
      for falsification -- in one way or another.

- Other concerning models include the evaluation of teachers,
  calculating credit scores, and citizenship scores (e.g., [social
  credit system in China](https://en.wikipedia.org/wiki/Social_Credit_System)).


- TODO: Add bit on adversarial neural networks. : https://towardsdatascience.com/breaking-neural-networks-with-adversarial-attacks-f4290a9a45aa?gi=c4e5bf7acb50

## There is no magic to machine learning


- If someone can't explain how their method works in a simple way, but
insists it solves all of your problems, they're selling snake oil.

- The most oversold method right now (neural networks and deep
  learning), is essentially a form of non-linear regression. As with any
  form of regression, the model is only as good as its assumptions and
  data. 
	  - If you are trying to make predictions for data that is not
  within the range of your sample, the model will almost certainly perform poorly.
	  - If your data suffer from a selection effect, an incorrectly
        randomized experiment/trial, or other problems with the data
        gathering process, applying a machine learning model (like any
        statistical model) will perform poorly.

- Overfitting and out-of-sample prediction are still both
  problems. The only known solution (at the present time) is to
  already know a close to true underlying model (as in much of
  physics).

 "Once men turned their thinking over to machines in the hope that this
would set them free. But that only permitted other men with machines
to enslave them.” Frank Herbert, Dune, 1965.

# Further reading

## Nontechnical

 - Weapons of Math Destruction by Cathy O’Neil.
 
## Specific and/or technical

### General introductions

 - Pattern Recognition and Machine Learning. Bishop, Christopher, 2011. https://www.amazon.com/Pattern-Recognition-Learning-Information-Statistics/dp/0387310738


 - Computer Age Statistical Inference:Algorithms, Evidence and Data Science. Efron et al, 2016. https://web.stanford.edu/~hastie/CASI_files/PDF/casi.pdf

 - An Introduction to Statistical Learning with Applications in R. James et al, 2013. http://www-bcf.usc.edu/~gareth/ISL/ISLR%20Sixth%20Printing.pdf

 - The Elements of Statistical Learning: Data Mining, Inference, and Prediction. 2009 http://www.stanford.edu/~hastie/ElemStatLearn/printings/ESLII_print12.pdf

## Deep learning

 - Deep Learning with Python. Chollet, Francois, 2017 https://www.amazon.com/Deep-Learning-Python-Francois-Chollet/dp/1617294438

 - Deep Learning. Goodfellow et al, 2016. https://www.amazon.com/Deep-Learning-Adaptive-Computation-Machine/dp/0262035618/




### Adversarial neural networks
 - Explaining and Harnessing Adversarial Examples. Goodfellow et al, 2015 https://arxiv.org/abs/1412.6572

 - Adversarial Examples in the Physical World. Kurakin et al, 2017 https://arxiv.org/pdf/1607.02533.pdf- 

 - Adversarial Patch. Brown et al, 2018 https://arxiv.org/pdf/1712.09665.pdf

 -  (Note: A non-technical summary of these papers is avalible here: https://towardsdatascience.com/breaking-neural-networks-with-adversarial-attacks-f4290a9a45aa?gi=c4e5bf7acb50)
 
 

