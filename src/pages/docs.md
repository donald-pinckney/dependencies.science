---
title: "Documentation"
description: "Documentation on how to download and use dependencies.science data"
sortOrder: 2
---

### Requirements

**Disk Space:** Currently (April 2023), 200 GB is sufficient space for downloading the metadata dataset and importing it to PostgreSQL. Running PostgreSQL on SSD storage probably provides a substantial speed up. 

**PostgeSQL**: The database currently uses PostgreSQL 15.x, though the database dumps should be importable to newer versions as well.
You can find instructions for installing PostgreSQL [here](https://www.postgresql.org/download/). 
Note that if you do not have root, then building from source allows you to use PostgreSQL as non-root.

Additionally, it is recommended to tweak the default PostgreSQL configuration to achieve better performance for data analysis queries. 
[This website](https://pgtune.leopard.in.ua) (using the Data warehouse option) is a useful tool for suggesting configuration options to change.

**A copy of the database**: If you haven't yet, see the [downloads](/downloads) page to download a dump of the metadata database.


### Importing the NPM Package Metadata Database

Once you have the database tarball downloaded and PostgreSQL setup, first untar the file:

```bash
tar -xvf latest.tar
rm latest.tar
```

You should now have a directory called `db_export`. 
You can now import this to a new Postgres databased named `npm_data` by running:

```bash
psql postgres -c "CREATE DATABASE npm_data WITH TEMPLATE = template0;"
psql npm_data -c "DROP SCHEMA public;"
pg_restore -d npm_data -e -O -j 4 db_export/
```

Note that depending on your Postgres installation, you may need to run the above command as the Postgres root user, commonly `postgres`. 
Additionally, you may need to pass various connection options (port, etc.) to `psql` and `pg_restore` depending on how Postgres is configured.

The `pg_restore` command may take up to around an hour or more, depending on hardware.

### Connecting to the Database and Issuing Queries

Once `pg_restore` completes, you can then connect to the database using `psql npm_data`, or any other SQL tool. 
To test that the import was successful, try to count the number of packages. 
It should look something like this:

```
$ psql npm_data
psql (15.2 (Homebrew))
Type "help" for help.

npm_data=# select count(*) from packages;
  count
---------
 2905313
(1 row)
```

### Database Schema Documentation

&#128679; &#128679; &#128679; Under construction &#128679; &#128679; &#128679; 
