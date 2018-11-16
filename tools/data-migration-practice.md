---
title: Synchronize Data Using Data Migration
summary: Use the Data Migration tool to synchronize the full data and the incremental data.
category: tools
---

# Synchronize Data Using Data Migration

This guide shows how to synchronize data using the Data Migration (DM) tool.

## Step 1: Deploy the DM cluster

It is recommended to deploy the DM cluster using DM-Ansible. For detailed deployment, see [Deploy Data Migration Using DM-Ansible](../tools/data-migration-deployment.md).

> **Notes**:
> 
> - For database related passwords in the DM configuration file, use the passwords encrypted by `dmctl`. If a database password is empty, it is unnecessary to encrypt it. See [Encrypt the upstream MySQL user password using dmctl](../tools/data-migration-deployment.md#encrypt-the-upstream-mysql-user-password-using-dmctl).
> - The user of the upstream and downstream databases must have the corresponding read and write privileges.

## Step 2: Check the cluster information

After the DM cluster is deployed using DM-Ansible, the configuration information is like what is listed below.

- The configuration information of related components in the DM cluster:

    | Component | IP | Service port |
    |------| ---- | ---- |
    | dm-worker | 172.16.10.72 | 10081 |
    | dm-worker | 172.16.10.73 | 10081 |
    | dm-master | 172.16.10.71 | 11080 |

- The information of upstream and downstream database instances:

    | Database instance | IP | Port | Username | Encrypted password |
    | -------- | --- | --- | --- | --- |
    | Upstream MySQL-1 | 172.16.10.81 | 3306 | root | VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU= |
    | Upstream MySQL-2 | 172.16.10.82 | 3306 | root | VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU= |
    | Downstream TiDB | 172.16.10.83 | 4000 | root | |

- The configuration in the dm-master process configuration file `{ansible deploy}/conf/dm-master.toml`:

    ```toml
    # Master configuration.

    [[deploy]]
    mysql-instance = "172.16.10.81:3306"
    dm-worker = "172.16.10.72:10081"

    [[deploy]]
    mysql-instance = "172.16.10.82:3306"
    dm-worker = "172.16.10.73:10081"
    ```

## Step 3: Configure the data synchronization task

Assuming that you need to synchronize all the `test_table` table data in the `test_db` database of both the upstream MySQL-1 and MySQL-2 instances, to the downstream `test_table` table in the `test_db` database of TiDB, in the full data plus incremental data mode, you can edit the `task.yaml` task configuration file as follows:

```yaml
# The task name. You need to use a different name for each of the multiple tasks that
# run simultaneously.
name: "test"
# The full data plus incremental data (all) synchronization mode
task-mode: "all"
# Disables the heartbeat synchronization delay calculation
disable-heartbeat: true

# The downstream TiDB configuration information
target-database:
  host: "172.16.10.83"
  port: 4000
  user: "root"
  password: ""

# All the upstream MySQL instances that the current task needs to use
mysql-instances:
-
  config:                 # MySQL-1 configuration
    host: "172.16.10.81"
    port: 3306
    user: "root"
    # The ciphertext generated by a certain encryption of the plaintext `123456`.
    # The ciphertext generated by each encryption is different.
    password: "VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU="  
  # The instance ID of MySQL-1, corresponding to the `mysql-instance` in "dm-master.toml"
  instance-id: "172.16.10.81:3306"
  # The configuration item name of the black and white lists of the name of the
  # database/table to be synchronized, used to quote the global black and white
  # lists configuration
  black-white-list: "global"
  # The configuration item name of mydumper, used to quote the global mydumper configuration
  mydumper-config-name: "global"

-
  config:                 # MySQL-2 configuration
    host: "172.16.10.82"
    port: 3306
    user: "root"
    password: "VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU="
  instance-id: "172.16.10.82:3306"
  black-white-list: "global"
  mydumper-config-name: "global"

# The global configuration of black and white lists. Each instance can quote it by the
# configuration item name.
black-white-list:
  global:
    do-tables:                        # The upstream tables to be synchronized
    - db-name: "test_db"              # The database name of the table to be synchronized
      tbl-name: "test_table"          # The name of the table to be synchronized

# mydumper global configuration. Each instance can quote it by the configuration item name.
mydumpers:
  global:
    mydumper-path: "./bin/mydumper"   # The file path of the mydumper binary
    extra-args: "-B test_db -T test_table"  # Only dumps the "test_table" table of the "test_db" database
```

## Step 4: Start the data synchronization task

1. Come to the dmctl directory `/home/tidb/dm-ansible/resource/bin/`.

2. Run the following command to start dmctl.

    ```bash
    ./dmctl --master-addr 172.16.10.71:11080
    ```

3. Run the following command to start the data synchronization tasks.

    ```bash
    # `task.yaml` is the configuration file that is edited above.
    start-task task.yaml
    ```

    - If the above command returns the following result, it indicates the task is successfully started.

        ```json
        {
            "result": true,
            "msg": "",
            "workers": [
                {
                    "result": true,
                    "worker": "172.16.10.72:10081",
                    "msg": ""
                },
                {
                    "result": true,
                    "worker": "172.16.10.73:10081",
                    "msg": ""
                }
            ]
        }
        ```

    - If the above command returns other information, you can edit the configuration according to the prompt, and then run the `start-task task.yaml` command to restart the task.

## Step 5: Check the data synchronization task

If you need to check the task state or whether a certain data synchronization task is running in the DM cluster, run the following command in the dmctl directory:

```bash
query-status
```

## Step 6: Stop the data synchronization task

If you do not need to synchronize data any more, run the following command in the dmctl directory to stop the task:

```bash
# `test` is the task name that you set in the `name` configuration item of
# the `task.yaml` configuration file.
stop-task test
```

## Step 7: Monitor the task and check logs

Assuming that Prometheus and Grafana are successfully deployed along with the DM cluster deployment using DM-Ansible, and the Grafana address is `172.16.10.71`, you can open <http://172.16.10.71:3000> in a browser, choose the DM dashboard to check the related monitoring metrics.

While the DM cluster is running, dm-master, dm-worker, and dmctl output the monitoring metrics information through logs. The log directory of each component is as follows:

- dm-master log directory: It is specified by the `--log-file` dm-master process parameter. For DM-Ansible deployment, the log directory is `{ansible deploy}/log/dm-master.log` in the dm-master node.
- dm-worker log directory: It is specified by the `--log-file` dm-worker process parameter. For DM-Ansible deployment, the log directory is `{ansible deploy}/log/dm-worker.log` in the dm-master node.
- dmctl log directory: It is the same as the binary directory of dmctl.