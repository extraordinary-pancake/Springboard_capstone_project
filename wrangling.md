### setting working directory and loading libraries ###
setwd('/Users/pkg/Springboard/Capstone project')
library(data.table)
library(dplyr)
library(tidyr)





### autos data set ###
## importing data ##
auto <- fread(input = 'autos.csv')

## cleaning the data ##
# what type of sellers are there, and how many of each are there?
table(auto$seller)
print(auto[auto$seller == 'gewerblich'])
# we only want private sellers, so let's clean those few
# that are commercial, and remove the column after that
auto <- auto[auto$seller == 'privat']
auto$seller <- NULL

# what about pictures?
table(auto$nrOfPictures)
# column is zero for all entries, and can safely be deleted
auto$nrOfPictures <- NULL

# people looking for cars (eg. not cars that are for sale)
# needs to be taken out of the data set
auto <- auto[auto$offerType == 'Angebot']
auto$offerType <- NULL

# see if cars with very high/low price are actual advertisements
print(auto[auto$price < 100])
print(auto[auto$price > 50000])

# cars that are advertised as either very expensive or very inexpensive
# seem to be priced incorrectly most times, but offers for car-for-car trades
# or other advertisements where the price is incorrect. These are therefore
# removed. In addition, since the goal is to create a search for most people,
# extremely expensive cars (even if it's a real ad) are removed.
auto <- auto[price < 50000 & price > 100]

# note that the dataset has not included whether prices are stated
# as 'VB' which stands for Verhandlungsbasis (negotiable) or not,
# which could have been interesting as not all advertisements
# include a price with 'VB'

# the variable 'abtest' seems to be linked to some sort av A/B-testing
# that may or may not be linked to the website rather than the
# advertisements. This won't be used, so it is deleted.
# dateCreated, dateCrawled, lastSeen, monthOfRegstration and postalCode
# are not attributes we will need for this analysis, so they are removed.
auto$abtest              <- NULL
auto$dateCrawled         <- NULL
auto$lastSeen            <- NULL
auto$dateCreated         <- NULL
auto$monthOfRegistration <- NULL
auto$postalCode          <- NULL

# cars advertised with an extremely powerful motor may not be
# accurately described in the ads. To control for this, cars that
# supposedly have more than 900 PS (about the same as 900 BHP) are
# removed. The same seems to be true for cars with 25 PS or less.
auto <- auto[powerPS <= 900 & powerPS > 25]

# the ads need to be connected by production year and brand/model
# if this information is not available, we'll drop the row
table(is.na(auto$brand))
table(is.na(auto$model))
# no empty values for brand or model

# the production year for the reliability data is between 2002 and 2015
# therefore cars that are newer or older than this are removed
# we'll assume that registration year from 'auto' and
# production year from 'rel' are the same, though this may not necessarily be true
auto <- auto[yearOfRegistration >= 2002 & yearOfRegistration <= 2015]

# fixing brand/model names in order to match with the other data sets
auto$brand <- ifelse(auto$brand == 'mercedes_benz', 'mercedes', auto$brand)
auto$brand <- ifelse(auto$brand == 'alfa_romeo', 'alfa romeo',  auto$brand)
auto$brand <- ifelse(auto$brand == 'land_rover', 'land rover',  auto$brand)
auto$model <- ifelse(auto$model == 'rangerover', 'range rover', auto$model)
auto$model <- gsub('_', ' ',      auto$model)
auto$model <- sub(' reihe', '',   auto$model)
auto$model <- sub(' klasse', '',  auto$model)
auto$model <- sub('1er', '1',     auto$model)
auto$model <- sub('3er', '3',     auto$model)
auto$model <- sub('5er', '5',     auto$model)
auto$model <- sub('6er', '6',     auto$model)
auto$model <- sub('7er', '7',     auto$model)
auto$model <- sub('mx', 'mx-5',   auto$model)
auto$model <- sub(' max', '-max', auto$model)

# translating properties in the data set
auto$notRepairedDamage <- sub('ja', 'yes',  auto$notRepairedDamage)
auto$notRepairedDamage <- sub('nein', 'no', auto$notRepairedDamage)

auto$gearbox <- sub('automatik', 'automatic', auto$gearbox)
auto$gearbox <- sub('manuell', 'manual',      auto$gearbox)

