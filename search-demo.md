---
title: Search Engine and Rankers
layout: page
category: demo
order: 1
---

The MeTA search and ranking demo is currently located at
[http://timan103.cs.illinois.edu/search-app/](http://timan103.cs.illinois.edu/search-app/).

The search engine is over Wikipedia articles with article titles as metadata.
Selecting a ranker from the dropdown on the left of the search bar changes the
ranking function that is used to score each document. The numbers to the right
of each document are the score that the ranker has given it. Clicking on a link
will open the respective Wikipedia article.

The demo code can be found [online on
GitHub](https://github.com/meta-toolkit/metapy-demos) in the [MeTA toolkit
organization](https://github.com/meta-toolkit). It makes use of MeTA's
(still experimental) Python bindings to create a simple web application.

![Search demo](search-demo.png)
