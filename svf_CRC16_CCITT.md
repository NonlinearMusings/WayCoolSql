# CRC16 in T-SQL

This one probably belongs in the category of just-because-you-can-doesn't-mean-you-should... but phooey on that!

Back in the days of SQL Server 2008-R2 I came across this fantastic article by Thomas Kejser [Writing New Hash Functions for SQL Server](https://techcommunity.microsoft.com/t5/sql-server-blog/writing-new-hash-functions-for-sql-server/ba-p/305123).
The other day I was thinking about SQL hashing again (doesn't everyone?) and thought that a set-based implementation of CRC16 would make a good add to Way Cool Sql... and here we are!

Firstly
* The core of this code is from AlwaysLearing's reply at [CRC-16 SQL CCTI](https://stackoverflow.com/questions/75839677/crc-16-sql-ccti)
* My adaptation relies on T-SQL's quirky update behavior, which is called out as an [antipattern](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/select-local-variable-transact-sql?view=sql-server-ver16#c-antipattern-use-of-recursive-variable-assignment) in Microsoft's SELECT @local_variable documentation.
* My implementation fails the [Inlineable scalar UDF requirements](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining?view=sql-server-ver16#requirements) thereby making it slower than it otherwise would be.

However
* After researching quirky updates a bit more I am reasonably convinced that the 'antipattern' designation should only apply when ORDER BY is used.
* Microsoft will not support this 'antipattern'.
* It **is** some pretty cool T-SQL...
* It works on my machine! :zany_face:

Read more about CRC at [On-line CRC calculation and free library](https://www.lammertbies.nl/comm/info/crc-calculation) and compare the outputs at [crccalc.com](https://crccalc.com/)

```sql
create function dbo.svf_CRC16_CCITT( @input varchar(1024) )
	returns int
as
begin
	-- CRC-16/AUG-CCITT using polynomial 0x1021 and initialization vector 0x1D0F
	-- Compare results against: 

    declare     @crc    int            = 0x1D0F -- initialization vector 
        ,    @lookup	binary(512)    =        -- 256 * 2-byte lookup values = 512 bytes
				0x0000102120423063408450A560C670E781089129A14AB16BC18CD1ADE1CEF1EF +
				0x123102103273225252B5429472F762D693398318B37BA35AD3BDC39CF3FFE3DE +
				0x246234430420140164E674C744A45485A56AB54B85289509E5EEF5CFC5ACD58D +
				0x365326721611063076D766F6569546B4B75BA77A97198738F7DFE7FED79DC7BC +
				0x48C458E5688678A70840186128023823C9CCD9EDE98EF9AF89489969A90AB92B +
				0x5AF54AD47AB76A961A710A503A332A12DBFDCBDCFBBFEB9E9B798B58BB3BAB1A +
				0x6CA67C874CE45CC52C223C030C601C41EDAEFD8FCDECDDCDAD2ABD0B8D689D49 +
				0x7E976EB65ED54EF43E132E321E510E70FF9FEFBEDFDDCFFCBF1BAF3A9F598F78 +
				0x918881A9B1CAA1EBD10CC12DF14EE16F108000A130C220E35004402570466067 +
				0x83B99398A3FBB3DAC33DD31CE37FF35E02B1129022F332D24235521462777256 +
				0xB5EAA5CB95A88589F56EE54FD52CC50D34E224C314A004817466644754244405 +
				0xA7DBB7FA879997B8E75FF77EC71DD73C26D336F2069116B06657767646155634 +
				0xD94CC96DF90EE92F99C889E9B98AA9AB584448657806682718C008E1388228A3 +
				0xCB7DDB5CEB3FFB1E8BF99BD8ABBBBB9A4A755A546A377A160AF11AD02AB33A92 +
				0xFD2EED0FDD6CCD4DBDAAAD8B9DE88DC97C266C075C644C453CA22C831CE00CC1 +
				0xEF1FFF3ECF5DDF7CAF9BBFBA8FD99FF86E177E364E555E742E933EB20ED11EF0;

	select	@crc = ((@crc << 8) ^ cast(substring(@lookup, ((@crc >> 8) ^ [ascii].code) * 2 + 1, 2) as int)) & 0xFFFF
	from
        ( 
            select  code = ascii(substring(@input, gs.[value], 1))
            from    generate_series(1, len(@input)) as gs
        ) as [ascii];
        
	return @crc;
end;
```
Way Cool, huh?
