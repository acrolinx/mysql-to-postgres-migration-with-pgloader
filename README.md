Migrating a Reporting Database from MySQL to Postgres with PGLoader
================================================================================

The [Acrolinx Platform's][acrolinx] [backup/restore mechanism][acrolinx-docs-reporting-backups] for the reporting database is designed to work with _all_ supported database servers. In all directions.
That high degree of flexibility comes at the price of moderate performance.
If you need to migrate a large reporting database from MySQL to Postgres, using the [PGLoader][pgloader-github] might be the better option for you. 
This project provides a tested example of how to do that.

Preliminary Considerations
---------------------------
:warning: The [builtin backup/restore utility][acrolinx-docs-reporting-backups] also covers [custom analytics reports][acrolinx-docs-custom-analytics-dashboards]. 
The PGLoader is entirely unaware of those reports. 
They won't be migrated.

Use PGLoader only to migrate the _reporting_ database. 
[For the terminology database the built-in tool is the better option.][acrolinx-docs-terminology-backups]

You don't need to migrate the [JReport databases][acrolinx-docs-connect-external-dbs]. 
Those are ephemeral.

How To
--------
### Install PGLoader and MySQL Client

Install the PGLoader as described [here][pgloader-github-installation].
You might also need the [mysql][mysql-dev-docs-getting-started] client.

### Create the PGLoader Configuration

#### From Template

This project contains a PGLoader configuration template [cfg.load][src-cfg-template].

That template has the following placeholders, which can _mostly_ be passed as [environment variables or per INI file][pgloader-github-templating] (:warning: except one):
* `ACROLINX_MYSQL_PG__SRC_USER`: MySQL user
* `ACROLINX_MYSQL_PG__SRC_PW`: MySQL password
* `ACROLINX_MYSQL_PG__SRC_HOST`: MySQL host
* `ACROLINX_MYSQL_PG__SRC_PORT`: MySQL port
* `ACROLINX_MYSQL_PG__SRC_DB`: MySQL database name
* `ACROLINX_MYSQL_PG__TARGET_USER`: Postgres user
* `ACROLINX_MYSQL_PG__TARGET_PW`: Postgres password
* `ACROLINX_MYSQL_PG__TARGET_HOST`: Postgres host
* `ACROLINX_MYSQL_PG__TARGET_PORT`: Postgres port
* `ACROLINX_MYSQL_PG__TARGET_DB`: Postgres database name
* `ACROLINX_MYSQL_PG__MYSQL_TIMEOUT`: MySQL [net_read_timeout and net_write_timeout][mysql-docs-system-variables] setting
* `ACROLINX_MYSQL_PG__PG_WORK_MEM`: Postgres [work_mem][postgres-docs-resource-consumption] setting
* `ACROLINX_MYSQL_PG__MAX_SEQ`: 
  * New value for the `seq_gen_sequence` in Postgres (see further below)
  * :warning: Not expandable from environment variable
  * :point_right: Needs manual replacement

For convenience, this project contains a little tool that expands template placeholders in an interactive way and creates an expanded output file:
* `chmod 755 bin/expand-cfg-interactive`
* `./bin/expand-cfg-interactive`  
  
#### Which Sequence Value?

In Postgres, the Acrolinx Platform uses only one sequence object (`seq_gen_sequence`). 
That sequence needs to be reset to a sufficiently high value after importing the data. 

(In the MySQL source database primary keys (PKs) were incremented per table. 
In the target database, the highest of those source PKs should be the basis of the next global sequence value.
We weren't able to accomplish such a mapping automatically in the test setup, however. 
Inspiration is always welcome.)

The `cfg.load` configuration sets the sequence to the value specified in `ACROLINX_MYSQL_PG__MAX_SEQ` after importing the data.

But what value should be entered in the first place? 
To find the maximum PK in the source database, we added a procedure [max-id-procedure.txt][src-max-id-procedure]:
* Connect to the source database with the [mysql][mysql-dev-docs-getting-started] client.
* `source src/mysql/max-id-procedure.txt`
* `CALL acrolinxMaxId(@max_id_in_database);`
* `SELECT @max_id_in_database;`
* `DROP PROCEDURE acrolinxMaxId;`

Add a buffer to the result and use it as parameter `ACROLINX_MYSQL_PG__MAX_SEQ`.


### Prepare the Target Database

The Acrolinx Platform needs particular index names in Postgres.
Those may differ from the MySQL index names.
Therefore, we recommend letting the Acrolinx Core Server create the target database schema.
And to migrate only the data with PGLoader.

To do so, perform the following steps:
* Start a Core Platform with the same version as with the source database against the target database. 
* Wait a few seconds.
* Stop the server again when seeing this in the `coreserver.log`: `Reporting database version OK!` 
* Take a schema dump of the target database with `pg_dump` (human readable format). For later sanity checking.

### Migrate

#### In Short
Assuming you created a PGLoader configuration `cfg.load` as described above, run 

```pgloader -v cfg.load```

#### Full Flow
Here's the full procedure:
* Install PGLoader and the MySQL client.
* Clone this repository:
* CD into the root folder.
* Find out the highest MySQL PK as described above.
* Generate the configuration from the template as described above.
* Run `pgloader -v cfg.load`.
  * Either set a log file: `pgloader -v cfg.load -L <logfile>`
  * Or capture the [default log][pgloader-docs-cmdline-options]: `/tmp/pgloader/pgloader.log`

### Sanity Checking

To be on the safe side, do a few sanity checks.

#### Logs

Anything suspicious in the logs? Errors? Warnings?

#### Compare Target Schema Before/After
During the migration the PGLoader drops and restores constraints and indexes. So, double-check if the target schema is still intact:
* Take another schema dump of the target database with `pg_dump` (human readable format).
* Compare it to the dump you took earlier.
  * Either with a simple `diff`.
  * Or with [PostgresCompare][postgres-compare].
  * Or [Liquibase][liquibase-diff]...
* They should be essentially identical.

#### Count Records

First, check the PGLoader summary columns "read" and "imported". Do the numbers match?

Compare the number of records in the databases. 

Count per table in MySQL:
```sql
SELECT SUM(TABLE_ROWS)
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = '{your_db}';
```
(Thanks to [stackoverflow][stackoverflow-count-records-per-table-mysql].)

Count per table in Postgres:
```sql
SELECT schemaname,relname,n_live_tup 
  FROM pg_stat_user_tables 
  ORDER BY n_live_tup DESC;
```
(Thanks to [stackoverflow][stackoverflow-count-records-per-table-postgres].)

Do the numbers make sense?

#### Sequence

Was the sequence updated correctly in the target database?
```sql
select nextVal('seq_gen_sequence')
```

### Disclaimer

The migration steps above represent the _first_ migration path we got to work.
We're not perfect.
We don't claim it can't be simplified.  



Links
------
* [Acrolinx]: Official Acrolinx website.
* [Connect to External Analytics Databases][acrolinx-docs-connect-external-dbs]: Info about the external databases needed by the Acrolinx Platform.
* [Analytics and Reporting Database Backups][acrolinx-docs-reporting-backups]: How to back up and restore a reporting database with the Acrolinx Platform tools.
* [Managing Terminology Database Backups][acrolinx-docs-terminology-backups]: How to back up and restore the terminology database.
* [PostgreSQL Client Applications: `pg_dump`][pg_dump]
* [Product Sunset Policy][acrolinx-docs-sunset-policy]: Acrolinx product sunset policy specifying the demise of MySQL support.
* [Custom Dashboards][acrolinx-docs-custom-analytics-dashboards]: How to create custom analytics reports for the Acrolinx Platform.
* [PGLoader on GitHub][pgloader-github]: PGLoader repository on GitHub.

[acrolinx]: https://www.acrolinx.com/
[acrolinx-docs-connect-external-dbs]: https://docs.acrolinx.com/coreplatform/latest/en/acrolinx-on-premise-only/external-databases/connect-to-external-analytics-databases
[acrolinx-docs-custom-analytics-dashboards]: https://docs.acrolinx.com/coreplatform/latest/en/advanced/analytics-configurations/custom-dashboards
[acrolinx-docs-reporting-backups]: https://docs.acrolinx.com/coreplatform/latest/en/acrolinx-on-premise-only/external-databases/analytics-and-reporting-database-backups
[acrolinx-docs-sunset-policy]: https://docs.acrolinx.com/coreplatform/latest/en/compatibility/product-sunset-policy
[acrolinx-docs-terminology-backups]: https://docs.acrolinx.com/coreplatform/latest/en/acrolinx-on-premise-only/external-databases/managing-terminology-database-backups
[liquibase-diff]: https://docs.liquibase.com/commands/community/diff.html
[mysql-dev-docs-getting-started]: https://dev.mysql.com/doc/mysql-getting-started/en/
[mysql-docs-system-variables]: https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html
[pg_dump]: https://www.postgresql.org/docs/10/app-pgdump.html
[pgloader-docs-cmdline-options]: https://pgloader.readthedocs.io/en/latest/pgloader.html#options
[pgloader-github]: https://github.com/dimitri/pgloader
[pgloader-github-installation]: https://github.com/dimitri/pgloader#install
[pgloader-github-templating]: https://pgloader.readthedocs.io/en/latest/pgloader.html?highlight=ini#templating-with-mustache
[postgres-docs-resource-consumption]: https://www.postgresql.org/docs/10/runtime-config-resource.html
[postgres-compare]: https://www.postgrescompare.com/
[src-cfg-template]: https://github.com/acrolinx/mysql-to-postgres-migration-with-pgloader/blob/main/src/templates/cfg.load
[src-max-id-procedure]: https://github.com/acrolinx/mysql-to-postgres-migration-with-pgloader/blob/main/src/mysql/max-id-procedure.txt
[stackoverflow-count-records-per-table-mysql]: https://stackoverflow.com/a/286048
[stackoverflow-count-records-per-table-postgres]: https://stackoverflow.com/a/2611745