auto$fuelType <- sub('benzin', 'petrol',              auto$fuelType)
auto$fuelType <- sub('elektro', 'electric',           auto$fuelType)
auto$fuelType <- sub('lpg', 'liquefied petroleum',    auto$fuelType)
auto$fuelType <- sub('cng', 'compressed natural gas', auto$fuelType)

auto$vehicleType <- sub('cabrio', 'convertible',   auto$vehicleType)
auto$vehicleType <- sub('kombi', 'station wagon',  auto$vehicleType)
auto$vehicleType <- sub('kleinwagen', 'hatchback', auto$vehicleType)
auto$vehicleType <- sub('bus', 'mini-van',         auto$vehicleType)
auto$vehicleType <- sub('limousine', 'sedan',      auto$vehicleType)

# removing the "other" classifications
auto <- auto[brand != 'sonstige_autos']
auto <- auto[fuelType != 'andere']
auto <- auto[vehicleType != 'andere']
auto <- auto[vehicleType != '']
auto <- auto[gearbox != '']
auto <- auto[fuelType != '']
auto <- auto[notRepairedDamage != '']

# change 'character'-variables to factors
auto$vehicleType <- as.factor(auto$vehicleType)
auto$gearbox <- as.factor(auto$gearbox)
auto$fuelType <- as.factor(auto$fuelType)
auto$notRepairedDamage <- as.factor(auto$notRepairedDamage)





### reliability data set ###
## importing data ##
rel <- fread('reliability.csv')

## cleaning the data ##
# remove strings and change fault_rate, mileage and car production year into numbers
rel$mileage <- sub(' km', '', rel$mileage)
rel$mileage <- sub(' '  , '', rel$mileage)
rel$mileage <-     as.integer(rel$mileage)

rel$fault_rate <- sub('%', '', rel$fault_rate)
rel$fault_rate <-   as.numeric(rel$fault_rate)

rel$car_prod_y <- as.numeric(rel$car_prod_y)

# separate car_make_model into separate models
rel <- rel %>%
  separate(car_make_model, c('brand', 'model'), extra = 'merge')

# substitue the special letters for regular letters
rel$brand <- sub('Š', 's', rel$brand)
rel$brand <- sub('ë', 'e', rel$brand)
rel$model <- sub('é', 'e', rel$model)
rel$model <- sub('ó', 'o', rel$model)
rel$model <- sub('´', '',  rel$model)

# set all characters to lower case
rel$brand <- tolower(rel$brand)
rel$model <- tolower(rel$model)

# fixing the brand names that have two names or that needs fixing (mercedes benz, alfa romeo and bmw mini)
rel$brand <- ifelse(rel$brand == 'mercedes', 'mercedes benz', rel$brand)
rel$model <- ifelse(rel$brand == 'mercedes benz', sub('benz ', '', rel$model), rel$model)
rel$brand <- ifelse(rel$brand == 'mercedes benz', 'mercedes', rel$brand)

rel$brand <- ifelse(rel$brand == 'alfa', 'alfa romeo', rel$brand)
rel$model <- ifelse(rel$brand == 'alfa romeo', sub('romeo ', '', rel$model), rel$model)

rel$model <- ifelse(rel$brand == 'bmw' & rel$model == 'mini', 'cooper', rel$model)
rel$brand <- ifelse(rel$brand == 'bmw' & rel$model == 'cooper', 'mini', rel$brand)
rel$model <- ifelse(rel$brand == 'bmw' & rel$model == 'mini countryman', 'countryman', rel$model)
rel$brand <- ifelse(rel$brand == 'bmw' & rel$model == 'countryman', 'mini', rel$brand)

# Reshaping fault rate to facilitate further manipulation
rel_fr <- rel %>% 
  spread(report_year, fault_rate) %>%
  select(2:3,5:10) %>% 
  gather(report_year, fault_rate, 4:8, na.rm = TRUE) %>%
  distinct(brand, model, car_prod_y, report_year, .keep_all = TRUE) %>%
  spread(report_year, fault_rate)

even_fr <- (rel_fr$car_prod_y == '2002' | rel_fr$car_prod_y == '2004' | rel_fr$car_prod_y == '2006' |
            rel_fr$car_prod_y == '2008' | rel_fr$car_prod_y == '2010' | rel_fr$car_prod_y == '2014')

# creating means for reports that overlap (see the read.me file for the reason why)
rel_fr$mean1617 <- rowMeans(rel_fr[,c("2016", "2017")], na.rm = TRUE)
rel_fr$'2016'[even_fr] <- rel_fr$mean1617[even_fr]
rel_fr$'2017'[even_fr] <- rel_fr$mean1617[even_fr]

