
查询 tablespace
---
```
select tablespace_name from dba_tablespaces;
```

查询 data files
---
```
select file_name, tablespace_name ,bytes ,status from dba_data_files ;
```

监控文件系统的 I/O 比例
---
```
SELECT a.file# "#" ,
       a.name "name",
       a.status ,
       a.bytes ,
       b.phyrds ,
       b.phywrts
  FROM v$datafile a, v$filestat b
  WHERE a.file# = b .file# ;
```

Tablespace 使用量查询
---
```
select t. *
  from (SELECT D .TABLESPACE_NAME ,
               SPACE "SUM_SPACE(M)" ,
               BLOCKS SUM_BLOCKS ,
               SPACE - NVL(FREE_SPACE , 0 ) "USED_SPACE(M)",
               ROUND(( 1 - NVL(FREE_SPACE , 0 ) / SPACE) * 100, 2 ) "USED_RATE(%)" ,
               FREE_SPACE "FREE_SPACE(M)"
          FROM (SELECT TABLESPACE_NAME ,
                       ROUND(SUM(BYTES ) / ( 1024 * 1024), 2 ) SPACE,
                       SUM(BLOCKS) BLOCKS
                  FROM DBA_DATA_FILES
                 GROUP BY TABLESPACE_NAME ) D ,
               (SELECT TABLESPACE_NAME ,
                       ROUND(SUM(BYTES ) / ( 1024 * 1024), 2 ) FREE_SPACE
                  FROM DBA_FREE_SPACE
                 GROUP BY TABLESPACE_NAME ) F
         WHERE D .TABLESPACE_NAME = F .TABLESPACE_NAME (+)
        UNION ALL
        SELECT D .TABLESPACE_NAME ,
               SPACE "SUM_SPACE(M)" ,
               BLOCKS SUM_BLOCKS ,
               USED_SPACE "USED_SPACE(M)" ,
               ROUND(NVL(USED_SPACE , 0 ) / SPACE * 100, 2 ) "USED_RATE(%)" ,
               SPACE - USED_SPACE "FREE_SPACE(M)"
          FROM (SELECT TABLESPACE_NAME ,
                       ROUND(SUM(BYTES ) / ( 1024 * 1024), 2 ) SPACE,
                       SUM(BLOCKS) BLOCKS
                  FROM DBA_TEMP_FILES
                 GROUP BY TABLESPACE_NAME ) D ,
               (SELECT TABLESPACE,
                       ROUND(SUM(BLOCKS * 8192 ) / ( 1024 * 1024), 2 ) USED_SPACE
                  FROM V$SORT_USAGE
                 GROUP BY TABLESPACE) F
         WHERE D .TABLESPACE_NAME = F .TABLESPACE(+)) t
order by "USED_RATE(%)" desc;
```

监控表空间的 I/O 比例
---
```
SELECT df.tablespace_name name,
         df.file_name "FILE" ,
         f.phyrds pyr ,
         f.phyblkrd pbr ,
         f.phywrts pyw ,
         f.phyblkwrt pbw
    FROM v$filestat f , dba_data_files df
   WHERE f.file# = df.file_id
ORDER BY df.file_name;
```
  
查询数据库默认永久表空间
---
```
select * from database_properties where property_name='DEFAULT_PERMANENT_TABLESPACE';
```

查看当前 TEMP Tablespace 临时表使用空间大小与正在占用临时表空间的sql语句
---
```
select
   srt.tablespace,
   srt.segfile# ,
   srt.segblk# ,
   srt.blocks,
   a.sid,
   a.serial# ,
   a.username ,
   a.osuser ,
   a.status
from
   v$session    a,
   v$sort_usage srt
where
   a.saddr = srt .session_addr
order by
   srt .tablespace, srt .segfile# , srt .segblk# ,
   srt .blocks;


SELECT sess.SID,
         segtype ,
         blocks * 8 / 1000 "MB" ,
         sql_text
    FROM v$sort_usage sort, v$session sess , v$sql sql
   WHERE sort.SESSION_ADDR = sess.SADDR AND sql.ADDRESS = sess.SQL_ADDRESS
ORDER BY blocks DESC;
```

找出所有 jobs
---
```
select * from dba_jobs ;
```

以 job 关键字寻找 job 详细资讯
---
```
select * from dba_jobs where WHAT like '%job_to_run%';
```

找出正在执行的 JOB 编号及其会话编号
---
```
select * from dba_jobs_running;
```

@找出 JOB 的 SID, SPID
---
```
select b.*, c.spid, a.* from v$session a, dba_jobs_running b , v$process c where a.sid=b.sid and c.addr=a.paddr;
```

停止该JOB的执行
---
```
EXEC DBMS_JOB.BROKEN(&JOB,TRUE);
commit;
```

找到正在运行job的会话，然后把会话kill掉
---
```
SELECT SID,SERIAL# FROM V$SESSION WHERE SID='&SID';
ALTER SYSTEM KILL SESSION '&SID,&SERIAL';
```

将job的BROKEN状态停止
---
```
EXEC DBMS_JOB.BROKEN(job#,FALSE):
```