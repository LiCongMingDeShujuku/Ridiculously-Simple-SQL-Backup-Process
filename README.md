![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 极其简单的SQL备份过程
#### Ridiculously Simple SQL Backup Process
**发布-日期: 2015年07月29日 (评论)**

![#](images/##############?raw=true "#")

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
以下是我传统上将其编写为SQL数据库服务器的备份过程。你可以使用本地驱动器（我在此示例中执行F：\ SQLBACKUPS）或通用网络备份共享驱动器（首选）。我会事先给你所有的SQL逻辑。它的作用的解释如下。你不必先阅读我博客里的所有信息，同时还要充分利用有价值的信息。我不打算使用Microsoft MVP，也不是试图给uber-sql社区留下深刻印象，而且我不是那些厌恶SQL博客的微软shills之一。我确信这会惹恼大多数核心微软数据库工程师。


## English
Here’s what I traditionally write up as the backup process for SQL Database Servers. You can use a local drive (as I’m doing in this example F:\SQLBACKUPS), or a universal network backup shared drive (preferred). I’ll give you all the SQL logic up front since that’s what you’re here for. The explanations of what it does is below. I’m not the blogger that forces you to read all my drivel first while peppering in the valuable information throughout. I’m not going for Microsoft MVP, not trying to impress the uber-sql community and I’m NOT one of those detested SQL blogging Microsoft shills. I’m sure this just pissed off most of the hardcore Microsoft Database Jockeys. Down with the Cronyism.



Anyway; lets get on with it.
话不多说，让我们继续吧。

We’ll create 2 Jobs:
我们将创建2个作业：

1. DATABASE BACKUP FULL – All Databases
2. DATABASE BACKUP LOG – All Databases
Note: I added an extra ridiculously simple Maintenance Job at the bottom if you’re interested.
注意：如果你有兴趣，我在底部添加了一个额外的简单维护作业。

Lets start with the first Full Database Backup Job.
让我们从第一个完整数据库备份作业开始。

DATABASE BACKUP FULL – All Databases
Step 1: Backup all databases.
~ Paste in the following logic…
第1步：备份所有数据库。
〜粘贴以下逻辑：


---
## Logic
```SQL
use master;
set nocount on
 
declare @backup_all_databases               varchar(max)
declare @get_time                                         varchar(25)
declare @get_day                                           varchar(25)
declare @get_date                                         varchar(25)
declare @get_month                                     varchar(25)
declare @get_year                                         varchar(25)
declare @get_timestamp                            varchar(255)
set          @get_day                                           = (select datename(dw, getdate()))
set          @get_date                                         = (select datename(dd, getdate()))
set          @get_month                                     = (select datename(mm, getdate()))
set          @get_year                                         = (select datename(yy, getdate()))
set          @get_time                                         = (select replace(replace(replace(replace(convert(char(20), getdate(), 22), '/', '-'), 'AM', 'am'), 'PM', 'pm'), ':', '-'))
set          @get_timestamp                            = (select @get_time + ' ' + @get_month + ' ' + @get_day + ' ' + @get_date + ' ' + @get_year + ' Full Database Bu ')
set          @backup_all_databases =           ''
select    @backup_all_databases =           @backup_all_databases +
'
                if exists 
                (
                select 1 
                command from master.sys.dm_exec_requests where
                command in (''backup database'', ''backup log'', ''restore database'') 
                and db_name(database_id) = ''' + upper(name) + '''
                )
                                begin
                                                print ''Database: [' + upper(name) + '] Has a backup or restore operation currently running.  Backup will be skipped.''
                                end
                                else
                                                backup database [' + upper(name) + '] to disk = ''F:\SQLBACKUPS\' + @get_timestamp + upper(name) + '.bak'' with format;
' + char(10)
from
                sys.databases sd join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where
                name    not in ('tempdb')
                and        state_desc = 'online'
                and        sdm.mirroring_role_desc is null
                or            sdm.mirroring_role_desc != 'mirror'
order by
                name asc
 
exec      master..sp_configure 'show advanced options', 1             reconfigure;
exec      master..sp_configure 'backup compression default', 1   reconfigure;
exec      master..sp_configure 'xp_cmdshell', 1                                   reconfigure;
exec      (@backup_all_databases)


```

Step 2: Delete old backups (2 weeks old).
~ Paste in the following logic…
第2步：删除旧备份（2周时长）。
〜粘贴以下逻辑：

```SQL
use master;
set nocount on
 
declare @delete_old_files          varchar(max)
declare @retention                        datetime
set          @retention                        = (select getdate() - 14) --> 14 Days
set          @delete_old_files          = ''
select    @delete_old_files          = @delete_old_files + 'exec master..xp_cmdshell ''del "' + bmf.physical_device_name + '"'';' + char(10)
from      msdb..backupset bs join msdb..backupmediafamily bmf on bs.media_set_id = bmf.media_set_id
where   bs.type in ('D', 'I', 'L' ) and bs.backup_finish_date < @retention
order by               bs.backup_finish_date desc
exec                      (@delete_old_files)
```

…on to the second Transaction Backup Job.
到第二个事务备份作业。

DATABASE BACKUP LOG – All Databases
Step 1: Backup all database transaction logs.
~ Paste in the following logic…

步骤1：备份所有数据库事务日志。 
〜粘贴以下逻辑：

```SQL
use master;
set nocount on
 
declare                 @backup_all_databases               varchar(max)
declare                 @get_time                                         varchar(25)
declare                 @get_day                                           varchar(25)
declare                 @get_date                                         varchar(25)
declare                 @get_month                                     varchar(25)
declare                 @get_year                                         varchar(25)
declare                 @get_timestamp                            varchar(255)
set                          @get_day                                           = (select datename(dw, getdate()))
set                          @get_date                                         = (select datename(dd, getdate()))
set                          @get_month                                     = (select datename(mm, getdate()))
set                          @get_year                                         = (select datename(yy, getdate()))
set                          @get_time                                         = (select replace(replace(replace(replace(convert(char(20), getdate(), 22), '/', '-'), 'AM', 'am'), 'PM', 'pm'), ':', '-'))
set                          @get_timestamp                            = (select @get_time + ' ' + @get_month + ' ' + @get_day + ' ' + @get_date + ' ' + @get_year + ' Transaction Log Bu ')
set                          @backup_all_databases =           ''
select                    @backup_all_databases =           @backup_all_databases +
'
                if exists 
                (
                select 1 
                command from master.sys.dm_exec_requests where
                command in (''backup database'', ''backup log'', ''restore database'') 
                and db_name(database_id) = ''' + upper(name) + '''
                )
                                begin
                                                print ''Database: [' + upper(name) + '] Has a backup or restore operation currently running.  Backup will be skipped.''
                                end
                                else
                                                backup log [' + upper(name) + '] to disk = ''F:\SQLBACKUPS\' + @get_timestamp + upper(name) + '.trn'' with format;
' + char(10)
from
                sys.databases sd join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where
                name    not in ('tempdb')
                and        recovery_model_desc = 'full'
                and        state_desc = 'online'
                and        sdm.mirroring_role_desc is null
                or            sdm.mirroring_role_desc != 'mirror'
order by
                name asc
 
exec      master..sp_configure 'show advanced options', 1             reconfigure;
exec      master..sp_configure 'backup compression default', 1   reconfigure;
exec      master..sp_configure 'xp_cmdshell', 1                                   reconfigure;
exec      (@backup_all_databases)

```

Step 2: Shrink transaction logs after backup.
~ Paste in the following logic…

第2步：备份后压缩事务日志。
〜粘贴以下逻辑：


```SQL
use master;
set nocount on
 
declare @shrink_logs     varchar(max)
set          @shrink_logs     = ''
select    @shrink_logs     = @shrink_logs +
'use [' + sd.name + '];' + char(10) +
'dbcc shrinkfile (' + cast(smf.file_id as varchar(3)) + ');' + char(10) + char(10)
from
                sys.databases sd join sys.master_files smf on sd.database_id = smf.database_id
                join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where
                smf.type_desc = 'log'
                and sd.recovery_model_desc = 'full'
                and sd.name not in ('tempdb')
                and        sdm.mirroring_role_desc is null
                or            sdm.mirroring_role_desc != 'mirror'
order by
                sd.name
,               smf.file_id asc
 
exec      (@shrink_logs)

```

Step 3: Delete old backup files ( 2 weeks ).
~ Paste in the following logic…
第3步：删除旧备份文件（2周时长）。
〜粘贴以下逻辑：


```SQL
use master;
set nocount on
 
declare @delete_old_files          varchar(max)
declare @retention                        datetime
set          @retention                        = (select getdate() - 14) --> 14 Days
set          @delete_old_files          = ''
select    @delete_old_files          = @delete_old_files + 'exec master..xp_cmdshell ''del "' + bmf.physical_device_name + '"'';' + char(10)
from      msdb..backupset bs join msdb..backupmediafamily bmf on bs.media_set_id = bmf.media_set_id
where   bs.type in ('D', 'I', 'L' ) and bs.backup_finish_date < @retention
order by               bs.backup_finish_date desc
exec      (@delete_old_files)


```

I’ll go ahead and add a 3rd basic DBCC Maintenance Job here.
我将继续在这里添加第三个基本的DBCC维护工作。

DATABASE MAINTENANCE – All Databases
Step 1: Run maintenance on all databases.
~ Paste in the following logic…
步骤1：对所有数据库运行维护。
〜粘贴以下逻辑：

```SQL
use master;
set nocount on
 
declare @run_maintenance                       varchar(max)
set          @run_maintenance                       = ''
select    @run_maintenance                       = @run_maintenance +
'dbcc checkdb(''' + upper(name) + ''') with no_infomsgs;' + char(10)
from     
                sys.databases sd join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where
                name not in ('tempdb')
                and        sdm.mirroring_role_desc is null
                or            sdm.mirroring_role_desc != 'mirror'
exec      (@run_maintenance)

```

About the Full and Transaction log backup jobs. This is what they are doing.

The full database backup job will do the following…

1. Get all database that are Online.
2. Get all databases excluding the ‘TempDB’
3. Get all database that are not the secondary ‘mirror’ partner in a database mirroring configuration. Remember there are 2 types of databases in a mirror. Principal (primary) and the Mirror(secondary). You want to focus on the live database.
4. Get all databases that does not have a Backup or Restore operation currently running against it.



关于完整和事务日志备份作业。这就是他们正在做的事情：

完整数据库备份作业将执行以下操作：

1. 获取所有在线数据库。
2. 获取除“TempDB”之外的所有数据库
3. 在数据库镜像配置中获取不是辅助“镜像”伙伴的所有数据库。请记住，镜像中有两种类型的数据库。本体（主要）和镜子（次要）。你希望专注于实时数据库。
4. 获取当前正在运行的没有备份或还原操作的所有数据库。




[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

