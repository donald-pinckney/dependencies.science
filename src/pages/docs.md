---
title: "Documentation"
description: "Documentation on the structure of dependencies.science data"
sortOrder: 3
---

## Metadata of the NPM Ecosystem

Below are the main aspects of the structure of the NPM package metadata PostgreSQL database. 
Full details of the database structure can be found among [the migration files on GitHub](https://github.com/donald-pinckney/npm-follower/tree/main/postgres_db/migrations).

If you do not yet have the metadata database setup, please follow [the setup instructions](/setup/#how-to-setup-package-metadata-db).

### Packages (`packages`)

The `packages` table lists all the packages scraped from [npmjs.com](https://npmjs.com). 

To see an example, run `select * from packages where name = 'react'`.


### Versions (`versions`)

The `versions` table lists all the versions of every package. Each version has a `package_id` for which package it is a version of, and a `semver` for its version number. In addtion to other interesting metadata, each version has arrays of `prod_dependencies`, `dev_dependencies`, `peer_dependencies` and `optional_dependencies`, consisting of dependency IDs (see below).

To see an example, run `select * from versions where id = 26027542`.


### Dependencies (`dependencies`)

Consider a version which has a dependency `"foo" : "^1.2.3"` listed in its `package.json`. We call `foo` the destination package,
and `^1.2.3` the constraint specification. The `dependencies` table lists all these *unique* dependencies.

Note that dependencies have complex relationships with versions & packages:

1. Each source version contains many dependencies, in its `prod_dependencies`, etc. arrays in the `versions` table.
2. Each dependency may be referenced by many different source versions / packages.
3. Each destination package may be referenced by many different dependencies (those with different constraint specifications).
4. Each dependency's constraint specification may imply potential dependency on multiple versions of the destination package. These could be computed directly from the `spec` column which encodes a constraint AST, or from the `raw_spec` column which can be understood by a [semver parsing library](https://github.com/npm/node-semver).

To see an example, run `select * from dependencies where id = 100750`.


### Advisories (`ghsa`)

The `ghsa` table lists *advisories* scraped from the reviewed NPM section of the [https://github.com/advisories](GitHub Advisory Database). Each advisory has a GHSA ID (`id`), severity, date, references, and natural language descriptions. Note that conceptually (and on GitHub) each advisory contains information about which packages and version ranges are vulnerable. The information is stored separately in the `vulnerabilities` table (see below).

To see an example, run `select * from ghsa where id = 'GHSA-6xrf-q977-5vgc'`.


### Vulnerabilities (`vulnerabilities`)

The `vulnerabilities` table lists what we call *vulnerabilities*, which are specific version ranges of packages which are vulnerable
to an security advisory. Each vulnerability consists of the advisory in question, as well as the name of the vulnerable package and lower and upper bounds on the vulnerable versions. In addition, each vulnerability lists the first patched version.

Note that a single advisory can have multiple vulnerabilities, meaning that it appears in multiple different packages and/or in multiple different version ranges.

To see an example, run `select * from vulnerabilities where id = 1`.


### Downloaded Tarballs (`downloaded_tarballs`)

The `downloaded_tarballs` table lists all the URLs and storage information of tarballs that we have downloaded (see below).

To see an example, run `select * from downloaded_tarballs limit(1)`.


## Source Code of NPM Packages

If you do not yet have the package source code blob store setup, please follow [the setup instructions](/setup/#source-code-of-npm-packages).

### Running the Blob Storage Client

To perform operations on the blob store, the client is used. For this example, we will be using the
`cp` command to copy a file from the blob store to the local machine.

For example, to retrieve the tarball with key `express-4.17.1.tgz`, run:

```bash
cargo run --release --bin blob_idx_client -- cp express-4.17.1.tgz ./express-4.17.1.tgz
```

The client will automatically connect to the filesystem server at the URL specified in the `.env` file,
and retrieve the tarball with key `express-4.17.1.tgz` and store it at `./express-4.17.1.tgz`. From there you can extract it as a normal tarball.

### Part 5: Using the Source Code Dataset

To know what tarball key to use when reading from the blob store, you **must** look up the appropriate key in the `downloaded_tarballs` table in the PostgreSQL metadata database. For example, to find the key for the tarball downloaded from the URL `https://registry.npmjs.org/express/-/express-4.17.1.tgz`, you can query:

```sql
npm_data=> select blob_storage_key from downloaded_tarballs where 
tarball_url = 'https://registry.npmjs.org/express/-/express-4.17.1.tgz';

  blob_storage_key
--------------------
 express-4.17.1.tgz
```

In turn, the tarball URLs can be looked up in the `versions` table.
You **can not** rely on blob store keys always having the format of `name-x.y.z.tgz`.