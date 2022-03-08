---
title: 电子病历迁移
description:
authors:
  - Ray
tags:
categories:
  - 数据库
series:
date: "2018-10-19"
featuredImage: featured-image.webp
summary: 记录电子病历迁移过程
---

## 目的

传统的传输表空间方式要求数据第一次由远端到目标端传输时，表空间必须置于 read only 模式，从而生产不可用。而 XTTS 方式则只需要在最后一次增量备份时将表空间置于 read only 模式，显著的减少了停机的时间

> XTTS can significantly reduce the amount of downtime required to move data between platforms using enhanced RMAN‘s bility

### oracle 建议使用场景

> from a big endian platform to linux: XTTS
> from a little endian platform to Linux: DATAGUARD

### 平台、数据库版本要求

> DATABASE:
> source 端：Oracle Database - Enterprise Edition - Version 10.2.0.1 to 12.1.0.2
> dest 端：如果使用 dbms_file_transfer（DFT），必须是 11.2.0.4 以上
> 如果是 Recovery Manager （RMAN），版本低于 11.2.0.4 时需要安装 11.2.0.4 的 RDBMS 运行 11.2.0.4 的实例

> OS:
> source 端: any platform provided the prerequisites：cannot be Windows
> dest 端: only 64-bit Oracle Linux or RedHat Linux certified

### 常见平台字节

```sql
SQL> COLUMN PLATFORM_NAME FORMAT A36
SQL> SELECT * FROM V$TRANSPORTABLE_PLATFORM ORDER BY PLATFORM_NAME;

PLATFORM_ID PLATFORM_NAME                        ENDIAN_FORMAT
----------- ------------------------------------ --------------
      6 AIX-Based Systems (64-bit)           Big
     16 Apple Mac OS                         Big
     19 HP IA Open VMS                       Little
     15 HP Open VMS                          Little
      5 HP Tru64 UNIX                        Little
      3 HP-UX (64-bit)                       Big
      4 HP-UX IA (64-bit)                    Big
     18 IBM Power Based Linux                Big
      9 IBM zSeries Based Linux              Big
     10 Linux IA (32-bit)                    Little
     11 Linux IA (64-bit)                    Little

PLATFORM_ID PLATFORM_NAME                        ENDIAN_FORMAT
----------- ------------------------------------ --------------
     13 Linux x86 64-bit                     Little
      7 Microsoft Windows IA (32-bit)        Little
      8 Microsoft Windows IA (64-bit)        Little
     12 Microsoft Windows x86 64-bit         Little
     17 Solaris Operating System (x86)       Little
     20 Solaris Operating System (x86-64)    Little
      1 Solaris[tm] OE (32-bit)              Big
      2 Solaris[tm] OE (64-bit)              Big

19 rows selected.
```

## 前提

source 端和 dest 端的要使用兼容性数据库字符集及国家语言字符集

> source 端：
>
> ```sql
> SQL@source> select * from nls_database_parameters where parameter='NLS_CHARACTERSET' or parameter='NLS_LANGUAGE';
>
> PARAMETER            VALUE
> -------------------- --------------------
> NLS_LANGUAGE         AMERICAN
> NLS_CHARACTERSET     ZHS16GBK
> ```
>
> dest 端：
>
> ```sql
> SQL@dest> select * from nls_database_parameters where parameter='NLS_CHARACTERSET' or parameter='NLS_LANGUAGE';
>
> PARAMETER                      VALUE
> ----------------------------------------------------------
> NLS_LANGUAGE                   AMERICAN
> NLS_CHARACTERSET               ZHS16GBK
> ```

传输的表空间必须自包涵，诸如物化视图，分区表，索引要特别注意检查

