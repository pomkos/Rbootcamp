---
title: "Logistic Regression Project"
author: "Peter"
date: "February 28, 2019"
output: html_document
---

GOAL: Predict if people belong in a certain class by salary, either making <=50k or >=50k

# Setup

```{r}
adult <- read.csv('adult_sal.csv')
head(adult)
table(adult$country)

library(dplyr)
adult <- select(adult,-X)
head(adult)
str(adult)
summary(adult)
```

# Data Cleaning

```{r}
table(adult$type_employer)
```

* There are 1836 null values, the smallest group is the 'never-worked' group

## Combine 'Never-worked' and 'Without-pay' into 'Unemployed'
```{r}
  unemp <- function(x){
      x <- as.character(x)
      if (x == 'Never-worked' | x=='Without-pay'){
        return('Unemployed')
      }else{
        return(x)
      }
    }
  
  adult$type_employer <- sapply(adult$type_employer, unemp)
  adult$type_employer <- factor(adult$type_employer)
```
  
## Combine 'State-gov' and 'Local-gov' into 'SL-gov'

```{r}
  gov <- function(x){
    x <- as.character(x)
    if (x=='State-gov' | x=='Local-gov'){
      return('SL-gov')
    }else{
      return(x)
    }
  }
  adult$type_employer <- sapply(adult$type_employer, gov)
  table(adult$type_employer)
```
    
## Combine 'Self-emp-inc' and 'Self-emp-not-inc' into 'Self-emp'

```{r}
  self <- function(x){
    x <- as.character(x)
    if (x=='Self-emp-inc' | x=='Self-emp-not-inc'){
      return('Self-emp')
    }else{
      return(x)
    }
  }
  adult$type_employer <- sapply(adult$type_employer, self)
  table(adult$type_employer)
```

## Reduce Marital column to 'Married', 'Not-Married', 'Never-Married'

```{r}
  table(adult$marital)
  SO <- function(x){
    x <- as.character(x)
    if (x=='Married-spouse-absent' | x=='Married-AF-spouse' | x=='Married-civ-spouse'){
      return('Married')
    }else if (x=='Divorced' | x=='Separated' | x=='Widowed'){
      return('Not Married')
    }else{
      return('Never Married')
    }
  }
  adult$marital <- sapply(adult$marital, SO)  
  adult$marital <- factor(adult$marital)  
```

## Group the countries column  

```{r}
  continents <- function(x){
    x <- as.character(x)
    if(x=='France'|x=='Holand-Netherlands'|x=='Germany'|x=='Cambodia'|x=='Ireland'
       |x=='Poland'|x=='Yugoslavia'|x=='Portugal'|x=='Greece'|x=='Italy'|x=='Hungary'|x=='England'|x=='Scotland'){
      return('Europe')
    }else if(x=='Vietnam'|x=='Philippines'|x=='Iran'|x=='Hong'|x=='South'|x=='Philippenes'|x=='Taiwan'|x=='Thailand'|x=='China'|x=='India'|x=='Japan'|x=='Laos'){
      return('Asia')
    }else if(x=='El Salvador'|x=='Guatemala'|x=='Jamaica'
             |x=='Outlying-US(Guam-USVI-etc)'|x=='Trinadad&Tobago'|x=='Mexico'
             |x=='Honduras'|x=='Nicaragua'|x=='Cuba'){
      return('Central America')
    }else if(x=='Columbia'|x=='Ecuador'|x=='Peru'|x=='Dominican-Republic'){
      return('South America')
    }else if(x=='Puerto-Rico'|x=='United-States'|x=='Canada'|x=='Haiti'|x=='El-Salvador'){
      return('North America')
    }else{
      return(x)
    }
  }
  adult$country <- sapply(adult$country,continents)
  adult$country <- factor(adult$country)
  table(adult$country)  
str(adult)
```
  
  * Could have also created a list of countries (continent <- c(ctry,ctry,ctry)) and then check if the cell string is in the continent list.
  * if x in continent, return 'continent.'
  
# Missing Data

```{r}
library(Amelia)
```

## Convert ? to NA

```{r}
adult[adult == '?'] <- NA # Cheated
```

## Refactor data

```{r}
adult$education <- factor(adult$education)
adult$marital <- factor(adult$marital)
adult$occupation <- factor(adult$occupation)
adult$relationship <- factor(adult$relationship)
adult$race <- factor(adult$race)
adult$sex <- factor(adult$sex)
adult$country <- factor(adult$country)
```

