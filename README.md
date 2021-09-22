# Predicting Non Performing Loans

### Background

A bank aims to predict whether a loan will be repaid to the bank. The bank is unable to accurately predict whether a client is able to pay their loan back on a continuous basis, and hence is unable to serive their clients better and prevent clients from defaulting. The bank's goals are to:

1. predict if a client is able to pay the installment in any given month
2. predict how much a client may be able to pay in any given period
3. accurately predict the risk of a client defaulting

---

### Data Description

The Berka dataset is a collection of financial information from real anonymized Czech bank transactions, account information, and loan records. The dataset consists of the following relations:
* relation account (4500 objects in the file ACCOUNT.ASC) — each record describes static characteristics of an account.
* relation client (5369 objects in the file CLIENT.ASC) — each record describes characteristics of a client.
* relation disposition (5369 objects in the file DISP.ASC) — each record relates together a client with an account. i.e., this relation describes the rights of clients to operate accounts.
* relation transaction (1056320 objects in the file TRANS.ASC) — each record describes one transaction on an account.
* relation loan (682 objects in the file LOAN.ASC) — each record describes a loan granted for a given account.
* relation demographic data (77 objects in the file DISTRICT.ASC) — each record describes demographic characteristics of a district.

---

### Problem Statement

The goal of this project is the following:

1. Predict whether or not a client will pay the loan installment for the current month
2. Predict the transaction amount a client will pay back for the current month
3. Predict the probability the client is in debt

---

### Summary of Analysis

**Assumptions on the Dataset**
The dataset used contains data up to the period of December 1998. Therefore, I built this model using data from December 1998 as the current month/target data I want to predict, and only considered data before December 1998 in my training features in order to avoid data leakage.

As there was no information in dataset on repayment structures, to simplify the problem I assumed all loans are term loans with a monthly repayment schedule, and based my analysis on the monthly transaction amounts from each account. If there are no transactions related to loan repayments from an account in a given month, I assumed the client has not paid the instalment for the month.

**Solution overview**

1. Data Gathering and Preprocessing:
For this stage, I combined the data tables and aggregated the daily transactional data to a monthly view.
I also ensured that all data related to December 1998 had been removed to avoid data leakage. I stored the loan status and transactional amounts for December 1998 separately, as these features would be used as the target variables for modelling.
Lastly, I engineered additional features including average balances and average transactional amounts, the number of times an account's balance is below a certain threshold, as well as the total payment amounts for transaction types.
2. Exploratory Data Analysis:
I used pandas profiling for a summary of my final dataset. 
To summarise my observations:
* Minimum loan size is 5K and maximum is 590K. The minimum monthly instalment due is 0.3K, and maximum is 9.9K.
* Most loans in the dataset are 5Y loans (140 loans), and least are 1Y loans (27 loans)
* The data set is highly imbalanced - if looking at whether or not a client has paid an instalment in December 1998, 97% of clients have paid an instalment and only 3% have not. Additionally, 90% of loans are running with no issue, and 10% of outstanding loans have clients that are in debt.
* 10% of loans belong to accounts that have previously had a negative balance. The maximum number of times a single account has had a negative balance is 117 times
* 98.5% of loans belong to accounts that have previously had less than 5K balance. The maximum number of times a single account has had a balance under 5K is 183 times
* Most outstanding loans are 1-2 years old. Most accounts are 3 years old.
* 51% of the outstanding loans are with female clients
* There is a positive correlation between whether or not the client paid an instalment in December 1998 and the number of times the account had a negative balance, the number of times the account had a balance less than 5K, and the duration since the last loan transaction.
* There is a negative correlation between whether or not the client paid an instalment in December 1998 and the account balance for November 1998, the previous 3 month average account balance, and the total size of the loan
* Many of the features (such as the loan instalment amounts for the last few months) are highly correlated - I may need to use methods for feature selection when modelling

3. Data Modeling
The goal of this project is to be able to predict whether an instalment for the next month will be paid: True means the instalment has not been paid, False means the instalment has been paid.
This means that:
* True Positives are loans where my model correctly predicts that the instalment has not been paid
* True Negatives are loans where my model correctly predicts that the instalment has been paid
* False Positives are loans where my model incorrectly predicts that the instalment has not been paid (but it actually has been paid)
* False Negatives are loans where my model incorrectly predicts that the instalment has been paid (but it actually has not been paid)

While ideally I would want to accurately predict all instalment payments correctly, in the case of NPLs a False Negative is a bigger liquidity risk for the bank than a False Positive since predicting instalments have been paid when they have not could lead to a shortage in short term liquidity (whereas a False Negative would mean excess cash). Therefore, I want to focus on minimiising my False Negative Rate (or maximising the recall score).

I tested both Random Forest and Logistic Regression binary classification models, utilising hyperparameter tuning as well as oversampling techniques to find the best model. The random forest classification model with hyperparameter tuning was the best performing model, with a recall score of 0.75 and an ROC AUC score of 0.97. For comparison against my baseline model, if I had assumed there was a 97% probability a client would pay the monthly loan instalment, the recall score would be 0 and the ROC AUC would be 0.5.

In order to predict the amount a client would pay in the month, I built a ridge regression model. Ridge regression is a linear regression model that uses a regularisation parameter to reduce the likelihood of overfitting. The model had an R2 score of 0.77, which means that 77% of the variation in the amount can be explained by my model.

Lastly, I used a random forest binary classification model to predict the probability that a client would default. This model performed extremely well on the train, with a recall score of 1 on the test set (i.e there were no false negatives), and only 2 false positives. This means no outstanding loans with clients currently in debt were misclassified as running smoothly.

### Conclusion and Further Steps

The actual total amount clients repaid in December 1998 is 1,724K.

If we assumed all clients paid the monthly instalment on schedule, we would have expected the total amount to be 1,786K. Therefore, the bank would have overestimated their cash position from client loans by 62K.

Instead, if the bank predicted which clients would pay the instalment on schedule using this model, we would have expected the total amount to be 1,733K. Therefore, the bank would have overestimated their cash position from client loans by 10K instead of 62K, which is a significant improvement.

Additionally, based on the feature weights of my model to assess the risk of a client defaulting, the following features are most important in determining if a client is in debt:
1. The number of times the account has had a negative balance
2. The total payment amount for sanction interest (or the amount a client has paid in late fees)
3. The number of times the client has paid sanction interest
4. The predicted instalment amount a client will pay (this prediction is based on a Ridge Linear Regression model)
5. The number of times the balance in the account is less than 5K


**Further Steps**
* In order to predict the total amount a client is able to pay over a given period, it would be interesting to look at a longer horizon, such as a prediction of how much a client will pay in a 3 month horizon. This could be more helpful in terms of predicting if a client will default, as typically at minimum a 90 day period with no payment would result in a loan being classified as non performing
* I used a threshold of 0.5 to determine if a loan instalment would be paid or not (i.e. if the model predicted that there was a 51% probability the client would pay the loan instalment, then the model would predict that the client paid the loan instalment for the month. However, for non performing loans, a bank may want to take a more conservative approach and use a higher threshold (i.e. the model should only predict that the client would pay the loan instalment if there was a 80% probability this would occur). Further analysis should be done to assess the optimal threshold for profit maximisation.
* The data set is quite small and there is a large class imbalance, which could contribute to why my models have a tendency to overfit the data. Collecting more loan data can help the model generalise better.
