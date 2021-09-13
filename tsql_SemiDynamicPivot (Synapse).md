## Need to pivot some data? (Azure Synapse Analytics)

The difference between the Synapse Analytics and SQL Server version is that FOR JSON PATH isn't supported in a distributed query. Thus, it is not possible to return the entire
result set as JSON. This necessitates logically splitting the query in two. The 'base' of the pivot table is a standard GROUP BY and the JSON magic still occurs inside
of JSON_QUERY(), but the extraction of the pivoted columns is pushed into a CROSS APPLY employing a WITH clause - where the columns are specified.  

NOTE: Documentation for this function may be found in the SQL Server version.  

```sql  
select  g.ApplicationUri
    ,   g.RelativeTimestamp
    ,   g.ReadingsFound
    ,   c.DipData
    ,   c.SpikeData
    ,   c.RandomSignedInt32
from    (
            (
                select  ApplicationUri
                    ,	RelativeTimestamp   = dateadd(millisecond, -datepart(millisecond, r.SourceTimestamp), r.SourceTimestamp)
                    ,	ReadingsFound       = count(1)
                    ,   [PivotedColumns]    = json_query( '{' + string_agg(concat('"', string_escape(r.DisplayName, 'json'), '" : ', r.[Value]), ',') + '}', '$')
                from    openrowset( bulk 'https://<storageAccount>.dfs.core.windows.net/root/raw/csv/IotData.csv'
                                ,   format = 'CSV'
                                ,   parser_version ='2.0'
                                ,   header_row = TRUE
                                )
                        with
                        (
                            SourceTimestamp datetime2
                        ,   ApplicationUri  varchar(64) collate Latin1_General_100_BIN2_UTF8
                        ,   DisplayName     varchar(64) collate Latin1_General_100_BIN2_UTF8
                        ,   [Value]         float
                        ) as r  -- raw
                group   by
                        r.ApplicationUri
                    ,   dateadd(millisecond, -datepart(millisecond, r.SourceTimestamp), r.SourceTimestamp) -- group records at the seconds level
            ) 
        ) as g  -- grouped data
cross   apply openjson(g.[PivotedColumns])
        with 
            (   DipData             float   'lax $.DipData'
            ,   SpikeData           float   'lax $.SpikeData'
            ,   RandomSignedInt32   float   'lax $.RandomSignedInt32'
            ) as c; -- column data
```  

Way Cool, Huh?
