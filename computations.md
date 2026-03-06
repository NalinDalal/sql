# Computations in SQL
Various like:
- Addition +
- Subtraction -
- Division /
- multiplication *
- Exponentiation **
- Enclosed ()

# Renaming Column Used with Expression List
```sql
SELECT <ColumnName> <AliasName>, <ColumnName> <AliasName>
FROM <TableName>;
```

# Logical Operators

## AND
AND operator allows creating an SQL statement based on two or more conditions being met.
```sql
SELECT * FROM Orders WHERE amount<13000 AND amount>0;
```

## OR
records are returned when any one of the conditions are met.
```sql
SELECT * FROM Orders WHERE amount=13000 OR amount>0;
```

**Both AND and OR can be combined for various purposes**

## Range Searching/BETWEEN
b/w a specified lower limit and upper limit
range coded is inclusive
```sql
SELECT * FROM TRANS MSTR WHERE TO_CHAR(DT, 'MM') BETWEEN 01 AND 03;
```


# Oracle Function
manipulate data item and return result. formats:
Function_Name(argument1, argumentz,..)

Group Functions (Aggregate Functions)act on a set of values. For example, SUM, is’a function, returns a single result row for a group of queried rows.
Scalar Functions (Single Row Functions)act on 1 value at time; ex: LENGTH

## Aggregate
- `AVG: `returns avg of n values-> `AVG ([<DISTINCT>|<ALL>] <n>)`
- `MIN: `returns min of expr-> `MIN([<DISTINCT>|<ALL>] <expr>)`
- `COUNT(expr):`returns umber of rows where expr is not null.->`COUNT([< DISTINCT >|<ALL>] <expr>)`
- `MAX:`return max value of expr->`MAX([<DISTINCT>|<ALL>] <expr>)`
- `SUM:` Returns the sum of the values of 'n'.-> `SUM([<DISTINCT>|<ALL>] <n>)`

## NUMERIC
- `ABS:` absolute value of 'n'.->`ABS(n)`; `SELECT ABS(-15) "Absolute" FROM DUAL;`
- `POWER:` `POWER(m,n)`-> m raised to power n
- `ROUND: `return n rounded to m places, if m removed then to 0.->`ROUND(n[, m])`; `SELECT ROUND(15.19,1) "Round" FROM DUAL;`
- `SQRT:`returns square root-> `SQRT(n)` ;Ex:`SELECT SQRT(25)`
- `EXP:`e to power n-> `EXP(n)`;Ex:`SELECT EXP(5) "Exponent" FROM DUAL;`
- `EXTRACT:`return extracted value from date-> 
    - `EXTRACT( {year | month | day | hour | minute | second | timezone_hour |
        timezone_minute | timezone_region | timezone_abbr}
        FROM { date_value | interval_value } )`
    - `SELECT EXTRACT(YEAR FROM DATE '2004-07-02') "Year",EXTRACT(MONTH FROM SYSDATE) "Month" FROM DUAL;`
- `GREATEST:`return greatest value->`GREATEST(expr1, expr2,’... expr_n)`;ex: `SELECT GREATEST(4, 5, 17) "Num", GREATEST(‘4’, '5', '17') "Text" FROM DUAL;`
- `LEAST:`return least value->`LEAST(expr1, expr2, ... expr_n)`;ex: `SELECT LEAST(4, 5, 17) "Num", LEAST(‘4’, '5', '17') "Text" FROM DUAL;`

- `MOD:`return n mod m->`MOD(m, n)`;ex: `SELECT MOD(15, 7) "Mod1", MOD(15.7, 7) "Mod2" FROM DUAL;`

- `TRUNC:`number truncated to no of decimal place->`TRUNC(number, [decimal_places])`;ex:`SELECT TRUNC(125.815, 1) "Truncl", TRUNC(125.815, -2) "Trunc2" FROM DUAL;`

- `FLOOR:`floor value of num->
    `FLOOR(n)`;
    ex:`SELECT FLOOR(24.8) "Fir1", FLOOR(13.15) "Fir2" FROM DUAL;`
- `CEIL:`ceil value of num->
    `CEIL(n)`;
    ex: `SELECT CEIL(24.8) "Ceil1",CEIL(13.15) "Ceil2" FROM DUAL;`

