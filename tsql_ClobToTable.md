# CLOBbering delimited files with T-SQL

Looking for a set-based ELT approach for processing data with T-SQL? Well, then you came to the right place!

The following is a four step process for ingesting CSV data files into SQL Server without using a format file:
1. Open the file using OPENROWSET
2. Create a row per record using STRING_SPLIT
3. Convert each row's data payload to a JSON array
4. Cast the JSON fields to their typed SQL Server columns
   
To ingest the file simply amend the code to become either an INSERT/SELECT or add an INTO clause.

NOTE: The code operates on a few assumptions, not that they cannot be overcome, just that they are not accounted for in the code itself,
but will quickly demand your attention if you try this with a less than compliant file: 
* The input file, SALES001.CSV, is
  * a character file
  * CSV delimited
  * has a single header row
* We want to assign row numbers so that they correspond with their line number in the source file (data lineage)
* The source data doesn't contain quoted strings or other JSON special characters in the data
* The source data doesn't contain nested commas (i.e. commas within strings)
* The source data is clean enough to directly cast to a SQL Server data type without error handling

Observations:
* The statement's overall speed is signifcantly hampered by the CE's inablity to correctly estimate the number of rows being processed.
* The speed of OPENJSON decreases with each column added.
* On my laptop, it's consistently taking about 6-seconds to process a 13.6MB file, for a processing throughput of ~2MB/sec.
* I'm processing an entire file with a single T-SQL statement!

``` sql
select  r.rowNum
    ,   tc.*
from    openrowset( bulk '/tmp/sales001.csv', single_clob ) as clob
cross   apply
    (
        select  rowNum	= row_number() over (order by (select (1))) +1				-- enum rows, accounting for filtered out header row
		    ,   [value]	= cast(replace(rd.[value], char(13), '') as varchar(1024))	-- remove CRs and cast to varchar holding the entire row
        from    string_split(clob.BulkColumn, char(10)) as rd -- row data			-- split at LF
        where   -- data rows only
                rd.value like '[0-9]%'
    ) as data
cross   apply
    (
        values ( '{"fields":["' + replace(data.value, ',', '","') + '"]}' )            -- convert CSV delimited row a JSON array
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
    ) as tc;    -- typed columns
```

Way Cool, huh?
