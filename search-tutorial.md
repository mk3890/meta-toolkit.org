---
title: Search
category: tut
layout: page
order: 4
---

At the heart of MeTA lies its indexes. Every piece of data that MeTA's
algorithms operate on must first reside in an index. In this tutorial, we'll
walk you through how to create your own `inverted_index` and how to use your
`inverted_index` to create a search engine.

## Initially setting up the config file

By default, there are several fields at the top of your config file. These are
general settings that deal with paths:

{% highlight toml %}
prefix = "../data/"
libsvm-modules = "../deps/libsvm-modules/"
punctuation = "../data/sentence-boundaries/sentence-punctuation.txt"
stop-words = "../data/lemur-stopwords.txt"
start-exceptions = "../data/sentence-boundaries/sentence-start-exceptions.txt"
end-exceptions = "../data/sentence-boundaries/sentence-end-exceptions.txt"
function-words = "../data/function-words.txt"
{% endhighlight %}

This assumes you put the 20 newsgroups dataset in your `meta/data/` directory.
You can leave the other paths as they are; they'll work by default.

Next, we have a section for the corpus (20newsgroups) and its analyzer.

{% highlight toml %}
corpus = "line.toml"  # corpus-specific config, described in overview tutorial
dataset = "20newsgroups"
index = "20newsgroups-idx"

[[analyzers]]
method = "ngram-word"
ngram = 1
filter = "default-unigram-chain"  # ignores <s> tags for retrieval
{% endhighlight %}

If the executable needs to search the index, we can add the following to say
which ranker to use:

{% highlight toml %}
[ranker]
method = "bm25"
{% endhighlight %}

For all the premade ranking functions, you can additionally specify any of the
ranking function parameters:

{% highlight toml %}
[ranker]
method = "bm25"
k1 = 1.2
b = 0.75
k3 = 500
{% endhighlight %}

