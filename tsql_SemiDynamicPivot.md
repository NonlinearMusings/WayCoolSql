## Need to pivot some data?

I've never been a fan of T-SQL's PIVOT() command. I just don't like the syntax. So when a collegue needed help pivoting IoT data from a narrow, append-only row format into
a wider, more analytical and ML friendly one I couldn't resist the opportunity to try to build a better mousetrap!

Inspired from the work at:
- https://jingyangli.wordpress.com/2020/02/28/pivot-a-table-with-json-in-sql-server2017-or-2019/
- https://stackoverflow.com/questions/49574006/sql-server-query-columns-to-json-object-with-group-by

INPUT: ```dbo.IotData```   
|SourceTimestamp|NodeId|ApplicationUri|DisplayName|Value|
|---|---|---|---|---|
|2021-07-30 21:45:53.0000000|http://localhost/Opc/OpcPlc/#s=DipData|urn:OpcPlc:opcserver1|DipData|99.80267284|
|2021-07-30 21:45:53.0000000|http://localhost/Opc/OpcPlc/#s=SpikeData|urn:OpcPlc:opcserver1|SpikeData|99.80267284|
|2021-07-30 21:45:52.0000000|http://localhost/Opc/OpcPlc/#s=DipData|urn:OpcPlc:opcserver3|DipData|-84.43279255|
|2021-07-30 21:45:52.0000000|http://localhost/Opc/OpcPlc/#s=SpikeData|urn:OpcPlc:opcserver3|SpikeData|-84.43279255|
|2021-07-30 21:45:52.0000000|http://localhost/Opc/OpcPlc/#s=SpikeData|urn:OpcPlc:opcserver2|SpikeData|6.43249E-14|
|2021-07-30 21:45:52.0000000|http://localhost/Opc/OpcPlc/#s=DipData|urn:OpcPlc:opcserver2|DipData|6.43249E-14|
|2021-07-30 21:45:48.0000000|http://localhost/Opc/OpcPlc/#s=DipData|urn:OpcPlc:opcserver1|DipData|99.80267284|
|2021-07-30 21:45:48.0000000|http://localhost/Opc/OpcPlc/#s=SpikeData|urn:OpcPlc:opcserver1|SpikeData|99.80267284|
|2021-07-30 21:45:47.0000000|http://localhost/Opc/OpcPlc/#s=RandomSignedInt32|urn:OpcPlc:opcserver1|RandomSignedInt32|1588001084|
|2021-07-30 21:45:47.0000000|http://localhost/Opc/OpcPlc/#s=DipData|urn:OpcPlc:opcserver3|DipData|-90.48270525|
|2021-07-30 21:45:47.0000000|http://localhost/Opc/OpcPlc/#s=SpikeData|urn:OpcPlc:opcserver3|SpikeData|-90.48270525|
|2021-07-30 21:45:47.0000000|http://localhost/Opc/OpcPlc/#s=RandomSignedInt32|urn:OpcPlc:opcserver3|RandomSignedInt32|-1198295099|

We see that the ```DisplayName``` column contains the following endpoint names: *DipData, SpikeData* and *RandomSignedInt32*.  
These are to be used as column names in our pivoted table. Looking at the ```SourceTimestamp``` column we can deduce that ```ApplicationUri``` sends 
readings every second, when it transmits. Thus, we want to group individual endpoint readings by both their ```ApplicationUri``` and ```SourceTimestamp``` 
time truncated to the nearest second.

The pivot magic starts inside the ```json_query``` command where ```DisplayName``` and ```[Value]``` are turned into JSON key/value pairs by leveraging ```string_agg()```'s
ability to concatenate row data. To ensure we're generating JSON compliant data we use ```concat()``` to inject colons and commas as required. Lastly, applying the ```for json path```
clause encodes the entire resultset in JSON allowing us to use the ```with``` clause to extract the columns we are interested in -- which is more performant than using individual
```json_value()``` functions per column.

```sql
select  r.ApplicationUri
    ,   r.RelativeTimestamp
    ,   r.ReadingsFound
    ,   r.DipData
    ,   r.SpikeData
    ,   r.RandomSignedInt32
from    openjson(
                    (
                        select  ApplicationUri
                            ,	RelativeTimestamp   = dateadd(millisecond, -datepart(millisecond, SourceTimestamp), SourceTimestamp)
                            ,	ReadingsFound       = count(1)
                            ,   [Values]            = json_query('{' + string_agg(concat('"', string_escape(DisplayName, 'json'), '":', [Value]), ',') + '}', '$')
                        from    dbo.IotData
                        group   by 
                                ApplicationUri
                            ,	dateadd(millisecond, -datepart(millisecond, SourceTimestamp), SourceTimestamp) -- group records at the seconds level
                        for json path
                    )
                ) with 
                    (   ApplicationUri      varchar(128)    'strict $.ApplicationUri'
                    ,   RelativeTimestamp   datetime2       'strict $.RelativeTimestamp'
                    ,   ReadingsFound       int             'strict $.ReadingsFound'
                    ,   DipData             float           'lax $.Values.DipData'
                    ,   SpikeData           float           'lax $.Values.SpikeData'
                    ,   RandomSignedInt32   float           'lax $.Values.RandomSignedInt32'
                    ) as r;      -- readings
```  

OUTPUT:  
|ApplicationUri|RelativeTimestamp|ReadingsFound|DipData|SpikeData|RandomSignedInt32|
|---|---|---|---|---|---|
|urn:OpcPlc:opcserver1|2021-07-30 21:45:47.0000000|1|NULL|NULL|1588000000|
|urn:OpcPlc:opcserver2|2021-07-30 21:45:47.0000000|3|6.43249E-14|6.43249E-14|680112000|
|urn:OpcPlc:opcserver3|2021-07-30 21:45:47.0000000|3|-90.4827|-90.4827|-1198300000|
|urn:OpcPlc:opcserver1|2021-07-30 21:45:48.0000000|2|99.8027|99.8027|NULL|
|urn:OpcPlc:opcserver2|2021-07-30 21:45:52.0000000|2|6.43249E-14|6.43249E-14|NULL|
|urn:OpcPlc:opcserver3|2021-07-30 21:45:52.0000000|2|-84.4328|-84.4328|NULL|
|urn:OpcPlc:opcserver1|2021-07-30 21:45:53.0000000|2|99.8027|99.8027|NULL|

What I really like about this query pattern is that it is disk I/O efficient, especially if the ```GROUP BY``` columns use a clustered index, and the core of the query itself 
does not need to be changed if new endpoint names start showing up in ```DisplayName```. (That's the *dynamic* part.)

In fact, it is the use of the ```with``` clause that gives us final editing control over the output!  
(That's the *semi* part as minimal edits must be made to account for structural changes to pivoted table itself.)

For example:  
- Don't want to see *DipData*? No problem, remove it from the ```with``` clause.  
- Want to see *MyNewEndPoint*? No problem, add it to the ```with``` clause.  

Done!  

BONUS: Data not matching the ```with``` clause's definitions is ignored without error!  

Way Cool, Huh?
