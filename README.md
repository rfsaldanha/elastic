elastic
=======



[![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![R-check](https://github.com/ropensci/elastic/workflows/R-check/badge.svg)](https://github.com/ropensci/elastic/actions?query=workflow%3AR-check)
[![cran checks](https://cranchecks.info/badges/worst/elastic)](https://cranchecks.info/pkgs/elastic)
[![rstudio mirror downloads](https://cranlogs.r-pkg.org/badges/elastic?color=E664A4)](https://github.com/metacran/cranlogs.app)
[![cran version](https://www.r-pkg.org/badges/version/elastic)](https://cran.r-project.org/package=elastic)
[![codecov.io](https://codecov.io/github/ropensci/elastic/coverage.svg?branch=master)](https://codecov.io/github/ropensci/elastic?branch=master)

**A general purpose R interface to [Elasticsearch](https://www.elastic.co/elasticsearch/)**


## Elasticsearch info

* [Elasticsearch home page](https://www.elastic.co/elasticsearch/)
* [API docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)


## Compatibility

This client is developed following the latest stable releases, currently `v7.10.0`. It is generally compatible with older versions of Elasticsearch. Unlike the [Python client](https://github.com/elastic/elasticsearch-py#compatibility), we try to keep as much compatibility as possible within a single version of this client, as that's an easier setup in R world.

## Security

You're fine running ES locally on your machine, but be careful just throwing up ES on a server with a public IP address - make sure to think about security.

* Elastic has paid products - but probably only applicable to enterprise users
* DIY security - there are a variety of techniques for securing your Elasticsearch installation. A number of resources are collected in a [blog post](https://recology.info/2015/02/secure-elasticsearch/) - tools include putting your ES behind something like Nginx, putting basic auth on top of it, using https, etc.

## Installation

Stable version from CRAN


```r
install.packages("elastic")
```

Development version from GitHub


```r
remotes::install_github("ropensci/elastic")
```


```r
library('elastic')
```

## Install Elasticsearch

* [Elasticsearch installation help](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)

__w/ Docker__

Pull the official elasticsearch image

```
docker pull elasticsearch
```

Then start up a container

```
docker run -d -p 9200:9200 elasticsearch
```

Then elasticsearch should be available on port 9200, try `curl localhost:9200` and you should get the familiar message indicating ES is on.

If you're using boot2docker, you'll need to use the IP address in place of localhost. Get it by doing `boot2docker ip`.

__on OSX__

+ Download zip or tar file from Elasticsearch [see here for download](https://www.elastic.co/downloads), e.g., `curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.0-darwin-x86_64.tar.gz`
+ Extract: `tar -zxvf elasticsearch-7.10.0-darwin-x86_64.tar.gz`
+ Move it: `sudo mv elasticsearch-7.10.0 /usr/local`
+ Navigate to /usr/local: `cd /usr/local`
+ Delete symlinked `elasticsearch` directory: `rm -rf elasticsearch`
+ Add shortcut: `sudo ln -s elasticsearch-7.10.0 elasticsearch` (replace version with your version)

You can also install via Homebrew: `brew install elasticsearch`

> Note: for the 1.6 and greater upgrades of Elasticsearch, they want you to have java 8 or greater. I downloaded Java 8 from here http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html and it seemed to work great.

## Upgrading Elasticsearch

I am not totally clear on best practice here, but from what I understand, when you upgrade to a new version of Elasticsearch, place old `elasticsearch/data` and `elasticsearch/config` directories into the new installation (`elasticsearch/` dir). The new elasticsearch instance with replaced data and config directories should automatically update data to the new version and start working. Maybe if you use homebrew on a Mac to upgrade it takes care of this for you - not sure.

Obviously, upgrading Elasticsearch while keeping it running is a different thing ([some help here from Elastic](http://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html)).

## Start Elasticsearch

* Navigate to elasticsearch: `cd /usr/local/elasticsearch`
* Start elasticsearch: `bin/elasticsearch`

I create a little bash shortcut called `es` that does both of the above commands in one step (`cd /usr/local/elasticsearch && bin/elasticsearch`).

## Initialization

The function `connect()` is used before doing anything else to set the connection details to your remote or local elasticsearch store. The details created by `connect()` are written to your options for the current session, and are used by `elastic` functions.


```r
x <- connect(port = 9200)
```

> If you're following along here with a local instance of Elasticsearch, you'll use `x` below to 
do more stuff.

For AWS hosted elasticsearch, make sure to specify path = "" and the correct port - transport schema pair.


```r
connect(host = <aws_es_endpoint>, path = "", port = 80, transport_schema  = "http")
  # or
connect(host = <aws_es_endpoint>, path = "", port = 443, transport_schema  = "https")
```

If you are using Elastic Cloud or an installation with authentication (X-pack), make sure to specify path = "", user = "", pwd = "" and the correct port - transport schema pair.


```r
connect(host = <ec_endpoint>, path = "", user="test", pwd = "1234", port = 9243, transport_schema  = "https")
```

<br>

## Get some data

Elasticsearch has a bulk load API to load data in fast. The format is pretty weird though. It's sort of JSON, but would pass no JSON linter. I include a few data sets in `elastic` so it's easy to get up and running, and so when you run examples in this package they'll actually run the same way (hopefully).

I have prepare a non-exported function useful for preparing the weird format that Elasticsearch wants for bulk data loads, that is somewhat specific to PLOS data (See below), but you could modify for your purposes. See `make_bulk_plos()` and `make_bulk_gbif()` [here](https://github.com/ropensci/elastic/blob/master/R/docs_bulk.r).

### Shakespeare data

Elasticsearch provides some data on Shakespeare plays. I've provided a subset of this data in this package. Get the path for the file specific to your machine:




```r
shakespeare <- system.file("examples", "shakespeare_data.json", package = "elastic")
# If you're on Elastic v6 or greater, use this one
shakespeare <- system.file("examples", "shakespeare_data_.json", package = "elastic")
shakespeare <- type_remover(shakespeare)
```

Then load the data into Elasticsearch:

> make sure to create your connection object with `connect()`


```r
# x <- connect()  # do this now if you didn't do this above
invisible(docs_bulk(x, shakespeare))
```

If you need some big data to play with, the shakespeare dataset is a good one to start with. You can get the whole thing and pop it into Elasticsearch (beware, may take up to 10 minutes or so.):

```sh
curl -XGET https://download.elastic.co/demos/kibana/gettingstarted/shakespeare_6.0.json > shakespeare.json
curl -XPUT localhost:9200/_bulk --data-binary @shakespeare.json
```

### Public Library of Science (PLOS) data

A dataset inluded in the `elastic` package is metadata for PLOS scholarly articles. Get the file path, then load:


```r
if (index_exists(x, "plos")) index_delete(x, "plos")
plosdat <- system.file("examples", "plos_data.json", package = "elastic")
plosdat <- type_remover(plosdat)
invisible(docs_bulk(x, plosdat))
```

### Global Biodiversity Information Facility (GBIF) data

A dataset inluded in the `elastic` package is data for GBIF species occurrence records. Get the file path, then load:


```r
if (index_exists(x, "gbif")) index_delete(x, "gbif")
gbifdat <- system.file("examples", "gbif_data.json", package = "elastic")
gbifdat <- type_remover(gbifdat)
invisible(docs_bulk(x, gbifdat))
```

GBIF geo data with a coordinates element to allow `geo_shape` queries


```r
if (index_exists(x, "gbifgeo")) index_delete(x, "gbifgeo")
gbifgeo <- system.file("examples", "gbif_geo.json", package = "elastic")
gbifgeo <- type_remover(gbifgeo)
invisible(docs_bulk(x, gbifgeo))
```

### More data sets

There are more datasets formatted for bulk loading in the `sckott/elastic_data` GitHub repository. Find it at <https://github.com/sckott/elastic_data>


<br>

## Search

Search the `plos` index and only return 1 result


```r
Search(x, index = "plos", size = 1)$hits$hits
#> [[1]]
#> [[1]]$`_index`
#> [1] "plos"
#> 
#> [[1]]$`_type`
#> [1] "_doc"
#> 
#> [[1]]$`_id`
#> [1] "0"
#> 
#> [[1]]$`_score`
#> [1] 1
#> 
#> [[1]]$`_source`
#> [[1]]$`_source`$id
#> [1] "10.1371/journal.pone.0007737"
#> 
#> [[1]]$`_source`$title
#> [1] "Phospholipase C-β4 Is Essential for the Progression of the Normal Sleep Sequence and Ultradian Body Temperature Rhythms in Mice"
```

Search the `plos` index, and query for _antibody_, limit to 1 result


```r
Search(x, index = "plos", q = "antibody", size = 1)$hits$hits
#> [[1]]
#> [[1]]$`_index`
#> [1] "plos"
#> 
#> [[1]]$`_type`
#> [1] "_doc"
#> 
#> [[1]]$`_id`
#> [1] "813"
#> 
#> [[1]]$`_score`
#> [1] 5.18676
#> 
#> [[1]]$`_source`
#> [[1]]$`_source`$id
#> [1] "10.1371/journal.pone.0107638"
#> 
#> [[1]]$`_source`$title
#> [1] "Sortase A Induces Th17-Mediated and Antibody-Independent Immunity to Heterologous Serotypes of Group A Streptococci"
```

## Get documents

Get document with id=4


```r
docs_get(x, index = 'plos', id = 4)
#> $`_index`
#> [1] "plos"
#> 
#> $`_type`
#> [1] "_doc"
#> 
#> $`_id`
#> [1] "4"
#> 
#> $`_version`
#> [1] 1
#> 
#> $`_seq_no`
#> [1] 4
#> 
#> $`_primary_term`
#> [1] 1
#> 
#> $found
#> [1] TRUE
#> 
#> $`_source`
#> $`_source`$id
#> [1] "10.1371/journal.pone.0107758"
#> 
#> $`_source`$title
#> [1] "Lactobacilli Inactivate Chlamydia trachomatis through Lactic Acid but Not H2O2"
```

Get certain fields


```r
docs_get(x, index = 'plos', id = 4, fields = 'id')
#> $`_index`
#> [1] "plos"
#> 
#> $`_type`
#> [1] "_doc"
#> 
#> $`_id`
#> [1] "4"
#> 
#> $`_version`
#> [1] 1
#> 
#> $`_seq_no`
#> [1] 4
#> 
#> $`_primary_term`
#> [1] 1
#> 
#> $found
#> [1] TRUE
```


## Get multiple documents via the multiget API

Same index and different document ids


```r
docs_mget(x, index = "plos", id = 1:2)
#> $docs
#> $docs[[1]]
#> $docs[[1]]$`_index`
#> [1] "plos"
#> 
#> $docs[[1]]$`_type`
#> [1] "_doc"
#> 
#> $docs[[1]]$`_id`
#> [1] "1"
#> 
#> $docs[[1]]$`_version`
#> [1] 1
#> 
#> $docs[[1]]$`_seq_no`
#> [1] 1
#> 
#> $docs[[1]]$`_primary_term`
#> [1] 1
#> 
#> $docs[[1]]$found
#> [1] TRUE
#> 
#> $docs[[1]]$`_source`
#> $docs[[1]]$`_source`$id
#> [1] "10.1371/journal.pone.0098602"
#> 
#> $docs[[1]]$`_source`$title
#> [1] "Population Genetic Structure of a Sandstone Specialist and a Generalist Heath Species at Two Levels of Sandstone Patchiness across the Strait of Gibraltar"
#> 
#> 
#> 
#> $docs[[2]]
#> $docs[[2]]$`_index`
#> [1] "plos"
#> 
#> $docs[[2]]$`_type`
#> [1] "_doc"
#> 
#> $docs[[2]]$`_id`
#> [1] "2"
#> 
#> $docs[[2]]$`_version`
#> [1] 1
#> 
#> $docs[[2]]$`_seq_no`
#> [1] 2
#> 
#> $docs[[2]]$`_primary_term`
#> [1] 1
#> 
#> $docs[[2]]$found
#> [1] TRUE
#> 
#> $docs[[2]]$`_source`
#> $docs[[2]]$`_source`$id
#> [1] "10.1371/journal.pone.0107757"
#> 
#> $docs[[2]]$`_source`$title
#> [1] "Cigarette Smoke Extract Induces a Phenotypic Shift in Epithelial Cells; Involvement of HIF1α in Mesenchymal Transition"
```

## Parsing

You can optionally get back raw `json` from `Search()`, `docs_get()`, and `docs_mget()` setting parameter `raw=TRUE`.

For example:


```r
(out <- docs_mget(x, index = "plos", id = 1:2, raw = TRUE))
#> [1] "{\"docs\":[{\"_index\":\"plos\",\"_type\":\"_doc\",\"_id\":\"1\",\"_version\":1,\"_seq_no\":1,\"_primary_term\":1,\"found\":true,\"_source\":{\"id\":\"10.1371/journal.pone.0098602\",\"title\":\"Population Genetic Structure of a Sandstone Specialist and a Generalist Heath Species at Two Levels of Sandstone Patchiness across the Strait of Gibraltar\"}},{\"_index\":\"plos\",\"_type\":\"_doc\",\"_id\":\"2\",\"_version\":1,\"_seq_no\":2,\"_primary_term\":1,\"found\":true,\"_source\":{\"id\":\"10.1371/journal.pone.0107757\",\"title\":\"Cigarette Smoke Extract Induces a Phenotypic Shift in Epithelial Cells; Involvement of HIF1α in Mesenchymal Transition\"}}]}"
#> attr(,"class")
#> [1] "elastic_mget"
```

Then parse


```r
jsonlite::fromJSON(out)
#> $docs
#>   _index _type _id _version _seq_no _primary_term found
#> 1   plos  _doc   1        1       1             1  TRUE
#> 2   plos  _doc   2        1       2             1  TRUE
#>                     _source.id
#> 1 10.1371/journal.pone.0098602
#> 2 10.1371/journal.pone.0107757
#>                                                                                                                                                _source.title
#> 1 Population Genetic Structure of a Sandstone Specialist and a Generalist Heath Species at Two Levels of Sandstone Patchiness across the Strait of Gibraltar
#> 2                                     Cigarette Smoke Extract Induces a Phenotypic Shift in Epithelial Cells; Involvement of HIF1α in Mesenchymal Transition
```

## Known pain points

* On secure Elasticsearch servers:
  * `HEAD` requests don't seem to work, not sure why
  * If you allow only `GET` requests, a number of functions that require
  `POST` requests obviously then won't work. A big one is `Search()`, but
  you can use `Search_uri()` to get around this, which uses `GET` instead
  of `POST`, but you can't pass a more complicated query via the body

## Screencast

A screencast introducing the package: <a href="https://vimeo.com/124659179">vimeo.com/124659179</a>

## Meta

* Please [report any issues or bugs](https://github.com/ropensci/elastic/issues)
* License: MIT
* Get citation information for `elastic` in R doing `citation(package = 'elastic')`
* Please note that this package is released with a [Contributor Code of Conduct](https://ropensci.org/code-of-conduct/). By contributing to this project, you agree to abide by its terms.

[![rofooter](https://ropensci.org/public_images/github_footer.png)](https://ropensci.org)
