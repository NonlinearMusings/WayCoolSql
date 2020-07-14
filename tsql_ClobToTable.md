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