```sql
SQL@source> execute dbms_tts.transport_set_check('ZEMR,ZEMRT,ZEMR_IDX,ZEMRT_IDX,XJCA_TABLESPACE,ISIGNATURESERVER_DATA,ISIGNATURESERVER_LOG,ISIGNATURESERVER_INDEX,ZEMR_EMR_CONTENT_2006,ZEMR_EMR_CONTENT_2007,ZEMR_EMR_CONTENT_2008,ZEMR_EMR_CONTENT_2009,ZEMR_EMR_CONTENT_2010,ZEMR_EMR_CONTENT_2011,ZEMR_EMR_CONTENT_2012,ZEMR_EMR_CONTENT_2013,ZEMR_EMR_CONTENT_2014,ZEMR_EMR_CONTENT_2015,ZEMR_CONTENT_2016,ZEMR_CONTENT_2017,ZEMR_CONTENT_2018',true);

SQL@source> select * from transport_set_violations;
no rows selected
```

## XTSS 操作步骤

使用 dbms_file_transfer 方式

### 1. 初始化

#### step 1:source 创建 directory：sourcedir，路径使用当前数据文件使用的路径

```sql
SQL@source> create or replace directory sourcedir as '/u01/app/oracle/oradata/rmyyzemr/';
```

#### step 2:dest 创建 directory：destdir，路径使用当前数据文件使用的路径

```sql
SQL@dest> create or replace directory destdir as '/u01/app/oracle/oradata/rmyyzemr/';
```

#### step 3：创建 dest 端到 source 端的 dblink

```sql
SQL@dest> create public database link ttslink connect to system identified by **** using '(DESCRIPTION =(ADDRESS_LIST =(ADDRESS =(PROTOCOL = TCP)(HOST =192.168.1.71)(PORT = 1521)) )(CONNECT_DATA = (SERVICE_NAME = rmyyzemr )) )';

SQL@dest> select * from v$version@ttslink;
BANNER
--------------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
PL/SQL Release 11.2.0.3.0 - Production
CORE    11.2.0.3.0      Production
TNS for IBM/AIX RISC System/6000: Version 11.2.0.3.0 - Production
NLSRTL Version 11.2.0.3.0 - Production
```

#### step 4：source 端和 dest 端创建 migration 需要使用的目录

```console
[oracle@source]$ mkdir -p /u01/rman-xttconvert
[oracle@dest]$ mkdir -p /u01/rman-xttconvert
```

在 source 端解压 XTTS 使用的脚本，同时将解压后的文件传到 dest 端

```console
[oracle@source]$ pwd
/u01
[oracle@source]$ unzip rman-xttconvert_2.0.zip -d rman-xttconvert
Archive:  rman-xttconvert_2.0.zip
inflating: rman-xttconvert/xttcnvrtbkupdest.sql
inflating: rman-xttconvert/xttdbopen.sql
inflating: rman-xttconvert/xttdriver.pl
inflating: rman-xttconvert/xttprep.tmpl
inflating: rman-xttconvert/xtt.properties
inflating: rman-xttconvert/xttstartupnomount.sql
[oracle@source]$ scp -r rman-xttconvert 192.168.1.180:/u01
```

设置环境变量

```console
[oracle@source]$ export TMPDIR=/u01/rman-xttconvert
[oracle@dest]$ export TMPDIR=/u01/rman-xttconvert
```

编辑 xtt.properties

```console
[oracle@source]$ vi xtt.properties
[oracle@source]$ scp xtt.properties 192.168.1.180:/u01/rman-xttconvert
```

### 2.准备阶段

```console
[oracle@source]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -S
[oracle@source]$ scp xttnewdatafiles.txt getfile.sql 192.168.1.180:/u01/rman-xttconvert
```

### 3.目标端开始抽取数据

```console
[oracle@dest]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -G
```

### 4.源端第一次增量备份

```console
[oracle@source]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -i
[oracle@source]$ scp xttplan.txt tsbkupmap.txt 192.168.1.180:/u01/rman-xttconvert
```

### 5.目标端应用增量备份

```console
[oracle@dest]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -r
```

### 6.源端推进 scn

```console
[oracle@source]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -s
```

### 7.源端最后一次增量备份

先将所有需传输的表空间修改为只读

