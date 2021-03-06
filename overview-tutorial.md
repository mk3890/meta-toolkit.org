---
title: MeTA System Overview
layout: page
category: tut
order: 1
---

Our task is to improve upon and complement the current body of open source
machine learning and information retrieval software. The existing environment of
this open source software tends to be quite fragmented.

There is rarely a single location for a wide variety of algorithms; a good
example of this is the [LIBLINEAR](http://www.csie.ntu.edu.tw/~cjlin/liblinear/)
software package for SVMs. While this is the most cited of the open source
implementations of linear SVMs, it focuses solely on kernel-less methods. If
presented with a nonlinear classification problem, one would be forced to find a
different software package that supports kernels (such as the same authors'
[LIBSVM](http://www.csie.ntu.edu.tw/~cjlin/libsvm/) package). This places an
undue burden on the researchers---not only are they required to have a detailed
understanding of the research problem at hand, but they are now forced to
understand this fragmented nature of the open source software community, find
the appropriate tools in this mishmash of implementations, and compile and
configure the appropriate tool.

Even when this is all done, there is the problem of data formatting---it is
unlikely that the tools have standardized upon a single input format, so a
certain amount of "data munging" is now required. This all detracts from the
actual research task at hand, which has a marked impact on the speed of
discovery.

**We want to address these issues.** In particular, we want to provide a
unifying framework for text indexing and analysis methods, allowing researchers
to quickly run controlled experiments. We want to modularize the feature
generation, instance representation, data storage formats, and algorithm
implementations; this will allow for researchers to make seamless transitions
along any of these dimensions with minimal effort.

## How does MeTA store data?

All processed data is stored in a `disk_index`. Currently, we have two
index types: `forward_index` and `inverted_index`. The former is keyed by
document IDs, and the latter is keyed by term IDs.

 - `forward_index` is used for applications such as topic modeling and
   most classification tasks
 - `inverted_index` is used to create search engines, or do classification with
   *k*-nearest-neighbor

Since each MeTA application takes an index as input, all processed data is
interchangeable between all the components. This also gives a great advantage to
classification: **MeTA supports out-of-core classification by default!** If
your dataset is small enough (like most other toolkits assume), you can
load an index into a `dataset` object to keep it all in memory without
sacrificing any speed.

Index usage is explained in much more detail in the [Search
Tutorial]({{site.baseurl}}/search-tutorial.html), though it might make more sense
to read the about [Filters and
Analyzers]({{site.baseurl}}/analyzers-filters-tutorial.html) first.

The main toml configuration file has a few options related to indexing.

{% highlight toml %}
uninvert = false          # default
indexer-ram-budget = 1024 # **estimated** total RAM budget for indexing in MB
                          # always set this lower than your physical RAM!
{% endhighlight %}

The key `uninvert` (if set to true) will create a `forward_index` by uninverting
an `inverted_index`. This is useful to set to true if you will be using both
indexes. If you'll only do classification or topic modeling, setting this to
false will not create an inverted index unless one is explicitly used.

The `indexer-ram-budget` is *approximately* how much total memory is
allocated during the indexing process. If there are four threads and
`indexer-ram-budget = 4096`, then each thread will store 1GB of processed
data before it flushes it to disk. Note that this limit is currently
heuristic-based and thus is not a hard limit (though the system does its
best to adhere to it).

## Corpus input formats
In the main toml configuration file, there are two parameters to specify a
corpus:

{% highlight toml %}
dataset = "dataset-name"
corpus = "name.toml"
{% endhighlight %}

The `dataset` key names a folder under the path `prefix` where the corpus
data resides. The key `corpus` is the path to the corpus configuration
file, relative to the corpus path.  Usually, we simply name the corpus
config file by the corpus input format type.  For example, if we are using
a line corpus (described below), we would say `corpus = "line.toml"`. The
file `line.toml` (or whatever it's called) contains corpus-specific
settings. Refer to the corpus-specific documentation below for what it
should contain.

There are currently four corpus input formats (and you can find examples of all
of these in the data directory for your MeTA checkout once you have built
the toolkit at least once).

### Line Corpus
A `line_corpus` consists of one or two files:

 - `line.toml`: the corpus configuration file
 - `corpusname.dat`: each document appears on one line.
 - `corpusname.dat.labels`: optional file that includes the class or label of
   the document on each line, again corresponding to the order in
   `corpusname.dat`. These are the labels that are used for the
   classification tasks.

Below is an example of what `line.toml` might look like:

{% highlight toml %}
type = "line-corpus"
encoding = "utf-8" # optional; this is the default
num-docs = 1000    # optional
metadata = [{name = "path", type = "string"}]    # metadata explained later
{% endhighlight %}

### GZipped Line Corpus
A `gz_corpus` is simlar to a `line_corpus`, but each of its files are
compressed using gzip compression:

 - `gz.toml`: the corpus configuration file
 - `corpusname.dat.gz`: a compressed version of `corpusname.dat`
 - `corpusname.dat.labels.gz`: a compressed version of
   `corpusname.dat.labels`

Below is an example of what `gz.toml` might look like:

{% highlight toml %}
type = "gz-corpus"
encoding = "utf-8" # optional; this is the default
num-docs = 1000    # required
metadata = [{name = "path", type = "string"}]    # metadata explained later
{% endhighlight %}

### File Corpus
In a `file_corpus`, each document is its own file. Your corpus
configuration file (typically `file.toml`) should specify a `list` key to
point MeTA at a list of all of the documents in your corpus. This file,
`list-full-corpus.txt` (where list is whatever you set the `list` key to in
your configuration file; typically just the name of the corpus again)
contains (on each line) a required label for each document followed by the
path to the file on disk (separated by a tab or a space). If the documents
don't have specified class labels, just precede the line with `[none]` or
similar.

For example, your `file.toml` might look like:

{% highlight toml %}
type = "file-corpus"
list = "dataset"
encoding = "utf-8" # optional; this is the default
num-docs = 1000    # optional
metadata = [{name = "path", type = "string"}]    # metadata explained later
{% endhighlight %}

and the file `dataset-full-corpus.txt` might look like:

~~~
class-label	relative/path/to/document01.txt
class-label2	relative/path/to/document02.txt
class-label	relative/path/to/document03.txt
class-label3	relative/path/to/document04.txt
~~~

### LIBSVM Corpus
If only being used for classification, MeTA can also take libsvm-formatted
input to create a `forward_index`. A `libsvm_corpus` contains the following
files:

 - `libsvm.toml`: the corpus configuration file
 - `corpus.dat`: the LIBSVM-formatted data file

The `libsvm.toml` file is very straightforward:

{% highlight toml %}
type = "libsvm-corpus"
{% endhighlight %}

## Metadata

The file `metadata.dat` may optionally accompany a corpus. The `metadata`
parameter in the corpus config file explains the schema of the file. Each line
in `metadata.dat` corresponds to the ordered documents in the corpus, and values
are tab-separated. Below is a simple `line.toml` corpus config file and
`metadata.dat` file.

{% highlight toml %}
# file: line.toml
type = "line"
num-docs = 4
metadata = [{name = "path", type = "string"},
            {name = "published", type = "uint"},
            {name = "conference", type = "string"}]
{% endhighlight %}

Below are the contents of `metadata.dat`:

~~~
papers/acl-short.txt	2015	ACL
papers/sigir-submission.txt	2016	SIGIR
papers/sigir-accepted.txt	2015	SIGIR
old-papers/nips.txt	2007	NIPS
~~~

Possible types are `int`, `uint`, `double`, and `string`. Note that the example
`path` variable isn't necessarily a path to an actual file (since this is a line
corpus), but it may be useful to the user to remember where the original files
came from.

To access the metadata of an indexed document, simply call

{% highlight cpp %}
doc_id d_id = // ...
auto mdata = idx->metadata(d_id);
std::cout << *mdata.get<std::string>("field-name") << std::endl;
{% endhighlight %}

If you have no metadata, simply do not specify a `metadata` key in your
corpus configuration file.

## Datasets

There are several public datasets that we've converted to the `line_corpus`
format:

 - [20newsgroups](https://meta-toolkit.org/data/20newsgroups.tar.gz), originally
   from [here](http://qwone.com/~jason/20Newsgroups/).
 - [IMDB Large Movie Review Dataset](https://meta-toolkit.org/data/imdb.tar.gz),
   originally from [here](http://ai.stanford.edu/~amaas/data/sentiment/).
 - Any [libsvm-formatted
   dataset](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/) can be
   used to create a `forward_index`.

## Tokenizers, Filters, and Analyzers

MeTA processes data with a system of tokenizers, filters and analyzers before it
is indexed.

**Tokenizers** come first. They define how to split a document's content into
tokens. Some examples are:

 - `icu_tokenizer`: converts documents into streams of tokens by following the
   unicode standards for sentence and word segmentation.
 - `character_tokenizer`: converts documents into streams of single characters.

**Filters** come next, and can be chained together. They define ways that text
can be modified or transformed. Here are some examples of filters:

 - `length_filter`: this filter accepts tokens that are within a certain length
   and rejects those that are not.
 - `icu_filter`: applies an [ICU](http://userguide.icu-project.org/)
   transliteration to each token in the sequence.
 - `list_filter`: this filter either accepts or rejects tokens based on a list.
   For example, you could use a stopword list and reject stopwords.
 - `porter2_filter`: this filter transforms each token according to the
   [Porter2 English
   Stemmer](http://snowball.tartarus.org/algorithms/english/stemmer.html) rules.

**Analyzers** operate on the output from the filter chain and produce token
counts from documents. Here are some examples of analyzers:

 - `ngram_word_analyzer`: collects and counts sequences of *n* words (tokens)
   that have been filtered by the filter chain
 - `ngram_pos_analyzer`: same as `ngram_word_analyzer`, but operates on
   part-of-speech tags from MeTA's CRF
 - `tree_analyzer`: collects and counts occurrences of parse tree structural
   features, currently the only known implementation of work described in [this
   paper](http://web.engr.illinois.edu/~massung1/files/icsc-2013.pdf).
 - `libsvm_analyzer`: converts a libsvm `line_corpus` into MeTA format.

For a more detailed intro, see [Filters and
Analyzers]({{site.baseurl}}/analyzers-filters-tutorial.html).

## Unit tests

We're using [Bandit](http://banditcpp.org/), which is configured to run all
of MeTA's tests. You may run

{% highlight bash %}
./unit-test --reporter=spec
{% endhighlight %}

to execute the unit tests while displaying output from failed tests.

The file `unit_test.cpp`, takes various tests as parameters. For example,

{% highlight bash %}
./unit-test --only=[inverted-index] --reporter=spec
{% endhighlight %}

only runs tests with the string `[inverted-index]` in their description. The
parameter `--reporter=spec` gives more detailed output on which unit tests are
being run.

Please note that you must have a valid configuration file (`config.toml`) in the
project root with `prefix = ../data/`, since many unit tests create input based
on paths stored there.

## Support

You should use the [MeTA forum](https://forum.meta-toolkit.org) for user
support. If you have a bug or feature request, see below.

## Bugs

On GitHub, create an issue and make sure to label it as a bug. Please be as
detailed as possible (including information such as your operating system
version, version of MeTA, configuration files, etc.). Including a code
snippet would be very helpful if applicable.

You many also add feature requests the same way, though of course the
developers will have their own priority for which features are addressed first.

## Contribute

Forks are certainly welcome. When you have something you'd like to add, submit
a pull request and a developer will approve it after review.

We're also looking for more core developers. If you're interested, don't
hesitate to contact us!

## Citing MeTA

For now, please cite the homepage (<https://meta-toolkit.org>) and note which
revision you used. Once (hopefully) we have a paper, you would be able to cite
that.

## Developers

[**Sean Massung**](http://web.engr.illinois.edu/~massung1/) started writing MeTA
in May 2012 as a tool to help with his senior thesis at UIUC. He is currently a
computer science PhD student there (since 2013), working with [Professor
ChengXiang Zhai](http://www.cs.uiuc.edu/homes/czhai/). His research interests
include information retrieval, natural language processing, and education.
Specifically, he is currently studying how non-native English speakers write in
English. Can we determine their native language and offer grammatical error
corrections?

[**Chase Geigle**](https://chara.cs.illinois.edu/sites/cgeigle/) is a computer
scientist and educator at the University of Illinois at Urbana-Champaign, where
he is currently pursuing his PhD (starting in 2013). His research interests
include text mining and machine learning, particularly when applied to the
domain of education. He is a member of the [Text Information Management and
Analysis (TIMAN) group](http://sifaka.cs.uiuc.edu/ir/) where he studies under
the supervision of [Dr. ChengXiang Zhai](http://www.cs.uiuc.edu/~czhai/). He is
a lead developer on the MeTA toolkit and a self-described C++ aficionado.

## License

MeTA is dual-licensed under the [MIT
License](http://opensource.org/licenses/MIT) and the [University of
Illinois/NCSA Open Source License](http://opensource.org/licenses/NCSA).
These are free, permissive licenses, meaning it is allowed to be included
in proprietary software.
