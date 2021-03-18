## Validating a LUHN number?

Implementing the algorithm from https://en.wikipedia.org/wiki/Luhn_algorithm

Using set-based logic, we can call the inlineable TVF ```dbo.tvf_IsLuhnNumber(@luhnNumber) ```
which returns an integer ```isValid``` of 1 for a valid LUHN number; otherwise it returns 0.

``` sql
create function dbo.tvf_IsLuhnNumber( @luhnNumber varchar(16) )
returns table
as
return  select  isValid =   cast(   case
                                    when (10 - (sum(luhn.positionalValue)  % 10)) = try_cast(right(@luhnNumber, 1) as int) then 1
                                    else 0
                                    end
                                as int)
        from
        (   -- double the value of even numbered positions (summing the digits of the product)
            select  positionalValue =   cast(   case
                                                when sp.numeral = 0 then sp.numeral
                                                when (row_number() over (order by (select 1))) % 2 = 0 
                                                then floor(log10((sp.numeral * 2))) + (sp.numeral * 2) % 10
                                                else sp.numeral
                                                end
                                            as tinyint)
            from
            (   -- null out non-numerics
                select  numeral = cast( case 
                                        when substring(@luhnNumber, i.[position], 1) like '[0-9]' 
                                        then substring(@luhnNumber, i.[position], 1) 
                                        else null 
                                        end 
                                    as tinyint)
                from 
                (   -- iterator
                    select  [position] = row_number() over (order by (select 1))
                    from 
                        ( 
                            values  (1),(1),(1),(1)
                                ,   (1),(1),(1),(1)
                                ,   (1),(1),(1),(1)
                                ,   (1),(1),(1),(1)
                        ) as sixteen(ones)
                ) as i
                where   i.[position] between 1 and (len(@luhnNumber) -1)  -- exclude the right most digit, which is the checkdigit
            ) as sp  -- string position
            where   sp.numeral is not null
        ) as luhn;
```
Example:

```
select  n.luhnNumber
    ,   isValid = (select isValid from dbo.tvf_IsLuhnNumber(n.luhnNumber))
from    dbo.checksumNumbers as n;
```
| luhnNumber | isValid |
|-----------|---------|
|79927398713|1
|79927398714|0
|NULL|0
|m1x3dup|0
|5531006517734651|1

Way Cool, huh?
