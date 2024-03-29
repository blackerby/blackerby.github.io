---
layout: post
title: "A High Level Metadata Workflow Overview"
date: 2022-11-22
---
The semester is winding down and so is my time as a remote metadata intern for the Law Library of Congress.  I ended up learning a ton about Unix text processing workhorses like aspell, awk, diff, find, grep, sed, and GNU Emacs as well as newer tools like [Miller](https://miller.readthedocs.io/en/latest/) and [Datasette](https://datasette.io).  I incorporated these tools into several shell scripts to expedite my workflow, which is built around a python script I wrote to extract specific text from the PDF I was working with using regular expressions.  I also did a fair amount of work with Google Sheets and SQLite, and towards the end of the internship I even dabbled with the Google Drive API.

My plan now is to write a series of posts detailing the steps in this workflow and my use of the tools mentioned above.  Below is a high-level overview of my workflow as it stands towards the end of the semester.

1. Extract text with the `prep.sh` script.
2. Paste initial metadata into Google Sheets.
- This initial metadata comes from running the command `mlr --c2t --headerless-csv-output cat 74_2_{PAGE_NUM}.csv | pbcopy`.
- `74_2_{PAGE_NUM}.csv` is one of the files generated by the the `prep.sh` script.
3. Review each of the text files generated by `prep.sh` (this includes running `aspell`).
4. Run `cleanup.sh` to delete backup files, move the working directory to its final location, and copy the names of the text files to the clipboard for pasting into Google Sheets.
5. Run `update_tsv_data.sh` to create local copies of the metadata, load the metadata and text files into a SQLite database, and publish that database to Heroku.

Stay tuned for more posts about each of the above steps, starting with the `prep.sh` script.
