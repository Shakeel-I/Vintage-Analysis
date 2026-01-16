# Vintage-Analysis
Vintage Tables and Vintage Curves

/* 1. Import CSV into SAS */
proc import datafile="credit_record.csv"
    out=work.loans_raw
    dbms=csv
    replace;
    getnames=yes;
    guessingrows=1000;
run;
/* 1a. Remove rows where STATUS is X or C, and create numeric STATUS_N */
data work.loans_clean;
    set work.loans_raw;
    length STATUS_C $1;
    STATUS_C = strip(STATUS);
    /* Drop X and C completely */
    if STATUS_C in ('X','C') then delete;
    /* Convert remaining STATUS to numeric */
    STATUS_N = input(STATUS_C, best12.);
run;
/* 2. Ensure dates are SAS dates and derive vintage quarter and MOB quarter */
data work.loans_prep;
    set work.loans_clean;
    format START_DATE SNAPSHOT_MONTH date9.;
    START_DATE     = input(put(START_DATE,  yymmdd10.), yymmdd10.);
    SNAPSHOT_MONTH = input(put(SNAPSHOT_MONTH, yymmdd10.), yymmdd10.);
    /* Vintage = origination *quarter* (e.g. 2019Q1) */
    length vintage $7;
    vintage = cats(put(year(START_DATE), 4.), 'Q', put(qtr(START_DATE), 1.));
    /* Months-on-books in months */
    mob_month = intck('month', START_DATE, SNAPSHOT_MONTH, 'c');
    /* MOB in *quarters* (0..8 for 9 quarters) */
    MOB_Q = floor(mob_month / 3);
    /* Keep only first 9 quarters (0..8) */
    if MOB_Q < 0 or MOB_Q > 8 then delete;
    /* Flags for delinquency buckets based on numeric STATUS_N */
    if STATUS_N in (1,2,3,4,5) then flag_30 = 1; else flag_30 = 0;
    if STATUS_N in (2,3,4,5)   then flag_60 = 1; else flag_60 = 0;
    if STATUS_N in (3,4,5)     then flag_90 = 1; else flag_90 = 0;
    /* Portfolio size indicator */
    obs = 1;
run;
/* 3. Aggregate by vintage quarter and MOB quarter */
proc summary data=work.loans_prep nway;
    class vintage MOB_Q;
    var obs flag_30 flag_60 flag_90;
    output out=work.vintage_agg
        sum(obs)      = n_accts
        sum(flag_30)  = n_30
        sum(flag_60)  = n_60
        sum(flag_90)  = n_90;
run;
/* 4. Compute delinquency rates by vintage quarter and MOB quarter */
data work.vintage_rates;
    set work.vintage_agg;
    if n_accts > 0 then do;
        rate_30 = n_30 / n_accts;
        rate_60 = n_60 / n_accts;
        rate_90 = n_90 / n_accts;
    end;
    else do;
        rate_30 = .;
        rate_60 = .;
        rate_90 = .;
    end;
run;
/* 5. Vintage tables: rates by vintage (rows) and MOB quarter (columns) */
proc sort data=work.vintage_rates;
    by vintage MOB_Q;
run;
proc transpose data=work.vintage_rates out=work.vintage30_table prefix=MOB_Q;
    by vintage;
    id MOB_Q;
    var rate_30;
run;
proc transpose data=work.vintage_rates out=work.vintage60_table prefix=MOB_Q;
    by vintage;
    id MOB_Q;
    var rate_60;
run;
proc transpose data=work.vintage_rates out=work.vintage90_table prefix=MOB_Q;
    by vintage;
    id MOB_Q;
    var rate_90;
run;
/* 5a. PRINT vintage tables formatted to 4 decimal places */
title "Vintage Table - >30 Days Overdue";
proc print data=work.vintage30_table noobs label;
    format MOB_Q0-MOB_Q8 percent8.4;
    var vintage MOB_Q0-MOB_Q8;
run;
title "Vintage Table - >60 Days Overdue";
proc print data=work.vintage60_table noobs label;
    format MOB_Q0-MOB_Q8 percent8.4;
    var vintage MOB_Q0-MOB_Q8;
run;
title "Vintage Table - >90 Days Overdue";
proc print data=work.vintage90_table noobs label;
    format MOB_Q0-MOB_Q8 percent8.4;
    var vintage MOB_Q0-MOB_Q8;
run;
/* 6. Vintage plots in quarters */
ods graphics on;
/* >30 days overdue */
proc sgplot data=work.vintage_rates;
    series x=MOB_Q y=rate_30 / group=vintage lineattrs=(thickness=2);
    yaxis label="Cumulative Bad Rates" grid;
    xaxis label="Quarters on Books (MOB quarter)";
    title "Vintage Curve - >30 Days Overdue (Quarterly)";
run;
/* >60 days overdue */
proc sgplot data=work.vintage_rates;
    series x=MOB_Q y=rate_60 / group=vintage lineattrs=(thickness=2);
    yaxis label="Cumulative Bad Rates" grid;
    xaxis label="Quarters on Books (MOB quarter)";
    title "Vintage Curve - >60 Days Overdue (Quarterly)";
run;
/* >90 days overdue */
proc sgplot data=work.vintage_rates;
    series x=MOB_Q y=rate_90 / group=vintage lineattrs=(thickness=2);
    yaxis label="Cumulative Bad Rates" grid;
    xaxis label="Quarters on Books (MOB quarter)";
    title "Vintage Curve - >90 Days Overdue (Quarterly)";
run;
ods graphics off;
/* 7. Bad-loan subset (ever 90+ DPD) */
proc sql;
    create table work.bad_ids as
    select distinct ID
    from work.loans_prep
    where flag_90 = 1;
quit;
proc sql;
    create table work.loans_bad as
    select a.*
    from work.loans_prep as a
    inner join work.bad_ids as b
        on a.ID = b.ID;
quit;
