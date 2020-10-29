# DemandWeatherPrice
Project - Fundamentals of Data Science  
Alex Heilman  
Kushagra Agarwal  

### Overview
Prediction of energy price and electric demand are immensely valuable to industry experts. This information is used for power generation planning and to execute electric power futures in the wholesale energy market. Bulk energy storage is scarcely implemented, meaning electric generation and demand must always balance. Due to poor generation planning and demand prediction, arbitrage opportunities occasionally emerge in the market. These opportunities can be capitalized on if generation assets in the power pool respond quickly. As utilities perform integrated resource planning, future system demand predictions provide key insights and identify necessary infrastructure upgrades. A robust prediction model for both energy price and electric demand answers critical questions for several applications.  

### Investigation and Exploration 
This dataset is an hourly record of electric generation by each source, actual electric demand, the weather conditions in five Spanish cities, and the price of energy in Spain over four years. The dataset was obtained from [Kaggle](https://www.kaggle.com/nicholasjhana/energy-consumption-generation-prices-and-weather) . Data regarding energy generation was acquired from ENTSO-E, the European Network of Transmission System Operators for Electricity, and the rates were collected from the Spanish operator Red Electric España. Weather data was acquired from Open Weather API, an open source collection of publicly available weather station data. Weather data for the five largest Spanish cities were used – Madrid, Barcelona, Bilbao, Seville, and Valencia. Although the accuracy of these datasets cannot be determined, the sources appear to be reputable.  

In order to use these six different time-series datasets, meticulous wrangling and synchronization was performed. This combined dataset contains hourly generation by type, energy price, electric consumption, and weather readings for January 1, 2015 through December 31, 2018. Due to the nature of time-series data, random selection of training data would not be suitable for this application. The training set was selected sequentially using the first 80% of the data. The intention is to use earlier data to test on later data.  

Several instances of duplicated weather entries were deleted during this process. Within the electric data, N/A values were corrected with mean imputation of the training set only, as to not improperly bias the data. If these rows were to have been deleted, certain hours and days would be weighted unintentionally. These N/A values were examples of “missingness completely at random”, because there was no systematic explanation and their occurrence had nothing to do with the covariates. Multiple weather covariates were determined to be irrelevant for this analysis. Colinear temperature covariates were deleted from the dataset. The remaining temperature covariates for each city were transformed from units of Kelvin into units of Celsius for easier interpretation. Weather ID number, weather description, and weather icon were all deleted. Threehour rainfall and snowfall readings were deleted in favor of the more applicable one-hour readings.  

The original timestamp for the observations was of the form “2015-01-31 17:00:00+01:00”, giving little flexibility for analysis. This covariate was transformed into four separate covariates: the year ranging from 2015 to 2018, the month ranging from 1 to 12, the day ranging from 1 to 31, and the hour ranging from 1 to 24. This crucial transformation informed the 1  relevant patterns between demand, price, time of day, and month of the year. This supported the expectation that higher electric demand correlated with morning hours, after-work hours, hot summer months, and cold winter months.  

Ggpairs was used to identify correlations between different covariates and implement interaction terms in the regression model. Strong correlations were observed in the generation covariates (natural gas, coal, etc.) and interaction terms were added accordingly. Temperature and relative humidity were negatively correlated - a relationship established by physics. Both temperature and pressure were highly correlated amongst the five cities. Due to their close geographic proximity, this result was expected. A strong correlation was observed between electric demand and temperature for all five cities. This result was anticipated, as electric heating and air conditioning are strong drivers of residential electric demand. However, a surprising result was Madrid and Barcelona temperatures correlating the least with electric demand. Given that these are the two largest cities in Spain by population, it was assumed they would demonstrate the strongest correlation. Perhaps the most spurious correlation observed was between the barometric pressure of Bilbao and price of energy.  

A strong positive correlation between price and demand was observed, although not immediately obvious. The coefficient for demand in kilowatts was positive, indicating the relationship we assumed, but of surprisingly low magnitude. The low magnitude was a result of the relative scaling and the strong correlation was evident after standardization. This result is slightly counterintuitive but reflects a common pricing structure designed to disincentivize high demand during peak hours. Figure 1, a scatterplot illustrating this relationship, is included below.  
| Figure 1 <br/> *Electric Demand vs. Price of Energy* |
| --- |
| <img align="center" width="450" height="450" src="/Figures/Figure_1.png"> |

The chosen continuous response variable is the energy price for small consumers (less than ten kilowatts) in euros per megawatt hour. The prediction of this outcome is valuable to the energy futures market, where day-ahead purchases are made in the day-ahead market. The binary response variable is an indicator of the demand at a given hour and takes a value of 1 only if the demand for that hour is expected to be greater than the previous hour - otherwise its value is 0.  

### Predictive Regression 
To ensure accurate predictions of energy price given the covariates of generation and weather at a specific hour, the objective of the predictive regression model was to minimize prediction error. Because OLS often results in a low prediction error for linear population models, the prediction error for a model with all single-ordered covariates was used as the baseline for comparison against other models. The estimate of the prediction error, however, could not be carried out by general cross-validation due to the time-series data. Prediction involving time-series data usually requires prediction for future observations based on past events. As cross validation does not maintain such sequentiality, the measured error would be invalid. Instead, a simple trainvalidate approach was employed after splitting the training dataset further into separate training and validation sets with an 80/20 ratio. After training a model with the new training dataset, the root-mean-squared error (RMSE) was calculated between actual price in the validation set and the predicted price generated by the model.  

Given that many covariates demonstrated high correlation with each other, it was expected that the inclusion of interaction terms in the regression model would decrease the prediction error. However, given the presence of 55 covariates in the dataset and 1485 corresponding pairs, identifying the relevant interaction terms was time-consuming and difficult. After the addition of interaction terms for all covariate pairs with high correlation as observed using ggpairs, rigorous optimization of the model was carried out by checking the new RMSE resulting from the exclusion of each interaction term. The same process was also applied to check for beneficial logarithmic transformations of the covariates. The model with the least validation error, labelled LM1 in the Appendix, contains mostly interaction terms and a single log transform, in addition to the other single-ordered covariates. The RMSE of LM1 was 7.99 while the baseline was 9.27. This model was chosen as the “best” regression model and is likely to provide an approximate test error higher than 7.99, since train-test generally underestimates the true prediction error and the number of observations in the validation set was low.  

This RMSE optimization provides an overly optimistic estimate of test error. To ensure that the reduction in RMSE was indeed justified and not simply due to the validation set being highly favorable to the model, bias-variance decomposition analysis was carried out. The density plots of the RMSE for LM1 and the baseline model (Figure 2 below) clearly indicate that LM1 has a lower bias than the baseline model, although LM1 appears to have higher variance. It is also inferred from the high bias in both models that key population covariates are missing from the dataset. Apart from OLS, k-nearest neighbors(k-NN) and ridge regression were also pursued. Since the linearity of the population model was uncertain, k-NN was implemented to circumvent assumptions about the population model. However, k-NN was not only slow but also seemed to saturate around an RMSE of 10, illustrated by Figure 3 below. Similarly, Ridge regression was unsuccessful in reducing the validation RMSE further than LM1.  
| Figure 2 <br/> *Density Plot of RMSE for Improved Model and Baseline* |
| --- |
| <img align="center" width="450" height="450" src="/Figures/Figure_2.png"> |  

| Figure 3 <br/> *RMSE vs. K for k-NN* |
| --- |
| <img align="center" width="450" height="450" src="/Figures/Figure_3.png"> |



### Predictive Classification 
The classifier was designed to predict if the demand for electricity at a given hour is greater than it was during the hour immediately preceding it. Such a classifier could be valuable for a utility to quickly anticipate a rise in demand, based on the amount of energy generated by different sources and the weather conditions at a certain time. While both false positives and negatives are undesirable, closer analysis reveals that false negatives would be much more consequential to the utility than false positives. If a rise in demand is missed, additional energy would have to be purchased from elsewhere at higher market prices, whereas surplus energy generation could either be stored or sold at market value. Therefore, instead of minimizing the average 0-1 loss, the objective of the classifier was to maximize the true positive rate (TPR) while keeping the false positive rate (FPR) below manageable limits.  

A new covariate was created in both the training and validation sets, with a value of 1 if the demand for the ith observation was greater than the demand in the (i-1)th observation, and value 0 otherwise. The missing element in the first observation of the training set was replaced by the majority between 0’s and 1’s in the covariate column. A logistic regression model containing all covariates was then trained on the training set, the corresponding log odds ratios were collected for the validation set, and the TPR and FPR were calculated for a threshold 0 ≤ t ≤ 1. The ROC curve, shown in Figure 4 below, was obtained by plotting the TPR vs FPR values generated while sweeping through t. The ROC curve indicates that, starting from the top-right corner of the plot, around t=0.25, while the TPR reduces by 1 unit, the FPR reduces by three units. The TPR and FPR values at t=0.25 were 0.86 and 0.64 respectively and were set as the baseline.  
| Figure 4 <br/> *ROC Curves for Base Model* |
| --- |
| <img align="center" width="450" height="450" src="/Figures/Figure_4.png"> |

It was expected that a logistic regression model containing the same terms as the formula of LM1 would have an improved classification. This expected result was not observed. Despite the optimization of the formula, 0.87 and 0.65 were the best values of TPR and FPR that could be achieved. Figure 5 below contains the ROC curves for the baseline model and the optimized logistic regression model plotted together.  
| Figure 5 <br/> *ROC Curves for Base Model and Improved Logistic Regression Model* |
| --- |
| <img align="center" width="450" height="450" src="/Figures/Figure_5.png"> |

Finally, the k-NN algorithm was applied as a classifier. By cycling through different values of k and L, it was found that k=25, L=25 resulted in TPR=1, and FPR=0. Hence this model, named ‘k-NN’ in the Appendix, classifies the validation set perfectly. Although this result is not indicative of a model generalizing perfectly for all test cases, it is certainly an indication of good performance. It should also be noted that the number of observations in the validation set was small. The TPR and FPR are expected to decrease and increase respectively during testing, but the classifier performs commendably, nevertheless.  

The OLS regression model with interaction terms and the k-NN binary classifier model both performed well in the validation stage of model testing. Due to validation bias, it is expected that these models will likely perform with higher test RMSE than observed by the validation RMSE. Even with this likely outcome, both models are performing well for this stage. 

### Test Prediction
In the first phase of the project, two predictive models were developed, with the goal of minimizing prediction error. The models made predictions using several weather and energy generation covariates. The first model used linear regression to predict the price of energy for small consumers (less than ten kilowatts) in euros per megawatt hour. The second model used k nearest neighbors (k-NN) to predict whether electric demand in kilowatts would be higher than the demand of the previous hour. The estimate of the prediction error, however, could not be carried out by general cross-validation due to the time-series data. Prediction involving time-series data usually requires prediction for future observations based on past events. As cross validation does not maintain such sequentiality, the measured error would be invalid. Instead, a simple train-validate approach was employed after splitting the training dataset further into separate training and validation sets with an 80/20 ratio. After training a model with the new training dataset, the root-mean-squared error (RMSE) was calculated between actual price in the validation set and the predicted price generated by the model.  

The best model for the regression task, code for which is included in the Appendix below, was optimized on the validation set to obtain an RMSE of 7.99 euros. This model
strategy was then trained on the entire set of training data before utilization on the test set. This model was then executed on the originally withheld testing data, resulting in an RMSE of 11.16 euros. This test error is significantly higher than anticipated. Although the test error was likely to be greater than estimated, a test error less than 10 was expected. This result is likely due to over-fitting on the validation set in a manner that did not generalize well. The chosen model did perform better than the base OLS model, however, which resulted in a test error of 12.57.  

A perfect classifier, which is available in the Appendix, had been developed for the binary classification task using the training data. This model utilized only 64% of the total dataset as the training data. Before a classifier could be applied to the test set, appropriate modifications were implemented to make the test set amenable to k-NN classification. After missing values were replaced by the covariate means in the test set, a covariate column was added containing 1 for observations where the demand was higher than in the previous hour and 0 otherwise. All covariate columns were then normalized. Normalization of the year covariate in the test set, however, resulted in undefined values because the year covariate had a value of 2018 for all observations. The year covariate was then set to 0. After the mentioned
modifications, the classifier was successful in perfectly classifying the test set as well. Despite the model classifying validation data perfectly, its performance on an arbitrary test set had not been guaranteed, since the size of the validation set was rather small. Perfect classification of the test set, therefore, establishes that the performance of the model is independent of the dataset to which it is applied.  

### Inference
For the regression output, statistical significance can be a rather narrow statement about the likelihood of observing the calculated coefficients, if the true coefficients had been  zero. This statement only holds under the assumption that the true population model is in fact linear and that the model contained all relevant covariates from this population model. Further, this assumes that all errors are normal, homoscedastic, independent, and identically distributed. Given these assumptions, very small p-values qualify the necessity of each covariate. However, a high p-value does not guarantee that the associated coefficient is
insignificant to the true population model.  

The chosen linear regression model for the inference task assumed that the outcome variable, the price of electricity at a given hour, is linearly related to all other covariates in the dataset. Such a choice allowed the interpretation of inference relations to be simpler by focusing on the direct effect of each covariate on the outcome, without the added complexity of interaction terms. The regression output obtained using the training dataset showed that 68.5% of all covariates, i.e., 37 covariates of 54, were statistically significant at the 99.9% level. Conversely, 14.8% of the covariates lacked significance, even at the 90% level. Analysis of the covariates indicates that the amount of energy generated from oil and the weather conditions in Barcelona are examples of such covariates with low significance. The observed p-values range from 2.3e-266 to 0.733. Covariates with the highest statistical significance include energy generated from hard coal, solar and biomass, the hour and month values, pressure in Bilbao, and the humidity in Valencia. High statistical significance is also observed for the direction of wind flow in Barcelona and Madrid, which is surprising as it is unclear how these could physically affect the price of energy. One possible explanation is the direction’s effect on wind turbine generation.  

An analysis of the dataset during the prediction task had revealed that many covariates were highly correlated to one another - temperature to humidity and the price of electricity to the demand. The existence of such correlations in the dataset and the absence of the corresponding interaction terms in the inference model means that it is erroneous to take statistical significance of covariates at face value, since the effect of an individual covariate on the outcome does not remain independent of other covariates. High bias observed during the bias-variance decomposition analysis of various linear regression models had also indicated that key covariates were likely missing from the dataset – an omitted variable bias. Hence, while the p-values obtained during inference testing give a measure of statistical significance, the population model might involve a missing covariate which could be collinear with an existing covariate. Additionally, the absence of higher-order terms had not been established either, therefore it is uncertain whether the population model is even linear. Even with these critiques, the covariates demonstrating exceptionally high significance (or extremely low p-values) were believed to be truly significant.  

When the same linear regression model with all covariates was applied on the test data, the fraction of covariates statistically significant at the 99.9% level reduced to 53.7%. The year covariate was dropped from the regression model entirely, since it took only one value (2018) in the test set and thus had no effect on the outcome. Further, 27.8% of all covariates had less than 90% significance. Hourly solar generation, which had one of the lowest p-values for the training dataset, was one such covariate. Other covariates with low significance included energy from onshore wind and oil, and weather conditions in Barcelona, Madrid, and Seville. The month covariate had a p-value almost equal to zero, indicating certain contribution to the population model. Other statistically significant covariates include the humidity in Bilbao and the energy generated from the consumption of pumped hydrogen storage and biomass.  

The change in the significance levels of all covariates between the train and test datasets was quite evident. One of the reasons behind the observed change could be the substantially lower number of observations in the test set compared to the training set, where the test set might not have been able to capture the population model sufficiently well. Additionally, it is possible that the relative significance of covariates changes with time, considering the time-series nature of the dataset, the training set being comprised of older data, and the test set with newer data. However, the limited number of observations in both the training and test sets precludes any conclusive determination of such a phenomenon.  

Bootstrapping was used to validate the standard errors reported by the regression output from the training set, and thus aid a better understanding of the population model. Table 1 in the Appendix shows the bootstrap estimated standard error and the regression standard error for each covariate. The bootstrap distribution for a few covariates has also been provided in the Appendix (Figs. 1, 2, 3). Analysis of the standard errors revealed that the bootstrap error was greater than the regression error for 37 of the 54 covariates, and lesser for the remainder. The difference in magnitudes of the bootstrap error and regression error was less than 1% for just 8 covariates. The bootstrap error for the pressure in Barcelona was 38.2% greater than the regression error but was 13.9% smaller for rain in Barcelona. The difference in standard error for many covariates suggests that the assumptions of a linear normal population model do not hold. For example, some covariates might have a non-linear relation to the outcome, or their errors are not normal and homoscedastic. The bootstrap distribution of the pressure in Barcelona, as shown in Fig 1 in the Appendix, is highly skewed to the right, because of which the bootstrap error is much higher than the regression standard error. The overestimation of regression error for some covariates could be the result of those covariates deviating from linear normality but having a tighter distribution around the mean than a normal distribution. One such coefficient is the rain in Barcelona, whose density plot is included in the Appendix (Fig 3).  

Since the chosen model already consisted of all covariates, an inverse approach was used: a new model was created containing only those covariates which had originally been found to be statistically significant at the 99.9% level. All covariates in this new model were also statistically significant at the 99.9% level, except for energy generated by brown coal lignite, which dropped down to the 99% level. The high statistical significance for the covariates can be explained intuitively, as those covariates which exhibited exceptional significance even in the presence of other covariates, should likely maintain their significance in the absence of less-significant covariates.

Potential problems exist in the analysis, particularly the presence of collinearity. Quite a few covariates in the dataset appeared collinear in the prediction task by examining the pair-wise graphs of the covariates. The reservations that arise from the omission of appropriate interaction terms in the model used for the inference task have been discussed earlier.

Since a 99.9% statistical significance level was selected as the criteria for identifying meaningful covariates, 0.1% of the 54 covariates could have been falsely adjudged significant during the p-value assessment. Despite 0.054 covariates being a meaningless number, applying the Bonferroni correction to the 99.9% cutoff resulted in the elimination of three more covariates from those which originally had a 99.9% significance level. As a result of the correction, the probability of falsely rejecting the null for any one of the remaining 34 covariates becomes less than 0.1%. While the application of the Bonferroni correction might  appear to be extreme, it markedly reduces the uncertainty of the remaining coefficients being 0 in the true population model. It was observed that energy generation from brown coal lignite was one of three covariates removed by the Bonferroni correction and the only covariate which dropped in significance between the chosen model and the new model.  

The chosen inference model includes all meaningful covariates from the dataset, therefore coefficient selection bias did not exist. However, post-selection inference could have manifested as a result of the deletion of observations containing missing data in the test set. For the prediction task, however, the sequential time-series data was preserved by handling missing values with mean imputation. Since the objective of inference was to identify significant covariates from the true population model, preserving the sequential dataset was not a priority as it was for prediction. Nevertheless, only 44 of the 28051 observations in the training set had missing data and these too were at random. Hence it is expected that the bias induced due to their deletion was not significant.  

Several intuitively casual relationships exist in the data, such as the kilowatt generation from coal, the hour of the day, and the price of energy in euros per megawatt hour. Both coal generation and hour of day have highly statistically significant relationships with the outcome variable, each with t-values greater than 34. Although these values are near claims of statistical certainty, their interpretations hinge on underlying assumptions about the data.

Even in the presence of strong statistical significance for any one covariate, statements of causality are difficult to qualify. Many of the covariates in this population model are interrelated, with some unquantifiable covariances. Any causal assertion would require first a refutation of the existence of confounding variables. For example, there exists a statistically significant relationship between decreasing temperatures in Seville with an increasing price of a megawatt hour. From the gas laws of physics, Gay-Lussac's Law establishes the causal relationship between temperature and pressure. Coincidentally, the pressure of Seville also happens to be statistically significant to the price of energy. These circular and confounding relationships make any valid claim of causality difficult between one covariate and the outcome.  

### Discussion  
The two prediction models developed would be used to drive resource allocation and generation decisions within electric utilities and cooperatives. The linear regression model predicting the price of electricity in euros per megawatt hour serves two purposes for industry professionals. First, these predictions would be used to execute electric power futures in the day-ahead wholesale energy market. Large investor-owned utilities sell energy on the order of several millions of megawatt hours, where marginal savings on energy purchases have a material impact on the bottom line. Second, arbitrage opportunities occasionally emerge in the market due to poor generation planning and demand prediction. These opportunities can be capitalized on if generation assets in the power pool respond quickly.  

For the binary classification model, utilities can process real-time weather and electric demand data via their SCADA infrastructure to make generation decisions for the next hour. Utilities must supply enough energy to meet or exceed demand at any given moment. During critical system peaks, natural gas generators and other emergency generation assets are energized to serve the demand. While utilities must have this capacity on reserve, it is economically prudent not to deploy more assets than needed to meet the demand at any moment. If utilities are incapable of meeting the system demand, power quality issues can occur, which were responsible for the northeast blackout of 2003. Electric utilities and cooperatives would benefit significantly from accurate predictive models.  

The inference model is important from the perspective of understanding the external factors which affect the price of energy. This knowledge would be especially useful for utilities as they prioritize resource allocation and forecast the impact of climate changes on energy prices.  

Although these models would perform well for months and possibly years, there is little reason not to refit regularly. Data availability and collection costs are typically the limiting factor in these decisions. With automatic meter reading systems becoming prevalent, granular demand data are made available to utilities real-time. Systems are constantly growing and evolving, so these models could only benefit from being retrained frequently.  

For the prediction task, mean imputation was chosen for handling missing data, therefore introducing a bias in the data. On the other hand, for the inference task, observations with missing data were deleted instead. These approaches should not be interchanged as they induce different levels of bias and were chosen specifically for the respective task. Data transformations included a change in temperature units and a change in the standard time covariate into separate year, month, day and hour covariates. An additional covariate for the classification task had to be added to the dataset. These transformations did not have a significant impact on the analysis but should be conveyed to users of this model. It also appears that the prediction model might be overfitted on the training and validation sets. Better prediction results might be obtained from a new model, using new data, which incorporates the results of the inference task.

One unfortunate issue with this dataset was the aggregated demand data. Unique weather data was available for the five largest cities in Spain, but the electric demand data
represented Spain as a whole. If these covariates were available independently for each of the five cities, more precise predictions could have been made for each city. This would have likely led to stronger correlations in the data. Data regarding the energy transacted between Spain and other countries was unavailable, and it could possibly have a marked impact on energy prices. Quantifiable data regarding the regulation of energy prices in Spain could be a critical covariate that was missing from the dataset.

If given another chance to attack this same dataset, a couple things could be done differently to improve the results. Instead of relying on weather data for the five largest cities, syncing in data for every publicly available weather station in Spain would improve the results. Although the aggregated electric demand data cannot be unaggregated, additional covariates could be introduced to give a more wholistic depiction of Spain’s weather. Secondly, instead of holding out a validation set, part one of this project would have been better served by a nested cross-validation technique that could preserve time-series data.  
