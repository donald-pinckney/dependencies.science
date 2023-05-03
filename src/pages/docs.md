---
title: "Documentation"
description: "Documentation on how to download and use dependencies.science data"
sortOrder: 2
---

## Metadata of the NPM Ecosystem

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


## Source Code of NPM Packages

### Requirements

**Disk Space:** Currently (April 2023), **over 20 TB** of disk space is needed to download and use the source code dataset. Please ensure that you have sufficient space first.

**Metadata Database:** In order to make meaningful use of the source code dataset, you must also have the [metadata database setup](/docs/#metadata-of-the-npm-ecosystem). 

**Rust:** You may install Rust via [rustup](https://rustup.rs). Version 1.66.0 or newer should work.

**Python:** Our code has been tested with Python 3.8.1, other versions may work as well.

**Redis:** Download Redis using your favorite package manager. For instance, if using snap: `sudo snap install redis`.

### Part 1: Downloading the Blob Store

1. Clone the `npm-follower` repository, and change to the directory with dataset downloading scripts: 
```bash
git clone https://github.com/donald-pinckney/npm-follower.git
cd npm-follower/database_exporting
```

2. Install Python requirements:
```bash
# Could be pip3 depending on your system.
# If you have conflicting installations, or if you prefer, feel free to use a Pip virtual environment.
pip install -r download_tarballs_requirements.txt
```

3. Download the contents of the HuggingFace repo:
```bash
# Could be python3 depending on your system.
python hf_download_repo.py /PATH/TO/DATASET/DIRECTORY
```
This will proceed to download the entire 20+ TB over the network, and after that verify the contents of all files, and redownload any incorrect files.
Needless to say, this will take quite a while. If you run into issues, please don't hesitate to contact us directly via email ([donald_pinckney@icloud.com](mailto:donald_pinckney@icloud.com)) and/or by [filling an issue](https://github.com/donald-pinckney/npm-follower/issues).

4. The dataset is split into pieces so that each file is under 50 GB. You now must restore the dataset:
```bash
python restore.py /PATH/TO/DATASET/DIRECTORY
```  

You should now have 1000 blob files in the `/PATH/TO/DATASET/DIRECTORY` directory.

### Part 2: Setting up Redis

1. Locate your Redis data directory: If you are unsure where you Redis data directory is, you may check with `redis-cli config get dir`. To do so you may need to start Redis, e.g.: `sudo systemctl start redis-server.service`.
   
2. Stop Redis, e.g.: `sudo systemctl stop redis-server.service`.

3. Unpack the Redis dump: the downloaded dump of the metadata database (downloaded in [the previous section](/docs/#metadata-of-the-npm-ecosystem)) includes a dump of a Redis database as well, named `db_export/redis.zip`. You now need to unpack it to your Redis data directory:
```bash
# ***NOTE***: If different, use the path to your 
# Redis data directory in place of /var/lib/redis

sudo su
mv db_export/redis.zip /var/lib/redis
cd /var/lib/redis
unzip redis.zip # WARNING: THIS WILL OVERWRITE YOUR CURRENT REDIS DATA
chown -R redis:redis .
rm redis.zip
```

4. Modify the Redis configuration file (typically `/etc/redis/redis.conf`) as follows:
```text
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"
```

5. Start Redis: To start Redis and enable it to automatically run at startup, run:
```bash
sudo systemctl enable --now redis-server.service
```

The Redis server with the blob metastore should be running at on the Redis database number `4`,
at the URL: `redis://127.0.0.1/4`


### Part 3: Setting up Blob Store Server

The blob store server implements an HTTP API for retrieving and managing file metadata in the
blob filesystem. 

1. Change directory to where you cloned the `npm-follower` repo: `cd /path/to/npm-follower/`
2. Modify `.env` with the following settings:

```text
BLOB_REDIS_URL=redis://127.0.0.1/4
BLOB_API_URL=http://127.0.0.1
BLOB_STORAGE_DIR=/PATH/TO/DATASET/DIRECTORY
```

- `BLOB_REDIS_URL`: This is the endpoint for your Redis database, including the database number (`4`).
- `BLOB_API_URL`: This is the URL of the blob store server. The blob store client will connect at this URL.
- `BLOB_STORAGE_DIR`: This is the directory where you downloaded the blob store to in [Part 1](/docs/#part-1-downloading-the-blob-store).

3. Modify `.secret.env` (create it if it doesn't exist) with the following setting:

```text
BLOB_API_KEY=put_a_secret_key_here
```

- `BLOB_API_KEY`: This is an API key for communicating with the blob store server. This can be useful if the blob store server is not located on the same machine hosting the underlying blob files, and therefore the HTTP API endpoint needs to be exposed to the public. You may choose any key you want.

4. Run the Blob Store Server:
```bash
cargo run --release --bin blob_idx_server -- 0 0
```

The server has capabilities for running distributed jobs across multiple machines in
a slurm cluster. The first two arguments correspond to the number of file-transfer workers and
compute workers. In this case, we are running the server on a single machine in order
to retrieve data from the blob store, so we set both of these arguments to `0`.


### Part 4: Reading From the Dataset

To perform operations on the blob store, the client is used. For this example, we will be using the
`cp` command to copy a file from the blob store to the local machine.

For example, to retrieve the tarball with key `4.17.1.tgz`, run:

```bash
cargo run --release --bin blob_idx_client -- cp 4.17.1.tgz ./express-4.17.1.tgz
```

The client will automatically connect to the filesystem server at the URL specified in the `.env` file,
and retrieve the tarball with key `4.17.1.tgz` and store it at `./express-4.17.1.tgz`. From there you can extract it as a normal tarball.

### Part 5: Using the Source Code Dataset

To know what tarball key to use when reading from the blob store, you **must** look up the appropriate key in the `downloaded_tarballs` table in the PostgreSQL metadata database. For example, to find the key for the tarball downloaded from the URL `https://registry.npmjs.org/express/-/express-4.17.1.tgz`, you can query:

```sql
npm_data=> select blob_storage_key from downloaded_tarballs where 
tarball_url = 'https://registry.npmjs.org/express/-/express-4.17.1.tgz';

  blob_storage_key
--------------------
 express-4.17.1.tgz
```

In turn, the tarball URLs can be lookedup in the `versions` table.
You **can not** rely on blob store keys always having the format of `name-x.y.z.tgz`.