# UNDERSTANDING THE SQL *PLUS ENVIRONMENT VARIABLES
Given below is a brief explanation of the functionality of the SQL *Plus environment variables and their
attribute/values.
## SQL Syntax Or Command Based Attributes
1. `ARRAY[SIZE] {15|n}`
Sets the number of rows called a batch, that SQL *Plus will fetch from the database at one time. Valid
values are | to 5000. A large value increases the efficiency of queries and sub-queries that fetch many
rows, but requires more memory. Values over approximately 100 provide little added performance.
ARRAYSIZE has no effect on the results of SQL *Plus operations other than increasing efficiency.
2. `AUTOT[RACE] {OFF|ON|TRACE{ONLY]} [EXP[LAIN]] [STAT[ISTICS]]`
Displays a report on the execution of successful SQL DML statements (SELECT, INSERT, UPDATE or
DELETE). The report can include execution statistics and the query execution path.
OFF does-not display a trace report. ON displays a trace report. TRACEONLY displays a trace report, but
does not print query data, if any. EXPLAIN shows the query execution path by performing an EXPLAIN
PLAN. STATISTICS displays SQL statement statistics.
Using ON or TRACEONLY with no explicit options defaults to EXPLAIN STATISTICS.
The TRACEONLY option may be useful to suppress the query data of large queries. If STATISTICS is
specified, SQL *Plus still fetches the query data from the server, however, the data is not displayed.

The AUTOTRACE report is printed after the statement has successfully completed.

When SQL*Plus produces a STATISTICS report, a second connection to the database is automatically
created. This connection is closed when the STATISTICS option is set to OFF, or the user logs out of
SOL*Plus.

The formatting of the AUTOTRACE report may vary depending on the version of the server to which one
is are connected and the configuration of the server.
AUTOTRACE is not available when FIPS flagging is enabled.
`BLO[CKTERMINATOR] {.|c}`
Sets the non-alphanumeric character used to end PL/SQL blocks to character expression. To execute the
block, issue a RUN or / (slash) command.

`CMDS[EP] {;|c|OFF|ON}`
Sets the non-alphanumeric character used to separate multiple SQL *Plus commands entered on one line to
c. ON or OFF controls whether one can enter multiple commands on a line. ON automatically sets the
command separator character to a semicolon (;).

`CON[CAT] {.|c|OFF|ON}`
Sets the character that can be used to terminate a substitution variable reference if a user wishes to
immediately follow the variable with a character that SQL *Plus would otherwise interpret as a part of the
substitution variable name. SQL *Plus resets the value of CONCAT to a period when a user switches
CONCAT on.

`COPYC[OMMIT] {0|n}`
Controls the number of batches after which the COPY command commits changes to the database. COPY
commits rows to the destination database each time it copies n row batches. Valid values are zero to 5000.
The size of a batch can be set with the ARRAYSIZE variable. If COPYCOMMIT is set to zero, COPY
performs a commit only at the end of a copy operation.

`COPYTYPECHECK {OFF|ON}`
Sets the suppression of the comparison of datatypes while inserting or appending to tables with the COPY
command. This is to facilitate copying to DB2, which requires that a CHAR be copied to a DB2 DATE.

`DEF[INE] {&|c|OFF|ON}`
Sets the character used to prefix substitution variables to character expression. ON or OFF controls
whether SQL *Plus will scan commands for substitution variables and replace them with their values. ON
changes the value of character expression back to the default '&', not the most recently used character. The
setting of DEFINE to OFF overrides the setting of the SCAN variable.

`ESC[APE] {\|c|(OFF|ON}`
Defines the character that a user enters as the escape character. OFF undefines the escape character. ON
enables the escape character. ON changes the value of character expression back to the default '\.

`FEED{BACK] {6|n|OFF|ON}`
Displays the number of records returned by a query when a query selects at least one record. ON or OFF
turns this display on or off. Turning feedback ON sets the number of records to 1. Setting feedback to zero
is equivalent to turning it OFF.

`LONG {80|n}`
Sets maximum width (in bytes) for displaying LONG, CLOB and NCLOB values; and for copying LONG
values. The maximum value of n is 2 gigabytes.

