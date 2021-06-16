---
title: "Using GitHub Actions to Integrate Google Sheets and Dolt"
date: "2021-06-16"
author: "Max Hoffman"
authorHref: "https://www.dolthub.com/team#max"
---

# Introduction

We will learn how to version Google Sheets data using Dolt and GitHub
Actions in this blog. This is intended for teams who use Google Sheets
tables to train models, and might benefit from a Git Pull-Request
process for managing data changes over time. We will touch on branch,
diff, merge and rollback operations in our demo.

# Background

Startups are releasing heavy new tools to monitor production drift,
validate expectation suites on batched data, and compute visualizations
of data embeddings. Getting in front of data changes catches bugs before
they reach production, and tracks data staleness as models adapt to the
real world. Understanding the latent space of enterprise data keeps
analysts informed and better poised to communicate with stakeholders.

These are all useful pieces of technology, but they are also complex and
can make the task of understanding data patterns more intimidating.
Beneath the nose of all this math is the data diff: seeing how data
changed from one version to another. Diffs are a simple and crude
hammer, but seeing a summary of row changes is always useful, even if
only for use cases like tracking image metadata.

Dolt is an SQL database with Git styled versioning. The Dolt storage
layer was designed to support branching, diffing and merging as first
class citizens. Avoiding data duplication is helpful, but as we hope to
show in this blog, the diffing, reproducibility and tracing afforded by
commit graphs improve data pipelines in the same way code versioning
strengthens web applications.

# Data Syncing

Today we consider the use case where data management is divided between
analysts and data scientists. Analysts work with data and train models
to identify and improve bad data. Data scientists iterate on model
architectures and track production quality. Analysts handoff cleaned
data to their data scientists’ best models. Google Sheets is an
organizational dividing line between two expert camps.

This handoff creates several problems. The data is always changing
because analysts are always improving, mixing and matching training
rows. Understanding how data differences impact successive model
versions is not feasible in the first setup. Rollbacks are also limited,
given the difficulty reproducing past training phases. Productionizing
this customer’s model in practice also requires packing dozens of
compositional models into one docker image. In the event of a
regression, our options are either rebuild everything with the newest
data versions (assuming the offending model was fixed), or rollback to
the last built image and clobber every model upgrade.

The next iteration of this data process adds versioning between analysts
and data scientists. If we were interested strictly in reproducibility,
we could export GSheets into CSV files named by timestamp. A forced
rollback would involve retraining the offending model on an old version
of data and pushing the individual fix.

The organization handoff and importance of data changes increases the
appeal of Git/Dolt style versioning workflows. Dolt workflows separate
the “production” version of data from “development” branches. Pull
requests let analysts comment on each other’s additions before they are
merged to a master branch. Data diffs provide a human friendly interface
to review proposed changes. And commit versioning provides release
tagging, rollbacks, and point-in-time queries, all particularly helpful
with the scale of data and model versions in each Docker image release.

![GSheets Architecture](./static/plugin_architecture.png)

# See It In Action

