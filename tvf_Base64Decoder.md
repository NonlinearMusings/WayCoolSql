## Need to decode a Base64 string?

Inspired from the work at: 
* https://base64.guru/learn/base64-algorithm/decode
* https://www.sqlservercentral.com/scripts/base64-encode-and-decode-in-t-sql
* http://dataeducation.com/bitmask-handling-part-4-left-shift-and-right-shift
* https://en.wikipedia.org/wiki/Base64

Using set-based logic, we can call the inlineable TVF ```dbo.tvf_Base64Decoder(@encodedPayload) ```
which returns the decoded, ASCII string.

``` sql
create function dbo.tvf_Base64Decoder( @encodedPayload varchar(8000) )
returns table
as
return  with cte8Rows as 
        (   -- generating 4096 rows, here are the first 8
            select  ones = 1
            from    
                (
                    values  (1),(1),(1),(1)
                        ,   (1),(1),(1),(1)
                ) as tens(ones)
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
                    setextOffset = ((row_number() over (order by (select 1))) * 4) -3
            from    cte64Rows      as sixtyfour1 
            cross   join cte64Rows as sixtyfour2
        )
        ,   cteMapping as
        (   -- Base64 mapping table
            select  Base64.[value]
                ,   Base64.[char]
            from 
                (   
                    values  ( 0, 'A'), ( 1, 'B'), ( 2, 'C'), ( 3, 'D'), ( 4, 'E'), ( 5, 'F'), ( 6, 'G'), ( 7, 'H'), ( 8, 'I'), ( 9, 'J'), (10, 'K'), (11, 'L'), (12, 'M'), (13, 'N'), (14, 'O'), (15, 'P')
                        ,   (16, 'Q'), (17, 'R'), (18, 'S'), (19, 'T'), (20, 'U'), (21, 'V'), (22, 'W'), (23, 'X'), (24, 'Y'), (25, 'Z'), (26, 'a'), (27, 'b'), (28, 'c'), (29, 'd'), (30, 'e'), (31, 'f')
                        ,   (32, 'g'), (33, 'h'), (34, 'i'), (35, 'j'), (36, 'k'), (37, 'l'), (38, 'm'), (39, 'n'), (40, 'o'), (41, 'p'), (42, 'q'), (43, 'r'), (44, 's'), (45, 't'), (46, 'u'), (47, 'v')
                        ,   (48, 'w'), (49, 'x'), (50, 'y'), (51, 'z'), (52, '0'), (53, '1'), (54, '2'), (55, '3'), (56, '4'), (57, '5'), (58, '6'), (59, '7'), (60, '8'), (61, '9'), (62, '+'), (63, '/')
                ) as Base64([value], [char])
        )
        ,   cteDecode as
        (   -- the fun stuff! (see explaination above)
            select  chars   =	concat(	char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset   , 1) collate Latin1_General_CS_AS) *  4) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset +1, 1) collate Latin1_General_CS_AS) / 16) as binary(1)) as tinyint)
                                            )
                                      , char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset +1, 1) collate Latin1_General_CS_AS) * 16) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset +2, 1) collate Latin1_General_CS_AS) /  4) as binary(1)) as tinyint)
                                            )
                                      , char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset +2, 1) collate Latin1_General_CS_AS) * 64) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.setextOffset +3, 1) collate Latin1_General_CS_AS) /  1) as binary(1)) as tinyint)
                                            )
                                      )
            from    cteSextet as s
        )
        select  decodedPayload = string_agg(chars, '')
        from    cteDecode;
 
 ```
Example:

```
select  b.encodedPayload
    ,   decodedPayload = (select decodedPayload from dbo.tvf_Base64Decoder(b.encodedPayload))
from    dbo.Base64 as b;
```
*encodedPayload*

TWFuIGlzIGRpc3Rpbmd1aXNoZWQsIG5vdCBvbmx5IGJ5IGhpcyByZWFzb24sIGJ1dCBieSB0aGlzIHNpbmd1bGFyIHBhc3Npb24gZnJvbSBvdGhlciBhbmltYWxzLCB3aGljaCBpcyBhIGx1c3Qgb2YgdGhlIG1pbmQsIHRoYXQgYnkgYSBwZXJzZXZlcmFuY2Ugb2YgZGVsaWdodCBpbiB0aGUgY29udGludWVkIGFuZCBpbmRlZmF0aWdhYmxlIGdlbmVyYXRpb24gb2Yga25vd2xlZGdlLCBleGNlZWRzIHRoZSBzaG9ydCB2ZWhlbWVuY2Ugb2YgYW55IGNhcm5hbCBwbGVhc3VyZS4=

*decodedPayload*

Man is distinguished, not only by his reason, but by this singular passion from other animals, which is a lust of the mind, that by a perseverance of delight in the continued and indefatigable generation of knowledge, exceeds the short vehemence of any carnal pleasure.

Way Cool, huh?
