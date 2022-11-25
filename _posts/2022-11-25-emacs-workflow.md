---
layout: post
title: "Emacs Workflow"
date: 2022-11-25
---
We've run [prep.sh](https://blackerby.github.io/2022/11/23/prep.html) (and with it, [get_summaries.py](https://blackerby.github.io/2022/11/24/get-summaries.html)), so now we've got a directory full of text files to review.  This is where [GNU Emacs](https://www.gnu.org/software/emacs/) comes in.

Initially, I was doing all of this work on the command line, using [iTerm2](https://iterm2.com/) on my Mac.  I had a [script](https://github.com/blackerby/llc_pdf_etl_workflow/commit/9416a664bf2e1b946060c4e7a1110edaf75c4557#diff-946a9298fc35569de8840dcfaabf1ab6a8e6334318c81b9a0b0590c2fad56184) that ran [aspell](http://aspell.net/) on each text file before opening [Neovim](https://neovim.io/) for each file for manual review.  It worked fine, but it wasn't very flexible, and to be honest, I've always wanted to become proficient in Emacs.  This project presented me with a great opportunity to it a daily driver.

In the interest of keeping this post shorter than the last one, I'll describe the workflow I landed on and not all the steps along the way.

# Finding the Directory
Within Emacs, I use `C-x d` to navigate to the directory created by `prep.sh`, e.g., `74_2_199`.  This opens [Dired](https://www.gnu.org/software/emacs/manual/html_node/emacs/Dired.html), the Emacs Directory Editor, where I next run `dired-omit-mode` with `C-x M-o` to hide uninteresting files (e.g., backup files that end with `~` and `.bak`).

# Editing the Files
Now, I use `n` and `p` to navigate between the listed files, hitting the enter key to open the file I'd like to edit.  Within each file, I run `M-x ispell` with aspell as the backend for some automated spell-checking.  I remove any weird entities introduced by OCR fix up line and paragraph breaks.

There are two really important elements of this step:
1. Enter metadata about bill actions besides introduction into the spreadsheet
2. Make sure the name of the file corresponds to the date of the latest action on the bill.

Putting the data into Google Sheets is pretty straightforward, except for a couple of steps at the end of this element of the workflow that I'll detail later in the post.

If I need to edit the name of the file, I do this back in the Dired buffer using the `R` key, making sure to change the name in the spreadsheet as well.

# Finalizing the spreadsheet data
Before wrapping up this element of the workflow, I do two things:

1. I type the date into the spreadsheet as written in the original document, e.g., May 1, 1936, but I eventually reformat it into [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) using `Format > Number` in Google Sheets for some easier processing in SQLite later in the workflow.
2. I fill down the `Session` column with the formula `=if(year(L2088)=1935, 1, 2)` (the L column is the `Action Date` column).

# Cleaning up and next steps
Finally within Emacs I run the script `cleanup.sh`, which will be explained in the next post, using the `M-x !` key with the argument `../cleanup.sh`.