rel_fr$mean1516 <- rowMeans(rel_fr[,c("2015", "2016")], na.rm = TRUE)
rel_fr$'2015'[!even_fr] <- rel_fr$mean1516[!even_fr]
rel_fr$'2016'[!even_fr] <- rel_fr$mean1516[!even_fr]

rel_fr$mean1415 <- rowMeans(rel_fr[,c("2014", "2015")], na.rm = TRUE)
rel_fr$'2014'[even_fr] <- rel_fr$mean1415[even_fr]
rel_fr$'2015'[even_fr] <- rel_fr$mean1415[even_fr]

rel_fr$mean1314 <- rowMeans(rel_fr[,c("2013", "2014")], na.rm = TRUE)
rel_fr$'2013'[!even_fr] <- rel_fr$mean1314[!even_fr]
rel_fr$'2014'[!even_fr] <- rel_fr$mean1314[!even_fr]

rel_fr <- rel_fr %>%
  select(1:8) %>% 
  gather(report_year, fault_rate, 4:8, na.rm = TRUE)

# Reshaping mileage data to facilitate further manipulation
rel_mil <- rel %>% 
  spread(report_year, mileage) %>%
  select(2:3,5:10) %>% 
  gather(report_year, mileage, 4:8, na.rm = TRUE) %>%
  distinct(brand, model, car_prod_y, report_year, .keep_all = TRUE) %>%
  spread(report_year, mileage)

even_mil <- (rel_mil$car_prod_y == '2002' | rel_mil$car_prod_y == '2004' | rel_mil$car_prod_y == '2006' |
             rel_mil$car_prod_y == '2008' | rel_mil$car_prod_y == '2010' | rel_mil$car_prod_y == '2014')

# creating means for reports that overlap (see the read.me file for the reason why)
rel_mil$mean1617 <- rowMeans(rel_mil[,c("2016", "2017")], na.rm = TRUE)
rel_mil$'2016'[even_mil] <- rel_mil$mean1617[even_mil]
rel_mil$'2017'[even_mil] <- rel_mil$mean1617[even_mil]

rel_mil$mean1516 <- rowMeans(rel_mil[,c("2015", "2016")], na.rm = TRUE)
rel_mil$'2015'[!even_mil] <- rel_mil$mean1516[!even_mil]
rel_mil$'2016'[!even_mil] <- rel_mil$mean1516[!even_mil]

rel_mil$mean1415 <- rowMeans(rel_mil[,c("2014", "2015")], na.rm = TRUE)
rel_mil$'2014'[even_mil] <- rel_mil$mean1415[even_mil]
rel_mil$'2015'[even_mil] <- rel_mil$mean1415[even_mil]

rel_mil$mean1314 <- rowMeans(rel_mil[,c("2013", "2014")], na.rm = TRUE)
rel_mil$'2013'[!even_mil] <- rel_mil$mean1314[!even_mil]
rel_mil$'2014'[!even_mil] <- rel_mil$mean1314[!even_mil]

rel_mil <- rel_mil %>%
  select(1:8) %>% 
  gather(report_year, mileage, 4:8, na.rm = TRUE)

#combining the data frames, and removing the temporary data frames and vectors
rel <- cbind(rel_fr, mileage = rel_mil$mileage)
rm(rel_fr, rel_mil, even_fr, even_mil)

# creating a variable for the age of the car based on report year and production year
rel$report_year <- as.numeric(rel$report_year)
rel$car_age <- rel$report_year - rel$car_prod_y

# creating nationality markers for each brand
rel$nationality <- ifelse(rel$brand == 'audi'    | rel$brand == 'bmw'     | rel$brand == 'mercedes' |
                          rel$brand == 'opel'    | rel$brand == 'porsche' | rel$brand == 'volkswagen',
                          'german', 'others')

rel$nationality <- ifelse(rel$brand == 'citroen' | rel$brand == 'peugeot' | rel$brand == 'renault' |
                          rel$brand == 'seat',
                          'french', rel$nationality)

rel$nationality <- ifelse(rel$brand == 'honda'   | rel$brand == 'mazda'   | rel$brand == 'mitsubishi' |
                          rel$brand == 'nissan'  | rel$brand == 'subaru'  | rel$brand == 'suzuki' |
                          rel$brand == 'toyota',
                          'japanese', rel$nationality)





### crash test data set ###
## importing data ##
crash <- fread('crash.csv')

