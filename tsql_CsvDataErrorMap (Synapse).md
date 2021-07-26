# Processing CSV files containing bad data with T-SQL

This technique allows a CSV file to be ingested via an Azure Synapse Analytics's SQL Serverless pool while:
* strongly typing each column
* preserving data lineage
* creating a 'data error map' detailing errors at both the row and column level
* using set-based logic for all processing (because that is how we roll around here!)

There are three steps to the processing:
1. Read in the file's raw data in as character columns
2. Cast each raw column to its strongly typed target
3. Create a data error mapping of the failed casts

NOTES:

* Assuming a CSV file that can be processed with OPENROWSET() if all the columns are ingested as character
* Follow Jovan Popovic's guidance to [Always use UTF-8 collations to read UTF-8 text in serverless SQL pool](https://techcommunity.microsoft.com/t5/azure-synapse-analytics/always-use-utf-8-collations-to-read-utf-8-text-in-serverless-sql/ba-p/1883633) and read the data in to columns large enough to hold the expected values using the WITH clause of OPENROWSET()
* Using the following aliases and projections to track the data along its journey
  * the (r)aw alias along with the 'raw' prefix for column names
  * the (c)onvert alias with the 'cvt' prefix for column names
  * (e)rrors alias for the data error bit map
* Using the prefixes allows us to easily discern the before and after state of each column in a single projection
  * i.e. rawUnits and cvtUnit can be rendered side-by-side for quick, visual inspection
* The column [bitMap] is a T-SQL BigInt allowing us to track up to 63 columns
  * We cannot map 64 columns to a BigInt as 2^64 is an overflow condition
  * If more than 63 columns per row need to be mapped/tracked then a segmentation strategy should be adopted whereby columns 1 thru 63 are mapped to bitMap1, column 64 to 127 to bitMap2, etc. as this maintains a [SARGalbe](https://blogs.msmvps.com/robfarley/2010/01/21/sargable-functions-in-sql-server) solution
* The 'secret' of the [bitMap] field is that every power of 2 is a unique number and by assiging raw columns ordinal values of my choosing I can individually track them 
  * I am deliberately mapping every column to a bit value eventhough this could be considered redundant for character columns
  * I am deliberately maintaining column ordering throughout the CROSS APPLYs to help faciliate the ability to verify the code's implementation correctness
* The (c)onvert CROSS APPLY, in this example, is responsible solely for converting column data. However, all manner of business logic could be added here to support more complex transformations. The caveat being that any failure must ultimately return a NULL value
* The (e)rrors CROSS APPLY generates the error bit map for the row based on the NULLs resulting from (c)onvert
* The code layout, while potentially verbose, is acutally very methodical and repeatable when traced by column. (Try following the [zip] column for example.)

```sql
select  r.rowNum
    ,   rawProductId    = r.ProductId
    ,   rawDate         = r.[Date]
    ,   rawZip          = r.Zip
    ,   rawUnits        = r.Units
    ,   rawRevenue      = r.Revenue
    ,   rawCountry      = r.Country
    ,   c.*
    ,   dataErrorMap    = e.bitMap
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
            ,   cvtZip          = case when r.Zip like '%z' then cast(null as varchar(5)) else try_cast(r.Zip as varchar(5)) end
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
