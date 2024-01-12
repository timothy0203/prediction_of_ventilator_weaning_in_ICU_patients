# Optimizing Patient Outcomes in Ventilated Sepsis Patients: An Interactive Machine Learning-Based Model
## [Paper overleaf link](https://www.overleaf.com/project/6586aa9c35f15cfc0f7bb70e)
## [Slides link](https://www.canva.com/design/DAF3yyr8etc/_CKBIdHigxC4TrWdRKFPtw/edit)
## [Discussion record hackMD link](https://hackmd.io/@TimothyChang/BJ89oWTWp/https%3A%2F%2Fhackmd.io%2F%40TimothyChang%2FBJ89oWTWp%2Fhttps%253A%252F%252Fhackmd.io%252Fc%252FBJ89oWTWp%252Fedit%253Fedit)
## Abstract
In this study, we explore the impact of ventilator settings on the survival of intensive-care patients with sepsis. Using the MIMIC-IV database, our investigation focuses on the perception of doctors' decision-making processes, observing patient responses, and developing predictive models for successful weaning.

The three models: Doctor, Patient, and Events, provide a comprehensive analysis of these interactions. The Doctor Model predicts physicians' choices, the Patient Model forecasts patient reactions, and the Events Model predicts the outcomes of weaning procedures.

Our findings unraveling these dynamics holds the potential to optimize ventilator strategies, thereby enhancing patient care and overall survival.

## Code
- BigQuerySQL.md: SQL for cohort selection, feature selection, and ground truth selection.
- data_generator.ipynb: Generate n (default n = 24) rows for each stay_id
- group_data_generator.ipynb: Generate group data by hour for doctor model and patient model
- models.ipynb: Including doctor model (A model) and Patient model (B model)
- run_model.ipynb load and run model C
- ABC_iterate.ipynb combine model ABC together
- modelC.ipynb train model C
## CSV
- data_by_table: query result from BigQuery, group by table on the MIMIC IV database
- model_data: data for the Doctor model and Patient model
- split_cohort_stay_id: train, val, and test stay_id
## EDA and Visualization
- DH_tableone.Rmd: Print out the table one and the skim summary for dataset.
- DH_project_pairplot.ipynb: Print out the feature pairplot for figuring out the feature importance pair by pair.
- 4_model_SHAP.ipynb: Print out the SHAP value visulization of four models other than our event model.
