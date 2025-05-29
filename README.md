# Deep-Neural-Network-from-scratch

This is a project where I built a Deep Neural Network to perform Photo classification. 
I build the model from scratch, where I calculated the Forward and Backward Propagation, calculated the loss function and updated parameters.

ically significant difference between the current model and the hyperopt model, further substantiated the selection of the current model. This investigation offers valuable insights into model comparison techniques and the limitations of specific statistical tests, contributing to a well-informed decision-making process.



==========


4. Review of Data Quality and Data Appropriateness

4.1 Data Quality

No additional data is used beyond what is used for component models.

During production, data quality assessment is handled by:

* Data Assessment Team (DAT) within the Enterprise Data Risk Oversight group
* RMO Data and Analytics Office (DAO)

They ensure accuracy of:

* Input data (PAD)
* Model production data (extracts)

Key testing components:

* Enterprise Data Risk Oversight (EDRO) – Data Accuracy Transaction Testing by DAT
* Data Imputation Report – by model owners
* DAO Data Quality & Controls Testing – documented in MDD Section 11.7

MOD 11783-Specific Controls:

* Data lineage is managed by data steward leads Ann Larson-White and Quocbao Vo.
* Data sources include Credit Lens (PDA), PRISM Batch (PDB), and PAD.
* These sources are governed by Data Lake Operations and EIS controls, including EIS-EDGE-ADLOPE-CDRP-100.
* PAD is governed by EDAO and subject to EDAO-FRDO-PAD-100 controls.
* OGM code is fully automated in SAS. Minimal values are manually updated each cycle.
* The SAS initialization script defines all update variables at the beginning:

```sas
%let pd0 = 202007;
%let pd0m = Jul2020;
%let pd0dt = 20200723;
...
%let pd6 = 202112;
%let pd6m = Dec2021;
%let pd6dt = 20211223;
```

* Loan accounting data is sourced from DAO and accessed via libname PAD '/sasdata/crptrs1/pad/Truist/Wholesale/outputs/';
* PRISM Batch data is accessed via DLZ\_VIEW without requiring SAS code.

Example:

```sas
data result.&model._pdb_&ver;
  set DLZ_VIEW.PRMS_T_RATING_ARCHV;
  where int(bat_id/100) = &ver
    and (prod_cd in (&prod_list))
    and RT_ACTN_CD in ('UPLOADED');
  if TOP_LEVEL_OBLIGATION='Y' then TOP_LEVEL_OBLIGATION_flag=1;
  else TOP_LEVEL_OBLIGATION_flag=0;
  obg_no = input(CLNT_OBLIGOR_NUM, z16.);
run;
```

Data reasonableness is supported by:

* Usage and trending metrics (e.g., PD trend, obligor counts).
* Reports highlight discrepancies that trigger review by the OGM lead and developer.

PRISM Batch Model Input Control Report (JUL 2023):

* Compares median value changes between staging and archive tables.
* Validates readiness for batch rating process.
* Reports are stored in /sasdata/mdevop1/Batch Control Reports Directory

Model Input Monitoring – Missing Rate:

* MOD 11783 only includes one internal attribute: Obligor Past Due days.

  * Values of 0 or missing both indicate no past due.
  * Thus, the missing rate is always 0.
* Other attributes are external and provided by Equifax, beyond MD’s access.

4.2 Data Appropriateness

No additional data is used outside of component models.

There has been no model redevelopment since the last validation. Data controls in Section 4.1 ensure no data issues exist during production.

Limitation: Data Lineage and Upstream Controls

Developers are coordinating with the RMO Business Data Executive to align with the BOF-DMG workstream. Key RMO-MD processes:

* BPID-11630525 – Development
* BPID-16101469 – Managing Exceptions
* BPID-11630541 – Ongoing Monitoring

Milestone DM-04 (metadata, lineage, quality, controls) is expected to complete by 1/5/2027. Interim compensating controls include:

* Performing data checks during development
* Resolving inconsistencies identified downstream
* Collaborating with Data Stewards and Business Executives

