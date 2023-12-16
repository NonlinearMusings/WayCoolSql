## Need to decode a Base64 string?

NOTES:
- This code isn't strictly required for SQL Server as its native T-SQL based XML processing capabilities can do Base64 transformations. 
However, Azure Synapse Analytics' Dedicated SQL Pools currently does not have XML capability, but using this function you can have a fast T-SQL
solution for decoding Base64 strings.
- Azure Synapse Analytics' currently does not support table-value constructors. Therefore, to use this code in Synapse the VALUES statements
will need to be replaced with SELECT/UNION ALL constructs. Otherwise, the code works AS-IS.

Inspired from the work at: 
* https://base64.guru/learn/base64-algorithm/decode
* https://www.sqlservercentral.com/scripts/base64-encode-and-decode-in-t-sql
* http://dataeducation.com/bitmask-handling-part-4-left-shift-and-right-shift
* https://en.wikipedia.org/wiki/Base64

Using set-based logic, we can call the inlineable TVF ```dbo.tvf_Base64Decoder(@encodedPayload) ```
which returns the decoded, ASCII string.

The blockers are that Base64 is a 6-bit encoding scheme living in an 8-bit world and SQL Server does not have native bit
shift operators nor is it fond of working with arbitrarily sized data types that are not whole byte multiples. 

Challenge Accepted!

As explained in the Base64 link above, 'QUJD' at the bit level is 00010000|00010100|00001001|00000011 and needs to transformed
into 01000001|01000010|01000011, which is 'ABC'.

Before we go any further:
* Base64 strings are comprised of sextets. Four byte character groupings each containing three ASCII characters.
* The leading two bits, positions 1-2 (left to right) are discardable prefixes. Their purpose is to let 6-bit values reside in an 8-bit representation.

To get the first character we need bits 3-8 from the 1st byte and bits 3-4 from the 2nd byte.
- Discard the first two bits from byte #1 using left shift 2 (multiply by 4) to get: 01000000.
- Get only bits 3 and 4 from byte #2 using shift right 4 (dividing by 16) to get...: 00000001.
- Bitwise OR the two intermediate results together to get 01000001 (65) or ASCII 'A'.

To get the second character we need bits 5-8 from the 2nd byte and bits 3-6 from the 3rd byte.
- Shift left 4 (multiply by 16) on byte #2 to get: 01000000.
- Shift right 2 (divide by 4) on byte #3 to get..: 00000010.
- Bitwise OR the two intermediate results together to get 01000010 (66) or ASCII 'B'.

To get the third character we need bits 7-8 from the 3rd byte and bits 3-8 from the 4th byte.
- Shift left 6 (multiply by 64) on byte #3 to get: 01000000.
- Shift right 2 (divide by 1) on byte #4 to get..: 00000011.
- Bitwise OR the two intermediate results together to get 01000011 (67) or ASCII 'C'.

Lastly, use ```STRING_AGG()``` to concatenate the decoded sextets together to reveal the fully decoded ASCII string!

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
                    sextetOffset = ((row_number() over (order by (select 1))) * 4) -3
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
            select  chars   =	concat(	char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset   , 1) collate Latin1_General_CS_AS) *  4) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset +1, 1) collate Latin1_General_CS_AS) / 16) as binary(1)) as tinyint)
                                            )
                                      , char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset +1, 1) collate Latin1_General_CS_AS) * 16) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset +2, 1) collate Latin1_General_CS_AS) /  4) as binary(1)) as tinyint)
                                            )
                                      , char(   cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset +2, 1) collate Latin1_General_CS_AS) * 64) as binary(1)) as tinyint) 
                                                | 
                                                cast(cast(((select m.[value] from cteMapping as m where m.[char] = substring(@encodedPayload, s.sextetOffset +3, 1) collate Latin1_General_CS_AS) /  1) as binary(1)) as tinyint)
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
