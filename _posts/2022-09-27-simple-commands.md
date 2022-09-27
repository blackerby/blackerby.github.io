---
layout: post
title: "Bacon Saving Commands in Google Sheets and the shell"
date: 2022-09-27
---

I have the good fortune to be a remote metadata intern for the Law Library of Congress this fall. As part of the project we are generating a large number of text files of bill summaries from the 74th Congress and naming them according to a well-structured schema, e.g., `74_2_s81_19360624.txt` contains a summary of S. 81 from the 74th Congress, Second Session, whose last action took place on June 25, 1936.

We are listing these file names in a column in a spreadsheet on Google Sheets. Last night, I realized I mislabeled the session for bills whose last action took place in 1935. I needed to change the filenames in the spreadsheet and rename the files themselves on my hard drive. It took a little while to research how to do this, but it was ultimately far less tedious than doing it manually. Here's what I did.

## Google Sheets

`=IF(REGEXMATCH(M14, "1935"), REPLACE(M14,4,1,"1"), M14)`

If the text in cell `M14` matches the text `1935`, replace the fourth character in the string with the character `1`, otherwise return the original text. `REGEXMATCH` may be overkill here, but it's all I could think of late last night.

## Shell

This was a little more involved.

```zsh
#!/usr/bin/env zsh

# https://askubuntu.com/questions/406313/change-multiple-filenames-by-replacing-a-character
# https://stackoverflow.com/questions/6355011/replacing-one-char-with-many-chars-with-using-tr
for f in *1935*; do mv -v "$f" $(echo "$f" | sed 's/2/1/' ); done
```

I relied heavily on the two links in the comments above the code, but as I understand it,

```zsh
for f in *1935*
```

loops through all the files in the directory that contain `1935` in their name.

```zsh
mv -v "$f"  $(echo "$f" | sed 's/2/1/' )
```

does a verbose rename of each of those files, using `sed` to substitute the first occurrence of the digit `2` with the digit `1`.

## Expediting the Copy and Paste Workflow

The assignment involves a lot of copying and pasting from a PDF into a text file. Here's a command I came up with to expedite that process. Before running it, the text from the PDF must be on the paste board.

```zsh
FILE=74_2_s1476_19350507.txt && pbpaste > $FILE && code $FILE
```

This assigns a file name to the shell variable `FILE`, puts the contents of the paste board into that file, and then opens the same file in my text editor. It saves some mousework and keystrokes.

## Final Thoughts

I've read up on this kind of stuff for years, so it's nice to finally have a practical application for it. I am eager to learn more, especially about using shell scripting.
