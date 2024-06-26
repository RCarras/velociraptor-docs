---
title: Linux.Events.TrackProcesses
hidden: true
tags: [Client Event Artifact]
---

This artifact uses auditd and pslist to keep track of running
processes using the Velociraptor process tracker.

The process tracker keeps track of exited processes, and resolves
process call chains from it in memory cache.

This event artifact enables the global process tracker and makes it
possible to run many other artifacts that depend on the process
tracker.

auditd is used to track processes, and it must be installed on the
system. If InstallAudit is enabled, the artifact will install it
using apt-get (which only works on Debian-like operating systems,
like Debian and Ubuntu).

When the artifact starts, it will insert audit rules that track
the syscalls *execve*, *exit* and *exit_group*. These rules are removed
when the event artifact is disabled. However, in case they are not
removed automatically, for instance if velociraptor were to crash,
the following commands (which must be run as root) will remove then:

```
auditctl -d exit,always -F arch=b64 -S execve -k procmon
auditctl -d exit,always -F arch=b32 -S execve -k procmon
auditctl -d exit,always -F arch=b64 -S exit,exit_group -k procmon_exit
auditctl -d exit,always -F arch=b32 -S exit,exit_group -k procmon_exit
```

Remember to replace the keys if you used keys other than the defaults.

Note that processes that are killed or do not shut down properly will
note get their exit timestamps registered.


<pre><code class="language-yaml">
name: Linux.Events.TrackProcesses
author: Andreas Misje – @misje
description: |
  This artifact uses auditd and pslist to keep track of running
  processes using the Velociraptor process tracker.

  The process tracker keeps track of exited processes, and resolves
  process call chains from it in memory cache.

  This event artifact enables the global process tracker and makes it
  possible to run many other artifacts that depend on the process
  tracker.

  auditd is used to track processes, and it must be installed on the
  system. If InstallAudit is enabled, the artifact will install it
  using apt-get (which only works on Debian-like operating systems,
  like Debian and Ubuntu).

  When the artifact starts, it will insert audit rules that track
  the syscalls *execve*, *exit* and *exit_group*. These rules are removed
  when the event artifact is disabled. However, in case they are not
  removed automatically, for instance if velociraptor were to crash,
  the following commands (which must be run as root) will remove then:

  ```
  auditctl -d exit,always -F arch=b64 -S execve -k procmon
  auditctl -d exit,always -F arch=b32 -S execve -k procmon
  auditctl -d exit,always -F arch=b64 -S exit,exit_group -k procmon_exit
  auditctl -d exit,always -F arch=b32 -S exit,exit_group -k procmon_exit
  ```

  Remember to replace the keys if you used keys other than the defaults.

  Note that processes that are killed or do not shut down properly will
  note get their exit timestamps registered.

precondition: SELECT OS From info() where OS = 'linux'

type: CLIENT_EVENT

required_permissions:
  - EXECVE

parameters:
  - name: InstallAudit
    type: bool
    default: False
    description: Run apt-get update and apt-get install to ensure that auditd is installed
  - name: AuditKeyExecve
    default: procmon
    description: Key to use for execve syscalls. Change to match an existing key if you are already using audit to track processes.
  - name: AuditKeyExit
    default: procmon_exit
    description: Key to use for exit syscalls. Change to match an existing key if you are already using audit to track processes.
  - name: RemoveRules
    type: bool
    default: True
    description: Remove rules when removing the event artifact
  - name: AlsoForwardUpdates
    type: bool
    description: Upload all tracker state updates to the server
  - name: MaxSize
    type: int64
    description: Maximum size of the in-memory process cache (default 10k)
  - name: AddEnrichments
    type: bool
    description: Calculate hashes on process binaries
  - name: AuditCtlExe
    description: Path to the auditctl binary
    default: /sbin/auditctl