```sql
SQL@source> ALTER TABLESPACE ZEMR READ ONLY;
SQL@source> ALTER TABLESPACE ZEMRT READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_IDX READ ONLY;
SQL@source> ALTER TABLESPACE ZEMRT_IDX READ ONLY;
SQL@source> ALTER TABLESPACE XJCA_TABLESPACE READ ONLY;
SQL@source> ALTER TABLESPACE ISIGNATURESERVER_DATA READ ONLY;
SQL@source> ALTER TABLESPACE ISIGNATURESERVER_LOG READ ONLY;
SQL@source> ALTER TABLESPACE ISIGNATURESERVER_INDEX READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2006 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2007 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2008 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2009 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2010 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2011 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2012 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2013 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2014 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_EMR_CONTENT_2015 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_CONTENT_2016 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_CONTENT_2017 READ ONLY;
SQL@source> ALTER TABLESPACE ZEMR_CONTENT_2018 READ ONLY;
```

```console
[oracle@source]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -i
[oracle@source]$ scp xttplan.txt tsbkupmap.txt 192.168.1.180:/u01/rman-xttconvert
```

### 8.目标端最后一次应用增量备份

```console
[oracle@dest]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -r
```

### 9.目标端数据库创建数据库用户

```sql
SQL@dest> CREATE USER ZEMR IDENTIFIED BY VALUES 'S:0F179388CE209C1F80D631E8B835DC25B997CEDC00AC3CE2E3B8EEEC4916;62535043B5AD5628';
SQL@dest> CREATE USER XJCAAUTH IDENTIFIED BY VALUES 'S:78667D3EFB0C05CD004C8867983446D3C9E9FFA753ABA585C0D7C7CED689;79E27982953ED7D1';
SQL@dest> CREATE USER GGSADMIN IDENTIFIED BY VALUES 'S:B5B383610EB30B0D7D7BD54B44EB3DF4ECCE75ACA0B52A82588954FD42A6;2942F3992588BDB4';
SQL@dest> CREATE USER ZABBIX IDENTIFIED BY VALUES 'S:62EA44321560A0AF5BC58274C399E8C819BC432E1237D257FF2A3EA9996B;9A31F4B8D0743A01';
SQL@dest> CREATE USER KINGGRID IDENTIFIED BY VALUES 'S:796496A0586D2B3AD15CE9B38D1D70F324BC1D4D9542235B14577DA24DFD;CCE9EA3CCF642DD1';
SQL@dest> CREATE USER DSG IDENTIFIED BY VALUES 'S:FEC505CC032D4EB0DC81DDDD3E8389A90003A37D23A2B2BAA203B4BA8D07;4360B1302DA4165F';
SQL@dest> CREATE USER XHLIS IDENTIFIED BY VALUES 'S:9CA74C2497FD002518D79754F54E2B507E15B5CA04DB4E0740F57EADE6D8;C202B88557552391';
SQL@dest> CREATE USER ZEMRNIS IDENTIFIED BY VALUES 'S:3045F8A7BFCF250CBA872D1F27E529703AAA04E0002BF34090D0917D4B01;D7C1F52A5EFCDAED';

SQL@dest> GRANT DBA TO ZEMR;
SQL@dest> GRANT CONNECT TO XJCAAUTH;
SQL@dest> GRANT CONNECT TO GGSADMIN;
SQL@dest> GRANT RESOURCE TO GGSADMIN;
SQL@dest> GRANT DBA TO GGSADMIN;
SQL@dest> GRANT CONNECT TO ZABBIX;
SQL@dest> GRANT CONNECT TO KINGGRID;
SQL@dest> GRANT RESOURCE TO KINGGRID;
SQL@dest> GRANT CONNECT TO DSG;
SQL@dest> GRANT DBA TO DSG;
SQL@dest> GRANT CONNECT TO XHLIS;
SQL@dest> GRANT CONNECT TO ZEMRNIS;
```

### 10.目标端元数据的恢复

