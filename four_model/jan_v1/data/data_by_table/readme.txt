csv file name are base on the table from the MIMIC IV
- cohort_subject_id_stay_id: subject_id, stay_id
- ground_truth: stay_id, starttime, endtime, next_starttime, next_endtime, re_vent_time_diff, dod, weaning_till_dod_hr, weaning_till_dod_day, hospital_expire_flag, re_vent_in_48, this_vent_failed, label
- baseline(patients, icustays, admissions, chartevents): age_now, gender, insurance, race, admission_type, first_careunit, weight_kg, height_cm, tobacco
- mimiciv_derived_ventilator_setting: peep, fio2, tidal_volume_observed, respiratory_rate_set, plateau_pressure
- mimiciv_hosp_labevents: O2_flow
- mimiciv_derived_vitalsign: heart_rate, sbp, dbp, mbp, resp_rate, spo2
- RSBI and Minute_ventilation need to be calculated by `tidal_volume_observed` from ventilator_setting and `resp_rate` from vitalsign