sources:
  - query: |
     /* Test whether the auditctl binary exists at the expected path: */
     LET AuditInstalled = SELECT *
       FROM stat(filename=AuditCtlExe)

     LET _ &lt;= SELECT *
       FROM if(
         condition=NOT AuditInstalled
          AND InstallAudit,
         then={
           SELECT *
           FROM chain(
             a_update={
               SELECT 
                      log(
                        message='Updating package index before installing auditd',
                        level='INFO')
               FROM execve(argv=['apt-get', '-y', 'update'])
             },
             b_install={
               SELECT log(message='Installing auditd using apt-get', level='INFO')
               FROM execve(
                 argv=['apt-get', '-y', '-o', 'Debug::pkgProblemResolver=yes', '--no-install-recommends', 'install', 'auditd'])
             })
         })

     LET AuditCtl(action, syscalls, key) = SELECT *
       FROM if(
         condition=AuditInstalled,
         then={
           SELECT *
           FROM foreach(
             row=('b64', 'b32', ),
             query={
               SELECT *
               FROM execve(
                 argv=['auditctl', action, 'exit,always', '-F', 'arch=' + _value, '-S', syscalls, '-k', key])
               WHERE AuditInstalled
             })
         })

     LET _ &lt;= SELECT *
       FROM AuditCtl(action='-a',
                     syscalls='execve',
                     key=AuditKeyExecve)
     LET _ &lt;= SELECT *
       FROM AuditCtl(action='-a',
                     syscalls='exit,exit_group',
                     key=AuditKeyExit)

     LET _ &lt;= atexit(
         query={
           SELECT *
           FROM if(
             condition=RemoveRules,
             then={
               SELECT 
                      log(
                        message='Removing audit rules',
                        level='INFO')
               FROM chain(
                 r1={
                   SELECT *
                   FROM AuditCtl(action='-d',
                                 syscalls='execve',
                                 key=AuditKeyExecve)
                 },
                 r2={
                   SELECT *
                   FROM AuditCtl(action='-d',
                                 syscalls='exit,exit_group',
                                 key=AuditKeyExit)
                 })
             })
         })

     LET Users &lt;= memoize(
         query={
           SELECT 
                  Uid AS UID,
                  User
           FROM Artifact.Linux.Sys.Users()
         },
         key='UID')

     LET UpdateQuery = SELECT *
       FROM foreach(
         row={
           SELECT *
           FROM audit()
           WHERE AuditKeyExecve IN Tags OR AuditKeyExit IN Tags
         },
         query={
           SELECT *
           FROM switch(
             start={
               SELECT 
                      Process.pid AS id,
                      Process.ppid AS parent_id,
                      'start' AS update_type,
                      dict(
                        Pid=Process.pid,
                        Ppid=Process.ppid,
                        Name=Process.name,
                        StartTime=timestamp(
                          string=Timestamp),
                        EndTime=NULL,
                        Username=get(
                          item=Users,
                          field=User.ids.uid).User,
                        Exe=Process.exe,
                        CommandLine=join(
                          sep=' ',
                          array=Process.args),
                        CurrentDirectory=get(
                          item=Process,
                          member='CWD'),
                        TerminalSessionId=Session,
                        User=User,
                        Process=Process) AS data,
                      timestamp(
                        string=Timestamp) AS start_time,
                      NULL AS end_time
               FROM scope()
               WHERE AuditKeyExecve IN Tags
             },
             end={
               SELECT 
                      Process.pid AS id,
                      NULL AS parent_id,
                      'exit' AS update_type,
                      dict() AS data,
                      NULL AS start_time,
                      timestamp(
                        string=Timestamp) AS end_time
               FROM scope()
               WHERE AuditKeyExit IN Tags
             })
         })

     LET SyncQuery = SELECT 
                            Pid AS id,
                            Ppid AS parent_id,
                            CreateTime AS start_time,
                            dict(
                              Name=Name,
                              Username=Username,
                              Exe=Exe,
                              CommandLine=CommandLine) AS data
       FROM pslist()

     LET Tracker &lt;= process_tracker(
         max_size=MaxSize,
         sync_query=SyncQuery,
         update_query=UpdateQuery,
         sync_period=60000,
         enrichments=if(
           condition=AddEnrichments,
           then=['''x =&gt; dict(Hashes=hash(
                          path=x.Data.Exe))'''],
           else=[]))

     SELECT *
     FROM if(
       condition=AuditInstalled,
       then={
         SELECT *
         FROM process_tracker_updates()
         WHERE update_type = 'stats' OR AlsoForwardUpdates
       },
       else={
         SELECT 
                log(
                  message='auditd is not installed, and it is either set not to be installed or failed to install, aborting',
                  level='INFO')
         FROM scope()
       })

</code></pre>