```console
[oracle@dest]$ $ORACLE_HOME/perl/bin/perl xttdriver.pl -e
[oracle@dest]$ cat xttplugin.txt
[oracle@dest]$ impdp directory=DATA_PUMP_DIR logfile=tts_imp.log \
network_link=ttslink transport_full_check=no \
transport_tablespaces=ZEMRT,ZEMRT_IDX,XJCA_TABLESPACE,ISIGNATURESERVER_DATA,ISIGNATURESERVER_LOG,ISIGNATURESERVER_INDEX,ZEMR,ZEMR_EMR_CONTENT_2006,ZEMR_EMR_CONTENT_2007,ZEMR_EMR_CONTENT_2008,ZEMR_EMR_CONTENT_2009,ZEMR_CONTENT_2016,ZEMR_CONTENT_2017,ZEMR_CONTENT_2018,ZEMR_EMR_CONTENT_2010,ZEMR_EMR_CONTENT_2011,ZEMR_EMR_CONTENT_2012,ZEMR_IDX,ZEMR_EMR_CONTENT_2013,ZEMR_EMR_CONTENT_2014,ZEMR_EMR_CONTENT_2015 \
transport_datafiles='/u01/app/oracle/oradata/rmyyzemr/ZEMRT02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMRT01.ORA','/u01/app/oracle/oradata/rmyyzemr/zemrt03.ora','/u01/app/oracle/oradata/rmyyzemr/zemr04.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt05.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt06.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt07.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt08.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt09.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt10.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt11.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt12.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt14.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt15.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt16.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt17.ora','/u01/app/oracle/oradata/rmyyzemr/zemrt13.ora','/u01/app/oracle/oradata/rmyyzemr/ZEMRT_IDX02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMRT_IDX01.ORA','/u01/app/oracle/oradata/rmyyzemr/xjca01.ora','/u01/app/oracle/oradata/rmyyzemr/ISIGNATURESERVER_DATA.DBF','/u01/app/oracle/oradata/rmyyzemr/ISIGNATURESERVER_LOG.DBF','/u01/app/oracle/oradata/rmyyzemr/ISIGNATURESERVER_INDEX.DBF','/u01/app/oracle/oradata/rmyyzemr/ZEMR01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2006_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2006_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2007_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2007_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2008_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2008_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2008_03.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2009_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2009_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_03.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_04.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_05.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2016_06.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_03.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_04.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_05.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2017_06.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2018_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2018_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2018_03.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2018_04.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2010_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2010_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2011_02.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_CONTENT_2011_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_EMR_CONTENT_2012_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_EMR_CONTENT_2012_02','/u01/app/oracle/oradata/rmyyzemr/ZEMR_IDX_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_EMR_CONTENT_2013_01.ORA','/u01/app/oracle/oradata/rmyyzemr/ZEMR_EMR_CONTENT_2013_02.ORA','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2014_01.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2014_02.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2014_03.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2015_01.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2015_02.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2015_03.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2015_04.ora','/u01/app/oracle/oradata/rmyyzemr/zemr_emr_content_2015_05.ora'
```

```sql
SQL@dest> ALTER TABLESPACE ZEMR READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMRT READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_IDX READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMRT_IDX READ WRITE;
SQL@dest> ALTER TABLESPACE XJCA_TABLESPACE READ WRITE;
SQL@dest> ALTER TABLESPACE ISIGNATURESERVER_DATA READ WRITE;
SQL@dest> ALTER TABLESPACE ISIGNATURESERVER_LOG READ WRITE;
SQL@dest> ALTER TABLESPACE ISIGNATURESERVER_INDEX READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2006 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2007 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2008 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2009 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2010 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2011 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2012 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2013 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2014 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_EMR_CONTENT_2015 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_CONTENT_2016 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_CONTENT_2017 READ WRITE;
SQL@dest> ALTER TABLESPACE ZEMR_CONTENT_2018 READ WRITE;
```

### 11.元数据有三个表创建失败，使用 expdp 重新导出导入

