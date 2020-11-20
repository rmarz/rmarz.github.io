---
layout: simple
title: Characterset and Locale Settings for SQLcl
ref: blog-2016-09-02-SQLcl_Charset
date: 2016-09-02
lang: en
permalink: /en/blog/sqlcl-charset/
---

# Characterset and Locale Settings for SQLcl

Anyone who wants to produce or process data in different character sets or location, must be able to adapt its client environment dynamically.

## Good old times: sqlplus & co
In the world of Oracle tools this behavior is controlled by the environment variable $NLS_LANG:

```bash
export NLS_LANG=GERMAN_GERMANY.WE8ISO8859P15
```
This causes Oracle tools to:

* output all messages in German language
* adjust decimal, currency symbol, and so on accordingly
* and set the input and output character set to WE8ISO8859P15.

This has been working for more than twenty years on both Windows and *nix environments:

```bash
[oracle@vbgeneric ~]$ env|grep NLS_LANG
NLS_LANG=GERMAN_GERMANY.utf8
[oracle@vbgeneric ~]$ sqlplus rmarz/rmarz
SQL*Plus: Release 12.1.0.2.0 Production on Mo Aug 29 19:59:11 2016
[...]
SQL> create table t_umlaut as select 'Umlaut: ÄÖÜäöüß' txt from dual;

Table created.

SQL> spool umlaut.utf-8.sqlplus.txt
SQL> select * from t_umlaut;

TXT
------------------------------------------------------------------
Umlaut: ÄÖÜäöüß

SQL> spool off
SQL> exit
[...]
[oracle@vbgeneric ~]$ export NLS_LANG=GERMAN_GERMANY.WE8ISO8859P15
[oracle@vbgeneric ~]$ sqlplus rmarz/rmarz
SQL*Plus: Release 12.1.0.2.0 Production on Mo Aug 29 20:07:08 2016
[...]
SQL> spool umlaut.8859-15.sqlplus.txt
SQL> select * from t_umlaut;

TXT
----------------------
Umlaut: ▒▒▒▒▒▒▒

SQL> spool off
[...]

[oracle@vbgeneric ~]$ file umlaut.*
umlaut.8859-15.sqlplus.txt: ISO-8859 text
umlaut.utf-8.sqlplus.txt:   UTF-8 Unicode text

```
In the second case the umlauts are not displayed properly on the screen due to the terminal running under UTF-8.

The generated files have the expected encoding, as can be seen from the output of the "file"-command.
On Windows, this behavior is identical.

## Modern Times: SQLcl

In contrast, the modern SQLcl does not evaluate $NLS_LANG. The tool is written in Java and automatically uses the settings of the session it was started from.

On Linux and similar systems session settings can be  easily customized. They are controlled by the environment variable $LANG:

```bash
[oracle@vbgeneric ~]$ unset NLS_LANG
[oracle@vbgeneric ~]$ env|grep LANG
LANG=en_US.UTF-8
[oracle@vbgeneric ~]$ sql rmarz/rmarz
SQLcl: Release 4.2.0.16.153.2014 RC on Mon Aug 29 20:27:44 2016
[...]
SQL> spool umlaut.utf-8.sqlcl.txt
SQL> select * from t_umlaut;

TXT
----------------------
Umlaut: ÄÖÜäöüß

SQL> spool off
SQL> exit

[...]
[oracle@vbgeneric ~]$ export LANG=de_DE.iso885915@euro
[oracle@vbgeneric ~]$ sql rmarz/rmarz
SQLcl: Release 4.2.0.16.153.2014 RC auf Mo Aug 29 20:40:27 2016
[...]

SQL> spool umlaut.8859-15.sqlcl.txt
SQL> select * from t_umlaut;

TXT
----------------------
Umlaut: ÄÖÜäöüß

SQL>
Abgemeldet von Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
[...]
[oracle@vbgeneric ~]$ file umlaut.*
umlaut.8859-15.sqlcl.txt:   ISO-8859 text
umlaut.8859-15.sqlplus.txt: ISO-8859 text
umlaut.utf-8.sqlcl.txt:     UTF-8 Unicode text
umlaut.utf-8.sqlplus.txt:   UTF-8 Unicode text
[oracle@vbgeneric ~]$
```

## And what about Windows?

On Windows, however, Java ignores the environment variable $LANG.
The configuration is read from the Windows system settings:
[How do I view and change the system locale settings to use my language of choice?](https://java.com/en/download/help/locale.xml)

Changing these settings affects all other applications on the machine as well.
Fortunately, Java evaluates additional variables. The variable $JAVA_TOOLS_OPTIONS come in handy:

```ps1
PS C:\Users\rmarz> sql.bat rmarz@192.168.56.100/orcl
SQLcl: Release 4.2.0.16.175.1027 RC auf Di Aug 30 03:45:21 2016
[...]

SQL> spool umlaut.8859p15.sqlcl.win.txt
SQL> select * from t_umlaut;

TXT
----------------------
Umlaut: ├ä├û├£├ñ├Â├╝├ƒ

SQL> spool off
[...]

PS C:\Users\rmarz>
PS C:\Users\rmarz>
PS C:\Users\rmarz> $env:JAVA_TOOL_OPTIONS='-Duser.language=en -Duser.region=US -Dfile.encoding=UTF-8'
PS C:\Users\rmarz>
PS C:\Users\rmarz> sql.bat rmarz/rmarz@192.168.56.100/orcl

Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.region=US -Dfile.encoding=UTF-8

SQLcl: Release 4.2.0.16.175.1027 RC on Tue Aug 30 03:47:12 2016
[...]
SQL> spool umlaut.utf-8.sqlcl.win.txt
SQL> select * from t_umlaut;

TXT
----------------------
Umlaut: ├ä├û├£├ñ├Â├╝├ƒ

SQL> spool off
SQL> exit
[...]
PS C:\Users\rmarz>

PS C:\Users\rmarz> bash

rmarz@Tinkerbell MINGW64 ~
$ file umlaut.*
umlaut.8859p15.sqlcl.win.txt: ISO-8859 text, with CRLF line terminators
umlaut.utf-8.sqlcl.win.txt:   UTF-8 Unicode text, with CRLF line terminators

rmarz@Tinkerbell MINGW64 ~
```
The line "Picked up JAVA_TOOL_OPTIONS:" shows that this trick worked.
The screen output is incorrect, but the results in the spool files are fine.
We got Windows line endings - which is to be expected on Windows, of course.

There is no "file"-command in powershell. I used new bash for Windows 10 by Microsoft instead.

Further information on localization in Java can be found here:
[Internationalization: Understanding Locale in the Java Platform](http://www.oracle.com/technetwork/articles/javase/locale-140624.html)
