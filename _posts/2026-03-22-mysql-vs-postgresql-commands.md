---
layout: post
title: "MySQL vs PostgreSQL — Comparison and Common Commands"
toc: true
categories: [MySQL, PostgreSQL]

---

MySQL and PostgreSQL are two of the most popular open-source relational databases. They share SQL foundations but differ in philosophy, features, and typical use-cases. Below is a concise comparison followed by a handy table of equivalent, commonly-used commands.

<!--more-->

## Short comparison

- **Philosophy:** MySQL prioritizes simplicity and speed (common for web apps). PostgreSQL emphasizes standards compliance, extensibility, and advanced features (good for complex applications and analytics).
- **Transactions & Concurrency:** PostgreSQL's MVCC and ACID guarantees are robust; MySQL (InnoDB) is ACID-compliant but historically optimized for performance trade-offs.
- **Extensibility & Types:** PostgreSQL supports user-defined types, functions in multiple languages, and many extensions (e.g., PostGIS). MySQL has plugins and storage engines but is less flexible.
- **JSON & NoSQL-like features:** PostgreSQL's `JSONB` and indexing are powerful; MySQL provides a `JSON` type and functions but is generally less feature-rich for complex JSON queries.
- **Performance:** Workload-dependent — MySQL often excels at simple read-heavy workloads, PostgreSQL shines for complex queries and analytics when tuned.

## Command equivalence table