```
SQL@source>CREATE DIRECTORY REMOTE AS '/mnt/oradata/full/';
[oracle@source]$ expdp system/***** directory=REMOTEDIR  tables=ZEMR.ZEMR_NURSE_PATIENT_INFO,ZEMR.ZEMR_SPEC_EMR_CONFIG,ZEMR.ZKB_LOGIC_MODEL_SOURCE,ZEMR.QUEST_SL_TEMP_EXPLAIN1 parallel=4 job_name=expemr1 dumpfile=expdata.DMP logfile=expdp.log

SQL@dest>CREATE DIRECTORY REMOTE AS '/u01/share/full/';
[oracle@dest]$ impdp zemr/swxp4101886 directory=REMOTEDIR  dumpfile=expdata.DMP parallel=4 logfile=impdp.log  table_exists_action=replace  job_name=impemr1
```

### 12.验证传输的数据

```sql
RMAN@dest> VALIDATE TABLESPACE ZEMR CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMRT CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_IDX CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMRT_IDX CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE XJCA_TABLESPACE CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ISIGNATURESERVER_DATA CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ISIGNATURESERVER_LOG CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ISIGNATURESERVER_INDEX CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2006 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2007 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2008 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2009 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2010 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2011 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2012 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2013 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2014 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_EMR_CONTENT_2015 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_CONTENT_2016 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_CONTENT_2017 CHECK LOGICAL;
RMAN@dest> VALIDATE TABLESPACE ZEMR_CONTENT_2018 CHECK LOGICAL;
```

### 13.更改用户默认表空间

```sql
SQL@dest> alter user KINGGRID default tablespace ISIGNATURESERVER_DATA;
SQL@dest> alter user XJCAAUTH default tablespace XJCA_TABLESPACE;
SQL@dest> alter user ZEMR default tablespace ZEMRT;

SQL@dest> alter user KINGGRID quota unlimited on ISIGNATURESERVER_DATA;
SQL@dest> alter user XJCAAUTH quota unlimited on XJCA_TABLESPACE;
SQL@dest> alter user ZEMR quota unlimited on ZEMRT;
```

### 14.用户权限

```sql
SQL@dest> grant SELECT on ZEMR.ITF_PAT_CHARACTER_IP to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.ITF_PAT_CHARACTER_IP to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.ITF_PAT_DIAGNOSIS_IP to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.ITF_PAT_DIAGNOSIS_IP to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.V_NIS_BASICINFO to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.V_NIS_BASICINFO to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.V_NIS_DIAGNOSIS to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.V_NIS_DIAGNOSIS to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.V_NIS_EMR to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.V_NIS_EMR to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.V_NIS_TEMPERATURE to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.V_NIS_TEMPERATURE to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.V_NIS_WEIGHT to ZEMRNIS with grant option;
SQL@dest> grant SELECT on ZEMR.V_NIS_WEIGHT to ZEMRNIS;
SQL@dest> grant SELECT on ZEMR.ZEMR_TEMPERATURE_POINT_RECORD to XHLIS with grant option;
SQL@dest> grant SELECT on ZEMR.ZEMR_TEMPERATURE_POINT_RECORD to XHLIS;

SQL@dest> grant SELECT on SYS.DBA_AUDIT_SESSION to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_DATA_FILES to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_FREE_SPACE to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_OBJECTS to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_REGISTRY to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_TABLESPACES to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_TEMP_FILES to ZABBIX;
SQL@dest> grant SELECT on SYS.DBA_USERS to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$ACTIVE_SESSION_HISTORY to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$BGPROCESS to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$INSTANCE to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$LATCH to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$LIBRARYCACHE to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$LOCK to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$LOG to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$LOG_HISTORY to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$PARAMETER to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$PGASTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$PROCESS to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$SESSION to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$SGASTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$SYSSTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$SYSTEM_EVENT to ZABBIX;
SQL@dest> grant SELECT on SYS.GV_$TABLESPACE to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$ACTIVE_SESSION_HISTORY to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$ARCHIVED_LOG to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$BGPROCESS to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$DATABASE_INCARNATION to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$INSTANCE to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$LATCH to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$LIBRARYCACHE to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$LOCK to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$LOG to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$LOG_HISTORY to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$PARAMETER to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$PGASTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$PROCESS to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$SESSION to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$SGASTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$SYSSTAT to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$SYSTEM_EVENT to ZABBIX;
SQL@dest> grant SELECT on SYS.V_$TABLESPACE to ZABBIX;
```