## transforming the data ##
# the data set only has useful information in every four rows
crash <- crash[seq(1, nrow(crash), 4),]

# separate brand_model into separate columns
crash <- crash %>%
  separate(brand_model, c('brand', 'model'), extra = 'merge')

# substitue the special letters for regular letters
crash$brand <- sub('ò', 'o', crash$brand)
crash$model <- sub('é', 'e', crash$model)
crash$model <- sub('`', '', crash$model)

# set all characters to lower case
crash$brand <- tolower(crash$brand)
crash$model <- tolower(crash$model)

# fixing the brand names that have two names or that needs fixing
crash$model <- sub('benz citan kombi', 'citan kombi', crash$model)
crash$brand <- sub('vw', 'volkswagen', crash$brand)
crash$brand <- sub('alfa', 'alfa romeo', crash$brand)
crash$model <- sub('3er', '3', crash$model)
crash$model <- sub('5er', '5', crash$model)

crash$model <- ifelse(crash$brand == 'alfa romeo', sub('romeo ', '', crash$model), crash$model)
crash$model <- ifelse(crash$brand == 'mercedes', sub('-klasse', '', crash$model), crash$model)

crash$brand <- ifelse(crash$brand == 'land', 'land rover', crash$brand)
crash$model <- ifelse(crash$brand == 'land rover', sub('rover ', '', crash$model), crash$model)

crash$brand <- ifelse(crash$brand == 'range', 'land rover', crash$brand)
crash$model <- ifelse(crash$brand == 'land rover', sub('rover', 'range rover', crash$model), crash$model)

crash$model <- ifelse(crash$brand == 'bmw' & crash$model == 'mini', 'mini', crash$model)
crash$brand <- ifelse(crash$model == 'mini', 'mini', crash$brand)
crash$model <- ifelse(crash$brand == 'mini' & crash$model == 'mini', '', crash$model)

crash$model <- ifelse(crash$brand == 'bmw' & crash$model == 'mini cooper', 'cooper', crash$model)
crash$brand <- ifelse(crash$model == 'cooper', 'mini', crash$brand)



# change "character"-variables to factors
crash$brand <- as.factor(crash$brand)
crash$model <- as.factor(crash$model)

# transforming the model year from characters to integers





### exploratory data analysis ###
## auto data set ##
library(ggplot2)
library(scales)

summary(auto)

# let's look at the kilometer and price variables
plot(auto$kilometer, auto$price)

# let's take a look at the kilometer variable
hist(auto$kilometer)

table(auto$kilometer)

hist(auto$yearOfRegistration[auto$kilometer == 150000])

ggplot(auto, aes(price, yearOfRegistration, col = vehicleType)) +
  geom_point(alpha = 0.01) +
  geom_smooth(se = TRUE) +
  facet_wrap(~vehicleType)

##reliability data set
summary(rel)

ggplot(rel, aes(car_age, fault_rate, col = brand), legend = FALSE) +
  geom_point(alpha = 0.2) +
  geom_smooth(se = TRUE) +
  facet_wrap(~brand) +
  theme(legend.position = 'none')

ggplot(rel, aes(mileage/1000, fault_rate, col = brand), legend = FALSE) +
  geom_point(alpha = 0.2) +
  geom_smooth(se = TRUE) +
  facet_wrap(~brand) +
  theme(legend.position = 'none')


#ggplot(rel, aes(mileage/1000, car_age, col = brand), legend = FALSE) +
#  geom_point(alpha = 0.2) +
#  geom_smooth(se = TRUE) +
#  facet_wrap(~brand) +
#  theme(legend.position = 'none')


ggplot(rel, aes(car_age, fault_rate, col = nationality), legend = FALSE) +
  geom_point(alpha = 0.2) +
  geom_smooth(se = TRUE) +
  facet_wrap(~nationality) +
  theme(legend.position = 'none')

ggplot(rel, aes(mileage/1000, fault_rate, col = nationality), legend = FALSE) +
  geom_point(alpha = 0.2) +
  geom_smooth(se = TRUE) +
  facet_wrap(~nationality) +
  theme(legend.position = 'none')

ggplot(rel, aes(car_age, fault_rate, col = nationality), legend = FALSE) +
  geom_smooth(se = FALSE)

ggplot(rel, aes(mileage/1000, fault_rate, col = nationality), legend = FALSE) +
  geom_smooth(se = FALSE)


## crash rating data set
summary(crash)