## Need to decode a Base64 string? (Azure Synapse Analytics version)

The difference between the Synapse Analytics and SQL Server versions of this function is the use of SELECT/UNION ALL versus VALUES
to build data sets.

NOTE: Documentation for this function may be found in the SQL Server version.

```sql
create function dbo.tvf_Base64Decoder ( @encodedPayload varchar(8000) )
returns table
as
return  with cte8Rows as 
        (   -- generating 4096 rows, here are the first 8
            select  ones = 1 union all 
            select  1 union all select  1 union all select  1 union all 
            select  1 union all select  1 union all select  1 union all 
            select  1
        )
        ,   cte64Rows as 
        (   -- generating 4096 rows, here are the first 64
            select  ones = 1
            from    cte8Rows       as eight1 
            cross   join cte8Rows  as eight2
        )
        ,   cteSextet as
        (   -- a Base64 encoded string is comprised of 4-byte sextets, create an iterator of starting offsets for each sextet
            select  top ((len(@encodedPayload) / 4)) 
                    [position] = ((row_number() over (order by (select 1))) * 4) -3
            from    cte64Rows      as sixtyFour1 
            cross   join cte64Rows as sixtyFour2
        )
        ,   cteMapping as
        (   -- Base64 mapping table
            select  [index] = 0, [char] = 'A' union all     
            select  1, 'B'  union all  select  2, 'C'  union all  select  3, 'D'  union all  select  4, 'E'  union all
            select  5, 'F'  union all  select  6, 'G'  union all  select  7, 'H'  union all  select  8, 'I'  union all  
            select  9, 'J'  union all  select 10, 'K'  union all  select 11, 'L'  union all  select 12, 'M'  union all  
            select 13, 'N'  union all  select 14, 'O'  union all  select 15, 'P'  union all  select 16, 'Q'  union all  
            select 17, 'R'  union all  select 18, 'S'  union all  select 19, 'T'  union all  select 20, 'U'  union all
            select 21, 'V'  union all  select 22, 'W'  union all  select 23, 'X'  union all  select 24, 'Y'  union all  
            select 25, 'Z'  union all  select 26, 'a'  union all  select 27, 'b'  union all  select 28, 'c'  union all  
            select 29, 'd'  union all  select 30, 'e'  union all  select 31, 'f'  union all  select 32, 'g'  union all  
            select 33, 'h'  union all  select 34, 'i'  union all  select 35, 'j'  union all  select 36, 'k'  union all  
            select 37, 'l'  union all  select 38, 'm'  union all  select 39, 'n'  union all  select 40, 'o'  union all  
            select 41, 'p'  union all  select 42, 'q'  union all  select 43, 'r'  union all  select 44, 's'  union all  
            select 45, 't'  union all  select 46, 'u'  union all  select 47, 'v'  union all  select 48, 'w'  union all  
            select 49, 'x'  union all  select 50, 'y'  union all  select 51, 'z'  union all  select 52, '0'  union all  
            select 53, '1'  union all  select 54, '2'  union all  select 55, '3'  union all  select 56, '4'  union all  
            select 57, '5'  union all  select 58, '6'  union all  select 59, '7'  union all  select 60, '8'  union all  
            select 61, '9'  union all  select 62, '+'  union all  select 63, '/'  
        )
        ,   cteDecode as
        (   -- the fun stuff!
            select  chars   =	concat(	char(   cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position]   , 1) collate Latin1_General_CS_AS) *  4) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position] +1, 1) collate Latin1_General_CS_AS) / 16) as binary(1)) as tinyint)
                                            )
                                    ,   char(   cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position] +1, 1) collate Latin1_General_CS_AS) * 16) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position] +2, 1) collate Latin1_General_CS_AS) /  4) as binary(1)) as tinyint)
                                            )
                                    ,   char(   cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position] +2, 1) collate Latin1_General_CS_AS) * 64) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[index] from cteMapping as m where m.[char] = substring(@encodedPayload, s.[position] +3, 1) collate Latin1_General_CS_AS) /  1) as binary(1)) as tinyint)
                                            )
                                    )
            from    cteSextet as s
        )
        select  decodedPayload = string_agg(chars, '')
        from    cteDecode;
