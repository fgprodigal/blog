---
title: DataGuard记录
description:
authors:
  - Ray
tags:
categories:
  - 数据库
series:
date: "2017-11-06"
featuredImage: featured-image.webp
summary: 记录 DataGuard 搭建过程
---

### Primary 开启归档

```sql
select force_logging from v$database;

alter database force logging;
alter database archivelog;
```

### 创建 standby 日志

```sql
select group#,type,member from v$logfile order by group#;

-- redo比online多一组
ALTER DATABASE ADD STANDBY LOGFILE GROUP 11 ('/u01/app/oracle/oradata/orcl/standby01.log') size 50M;
```

### 参数文件:

```sql
create pfile='/initxxx.ora' from spfile;

--standby
create spfile from pfile='/initxxx.ora';
```

> Primary 参数:

```sql
alter database archivelog;
alter database force logging;
alter database open;
alter system set log_archive_config = 'DG_CONFIG=(pri,sty)' scope=spfile;
alter system set log_archive_dest_1 = 'LOCATION=/DBBackup/Archive VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=pri' scope=spfile;
alter system set log_archive_dest_2 = 'SERVICE=sty LGWR SYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=sty' scope=spfile;
alter system set log_archive_dest_state_1 = ENABLE;
alter system set log_archive_dest_state_2 = ENABLE;
alter system set fal_server=sty scope=spfile;
alter system set fal_client=pri scope=spfile;
alter system set standby_file_management=AUTO scope=spfile;
alter system set log_file_name_convert='/u01/app/oracle/oradata/orcl','/u01/app/oracle/oradata/dave';
alter system set db_file_name_convert='/u01/app/oracle/oradata/orcl','/u01/app/oracle/oradata/dave';
```

> Standby 参数:

```sql
alter system set db_unique_name=sty scope=spfile;
alter system set log_archive_config= 'DG_CONFIG=(pri,sty)' scope=spfile;
alter system set log_archive_dest_1 = 'LOCATION=/DBBackup/Archive VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=sty' scope=spfile;
alter system set log_archive_dest_2 = 'SERVICE=pri LGWR SYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=pri' scope=spfile;
alter system set fal_server=pri scope=spfile;
alter system set fal_client=sty scope=spfile;
alter system set standby_file_management=AUTO scope=spfile;
```

### Primary 生成密码文件:

```console
$ orapwd file=?/dbs/orapwxxx password=xxx entries=10 force=y ignorecase=Y
$ scp orapwwoo db02:/u01/product/11.2.4/dbhome_1/dbs/
```

### 配置 Primary 和 Standby 监听及 tnsnames.ora。

```
ADR_BASE_LISTENER = /u01/app/oracle

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl)
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1)
      (SID_NAME = orcl)
    )
  )
```

```
sty =
  (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = sty)
     )
  )
```

### 创建备库目录:

```console
# audit_file_dest
$ mkdir -p /u01/app/oracle/admin/xxx/adump
# db_recovery_file_dest
$ mkdir -p /u01/app/oracle/fast_recovery_area
# db_file
$ mkdir -p /u01/oradata/xxx
$ mkdir -p /u01/app/oracle/fast_recovery_area/xxx
```

### 通过 RMAN 备份 Primary:

```console
$ rman target sys/oracle@pri auxiliary sys/oracle@sty nocatalog
RMAN> duplicate target database for standby from active database dorecover nofilenamecheck;
```

### 后续工作:

```sql
show parameter spfile;
```

### 日常维护:

> 启动日志应用

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```

> 停止恢复

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```

> 确认归档目的地有效性

```sql
select DEST_ID, STATUS, DESTINATION, ERROR from V$ARCHIVE_DEST where DEST_ID<=2;
```

> 查看日志应用状态

```sql
select SEQUENCE#, FIRST_TIME, NEXT_TIME, APPLIED, ARCHIVED from V$ARCHIVED_LOG order by FIRST_TIME;
```

> 进程活动状态

```sql
select PROCESS,CLIENT_PROCESS,SEQUENCE#,STATUS from V$MANAGED_STANDBY;
```

> 重做日志缺口

```sql
select STATUS, GAP_STATUS from V$ARCHIVE_DEST_STATUS where DEST_ID = 2;
```

> 状态日志表

```sql
select * from V$DATAGUARD_STATUS order by TIMESTAMP;
```

> Primary 与 Standby 切换

```sql
select switchover_status,database_role from v$database;

alter system switch logfile;
alter system archive log current;
alter database commit to switchover to physical standby;
-- alter database commit to switchover to physical standby with session shutdown; --session active
alter database commit to switchover to primary;
```
