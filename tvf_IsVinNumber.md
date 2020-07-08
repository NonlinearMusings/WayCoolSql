## Validating a Vehicle Identification Number (VIN)?

Following the guidance from https://vin.dataonesoftware.com/vin_basics_blog/bid/112040/use-vin-validation-to-improve-inventory-quality for the decoding.

Using set-based logic, we can call the inlineable TVF ```dbo.tvf_IsVinNumber(@vinNumber)``` which returns an integer ```isValid``` of 1 for a valid VIN; otherwise it returns 0.

```sql
create function dbo.tvf_IsVinNumber( @vinNumber char(17) )
returns table
as
return  with cteVIN as
        (
            select  vin.[position]
                ,   vin.[value]
            from
                (   -- each position in the VIN encoding has an assigned weight
                    values  (1, 8), ( 2, 7), ( 3, 6), ( 4, 5), ( 5, 4), ( 6, 3), ( 7, 2), ( 8, 10)
                        ,   (9, 0), (10, 9), (11, 8), (12, 7), (13, 6), (14, 5), (15, 4), (16,  3), (17, 2)
                ) as vin([position], [value])
        )     
        select  isValid =   cast(   case 
                                    when decoded.checkSum % 11 = 10 and substring(@vinNumber, 9, 1) = 'X' then 1        -- valid checksum (10 is the Roman Numeral X)
                                    when decoded.checkSum % 11 = cast(substring(@vinNumber, 9, 1) as smallint) then 1   -- valid checksum (remainder equals the checkdigit)
                                    else 0                                                                              -- invalid checksum
                                    end
                                as int)
        from
            (
                select  checkSum = sum( case 
                                        when xlt.[value] is null then isnull(try_cast(substring(@vinNumber, cteVIN.[position], 1) as smallint), 0)  -- self value, if digit
                                        else cast(xlt.[value] as smallint)                                                                          -- transliterated value
                                        end
                                        *
                                        cteVIN.[value]                                                                                              -- position's value (weight)
                                    )
                from    cteVIN
                outer   apply
                    (
                        select  mapping.[value]
                        from    
                            (   -- each letter has an assinged value, noting that I, O and Q are not used
                                values  ('A', 1),   ('B', 2),   ('C', 3),   ('D', 4),   ('E', 5),   ('F', 6),   ('G', 7),   ('H', 8)    /*  I  */
                                    ,   ('J', 1),   ('K', 2),   ('L', 3),   ('M', 4),   ('N', 5),   /*  O  */   ('P', 7),   /*  Q  */   ('R', 9)
                                    ,   /* n/a */   ('S', 2),   ('T', 3),   ('U', 4),   ('V', 5),   ('W', 6),   ('X', 7),   ('Y', 8),   ('Z', 9)
                            )   as mapping(letter, [value])
                        where   mapping.letter = substring(@vinNumber, cteVIN.[position], 1)
                    )   as xlt  -- transliteration
            ) as decoded;
```
Example:

```
select  n.vinNumber
    ,   isValid = (select isValid from dbo.tvf_IsVinNumber(n.vinNumber))
from    dbo.checksumNumbers as n;
```
| vinNumber | isValid |
|-----------|---------|
|1FTRX12V69FA11242|1
|5GZCZ43D13S812715|1
|JF2SJADC1FZ819871|0
|12345678901234567|0
|NULL|0
abcd|0

Way cool, huh?
