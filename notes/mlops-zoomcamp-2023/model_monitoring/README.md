# Model Monitoring using evidently.ai

For machine learning models, we need to monitor not only **service health** (eg. resource usage like CPU or RAM usage) but also **model health**.

Model health means

- Model Performance(**rmse**, **accuracy score**)
- Data Quality and integritity (Since the error-based metrics like **rmse** need the ground-truth label to calculate, we need to check other metrics like **data quality** and **data integrity** that don't need ground-truth label to calculate.)
- **Data** and **Concept Drift**, the metrics that calculate the input distribution and model output in reference dataset

We can also monitor the other things like

- Model Bias / fairness
- Outliers
- Model Explainability 

## How to monitor data quality??

### Monitoring Data Quality in batch Deployment

- Some metrics can only be monitored in batch mode deployment.
- We need two datasets, **current dataset** which is the data we want to **score**(aka. predict) in batch mode and **reference dataset** which is the data (**training** set or **validation** set)  used to train the models. Then we will compare these two datasets to create the **metrics**. We can monitor the metrics like

    - Data Distribution (eg. normality)
    - Discriptive statistics: **average**, **median**, **quantile**, **min-max** for individual features
    - statistical tests to get **confidence level**

### Monitoring Data Quality in non-batch Deployment

- Non-batch deployment means deploying the model in **REST api** as web service.
- But in non-batch deployment, we can't calculate the **data drift** in a single prediction. We can only calculate these metrics when we get some amount of the data.
- In this kind of scenario, we need to pick a window function (eg. moving window with or without moving reference) and compare these windows.

<br>
<img src="img/ml_monitoring_architecture.png" alt="Model monitoring Architecture" style="vertical-align:middle;margin:0px 50px">
<br>

## Example Folder Struture

The code example can be found at [here](https://github.com/Michael95-m/mlops-zoomcamp/tree/main/Homework/Homework-5).

## Reading Order

This is the order to read.

1. Environment Setup <br>
    1.1 [Installation of requirements](./Requirements.md) <br>
    1.2 [Creation of Docker_compose](./Docker-compose.md)<br>
    1.3 [Setting Data Source in Grafana Dashboard](./Grafana_data_source.md)
2. [Baseline Monitoring](./Baseline_monitoring_example.md)
3. [Metrics calcuation in Evidently](./Evidently_metrics_calculation.md)
4. [Saving Grafana Dashboards](./Saving_grafana_dashboards.md)
5. [Debugging](./Debugging.md)

[Next](./Requirements.md)








