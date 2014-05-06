---
layout: post
title: "coursework: parsing and indexing Wikipedia articles"
date: 2014-05-06 12:50
comments: true
tags: java, wikipedia, apache lucene, information retrieval
---

In last winter semester I took Information Retrieval course in the university. During the course we have implemented several projects.
One of them was to use [Apache Lucene](http://lucene.apache.org/core/) to index, rank and query Wikipedia articles.
A small example of Wikipedia data can be found [here](http://en.wikipedia.org/wiki/Wikipedia:Database_download#English-language_Wikipedia).
The application consists of four main modules:
 - Indexer: this module handles indexing documents
 - Parser: this module parses the XML data file, creates documents(Page object) and send it to Indexer for indexing
 - Searcher: this module handles searching i.e querying
 - WikipediaRetriever: this is a wrapper module for all functionalities. You should run this module if you want to use provided functionalities of the program. After running this module you will have 2 options: first one is indexing the data and the second option is to query in already created index.

[Click to see the source code of the application](https://github.com/ElvinEfendi/wikipedia-retrieval)

In the repository I've also put ready Lucene index to the folder called "small_lucene_index". One can give path to this folder as an argument to
the program and to search.
