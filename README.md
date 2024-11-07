# MariaDB Master-Slave Replication Setup

This guide provides step-by-step instructions to set up **MariaDB Master-Slave Replication** on two servers. This setup enables the replication of database changes from a **master** to a **slave** server.

## Prerequisites
- **MariaDB installed** on both the master and slave servers.
- **Static IP addresses** or network accessibility between the master and slave servers.
- **SSH access** to both servers.

For this setup:
- **Master IP**: `192.168.1.100`
- **Slave IP**: `192.168.1.101`

## 1. Configure the Master Server

1. **Edit the MariaDB configuration file** on the master (`/etc/my.cnf` or `/etc/mysql/my.cnf`):
    ```ini
    [mysqld]
    server-id=1             # Unique ID for the master (must be unique in the replication setup)
    log_bin=binlog          # Enable binary logging
    binlog_format=row       # Use row-based replication (recommended for consistency)
    ```

2. **Restart MariaDB** on the master to apply the changes:
    ```bash
    sudo systemctl restart mariadb
    ```

3. **Create a Replication User**:
    Log in to MariaDB on the master and run the following commands:
    ```sql
    CREATE USER 'repl_user'@'192.168.1.101' IDENTIFIED BY 'password';
    GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'192.168.1.101';
    FLUSH PRIVILEGES;
    ```

4. **Get the Master Status**:
    Run the following command to get the binary log file and position. Note the `File` and `Position` values as they are needed for setting up the slave:
    ```sql
    SHOW MASTER STATUS;
    ```

## 2. Configure the Slave Server

1. **Edit the MariaDB configuration file** on the slave (`/etc/my.cnf` or `/etc/mysql/my.cnf`):
    ```ini
    [mysqld]
    server-id=2             # Unique ID for the slave (must be unique in the replication setup)
    ```

2. **Restart MariaDB** on the slave to apply the changes:
    ```bash
    sudo systemctl restart mariadb
    ```

3. **Set Up Replication on the Slave**:
    Log in to MariaDB on the slave and run the following command. Replace values with those obtained from the master:
    ```sql
    CHANGE MASTER TO
        MASTER_HOST='192.168.1.100',
        MASTER_USER='repl_user',
        MASTER_PASSWORD='password',
        MASTER_LOG_FILE='binlog.000001',  -- Replace with the 'File' value from SHOW MASTER STATUS
        MASTER_LOG_POS=12345;             -- Replace with the 'Position' value from SHOW MASTER STATUS
    ```

4. **Start the Slave**:
    ```sql
    START SLAVE;
    ```

5. **Verify the Slave Status**:
    Run the following command to check the replication status:
    ```sql
    SHOW SLAVE STATUS\G
    ```
    Ensure `Slave_IO_Running` and `Slave_SQL_Running` are both `Yes`. This indicates that replication is working.

## 3. Testing Replication

1. On the master, create a test database and table:
    ```sql
    CREATE DATABASE test_db;
    USE test_db;
    CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(50));
    INSERT INTO test_table VALUES (1, 'Test');
    ```

2. On the slave, check if the data has been replicated:
    ```sql
    USE test_db;
    SELECT * FROM test_table;
    ```
    You should see the same data as on the master.

## Troubleshooting

- Ensure firewall settings allow MariaDB traffic between the master and slave.
- If replication isn’t working, check `SHOW SLAVE STATUS\G` on the slave for errors.
- Make sure both servers have unique `server-id` values.

---

This completes the **MariaDB Master-Slave Replication** setup. With this configuration, changes made on the master will be automatically replicated to the slave server.