For a list of MeTA's ranking functions, see the
[index::ranker](http://meta-toolkit.org/doxygen/classmeta_1_1index_1_1ranker.html)
documentation.

## Using the example apps

Now that we know how to set up our config file, let's experiment with some
indexes. **It's possible to index and search a corpus without writing any
code!** In the root directory of MeTA, there are several example apps that deal
with creating indexes for search engines or information retrieval tasks:

 - `index.cpp`: indexes a corpus
 - `interactive_search.cpp`: searches for documents based on user input
 - `query_runner.cpp`: runs a list of queries specified by a file

Let's go through these one by one:

### Index

{% highlight bash %}
./index config.toml
{% endhighlight %}

You can then see it being indexed. After it's done, some stats are printed out,
such as number of documents, average document length, and number of unique
terms. If you run it again, it will just print out the stats since it detects
that the index has already been created.

### Interactive Search

{% highlight bash %}
./interactive-search config.toml
{% endhighlight %}

Running this will first create an index if one does not exist. Then, it will
prompt the user to type a query. The current analyzer is run on the text query
to turn it into a document, and then that document is used to search. The top
results are returned.

### Query Runner

For this app, we need to add a path to the query file.

{% highlight toml %}
query-path = "./path-to-queries.txt"
{% endhighlight %}

The query file contains one query per line, and they are run on the current
index. You can quickly make your own query file in the current directory if you
don't have one.

{% highlight bash %}
./query-runner config.toml
{% endhighlight %}

The top 10 documents for each query are then displayed along with the time
taken to run all the queries.

### Relevance judgements

Check out the
[index::ir_eval](http://meta-toolkit.org/doxygen/classmeta_1_1index_1_1ir__eval.html)
class if you have relevance judgements for your queries.

The format for the relevance judgements follow the format

```
queryid docid relevance
```

For two queries with a few judgements, you could have the following file:

~~~
0 6 2
0 7 1
0 45 2
0 21 1
0 99 1
1 6 1
1 4 1
1 89 2
~~~

This corresponds to query id 0 having docids 6, 7, 45, 21, and 99 being relevant
at levels 2, 1, 2, 1, and 1. For query id 1, the docids 6, 4, and 89 are
relevant at levels 1, 1, and 2. These lines can appear in any order. Docids are
assigned sequentially when indexing, or you can find docids by running search
yourself and creating manual relevance judgements. The different relevance
levels are used for NDCG scoring. If using something like precision or recall,
any nonzero scores are counted as relevant.

Note that it's not necessary to store irrelevant documents (relevance zero)
since any document not in the relevance judgements file is assumed to be
nonrelevant.

The path to the relevance judgements file is set in the config file as

{% highlight toml %}
query-judgements = "path/to/judgements.txt"
{% endhighlight %}

If a relevance file is found when running `./query-runner`, then its evaluation
results are displayed for each query.

## Writing code

Now we'll look at some examples of how to create and search an index. All this
code can be found in the demo apps as well.

### Creating an index

This is the simplest way to make an inverted index:

{% highlight cpp %}
auto config = cpptoml::parse_file(argv[1]);
auto idx = index::make_index<index::inverted_index>(*config);
{% endhighlight %}

This assumes that `argv[1]` is the path to the TOML config file, which is
standard MeTA practice.

Warning: this index does not have a cache! A cache is used when running queries
on the index. The cache saves frequently used `postings_data` objects so we
don't need to do a disk seek for every term while scoring.

Here are two different ways to specify a cache:

{% highlight cpp %}
// Create an inverted index using a DBLRU cache. The arguments forwarded to
//  make_index are the config file for the index and any parameters for the
//  cache. In this case, we set the maximum hash table size for the
//  dblru_cache to be 10000.
auto idx = index::make_index<index::dblru_inverted_index>(*config, 10000);

// Create an inverted index using a splay cache. The arguments forwarded
//  to make_index are the config file for the index and any parameters
//  for the cache. In this case, we set the maximum number of nodes in
//  the splay_cache to be 10000.
auto idx = index::make_index<index::splay_inverted_index>(*config, 10000);
{% endhighlight %}

### Searching an index

Now that we have an index, let's search it! First, we need to create a `ranker`:

{% highlight cpp %}
auto config = cpptoml::parse_file(argv[1]);
auto group = config.get_table("ranker");
if (!group)
    throw std::runtime_error{"\"ranker\" group needed in config file!"};
auto ranker = index::make_ranker(*group);
{% endhighlight %}

We can also manually specify our `ranker` instead of reading from the config
file:

{% highlight cpp %}
index::okapi_bm25 ranker{1.2, 0.75, 500.0};
{% endhighlight %}

We don't have to specify the arguments for the ranking parameters, but we did
anyway.

Finally, we are able to search our index with our ranker. Let's assume we used
`make_ranker` to create the ranker, so the `ranker` object is a
`std::unique_ptr` to the actual object.

{% highlight cpp %}
corpus::document query{doc_id{0}};
query.content("my query text");

auto ranking = ranker->score(*idx, query);
{% endhighlight %}

`ranking` is a sorted vector of `pair`s of (`doc_id`,`double`). By default, the
top 10 documents are returned. To return a different number, just add a third
parameter:

{% highlight cpp %}
auto ranking = ranker->score(*idx, query, 25);
{% endhighlight %}

Iterate through `ranking` and display your results!

## Writing your own ranker

Perhaps you'd like to experiment and write your own ranking function. There
are two main base classes to pick from when writing your own ranker:

- `index::ranking_function` should be used in nearly all cases. The main
  definition of your ranking function is then provided by the virtual
  function `score_one(const score_data&sd)`, where `score_data` is a struct
  that contains useful information for writing ranking functions. You can
  [read more about it here](http://meta-toolkit.org/doxygen/structmeta_1_1index_1_1score__data.html).

  Note that `score_one` is scoring a *single* matched query term for a
  document. The `index::ranking_function` base class abstracts away the
  iteration over the matched postings lists for you.

- `index::ranker` should only be used in the special cases where you need
  more control over the ranking process than `index::ranking_function` can
  provided. The function to override here is `rank()`, which must provide
  the search results directly itself by using the provided
  `ranker_context`.

In either case, you will also need to provide the following:

1. a public static `id` member of type `util::string_view` that is unique;
2. a `save(std::ostream&) const` function to save your ranker in binary
   format to the provided stream; and
3. a constructor from `std::istream&`.

We recommend using `io::packed` for stream reading/writing functions. Note
that the very first line of `save()` should be

{% highlight cpp %}
io::packed::write(out, id);
{% endhighlight %}

to ensure that your ranker's `id` is written first for reading later. After
that line, you should write any data to the stream that is necessary for
re-creating your ranker later.

In your constructor from `std::istream`, **the id will have already been
read from the stream**, so you should begin immediately reading the things
you wrote *after* the first line of `save()`.

### Registering Rankers

Once your ranker is written, you should register it with the toolkit to
enable creating it from a configuration file and loading it from disk. To
do so, register it somewhere in `main()` like this:

{% highlight cpp %}
meta::index::register_ranker<my_ranker>();
{% endhighlight %}

If you need to read parameters from the configuration group, you should
specialize the `make_ranker()` function as follows:

{% highlight cpp %}
namespace meta
{
namespace index
{
template <>
std::unique_ptr<ranker> make_ranker<my_ranker>(const cpptoml::table& config)
{
    // read config file
    // set default params if config options not found (for example)
}
}
}
{% endhighlight %}
