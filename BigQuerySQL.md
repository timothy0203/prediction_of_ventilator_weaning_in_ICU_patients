# SELECTED SQL ON BigQuery
## Cohort selection: Total number 2840
- age  > 20
- continuous intubation time > 48hours
- ICU Stay > 1 day
- included ventilation modes: Complete, Minimal support
- exclude the "stay_id" which has DNR / DNI
- for each subject_id in the patients we want, find min icu intime and output the corresponding subject_id, stay_id. 
- // Save it as `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` in local
```sql=
WITH sepsis_cohort_subject_id_stay_id_hadm_id_multiple AS (
  SELECT DISTINCT s.subject_id, s.stay_id, a.hadm_id
  FROM physionet-data.mimiciv_derived.sepsis3 AS s
  INNER JOIN physionet-data.mimiciv_hosp.patients AS p
  ON s.subject_id = p.subject_id
  INNER JOIN physionet-data.mimiciv_hosp.admissions AS a
  ON s.subject_id = a.subject_id
  INNER JOIN physionet-data.mimiciv_derived.ventilation AS v
  ON s.stay_id = v.stay_id
  INNER JOIN (
    SELECT subject_id, stay_id, charttime, ventilator_mode FROM `physionet-data.mimiciv_derived.ventilator_setting` AS choose_ventilator 
    WHERE choose_ventilator.ventilator_mode IN ('PRVC/AC', 'PCV+Assist', 'PCV+', 'MMV/AutoFlow', 'APRV', 'CMV/AutoFlow', 'CMV', 'PRES/AC (PCAC)', 'APV (cmv)', 'PRVC/SIMV (=aprv)', 'MMV', 'VOL/AC', 'APRV/Biphasic+ApnVol', 'APRV/Biphasic+ApnPress', '(S) CMV', 'P-CMV', 'CMV/ASSIST', 'MMV/PSV/AutoFlow', 'CMV/ASSIST/AutoFlow') -- control mode for at least 24 consecutive hours
    OR choose_ventilator.ventilator_mode IN ('CPAP/PSV+ApnVol', 'CPAP/PPS', 'PCV+/PSV', 'Apnea Ventilation', 'CPAP', 'MMV/PSV', 'SPONT', 'CPAP/PSV+ApnPres', 'Ambient', 'CPAP/PSV+Apn TCPL(time cycle pressure limit)', 'null', 'PSV/SBT', 'Standby', 'CPAP/PSV')
  -- minimal support 
  )
  AS ventilator_setting
  ON s.stay_id = ventilator_setting.stay_id
  INNER JOIN (
    SELECT subject_id, stay_id, charttime, itemid, value FROM `physionet-data.mimiciv_icu.chartevents`
    WHERE stay_id NOT IN (
      SELECT stay_id FROM`physionet-data.mimiciv_icu.chartevents` 
      AS patient_have_DNRDNI
      WHERE patient_have_DNRDNI.itemid = 223758 AND patient_have_DNRDNI.value IN 
      ('DNR / DNI','DNI (do not intubate)', 'Comfort measures only', 'DNR (do not resuscitate)' )
    )
  ) AS no_DNR_patient -- no 'DNR/DNI', no 'DNI', no 'Comfort measures only', no 'DNR'   
  ON s.stay_id = no_DNR_patient.stay_id
  INNER JOIN (
    SELECT stay_id, los FROM `physionet-data.mimiciv_icu.icustays` 
    WHERE los>1
  ) AS icustays_los
  ON s.stay_id = icustays_los.stay_id
  WHERE p.anchor_age > 20 
  AND v.ventilation_status = 'InvasiveVent'
  AND TIMESTAMP_DIFF(v.endtime, v.starttime, SECOND) >= 48 * 3600
  AND no_DNR_patient.itemid = 223758
  AND no_DNR_patient.charttime BETWEEN v.starttime AND v.endtime -- full code when ventilation
),

ranked_data AS (
  SELECT
    subject_id,
    stay_id,
    hadm_id,
    ROW_NUMBER() OVER (PARTITION BY subject_id ORDER BY stay_id) AS row_num
  FROM sepsis_cohort_subject_id_stay_id_hadm_id_multiple
),

Cohort AS (
  SELECT subject_id, stay_id
  FROM ranked_data
  WHERE row_num = 1
)

SELECT * FROM Cohort
```
## Ground Truth
- stay_id, starttime, endtime, next_starttime, next_endtime, re_vent_time_diff, dod, weaning_till_dod_hr, weaning_till_dod_day, hospital_expire_flag, re_vent_in_48, this_vent_failed, label
1. Select out all the InvasiveVent record in Cohort: 4890
- // save it as "ground_truth_meta_4890.csv" in local
```sql=
WITH VentilationWithNext AS (
  SELECT
    c.stay_id,
    v.starttime,
    v.endtime,
    v.ventilation_status,
    a.hospital_expire_flag,
    LEAD(v.endtime) OVER (PARTITION BY c.stay_id ORDER BY v.starttime) AS next_endtime,
    LEAD(v.starttime) OVER (PARTITION BY c.stay_id ORDER BY v.starttime) AS next_starttime
  FROM
    `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` AS c
  JOIN
    `physionet-data.mimiciv_derived.ventilation` AS v ON c.stay_id = v.stay_id
  JOIN
    `physionet-data.mimiciv_icu.icustays` AS icu ON icu.subject_id = c.subject_id AND icu.stay_id = c.stay_id
  JOIN
    `physionet-data.mimiciv_hosp.admissions` AS a ON a.subject_id = c.subject_id AND a.hadm_id = icu.hadm_id
  WHERE v.ventilation_status LIKE "InvasiveVent"
),
death AS (
  SELECT stay_id, dod FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
  JOIN `physionet-data.mimiciv_hosp.patients` as p ON c.subject_id = p.subject_id
)

SELECT
  VentilationWithNext.stay_id,
  starttime,
  endtime,
  next_starttime,
  next_endtime,
  TIMESTAMP_DIFF(endtime, starttime, HOUR) AS this_duration,
  TIMESTAMP_DIFF(next_endtime, next_starttime, HOUR) AS next_duration,
  TIMESTAMP_DIFF(next_starttime, endtime, HOUR) AS re_vent_time_diff,
  -- ventilation_status,
  hospital_expire_flag,
  CASE
    WHEN TIMESTAMP_DIFF(endtime, starttime, HOUR) >= 48 AND TIMESTAMP_DIFF(next_starttime, endtime, HOUR) < 48
    THEN 1
    ELSE 0
  END AS re_vent_in_48,
  CASE
    WHEN next_starttime IS NULL
    THEN 1
    ELSE 0
  END AS is_last_vent,
  CASE
    WHEN (TIMESTAMP_DIFF(endtime, starttime, HOUR) >= 48 AND TIMESTAMP_DIFF(next_starttime, endtime, HOUR) < 48) OR ((TIMESTAMP_DIFF(endtime, starttime, HOUR) >= 48 AND next_starttime IS NULL) AND hospital_expire_flag = 1) OR (next_starttime IS NULL AND hospital_expire_flag = 1)
    THEN 1
    ELSE 0
  END AS this_vent_failed,
  dod
FROM
  VentilationWithNext
JOIN death on death.stay_id = VentilationWithNext.stay_id
WHERE
  ventilation_status LIKE "InvasiveVent"
ORDER BY
  stay_id, starttime;
```
2. Use python to select out the first satisfied record (ventilation time > 48hr)
```python=
import pandas as pd

df = pd.read_csv("data\ground_truth_meta_4890.csv")

df['first_satisfied_48'] = 0

# Identify rows where this_duration >= 48hr
mask = df['this_duration'] >= 48

# Identify the first row for each stay_id where this_duration >= 48hr
first_satisfied_48_index = df[mask].groupby('stay_id').head(1).index

# Update the 'first_satisfied_48' column for the selected rows
df.loc[first_satisfied_48_index, 'first_satisfied_48'] = 1

filtered_df = df[df['first_satisfied_48'] == 1]

filtered_df['label'] = 1 - filtered_df['this_vent_failed']


# Assuming your dataframe is named filtered_df
filtered_df['endtime'] = pd.to_datetime(filtered_df['endtime'])
filtered_df['dod'] = pd.to_datetime(filtered_df['dod'])

# Calculate the time difference and convert it to hours
filtered_df['weaning_till_dod_hr'] = (filtered_df['dod'] - filtered_df['endtime']).dt.total_seconds() / 3600
filtered_df['weaning_till_dod_day'] = (filtered_df['dod'] - filtered_df['endtime']).dt.total_seconds() / (3600 * 24)
filtered_df


# Select the desired columns
selected_columns = ['stay_id', 'starttime', 'endtime', 'next_starttime', 'next_endtime', 're_vent_time_diff', 'dod', 'weaning_till_dod_hr', 'weaning_till_dod_day', 'hospital_expire_flag', 're_vent_in_48', 'this_vent_failed', 'label']

# Save the filtered DataFrame to CSV
filtered_df[selected_columns].to_csv('./data/ground_truth_11_21.csv', index=False)

```
## Features
### Baseline(9 features)
- age_now, gender, insurance, race, admission_type, first_careunit, weight_kg, height_cm, tobacco

