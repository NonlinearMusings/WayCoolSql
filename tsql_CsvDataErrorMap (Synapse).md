# Processing CSV files containing bad data with T-SQL

This technique allows a CSV file to be ingested via an Azure Synapse Analytics's SQL Serverless pool while:
* strongly typing each column
* preserving data lineage
* creating a 'data error map' detailing errors at both the row and column level
* using set-based logic for all processing (because that is how we roll around here!)

There are three steps to the processing:
1. Read in the file's raw data in as character columns
2. Convert (or transform) each raw column to its strongly typed target
3. Create a data error mapping of failed conversions

NOTES:

* Assuming a CSV file that can be processed with OPENROWSET() when all columns are ingested as character
* Follow Jovan Popovic's guidance to [Always use UTF-8 collations to read UTF-8 text in serverless SQL pool](https://techcommunity.microsoft.com/t5/azure-synapse-analytics/always-use-utf-8-collations-to-read-utf-8-text-in-serverless-sql/ba-p/1883633) and read the data in to columns large enough to hold the expected values using the WITH clause of OPENROWSET()
* Using the following aliases and projections to track the data along its journey
  * the (r)aw alias along with the 'raw' prefix for column names
  * the (c)onvert alias with the 'cvt' prefix for column names
  * (e)rrors alias for the data error bit map
* Using the prefixes allows us to easily discern the before and after state of each column in a single projection
  * i.e. rawUnits and cvtUnit can be rendered side-by-side for quick, visual inspection
