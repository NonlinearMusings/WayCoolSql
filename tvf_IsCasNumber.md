## Validating a Chemical Abstracts Service (CAS) Registry number?

Quoting from https://en.wikipedia.org/wiki/CAS_Registry_Number
> 
> CASRN is separated by hyphens into three parts, the first consisting from two up to seven digits, the second consisting of two digits, 
> the third consisting of a single digit serving as a check digit.
> 
> The check digit is found by taking the last digit times 1, the preceding digit times 2, the preceding digit times 3 etc., 
> adding all these up and computing the sum modulo 10. 
> 
> For example, the CAS number of water is 7732-18-5
>    * the checksum 5 is calculated as (8×1 + 1×2 + 2×3 + 3×4 + 7×5 + 7×6) = 105
>    * and 105 mod 10 = 5.
> 

Using set-based logic, we can call the inlineable TVF ```dbo.tvf_IsCasNumber(@casNumber) ```
which returns an integer ```isValid``` of 1 for a valid CAS number; otherwise it returns 0.

``` sql
create function dbo.tvf_IsCasNumber( @casNumber varchar(12) )
returns table
as
return	with cte12Rows as
        (
            select  rowNumber = row_number() over (order by (select 1))
            from 
                ( 
                    values  (1),(1),(1),(1)
                        ,   (1),(1),(1),(1)
                        ,   (1),(1),(1),(1)
                ) as twelve(ones)
        )
        select  isValid	= cast(	case
                                when right(@casNumber, 1) like '[^0-9]' then 0                                  -- invalid checkdigit
                                when (sum(cas.positionalValue) % 10) = cast(right(@casNumber, 1) as int) then 1	-- valid checksum
                                else 0                                                                          -- invalid checksum
                                end
                            as int)
        from
        (   -- multiply each digit by its position (row number) in the numerics-only CAS number, from right to left
            select  positionalValue = cast(cdp.digit * (row_number() over (order by (select 1)))  as tinyint)
            from
            (   -- suppress non-numerics in the processing stream, from right-to-left (aka reverse), by nulling them out of the result set
                select  digit =	cast(	case 
                                        when substring(reverse(@casNumber), i.[start], 1) like '[0-9]' then substring(reverse(@casNumber), i.[start], 1) 
                                        else null 
                                        end 
                                    as tinyint)
                from 
                (
                    select  [start] = r.rowNumber
                    from    cte12Rows as r
                    where   r.rowNumber between 3 and 12 - (12 - len(@casNumber)) -- exclude the right most checkdigit and its preceeding dash from processing
                ) as i	-- iterator
            ) as cdp  -- CAS digit position
            where cdp.digit is not null
        ) as cas;
``` 
Example:

```
select  n.casNumber
    ,   isValid = (select isValid from dbo.tvf_IsCasNumber(n.casNumber))
from    dbo.checksumNumbers as n;
```
| casNumber | isValid |
|-----------|---------|
|2051-24-3|1
|107-07-3|1
|7732-18-5|1
|NULL|0
|7732-18-4|0
|abcd|0

Way Cool, huh?
