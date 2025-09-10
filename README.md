/* Define a permanent library where you want to store outputs */
libname ogm "/sasdata/mrmg1/MOD1497/override_dedup";  

/* Copy dedup tables from WORK into your permanent library */
proc datasets lib=work nolist;
  copy in=work out=ogm memtype=data;
  select 
    OVERRIDE_2024Q2_DEDUP_ABL
    OVERRIDE_2024Q2_DEDUP_SF
    OVERRIDE_2024Q3_DEDUP_ABL
    OVERRIDE_2024Q3_DEDUP_SF
    OVERRIDE_2024Q4_DEDUP_ABL
    OVERRIDE_2024Q4_DEDUP_SF
    OVERRIDE_2025Q1_DEDUP_ABL
    OVERRIDE_2025Q1_DEDUP_SF;
quit;