* The column [bitMap] is an 8-byte, T-SQL, BigInt which enables us to track up to 63 columns (0 - 62 inclusive)
  * We cannot use all 64 bits as 2^64 is an overflow condition - so stopping at 2^63 keeps things simple
  * If more than 63 columns per row need to be mapped then a segmentation strategy can be adopted whereby columns 1 thru 63 are mapped to [bitMap1], columns 64 to 127 to [bitMap2], etc. as this maintains a [SARGalbe](https://blogs.msmvps.com/robfarley/2010/01/21/sargable-functions-in-sql-server) solution
* The 'secret' of the [bitMap] field is that every power of 2 is a unique number and by assiging raw columns ordinal values of my choosing I can individually track them 
  * I am deliberately mapping every column to a bit value eventhough this could be considered redundant for character columns
  * I am deliberately using zero-based counting to get 63 columns per BigInt. If you choose to use ones-based counting then you'll get 62 columns per BigInt
  * I am deliberately maintaining column ordering throughout the CROSS APPLYs to help faciliate the ability to verify the code's implementation correctness
* The (c)onvert CROSS APPLY, in this example, is responsible solely for converting column data. However, all manner of business logic could be added here to support more complex transformations. The caveat being that any failure must ultimately return a NULL value
* The (e)rrors CROSS APPLY generates the error bit map for the row based on the NULLs resulting from (c)onvert
* The code layout, while potentially verbose, is acutally very concise, methodical and repeatable when traced by column. (Try following the [zip] column for example.)

```sql
select  r.rowNum
    ,   dataErrorMap    = e.bitMap
    ,   rawProductId    = r.ProductId
    ,   rawDate         = r.[Date]
    ,   rawZip          = r.Zip
    ,   rawUnits        = r.Units
    ,   rawRevenue      = r.Revenue
    ,   rawCountry      = r.Country
    ,   c.*
from    openrowset( bulk            'https://storageinator.dfs.core.windows.net/root/raw/csv/dataErrorMap-Data-Small.csv'
                ,   format          = 'CSV'
                ,   firstrow        = 2
                ,   parser_version  = '2.0'
                )
        with
        (
            rowNum      varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   ProductId   varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   [Date]      varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   Zip         varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   Units       varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   Revenue     varchar(64) collate Latin1_General_100_BIN2_UTF8
        ,   Country     varchar(64) collate Latin1_General_100_BIN2_UTF8
        ) as r  -- raw
cross   apply
    (
        select  cvtProductId    = try_cast(r.ProductId as varchar(16))
            ,   cvtDate         = try_cast(r.[Date] as date)
            ,   cvtZip          = try_cast(r.Zip as varchar(6))
            ,   cvtUnits        = try_cast(r.Units as int)
            ,   cvtRevenue      = try_cast(r.Revenue as money)
            ,   cvtCountry      = try_cast(r.Country as varchar(16))
    )   as c    -- convert
cross   apply
    (
        select  bitMap  =   case when c.cvtProductId    is null then power(cast(2 as bigint), 0) else 0 end -- 1
                            +
                            case when c.cvtDate         is null then power(cast(2 as bigint), 1) else 0 end -- 2
                            +
                            case when c.cvtZip          is null then power(cast(2 as bigint), 2) else 0 end -- 4
                            +
                            case when c.cvtUnits        is null then power(cast(2 as bigint), 3) else 0 end -- 8
                            +
                            case when c.cvtRevenue      is null then power(cast(2 as bigint), 4) else 0 end -- 16
                            +
                            case when c.cvtCountry      is null then power(cast(2 as bigint), 5) else 0 end -- 32
    ) as e;     -- errors
```
 **OUTPUT**
 |rowNum | dataErrorMap | rawProductId | rawDate | rawZip | rawUnits | rawRevenue | rawCountry | cvtProductId | cvtDate  | cvtZip  | cvtUnits | cvtRevenue | cvtCountry |
|-------|--------------|--------------|---------|--------|----------|------------|------------|--------------|----------|---------|----------|------------|------------|
|1      |62            |1             |(null)   |(null)  |(null)    |(null)      |(null)      |1             |(null)    |(null)   |(null)    |(null)      |(null)
2|10|726|(null)|75056|a|115.45|France|726|(null)|75056|(null)|115.45|France
3|12|1909|1/15/1999|(null)|a|398.9|France|1909|1/15/1999|(null)|(null)|398.9|France
4|8|1961|2/15/1999|75056|(null)|97.07|France|1961|2/15/1999|75056|(null)|97.07|France
5|24|1517|2/15/1999|75056|a|(null)|France|1517|2/15/1999|75056|(null)|(null)|France
6|40|606|2/15/1999|75056|a|314.74|(null)|606|2/15/1999|75056|(null)|314.74|(null)
7|8|1518|2/15/1999|75056|a|141.65|France|1518|2/15/1999|75056|(null)|141.65|France
8|0|786|5/31/2002|75001|10|68.2|France|786|5/31/2002|75001|10|68.2|France

THE PAYOFF

If the query is a CETAS writing to ```lz.ImportedCSV``` we can query ```dataErrorMap``` to measure data quality and programmatically control next steps. In short, you now have column level control over your data processing.

For example:
```sql
-- process all rows with no column level errors
select  *
from    lz.ImportedCSV
where   dataErrorMap = 0;

-- process all records with good Date and Zip columns
select  *
from    lz.ImportedCSV
where   dataErrorMap & 2 = 0    -- Date
  and   dataErrorMap & 4 = 0;   -- Zip

-- get a high-level good/bad row count
select  BadRecords  = sum(case when dataErrorMap > 0 then 1 else 0 end)
    ,   GoodRecords = sum(case when dataErrorMap > 0 then 0 else 1 end)
    ,   TotalRecords= count(1)
from    lz.ImportedCSV;

-- get error counts by column
select  BadProduct = sum( case when dataErrorMap & 1 = 1 then 1 else 0 end)
    ,   BadDate    = sum( case when dataErrorMap & 2 = 2 then 1 else 0 end)
    ,   BadZip     = sum( case when dataErrorMap & 4 = 4 then 1 else 0 end)
    ,   BadUnits   = sum( case when dataErrorMap & 8 = 8 then 1 else 0 end)
    ,   BadRevenue = sum( case when dataErrorMap & 16 = 16 then 1 else 0 end)
    ,   BadCountry = sum( case when dataErrorMap & 32 = 32 then 1 else 0 end)
from    lz.ImportedCSV;
```
So, all of this is pretty cool, but we can take it up another level and make it way cool!

Let's start by adding code to the (c)onvert CROSS APPLY to make it NULL aware. (You can actually add as much business/validation logic here as you like. It's your call!)
```sql
cross   apply
    (
        select  cvtProductId    = try_cast(r.ProductId as varchar(16))
            ,   cvtDate         = try_cast(r.[Date] as date)
            ,   DateIsNull      = case when r.[Date] is null then 1 else 0 end
            ,   cvtZip          = try_cast(r.Zip as varchar(6))
            ,   ZipIsNull       = case when r.Zip is null then 1 else 0 end
            ,   cvtUnits        = try_cast(r.Units as int)
            ,   UnitsNull       = case when r.Units is null then 1 else 0 end
            ,   cvtRevenue      = try_cast(r.Revenue as money)
            ,   RevenueIsNull   = case when r.Revenue is null then 1 else 0 end
            ,   cvtCountry      = try_cast(r.Country as varchar(16))
            ,   CountryIsNull   = case when r.Country is null then 1 else 0 end
    )   as c    -- convert
```
We now have the ability to distingush between columns that arrived as NULLs versus columns that failed conversion and were cast to NULL. So, if we have columns where 'natural' NULL values should not be considered an error condition we can modify the (e)rrors CROSS APPLY to account for that
```sql
cross   apply
    (
        select  bitMap  =   case when c.cvtProductId    is null then power(cast(2 as bigint), 0) else 0 end -- 1
                            +
                            case when c.cvtDate         is null then power(cast(2 as bigint), 1) else 0 end -- 2
                            +
                            case when c.cvtZip          is null then power(cast(2 as bigint), 2) else 0 end -- 4
                            +
                            case when c.cvtUnits        is null then power(cast(2 as bigint), 3) else 0 end -- 8
                            +
                            case when c.cvtRevenue      is null then power(cast(2 as bigint), 4) else 0 end -- 16
                            +
                            case
                            when c.CountryIsNull = 1 then 0 -- Country IS NULL is not an error condition
                            when c.cvtCountry is null then power(cast(2 as bigint), 5) 
                            else 0 
                            end -- 32

    ) as e;     -- errors
```
In summary, we have created a set-based data processing pipeline with the ability to track data errors by row and column!

Way Cool, huh?
