---
layout: post
title: "get_summaries.py"
date: 2022-11-24
---
I wrote this lengthy post, the third in my [metadata workflow series](https://blackerby.github.io/workflow-series/), in [org-mode](https://orgmode.org/), which helpfully generates a table of contents on exporting to markdown. I hope it's handy for navigation.

I also hope to convert this file to [ipynb format](https://ipython.org/notebook.html) so interested folks can try the code out on their own.  The code is currently published on [GitHub](https://github.com/blackerby/llc_pdf_etl_workflow/blob/main/get_summaries.py).

# Table of Contents

1.  [Environment Setup](#org74c4f0f)
    1.  [The Shebang Line](#orgbbe4192)
    2.  [Necessary Libraries](#orgf23cd14)
2.  [Constants](#orgb285bec)
    1.  [(No) Magic Numbers](#org2c5204a)
    2.  [The Month Dictionary](#org9ae5b31)
3.  [Big, Ugly Regular Expressions](#orga8bf1f8)
    1.  [The Summary Header Pattern](#orgdb31e26)
    2.  [The Date Pattern](#orgbf64b5f)
4.  [Functions](#orgc42185f)
    1.  [Argument Parsing](#org6c807de)
    2.  [Naming the CSV data file](#orge6f6ba9)
    3.  [Getting the Data](#orgb3d0a3d)
    4.  [Dates](#org9048eeb)
    5.  [Output File Names](#orgf4b7948)
    6.  [Writing out the files](#org2e11b5f)
        1.  [Writing the CSV File](#orgdc07a37)
        2.  [Writing the Summary Files](#org09e3803)
5.  [Putting it all together](#org6866364)
6.  [Up next](#orgf96c958)

# Environment Setup

## The Shebang Line

We add a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line in case we want to make this file executable.
```python
#!/usr/bin/env python
```

## Necessary Libraries

Now we import the libraries we need.
```python
import argparse
import sys
import re
import csv
from pathlib import Path

from more_itertools import chunked
from PyPDF2 import PdfReader
```
The role of each library is explained below:

-   **`argparse`:** handles command-line argument parsing
    -   In this script, those command line arguments are the path to the PDF file we are processing, the page we will start processing from, and an optional argument for where we will stop processing.
-   **`sys`:** handles a variety of interactions between python and its host system
    -   In this script, it is used once: to exit the program in case of a fatal error
-   **`re`:** is python's regular expression library
    -   In this script, it does the bulk of the work.
-   **`csv`:** is python's library for dealing delimited data files.
    -   In this script, it is used to write metadata from the bill summaries to a csv file.
-   **`from pathlib import Path`:** imports the Path object for dealing with file paths
-   **`from more_itertools import chunked`:** imports the chunked function from the `more_itertools` library
    -   `more_itertools` is not part of a standard python distribution and needs to be installed using `pip`.
-   **`from PyPDF2 import PdfReader`:** is a third-party library for dealing PDF files that be installed using `pip`.

# Constants

## (No) Magic Numbers

There are three numbers that play key roles in the script.  To avoid having [magic numbers](https://en.wikipedia.org/wiki/Magic_number_(programming)), we make them constants with descriptive names.
```python
ELEMENT_COUNT = (
    7  # full header, bill type, bill number, sponsor(s), action date, committee, text
)
CONGRESS_POSITION = 2
SESSION_POSITION = 3
```
-   **`ELEMENT_COUNT`:** is adequately explained by the comment in the above code block.
-   **`CONGRESS_POSITION`:** refers to starting index of the number in the name of the file passed to the script that refers to the congress (e.g., 74th) the summaries were written for.
-   **`SESSION_POSITION`:** is like `CONGRESS_POSITION`, but for the session of Congress (e.g., second).


<a id="org9ae5b31">Back to Top</a>

## The Month Dictionary

We use a python dictionary to map between names (or abbreviations, or bad OCR text) of months as found in the bill summaries and two character strings of digits that will be used in naming the files containing the extracted summaries.
```python
MONTHS = {
    "January": "01",
    "February": "02",
    "Feb.": "02",
    "March": "03",
    "Mar.": "03",
    "April": "04",
    "Apr.": "04",
    "May": "05",
    "June": "06",
    "July": "07",
    "August": "08",
    "September": "09",
    "October": "10",
    "November": "11",
    "December": "12",
    "Alay": "05", # bad OCR, should be "May"
}
```

# Big, Ugly Regular Expressions

The script relies on two regular expressions to do its work.  They're big, they're ugly, they're poorly tested, and they match a lot more than they are intended to match, but this script would be useless without them.  In a later draft of this post, and a later version of the script, I will rewrite the regular expressions with the [verbose flag](https://docs.python.org/3/library/re.html#re.X), which they obviously need.  I decided to demonstrate the version of the regexes the script actually uses.

## The Summary Header Pattern

This regular expression (usually) matches the first line of a bill summary and [groups (see the second row of the table)](https://en.wikipedia.org/wiki/Regular_expression#Examples) parts of the line that will be extracted into a starter CSV metadata file.  The abundance of punctuation options has to do primarily with inconsistency in how OCR recognizes the punctuation characters in the original text, but there is also some inconsistency in how the original text is punctuated.
```python
HEADER_PATTERN = re.compile(
    r"(((?:S|H)\.? ?(?:R\.?)? (?:J\.? Res\. ?)?)(\w{1,5})\.? ((?:M(?:rs?|essrs)\.) .+?)(?:[;,:])? (\w{1,9} \d{1,2}[.,] \d{4})[.—]? ?\n?(?:\((['0-9a-zA-Z ]+)\))?(?:\.|.+\.|:|.+:)?)",
    re.MULTILINE,
)
```
The regex captures the bill type, bill number, sponsors, date of introduction, committee, and text of the bill summary.  It also captures the entire first line of the summary.

## The Date Pattern

This regular expression is intended to match dates like the following: `January 16, 1936`.  The year is optional.
```python
DATE_PATTERN = re.compile(r"([JFMASOND][a-z]{2,8}\.?) (\d{1,2})[-—.,;: ]( \d{4})?")
```

# Functions

Now we turn our attention to the functions that will be called in the main body of the script.

## Argument Parsing

Here, we use the [argparse](https://docs.python.org/3/library/argparse.html) package to handle three command line arguments: the name of the file, the page to start from, and the page to end on.
```python
def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "file_path", help="path to a bill digest PDF, e.g., ../74_2.pdf"
    )
    parser.add_argument(
        "start_page",
        help="PDF page number (not printed page number) for where program should start working",
        type=int,
    )
    parser.add_argument(
        "end_page",
        help="last PDF page number (not printed page number) program should process",
        type=int,
        nargs="?",
    )
    return parser.parse_args()
```

## Naming the CSV data file

The CSV file we generate needs a meaningful name, e.g., `74_2_7_8.csv`, that is, the Congress and Session (`74_2`, which we get from the file stem of the PDF file we're processing, followed by the start page and the end page (if one is specified).
```python
def name_output_file(file_stem, start_page, end_page):
    if end_page:
        return "_".join([file_stem, str(start_page), str(end_page)]) + ".csv"
    else:
        return "_".join([file_stem, str(start_page)]) + ".csv"
```

## Getting the Data

This is where most of the work happens.  The function could and should probably be broken out into some smaller functions, but it works.  See the comments in the function definition for details, and the [more<sub>itertools</sub> chunked documentation](https://more-itertools.readthedocs.io/en/stable/api.html#more_itertools.chunked) for information about the use of that function.
```python
def extract_summaries_and_metadata(file_path, start_page, end_page):
    # Set up some empty containers for our data
    metadata = []
    summaries = []
    text_file_names = []
    
    # Instantiate the PdfReader object
    reader = PdfReader(file_path)
    # Get the file stem, e.g., 74_2
    file_stem = file_path.stem
    
    # Handle the presence of absence of an ending page number
    if end_page == None:
        end = start_page + 1
    else:
        end = end_page
    
    # Iterate over page numbers
    for i in range(start_page, end):
        # Get the text from each page
        page_text = reader.pages[i].extract_text()
    
        # Find the first line that matches HEADER_PATTERN from above
        first_header_pos = re.search(HEADER_PATTERN, page_text).start()
        # Get the text from that point on
        page_text = page_text[first_header_pos:]
        # Split the text based on the header pattern
        raw_summaries = re.split(HEADER_PATTERN, page_text)[1:]
        # Group the summaries
        raw_summaries = list(chunked(raw_summaries, ELEMENT_COUNT))
    
    for item in raw_summaries:
        # Give a name to each grouped piece of each summary
        header, bill_type, bill_number, sponsor, date, committee, text = item
        # Normalize the bill type formatting
        formatted_bill_type = bill_type.strip()
        # Normalize bill type for the summary text file name
        lower_bill_type = bill_type.lower().replace(" ", "").replace(".", "")
        # Concatenate the summary header and summary text
        summary = header + text
        # Add a row of metadata to the list
        metadata.append(
            [formatted_bill_type, bill_number, sponsor, date, committee]
        )
        # Add a summary to the list
        summaries.append(summary)
        # Add the name of the text file to the list
        text_file_names.append(f"{file_stem}_{lower_bill_type}{bill_number}")

    # return a tuple of lists of data
    return (metadata, summaries, text_file_names)
```

## Dates

Here's an example of a bill summary in the original PDF:  
![Bill Summary](/assets/bill.png)

And here's what it looks like after processing:  
> S. 1019. Mr. King; January 15, 1935 (District of Columbia).  
As passed by the Senate, February 12, and referred to House Committee on the District of Columbia. February 15, 1935:  
Requires real estate brokers and salesmen in the District of Columbia (including nonresidents but excluding auctioneers, banks, trust companies, building and loan associations or land-mortgage or farm-loan associations) to secure annual licenses from a real estate commission hereby established (composed of three members—two appointed by the Commissioners and the assessor ex-officio). Applicants for licenses must be 21 years old, able to read and write English and must give proof of trustworthiness and competence in the business (not required if applicant has 2 years experience as a broker, etc., or in connection with real estate business in the District of Columbia). Bond required—$2,500 for brokers, $1,000 for salesmen—running to the District of Columbia; and a fee for broker’s license of $25 or $5 for salesman’s license. License may be revoked by the Commission, upon its own motion or on a verified complaint, after a hearing, for any fraudulent or dishonest dealing or conviction of a crime involving fraud or dishonesty. Revocation of a broker’s license suspends the license of every salesman under him. 

You'll notice that there are multiple dates specified.  The date from the first line goes into the CSV file, but we want the latest action to go into the name of the text file for each summary.  The function below handles that.
```python
def format_latest_action_dates(summaries):
    actions = []
    date_tags = []
    intro_dates = []
    
    for summary in summaries:
        # Break the summary text into lines
        lines = summary.splitlines()
        # Get the date on the first
        intro_dates.append(re.findall(DATE_PATTERN, lines[0])[0])
        # If there is more than one line in the summary, store the second line, otherwise, store an empty string
        if len(lines) > 1:
            actions.append(lines[1])
        else:
            actions.append("")
    
    # Process the second line of each summary
    for action in actions:
        # Get the whole date the bill was introduced
        intro_date = intro_dates[actions.index(action)]
        # Get the year from that date
        intro_year = intro_date[2]
        # Find all the dates in the second line of the summary
        dates = re.findall(DATE_PATTERN, action)
        # If there are dates in the second line, handle that
        if dates:
            # Sometimes no years are specified in the second line of the summary, as in the example above
            default_year = dates[0][2] or intro_year
            date = list(dates[-1])
            if date[2] == "":
                date[2] = default_year
            # Default to the year the bill was introduced
          else:
              date = list(intro_date)
          # Convert the month text to a two digit numerical string
          date[0] = MONTHS[date[0]]
          # Create the date string that will go into the name of the text file
          date_tag = f"{date[2].strip()}{date[0]}{date[1].zfill(2)}"
          date_tags.append(date_tag)
    
      return date_tags
```

## Output File Names

Here's how we name the text file associated with each summary.
```python
def format_output_files(text_file_names, date_tags):
    # If the number of text file base names and the number of date tags doesn't match, we've got a problem
    if len(text_file_names) != len(date_tags):
        print("Text files names list and date tags list are not the same length.")
        sys.exit(1)
    # Zip the text file base names and date strings together, join them with "_", and add the ".txt" extension
    filenames = map(
        lambda name: "_".join(name) + ".txt", list(zip(text_file_names, date_tags))
    )
    return list(filenames)
```

## Writing out the files

Here's how we write the files to disk.

### Writing the CSV File

This is decently straightforward thanks to the [csv](https://docs.python.org/3/library/csv.html) module in the python standard library.
```python
def write_metadata(metadata, output_file_name, congress):
    headers = [
        "congress",
        "session",
        "pl_num",
        "bill_type",
        "bill_number",
        "sponsor",
        "committee",
        "action_red",
        "summary_version_code",
        "report_number",
        "action",
        "action_date",
        "associated_summary_file",
        "questions_comments",
    ]

    with open(output_file_name, "w") as f:
        writer = csv.DictWriter(f, fieldnames=headers)
        writer.writeheader()
        for row in metadata:
            writer.writerow(
                {
                    "congress": congress,
                    "session": "",
                    "pl_num": "",
                    "bill_type": row[0],
                    "bill_number": row[1],
                    "sponsor": row[2],
                    "committee": row[4] or "N/A",
                    "action_red": "",
                    "summary_version_code": "",
                    "report_number": "",
                    "action": "Introduced",
                    "action_date": row[3],
                    "associated_summary_file": "",
                    "questions_comments": "",
                }
            )

```

### Writing the Summary Files

This is the most straightforward function in the script.  Essentially, we iterate over the summaries and match them up with the text file names we created.
```python
def write_summaries(summaries, text_file_names):
    for i in range(len(summaries)):
        with open(text_file_names[i], "w") as f:
            f.write(summaries[i])
```

# Putting it all together

Now we actually do what we set out to do.
```python
if __name__ == "__main__":
    args = parse_args()
    file_path = args.file_path
    start_page = args.start_page
    start_page_index = start_page - 1
    end_page = args.end_page

    file_path = Path(file_path)
    file_stem = file_path.stem
    output_file_name = name_output_file(file_stem, start_page, end_page)

    congress = file_stem[:CONGRESS_POSITION]
    session = file_stem[SESSION_POSITION]

    metadata, summaries, text_file_names = extract_summaries_and_metadata(
        file_path, start_page_index, end_page
    )

    action_dates = format_latest_action_dates(summaries)
    output_file_names = format_output_files(text_file_names, action_dates)

    write_metadata(metadata, output_file_name, congress)
    write_summaries(summaries, output_file_names)

```

# Up next

Now it's time for the manual work.  In the next post, I'll detail how I use GNU Emacs to review each text file and Google Sheets to input additional metadata about each bill.

