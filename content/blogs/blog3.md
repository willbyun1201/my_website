---
categories:
- ""
- ""
date: "2021-10-15"
description: 
draft: false
image: germany.jpg
keywords: ""
slug: lol_123
title: Airbnbs in Berlin
---
![The famous Brandenburger Tor in Berlin](https://i.natgeofe.com/n/9e138c12-712d-41d4-9be9-5822a3251b5a/brandenburggate-berlin-germany.jpg)

# Introduction

Airbnb is among the most frequently used platforms to book short-term rentals all over the world. In this analysis, we put ourselves in the shoes of a tech-savy couple that currently plans a trip to Berlin and wants to book an apartment via Airbnb. Having access to city-specific Airbnb data, the goal of the analysis is therefore to find a regression model, which predicts the price that this couple would have to pay for a 4-night stay at some Airbnb apartment.

There are three main steps to this analysis: (i) the data exploration and feature selection, (ii) the model selection and validation, (iii) a quick summary on findings and recommendation. 


# Data Exploration and Feature Selection

First, we import the relevant libraries and define some of the basic settings for the analysis.

```{r setup, include=FALSE}

# basic settings for the analysis
options(knitr.table.format = "html") 
knitr::opts_chunk$set(warning = FALSE, message = FALSE, 
  comment = NA, dpi = 300)
```


```{r load-libraries, echo=FALSE}

#import of relevant libraries
library(tidyverse) # the usual stuff: dplyr, readr, and other goodies
library(lubridate) # to handle dates
library(GGally) # for correlation-scatter plot matrix
library(ggfortify) # to produce residual diagnostic plots
library(rsample) # to split dataframe in training- & testing sets
library(janitor) # clean_names()
library(broom) # use broom:augment() to get tidy table with regression output, residuals, etc
library(huxtable) # to get summary table of all models produced
library(kableExtra) # for formatting tables
library(moderndive) # for getting regression tables
library(skimr) # for skim
library(mosaic)
library(leaflet) # for interactive HTML maps
library(tidytext)
library(viridis)
library(vroom)
library(stringr)
```

Next, we load the relevant data from insideairbnb.com. We cache this data so that it does not download every time that the document is knitted. 

```{r load_data, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

# we use cache=TRUE so we dont donwload the data everytime we knit
listings <- vroom("http://data.insideairbnb.com/germany/be/berlin/2021-09-21/data/listings.csv.gz") %>% 
       clean_names()

```

Now that the data is loaded, it helps to understand get a feel for the different variables. This part of the analysis is known as *Exploratory Data Analysis*. There are three substeps to this: 

## Looking at the raw values via the glimpse() command 

This tells us that we are looking at more than 18k Airbnb rentals in London, for which we have 74 variables. "Glimpse" also tells us that the variables are in all kinds of formats and likely require some manipulation for the actual analysis. For instance, "host_acceptance_rate" is in character format even though it is clearly a numeric variable. 

```{r glimpse, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}
#we use glimpse to get an overview of the data
#"kable" is used throughout the code to get clean html tables but not used here due to rendering issues
dplyr::glimpse(listings)
  #kable(caption = "Data Set Overview") %>%
  #kable_styling(bootstrap_options = "striped", full_width = F)
```

## Computing summary statistics of the variables of interest 

Using "favstats", we can get a feel for the values that individual variables take on. We chose "accommodates", "review_scores_rating", "number_of_reviews", and "beds" because our intuive sense was that these could all impact price in our eventual regression model.

From "favstats", we learn that the median for accommodates is 2, while the maximum goes up to 16. Also, the average Airbnb rental has a review score of c. 4.6. Finally, there is one Airbnb with 17 beds. These are just some exemplary figures from this descriptive analysis that help us to get a better feel for the data. Also notice that we cannot yet run the command on "price", since it is still saved as a character variable. 

Using "skim", we can see that there are certain variables where many values are missing (e.g., host_about). It is good to see that "price", our dependent variable in the regression model, is not missing for any of the rentals. 

```{r stats, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#running the favstats command on some variables of interest
accomodates<-mosaic::favstats(~accommodates,data = listings) %>% 
  kable(caption = "Accommodates") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
accomodates

#stats on review_scores_rating
reviews<-mosaic::favstats(~review_scores_rating,data = listings) %>% 
  kable(caption = "Review Scores Rating") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
reviews

#stats on number_of_reviews
reviewcount<-mosaic::favstats(~number_of_reviews,data = listings) %>% 
  kable(caption = "Number of Reviews") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
reviewcount

#stats on beds
beds<-mosaic::favstats(~beds ,data = listings) %>% 
  kable(caption = "Beds") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
beds

#getting a feel for missing values via the skim command
skim<-(skimr::skim(listings)) %>% 
  kable(caption = "Skim Summary") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
skim
  
```

In a next step, we transform "price" and some of the other variables into numerics. Also, we use "ggpairs" to get a feel for the correlation between some of the variables. For instance, it is interesting to find out whether "accommodates" correlates with "minimum_nights". Our intuition was that very large Airbnbs may have a higher minimum_nights number, since the cleaning effort for the host is increased.

The output below indicates that this intuition is not confirmed by the data, since there is actually a slightly negative correlation between minimum_nights and accommodates. As one would expect, a higher number of accommodates is correlated with a higher price. The density plots also help us see that for example review_scores_rating is left-skewed with a large number of rentals having very high ratings. Another interesting observation is that maximum_nights has a peak at 365, which means that many rentals cannot be booked for more than a year. This may be due to regulatory reasons, which keeps hosts to from renting out their properties for very long periods of time. 


```{r numerics, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#turning character variables into numeric variables
listings_new <- listings %>% 
  mutate(price = parse_number(price),        
         host_acceptance_rate = parse_number(host_acceptance_rate),
         host_response_rate = parse_number(host_response_rate))

#looking at the correlation for some of the variables 
listings_new %>% 
  select(price, accommodates, number_of_reviews,review_scores_rating,maximum_nights,minimum_nights) %>%
  ggpairs(aes(alpha = 0.2))+
  theme_bw() 
  

```

**These are some other questions that we can now answer**

*How many variables/columns? How many rows/observations?*

There are 74 variables and 18,288 observations. 

*Which variables are numbers?*

The following variables are numbers: id, scrape_id, host_id, latitude, longitude, accommondates, bathrooms, bedrooms, beds, price, maximum_nights, minimum_nights, number_of_reviews, number_of_reviews_ltm, number_of_reviews_130d, reviews_per_month,calculated_host_listings_count, calculated_host_listings_count_entire_homes, calculated_host_listings_count_private_rooms, calculated_host_listings_count_shared_rooms, reviews_per_month; 

*Which are categorical or factor variables (numeric or character variables with variables that have a fixed and known set of possible values?)*

The following variables are factors: host_response_rate, host_acceptance_rate, host_is_superhost, host_has_profile_pic, host_identity_verified, review_scores_rating, review_scores_accuracy, review_scores_cleanliness, review_scores_checkin, review_scores_communication, review_scores_location, review_scores_value, instant_bookable;

*What are the correlations between variables? Does each scatterplot support a linear relationship between variables? Do any of the correlations appear to be conditional on the value of a categorical variable?*

We were not able to observe strong correlation between any of the variables we selected for testing. It therefore appears that there is no linear relationship between the price, the accommodates, number of reviews, review scores rating, maximum or minimum nights. We log-transform the price variable at a later stage in order to normalize higher dispersion in very expensive rentals. 

We are now at the third step of the Exploratory Data Analysis section.

## Creating informative visualizations 

In this step, we plot some graphs in order to deepen our understanding of how different variables are distributed. We do not exclusively focus on variables and relationships that may impact price in our regression model, but rather try to get a feel of the dataset in general.

In the *first* chart, we learn that the distribution of beds varies with the nr. of accommodates of a specific rental. This is a rather straight-forward relationship, but it helps to start with something that confirms the intuition. In general, the interquartile range increases with the nr. of accommodates per Airbnb. One can assume that this is due to any extra beds in the form of sofa beds, which are likely more frequent in larger rentals. These more "improvised" beds are less likely to be found in smaller rentals. 

```{r property_type, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#plotting the relationship between accommodates and beds per Airbnb rental
ggplot(listings_new, aes(x=as.factor(accommodates), y=beds, color=accommodates))+
  geom_boxplot()+
  labs(title="1. Relationship between accommodates and beds per Airbnb rental",
       x="Accommodates",
       y="Nr. of beds")+
  theme(legend.position = "none")
```

The *second* chart tells us that superhosts (those with many rentals and a lot of experience) have a higher median review rating and a smaller interquartile range. One can assume that superhosts more consistently provide a high quality rental experience and therefore the spread of different ratings is smaller. We can also see that there are certain rentals for which the data set does not provide information on host status ("NA").

```{r property_type2, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}


#comparing Review Scores Rating between normal hosts and superhosts
ggplot(listings_new, aes(x=as.factor(host_is_superhost), y=review_scores_rating,color=host_is_superhost))+
  geom_boxplot()+
  labs(title="2. Relationship between host type and review rating",
       x="Is the host a superhost?",
       y="Review Scores Rating")+
  theme(legend.position = "none")
```

The *third* chart shows that review ratings among different room types vary. Shared rooms tend to have the worst ratings, which is likely due to the fact that the rental experience is dependent on another visitor. 

```{r property_type3, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}


#plotting relationship between room type and review ratings
ggplot(listings_new, aes(x=as.factor(room_type), y=review_scores_rating, alpha=0.01, color=room_type))+
  geom_boxplot()+
  labs(title="3. Relationship between room type and review ratings",
       x="Room type",
       y="Review Scores Rating")+
   theme(legend.position = "none")
```


The *fourth* chart shows the availability of rentals in different neighbourhoods. For example, in "Mitte" the availability is a lot lower than in Spandau. This is likely due to the fact that Mitte is in a very central location, where the demand for Airbnbs is really high.  

```{r property_type4, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}


#ploting availability of rental by neighbourhood
ggplot(listings_new, aes(x=as.factor(neighbourhood_group_cleansed), y=availability_365, color=neighbourhood_group_cleansed))+
  geom_boxplot()+
  labs(title="4. Availability of rentals by neighbourhood",
       x="Neighbourhood",
       y="Nr. of days available in the next 365 days")+
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))+
  theme(legend.position = "none") 
```

For the *fifth* chart, we filter out all the rentals that have a price >400 to avoid the distorting effect of very expensive rentals. In a later step, we will log-transform the price variable to achieve this. For now, the chart tells us that different room types have different price distributions. The hotel room category, where you also pay for using the amenities of the respective hotel, is unsurprisingly the most expensive one. What is more interesting is that shared rooms and private rooms have very similar distributions. One reason could be that shared rooms are a lot larger, which makes up for the lack of privacy in terms of price. 

```{r property_type5, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}


#filtering out the very expensive rentals to avoid strong skewness in the data 
cheap_listings <- listings_new %>% 
  filter(price<400)

#plotting the relationship between room type and price
ggplot(cheap_listings, aes(x=as.factor(room_type), y=price, color=room_type))+
  geom_boxplot()+
  labs(title="5. Relationship between room type and price",
       x="Room type",
       y="Price")+
  theme(legend.position = "none")
```

From the *sixth* chart, we learn that whether a host has a profile picture seems to impact the communication rating for a specific rental. Hosts that have a picture tend to score higher in this category. After all, Airbnb customers seem to like to see who their host is and incorporate that into the communication rating they give. 

```{r property_type6, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#Relationship between host demeanor and communication ratings
ggplot(listings_new, aes(x=as.factor(host_has_profile_pic), y=review_scores_communication, color=host_has_profile_pic))+
  geom_boxplot()+
  labs(title="6. Relationship between host demeanor and communication ratings",
       subtitle="Host profile picture is used as one indicator of host demeanor",
       x="Does the host have a profile picture?",
       y="Communication Rating")+
    theme(legend.position = "none")

```


Now, we focus on getting our data set in the right format for our regression analysis.  

First, we look at the variable `property_type`. We can use the `count` function to determine how many categories there are and their frequency. The four most common property types are entire rental units (~48.0%), private rooms in rental units (~35.7%), entire condominiums (~2.7%), and entire serviced apartments (~2.0%). Together, these property types make up for ~90.3% of the whole sample.

```{r property_type_2, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#counting the nr. and % of airbnbs across different property types
property_type <- listings_new %>% 
  filter(!is.na(property_type)) %>% 
  group_by(property_type) %>% 
  summarise(count=n(),) %>% 
  arrange(desc(count)) %>% 
  mutate(prop_in_percentage = count/sum(count)*100) %>% 
  kable(caption = "Property Type Overview") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
property_type

```


Since the vast majority of the observations in the data are one of the top four or five property types, we would like to create a simplified version of `property_type` variable that has 5 categories: the top four categories and `Other`. 


```{r property_type_four, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#simplifying the property_type variable to only include the top 4 types and group everything else in "Other"
listings_type <- listings_new %>%
  mutate(prop_type_simplified = case_when(
    property_type %in% c("Entire rental unit","Private room in rental unit", "Entire condominium (condo)","Entire serviced apartment") ~    property_type, 
    TRUE ~ "Other"
  ))

```

We can quickly check if the simplification worked.

```{r property_type_check, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#creating summary table for the simplified property_type variable 
listings_type %>%
  count(property_type, prop_type_simplified) %>%
  arrange(desc(n)) %>% 
  kable(caption = "Simplification of Property Type") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)

```

Next, we look at the Minimum_nihts variabe to only include listings in our regression analysis that are intended for travel purposes. At first, we check the distribution of minimum_nights. 

```{r property_nights, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#checking the distribution of minimum_nights
property_min_nights <- listings_type %>% 
  filter(!is.na(property_type)) %>% 
  group_by(minimum_nights) %>% 
  summarise(count=n(),) %>% 
  arrange(desc(count)) %>% 
  kable(caption = "Minimum Nights Data") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)

#find the most common values for "minimum_nights"
property_min_nights 

```

**We can now answer some more questions**

*What are the  most common values for the variable `minimum_nights`?*

The most common values for the variable `minimum_nights` are 2, 1, and 3 nights. This answer also makes sense, given many people use Airbnb for city trips, so the mininmal duration should not be too limited, but short stays and the cost or work to clean an Airbnb for a one night booking might not be worth it for many hosts.

*Is there any value among the common values that stands out?*

Especially the 30, 14 and 60 night minimum limits stand out at a first glance. These are usually longer-term Airbnbs that are used by interns or workers that are on assembly trips. It is also logical for some landlords to rent out their rooms over the longer term, as also for a longer stay the room only has to be tidied once. The highest minimum night requirement is 1,124 nights. This observation could be investigated as part of further analysis.


*What is the likely intended purpose for Airbnb listings with this seemingly unusual value for `minimum_nights`?*

The usual reasons for these longer minimum stays are to draw bookings from people that are on work projects, internships or are looking for a temporary stay while looking for a permanent accommodation. The benefit for the host is the lower frequency of cleaning and setting up the rooms. 

Next, we filter the airbnb data so that it only includes observations with `minimum_nights <= 4`.


```{r less than four, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#filtering the data to only include listings where the minimum_nights variable is <=4
listings_less_four <- listings_type %>% 
  filter(!is.na(minimum_nights)) %>% 
  filter(minimum_nights <= 4)

```
   
After making these adjustments, we want to analyze the distribution of rentals in Berlin. As the chart below shows, there are certain quarters with particularly many rentals. For instance, in Kreuzberg (a southern quarter in the city), there are many rentals available. This may be due to the types of buildings and the general infrastructure in the area. Kreuzberg is home to many restaurants and bars, which makes it an interesting area for tourists. Interestingly, there are fewer Airbnb in the heart of the city. Likely this is because the political district as well as many high-end hotels are located here, which leaves less room for Airbnbs.   

```{r, out.width = '80%',echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#data visualization that assigns each rental to a specific map location using longitude and latitude figures
leaflet(data = filter(listings, minimum_nights <= 4)) %>% 
  addProviderTiles("OpenStreetMap.Mapnik") %>% 
  addCircleMarkers(lng = ~longitude, 
                   lat = ~latitude, 
                   radius = 0.5, 
                   fillColor = "red", 
                   fillOpacity = 0.3, 
                   popup = ~listing_url,
                   label = ~property_type)
```

As we get closer to our regression model, we create a new variable called `price_4_nights` that uses `price`, and `accomodates` to calculate the total cost for two people to stay at the Airbnb property for 4 nights. This is the variable $Y$ we want to explain.

```{r cost, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#We filter out the room with one accommodate because 2 people require accommodates greater or equal to 2.
#We divide the total price by the number of people. And then we times 2 for 2 people, and times 4 for 4 nights.
cost_four_nights <- listings_less_four %>%
  filter(accommodates >1) %>%   
  mutate(price_4_nights = price/accommodates*2*4) 
  
```


In the next section, we create a new column called "log(price_4_nights)". We should use `log(price_4_nights)` because there are some outlier dentals in `price_4_nights` and using `log(price_4_nights)` could help normalize the dataset. In addition, the use of log can make the distribution behave better and help with finding the regression model. The regression model assumes normality and running a log-transformation helps to come closer to this assumption. It also ensures that the assumption of constant variance is met.  

We can use histograms to examine the distributions of `price_4_nights` and `log(price_4_nights)`.

```{r price 4 nights graph, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#cleaning the data to exclude entries with missing values in the relevant variables
cost_four_nights <- cost_four_nights %>% 
  mutate(log_price_four_nights= log(price_4_nights)) %>% 
  filter(!is.na(log_price_four_nights)) %>% 
  filter(!is.nan(log_price_four_nights)) %>% 
  filter(!is.infinite(log_price_four_nights)) %>% 
  filter(!is.na(prop_type_simplified)) %>% 
  filter(!is.na(number_of_reviews)) %>% 
  filter(!is.na(review_scores_rating)) 
  
#plotting the log-transformed price distribution
cost_four_nights_graph_log <- ggplot(cost_four_nights,aes(x=log_price_four_nights))+
  geom_histogram()+
  labs(title="Log plot for the variable price_four_nights",
       x="log(Price_four_nights)",
       y="Density")
cost_four_nights_graph_log

#plotting the non-log-transformed price distribution
cost_four_nights_graph <- ggplot(cost_four_nights,aes(x=price_4_nights))+
  geom_histogram()+
  labs(title="Plot for the variable price_four_nights",
       x="Price_four_nights",
       y="Density")
cost_four_nights_graph

  
```


# Model Selection and Validation

We now have all variables in the correct format and can start model selection and validation.We start with a model called `model1` with the following explanatory variables: `prop_type_simplified`, `number_of_reviews`, and `review_scores_rating`. Before running the first model, we split the data into a training and testing part. This will be key in order to test the explanatory power of the model when applying it to data that it has not been trained on - more on this later.


```{r split, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}
#cleaning the data set of NAs for final analysis
cleaned_cost_four_nights <- cost_four_nights %>% 
  filter(!is.na(number_of_reviews)) %>% 
  filter(!is.na(review_scores_rating)) %>% 
  filter(!is.na(room_type)) %>% 
  filter(!is.na(bedrooms)) %>% 
  filter(!is.na(beds)) %>% 
  filter(!is.na(accommodates)) %>% 
  filter(!is.na(host_is_superhost)) %>% 
  filter(!is.na(instant_bookable)) %>% 
  filter(!is.na(availability_30)) 

#setting a seed to ensure repeatability of the analysis
set.seed(1234)

#splitting the data into a training and testing set
train_test_split <- initial_split(cleaned_cost_four_nights, prop = 0.75)
train <- training(train_test_split)
test <- testing(train_test_split)

``` 




## Model 1

```{r model1, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#model 1 with explanatory variables: `prop_type_simplified`, `number_of_reviews`, and `review_scores_rating
model1 <- lm(log_price_four_nights~prop_type_simplified+
                   number_of_reviews+
                   review_scores_rating,
                   data = train)

#summary of model1 (command is used throughout the code)
msummary(model1)

#with the vif, we measure the amount of multicollinearity, which appears not to be a problem in this regression
car::vif(model1)

#checking the residuals using autoplot
autoplot(model1)


``` 


Because the dependent variable (i.e., price_4_nights) is log-transformed, the interpretation of the coefficients requires one additional step. The coefficient has to be exponentiated to reverse the log-transformation: (e^4.41x10^-2-1)x100=4.5087. This adjusted coefficient means that for every unit change in review_scores_rating, the price_4_nights increases by about 4.5%. This makes intuitive sense: the higher the rating, the more the host can charge. The t-value of >6 indicates that this relationship is statistically significant.

To interpret the coefficients, they have to be transformed like in the previous section. This leads to the following values: 

prop_type_simplifiedEntire rental unit: -11.66
prop_type_simplifiedEntire serviced apartment: 44.77
prop_type_simplifiedOther: -13.06
prop_type_simplifiedPrivate room in rental unit: -40.19

The variable "Entire condominium (condo)" is taken as the base value. Hence, the coefficients correspond to the %-change in price_4_nights over the base case that the Airbnb is of prop_type "Entire condominium (condo)". For instance, if you rent an "Entire serviced apartment", the price_4_nights is increased by 52% over the price that it would cost you if you had rented an "Entire condominium (condo)". The same logic also applies to the other variables, which are also all statistically significant. It also makes intuitive sense that for example "Entire serviced apartments" will be significantly more costly, because you pay for amenities such as regular cleaning or even breakfast. In a further analysis, one could split up the "Other" category further, to find out more about other property types. 

Next, we want to determine if `room_type` is a significant predictor of the cost for 4 nights, given everything else in the model. We fit a regression model called model2 that includes all of the explanatory variables in `model1` plus `room_type`. 

## Model 2

```{r model2 with room type, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

# Find the top 4 most popular room type, which turns out that there are only 4 room types
popular_room_type <- train %>% 
  group_by(room_type) %>% 
  summarise(count = n()) %>% 
  arrange(desc(count)) %>% 
  kable(caption = "Room Types") %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
popular_room_type

#running the model to test for multicollinearity
model2_test<- lm(log_price_four_nights~prop_type_simplified+number_of_reviews+review_scores_rating+room_type,data = train)

#summary of model
msummary(model2_test)

# As the VIF for property_type and room_type is 12 which is above the 10 threshold. We will remove property_type for next models.
car::vif(model2_test)

#model_2 with room_type but without property_type
model2<- lm(log_price_four_nights~+number_of_reviews+review_scores_rating+room_type,data = train)
msummary(model2)

#we measure the amount of multicollinearity
car::vif(model2)

#checking the residuals using autoplot
autoplot(model2)

``` 

There is some multicollinearity between room_type and property_type, as one would expect. Because room_type adds more explanatory power to the model, we therefore exclude property_type from the model. All room_type variables are statistically significant and tell us different things about price_4_nights: 

(i) Hotel rooms increase the price for an Airbnb over the base case scenario that an entire home is rented (the excluded variable). This makes sense, since the tenants also pay for the additional hotel infrastructure that they get to use.
(ii) Renting a private room reduces the price of the rental compared to the base case. This also makes intuitive sense, since these rooms may be in the hosts own house or otherwise less valuable than renting an entire apartment.
(iii) Renting a shared room also reduces price, which makes sense since the cost of the rental is split among a greater number of heads. 

We now go on by adding other variables to the model to increase its explanatory power. Currently, we can only explain c. 19% of the variation in price with our model. We therefore include more variables to improve on this. Model3 includes the number of `bathrooms`, `bedrooms`, `beds`, and size of the house (`accomodates`) of a rental.

## Model 3

```{r model3 with number of different rooms, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#for our dataset, bathrooms is NA for all entries, so we don't take bathroom into consideration
train <- train %>% 
  filter(!is.na(bedrooms)) %>% 
  filter(!is.na(beds)) %>% 
  filter(!is.na(accommodates))

#running regression model3
model3 <- lm(log_price_four_nights~+number_of_reviews+review_scores_rating+room_type+bedrooms+beds+accommodates,data = train)

#model summary
msummary(model3)

#measuring multicollinearity (as the VIF are below 5, we believe they are not co-linear)
car::vif(model3)

#checking the residuals using autoplot
autoplot(model3)
``` 

Based on this model, we learn that bedrooms and the size of the house are significant predictors of price_4_nights, which can be seen from a high absolute t-statistic. As the nr. of bedrooms increases, the price of the rental also increases. As house size increases, the price per person actually decreases (remember that we divided by "accommodates" when adjusting the price_4_nights variable). This makes sense, since the price is then shared among a greater number of heads. Beds is not a statistically significant predictor of price_4_nights. Interestingly, there is some multicollinearity between bedrooms, beds, and accommodates but not enough to disregard the model. 

Comparing Model3 to Model2, we increase the adjusted R-squared to 0.26, which means that we can now explain more than a quarter of the variation in price. In Model4, we add the impact of the superhost variable `(host_is_superhost`) and check whether they can command a pricing premium, after controlling for other variables.

## Model 4

```{r model4 with super host, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#running regression model4
model4 <- lm(log_price_four_nights~number_of_reviews+review_scores_rating+room_type+bedrooms+beds+accommodates+host_is_superhost,data = train)

#summary of model4
msummary(model4)

#measuring multicollinearity (as the VIF are below 5, no variables are removed)
car::vif(model4)  

#checking the residuals using autoplot
autoplot(model4)

``` 

Based on this model, superhosts charge a pricing premium, which can be seen from the positive coefficient and the high t-statistic. This makes sense, since these kinds of hosts are typically very professional in the way that they manage their apartments, which translates into higher customer value and thereby the ability to charge higher prices.  

For Model5, we include the fact that some hosts allow you to immediately book their listing (`instant_bookable == TRUE`), while a non-trivial proportion don't. 

## Model 5

```{r model5 with instant_bookable, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#running regression model5
model5<- lm(log_price_four_nights~number_of_reviews+review_scores_rating+room_type+bedrooms+beds+accommodates+host_is_superhost+instant_bookable,data = train)

#model summary
msummary(model5)

#measuring multicollinearity (as the VIF are below 5, no variables are removed)
car::vif(model5) 

#checking the residuals using autoplot
autoplot(model5)
``` 

As can be seen from the summary statistics, the  variable "instant_bookable" is also a significant predictor of price_4_nights. The regression analysis reveals that when controlling for the other listed variables, a rental with an instant-booking option is c. 6.09% more expensive than one without. The customer pays a premium for instant confirmation that the rental can be booked. The t-statistic for the variable is high and there is little multicollinearity with other variables, which is why it should be kept in the model. 

For Model6, we look at neighbourhoods. For all cities, there are 3 variables that relate to neighbourhoods: `neighbourhood`, `neighbourhood_cleansed`, and `neighbourhood_group_cleansed`. There are typically more than 20 neighbourhoods in each city, and it wouldn't make sense to include them all in the model. Instead, we manipulate the `neighbourhood_group_cleansed` variable and divide neighbourhoods into the following 4 groups:

City West: Steglitz - Zehlendorf, Spandau, Charlottenburg-Wilm.
City North: Reinickendorf, Pankow, Lichtenberg
City Central: Mitte, Friedrichshain-Kreuzberg
City East: Marzahn - Hellersdorf, Treptow - K??penick, Neuk??lln, Tempelhof - Sch??neberg

This grouping is based on (i) the geographic location of the neighbourhoods and (ii) the judgement of a Berlin local. It pays special consideration for the particularly sought-after quarters of "Mitte" and "Friedrichshain-Kreuzberg", which create their own group. 

## Model 6

```{r model6 neighbourhood, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#counting the nr. of different neighborhoods across the data se
train_model6_test <- train %>% 
  select(neighbourhood_group_cleansed) %>% 
  group_by(neighbourhood_group_cleansed) %>% 
  summarise(count = n())

#Create a new column to group the `neighbourhood_group_cleansed` into 4 groups 
result <- c()
for (i in train$neighbourhood_group_cleansed){
  word <- ''
  if (i %in% c("Steglitz - Zehlendorf", "Spandau", "Charlottenburg-Wilm.")){
    word <- "City West"
  }else if (i %in% c("Reinickendorf","Pankow","Lichtenberg")){
    word <- "City North"
  }else if (i %in% c("Mitte","Friedrichshain-Kreuzberg")){
    word <- "City Central"
  }else if (i %in% c("Marzahn - Hellersdorf", "Treptow - K??penick", "Neuk??lln", "Tempelhof - Sch??neberg")){
    word <- "City East"  
  } 
  result <- c(result, word)
}

#add the new column to the dataset
train$areas<- result

#running regression model6
model6<- lm(log_price_four_nights~number_of_reviews+review_scores_rating+room_type+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+areas,data = train)

#model summary
msummary(model6)

#checking for multicollinearity
car::vif(model6) 

#checking the residuals using autoplot
autoplot(model6)

``` 

The regression table confirms that neighbourhood is indeed a significant driver or price. City_Central is the base category for the analysis and is omitted in the model. Relative to this base case, all other neighbourhoods are cheaper. For example, an Airbnb in City_East will be c. 15.3% less expensive compared to the same apartment in City_Centre. Taking Berlin's history into account, this makes logical sense. The eastern part of the city is the former DDR part, where prices tend to be lower.   

For Model7, we include the effect of `avalability_30` and `reviews_per_month` on price.

The variable "availability_30" is also a significant predictor of price_4_nights. The t-statistic is very high and the coefficient is positive, which means that, controlling for all the other variables, the impact of availability in the next month on price is positive.  

The variable reviews_per_month does not seem to be a significant predictor as the t value is less than 2. This makes sense, since number of reviews per month are not necessarily related to the quality of the properties and therefore the price of the properties. A cheap rental could equally well have a high number of reviews per month as a medium-priced or more expensive rental. Therefore, this variable is removed from the final version of model 7, along with "beds" and "instant_bookable" which also have a t-statistic <2 and is thereby not statistically relevant. 


## Model 7

```{r model7, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

# Evaluate the effect of `avalability_30` and `reviews_per_month`
model7_test <- lm(log_price_four_nights~number_of_reviews+review_scores_rating+room_type+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+reviews_per_month+availability_30+areas,data = train)
msummary(model7_test)

# As variable reviews_per_month, beds, and instant_bookable do not seem to be a significant predictor, we remove it from model 7
model7 <- lm(log_price_four_nights~number_of_reviews+review_scores_rating+room_type+bedrooms+accommodates+host_is_superhost+availability_30+areas,data = train)

#model summary
msummary(model7)

#checking for multicollinearity
car::vif(model7) 
``` 

As the summary statistics above indicate, our final model comprises of statistically significant variables only and has an adjusted R-squared value of 0.378. This means that the model helps to explain 37.8% of the variation in the log-transformed price. At first, this seems like a mediocre model, since almost 2/3 of the variation in price remains unexplained. However, given the fact that rental prices are very subjective to their specific location (as opposed to the mere neighborhood), the quality of the amenities, the last date of redevelopment, and many other factors, we consider an R-squared of almost 40% as satisfactory. For instance, the addition of the simplified neighborhood variable only added c. 2 percentage points of explanatory power in our analysis and we are confident that in a future investigation one should put more emphasis on this variable and possibly consider factors such as "distance to public transport" or "distance to airport". 


## Model 7 RMSE

Next to the looking at explanatory power, we should also analyze our model using RMSE. This analysis reveals whether the model actually works on unknown data, or whether it is overfitted to the specifics of the training data. The analysis below proves that model 7 is a good model based on two things. First, rmse_train value is small (0.419), which means predicated value and actual value are pretty close. Second, the difference (0.002) between rmse_train and rmse_test is small, which means it is a generalized model and can be applied to not only the training set, but also has the predict power to new data. 

```{r rmse, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#pulling the rmse from the training data
rmse_train <- train %>% 
  mutate(predictions = predict(model7,.)) %>% 
  select(predictions,log_price_four_nights) %>% 
  mutate(squared_error = (predictions - log_price_four_nights)^2) %>% 
  summarise(rmse = sqrt(mean(squared_error))) %>% 
  pull()
rmse_train


# Add the `area` column to the train dataframe
result <- c()
for (i in test$neighbourhood_group_cleansed){
  word <- ''
  if (i %in% c("Steglitz - Zehlendorf", "Spandau", "Charlottenburg-Wilm.")){
    word <- "City West"
  }else if (i %in% c("Reinickendorf","Pankow","Lichtenberg")){
    word <- "City North"
  }else if (i %in% c("Mitte","Friedrichshain-Kreuzberg")){
    word <- "City Central"
  }else if (i %in% c("Marzahn - Hellersdorf", "Treptow - K??penick", "Neuk??lln", "Tempelhof - Sch??neberg")){
    word <- "City East"  
  } 
  result <- c(result, word)
}

#add the new column to the dataset
test$areas<- result


#pulling the rmse from the testing data
rmse_test <- test %>% 
  mutate(predictions = predict(model7,.)) %>% 
  select(predictions,log_price_four_nights) %>% 
  mutate(squared_error = (predictions - log_price_four_nights)^2) %>% 
  summarise(rmse = sqrt(mean(squared_error))) %>% 
  pull()
rmse_test
``` 

In a future study, it would be interesting to apply the same model to different cities and test how it performs there. One can hypothesize, that in different regions of the world, some variables may have a particularly strong effect on price. For example, in regions that are more unsafe or more heterogeneous than Berlin, the neighborhood variable may be of greater significance. In this case, the RMSE would reveal that the model must be adapted because the accuracy in the test data set would be a lot lower than in the training data. 


## Summary of All Models

To provide an overview of the models that we worked with, we can create a summary table of the important parameters. From this table, we can see that between model 2 and 3, as well as between model 6 and 7, we could increase the explanatory power significantly. Please see the earlier sections for more comparison between the differences in the models. 

```{r summary table, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#Creating a summary model for the 7 models we worked on. 
summary_table <- huxreg(model1, model2, model3,model4,model5,model6,model7,
                         statistics = c('#observations' = 'nobs', 
                                'R squared' = 'r.squared',
                                'Adj. R Squared' = 'adj.r.squared', 
                                'Residual SE' = 'sigma'), 
                         bold_signif = 0.05, 
                         stars = NULL)%>%
                         kable(caption = "Comparison of models") %>%
                         kable_styling(bootstrap_options = "striped", full_width = F)

#output table
summary_table

``` 


## Test on a Specific Target Listing

We now apply the following criteria to find our target listings:

- Review score value is higher than 90% of full score 

- number of reviews are larger than 10

- It has a private room 

- The host identity is verified and is a super host

- It is in the neighborhood of Friedrichshain-Kreuzberg

- Thee are two beds.

```{r Find our vocation Airbnb, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#filtering the data set for relevant rentals
our_target_listings <- cost_four_nights %>% 
  filter(review_scores_value>= 4.5) %>% #have a rating of at least 90% of score 5
  filter(number_of_reviews >= 10) %>% 
  filter(room_type == "Private room") %>% 
  filter(host_identity_verified==TRUE) %>% 
  filter(neighbourhood_group_cleansed== "Friedrichshain-Kreuzberg") %>%  #We would like to stay here as it is one of the coolest neighborhood
  filter(host_is_superhost == TRUE) %>% 
  filter(beds == 2)#We would like to have 2 beds 

# Add area column to our target listing
result <- c()
for (i in our_target_listings$neighbourhood_group_cleansed){
  word <- ''
  if (i %in% c("Steglitz - Zehlendorf", "Spandau", "Charlottenburg-Wilm.")){
    word <- "City West"
  }else if (i %in% c("Reinickendorf","Pankow","Lichtenberg")){
    word <- "City North"
  }else if (i %in% c("Mitte","Friedrichshain-Kreuzberg")){
    word <- "City Central"
  }else if (i %in% c("Marzahn - Hellersdorf", "Treptow - K??penick", "Neuk??lln", "Tempelhof - Sch??neberg")){
    word <- "City East"  
  } 
  result <- c(result, word)
}

#add the new column to the dataset
our_target_listings$areas<- result


``` 

We believe the best model is model 7 as it has the highest adjusted square and the lowest residual SE. We will apply model 7 for price prediction and an interval for the lower and upper bound. Based on the output below, we can see that the prices range from c. 130??? to 310???. However, the "lwr" and "upr" columns tell us that we the spread for each estimated price is extremely high. This should not come as a surprise, since the explanatory power of our model is limited to less than 40%. In the next chapter, we briefly discuss options on how to improve on this in a future analysis. 

```{r prediction, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE}

#predicting the price for our target listings
prediction<-exp(predict(model7,our_target_listings,interval = "prediction")) %>% 
    kable(caption = "Price Prediction") %>%
    kable_styling(bootstrap_options = "striped", full_width = F)
prediction
  
test_rentals<-augment(model7, newdata=our_target_listings) %>% 
    kable(caption = "Price Prediction") %>%
    kable_styling(bootstrap_options = "striped", full_width = F)

``` 

# Findings and Recommendation

In this final section, we summarize the results of our selected model and discuss possible steps that could further improve the analysis. 

As mentioned in the introduction, the overall goal of this analysis was to find a set of variables that would help us to predict Airbnb rental prices in Berlin. Our final model defines the following variables as statistically significant drivers of said rental prices: 

- Number of reviews
- Review scores rating
- Room type
- Nr. of bedrooms
- Nr. of people the rental accommodates
- Host status (superhost/non-superhost)
- Instant bookability
- Availability in the next 30 days
- Location of the rental

The coefficients for each of these variables tell us how rental prices are impacted. For instance, the 1.9 coefficient for "Nr. of bedrooms" tells us that rental prices tend to increase with a higher nr. of bedrooms. The standard error for each of the coefficient estimates provides us with an idea of how far we are away from the "true" value of the coefficient. Whenever the the ratio of coefficient to standard error is >2, we can be relatively sure that the variable is in fact statistically significant. In our model, this is the case for all variables. 

If we look at the p-value of the overall model (c. 2.2*e^-16), we notice that this is extremely small. This simply means that our overall model helps to explain rental prices with almost absolute certainty. As previously states, the adjusted R-squared (adjusted for the nr. of variables) tells us how much of the variation in price can be explained, where 37.8% is clearly substantial.

Our RMSE analysis also showed us that the model works well on different sub groups of the data. In a further analysis, it would be interesting to apply the model to other cities as well and compare how the explanatory power changes and if any variable becomes an insignificant predictor of price.

Additionally, the completeness of the data set could be improved in further analysis. We had to leave out thousands of rentals because they missed the relevant values for our chosen predictor variables. 

Finally, we recommend to be aware of the impact of seasonality and weekday on prices. There are certainly some season where demand for rentals is particularly high (e.g., on national holidays or during summertime). The same goes for certain days of the week (e.g., the weekend being in higher demand than weekdays). In a next analysis, we would therefore like to focus on the impact of these time-related variable on the variation in price. 


# Acknowledgements

The data for this project is from [insideairbnb.com](insideairbnb.com). 