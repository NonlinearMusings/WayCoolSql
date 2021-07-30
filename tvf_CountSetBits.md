## Counting the number of bits set in a BigInt?

This function was developed as a companion to the Data Error Map processing covered in [tsql_CsvDataErrorMap (Synapse)](tsql_CsvDataErrorMap%20(Synapse).md), but it is also a technology demonstrator for bit counting.

NOTES:
* BigInts are 8-bytes (64-bits) in size and can be inspected as a Binary(8) variable
* Using the lessons from [tsql_ForEach](tsql_ForEach.md) we can iterate over all 64 bits a byte at a time with SUBSTRING() and a bit at a time using Bitwise AND (&) to see which bits are set
* Each set bit is counted as 1 and their SUM() is the count of bits set -- which in the context of Data Error Map gives us the number of bad columns per row

```sql
create function dbo.tvf_CountSetBits( @dataErrorMap binary(8) )
returns table
as
return  select  [count] =   sum(    case
                                    when substring(@dataErrorMap, [byte].[index], 1) & [bit].[value] = 0
                                    then 0
                                    else 1
                                    end
                                )
        from 
            (   -- generating 8 rows, one for each byte of a bigint in binary(8) form
                select  [index] = [binary].[byte]
                from    
                    (
                        values  (1),(2),(3),(4),(5),(6),(7),(8)
                    ) as [binary]([byte])
            ) as [byte]
        outer apply
            (   -- generating 8 rows, one for every power of two value in a byte
                select  [value] = two.[value]
                from    
                    (
                        values  (1),(2),(4),(8),(16),(32),(64),(128)
                    ) as two([value])
            ) as [bit];
```

The ineffeciency of this algorithim is that all 64 bits are inspected as opposed to Brian Kernighan's wickedly effecient C language algorithm. Buy hey! This is still bit-level, set-based processing using T-SQL!

Way Cool, huh?
