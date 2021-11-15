---
layout: post
title: "OpenRefine"
date: 2021-11-14 19:39:42.950781 -0600
---

_This was written for an assignment for LS 566, Metadata and the Semantic Web, at SLIS at the University of Alabama._

I'm blown away by [OpenRefine](https://openrefine.org). I had heard of it before when doing some self-directed, purposeless study of data analysis on my own. At the time, it seemed overwhelming or unintuitive, or maybe even unnecessary in a world where Excel and Google Sheets exist. The [Library Carpentry tutorial](https://librarycarpentry.org/lc-open-refine/) showed me that I had no need to be overwhelmed and that OpenRefine has some incredible capabilities out-of-the-box. It's an excellent introduction.

I enjoy the familiarity of working in a web browser much more than working in Excel, and I also find the syntax of the Google Refine Expression Language much more readable than Excel formulas. Working with dates in OpenRefine was a breeze in the tutorial. Facets (and the associated mass edit functionality) are so easy to use and far more convenient than trying to do something similar in Excel (or even Google Sheets) with filters and formulas. Years ago I started teaching myself regular expressions. I love how OpenRefine puts these front and center in creating transformations and uses a syntax familiar to people who have worked with regular expressions in programming languages like JavaScript and Ruby. I'm also really impressed by how the "Undo / Redo" feature works and that it not only documents your history with the dataset but also allows you to export your steps as JSON. That's brilliant!

After splitting out the different authors' names, the data was not in tidy format, so I did a little googling to see how to fill in that information and discovered the "Fill Down" feature which, according to this [web page](https://kb.refinepro.com/2012/03/fill-down-right-and-secure-way.html), can be unreliable. Experimenting with the GREL solution explained there will have to wait for another day.

The tutorial section on advanced functionality was mindblowing. I enjoyed dabbling with pulling in data through the [Crossref API](https://api.crossref.org/swagger-ui/index.html) and though I didn't understand everything about VIAF reconciliation, I thought it was really cool.

I'm excited to learn more about this fascinating tool.