`LONGC[HUNKSIZE] {80|n}`
Sets the size (in bytes) of the increments in which SQL *Plus retrieves a LONG, CLOB or NCLOB value.

`NUMF[ORMAT] format`
Sets the default format for displaying numbers. Enter a number format for format.

`NUM[WIDTH] {10jn}`
Sets the default width for displaying numbers.

`SQLC[ASE] {MIX[ED]|/LO[WER]|UP[PER]}`
Converts the case of SQL commands and PL/SQL blocks just prior to execution. SQL *Plus converts all
text within the command, including quoted literals and identifiers, as follows:
Q uppercase if SQLCASE equals UPPER
Q lowercase if SQLCASE equals LOWER
Q unchanged if SQLCASE equals MIXED
Q SQLCASE does not change the SQL buffer itself

`SQLBL[ANKLINES] {ON|OFF}`
Controls whether SQL *Plus allows blank lines within an SQL command. ON interprets blank lines and
new lines as part of an SQL command. OFF, the default value, does not allow blank lines or new lines in
an SQL command. SQL *Plus returns to its default behavior when an SQLTERMINTATOR or
BLOCKTERMINATOR is encountered.

`SQLCO[NTINUE] {> |text}`
Sets the character sequence SQL *Plus displays as a prompt after a user continues SQL *Plus command on
an additional line using a hyphen (-).

`SQLN[UMBER] {OFF\ON}`
Sets the prompt for the second and subsequent lines of an SQL command or PL/SQL block. ON sets the
prompt to be the line number. OFF sets the prompt to the value of SQLPROMPT.

`SQLPRE[FIX] {#{\c}`
Sets the SQL *Plus prefix character. While a user is entering an SQL command or PL/SQL block, the user
can enter an SQL *Plus command on a separate line, prefixed by the SQL *Plus prefix character. SQL
*Plus will execute the command immediately without affecting the SQL command or PL/SQL block that is
being entered. The prefix character must be a non-alphanumeric character.

`SQLT[ERMINATOR] {;|c|OFF|ON}`
Sets the character used to end and execute SQL commands. OFF means that SQL *Plus recognizes that
there is no command terminator and sq! should terminate an SQL command by entering an empty line. ON
resets the terminator to the default semicolon (;).

`ECHO {OFEF|ON}`
Controls whether the START command lists each command in a command file as the command is
executed. ON lists the commands; OFF suppresses the listing.

`EDITF[ILE] file_name|.ext]`
Sets the default filename for the EDIT command.
A user can include a path and/or file extension. For information on changing the default extension, see the
SUFFIX variable of this command. The default filename and maximum filename length are operating
system specific.

`FLAGGER {OFF|ENTRY |INTERMED|[IATE]|FULL}`
Checks to make sure that SQL statements conform to the ANSI/ISO SQL92 standard. If any non-standard
constructs are found, the Oracle Server flags them as errors and displays the violating syntax. This is the
equivalent of the SQL language ALTER SESSION SET FLAGGER command.
A user may execute SET FLAGGER even if user is not connected to a database. FIPS flagging will remain
in effect across SQL *Plus sessions until a SET FLAGGER OFF (or ALTER SESSION SET FLAGGER =
OFF) command is successful or the user exits SQL *Plus.
When FIPS flagging is enabled, SQL *Plus displays a warning for the CONNECT, DISCONNECT, and
ALTER SESSION SET FLAGGER commands, even if they are successful.

`FLU[SH] {OFF|ON}`
Controis when output is sent to the user's display device. OFF allows the host operating system to buffer
output. ON disables buffering.
Use OFF only when a user wants to run a command file non-interactively (that is, when there is no need to
see output and/or prompts until the command file finishes running). The use of FLUSH OFF may improve
performance by reducing the amount of program I/O.

`INSTANCE [instance_path|LOCAL]`
Changes the default instance for SQL *PLUS session to the specified instance path. Using the SET
INSTANCE command does‘not connect to a database. The default instance is used for commands when no
instance is specified.
Any commands preceding the first use of SET INSTANCE communicate with the default instance.
To reset the instance to the default value for a user's operating system, a user can either enter SET
INSTANCE with no instance_path or SET INSTANCE LOCAL.

