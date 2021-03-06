## Search and Dedupe for Mediachain

Entry Point                 |  Info
----------------------------|---------------------
mediachain-indexer-datasets | Download training and ingestion datasets.
mediachain-indexer-ingest   | Ingest media from local sources or Mediachain API, for search & dedupe.
mediachain-indexer-models   | Search and deduplication models. (Re-)Generate dedupe lookup tables.
mediachain-indexer-eval     | Hyper-parameter optimization and evaluation of models.
mediachain-indexer-web      | Search & dedupe REST API.
mediachain-indexer-test     | Tests and sanity checks.

## Contents

- Setup:
    - [Core Setup](https://github.com/mediachain/mediachain-indexer#core-setup)
    - [Quick Test](https://github.com/mediachain/mediachain-indexer#quick-test)
    - [Ingest Data](https://github.com/mediachain/mediachain-indexer#create-dataset-to-be-ingested)
    - [Query Server](https://github.com/mediachain/mediachain-indexer#query-server)
- Customizing:
    - [Customizing and Plugins](https://github.com/mediachain/mediachain-indexer#customizing-and-plugins)
- Internals:
    - [Detailed System Diagram](https://github.com/mediachain/mediachain-indexer#system-diagram)

## Getting Started

#### Core Setup

1) Install Elasticsearch. Version `2.3.2` required:

  - [General Instructions](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html).
  - OSX: `brew install homebrew/versions/elasticsearch22`	       	    `
  - Ubuntu:

```
        $ sudo add-apt-repository -y ppa:webupd8team/java
        $ sudo apt-get update
        $ sudo apt-get -y install oracle-java8-installer
        $ wget "https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/deb/elasticsearch/2.3.3/elasticsearch-2.3.3.deb"
        $ sudo dpkg -i  elasticsearch-2.3.3.deb
        $ sudo update-rc.d elasticsearch defaults
```

2) Launch Elasticsearch server:

`$ elasticsearch` or `$ sudo /etc/init.d/elasticsearch start`

3) Install Indexer:

```
$ pip install git+https://github.com/mediachain/mediachain-indexer.git
```

4) Run any of the entry points list above, to see sub-command details:

```
$ mediachain-indexer-ingest

USAGE: mediachain-indexer-ingest <function_name>

Available Functions:
ingest_bulk_blockchain.................. Ingest media from Mediachain blockchain.
ingest_bulk_gettydump................... Ingest media from Getty data dumps into Indexer.
config.................................. Print current environment variables.
```

5) Inspect environment variables and adjust as necessary:

```
$ mediachain-indexer-ingest config

### CONFIG:

## 1. Elasticsearch Settings:

  MC_NUMBER_OF_SHARDS_INT  = 1                           <INT>
  MC_NUMBER_OF_REPLICAS_INT= 0                           <INT>
  MC_INDEX_NAME            = 'getty_test'                <STR>
  MC_DOC_TYPE              = 'image'                     <STR>

  # One or more comma-separated RFC-1738 formatted URLs.
  # e.g. "http://user:secret@localhost:9200/,https://user:secret@other_host:443/production":
  MC_ES_URLS               = ''                          <STR>

## 2. Ingestion Settings:

  # Mediachain datastore settings:
  MC_DATASTORE_HOST        = None                        <STR>
  MC_DATASTORE_PORT_INT    = 10002                       <INT>

  # Getty key, for creating local dump of getty images:
  MC_GETTY_KEY             = ''                          <STR>

## 3. Settings for Automated Tests:

  MC_TEST_WEB_HOST         = 'http://127.0.0.1:23456'    <STR>
  MC_TEST_INDEX_NAME       = 'mc_test'                   <STR>
  MC_TEST_DOC_TYPE         = 'mc_test_image'             <STR>

## 4. Transactor settings:

  MC_TRANSACTOR_HOST       = '127.0.0.1'                 <STR>
  MC_TRANSACTOR_PORT_INT   = 10001                       <INT>
```


#### Quick Test

6) Run a basic sanity check:

```
$ mediachain-indexer-test sanity_check
```

#### Create Dataset to be Ingested

7) Collect small Getty testing dataset:

```
$ MC_GETTY_KEY="<your_getty_key>" mediachain-indexer-datasets getty_create_dumps
```

#### Ingest Directly, Bypassing Blockchain


8a) Ingest Getty dataset:

```
$ mediachain-indexer-ingest ingest_bulk_gettydump
```

#### Ingest via Blockchain

Alternatively:

8b) Prepare Indexer to accept blocks from the blockchain:

```
$ mediachain-indexer-ingest ingest_bulk_blockchain
```

9b) Initiate import to blockchain from media in directory `getty_small` using
[mediachain-client](https://github.com/mediachain/mediachain-client):

```
  python -m mediachain.cli.main -s localhost -p 10001 -e http://localhost:8000 ingest getty_small
```

#### Deduplicate & Prepare to Serve Queries

10) Deduplicate ingested media:

```
$ mediachain-indexer-models dedupe_reindex
```

11) Start REST API server:

```
$ mediachain-indexer-web web
```


#### Query Server

11) Query the [REST API](https://github.com/mediachain/mediachain/blob/master/rfc/mediachain-rfc-3.md#rest-api-overview):

Search by text:

```
$ curl "http://127.0.0.1:23456/search" -d '{"q":"crowd", "limit":5}'
{
    "next_page": null,
    "prev_page": null,
    "results": [
        {
            "_id": "getty_531746924",
            "_index": "getty_test",
            "_score": 0.08742375,
            "_source": {
                "artist": "Tristan Fewings",
                "caption": "CANNES, FRANCE - MAY 16:  A policeman watches the crowd in front of the Palais des Festival during the red carpet arrivals of the 'Loving' premiere during the 69th annual Cannes Film Festival on May 16, 2016 in Cannes, France.  (Photo by Tristan Fewings/Getty Images)",
                "collection_name": "Getty Images Entertainment",
                "date_created": "2016-05-16T00:00:00-07:00",
                "editorial_source": "Getty Images Europe",
                "keywords": "People Vertical Crowd Watching France Police Force Cannes Film Premiere Premiere Arrival Photography Film Industry Red Carpet Event Arts Culture and Entertainment International Cannes Film Festival Celebrities Annual Event Palais des Festivals et des Congres 69th International Cannes Film Festival Loving - 2016 Film",
                "title": "'Loving' - Red Carpet Arrivals - The 69th Annual Cannes Film Festival"
            },
            "_type": "image"
        }
    ]
}
```

Search by media content:

```
$ curl "http://127.0.0.1:23456/search" -d '{"q_id":"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==", "limit":5, "index_name":"mc_test", "doc_type":"mc_test_image"}'
{
    "next_page": null,
    "prev_page": null,
    "results": [
        {
            "_id": "getty_1234",
            "_index": "mc_test",
            "_score": 1.0,
            "_source": {
                "artist": "test",
                "caption": "test",
                "collection_name": "test",
                "date_created": "2016-05-31T17:41:06.929234",
                "editorial_source": "",
                "keywords": "test",
                "title": "Crowd of people walking"
            },
            "_type": "mc_test_image"
        }
    ]
}
```

Search by ID:

```
$ curl "http://127.0.0.1:23456/search" -d '{"q_id":"getty_1234", "limit":5, "index_name":"mc_test", "doc_type":"mc_test_image"}'
{
    "next_page": null,
    "prev_page": null,
    "results": [
        {
            "_id": "getty_1234",
            "_index": "mc_test",
            "_score": 1.0,
            "_source": {
                "artist": "test artist",
                "caption": "test caption",
                "collection_name": "test collection name",
                "date_created": "2016-06-01T15:21:57.796894",
                "editorial_source": "test editorial source",
                "keywords": "test keywords",
                "title": "Crowd of People Walking"
            },
            "_type": "mc_test_image"
        }
    ]
}
```

Duplicate lookup by ID:

```
$ curl "http://127.0.0.1:23456/dupe_lookup" -d '{"q_media":"getty_531746790"}'
{
    "next_page": null,
    "prev_page": null,
    "results": [
        {
            "_id": "getty_1234"
        }
    ]
}
```

Pass `{"help":1}` to any endpoint, to get a plain-text help string:

```
$ curl "http://127.0.0.1:23456/dupe_lookup" -d '{"help":1}' | head

Find all known duplicates of a media work.

Args - passed as JSON-encoded body:
    q_media:          Media to query for. See `Media Identifiers`.
    lookup_name:      Name of lookup key for the model you want to use. See `lookup_name` of `dedupe_reindex()`.
                      Note: must use 'dedupe_hsh' as lookup_name if v1_mode is True.
    incremental:      If True, only update clusters affected by newly ingested media. Otherwise, regenerate
                      all dedupe clusters. Note: the more records that are deduped simultaneously, the greater
                      the efficiency.
    include_self:     Include ID of query document in results.
[...]
```


## Customizing and Plugins

TODO: WIP.

The selection of model components and hyper-parameters can be customized.
Note that since the architecture has fixed, pre-defined connections between components,
as shown in the [Detailed System Diagram](https://github.com/mediachain/mediachain-indexer#system-diagram),
only the hyper-parameters to the components can be adjusted, not the connections between components.
This should not be a major limitation in practice.


#### System Components

Descriptor Generator Name      | Info
-------------------------------|--------------------------
VectorsBaseline                | Perceptual hashing baseline. Produces sparse or dense descriptors. Location: `mediachain.indexer.mc_models.VectorsBaseline`.
VectorsBaselineNG              | Simple visual bag of image words model. Produces sparse descriptors. Location: `mediachain.indexer.mc_models.VectorsBaselineNG`.
NeuralBaseline                 | (WIP) Baseline neural model. Produces dense descriptors. Location: `mediachain.indexer.mc_models.NeuralBaseline`.


Nearest Neighbor Lookup Name   | Info
-------------------------------|--------------------------
ElasticSearchNN                | ElasticSearch-backed nearest neighbor lookups. Accepts sparse descriptors. Location: `mediachain.indexer.mc_neighbors.ElasticSearchNN`.
AnnoyNN                        | (WIP) Annoy-backed nearest neighbor lookups. Accepts dense descriptors. Location: `mediachain.indexer.mc_neighbors.AnnoyNN`.


Re-Ranking Model Name          | Info
-------------------------------|--------------------------
ReRankingBasic                 | Simple re-ranking model that allows you to pass custom re-ranking equations at query time. Location: `mediachain.indexer.mc_rerank.ReRankingBasic`.


#### Passing Custom Hyper-Parameters at Runtime

Hyper-parameters of the model components can be configured prior to ingestion or on-the-fly at query time. Multiple separate pipelines can be created, and later identified by name when using API endpoints such as `/search` or '/dupe_lookup'.

Note that some hyper-parameters can only be set at ingestion time, prior to search or dedupe lookups. Add `'is_temp':true` to a model configuration to indicate that this is for one-off use and should be deleted after the current API call, e.g. for use in hyper-parameter optimization routines.

Below is an example configuration, which can be passed in via `MC_MODELS_JSON` and `MC_MODELS_FJSON`, or the endpoints `/search` and `/dupe_lookup`:

```
{"model_1":{"descriptors":{"name":"VectorsBaseline",
                           "params":{"hash_size":64,
                                     "use_hash":"phash",
                                     "patch_size":10,
                                     "max_patches":64
                                    }
                           },
            "neighbors":{"name":"ElasticSearchNN"},
            "rerank":{"name":"ReRankBasic",
                      "params":{"eq":"item['_score']"},
                     }
           }
"model_2":{"descriptors":{"name":"VectorsBaseline",
                          "params":{"hash_size":42,
                                     "use_hash":"dhash",
                                     "patch_size":5,
                                     "max_patches":32
                                    }
                           },
            "neighbors":{"name":"ElasticSearchNN"},
            "rerank":{"name":"ReRankBasic",
                      "params":{"eq":"item['_score']"},
                     }
           }
}
```

#### External Plugins

For model components listed above, you may pass just the class name, and the packages `mediachain.indexer.mc_models`, `mediachain.indexer.mc_rerank`, and `mediachain.indexer.mc_neighbors` will be searched.

External plugins may also be passed, by specifying the full import path for the class in the `"name"` field.

Note that some plugins may require separate pre-training on external data sources before they can be used by the Indexer.


## Detailed System Diagram

Modes of operation -versus- components involved in each mode of operation. Components links:
[Indexer](https://github.com/mediachain/mediachain-indexer),
[Core](https://github.com/mediachain/mediachain),
Frontend,
[Client](https://github.com/mediachain/mediachain-client).


```
                       INGESTION:                  DEDUPE:                      SEARCH:
                 _________ ^ __________    __________ ^ _____________   __________ ^ ______________
                /                      \  /                          \ /                           \

                +-----------------------------------------------------------------------------------+
              / |  +------------------------------------------------------------------------------+ |
             |  |  |                                 KNN Index                                    | |
             |  |  +------^--------------------^-----------+-------------^---------------+--------+ |
             |  |         |                    |           |             |               |          |
             |  |         ^                    ^           v             ^               v          |
             |  |   (Binary Codes)         (Media IDs) (Media IDs) (Binary Codes)     (Media IDs)   |
             |  |         |                    |           |             |            (Ranked by)   |
             |  |  +------+-----------+        |           |      +------+-----------+(Basic Scores)|
mc_neighbors<   |  |Feature Compacting|        |           |      |Feature Compacting|   |          |
   .py       |  |  +------^-----------+        |           |      +------^-----------+   |          |
             |  |         |                    |           |             |               |          |
             |  |         ^                    ^           v             ^               v          |
             |  |   (Descriptors)          (Media IDs) (Media IDs) (Descriptors)      (Media IDs)   |
             |  |         |                    |           |             |            (Ranked by)   |
             |  |         |                    |           |             |            (Basic Scores)|
             |  |  +------+-----------+        |           |      +------+-----------+   |          |
             |  |  |Generate Features |        |           |      |Generate Features |   |          |
              \ |  +------^-----------+        |           |      +------^-----------+   |          |
                |         |                    |           |             |               |          |
              / |         |              +-----+-----------v-------+     |               |          |
             |  |         |              |   Dedupe All-vs-All NN  |     |               |          |
             |  |         |              +----------+--------------+     |               |          |
             |  |         |                         |                    |               |          |
             |  |         ^                         v                    ^               v          |
             |  |    (Raw Media)          (IDs for Candidate Groups)(Raw Media/)      (Media IDs)   |
             |  |    (& Metadata)                   |               (Text Query)      (Ranked by)   |
             |  |         |                         |                    |            (Basic Scores)|
             |  |         |              +----------v--------------+     |    +----------v--------- |
             |  |         |              |  Dedupe Pairwise Model  |     |    |  Personalization  | |
             |  |         |              +----------+--------------+     |    +----------+--------+ |
             |  |         ^                         v                    ^               v          |
             |  |    (Raw Media)            (Pair IDs+Split/Merge)  (Raw Media/)    (Media IDs)     |
mc_models.py<   |    (& Metadata)                   |               (Text Query)    (& More Scores) |
             |  |         |                         |                    |               |          |
             |  |         |              +----------v--------------+     |    +----------v--------+ |
             |  |         |              |    Dedupe Clustering    |     |    | Search Re-Ranking | |
             |  |         |              +----------+--------------+     |    +----------+--------+ |
             |  |         ^                         v                    ^               v          |
             |  |    (Raw Media)            (Artefact-Linkage)      (Raw Media/)    (Ranked)        |
             |  |    (& Metadata)                   |               (Text Query)    (Media IDs)     |
             |  |         |                         |                    |               |          |
             |  |         |              +----------v--------------+     |    +----------v--------+ |
             |  |         |              |   Dedupe Staging        |     |    | Search Override   | |
             |  |         |              +--+----------------^--v--+     |    +----------+--------+ |
             |  |         |                 |                |  |        |               |          |
              \ |         |                 |                |  |        |          (Ranked)        |
                |         |                 |                |  |        |          (Media IDs)     |
              / |         |                 |                |  |        |               |          |
             |  |         |                 |                |  |  +-----+---------------v--------+ |
mc_web.py   <   |         ^                 v                ^  v  |Search & Autocomplete REST API| |
             |  |         |                 |                |  |  +-----^------^--+-----+--------+ |
             |  |         |                 |                |  |        |      |  |     |          |
              \ |    (Raw Media)        (Linkage)          (Lookups)(Raw Media/)|  |  (Ranked)      |
                |    (& Metadata)       (Artefact)         (Dupe)   (Text Query)|  |  (Media IDs)   |
              / |         |                 |                |  |        |      |  |     |          |
             |  |  +------+----------+      |                |  |        |   (Typeahead) |          |
mc_ingest.py<   |  | Media Ingestion |      v                ^  v        ^      ^  v     v          |
             |  |  +------^----------+      |                |  |        |      |  |     |          |
              \ |         |                 |                |  |        |      |  |     | -indexer-|
                +---------^-----------------v----------------^- v--------^------^- v-----+----------+
                          |                 |                |  |        |      |  |     |
                          ^                 v                ^  v        ^      ^  v     v
                     (copycat/gRPC)     (Artefact)         (JSON/)    (JSON/)  (JSON/) (JSON/)
                          |             (Linkage)          (REST)     (REST)   (REST)  (REST)
             /  +---------^-----------------v------------+   |  |        |      |  |     |
            |   |         |                 |            |   |  |        |      |  |     |
mediachain  |   |  +------+-----------------v---------+  |   |  |        |      |  |     |
 -client   <    |  |      Client Writer / Reader      |  |   ^  v        ^      ^  v     v
 reader &   |   |  +------^-----------------+---------+  |   |  |        |      |  |     |
  writer    |   |         |                 | --client-- |   |  |        |      |  |     |
             \  +---------^-----------------v------------+   |  |        |      |  |     |
                          |                 |               (JSON/)   (JSON/)  (JSON/) (JSON/)
                  (copycat/gRPC)     (copycat/gRPC)         (REST)    (REST)   (REST)  (REST)
                          ^                 v                ^  v        ^      ^  v     v
                          |                 |                |  |        |      |  |     |
             /  +---------^-----------------v------------+   |  |        |      |  |     |
            |   |         |                 |            |   |  |        |      |  |     |
mediachain  |   |  +------+-----------------v---------+  |   |  |        |      |  |     |
 -core     <    |  |            Transactors           |  |   |  |        |      |  |     |
(blockchain)|   |  +---------------^------------------+  |   |  |        |      |  |     |
            |   |                  |            --core-- |   |  |        |      |  |     |
             \  +------------------^---------------------+   |  |        |      |  |     |
                                   |                         ^  v        ^      ^  v     v
                             (copycat/gRPC)                 (Lookups)(Search)(Typeahead)(Results
                                   |                        (Dupe)       |      |  |    (Search)
             /  +------------------^----------------------+  |  |        |      |  |     |
            |   |                  |                      |  |  |        |      |  |     |
            |   |  +---------------+------------------+   |  |  |        |      |  |     |
mediachain <    |  |         Client Writer            |   |  ^  v        ^      ^  v     v
 -client    |   |  +---------------^------------------+   |  |  |        |      |  |     |
 writer     |   |                  |           --client-- |  |  |        |      |  |     |
             \  +------------------^----------------------+  |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
                             (Norm Media &)                 (Lookups)(Search)(Typeahead)(Results)
                             (Norm Metadata)                (Dupe)       |      |  |    (Search)
                                   |                         |  |        |      |  |     |
             /  +------------------^---------------------+   |  |        |      |  |     |
            |   |                  |                     |   |  |        |      |  |     |
            |   |  +---------------+-----------------+   |   |  |        |      |  |     |
            |   |  |          Pre-Ingestion          +--->--->-->        |      |  |     |
mc_ingest.py<   |  |          Dupe Check             <---<---+  v        ^      ^  v     v
            |   |  +---------------^-----------------+   |   |  |        |      |  |     |
            |   |                  |         --indexer-- |   |  |        |      |  |     |
             \  +------------------^---------------------+   |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
                                   ^                         ^  v        ^      ^  v     v
                             (Norm Media &)                 (Lookups)(Search)(Typeahead)(Results)
                             (Norm Metadata)                (Dupe)       |      |  |    (Search)
                                   |                         |  |        |      |  |     |
             /  +------------------^----------------------+  |  |        |      |  |     |
            |   |                  |                      |  |  |        |      |  |     |
            |   |  +---------------+------------------+   |  |  |        |      |  |     |
mc_normalize<   |  |   Field-Level Metadata Mapping   |   |  |  |        |      |  |     |
   .py      |   |  +---------------^------------------+   |  |  |        |      |  |     |
            |   |                  |                      |  |  |        |      |  |     |
             \  -------------------^----------------------+  |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
                             (Norm Media &)                  |  |        |      |  |     |
                             (Partial-Norm Metadata)         |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
             /  +------------------^----------------------+  |  |        |      |  |     |
            |   |                  |                      |  |  |        |      |  |     |
            |   |  +---------------+------------------+   |  |  |        |      |  |     |
mc_normalize<   |  |  Inter-field Metadata Decoding   |   |  |  |        |      |  |     |
   .py      |   |  +---------------^------------------+   |  |  |        |      |  |     |
            |   |                  |                      |  |  |        |      |  |     |
             \  -------------------^----------------------+  |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
                           (Media & Metadata)                |  |        |      |  |     |
               (From Bulk Uploader, Crawlers, or Web App)    |  |        |      |  |     |
                                   |                         |  |        |      |  |     |
                                   ^                         |  |        |      |  |     |
                                   /\                        |  |        |      |  |     |
                                  /  \                       |  |        |      |  |     |
                    ------------>/ OR \<---------------      |  |        |      |  |     |
                   /             \    /                \     |  |        |      |  |     |
                  |               \  /                 |     |  |        |      |  |     |
                  |                \/                  |     |  |        |      |  |     |
                  |                ^                   |     |  |        |      |  |     |
                  ^                |                   ^     ^  v        ^      ^  v     v
             (Norm Media&)   (Norm Media &)   (Norm Media&) (Lookups)(Search)(Typeahead)(Results)
             (Raw Metadata)  (Raw Metadata)   (Raw Metadata)(Dupe)       |      |  |    (Search)
                  |                |                   |     |  |        |      |  |     |
             /    |   +------------^---------------+   |     |  |        |      |  |     |
            |     |   |            |               |   |     |  |        |      |  |     |
            |     |   |  +---------+------------+  |   |     |  |        |      |  |     |
mc_datasets<      |   |  | Join To CompactSplit |  |   |     |  |        |      |  |     |
   .py      |     |   |  +----^---------^-------+  |   |     |  |        |      |  |     |
            |     |   |       |         |          |   |     |  |        |      |  |     |
             \    |   +-------^---------^----------+   |     |  |        |      |  |     |
                  |           |         |              |     |  |        |      |  |     |
                  ^           ^         ^              ^     ^  v        ^      ^  v     v
             (Norm Media)   (Raw) (Normalized)(Norm Media&) (Lookups)(Search)(Typeahead)(Results)
             (& Metadata)(Metadata) (Images)  (Raw Metadata)(Dupe)       |      |  |    (Search)
                  |           |         |              |     |  |        |      |  |     |
             /    |   +-------^---------^----------+   |     |  |        |      |  |     |
            |     |   |       |         |          |   |     |  |        |      |  |     |
            |     |   |  +----+---------+-------+  |   |     |  |        |      |  |     |
mc_datasets<      |   |  | Random-Access Cache  |  |   |     |  |        |      |  |     |
   .py      |     |   |  +----^---------^-------+  |   |     |  |        |      |  |     |
            |     |   |       |         |          |   |     |  |        |      |  |     |
             \    |   +-------^---------^----------+   |     |  |        |      |  |     |
                  |           |         |              |     |  |        |      |  |     |
                  ^           ^         ^              ^     ^  v        ^      ^  v     v
             (Norm Media)   (Raw) (Normalized)(Norm Media&) (Lookups)(Search)(Typeahead)(Results)
             (& Metadata)(Metadata) (Images)  (Raw Metadata)(Dupe)       |      |  |    (Search)
                  |           |         |              |     |  |        |      |  |     |
             /    |   +-------^---------^----------+   |     |  |        |      |  |     |
            |     |   |       |         |          |   |     |  |        |      |  |     |
            |     |   |  +----+---------+-------+  |   |     |  |        |      |  |     |
mc_crawler <      |   |  | Crawler Orchestrator |  |   |     |  |        |      |  |     |
   .py      |     |   |  +-------^----^---------+  |   |     |  |        |      |  |     |
            |     |   |          |     \           |   |     |  |        |      |  |     |
             \    |   +----------^------^----------+   |     |  |        |      |  |     |
                  |              |       \             |     |  |        |      |  |     |
                  ^              ^        ^            ^     ^  v        ^      ^  v     v
             (Norm Media)(Images &) (Images &)(Norm Media&) (Lookups)(Search)(Typeahead)(Results)
             (& Metadata)(Metadata) (Metadata)(Raw Metadata)(Dupe)       |      |  |    (Search)
                  |              |          \          |     |  |        |      |  |     |
             /    |         +----^----+ +----^----+    |     |  |        |      |  |     |
            |     |         |    |    | |    |    |    |     |  |        |      |  |     |
            |     |         |+---+---+| |+---+---+|    |     |  |        |      |  |     |
Lightweight<      |         ||Crawler|| ||Crawler||    |     |  |        |      |  |     |
 Proxies    |     |         |+-------+| |+-------+|    |     |  |        |      |  |     |
            |     |         |         | |         |    |     |  |        |      |  |     |
             \    |         +---------+ +---------+    |     |  |        |      |  |     |
                  |                                    |     |  |        |      |  |     |
                  ^                                    ^     ^  v        ^      ^  v     v
               (Norm Media)                   (Norm Media&) (Lookups)(Search)(Typeahead)(Results)
               (& Metadata)                   (Raw Metadata)(Dupe)       |      |  |    (Search)
                  |                                    |     |  |        |      |  |     |
                  |                               +----^-----^--v--------^------^--v-----v----------+
             /    |                               |    |     |  |        |      |  |     |          |
            |     |                               |  +-+-----+--v--------+------+--v-----v--------+ |
 -frontend  |     |                               |  |      Javascript / HTML Web App             | |
mediachain <      |                               |  +-^-----^--+--------^------^--+-----+--------+ |
            |     |                               |    |     |  |        |      |  |     |-frontend-|
            |     |                               +----^-----^--v--------^------^--v-----v----------+
             \    |                                    |     |  |        |      |  |     |
                  ^                                    ^     ^  v        ^      ^  v     v
               (Raw Images)                        (Upload) (Lookups)(Search)(Typeahead)(Results)
               (& Metadata)                        (Images) (Dupe)       |      |  |    (Search)
                  ^                                    ^     ^  v        ^      |  |     v
                  |                                    |     |  |        |      |  |     |
             / +--+----------------------------+  +----+-----+--v--------+------+--v-----v----------+
  End User <   |        Bulk Uploader App      |  |              End-User Web Browser               |
             \ +-------------------------------+  +-------------------------------------------------+
```
