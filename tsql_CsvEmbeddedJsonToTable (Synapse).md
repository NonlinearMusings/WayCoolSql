# Reading CSV files with embedded JSON with T-SQL

Need to read/import CSV data with embedded JSON using Synapse SQL Serverless? Well, then you came to the right place!

The reason this code exists is because both CSV and JSON use commas as delimiters and understandably Synapse’s 
CSV parser cannot distinguish between the two when reading the file. Therefore, the trick is to hide the embedded 
JSON from Synapse while reading the file but ensure its data is available when generating the file's result set.

The following is a five-step process for accomplishing this:
1.	In effect, parse the file row-by-row as opposed to column-by-column
2.	Extract the embedded JSON from each row
3.	Convert each row’s data payload to a JSON array
4.	Reinsert the extracted JSON back into each row; nesting the JSON
5.	Cast the combined JSON fields to their typed Synapse SQL columns

NOTES:

The code operates on a few assumptions, not that they cannot be overcome, just that they are not accounted for in the code itself,
but will quickly demand your attention if you try this with a less than compliant file: 
* The input file is
  * a character file
  * CSV delimited
  * has a single header row
* The source data contains exactly one embedded JSON string per row
* All character columns, including the embedded JSON string, are double-quote delimited and properly escaped
* The source data is clean enough to directly cast to a Synapse SQL data type without error handling
* Follow Jovan Popovic's guidance to [Always use UTF-8 collations to read UTF-8 text in serverless SQL pool](https://techcommunity.microsoft.com/t5/azure-synapse-analytics/always-use-utf-8-collations-to-read-utf-8-text-in-serverless-sql/ba-p/1883633)

OBSERVATIONS:

To test the algorithm’s performance I created two roughly equivalent CSV files. Both have the same number of rows and columns but differ slightly in file size. 
One file contained embedded JSON; the other a regular character string in-lieu of the JSON string. (i.e., a vanilla CSV file.)
I created two views, each returning a strongly-typed result set (one for the CSV file with embedded JSON and another for the pure CSV file) and called them 
from SQL Server Management Studio (SSMS).
* The pure CSV file's total roundtrip processing time averaged ~20 seconds with Synapse showing a query duration of ~2 seconds for 96MiB of data processed
* The CSV with embedded JSON file's total roundtrip processing time averaged ~27 seconds with Synapse showing a query duration of ~9 seconds for 103MiB of data processed

The interesting item of note is the query duration. The processing impact from handling the embedded JSON required 450% more compute time but the actual difference (roundtrip) in processing time was only 40%, by comparison.

Being a Technologist it’s tempting to poo-poo the 40% 'processing penalty' until you consider the alternatives of:
* Cannot process the file(s) at all due to the embedded JSON
* The time required to have the CSV files recreated using a field delimiter other than a comma (but what's the fun of that?)

```sql  
select  result.*
from    openrowset( -- parse file as rows of Character Large OBjects
                    bulk 'https://<container>.dfs.core.windows.net/<fileName.csv>'
                ,   format          = 'CSV'
                ,   parser_version  = '2.0'
                ,   fieldterminator = '0x01'    -- unlikely field termination character
                ,   fieldquote      = '0x02'    -- unlikely field quote character
                ,   escapechar      = '0x5C'
                ,   firstrow        = 2         -- skip header
                ,   rowterminator   = '0x0A'    -- line feed
                )
        with
        (
            clob varchar(4000) collate Latin1_General_100_BIN2_UTF8
        )   as  [raw]
cross apply
    (   -- get the embedded JSON string
        select [string] = cast( case
                                when charindex('{', [raw].clob) = 0 then ''
                                else substring([raw].clob, charindex('{', [raw].clob), charindex('}', [raw].clob) - charindex('{', [raw].clob) +1)
                                end
                                as varchar(4000))
    ) as    embeddedjson
cross apply
    (   -- remove the embedded JSON string
        select [string] = cast( case
                                when len(embeddedjson.[string]) = 0 then [raw].clob
                                else replace([raw].clob, embeddedjson.[string], '^^^')
                                end
                                as varchar(4000))
    ) as    jsonless
cross apply
    (   -- convert the JSON-less string to JSON
        select [string] = cast('{"fields":["' + replace(replace(jsonless.[string], '"', '\"'), ',', '","') + '"]}'
                                as varchar(4000))
    ) as    jsonified
cross apply
    (   -- re-embed the JSON string while removing its outer double-quote delimiters to ensure a combined, well-formed JSON string
        select [string] = cast( case
                                when len(embeddedjson.[string]) = 0 then jsonified.[string]
                                else replace(jsonified.[string], '"\^^^\""', embeddedjson.[string])
                                end
                                as varchar(4000))
    ) as    combined
cross apply openjson(combined.[string], 'strict $')
    with
    (   -- cast JSON string into a strongly-typed SQL table
        [id]            bigint          '$.fields[0]'
    ,   track_id        bigint          '$.fields[1]'
    ,   device_id       varchar(20)     '$.fields[2]'
    ,   email           varchar(50)     '$.fields[3]'
    ,   [type]          smallint        '$.fields[4]'
    ,   time_created    time            '$.fields[5]'
    ,   [value]         nvarchar(max)   '$.fields[6]' as json
    ,   [user_id]       varchar(50)     '$.fields[7]'
    )   as result;
```
**INPUT**
```text
id,track_id,device_id,email,type,time_created,value,user_id
308,28,"ab9105a",,3,2021-01-31 00:00:00.0000000,"{\"activity\": \"Channel\", \"position\": 137, \"time_spent\": 497}",418
306,34,"ab9105a",,1,2021-01-31 00:00:00.0000000,,418
307,6,"d9f175e",,1,2021-01-31 00:00:00.0000000,,518
309,2,"6ce0bcb",,1,2021-01-31 00:00:00.0000000,,394
310,2,"d9f175e",,3,2021-01-31 00:00:00.0000000,"{\"activity\": \"Channel\", \"position\": 122, \"time_spent\": 387}",518
311,4,"6ce0bcb",,3,2021-01-31 00:00:00.0000000,"{\"activity\": \"Channel\", \"position\": 205, \"time_spent\": 190}",394
312,8,"03b847f",,1,2021-01-31 00:00:01.0000000,,824
```

**OUTPUT**
|id |track_id|device_id|email   |type|time_created|value                                              |user_id|
|---|--------|---------|--------|----|------------|---------------------------------------------------|-------|
|308|28      | ab9105a | \<null\> | 3  |      0     |{activity: Channel, position: 137, time_spent: 497}|418
|306|34|ab9105a|\<null\>|1|0|\<null\>|418
|307|6|d9f175e|\<null\>|1|0|\<null\>|518
|309|2|6ce0bcb|\<null\>|1|0|\<null\>|394
|310|2|d9f175e|\<null\>|3|0|{activity: Channel, position: 122, time_spent: 387}|518
|311|4|6ce0bcb|\<null\>|3|0|{activity: Channel, position: 205, time_spent: 190}|394
|312|8|03b847f|\<null\>|1|0|\<null\>|824

Way Cool, huh?