`LIN[ESIZE] {80|n}`
Sets the total number of characters that SQL *Plus displays on one line before beginning a new line. It also
controls the position of centered and right-aligned text in TTITLE, BTITLE, REPHEADER and
REPFOOTER. A user can define LINESIZE as a value from | to a maximum that is system dependent.
Refer to the Oracle installation and user's manual(s) provided for the operating system.

`SERVEROUT[PUT] {OFF|ON} [SIZE n] [FOR[MAT] {WRA[PPED]|WOR[D_WRAPPED]|TRU[NCATED}]}]`
Controls whether to display the output (i.e. DBMS _OUTPUT.PUT_LINE) of stored procedures or PL/SQL
blocks in SQL *Plus. OFF suppresses the output of DBMS OUTPUT.PUT_LINE; ON displays the
output.
The SIZE attribute sets the number of bytes of the output that can be buffered within the Oracle9i database
server. The default value for the number of bytes is 2000. Additional, the number of bytes cannot be less
than 2000 or greater than 1,000,000.
When WRAPPED is enabled SQL *Plus wraps the server output within the line size specified by SET
LINESIZE, beginning new lines when required.
When WORD WRAPPED is enabled, each line of server output is wrapped within the line size specified
by SET LINESIZE. Lines are broken on word boundaries. SQL *Plus left justifies each line, skipping all
leading whitespace.
When TRUNCATED is enabled, each line of server output is truncated to the line size specified by SET
LINESIZE.
For each FORMAT, every server output line begins on a new output line.

`SQLP[ROMPT] {SQL>|text}`
Sets the SQL *Plus command prompt.

`SUF[FIX] {SQL|text}`
Sets the default file extension that SQL *Plus uses in commands that refer to command files. SUFFIX does
not control extensions for spool files.

`SHOW[MODE] {OFF|ON}`
Controls whether SQL *Plus lists the old and new settings of a SQL *Plus system variable when a user
changes the setting with SET. ON lists the settings; OFF suppresses the listing. SHOWMODE ON has the
same behavior as the obsolete SHOWMODE BOTH.

`TAB {OFF|ON}`
Determines how SQL *Plus formats white space in terminal output. OFF uses spaces to format white space
in the output. ON uses the TAB character. TAB settings are every eight characters. The default value for
TAB is system dependent.

`TERM[OUT] {OFF|ON}`
Controls the display of output generated by commands executed from a command file. OFF suppresses the
display so that a user can spool output from a command file without seeing the output on the screen. ON
displays the output. TERMOUT OFF does not affect output from commands you enter interactively.

`TI[ME] {QFF|ON}`
Controls the display of the current time. ON displays the current time before each command prompt. OFF
suppresses the time display.

`TIMI[NG] {OFF|ON}`
Controls the display of timing statistics. ON displays timing statistics on each SQL command or PL/SQL
block run. OFF suppresses timing of each command.

`TRIM[OUT] {OFF|ON}`
Determines whether SQL *Plus allows trailing blanks at the end of each displayed line. ON removes blanks
at the end of each line, improving performance especially when a user accesses SQL *Plus from a slow
communications device. OFF allows SQL *Plus to display trailing blanks. TRIMOUT ON does not affect
spooled output.

`TRIMS[POOL] {ON|OFF}`
Determines whether SQL *Plus allows trailing blanks at the end of each spooled line. ON removes blanks
at the end of each line. OFF allows SQL *Plus to include trailing blanks. TRIMSPOOL ON does not affect
terminal output.

`VER[IFY] {OFF|ON}`
Controls whether SQL *Plus lists the text of an SQL statement or PL/SQL command before and after SQL
*Plus replaces substitution variables with values. ON lists the text; OFF suppresses the listing.

`WRA|P] {OFF|ON}`
Controls whether SQL *Plus truncates the display of a SELECTed row if it is too long for the current line
width. OFF truncates the SELECTed row; ON allows the SELECTed row to wrap to the next line.
Use the WRAPPED and TRUNCATED clauses of the COLUMN command to override the setting of
WRAP for specific columns.

# ER Model
some sample bank accounts model-> name, acct_no, sf_no, lf_no, branch_no etc
