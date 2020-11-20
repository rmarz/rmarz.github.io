---
layout: simple
title: Zeichensatz und Spracheeinstellungen bei SQLcl
ref: blogSqlclCharset
date: 2016-09-02
lang: de
permalink: /de/blog/sqlcl-charset/
---

# Zeichensatz und Spracheinstellungen bei SQLcl
Wer Daten in unterschiedlichen Zeichensätzen oder Lokalisierung erzeugen oder verarbeiten will, muss seine Client-Umgebung flexibel anpassen können.

## Die gute alte Zeit: sqlplus & co
In der Oracle-Tool Welt passiert das über die Umgebungsvariable $NLS_LANG:

```bash
export NLS_LANG=GERMAN_GERMANY.WE8ISO8859P15
```
veranlasst Oracle Tools dazu:

* Meldungen in deutsch auszugeben
* Dezimaltrennzeichen, Währungssymbol, usw. entsprechend anzupassen
* und den Ein- und Ausgabezeichensatz auf WE8ISO8859P15 einzustellen.

Das funktioniert schon seit mehr als zwanzig Jahren sowohl unter Windows als auch in *nix Umgebungen:

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
Dass die Umlaute im zweiten Fall nicht richtig am Bildschirm ausgegeben werden, liegt daran, dass das Terminal selbst  unter UTF-8 läuft.

Die erzeugten Dateien haben das erwartete Encoding, wie die Ausgabe des "file"-Befehles zeigt.
Unter Windows ist das Verhalten identisch.

## Moderne Zeiten: SQLcl

Das moderne SQLcl hingegen wertet $NLS_LANG nicht aus. Das Werkzeug ist in Java geschrieben und übernimmt automatisch die Einstellung der Session aus der es gestartet wird.

In der Linux-Welt kann man diese einfach anpassen: sie wird durch die Umgebungsvariable $LANG gesteuert:

```bash
[oracle@vbgeneric ~]$ unset NLS_LANG
[oracle@vbgeneric ~]$ env|grep LANG
LANG=en_US.UTF-8
[oracle@vbgeneric ~]$ sql rmarz/rmarz
SQLcl: Release 4.2.0.16.153.2014 RC on Mon Aug 29 20:27:44 2016
[...]
SQL> spool umlaut.utf-8.SQLcl.txt
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

## Und was ist mit Windows?

Unter Windows wertet Java die  Umgebunsgvariable $LANG leider nicht aus.
Die Einstellungen werden aus den Windows-Systemeinstellungen gelesen:
[How do I view and change the system locale settings to use my language of choice?](https://java.com/en/download/help/locale.xml)

Änderungen an diesen Einstellungen gelten für alle Anwendungen.
Zum Glück wertet Java noch weitere Variablen aus. Wir machen uns die $JAVA_TOOLS_OPTIONS zu nutze:

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
Dass der Trick funktioniert hat, ist an der Zeile "Picked up JAVA_TOOL_OPTIONS:" zu erkennen.
Die Bildschirmausgabe ist nicht korrekt. Das Ergebnis in den Spool-Dateien hingegen schon.
Wir haben Windows-Zeilenabschlüsse erhalten - das ist unter Windows natürlich zu erwarten.

Um das "file"-Kommando auch in der powershell benutzen zu können, wurde hier die neue bash für Windows 10 von Microsoft eingesetzt.


Weiterführene Infos zur Lokalisierung von Java gibt es hier:
[Internationalization: Understanding Locale in the Java Platform](http://www.oracle.com/technetwork/articles/javase/locale-140624.html)
