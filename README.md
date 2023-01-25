# Project

> This repo has been populated by an initial template to help get you started. Please
> make sure to update the content to build a great experience for community-building.

As the maintainer of this project, please make a few updates:

- Improving this README.MD file to provide a great experience
- Updating SUPPORT.MD with content about this project's support experience
- Understanding the security reporting process in SECURITY.MD
- Remove this section from the README

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

## How To

What follows are some ready made scripts that can help when troubleshooting SAP on Oracle deployments. These scripts are made to be run by Cloud Architects, DBAs, escalation teams, or anyone that is responsible for troubleshooting SAP on Oracle performance issues. There are separate categories for the scripts. Please use the navigation links below:

[General Issues](#general-issues)

[IO Issues](#io-issues)

[Enqueue Issues](#enqueue-issues)

[SQL Statement Issues](#sql-statement-issues)

[Long Running Background Job Issue](#long-running-background-job-issues)

[Further thoughts](#further-thoughts)

## General Issues

[Configuration Patch History](3_Configuration_Patches_History.txt)

[DB Time History](4_DB_Time_History.txt)

[CPU Time History](6_CPU_Time_History.txt)

[DB key figures history](7_DB_Key_Figures_History.txt)

[Top segments per segment statistics](8_TopSegmentsPerSegmentStatistics.txt)

[SQL Statements that consumed the most DB server time](9_SQL_TopSQLInAWRWithSearchOptionsAndHistograms.txt)

## IO Issues

There are two reasons for more absolute IO time per hour: more IO operations or a higher average time per IO operations. It is crucial to figure out which is responsible for an absolute IO operation time increase. In the event of higher IO operations, it is imperative to focus on the statements responsible for the higher number of IOs. In the latter case, the root cause is likely outside of the database, if the database does not show higher IO activity than at times that do not have issues. Use the DB time history to try and narrow this down.

[IO activity per AWR interval](11_IO_IOActivityPerAWRInterval.txt)

[Histogram db file sequential read](12_Histogram_db_file_sequential_read.txt)

[Histogram db file scattered read](13_Histogram_db_file_scattered_read.txt)

[Histogram log file sync](14_Histogram_log_file_sync.txt)

[Histogram log file parallel write](15_Histogram_log_file_parallel_write.txt)

[Histogram disk file operations I/O](16_Histogram_Disk_file_operations_IO.txt)

[LFS Analyzer](17_LFS_Analyzer.txt)

## Enqueue Issues

se these statements if the absolute enqueue time is significantly higher than usual at the time of the performance issue. To reduce enqueue time, either the risk of contention needs to be reduces, by non-DB tuning or the root blocker activity of the block waiters needs to be tuned. In the former situation, buffering number ranges (table NRIV), or using less parallelism could help to resolve the issue. In the latter case, it depends on where the root blocker is mainly active:

DB: collect and tune that DB activity

ABAP CPU: check the ABAP for tuning potential

RFC: check with /SDF/(S)MON if the RFCs are active mainly on the database or in ABAP CPU and tune accordingly.

[Blocking locks in history](18_Locks_BlockingLocksInHistory_11g+.txt)

[Root Blocker Activity](19_Lock_Analyzer_Root_Blocker_Activity.txt)

[Blocked Statements](20_Lock_Analyzer_Blocked_Statements.txt)

[Blocked Rows](21_Lock_Analyzer_Blocked_Rows.txt)

## SQL Statement Issues

If a can technically be tuned and to what degree it can be tuned depends on a lot of factors. Some guiding questions include:

Is the effort to select the rows plausible or too high?

Where is the time lost in the access path (table or index)?

Does Oracle provide the chance to do it better (index creation)?

[SQL ID data collector (statement tuning)](22_SQL_SQL_ID_DataCollector_11g+.txt)

[SQL ID cache statements](23_SQL_ID_Cache_Snapshots.txt)

## Long Running Background Job Issues

A very often upcoming performance problem is that a Background Job shows a high runtime so it should be evaluated why. For systematic tuning, it is crucial to know where most of the time is lost, otherwise a component be tuned having just minor contribution to the overall runtime. The job time includes: Non-DB time (measured by SAP), DB time (measured by SAP), communication/network time between SAP and DB and the DB server time. The DB server time includes the SQL statement(s) run time Collect the following components:

***Communication/Network Time between SAP and DB, Non-DB time:***
Can be calculated as difference when the other times are known.

***Job Time:***
SM37 **or**
ST03 -> Transaction Profile -> Background -> Total Response Time **or**
SDF/(S)MON samples of background job * time between samples **or**
STAD -> Response Time

***DB Time:***
ST03 –> Transaction profile -> Background -> Total DB Time **or**
/SDF/(S)MON samples of job showing DB activity (columns Current Action, Table) * time between samples **or**
STAD -> DB Time

***DB Server Time:***
You will need the time-frame of the job run (SM37) and the DB session that served the job. THere is exactly once session which can be found as follows (this will only work if the SAP server was not restarted since the job run. If there was a SINGLE restart the dev_w*.old trace file can be evaluated exactly in the same way via transaction AL11).

This script is based on a 10-second active history session history samples, therefore, not all SQL statements executed by the job are shown. In any case, if a statement has a significant contributions to the DB server time, it will be sampled and be shown. Using the above information, run the following script:
```sql
    select nvl(sql_id,decode(grouping_id(sql_id),1,'DB Server Time','No Statement')) statement, count(*)*10 seconds
    from
    
      DBA_HIST_ACTIVE_SESS_HISTORY  
    
    where
    
      session_id=<session_id> and
    
      sample_time between
    
        to_timestamp('<Job Start Time YYYY-MM-DD HH24:MI:SS>','YYYY-MM-DD HH24:MI:SS') and 
    
        to_timestamp('<Job End Time YYYY-MM-DD HH24:MI:SS>','YYYY-MM-DD HH24:MI:SS') 
    
     group by rollup
    
       (sql_id)
    
     order by
    
       grouping_id(sql_id) desc,
    
       count(*) desc; 
```

## Further Thoughts

Depending on where the most time is being spent, further analysis and tuning is needed, but only in areas having the dominant or at least significant contribution. Consider the following areas:

***Non-DB Time (measured from SAP) is dominating the job time***
Check for what time was spent on the SAP layer and if this activity can be tuned. As an example, use /SDF/(S)MON for this analysis. If /SDF/(S)MON data is not available, set up /SDF/SMON according to [3007524](https://launchpad.support.sap.com/#/notes/3007524).

***Communication/Network time between SAP and DB is dominating the job time***
This can be due to either long running executions or a high volume of executions. In the former case, use tools such as ABAP Meter and niping as described in [3007524](https://launchpad.support.sap.com/#/notes/3007524). In the latter case, use the scripts for SQL statement issues.

***SQL Statement time dominates the job time***
Use the scripts for SQL statement issues.
