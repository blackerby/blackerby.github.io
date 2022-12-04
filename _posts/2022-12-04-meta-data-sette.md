---
title: "(meta)data(sette)"
layout: post
date: "2022-12-04"
---

This post is part of my [Metadata Workflow Series](https://blackerby.github.io/workflow-series/).

## Intro

After running [cleanup.sh](https://blackerby.github.io/2022/11/25/cleanup.html) I like to make a local copy of it as a tab-separated file I can do various things with, including storing it in SQLite database with [Datasette](https://datasette.io/) which, among other things, allows me to link the metadata with their associated text files.

## The Code

As usual, it's a shell script.

```zsh
#!/usr/bin/env zsh
```

I use [wget](https://www.gnu.org/software/wget/) to save the data to my local machine. Getting the URL right took a little Googling and turned up this great answer on [Stack Overflow](https://stackoverflow.com/a/28494469). The spreadsheet has to be published to the web in order for this to work.

```zsh
wget -q -O ./all_data.tsv https://docs.google.com/spreadsheets/d/e/2PACX-1vQkoFBuqjcBX-a_xT3IfTmLmUJ0LZme8Mj16sIdqqEC9Z_vPmyHJKiTCRrZHPaSsxI-4lKaLkWD_XSk/pub\?gid\=0\&single\=true\&output\=tsv
```

### The Database

I then (re)create the SQLite database containing the bill metadata and the summary text files using Simon Willison's extraordinarily easy to use [sqlite-utils](https://sqlite-utils.datasette.io/en/stable/).

First, I drop the existing tables that hold the core data.

```zsh
sqlite-utils drop-table summaries.db actions
sqlite-utils drop-table summaries.db files
```

Then I recreate the `actions` table, which is the table that holds the bill metadata

```zsh
sqlite-utils insert summaries.db actions all_data.tsv --tsv -d
```

I rename the columns for from the spreadsheet for use in the database. There may be a better way to do this, but this works.

```zsh
sqlite-utils transform summaries.db actions --rename "Congress" "congress" --rename "Session" "session" --rename "PL Num (if applicable)/PVTL Num" "pl_num" --rename "Bill Type" "bill_type" --rename "Bill Number " "bill_number" --rename "Sponsor" "sponsor" --rename "Committee" "committee" --drop "Action Code" --drop "Summary Version Code" --drop "Report Number" --rename "Action" "action" --rename "Action Date" "action_date" --rename "Associated Summary Text File Name" "summary_text_file_name" --rename "Questions/Comments" "questions_comments"
```

Now I insert all the text file summaries on my local machine into the database.

```zsh
sqlite-utils insert-files --text summaries.db files 74*/*.txt
```

This ends up saving the directory name and file name in the `files` tables's `path` column. We just want the file name since that's what is recorded in the `actions` table. The following fixes that for us.

```zsh
sqlite-utils convert summaries.db files path 'value.split("/")[1]'
```

Now we can link up the two tables

```zsh
sqlite-utils add-foreign-key summaries.db actions summary_text_file_name files path --ignore
```

and publish the database as an app on Heroku.

```zsh
datasette publish heroku summaries.db --name "llc" --tar "/usr/local/bin/gtar" --install=datasette-saved-queries
```

### Local Copies of the Data

Next I massage the data a little bit to allow for some simple analysis on the command line with [MIller](https://miller.readthedocs.io). The `awk` command drops the columns that don't have any data in them and the Miller command fills empty spots in the `Sponsor` and `Committee` columns, making the data bit [tidy](https://vita.had.co.nz/papers/tidy-data.pdf)-er.

```zsh
awk 'BEGIN{FS=OFS="\t"}{print $1, $2, $4, $5, $6, $7, $11, $12}' all_data.tsv | mlr --tsv fill-down -f Sponsor,Committee then cat > filled_metadata.tsv
```

Finally, to allow for easier reading of the spreadsheet by humans, the following awk command puts a blank line between each bill. `$5` refers to the `Bill Number` column.

```zsh
# insert blank lines between bills for pasting into final spreadsheet
awk 'BEGIN{FS="\t"} {cur=$5} NR>1 && cur!=prev {print ""} {prev=cur; print}' all_data.tsv > spaced_metadata.tsv
```

## Next Time

This wraps up the core workflow, so now it's time to move on how I actually use the Datasette web interface and SQL queries to check metadata quality.