## STRING
- `LOWER(char)`:Returns char, with all letters in lowercase;`SELECT LOWER(IVAN BAYROSS') "Lower" FROM DUAL;`
- `INITCAP(char)`:eturns a string with the first letter of each word in uppercase.;`SELECT INITCAP(IVAN BAYROSS') "Title Case" FROM DUAL;`
- `UPPER (char)`:Returns char, with all letters forced to uppercase.;`SELECT UPPER('Ms. Carol’) "Capitalised" FROM DUAL;`
- `SUBSTR`:Return portion of characters;`SUBSTR(<string>, <start_position>, [< length>])`;Ex:`SELECT SUBSTR(‘SECURE',3,4) "Substring" FROM DUAL;`
- `ASCII`:Return number code representing specified character;`SELECT ASCII(‘a') "ASCII", ASCII(‘A') "ASCII2" FROM DUAL;`
- `COMPOSE`:Returns a Unicode string.`COMPOSE(<single >)`;ex: `SELECT COMPOSE(‘a' || UNISTR(‘\0301')) "Composed" FROM DUAL;`
- `DECOMPOSE`:Accepts a string and returns a Unicode string.`DECOMPOSE(<single >)`;`SELECT DECOMPOSE(COMPOSKE(‘a' || UNISTR(‘\0301'))) "Decomposed" FROM DUAL;`
- `INSTR`:Returns the location of a substring in a string.syntax: `INSTR(<string1>, <string2>, [<start_position>], [<nth_appearance >])`
    ex: SELECT INSTR(‘SCT on the net’, 't') "Instr1", INSTR(‘SCT on the net’, 't', 1, 2) "Instr2" FROM DUAL;
- `TRANSLATE`:Replaces a sequence of characters in a string with another set of characters.replaces a single character at a time.
    Syntax:`TRANSLATE(<string1>, <string_to_replace>, <replacement_string>)`
    ex:`SELECT TRANSLATE(Isct523’, '123', '7a9') "Change" FROM DUAL;`
- `LENGTH`:Returns the length of a word.
    `LENGTH(word)`;
    ex: `SELECT LENGTH('SHARANAM)) "Length" FROM DUAL;`
- `LTRIM`:Removes characters from the left of char with initial characters removed upto the first character not in set.
    `LTRIM(char[,set])`
    `SELECT LTRIM(‘NISHA'|N’') "LTRIM" FROM DUAL;`
- `RTRIM`:remove the char after last not in set, and then return char;`RTRIM (char, [set])`
    `SELECT RTRIM(‘SUNILA','A’) "RTRIM" FROM DUAL;`
- `TRIM`:Removes all specified characters either from the beginning or the ending of a string;
    `TRIM( [leading | trailing | both [<trim_character> FROM ]] <string1> )`
    leading - remove trim_string from the front of string].
    trailing - remove trim_string from the end of string].
    both - remove trim_string from the front and end of string].
    If none of the above option is chosen, the TRIM function will remove trim_string from both the front and end of string1.
    trim_character is the character that will be removed from string]. If omitted, will remove all leading and trailing spaces from string1.
    string1 is the string to trim.
   ex: `SELECT TRIM(' Hansel ') "Trim both sides" FROM DUAL;`
- `LPAD`:Returns charl, left-padded to length n with the sequence of characters specified in char2.if char2 not specified, then use blank by default.
    syntax: `LPAD(char1,n [,char2])`
    ex: `SELECT LPAD(Page 1',10,'*') "LPAD" FROM DUAL;`

- `RPAD`:Returns charl, right-padded to length n with the characters specified in char2. If char2 is not specified, Oracle uses blanks by default.
    Syntax:`RPAD(char1 ,n[,char2])`
    Example:`SELECT RPAD(FNAME,10,'x') "RPAD Example" FROM CUST MSTR WHERE FNAME = 'Ivan';`

- `VSIZE`:Returns the number of bytes in the internal representation of an expression.
    Syntax:`VSIZE(<expression>)`
    ex: `SELECT VSIZE(‘SCT on the net’) "Size" FROM DUAL;`

## Conversion Function
- `TO_NUMBER`: Converts char, a CHARACTER value expressing a number, to a NUMBER datatype.
    Syntax:`TO_NUMBER(char)`
    Example:`UPDATE ACCT_MSTR SET Curbal = Curbal + TO_NUMBER(SUBSTR(‘$100',2,3));`
- `TO_CHAR`:Converts a value of a NUMBER datatype to a character datatype,
    Syntax:`TO_CHAR (n{[, fmt])`
    Example:`SELECT TO_CHAR(17145, '$099,999') "Char" FROM DUAL;`
- `TO_CHAR(date conv.)`:convert DATE to CHAR
    Syntax:`TO_CHAR(date[, fmt])`
    Example: `SELECT TO_CHAR(DT, 'Month DD, YYYY’') "New Date Format" FROM Trans_Mstr WHERE Trans No ='T1';`

