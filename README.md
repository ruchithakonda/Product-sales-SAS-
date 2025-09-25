# Product-sales-SAS-
This dataset is extracted from Kaggle.The name of the dataset is product sales.This dataset contains 9939 observations and 7 variables  .The columns includes  things like agent id and other features related to financial product sales.
/*Import  the data from CSV */
proc import
  datafile="/mnt/data/Product_Sales.csv" 
  out=work.raw_sales
  dbms=csv
  replace;
  getnames=yes;
  guessingrows=32767;
run;

/* Data preparation*/
data work.sales;
  set work.raw_sales;
  /* Convert Date if character */
  if vtype(Date)='C' then do;
     _tmp=input(strip(Date),anydtdte.);
     if not missing(_tmp) then Date=_tmp;
     drop _tmp;
  end;
  format Date date9.;
  /* Derive Profit if not available */
  if missing(Profit) and not missing(Cost) then Profit=Sales-Cost;
  /*Using datepart function*/
  Year=year(Date);
  Month=month(Date);
  YearMonth=intnx('month',Date,0,'beginning');
  format YearMonth yymmn6.;
run;

/* Using Style Template */
proc template;
  define style MyDashTheme;
    parent=styles.htmlblue;
    style GraphData1 / color=cx1f77b4;
    style GraphData2 / color=cxff7f0e;
    style GraphData3 / color=cx2ca02c;
    style GraphData4 / color=cxd62728;
    style GraphData5 / color=cx9467bd;
  end;
run;

/* 4. Dashboard Macro (with filters) */
%macro dashboard(product=ALL, region=ALL, start='01JAN2000'd, end='31DEC2099'd);

  /* Building the filter */
  %let wherecl=1=1;
  %if %upcase(&product) ne ALL %then %let wherecl=&wherecl and Product="&product";
  %if %upcase(&region) ne ALL  %then %let wherecl=&wherecl and Region ="&region";

  /* filtered data*/
  proc sql;
    create table work.filtered as
    select *, intnx('month',Date,0,'beginning') as YearMonth format=yymmn6.
    from work.sales
    where &wherecl and Date between &start and &end;
  quit;

  /* Totals */
  proc sql noprint;
    select put(sum(Sales),comma15.2) into :tot_sales trimmed from work.filtered;
    select put(sum(Profit),comma15.2) into :tot_profit trimmed from work.filtered;
  quit;

  /* Monthly aggregation */
  proc sql;
    create table work.monthly as
    select YearMonth,
           sum(Sales) as Total_Sales,
           sum(Profit) as Total_Profit
    from work.filtered
    group by YearMonth
    order by YearMonth;
  quit;

  /* Growth MoM */
  data work.monthly;
    set work.monthly;
    PrevSales=lag(Total_Sales);
    if _n_=1 then Growth_MoM=.;
    else Growth_MoM=(Total_Sales-PrevSales)/PrevSales*100;
    format Growth_MoM 8.2;
  run;

  /* Moving Average */
  proc expand data=work.monthly out=work.trend method=none;
    id YearMonth;
    convert Total_Sales=MA3 / transformout=(movave 3);
  run;

  /* HTML Dashboard */
  ods html5 file="dashboard.html" style=MyDashTheme;
  title "Sales Dashboard (&product / &region)";

  /* Navigation */
  data _null_;
    file print;
    put '<div style="padding:10px;font-family:Arial,Helvetica,sans-serif;">';
    put '<a href="#overview">Overview</a> | <a href="#timeseries">Time Series</a>';
    put '</div><hr>';
  run;

  /* Overview & KPI Cards */
  data _null_;
    file print;
    put '<a id="overview"></a><h2>Overview</h2>';
    put '<div style="display:flex;gap:16px;">';
    put '<div style="padding:12px;border-radius:6px;box-shadow:0 1px 4px rgba(0,0,0,0.1);width:200px">';
    put '<div style="font-size:12px;color:#666">Total Sales</div>';
    put "<div style='font-size:24px;font-weight:700'>&tot_sales</div></div>";
    put '<div style="padding:12px;border-radius:6px;box-shadow:0 1px 4px rgba(0,0,0,0.1);width:200px">';
    put '<div style="font-size:12px;color:#666">Total Profit</div>';
    put "<div style='font-size:24px;font-weight:700'>&tot_profit</div></div>";
    put '</div><hr>';
  run;

  /* Time Series Chart */
  data _null_;
    file print;
    put '<a id="timeseries"></a><h3>Monthly Sales Trend</h3>';
  run;

  proc sgplot data=work.trend;
    series x=YearMonth y=Total_Sales / markers lineattrs=(thickness=2);
    series x=YearMonth y=MA3 / lineattrs=(pattern=shortdash color=red thickness=2)
           curvelabel="3-mo MA";
    xaxis label="Month";
    yaxis label="Sales";
  run;

  ods html5 close;

%mend dashboard;

/* 5. Run Dashboard (example) */
%dashboard(product=ALL, region=ALL, start='01JAN2023'd, end='31DEC2023'd);
