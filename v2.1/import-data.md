---
title: Import Data
summary: Learn how to import data into a CockroachDB cluster.
toc: false
---

CockroachDB supports importing data from CSV/TSV or SQL dump files.

{{site.data.alerts.callout_info}}To import/restore data from CockroachDB-generated <a href="backup.html">enterprise license backups</a>, see <a href="restore.html"><code>RESTORE</code></a>.{{site.data.alerts.end}}

<div id="toc"></div>

## Import from tabular data (CSV)

If you have data exported in a tabular format (e.g., CSV or TSV), you can use the [`IMPORT`](import.html) statement.

To use this statement, though, you must also have some kind of remote file server (such as Amazon S3 or a custom file server) that all your nodes can access.

## Import from generic SQL dump

You can execute batches of `INSERT` statements stored in `.sql` files (including those generated by [`cockroach dump`](sql-dump.html)) from the command line, importing data into your cluster.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --database=[database name] < statements.sql
~~~

{{site.data.alerts.callout_success}}Grouping each <code>INSERT</code> statement to include approximately 500-10,000 rows will provide the best performance. The number of rows depends on row size, column families, number of indexes; smaller rows and less complex schemas can benefit from larger groups of <code>INSERTS</code>, while larger rows and more complex schemas benefit from smaller groups.{{site.data.alerts.end}}

## Import from PostgreSQL dump

If you're importing data from a PostgreSQL deployment, you can import the `.sql` file generated by the `pg_dump` command to more quickly import data.

{{site.data.alerts.callout_success}}The <code>.sql</code> files generated by <code>pg_dump</code> provide better performance because they use the <code>COPY</code> statement instead of bulk <code>INSERT</code> statements.{{site.data.alerts.end}}

### Create PostgreSQL SQL file

Which `pg_dump` command you want to use depends on whether you want to import your entire database or only specific tables:

- Entire database:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ pg_dump [database] > [filename].sql
    ~~~

- Specific tables:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ pg_dump -t [table] [table's schema] > [filename].sql
    ~~~

For more details, see PostgreSQL's documentation on [`pg_dump`](https://www.postgresql.org/docs/9.1/static/app-pgdump.html).

### Reformat SQL file

After generating the `.sql` file, you need to perform a few editing steps before importing it:

1. Remove all statements from the file besides the `CREATE TABLE` and `COPY` statements.
2. Manually add the table's [`PRIMARY KEY`](primary-key.html#syntax) constraint to the `CREATE TABLE` statement.
  This has to be done manually because PostgreSQL attempts to add the primary key after creating the table, but CockroachDB requires the primary key be defined upon table creation.
3. Review any other [constraints](constraints.html) to ensure they're properly listed on the table.
4. Remove any [unsupported elements](sql-feature-support.html).

### Import data

After reformatting the file, you can import it through `psql`:

{% include copy-clipboard.html %}
~~~ shell
$ psql -p [port] -h [node host] -d [database] -U [user] < [file name].sql
~~~

For reference, CockroachDB uses these defaults:

- `[port]`: **26257**
- `[user]`: **root**

## Import from MySQL dump

{% include experimental-warning.md %}

<span class="version-tag">New in v2.1:</span> This section has instructions for getting data from MySQL dump files into CockroachDB using [`IMPORT`](import.html). It uses the [employees data set](https://github.com/datacharmer/test_db) that is also used in the [MySQL docs](https://dev.mysql.com/doc/employee/en/).

- [Step 1. Dump the MySQL database](#step-1-dump-the-mysql-database)
- [Step 2. `IMPORT` the dump files](#step-2-import-the-dump-files)

### Step 1. Dump the MySQL database

{% include copy-clipboard.html %}
~~~ shell
$ mysqldump -uroot employees employees > employees.sql
~~~

### Step 2. `IMPORT` the dump files

To import the MySQL dump file for a table, issue an `IMPORT` statement like the one shown below.

You will need to look at the dump file and translate the MySQL [`CREATE TABLE`](create-table.html) statement there into something CockroachDB understands.

This example uses S3. For a complete list of the types of cloud storage `IMPORT` can pull from, see [Import File URLs](import.html#import-file-urls).

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  MYSQLDUMP DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=ACCESSKEY&AWS_SECRET_ACCESS_KEY=SECRET');
~~~

Success will look like:

~~~
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
|       job_id       |  status   | fraction_completed | rows | index_entries | system_records | bytes |
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
| 352938237301293057 | succeeded |                  1 |    0 |             0 |              0 |     0 |
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
(1 row)
~~~

To load the data from local files, you have 2 options:

- Start up a [local HTTP server](create-a-file-server.html#using-caddy-as-a-file-server) to serve the files, and point `MYSQLDUMP` at the server's URL. This is the recommended option.

- Create an `extern` subdirectory on each node and copy the MySQL dump files there. Note that you will need to copy the dump files onto every node, since CockroachDB may execute the `IMPORT` statement on any of the nodes.

If you decide to load the data from the `extern` subdirectory, you will need to use [`IMPORT`'s `nodelocal` URL scheme](import.html#import-file-urls) as shown below.

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  MYSQLDUMP DATA ('nodelocal:///employees_table.sql');
~~~

## See also

- [SQL Dump (Export)](sql-dump.html)
- [Back up Data](back-up-data.html)
- [Restore Data](restore-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)
