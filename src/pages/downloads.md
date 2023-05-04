---
title: "Downloads"
description: "Download snapshots of dependencies.science data"
sortOrder: 1
hideDescription: true
--- 

### NPM Package Metadata (PostgreSQL, ~30 GB)

Download the [latest snapshot of the metadata database](https://downloads.dependencies.science/metadata/latest.tar), or see a [list of the most recent snapshots](https://downloads.dependencies.science/metadata/).

After downloading the database, please see the [documentation](/setup/#metadata-db-of-npm-packages) for how to import this to PostgreSQL, and more information on the database schema.

### NPM Package Code (~20 TB)

The full source code of all scraped packages is hosted on [HuggingFace](https://huggingface.co/datasets/nuprl/npm-follower-data).

To correctly download the dataset, verify downloaded files, and prepare the dataset for loading, please follow the instructions in the [documentation](/setup/#source-code-of-npm-packages) rather than downloading it directly from HuggingFace.