```sql=
WITH baseline AS (
  SELECT 
  c.subject_id,
  c.stay_id,
  a.hadm_id,
  p.gender,
  a.race,
  a.admission_type,
  a.insurance,
  icu.first_careunit,
  EXTRACT(YEAR FROM icu.intime) - p.anchor_year + p.anchor_age AS age_now,
  FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
  JOIN `physionet-data.mimiciv_hosp.patients` as p ON p.subject_id = c.subject_id
  JOIN `physionet-data.mimiciv_icu.icustays` as icu ON icu.subject_id = c.subject_id AND icu.stay_id = c.stay_id
  JOIN `physionet-data.mimiciv_hosp.admissions` as a ON a.subject_id = c.subject_id AND a.hadm_id = icu.hadm_id
),
weight_height_tobacco AS (
  WITH all_weight_height_tobacco AS (
    SELECT
    c.stay_id,
    (CASE WHEN c_event.itemid = 226512 THEN c_event.value ELSE NULL END) AS weight_kg,
    (CASE WHEN c_event.itemid = 226730 THEN c_event.value ELSE NULL END) AS height_cm,
    (CASE WHEN c_event.itemid = 227687 THEN 1 ELSE 0 END) AS tobacco
    FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
    JOIN `physionet-data.mimiciv_icu.chartevents` AS c_event ON c.stay_id = c_event.stay_id
  )
  SELECT stay_id, MAX(weight_kg) as weight_kg, MAX(height_cm) as height_cm, MAX(tobacco) as tobacco
  FROM all_weight_height_tobacco
  GROUP BY stay_id
)

SELECT
subject_id,
baseline.stay_id,
hadm_id,
age_now,
gender,
insurance,
race,
admission_type,
first_careunit,
weight_kg,
height_cm,
tobacco,
FROM baseline
JOIN weight_height_tobacco ON baseline.stay_id = weight_height_tobacco.stay_id
```
### Charttime
- ventilator_setting(6 features): peep, fio2, tidal_volume_observed, respiratory_rate_set, plateau_pressure, ventilator_mode
```sql=
WITH ven_setting AS (
  SELECT
  c.stay_id,
  c.subject_id,
  charttime,
  peep,
  fio2,
  tidal_volume_observed,
  respiratory_rate_set,
  plateau_pressure,
  ventilator_mode,
  FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
  JOIN `physionet-data.mimiciv_derived.ventilator_setting` AS v ON c.stay_id = v.stay_id
)
SELECT * FROM ven_setting
```
- O2_flow
```sql=
-- WITH O2_flow AS (
--   SELECT
--   c.stay_id,
--   c.subject_id,
--   charttime,
--   (CASE WHEN l.itemid = 50821 THEN l.value ELSE NULL END) AS O2_flow,
--   FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
--   JOIN `physionet-data.mimiciv_hosp.labevents` as l ON l.subject_id = c.subject_id
-- )
-- SELECT * FROM O2_flow
-- WHERE O2_flow.O2_flow IS NOT NULL
-- ORDER BY stay_id, charttime
WITH O2_flow AS (
  SELECT
  c.stay_id,
  c.subject_id,
  charttime,
  (CASE WHEN l.itemid = 50821 THEN l.value ELSE NULL END) AS O2_flow,	-- or 50815
  FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
  JOIN `physionet-data.mimiciv_hosp.labevents` as l ON l.subject_id = c.subject_id
)
SELECT * FROM O2_flow
WHERE O2_flow.O2_flow IS NOT NULL AND O2_flow.O2_flow != "___"
ORDER BY stay_id, charttime


```
- vitalsign(6 features): heart_rate, sbp, dbp, mbp, resp_rate, spo2
```sql=
WITH vitalsign AS (
  SELECT 
  c.subject_id,
  c.stay_id,
  vital.charttime,
  vital.heart_rate,
  vital.sbp,
  vital.dbp,
  vital.mbp,
  vital.resp_rate,
  vital.spo2
  FROM `dh-project-00.sepsis_cohort_v1.sepsis_cohort_subject_id_stay_id_done` as c
  JOIN `physionet-data.mimiciv_derived.vitalsign` as vital ON vital.stay_id = c.stay_id
)
SELECT * FROM vitalsign
```
- RSBI and Minute_ventilation
	- RSBI during SBT vs. during MV
		- RSBI = Tidal volume / respiratory rate
	- Minute ventilation SBT vs. during MV 
		- Minute ventilation = Tidal volume * respiratory rate
	- These two features generate after fix into 24 rows for each patients (already have `tidal_volume_observed` and `resp_rate` from vitalsign)
