# Optimizing an ML Pipeline in Azure

## Overview
This project is part of the Udacity Azure ML Nanodegree.
In this project, we build and optimize an Azure ML pipeline using the Python SDK and a provided Scikit-learn model.
This model is then compared to an Azure AutoML run.

## Useful Resources
- [ScriptRunConfig Class](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.scriptrunconfig?view=azure-ml-py)
- [Configure and submit training runs](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-set-up-training-targets)
- [HyperDriveConfig Class](https://docs.microsoft.com/en-us/python/api/azureml-train-core/azureml.train.hyperdrive.hyperdriveconfig?view=azure-ml-py)
- [How to tune hyperparamters](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-tune-hyperparameters)


## Summary
**In 1-2 sentences, explain the problem statement: e.g "This dataset contains data about... we seek to predict..."**

The dataset contains customer and campaign related information on direct marketing campaigns of a Portuguese financial institution. We seek to predict whether the customer will subscribe a term deposit or not (variable `y`), making this a classification task.

**In 1-2 sentences, explain the solution: e.g. "The best performing model was a ..."**

The best performing model was a VotingEnsemble that resulted from applying an AutoML job, creating a pipeline for preprocessing, training and testing 8 different models which ended up being ensembled by giving each a weight in the final predicted class probabilities.

This ensemble is composed of 4 XGBoost Classifiers, 1 LightGBM Classifier, 1 Logistic Regression Classifier, 1 Support Vector Machine with Stochastic Gradient Descent training and 1 Random Forest Classifier.

## Scikit-learn Pipeline
**Explain the pipeline architecture, including data, hyperparameter tuning, and classification algorithm.**
* We have a training script `train.py` that is responsible for the following:
  * Creating a `TabularDatasetFactory` from an online CSV file.
  * Data cleansing and one-hot encoding categorical data
  * Splitting the cleansed data in train and test subsets. We do this in order to score the model trained with the train set in a new set whose examples the model has not been trained on.
  * Training a Logistic Regression Classifier from the `scikit-learn` library. This model is used to explain the relationship between the target variable and the features contained in the dataset.
  * Saving the model as a pickle.
* This script takes two arguments, i.e. the two hyperparameters that can be tuned in this experiment.
  * The first is the `C` hyperparameter, which is the inverse of regularization strength.
  * The second is the `max_iter` hyperparameter, which is the maximum number of iterations to converge.
* In the notebook, we specified a hyperdrive job that will execute our `train.py` with different configurations automatically.
  * First we specified a parameter sampler so that the hyperdrive job knows which parameter range to explore.
  * Then we specified a policy for early stopping
  * Last, we specified the hyperdrive configuration and ran the experiment.

**What are the benefits of the parameter sampler you chose?**

It lets us randomly explore a hyperparameter space, saving a lot of time compared to an exhaustive, brute-force search.
* In our case we let the `C` parameter take a uniform distribution bounded to two values that we had previously narrowed from a past experiment.
* The `max_iter` parameter can be chosen from a given set as there is not much difference in the result when choosing consecutive values, so we are ok by exploring values ranging from 50 to 200 iterations that we eyeballed.

**What are the benefits of the early stopping policy you chose?**

We chose a `BanditPolicy`, that defines an early stopping policy based on slack criteria and a frequency and delay interval for evaluation.

We used the `slack_factor` instead of the `slack_amount` parameter. The `slack_factor` is the ratio used to calculate the allowed distance from the best performing run. We chose a value of 10%, so if in each time the policy is evaluated (as defined by `evaluation_interval`), the metric falls below the slack respect the best performing model, the job is terminated. This allows us to have a hyperdrive job that does not deplete its `max_total_runs` if the model is not improving with each iteration, therefore saving computing time.

## AutoML
**In 1-2 sentences, describe the model and hyperparameters generated by AutoML.**

We defined an AutoML configuration, with exit criterion `experiment_timeout_minutes` that defines the maximum time the experiment should run. We selected 30 minutes. We also chose 30 `iterations`, so that it will try 30 different pipelines throughout the experiment.

We chose the `task` to be classification, as it has been stated above. The `primary_metric` to maximize should be Accuracy. We then let our `training_data` to be `train_ds` and our `validation_data` to be `test_ds`, which where obtained exactly the same as in the sklearn/hyperdrive pipeline. We chose not to use cross-validation as it would be easier to compare pure performance of both solutions if we kept all modelling decisions as similar as possible.

AutoML generated a total of 30 pipelines, where the best perfoming one (Accuracy 91.62%) was a VotingEnsemble composed of a pool of 8 weaker-learning pipelines that together form a strong learner. The best performing non-ensemble pipeline (Accuracy 91.47%) was an XGBoost Classifier with a StandardScalerWrapper preprocessing.

The hyperparameters for the VotingEnsemble can be checked when printing `automl_run.get_output()`, and as expected, it will show the individual hyperparameters of each of the 8 pipelines that compose the model. The weights of the VotingEnsemble are the following:
```weights=[0.09090909090909091, 0.09090909090909091, 0.09090909090909091, 0.36363636363636365, 0.09090909090909091, 0.09090909090909091, 0.09090909090909091, 0.09090909090909091]```

The only weight that stands out is the one that gives more importance to the LightGBM Classifier over the rest of models.

## Pipeline comparison
**Compare the two models and their performance. What are the differences in accuracy? In architecture? If there was a difference, why do you think there was one?**

The AutoML best perfoming model (91.62% Accuracy) achieved a slight improvement with respect to the best hyperparameter tuning configuration (90.76%) of sklearn's Logistic Regression Classifier. The increase was expected as LRC is not tipically among the best ML algorithms when it comes to capturing non-linearities in the data, whereas models such as LightGBM or XGBoost are.

The architecture of both pipelines is quite different. Both models share the same data cleansing, one-hot encoding and train-test split steps, but other than that they follow different processes.
* The LRC pipeline uses a single ML model and only said model's hyperparameter space is searched. The AutoML pipeline explores combinations of different preprocessing steps, different ML algorithms, and within the same algorithm, multiple hyperparameter configurations.
* The LRC pipeline does not have a data normalization or standarization step.
* The AutoML best performing model is an ensemble of weaker learners, a possibility that has not been implemented in the LRC pipeline.

## Future work
**What are some areas of improvement for future experiments? Why might these improvements help the model?**
* As the AutoML job correctly identified, the dataset is quite inbalanced. There are 2771 positive samples out of 24712, which is a 11% positivity ratio. A suggested area of improvement would be to downsample the data so that the classes are balanced.
* We could explore a more exhaustive hyperparameter grid search, but I doubt the performance would increase significantly.
* We could apply some scaling method to the LRC pipeline and check if performance is on par with the best AutoML pipelines.
* If we were to put one of these models into production, I would definitely not select the VotingEnsemble as my champion model, but the best performing pipeline without ensembling. The marginal improvement over, for example, an XGBoost Classifier, in my opinion, does not justify losing inference speed and model explainability.