## Date Conversion
DATE is spcl data type used to store date and time information. format:'DD-MON-YY HH:MI:SS'.
- `TO_DATE()`:used to specify the required format.Converts a character field to a date field.
    syntax:`TO_DATE(char [, fmt])`
    Example:`INSERT INTO CUST_MSTR(CUST_NO, FNAME, MNAME, LNAME, DOB _INC)
VALUES(‘C1', Ivan’, 'Nelson’, 'Bayross',
TO_DATE('25-JUN-1952 10:55 A.M.',"DD-MON-YY HH:MI A.M.'));`

## Date Functions
- `ADD_MONTHS:` Returns date after adding the number of months specified in the function.
    Syntax:`ADD_MONTHS(d,n)`
    Example:`SELECT ADD_MONTHS(SYSDATE, 4) "Add Months" FROM DUAL;`
- `LAST_DAY:` Returns the last date of the month specified with the function.
    Syntax: `LAST_DAY(d)`
    Example: `SELECT SYSDATE, LAST_DAY(SYSDATE) "LastDay" FROM DUAL;`
- `MONTHS _BETWEEN`: Returns number of months between di and d2.
    Syntax:`MONTHS_BETWEEN(d1, d2)` 
    Example:`SELECT MONTHS BETWEEN('02-FEB-92', '02-JAN-92') "Months" FROM DUAL;`
- `NEXT_DAY`: Returns the date of the first weekday named by char that is after the date named by date.char must be a day of the week.
    Syntax:`NEXT_DAY(date, char)`
    Example:`SELECT NEXT_DAY('06-JULY-02', 'Saturday') "NEXT DAY" FROM DUAL;`
-   `ROUND`: Returns a date rounded to a specific unit of measure. If the second parameter is omitted, the ROUND function will round the date to the nearest day.
    Syntax:`ROUND(date, [format])`
    Example:`SELECT ROUND(TO_DATE(‘'01-JUL-04'), "'YYYY’') "Year" FROM DUAL;`
- `NEW_TIME`: Returns the date after converting it from time zonel to a date in time zone2.
    Syntax:`NEW_TIME(date, zone1, zone2)`
    Ex: `SELECT NEW_TIME(TO_DATE('2004/07/01 01:45’, 'yyyy/mm/dd HH24:MI'), 'AST', 'MST') "MST" FROM DUAL;`

## MANIPULATING DATES 
- `TO_CHAR`:facilitates the retrieval of data in a format different from the default format. 
    Syntax:`TO_CHAR(<date value> [,<fmt>])`;date value stands for the date and fmt is the specified format in which date is to be displayed.
    Ex: `SELECT TO_CHAR(SYSDATE, 'DD-MM-YY') FROM DUAL;`

- `TO_DATE`:converts a char value into a date value.allows a user to insert date into a date column in anyrequired format, by specifying the character value of the date to be inserted and its format.
    Syntax:`TO_DATE(<char value>[,<fmt>])`
    Ex: `SELECT TO_DATE ('06/07/02', 'DD/MM/YY') FROM DUAL;`

# Miscellaneous
- `UID`: This function returns an integer value corresponding to the UserID of the user currently logged in.
    `UID [INTO <variable >]` where, variable will now contain the id number for the user's session.
    Ex: `SELECT UID FROM DUAL;`
- `USER`: This function returns the user name of the user who has logged in. The value returned is in varchar2 data type.
    Syntax:`USER`
    Example:`SELECT USER FROM DUAL;`
- `SYS_CONTEXT`: Can be used to retrieve information about Oracle’s environment.
    Syntax:`SYS_CONTEXT (<namespace>, <parameter>, [<length>])`
    ex:`SELECT SYS_CONTEXT(‘USERENV', 'NLS_ DATE FORMAT’) "SysContext" FROM DUAL;`

- `USERENYV`: Can be used to retrieve information about the current Oracle session.
    Syntax: `USERENV(<parameter >)`where, parameter is the value to return from session
    ex: `SELECT USERENV(‘LANGUAGE') FROM DUAL;`

- COALESCE: Returns the first non-null expression in the list. If all expressions evaluate to null, then the
coalesce function will return null.
Syntax:
COALESCE(<expri>, <expr2>, ... <expr_n>)
Example:
SELECT COALESCE(FNAME, CUST_NO) Customers FROM CUST_MSTR;
The above coalesce statement is equivalent to the following IF-THEN-ELSE statemen
