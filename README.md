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
## four_model
Some experiments for Decision Tree, Random Forest, SVM, XGBoost.
 - run_model_output_inputdata_csv.ipynb: generate 0109tryinputmodelC.csv file which contains total data set (c1, c2, and c3). I add some code from run_model.ipynb provided by Han-Jay Shu. 
 - run_model_output_nnroccurve_csv.ipynb: generate nn_for_roc.csv file to print ROC curve for NN model. I add some code from run_model.ipynb provided by Han-Jay Shu. 
 - c1_c2.ipynb: generate 0108tryc3_stayid.csv file which contains only c1 set and c2 set, without c3 set. And generate Tableone of before_weaning_hr = 0 and before_weaning_hr = 23. 
 - 4_model_using_C_input_with_0108newdata.ipynb: train four models (Decision Tree, Random Forest, SVM, XGBoost) with the total data set containing c1, c2, and c3. And can print accuracy, feature importance for the four models, and ROC curve for the five models. 
 - 4_model_using_C_input_only_contain_c1c2.ipynb: train four models (Decision Tree, Random Forest, SVM, XGBoost) with the total data set containing c1, c2, without c3. And can print accuracy, feature importance, and ROC curve for the four models. 
 - ground_truth: containing the ground truth for each stay_id

