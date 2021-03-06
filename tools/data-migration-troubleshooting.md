---
title: Data Migration Troubleshooting
summary: Learn how to diagnose and resolve issues when you use Data Migration.
category: tools
---

# Data Migration Troubleshooting

This document summarizes some commonly encountered errors when you use Data Migration, and provides the solutions.

If you encounter errors while running Data Migration, try the following solution:

1. Check the log content related to the error you encountered. The log files are on the dm-master and dm-worker deployment nodes. You can then view [common errors](#common-errors) to find the corresponding solution.

2. If the error you encountered is not involved yet, and you cannot solve the problem yourself by checking the log or monitoring metrics, you can contact the corresponding sales support staff.

3. After the error is solved, restart the task using dmctl.

    ```bash
    resume-task ${task name}
    ```

However, you need to reset the data synchronization task in some cases. For details about when to reset and how to reset, see [Reset the data synchronization task](#reset-the-data-synchronization-task).

## Common errors

### `Access denied for user 'root'@'172.31.43.27' (using password: YES)` shows when you query the task or check the log

For database related passwords in all the DM configuration files, use the passwords encrypted by `dmctl`. If a database password is empty, it is unnecessary to encrypt it. For how to encrypt the plaintext password, see [Encrypt the upstream MySQL user password using dmctl](../tools/data-migration-deployment.md#encrypt-the-upstream-mysql-user-password-using-dmctl).

In addition, the user of the upstream and downstream databases must have the corresponding read and write privileges. Data Migration also checks the following privileges automatically while starting the data synchronization task:

+ MySQL binlog configuration

    - Whether the binlog is enabled (DM requires that the binlog must be enabled)
    - Whether `binlog_format=ROW` (DM only supports the binlog synchronization in the ROW format
    - Whether `binlog_row_image=FULL` (DM only supports `binlog_row_image=FULL`)

+ The privileges of the upstream MySQL instance

    The MySQL user in DM configuration needs to have the following privileges at least:
    
    - REPLICATION SLAVE
    - REPLICATION CLIENT
    - RELOAD
    - SELECT

+ The compatibility of the upstream MySQL table schema

    Some differences exist between the compatibility of TiDB and that of MySQL.

    - No support for the foreign key
    - [Character set compatibility](../sql/character-set-support.md)

+ The consistency check on the upstream MySQL multiple-instance shards

    - The consistency of the table schema

        - Column name, type
        - Index
        
    - Whether the auto increment primary key that conflicts during merging exists

## Reset the data synchronization task

You need to reset the entire data synchronization task in the following cases:

- `RESET MASTER` is artificially executed in the upstream database, which causes an error in the relay log synchronization.
- The relay log or the upstream binlog is corrupted or lost.

Generally, at this time, the relay unit exits with an error and cannot be automatically restored gracefully. You need to manually restore the data synchronization and the steps are as follows:

1. Use the `stop-task` command to stop all the synchronization tasks that are currently running.
2. Use Ansible to [stop the entire DM cluster](../tools/data-migration-deployment.md#step-10-stop-the-dm-cluster).
3. Manually clean up the relay log directory of the dm-worker corresponding to the MySQL master whose binlog is reset.

    - If the cluster is deployed using DM-Ansible, the relay log is in the `<deploy_dir>/relay_log` directory.
    - If the cluster is manually deployed using the binary, the relay log is in the directory set in the `relay-dir` parameter.

4. Clean up downstream synchronized data.
5. Use Ansible to [start the entire DM cluster](../tools/data-migration-deployment.md#step-9-deploy-the-dm-cluster).
6. Restart data synchronization with the new task name, or set `remove-meta` to `true` and `task-mode` to `all`.

## Skip or replace abnormal SQL statements

If the sync unit encounters an error while executing a SQL (DDL/DML) statement, DM supports manually skipping the SQL statement using dmctl or replacing this execution with another user-specified SQL statement.

When you manually handle the SQL statement that has an error, the frequently used commands include `query-status`, `sql-skip`, and `sql-replace`.

### The manual process of handling abnormal SQL statements

1. Use `query-status` to query the current running status of the task.

    - Whether an error caused the `Paused` status of a task in a dm-worker
    - Whether the cause of the task error is an error in executing the SQL statement

2. Record the returned binlog pos (`SyncerBinlog`) that Syncer has synchronized when executing `query-status`.
3. According to the error condition, application scenario and so on, decide whether to skip or replace the current SQL statement that has an error.
4. Skip or replace the current error SQL statement that has an error:

    - To skip the current SQL statement that has an error, use `sql-skip` to specify the dm-worker, task name and binlog pos that need to perform SQL skip operations and perform the skip operations.
    - To replace the current SQL statement that has an error, use `sql-replace` to specify the dm-worker, task name, binlog pos, and the new SQL statement(s) used to replace the original SQL statement. (You can specify multiple statements by separating them using `;`.)

5. Use `resume-task` and specify the dm-worker and task name to restore the task on the dm-worker that was paused due to an error.
6. Use `query-status` to check whether the SQL statement skip or replacement is successful.

#### How to find the binlog pos that needs to be specified in the parameter?

In `dm-worker.log`, find `current pos` corresponding to the SQL statement has an error.

### Command line parameter description

#### sql-skip

- `worker`: flag parameter, string, `--worker`, required; specifies the dm-worker where the SQL statement that needs to perform the skip operation is located
- `task-name`: non-flag parameter, string, required; specifies the task where the SQL statement that needs to perform the skip operation is located
- `binlog-pos`: non-flag parameter, string, required; specifies the binlog pos where the SQL statement that needs to perform the skip operation is located; the format is `mysql-bin.000002:123` (`:` separates the binlog name and pos)

#### sql-replace

- `worker`: flag parameter, string, `--worker`, required; specifies the dm-worker where the SQL statement that needs to perform the replacement operation is located
- `task-name`: non-flag parameter, string, required; specifies the task where the SQL statement that needs to perform the replacement operation is located
- `binlog-pos`: non-flag parameter, string, required; specifies the binlog pos where the SQL statement that needs to perform the replacement operation is located; the format is `mysql-bin.000002:123` (`:` separates the binlog name and pos)
- `sqls`: non-flag parameter, string, required; specifies new SQL statements that are used to replace the original SQL statement (You can specify multiple statements by separating them using `;`)