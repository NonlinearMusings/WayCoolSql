# ForEach loops in T-SQL

Did you know T-SQL supports ForEach? 
No? Well, don't feel too bad because it's called OUTER APPLY!

Loops are incredibly useful, unless you're trying to write fast T-SQL code. *\(Can you say [RBAR?](https://www.red-gate.com/simple-talk/sql/t-sql-programming/rbar-row-by-agonizing-row/)\)*
Too frequently SQL Developers fall into the non-set-based mindset of iterating over tables using T-SQL's
WHILE statement; solving the problem but sacrificing performance while doing so. *\(pun intended\)*
>Now, I haven't seen everything, but I have seen a lot, and I've yet to see a WHILE loop that cannot be
>refactored out of the algorithm unless a stored procedure is being called from within the loop. There may be other cases,
>but I've yet to encounter them.

**Alright already - shut up and show me the code!**

```sql
select  iteration       = row_number() over (order by (select 1))
    ,   [data.name]     = d.[name]
    ,   [data.id]       = d.id
    ,   [outer.i]       = [outer].i
    ,   [inner.j]       = [inner].j
    ,   [exp1]          = concat(d.id, [outer].i, [inner].j)
    ,   [exp2]          = d.id * [outer].i * [inner].j
from
    (
        values  (1, 'one')
            ,   (2, 'two')
            ,   (3, 'three')
    ) as d(id, [name]) -- data table
outer apply
    (
        select n.i
        from
            (   values (10)
                    ,  (20)
            ) as n(i)
    ) as [outer] -- outer loop
outer apply
    (
        select n.j
        from
            (   values (100)
                    ,  (200)
                    ,  (300)
            ) as n(j)
    ) as [inner]; -- inner loop
```
Take note of how the data table's rows are being iterated over as if a ForEach were being used
and how the OUTER APPLY operators are applying the i and j data sets as nested loops!

Way Cool, huh?

| iteration | data.name | data.id | outer.i | inner.j |exp1   | exp2   |
|-----------|-----------|---------|---------|---------|-------|--------|
| 1         | one	    | 1       | 10      | 100     |110100 | 1000   |
2|one|1|10|200|110200|2000
3|one|1|10|300|110300|3000
4|one|1|20|100|120100|2000
5|one|1|20|200|120200|4000
6|one|1|20|300|120300|6000
7|two|2|10|100|210100|2000
8|two|2|10|200|210200|4000
9|two|2|10|300|210300|6000
10|two|2|20|100|220100|4000
11|two|2|20|200|220200|8000
12|two|2|20|300|220300|12000
13|three|3|10|100|310100|3000
14|three|3|10|200|310200|6000
15|three|3|10|300|310300|9000
16|three|3|20|100|320100|6000
17|three|3|20|200|320200|12000
18|three|3|20|300|320300|18000
