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

Once `pg_restore` completes, you can then connect to the database using `psql npm_data`, or any other SQL tool. 
To test that the import was successful, try to count the number of packages. 
It should look something like this:

```sql
$ psql npm_data
psql (15.2 (Homebrew))
Type "help" for help.

npm_data=# select count(*) from packages;
  count
---------
 2905313
(1 row)
```

If this all works, then you are ready to start developing your own NPM package metadata queries. See below to understand the structure of the tables in the database.

### Structure of the NPM Package Metdata Database

We now describe the main aspects of the structure of the NPM package metadata PostgreSQL database. 
Full details of the database structure can be found among [the migration files on GitHub](https://github.com/donald-pinckney/npm-follower/tree/main/postgres_db/migrations).

#### Packages (`packages`)

The `packages` table lists all the packages scraped from [npmjs.com](https://npmjs.com). 

To see an example, run `select * from packages where name = 'react'`.


#### Versions (`versions`)

The `versions` table lists all the versions of every package. Each version has a `package_id` for which package it is a version of, and a `semver` for its version number. In addtion to other interesting metadata, each version has arrays of `prod_dependencies`, `dev_dependencies`, `peer_dependencies` and `optional_dependencies`, consisting of dependency IDs (see below).

To see an example, run `select * from versions where id = 26027542`.


#### Dependencies (`dependencies`)

Consider a version which has a dependency `"foo" : "^1.2.3"` listed in its `package.json`. We call `foo` the destination package,
and `^1.2.3` the constraint specification. The `dependencies` table lists all these *unique* dependencies.

Note that dependencies have complex relationships with versions & packages:

1. Each source version contains many dependencies, in its `prod_dependencies`, etc. arrays in the `versions` table.
2. Each dependency may be referenced by many different source versions / packages.
3. Each destination package may be referenced by many different dependencies (those with different constraint specifications).
4. Each dependency's constraint specification may imply potential dependency on multiple versions of the destination package. These could be computed directly from the `spec` column which encodes a constraint AST, or from the `raw_spec` column which can be understood by a [semver parsing library](https://github.com/npm/node-semver).

To see an example, run `select * from dependencies where id = 100750`.


#### Advisories (`ghsa`)

The `ghsa` table lists *advisories* scraped from the reviewed NPM section of the [https://github.com/advisories](GitHub Advisory Database). Each advisory has a GHSA ID (`id`), severity, date, references, and natural language descriptions. Note that conceptually (and on GitHub) each advisory contains information about which packages and version ranges are vulnerable. The information is stored separately in the `vulnerabilities` table (see below).

To see an example, run `select * from ghsa where id = 'GHSA-6xrf-q977-5vgc'`.


#### Vulnerabilities (`vulnerabilities`)

The `vulnerabilities` table lists what we call *vulnerabilities*, which are specific version ranges of packages which are vulnerable
to an security advisory. Each vulnerability consists of the advisory in question, as well as the name of the vulnerable package and lower and upper bounds on the vulnerable versions. In addition, each vulnerability lists the first patched version.

Note that a single advisory can have multiple vulnerabilities, meaning that it appears in multiple different packages and/or in multiple different version ranges.

To see an example, run `select * from vulnerabilities where id = 1`.


#### Downloaded Tarballs (`downloaded_tarballs`)

The `downloaded_tarballs` table lists all the URLs and storage information of tarballs that we have downloaded (see below).

To see an example, run `select * from downloaded_tarballs limit(1)`.

### Using the NPM Tarball Dataset

&#128679; &#128679; &#128679; Under construction &#128679; &#128679; &#128679; 
