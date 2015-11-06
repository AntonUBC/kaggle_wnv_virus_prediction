## My solution to [Kaggle West Nile virus prediction challenge] (https://www.kaggle.com/c/predict-west-nile-virus)

### Project Description 

Given [weather, location, and spraying data](https://www.kaggle.com/c/predict-west-nile-virus/data), this competition asked kagglers to predict when 
and where different species of mosquitoes will test positive for West Nile virus. In other words, the prediction target
was the probability of finding a WNV infected mosquito in a particular day, at a particular location, and in a particular type of mosquito.
The evaluation metric for prediction accuracy was [ROC AUC](https://en.wikipedia.org/wiki/Receiver_operating_characteristic).
The data set included 8 years: 2007 to 2014. Years 2007, 2009, 2011 and 2013 were used for training while remaining four years
were included in test data. This competition was particularly challenging for two reasons:

- it required a considerable amount of feature engineering to extract useful information from weather and spraying data
- the modelling strategy must have taken into account the time series nature of the data (e.g. potentially different
yearly patterns in WNV outbreaks) 

In addition, the data was organized in such a way that when the number of mosquitoes exceeded 50, they were split into
another record (another row in the dataset), such that the number of mosquitoes in each row was capped at 50. This
nature of data representation generated duplicate rows for each unique date, trap, species combination in the data.
Moreover, WNV could be recorded as "present" in one duplicated row, and "absent" in another which introduced a substantial variation
in the distribution of WNV even conditional on date, location and the type of mosquitoes in the trap.

### Feature Engineering

As it turned out, duplicated rows could be used to substantially improve the accuracy of predictions since the number of duplicated rows
introduced a "leakage" in data from which the rows with large number of mosquitoes could be detected. Since duplicated rows
were observed in both train and test data, as opposed to the number of mosquitoes, one could easily generate a very useful 
feature: number of duplicated rows for a particular date-trap-species combination. Furthermore, it was tempting to use the 
number of mosquitoes directly in the model. To address the absence of this information in test data, I used the number of 
mosquitoes per date-trap-species combination in train data to generate meta-features by constructing out-of-fold predictions
for log(number of mosquitoes).
In addition to these features, I used current and lagged weather conditions such as max and min temperature, [wet bulb](https://en.wikipedia.org/wiki/Wet-bulb_temperature),
[dew point](https://en.wikipedia.org/wiki/Dew_point), and etc. The lags for each weather variable were chosen at 1, 3, 5, 8, and 12 days.
I struggled trying to extract something useful from spraying data. Since it did not cover all training years (only 2011 and 2013),
it was unclear how to implement it in the model. In the end, (as did many other kagglers) I ended up not using it at all.

### Model Description

In this competition I constructed a three-level learning architecture to obtain predicted probabilities:

- I used out-of-fold predictions of 9 different regression models to estimate log(number of mosquitoes) for each date-trap-species
combination and added them as meta-features to the data, creating a stacked dataset

- I trained Gradient Boosting and Neural Network classifiers on stacked data to obtain two sets of second level predictions

- I combined predictions of second stage models using a geometric mean

I tuned hyper-parameters of 1st and 2d stage models using nice [hyperopt package] (https://github.com/hyperopt/hyperopt) 
splitting training data by year for cross-validation.

Thanks to this competition, I discovered two great machine learning libraries: [XGBoost](https://github.com/dmlc/xgboost) and
[Lasagne](https://github.com/Lasagne/Lasagne). The first one is a gradient boosting library with parallelized tree building
and numerous regularization capabilities (it also includes a linear booster). In my experience, it outperforms sklearn
[Gradient Tree Boosting] (http://scikit-learn.org/stable/modules/ensemble.html#gradient-tree-boosting) in every single
dimension including speed and accuracy. I think that literally EVERY ONE of recent Kaggle winners used this library alone or
in ensemble with other models.
Lasagne is a great deep learning library which is built on top of [Theano](https://github.com/Theano/Theano) and therefore
can be configured for [GPU](http://deeplearning.net/software/theano/tutorial/using_gpu.html) which gives a considerable gain
in speed (this is especially useful since Neural Net predictions must often be aggregated with a bootstrap procedure to reduce their
variability). I want to thank [Adam Harasimowicz](https://www.kaggle.com/aharasim) whose [implementation of Lasagne](https://github.com/ahara/kaggle_otto)
I adopted in my solution.

### Post-deadline Improvements

I really wish I went further with multirow feature engineering during competition and calculated
the number of duplicated rows per species-year, trap-year, species-month, trap-month, and etc. Some of top competitors
successfully adopted a similar approach getting very high scores on the LB.
It turned out that adding these features to my model improves the LB score by about 0.017-0.18 and places somewhere between 50th and 60th positions on the LB. I implemented this modification in the script as "the post-deadline improvement".
In addition, it could have been figured out from the competition forum that year 2012 in test data had a particularly large
outbreak of WNV which was really hard to predict given the data (this could be inferred upon the Public LB feedback).
Consequently, almost all top competitors used manual tuning of predicted probabilities to improve their LB score.
I did not implement this approach in my submission (rookie mistake not reading the competition forum). Moreover, I think
that manual tuning upon the LB feedback, while suitable for Kaggle competition, cannot be used to produce a good predictive
mode in the real life (something that I want to learn on Kaggle). Nevertheless, shortly after competition had ended,
I added a simple manual tuning procedure to my solution, multiplying 2012 probabilities by 3 (if the result was greater than one, I set it to 0.99).
Applying this simple procedure to my model generates a huge increase in the LB score giving something around 0.824-0.825 and 27th-28th position on the LB. Moreover, applying this tuning to a model with added multirow features produces a score around 0.834-0.835 and a very high 17th-18th position on the LB.

Consequently, my script can generate 4 types of output: baseline model with and without manual tuning of 2012 probabilities (the latter was used for submission),
post-deadline model with and without tuning. In addition, there is a possibility to bootstrap the Neural Net predictions, since they usually have high variance.
Depending on the number of bootstrap samples, this produces 0.005-0.008 improvement in the LB score, relatively simple improvement which I did not adopt in my official submission due to lack of time.

### Instruction

- download train, test, and weather data from the [competition website](https://www.kaggle.com/c/predict-west-nile-virus/data) and put all the data
into folder ```./data``` (you may need to adjusts its path according to your location using ```/kaggle_wnv_virus_prediction//wnv_utils/paths.py```). You must also create a folder ```./submission``` in the same subfolder. This folder
will be used for saving predictions.

- run ```/kaggle_wnv_virus_prediction/ensemble/ensemble_submission.py``` to generate the file of predictions in csv format. You can submit obtained predictions [here](https://www.kaggle.com/c/predict-west-nile-virus/submissions/attach).
 Moreover, you can produce 4 different types of predictions with this script: 2 versions of data (baseline, baseline plus added features) times 2 submission versions
 (with and without manual adjustment of 2012 probabilities). In addition, you may choose to bootstrap Neural Net predictions
 setting ```bagging=True``` in the beginning of the script. Be careful to not set too large number of bootstrap repetitions providing your Theano is configured for GPU.
 
### Dependencies
- Python 3.4 (Python 2.7 would also work, just type: ```from __future__ import print_function``` in the beginning of the script)
- Pandas (any relatively recent version would work)
- Numpy (any relatively recent version would work)
- Sklearn (any relatively recent version would work)
- Lasagne 0.1.dev0
- XGBoost 0.40
- Theano 0.7.0.dev