| Task | MySQL | PostgreSQL | Notes |
|---|---:|---:|---|
| Connect to server | `mysql -u user -p -h host` | `psql -U user -h host -d dbname` | `psql` can run single commands with `-c`.
| List databases | `SHOW DATABASES;` | `\l` or `SELECT datname FROM pg_database;` | `\l` is a psql meta-command.
| Create database | `CREATE DATABASE dbname;` | `CREATE DATABASE dbname;` | Same SQL; `createdb dbname` is a wrapper in PostgreSQL.
| Drop database | `DROP DATABASE dbname;` | `DROP DATABASE dbname;` | `dropdb dbname` is a wrapper.
| Select / use database | `USE dbname;` | `\c dbname` | `\c` switches database in psql.
| List tables | `SHOW TABLES;` | `\dt` or `SELECT tablename FROM pg_tables WHERE schemaname='public';` | `\dt` lists tables in psql.
| Describe table structure | `DESCRIBE tablename;` | `\d tablename` or query `information_schema.columns` | `\d+ table` shows extra info.
| Show CREATE TABLE | `SHOW CREATE TABLE tablename;` | `pg_dump -s -t tablename dbname` or `\d+ tablename` | PostgreSQL has no single `SHOW CREATE TABLE` SQL.
| Show processes/connections | `SHOW PROCESSLIST;` | `SELECT pid, usename, state, query FROM pg_stat_activity;` | `SHOW FULL PROCESSLIST;` shows full queries in MySQL.
| Kill query/connection | `KILL <thread_id>;` | `SELECT pg_terminate_backend(<pid>);` | Requires appropriate privileges.
| Create user | `CREATE USER 'user'@'host' IDENTIFIED BY 'pw';` | `CREATE ROLE user WITH LOGIN PASSWORD 'pw';` | PostgreSQL uses roles; `CREATE USER` is an alias for `CREATE ROLE ... LOGIN`.
| Grant privileges | `GRANT SELECT ON db.table TO 'user'@'host';` | `GRANT SELECT ON table TO user;` | PostgreSQL privileges are role-based.
| Change password | `ALTER USER 'user'@'host' IDENTIFIED BY 'newpw';` | `ALTER ROLE user WITH PASSWORD 'newpw';` | In psql: `\password user`.
| Show users | `SELECT User, Host FROM mysql.user;` | `\du` or `SELECT rolname FROM pg_roles;` |  |
| Show server version | `SELECT VERSION();` | `SELECT version();` | Same function name.
| Show variables/settings | `SHOW VARIABLES;` | `SHOW all;` or `SELECT name, setting FROM pg_settings;` | PostgreSQL's `pg_settings` is informative.
| Start/stop service (systemd) | `sudo systemctl start mysqld` (or `mysql`) | `sudo systemctl start postgresql` | Service names vary by distribution.
| Backup (logical) | `mysqldump -u user -p dbname > dump.sql` | `pg_dump -U user -d dbname -F p > dump.sql` | Use `-F c` for custom format with `pg_dump`.
| Restore SQL file | `mysql -u user -p dbname < dump.sql` | `psql -U user -d dbname -f dump.sql` | For custom-format use `pg_restore`.
| Restore custom-format | N/A (mysqldump produces SQL) | `pg_restore -U user -d dbname dumpfile` | PostgreSQL has non-SQL dump formats.
| Dump single table | `mysqldump -u user -p dbname tablename > table.sql` | `pg_dump -U user -d dbname -t tablename > table.sql` |  |
| Export table to CSV | `SELECT * INTO OUTFILE '/path/file.csv' FIELDS TERMINATED BY ',' FROM tablename;` | `COPY tablename TO '/path/file.csv' WITH CSV HEADER;` | `COPY` runs on server; `\copy` runs client-side.
| Import CSV | `LOAD DATA INFILE '/path/file.csv' INTO TABLE tablename FIELDS TERMINATED BY ',';` | `COPY tablename FROM '/path/file.csv' WITH CSV HEADER;` or `\copy` | `\copy` useful when server cannot access path.
| Transactions | `START TRANSACTION; ... COMMIT;` | `BEGIN; ... COMMIT;` | Both support isolation levels and savepoints.
| Row locking | `SELECT ... FOR UPDATE` | `SELECT ... FOR UPDATE` | Similar syntax; MVCC semantics differ.
| Auto-increment PK | `INT AUTO_INCREMENT PRIMARY KEY` | `SERIAL`/`BIGSERIAL` or `GENERATED ALWAYS AS IDENTITY` | PostgreSQL `IDENTITY` is SQL-standard.
| Upsert | `INSERT ... ON DUPLICATE KEY UPDATE ...` | `INSERT ... ON CONFLICT (cols) DO UPDATE ...` | `ON CONFLICT` is flexible with constraints.
| Show indexes | `SHOW INDEX FROM tablename;` | `\d tablename` or `SELECT * FROM pg_indexes WHERE tablename='tablename';` |  |
| Create index without blocking | N/A (MySQL index creation may lock) | `CREATE INDEX CONCURRENTLY idx_name ON table(col);` | Use `CONCURRENTLY` to avoid blocking writes.
| Full-text search | `FULLTEXT` indexes, `MATCH()/AGAINST()` | `tsvector`/`tsquery`, GIN indexes | PostgreSQL full-text search is feature-rich.
| Extensions / plugins | Storage engines (InnoDB, MyISAM), plugins | `CREATE EXTENSION extension_name;` (e.g., `postgis`) | PostgreSQL extension ecosystem is large.
| Sequences | `ALTER TABLE ... AUTO_INCREMENT = ...` | `SELECT nextval('seq'); SELECT currval('seq'); ALTER SEQUENCE seq RESTART WITH n;` | Sequences are first-class in PostgreSQL.
| Temporary tables | `CREATE TEMPORARY TABLE ...` | `CREATE TEMP TABLE ...` | Session-scoped temporary tables in both.
| Explain plan | `EXPLAIN SELECT ...;` | `EXPLAIN ANALYZE SELECT ...;` | `EXPLAIN ANALYZE` runs the query and shows real costs.
| Vertical/expanded output | Append `\G` to the statement in the `mysql` client (e.g. `SELECT * FROM t\G`) | `\x` or `\pset expanded` in `psql` (toggle expanded display); `\x auto` for automatic switching | `\G` affects only that one statement in the MySQL client; `\x` toggles mode in psql and persists until toggled off. |

## Final notes

Both databases are excellent choices; pick based on project needs:

- Choose MySQL for simple, widely-supported web apps and when ecosystem/tooling matters.
- Choose PostgreSQL for advanced SQL features, extensibility, complex schemas, and analytical workloads.
