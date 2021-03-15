# CLOBbering delimited files with T-SQL

Looking for a set-based ELT approach for processing data with T-SQL? Well, then you came to the right place!

The following is a four step process for ingesting CSV data files into SQL Server without using a format file:
1. Open the file using OPENROWSET
2. Create a row per record using STRING_SPLIT
3. Convert each row's data payload to a JSON array
4. Cast the JSON fields to their typed SQL Server columns
   
To ingest the file simply amend the code to become either an INSERT/SELECT or add an INTO clause.

NOTE:

The code operates on a few assumptions, not that they cannot be overcome, just that they are not accounted for in the code itself,
but will quickly demand your attention if you try this with a less than compliant file: 
* The input file, SALES001.CSV, is
  * a character file
  * CSV delimited
  * has a single header row
* We want to assign row numbers so that they correspond with their line number in the source file (data lineage)
* The source data doesn't contain JSON special characters other than possibly double quoted strings, per CSV standards
* The source data doesn't contain nested commas (i.e. commas within strings)
* The source data is clean enough to directly cast to a SQL Server data type without error handling

OBSERVATIONS:
* The statement's overall speed is signifcantly hampered by the CE's inablity to correctly estimate the number of rows being processed.
* The speed of OPENJSON decreases with each column added.
* On my laptop, it's consistently taking about 6-seconds to process a 13.6MB file, for a processing throughput of ~2MB/sec.
* I'm processing an entire file with a single T-SQL statement!

THOUGHTS:
* Clearly 1GB per 8.5 minutes isn't that impressive, but think of this T-SQL as a technology demonstrator -- it shows what's possible!
  The choke points are in STRING_SPLIT and OPENJSON. Not that they are in and of themselves necessarily slow, but rather that's where the work IS happening.
  If you do a Show Plan you'll see that an underlying issue is the CE's inability to correctly forecast row counts. Thus, for larger files it may be prudent 
  to split the statements up so that the file is ingested into an intermediate table as JSON and then a second statement that parses the JSON to a final SQL table
  (or maybe even better: leverages SQL Server's Computed Columns capabilities to parse the JSON data to columns in the table that can be indexed and be done! 
  See [Index JSON data](https://docs.microsoft.com/en-us/sql/relational-databases/json/index-json-data?view=sql-server-ver15) for details.)

``` sql
select  rd.rowNum
    ,   tc.*
from    openrowset( bulk '/tmp/sales001.csv', single_clob ) as clob
cross   apply
    (
        select  rowNum  = row_number() over (order by (select (1))) +1              -- enum rows, accounting for filtered out header row
            ,   [value] = cast(replace(rd.[value], char(13), '') as varchar(1024))  -- remove CRs and cast to varchar holding the entire row
        from    string_split(clob.BulkColumn, char(10)) as rd -- row data           -- split at LF
        where   -- data rows only (the files data rows begin with numeric values)
                rd.value like '[0-9]%'
    ) as data
cross   apply
    (
        values  ( '{"fields":["' + replace(
                                        replace(data.value, '"', '\"')  -- escape double quotes
                                    , ',', '","')                       -- JSON-ify columns
                                 + '"]}'
                )   -- convert CSV delimited to JSON array
    ) as json(array)
cross   apply openjson(json.array,  'strict $')
    with 
    (
        productID   varchar(16) '$.fields[0]'
    ,   [date]      date        '$.fields[1]'
    ,   zip         varchar(5)  '$.fields[2]'
    ,   units       int         '$.fields[3]'
    ,   revenue     money       '$.fields[4]'
    ,   country     varchar(16) '$.fields[5]'
    ,   comments    varchar(64) '$.fields[6]'
    ) as tc;    -- typed columns
```

H/T to [David Browne](https://www.linkedin.com/in/david-browne-737806/) for suggesting using OPENJSON as opposed to my original solution individually
extracting the fields with JSON_VALUE. The change resulted in a 33% boost in query performance.

**INPUT** *SALES001.CSV*  
ProductID,Date,Zip,Units,Revenue,Country,Comment  
726,1/15/1999,75056 CEDEX 01,1,115.45,France,Just the good ol' boys  
1909,1/15/1999,75056 CEDEX 01,2,398.90,France,never meanin' no harm  
1961,2/15/1999,75056 CEDEX 01,1,97.07,France,"beats all you never saw"  
1517,2/15/1999,75056 CEDEX 01,1,141.65,France,"been in ""trouble"" with the law"  
606,2/15/1999,75056 CEDEX 01,1,314.74,France, since the day they was born  

**OUTPUT** *(note that garbage data in Zip has been truncated due to varchar(5) being used)*
|ProductID |   Date    |  Zip  | Units | Revenue | Country | Comment                 |
|----------|-----------|-------|-------|---------|---------|-------------------------|
|726       | 1/15/1999 | 75056 | 1     | 115.45  | France  | Just the good ol' boys
1909|1/15/1999|75056|2|398.90|France|never meanin' no harm
1961|2/15/1999|75056|1|97.07|France|"beats all you never saw"
1517|2/15/1999|75056|1|141.65|France|"been in ""trouble"" with the law"
606|2/15/1999|75056|1|314.74|France|since the day they was born

Way Cool, huh?
