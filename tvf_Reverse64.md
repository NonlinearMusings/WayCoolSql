# Reversing a BigInt

Need a way to quickly insert rows into a table while minimizing pagelatch-ex contention? This function reverses (or mirrors) a 64-bit bigint thereby distributing inserts around the table, no table partitioning required!

H/T to Rick and his [Bit reversion](https://dangerousdba.blogspot.com/2011/10/bit-reversion.html) blog entry for the idea and explaination. I've simply T-SQL-ized it. (Yes, I just made that word up.)

The challenge, as with the CRC16_CCITT algorithm, is that T-SQL doesn't have a way to do updates in-place apart from using a variable with the quirky update feature. 
After several failed attempts to be clever I realized that simple is better and broke the problem down into reversing each byte in a BigInt and reassembling them in reverse order.
This algorithm can probably be tweaked a bit more, but I'm tired of fussing with it at the moment and am opting for this implementation's clarity.

Also, [Resolve last-page insert PAGELATCH_EX contention in SQL Server](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/performance/resolve-pagelatch-ex-contention) list several other approaches to this problem.
Method #4's [Add a non-sequential value as a leading key](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/performance/resolve-pagelatch-ex-contention#method-4-add-a-non-sequential-value-as-a-leading-key)
approach is similar to this routine, except we **are** using sequential values! 

```sql
create function dbo.tvf_Reverse64( @number bigint )
returns table as
return	select	[value]	=    cast(  -- individually reverse each byte and reassemble in reverse order thereby completely reversing all 64-bits
                                    cast(((((byte.eight & 0xAA) >> 1) | ((byte.eight & 0x55) << 1) & 0xF0) >> 4) | ((((byte.eight & 0xAA) >> 1) | ((byte.eight & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.seven & 0xAA) >> 1) | ((byte.seven & 0x55) << 1) & 0xF0) >> 4) | ((((byte.seven & 0xAA) >> 1) | ((byte.seven & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.six & 0xAA) >> 1) | ((byte.six & 0x55) << 1) & 0xF0) >> 4) | ((((byte.six & 0xAA) >> 1) | ((byte.six & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.five & 0xAA) >> 1) | ((byte.five & 0x55) << 1) & 0xF0) >> 4) | ((((byte.five & 0xAA) >> 1) | ((byte.five & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.four & 0xAA) >> 1) | ((byte.four & 0x55) << 1) & 0xF0) >> 4) | ((((byte.four & 0xAA) >> 1) | ((byte.four & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.three & 0xAA) >> 1) | ((byte.three & 0x55) << 1) & 0xF0) >> 4) | ((((byte.three & 0xAA) >> 1) | ((byte.three & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.two & 0xAA) >> 1) | ((byte.two & 0x55) << 1) & 0xF0) >> 4) | ((((byte.two & 0xAA) >> 1) | ((byte.two & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                    +
                                    cast(((((byte.one & 0xAA) >> 1) | ((byte.one & 0x55) << 1) & 0xF0) >> 4) | ((((byte.one & 0xAA) >> 1) | ((byte.one & 0x55) << 1) & 0x0F) << 4) as varbinary(1))
                                as bigint)
		from
            (	-- split out each bigint byte into its own tinyint column because of the following...
                -- NOTE: In a bitwise operation, only one expression can be of either binary or varbinary data type.
                select	eight	= cast(substring(b.n, 8, 1) as tinyint)
                    ,	seven	= cast(substring(b.n, 7, 1) as tinyint)
                    ,	six     = cast(substring(b.n, 6, 1) as tinyint)
                    ,	five	= cast(substring(b.n, 5, 1) as tinyint)
                    ,	four	= cast(substring(b.n, 4, 1) as tinyint)
                    ,	three	= cast(substring(b.n, 3, 1) as tinyint)
                    ,	two     = cast(substring(b.n, 2, 1) as tinyint)
                    ,	one     = cast(substring(b.n, 1, 1) as tinyint)
                from
                    (
                        values ( cast(@number as binary(8)) )
                    )	as b(n)	-- binary number
            )	as byte;	-- byte per column
```
INPUT
```sql
select   [Sequence]  = gs.[value]
    ,    Reversed    = r.[value]
from     generate_series(1, 10) as gs
cross    apply dbo.tvf_Reverse64(gs.[value]) as r;
```

OUTPUT
|Sequence|Reversed|
|-----|--------|
1|2305843009213693952
2|1152921504606846976
3|3458764513820540928
4|-9223372036854775808
5|-6917529027641081856
6|-8070450532247928832
7|-5764607523034234880
8|4611686018427387904
9|6917529027641081856
10|5764607523034234880

As you can see, simple sequential inputs when reversed result in well distributed values that alleviate pagelatch-ex contention.

Way Cool, huh?