Our Github Actions workflow file ([found
here](https://github.com/dolthub/gsheets-sync-demo/blob/master/.github/workflows/test.yml)
includes three steps:


Download sheets into temporary CSV files
([dolthub/gsheets-to-csv](https://github.com/dolthub/gsheets-to-csv)).
Import the CSV files into a Dolt table and push results to DoltHub
([dolthub/dolt-action](https://github.com/dolthub/dolt-action)).
Log the import diff (if any) in our workflow logs.

```yml
name: Import GSheets into Dolt
on:
  workflow_dispatch:
env:
  id: '16iUpjhWnPYDo4OUwxv1TlF9x975iPJ2WABgmpoJJdNY'
  remote: 'max-hoffman/dolt_action_test'
  branch: 'dev'
jobs:
  plugin_tests:
    runs-on: ubuntu-latest
    name: Test Plugin
    steps:
      - name: Step 1 - Download Agent Data
        id: 'sheet_to_csv'
        uses: dolthub/gsheets-to-csv@v0.5
        with:
          creds: ${{ secrets.GOOGLE_CREDENTIAL }}
          tempdir: ${{ github.workspace }} #/github/workspace
          sheets: |
            [
              { "id": "${{ env.id }}" },
              { "id": "${{ env.id }}", "title": "Sheet2", "range":
              "A1:C10" }
            ]
      - name: Step 2 - Dolt Import
        uses: dolthub/dolt-action@v0.5
        id: 'dolt_import'
        env:
          FILES: ${{ steps.sheet_to_csv.outputs.results }}
        with:
          before: |
            file_list=($FILES)
            head "${file_list[1]}"
            for file in "${file_list[@]}"; do
              dolt table import --update-table -pk id aminals "$file"
            done
          remote: ${{ env.remote }}
          branch: ${{ env.branch }}
          commit_message: 'Dolt action commit'
          dolthub_credential: ${{ secrets.DOLTHUB_CREDENTIAL }}
          push: true
      - name: Step 3 - Table Diff
        uses: dolthub/dolt-action@v0.5
        id: 'training_diff'
        if: steps.dolt_import.outputs.commit != ''
        with:
          remote: ${{ env.remote }}
          branch: ${{ env.branch }}
          dolthub_credential: ${{ secrets.DOLTHUB_CREDENTIAL }}
          before: |
            dolt diff HEAD^ HEAD
```

Two inputs are required to run this workflow: `DOLTHUB_CREDENTIAL` and
`GSHEETS_CREDENTIAL`. The DoltHub credential can be substituted for AWS
or GCP credentials to access respective database remotes. Your local
DoltHub credential can be found in `~/.dolt/creds`:

```bash
$ ls ~/.dolt/creds
-rw-------  1 max-hoffman  staff  175 Mar 31 15:39 7on1utvveb1uoakrc9ku9m2abotsg2q9mhph68i072un0.jwk
-rw-------  1 max-hoffman  staff  175 Mar 31 15:40 jsgj5bnsu0olp8n95gom7vfmahi6h4n2eaf578kuti2aq.jwk
```

Google Cloud [service account
credentials](https://cloud.google.com/iam/docs/service-accounts) are
currently required, even for public Google Sheets. The Google Cloud
service credential can double as authorization for a GCP Dolt remote.

```bash
{
 "type": "service_account",
 "project_id": "dry-icicle-281492,
 "private_key_id": "a3cdbe532c4ad7g6x4vv700cgge4683e131331gg",
 "private_key": "-----BEGIN PRIVATE KEY-----\...n-----END PRIVATE
 KEY-----\n",
 "client_email": "runner@dry-icicle281492.iam.gserviceaccount.com",
 "client_id": "131822607917526703563",
 "auth_uri": "https://accounts.google.com/o/oauth2/auth",
 "token_uri": "https://oauth2.googleapis.com/token",
 "auth_provider_x509_cert_url":
 "https://www.googleapis.com/oauth2/v1/certs",
 "client_x509_cert_url":
 "https://www.googleapis.com/robot/v1/metadata/x509/runner%40dry-icicle-281492.iam.gserviceaccount.com"
}
```

With the proper credentials set, we can run the workflow:

![Workflow Dispatch](./static/workflow_dispatch.png)


Successful output logs (simplified for clarify) are detailed in the
following section.

## Step 1:

Download the entire first sheet, and A1 through C10 of the sheet named
Sheet2.

```bash
downloading sheet: ***'id':
'16iUpjhWnPYDo4OUwxv1TlF9x975iPJ2WABgmpoJJdNY'***
downloading sheet: ***'id':
'16iUpjhWnPYDo4OUwxv1TlF9x975iPJ2WABgmpoJJdNY', 'title': 'Sheet2',
'range': 'A1:C10'***
```

## Step 2:

Head the first file, and import both into a Dolt repository.

```bash
+ dolt clone max-hoffman/dolt_action_test -b master
+ /github/workspace/doltdb
cloning https://doltremoteapi.dolthub.com/max-hoffman/dolt_action_test
Retrieving remote information.0 of 49 chunks complete. 0 chunks being
downloaded currently.0 of 49 chunks complete. 1 chunks being downloaded
currently.1 of 49 chunks complete. 0 chunks being downloaded currently.1
of 49 chunks complete. 4 chunks being downloaded currently.1 of 49
chunks complete. 6 chunks being downloaded currently.5 of 49 chunks
complete. 2 chunks being downloaded currently.7 of 49 chunks complete. 0
chunks being downloaded currently.7 of 49 chunks complete. 1 chunks
being downloaded currently.8 of 49 chunks complete. 0 chunks being
downloaded currently.8 of 49 chunks complete. 2 chunks being downloaded
currently.10 of 49 chunks complete. 0 chunks being downloaded
currently.10 of 49 chunks complete. 6 chunks being downloaded
currently.10 of 49 chunks complete. 14 chunks being downloaded
currently.16 of 49 chunks complete. 8 chunks being downloaded currently.
24 of 49 chunks complete. 0 chunks being downloaded currently.24 of 49
chunks complete. 4 chunks being downloaded currently.28 of 49 chunks
complete. 0 chunks being downloaded currently.28 of 49 chunks complete.
4 chunks being downloaded currently.28 of 49 chunks complete. 7 chunks
being downloaded currently.28 of 49 chunks complete. 10 chunks being
downloaded currently.31 of 49 chunks complete. 7 chunks being downloaded
currently. 34 of 49 chunks complete. 4 chunks being downloaded
currently.38 of 49 chunks complete. 0 chunks being downloaded
currently.38 of 49 chunks complete. 4 chunks being downloaded
currently.38 of 49 chunks complete. 8 chunks being downloaded
currently.42 of 49 chunks complete. 4 chunks being downloaded
currently.46 of 49 chunks complete. 0 chunks being downloaded
currently.46 of 49 chunks complete. 3 chunks being downloaded
currently.49 of 49 chunks complete. 0 chunks being downloaded currently.

+ head /github/workspace/tmp1.csv
id,name,speed
11,crocodile,20
12,pigeon,93

+ dolt table import --update-table -pk id aminals
+ /github/workspace/tmp0.csv
Import completed successfully.
Rows Processed: 6, Additions: 0, Modifications: 0, Had No Effect: 6

+ dolt table import --update-table -pk id aminals
+ /github/workspace/tmp1.csv
Import completed successfully.
Rows Processed: 1, Additions: 1, Modifications: 0, Had No Effect: 0

+ dolt commit -m 'Dolt action commit'
commit hph1gdos395fnfkrenfts9q8hd6r2qam
Author: GitHub Actions <actions@github.com>

Date:   Tue Jun 15 21:33:44 +0000 2021

    Dolt action commit

+ dolt push origin master
Tree Level: 7 has 6 new chunks. Determining how many are needed.Tree
Level: 7 has 6 new chunks of which 5 already exist in the database.
Buffering 1 chunks.
Tree Level: 7. 100.00% of new chunks buffered.
Tree Level: 1 has 3 new chunks. Determining how many are needed.Tree
Level: 1 has 3 new chunks of which 2 already exist in the database.
Buffering 1 chunks.
Tree Level: 1. 100.00% of new chunks buffered.
```

## Step 3:

Log the difference between the last two commits if we added data during
the import step.

```bash
+ dolt diff 'HEAD^' HEAD
diff --dolt a/aminals b/aminals
--- a/aminals @ lr8e0ubininvlj936vfvsp23ctvklaoo
+++ b/aminals @ 0988hnjk7jftb9h30sfau81g7b8k7b3n
+-----+----+--------+-------+
|     | id | name   | speed |
+-----+----+--------+-------+
|  +  | 12 | pigeon | 93    |
+-----+----+--------+-------+
```

Afterwards, we verify our workflow completed successfully:

![Successful Workflow](./static/successful_workflow.png)

And observe our updates in DoltHub:

![DoltHub Table](./static/dolthub_table.png)

We pushed data to the `dev` branch. We can create a Pull Request to
apply these changes to `master`:

![Create PR](./static/pr_create.png)

And view the diff in DoltHub before merging to master:

![PR Diff](./static/pr_diff.png)

# Technical Details

Comfortably editing YAML files is one speedbump of GitHub actions. I
tested a lot of code locally with the [act
tool](https://github.com/nektos/act), which allows you to test workflows
without hopping between your terminal and the GitHub Actions website.

The local `act` experience has small discrepancies between running
workflows on GitHub’s platform. One discrepancy relevant to our demo
workflow concerns temporary file persistence between jobs. Act is
designed to mount local files in the workflow volume to test work in
progress. GitHub actions also does this, but it can be confusing to keep
track of which directories are mounted where, especially when `use:
actions/checkout@2` is included in a workflow.

A few things are important to understand:

GitHub Actions differentiates between `uses` and `run` steps. A step can
only be one or the other. A `run` step is the default workflow runtime,
presumably a docker image with a set of workspace and home directories.
A `uses` step is a docker-in-docker context that mounts directories into
a new container.
The default working directory for a workflow in the parent container is
either `/github/workspace`, or a separate repo directory if `uses:
actions/checkout@2` precedes a workflow step.
The working directory of a child container (a `uses` step) is always
`/github/workspace`. A checkout step changes the parent container
working directory and the`${{ github.workspace }}` context variable, but
the home directory will still mount to `/github/workspace` in child
containers of `uses` blocks.

[print directory examples to clarify the above]

A plugin user is mainly concerned with the where temporary files, and
how to access custom scripts in a `uses: dolthub/dolt-action`.

Our demo workflow steps are `uses` blocks, so files are written to
`dolthub/workspace`. If you tried accessing the output files in a `run`
block without any other changes, you would error for referencing
non-existent file paths.

To fix this problem and pass temporary files to a `run` block, you would
use relative paths.

Custom scripts in a `dolthub/dolt-action` are relative to
`/github/workspace`, the mount target of a `uses` step. You could use a
environment variable to capture the default before a workflow run with a
checkout edits this variable:

```bash
env:
  NAME: ${{ github.workspace }}
With:
  Before: |
    $NAME/myscript.sh ${{ steps.gsheets_to_csv.outputs.files }}
```

`dolthub/dolt-action` changes the working directory to
`/github/workspace/doltdb`, so the following also works, although is
liable to break if we implement DoltHub repo caching and move the
database directory:

```bash
with:
  before: |
    ../myscript.sh ${{ steps.gsheets_to_csv.outputs.files }}
```

Both Google Sheets and GitHub Actions have memory limits. Individual
Google Sheets were limited to 20MB and GitHub Action runners allotted
1GB of memory at the time of writing.

# Room For Improvement

As mentioned previously, Google service account credentials are
required, even for public data. We should make this config field
optional in the future.

Database caching between steps is another potential improvement.
Databases currently self-delete at the conclusion of a workflow step.
Two sequential steps, like importing into a database followed by
printing a diff log in the event of a commit, require two full clones.
The complexity of GitHub actions working directories between steps make
this feature a bit more difficult to implement, but more valuable for
large workflows.

# Summary

We have discussed how to sync Google Sheets data into Dolt with GitHub
Actions. This integration was motivated by a customer whose analytics
and model management responsibilities are divided between teams who
stand to benefit from data versioning, a branch and merge PR process to
standardize data handoff, and diffs/rollbacks for responding to
production regressions.

If you are interested in trying these plugins yourself, you can find
[dolthub/gsheets-to-csv](https://github.com/dolthub/gsheets-to-csv) and
[dlthub/dolt-actions](https://github.com/dolthub/dolt-action) on GitHub.
If you are interested in asking the Dolt team questions about your data
infrastructure, feel free to [reach out to us on
Discord](https://discord.com/invite/RFwfYpu).