## Check for NaNs using missmap

```{r}
missmap(adult)
```


```{r}
missmap(adult,y.at=c(1),y.labels = c(''),col=c('yellow','black'))
```

* Sample args. yellow = missing, black = observed.

## Get rid of NaNs

```{r}
adult <- na.omit(adult)
str(adult)
```


```{r}
missmap(adult,y.at=c(1),y.labels = c(''),col=c('yellow','black'))
```

# Exploratory Data Analysis

```{r}
str(adult)
library(ggplot2)
```

## Histogram of ages, by income

```{r}
pl <- ggplot(data=adult,aes(age))
pl + geom_histogram(aes(fill=income),color='black',binwidth = 1)
```

## Histogram of hours worked/week

```{r}
pl2 <- ggplot(data=adult,aes(hr_per_week))
pl2 + geom_histogram(aes(fill=income))
```

## Rename country column

```{r}
adult$region <- adult$country
adult <- select(adult,-country) # remove the country column using dplyr
pl3 <- ggplot(data=adult,aes(region))
pl3 + geom_bar(aes(fill=income),color='black')
```

* Can also use dplyr: adult <- rename(adult,region = country)

# Build a Model

## Split the data

```{r}
head(adult)
library(caTools)
set.seed(101)
split <- sample.split(adult$income,SplitRatio=0.7)  # 70% will be TRUE
ad.train <- subset(adult,split==T)
ad.test <- subset(adult,split==F)

str(ad.train)
# ?glm

adult.logit <- glm(income~.,family=binomial(logit),data=ad.train)
summary(adult.logit)
```

* Warning message says that model predicts data either 0 or 100% chance occurring in one of two categories

## Refine the model

### ?step 

Compares different logistic models. Tries a bunch of different combinations of variables in logistic regression model, keeps the ones that are significant. 

* Uses akaike information criterion to remove predictors not relevant to the model
* AIC estimates the relative amount of information lost by a given model: the less information a model loses, the higher the quality of that model. 
* In estimating the amount of information lost by a model, AIC deals with the trade-off between the goodness of fit of the model and the simplicity of the model. In other words, AIC deals with both the risk of overfitting and the risk of underfitting. 
* The smaller the AIC the better the model
* AIC only tells the qualitive of a model relative to other models
* Note that AIC tells nothing about the absolute quality of a model, only the quality relative to other models. Thus, if all the candidate models fit poorly, AIC will not give any warning of that. Hence, after selecting a model via AIC, it is usually good practice to validate the absolute quality of the model. Such validation commonly includes checks of the model's residuals (to determine whether the residuals seem like random) and tests of the model's predictions. For more on this topic, see statistical model validation. 
* Source: [Wikipedia](https://en.wikipedia.org/wiki/Akaike_information_criterion)

```{r}
new.model <- step(adult.logit)
summary(new.model)
```

## Analyze the model

### Create a Confusion Matrix

```{r}
fitted.probabilities <- predict(new.model,newdata=ad.test, type='response')
table(ad.test$income, fitted.probabilities > 0.5)
```

* True Pos + True Neg / everything
* Correctly predicted that 6282 observations had income < 50k
* Incorrectly predicted that 514 observations had income < 50k
* Correctly predicted that 1349 observations had income > 50k
* Incorrectly predicted that 903 observations had income > 50k

**Cheated below this, looked up answers**

### Check the performance of the model

#### Accuracy

```{r}
  fitted.results <- ifelse(fitted.probabilities > 0.5, 1, 0)
  misClasificError <- mean(fitted.results != ad.test)
  print(paste('Accuracy was: ', misClasificError))
  
  # Answer: 
  (6282+1349)/(6282+902+514+1349)
```

* Randomly putting observations into the two categories would have given us 50% accuracy
* Our model gave us 84% accuracy.
* Our model correctly predicted the classification of people's income based on various other variables 84% of the time.

#### Recall:

How many relevant items are selected?

```{r}  
  (6282)/(6282+514)
```

* 92% of the observations predicted as <50k category should have been in the <50k category.
* 8% of the observations predicted as <50k category should have been in the >50k category.

#### Precision:

How many selected items are relevant?

```{r}  
  (6282)/(6282+903)
```

* 87% of the observations that should have been in the <50k category were predicted as <50k.
* 13% of the observations that should have been in the <50k category were predicted as >50k.

# Conclusion:

Since we don't know what it will be used for, we can't say whether this is a good model or not.
