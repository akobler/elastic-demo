
## Interactive multimedia archive powered by

<div style="display: flex; flex-direction: column; "> 
  <img src="images/elastic.png" style="margin-top: 100px;"/>

  <a href="https://twitter.com/andikobler" target="_blank" style="margin-top: 400px">@andikobler</a> <!-- .element: style="margin-top: 100px;" -->
</div>


## Agenda
* Architecture overview
* Demo
* Used elastic features
* Hurdles
* Some more tips



## recapp application

<img src="images/elastic_overview.png" style="width: 800px; margin-top: 100px;"/>


## Architecture

<img src="images/elastic_arch1.png" style="width: 500px; margin-top: 100px;"/>


## Architecture

<img src="images/elastic_arch2.png" style="width: 800px; margin-top: 100px;"/>


## Architecture

<img src="images/elastic_arch3.png" style="width: 800px; margin-top: 100px;"/>




## Demo

* debates of Grand Conseil de Valais:
  <a href="https://vs.recapp.ch" target="_blank">vs.recapp.ch</a>



### Used elastic features

* tour through used elastic features that
  * create a nice search experience with litte effort
  * caused AHA effects when I started with elastic


### Used elastic features: search
* simple search queries: match, prefix, filter

```json
GET /recapp_viewer/_search
{
    "query": {
          "match": {"segmentTranscription": "amendement"}
    }
}
```

```json
GET /_search
{ 
    "query": {
        "prefix" : { "segmentIndex" : "2016-09-" }
    }
}
```

* fuzzy search possible (to catch misspelling) <!-- .element: class="fragment" data-fragment-index="1"-->
  * can be confusing
  * we prefer: autocompletion and "did you mean"


### Used elastic features: search
* customized search analyzers
  * remove stop words
  * normalisation (Ä -> a, etc.)

```json
{
  "analyzer": {
    "searchAnalyzer": {
        "type": "custom",
        "filter": [
          "fr-stopwords",
          "de-stopwords",
          "lowercase",
          "asciifolding"
        ],
        "tokenizer": "letter"
    }
  }
}
``` 


### Used elastic features: search
* highlighting

```json
GET /recapp_viewer/_search
{
    "query": {
          "match": {"segmentTranscription": "Lesung"}
    },
    "highlight": {
        "fields": {
            "objectTitle.de": {},
            "objectTitle.fr": {}
        }
    }
}
```


### Used elastic features: search
* multi match query
  * search in title, description and in transcription field
  * <a href="https://www.elastic.co/guide/en/elasticsearch/reference/1.4/query-dsl-multi-match-query.html" target="_blank">multi match query</a>


### Used elastic features: analyzer
* multi fields
  * different indeces for different requirements
  * different analyzers on same document property: <a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-fields.html" target="_blank">Elastic multi fields</a>
  * adressing: `segmentTranscription.tags`

```json
{
  "segmentTranscription": {
    "type": "string",
    "analyzer": "searchAnalyzer",
    "fields": {
      "tags": {
        "type": "string",
        "analyzer": "tagAnalyzer"
      }
    }
  }
}
```


### Used elastic features: aggregation
* aggregations
  * list of speakers that spoke the most

```json
POST /recapp_viewer/_search?pretty
{
    "size": 0,
    "query": {
        "match": { "sessionDate" : "2016-09-01"}
    },
    "aggs" : {
        "top_speaker_durations" : {
            "terms" : {
                "field" : "deputyFullname",
                "order" : { "durations_per_speaker" : "desc" },
                "size" : 3
            },
            "aggs": {
                "durations_per_speaker" : {
                    "sum": { "field": "segmentDuration" }
                }
            }
        }
    }
}
```


### Used elastic features: aggregation
* significant terms (word cloud)

```json
POST recapp_viewer/_search?pretty
{
  "size": 0,
  "aggs": {
    "tags": {
      "significant_terms": {
        "field": "segmentTranscription.tags"
      }
    }
  },
  "query": {
   "match": { "sessionDate" : "2016-09-01"}
  }
}
```


### Used elastic features: aggregation
* correction suggestions for misspelled words

```json
GET recapp_viewer/_suggest
{
    "correction" : {
      "text" : "tourisus",
      
      "term" : {
        "field" : "segmentTranscription.tags",
        "min_doc_freq" : 2,
        "prefix_length": 1,
        "suggest_mode": "missing"
      }
    }
}
```


### Used elastic features: maintenance

* aliases for uninterrupted reindexing
  * existing documents in index cannot be modified
  * rebuild entire index and switch alias

```json
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "recapp_viewer_alias-1", "alias" : "recapp_viewer" }}
    ]
}
```
* <a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html" target="_blank">Elastic indices aliases</a>



### Hurdles

* in general no real big hurdles, just... :)

* do not think the classic DB-way - create a freshh elastic mindset
* get used to the query DSL
  * what query type to use? (term, match, prefix, filtered, ...)
* fields were automatically indexed, but I needed it raw
  * mark field with
```json
  "index": "not_analyzed"
```
* address nested properties
```json
  "index": "not_analyzed"
```
* MEMORY consumption


### Some more tips
* explore elastic with Sense Chrome Plugin
  * <a href="https://chrome.google.com/webstore/detail/sense-beta/lhjgkmllcaadmopgmanpapmpjgmfcfig" target="_blank">Sense Plugin</a>
* use /_analyze to investigate analyzers
```json
POST recapp_viewer/_analyze?field=segmentTranscription.tags
{
    "l'amendement cinquante cinq monsieur le conseiller haut-valais"       
}
```

* fetch more than 10 results
```json
POST /recapp_viewer/_search?pretty
{
    "size": 100
}
```


## Wrap up
* in-memory + indexes = fast
* POC system could be setup very quickly (1-2 days)
* gradually towards more sophisticated search features