# Deep-Neural-Network-from-scratch

4. Review of Data Quality and Data Appropriateness

4.1 Data Quality

No additional data is used beyond what is used for the component models of MOD 11783.

Data quality testing is conducted as part of the Ongoing Monitoring (OGM) process to ensure the accuracy and reliability of model inputs and outputs. Data lineage for the Predictive Analytics Dataset (PAD) is overseen by the designated data steward leads, Ann Larson-White and Quocbao Vo. The OGM data control is designed to prevent data processing errors during the generation of reports.

The data used for MOD 11783 is sourced from Credit Lens (for PDA), PRISM Batch (for PDB), and Loan Accounting Systems within PAD. Data obtained from Credit Lens and PRISM Batch is stored in the Data Lake, which is governed by the EIS Data Lake Operations and subject to data controls, including EIS-EDGE-ADLOPE-CDRP-100. PAD data is governed by the Risk and Finance Analytics Enablement & Execution team under the EDAO-FRDO-PAD-100 Data for Modeling Procedure.

DAO-provided datasets are considered authoritative. Loan accounting system data is accessed using the SAS library reference:

```sas
libname PAD '/sasdata/crptrs1/pad/Truist/Wholesale/outputs/';
```

PRISM Batch data is available via SAS Grid Libraries using DLZ\_VIEW mappings, and does not require custom SAS code for access.

The entire OGM generation process is fully automated in SAS. Only a minimal set of period variables need to be updated manually at the start of each reporting cycle, which reduces the risk of data errors and ensures consistency. These update variables are consolidated in the initialization section of the SAS code.

Automated SAS procedures and queries are installed as internal data processing checkpoints. Additionally, usage and trending metrics such as Model Portfolio Coverage and Average PD are reviewed each quarter. These serve as data reasonableness checks. Any discrepancies are reviewed by the OGM lead and development team. Where necessary, follow-up actions are taken with model users and stakeholders.

The PRISM Batch Model Input Control Report (e.g., July 2023) compares staging table data against archived data based on median value shifts. This control ensures that the most recent data loads are valid before execution of the Risk Rating Process. All control reports are saved in the following directory:

```
/sasdata/mdevop1/Batch Control Reports Directory
```

Missing data rates are also tracked. MOD 11783 includes only one internal attribute—"Obligor Past Due Days." Values of zero or missing both indicate the absence of past due, and the missing rate is therefore always zero. The remaining attributes are external and provided by Equifax. These are outside of model developer control, and no threshold is applied.

4.2 Data Appropriateness

There is no additional data used outside the scope of MOD 11783 and its component models. The model has not been redeveloped since the last validation. During the production period, data controls described in Section 4.1 ensure that data is appropriate for its intended use.

Limitation – Data Lineage and Upstream Controls

MOD 11783 is part of a broader effort to align with the Building Our Future Data Management and Governance (DMG) workstream. The relevant business processes include Development (BPID-11630525), Managing Exceptions (BPID-16101469), and Ongoing Monitoring (BPID-11630541). The milestone DM-04, which covers the implementation of metadata, data lineage, data quality, and data controls, is scheduled for completion by January 5, 2027.

Until this milestone is achieved, visibility into model CDE data lineage and upstream control environments remains limited. As a compensating measure, appropriate data checks are performed, and inconsistencies detected downstream are followed up and resolved. The development team also coordinates with Data Stewards and the Business Data Executive as part of the DMG initiative.


