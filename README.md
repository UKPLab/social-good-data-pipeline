# Data Collection and Annotation Pipeline for Social Good Projects: Walkthrough

In this document we introduce the practical site of our data collection and annotation pipeline. This walkthrough gives an idea of how to use our pipeline to design an end-to-end project, but each part can also be integrated separately in other projects. Throughout this document we use COVID-19 case number prediction as a sample case study.

## Citation

Please use the following citation:

    TBD

> **Abstract:** Vast amounts of data are generated during crisis events through both formal and informal sources, and this data can be used to make a positive impact in all phases of crisis events. However, collecting and annotating data quickly and effectively in the face of crises is a challenging task. Crises require quick, robust, and efficient annotation to best respond to unfolding events. Data must be accessed and aggregated across different platforms and sources, and annotation tools must be able to utilize this data effectively.
>
> This work describes an architecture built for rapid collection and annotation of data from multiple sources which can then be built into machine learning and data analysis models. We extract data from social media via multiple systems for Twitter data collection, as well as building architecture for the collection of news articles from diverse sources. These can then be input into the INCEpTION annotation framework, which has been adapted to allow for easy management of multiple annotators, aiming to improve functionality to facilitate the application of citizen science. This allows us to rapidly prototype new annotation schema across a diverse array of data sources, which can then be deployed for machine learning. As a use case, we explore annotation of COVID-19 related Tweets and news articles for case prediction.

**Contact Person:** Kevin Stowe, <stowe@ukp.informatik.tu-darmstadt.de>

<https://www.ukp.tu-darmstadt.de>

<https://www.tu-darmstadt.de>

## News Data Crawling and Preparation

For news crawling we provide API endpoints for two data sources, namely (1) [The GDELT Project](https://www.gdeltproject.org) and (2) [News API](https://newsapi.org). These endpoints are hosted on UKP servers and therefore require zero setup.

### News Source Comparison

For the most part both of our GDELT and News API endpoints provide the same functionality. The key differences are in data quality and quantity. GDELT provides free access to three month of data for world wide news and blogs while News API crawls handpicked news outlets and has restrictions depending on different [access tiers](https://newsapi.org/pricing). The only additional difference is a country selection filter for the GDELT endpoint.

Both provide filters for language, size, date range, and source selection detailed below.

_GDELT endpoint parameters and filters:_

| Name | Required | Description | Example |
| --- | --- | --- | --- |
| language | required | ISO 639-1 language code. Currently English, German, and Italian are supported. | `en` |
| size | optional | Upper limit of crawled news sites. Maximum value is 250. | `75` (Default: `250`) |
| occurrence_type | required | `must` and `should` are supported. I case of `must` an exact match of the query has to appear in the document. | `must` |
| search_query | required | GDELT search term. If occurrence type `should` or a simple query string is used, GDELT has additional (undocumented) constraints. Words with dashes or tokens with less then 4 characters require quotes around them, e.g. `"COVID-19"`, or `"TU Darmstadt"`. These constraints are not present for occurrence type `must`. For occurrence type `should` and simple query strings the `(a OR b)` and `-` exclusion syntax can be used. | `("COVID-19" OR lockdown OR mask)`|
| date_from | optional | A lower bound date matching the `yyyy-MM-dd'T'HH:mm:ss` format. GDELT provides data 3 month back. | `2020-06-28T09:41:00` |
| date_to | optional | An upper bound date matching the `yyyy-MM-dd'T'HH:mm:ss` format. GDELT provides data 3 month back. | `2020-06-29T09:41:00` |
| country | optional | FIPS country code. Available countries listed here: <http://data.gdeltproject.org/api/v2/guides/LOOKUP-COUNTRIES.TXT> | `us` |
| source | optional | Search a specific domain. A wildcard `*` can be set as a _prefix_, e.g. `*zeitung.de`. No other wildcard placement is allowed! | `tagesschau.de` |

_News API endpoint parameters and filters:_

| Name | Required | Description | Example |
| --- | --- | --- | --- |
| api_key | required | News API authentication key obtained by registering on <https://newsapi.org>. Has to be passed as an authorization header of type `Basic`. | `Basic abcdefghijklmnopqrstuvwxyz123456` |
| language | required | ISO 639-1 language code. Currently English, German, and Italian are supported. | `en` |
| size | optional | Upper limit of crawled news sites. Maximum value is 100. | `75` Default: `100` |
| occurrence_type | required | `must` and `should` are supported. I case of `must` an exact match of the query has to appear in the document. | `must` |
| search_query | required | News API search term. `AND`, `OR`, and `NOT` keywords and parenthesis for grouping can be used. See [News API documentation](https://newsapi.org/docs/endpoints/everything) parameter `q` for more options. | `COVID-19 AND (lockdown OR masks)`|
| date_from | optional | A lower bound date matching the `yyyy-MM-dd'T'HH:mm:ss` format. News API provides data 1 month back. | `2020-06-28T09:41:00` |
| date_to | optional | An upper bound date matching the `yyyy-MM-dd'T'HH:mm:ss` format. News API provides data 1 month back. | `2020-06-29T09:41:00` |
| source | optional | Search a specific domain. | `tagesschau.de` |

### Data Collection

To get data you have to call the API endpoint. We host an endpoint for GDELT at <http://bart.ukp.informatik.tu-darmstadt.de:11203/gdelt-{language}/_search> and one for News API at <http://bart.ukp.informatik.tu-darmstadt.de:11204/newsapi-{language}/_search>. These endpoints have to be called over a HTTP POST request and search queries and filters have to be provided in the JSON format specified below.

_POST data send to the GDELT endpoint. The parameters `{size}`, `{type}`, `{search query}`, `{date from}`, `{date to}`, `{country}`, `{source}`, and `{*source}` have to be replaced. Most filters are optional, see the GDELT parameters table above for additional details:_

    {
      "size": {size},
      "query": {
        "bool": {
          {occurrence_type}: {
            "multi_match": {
              "query": {search_query},
              "fields": ["doc.text"],
              "type": "most_fields"
            }
          },
          "filter": [{
            "range": {
              "metadata.timestamp": {
                "gte": {date_from},
                "lte": {date_to},
                "format": "date_hour_minute_second"
              }
            }
          }, {
            "term": {
              "metadata.country": {country}
            }, {
              "term": {
                "metadata.source": {source}
              }
            }, {
              "wildcard": {
                "metadata.source": {
                  "value": {*source}
                }
              }
            }]
        }
      }
    }


_POST data send to the News API endpoint. The parameters `{size}`, `{type}`, `{search query}`, `{date from}`, `{date to}`, and `{source}` have to be replaced. Most filters are optional, see the News API parameters table above for additional details:_

    {
      "size": {size},
      "query": {
          "bool": {
            {occurrence_type}: {
              "multi_match": {
                "query": {search_query},
                "fields": ["doc.text"],
                "type": "most_fields"
              }
            },
            "filter": [{
              "range": {
                "metadata.timestamp": {
                  "gte": {date_from},
                  "lte": {date_to},
                  "format": "date_hour_minute_second"
                }
              }
            }, {
              "term": {
                "metadata.source": {source}
              }
            }]
        }
      }
    }


_A Python reference implementation for a COVID-19 search query can be found below. The config dictionary can be extended with additional filters:_

    import requests

    config = {
      'query': {
        'bool': {
          'must': {
            'multi_match': {
              'query': 'COVID-19',
              'fields': ['doc.text'],
              'type': 'most_fields'
            }
          }
        }
      }
    }
    request = requests.post('http://bart.ukp.informatik.tu-darmstadt.de:11203/gdelt-en/_search', data=str(config))
    response = request.json()

### Data Structure

The returned data from GDELT or News API is a JSON object in the Elasticsearch JSON format. An example can be found below. It contains so called `hits` with each `hit` representing one news article. Each article is returned as a parsed full text string with removed boilerplate, sentence segmented spans, and additional metadata like description, header, keywords, language, source, and timestamps. The data can be further processed or parsed for annotation in INCEpTION.

    {"took" : 6295,
    "timed_out" : false,
    "hits" : {
    "total" : 228,
    "hits" : [
    {"_id": "db20c4e95ab3394596adcf070cc3a6cc","_source": {
      "doc" : {
        "text" : "Corona-Warn-App GeglÃ¼ckter Start - mit Irritationen\nStand: 23.06.2020 04:14 Uhr\nVon Dennis Horn, WDR\n\"Am Tag nach der VerÃ¶ffentlichung haben wir im Schnitt neun Leute in unserer direkten Umgebung gesehen, die die App wirklich nutzen. Heute waren es schon 20 Menschen\", sagt Merlin Chlosta. ...",
        "annotations" : {
          "spanAnnotations" : [ {
            "spanText" : {
              "span" : {
                "begin" : 0,
                "end" : 69
              },
              "coveredText" : "Corona-Warn-App GeglÃ¼ckter Start - mit Irritationen Stand : 23.06.2020"
            },
            "label" : "Sentence",
            "attributes" : { }
          }, ... ],
          "relations" : [ -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1 ]
        }
      },
      "metadata" : {
        "country" : "Germany",
        "description" : "Vor einer Woche wurde die Corona-Warn-App verÃ¶ffentlicht - und sie wird angenommen: Fast zwÃ¶lf Millionen Menschen haben die App inzwischen auf ihr Smartphone geladen. Dennis Horn erklÃ¤rt, warum das eine wichtige Marke ist.",
        "header" : "Corona - Warn - App : Wichtige Marke von Downloads erreicht",
        "id" : "db20c4e95ab3394596adcf070cc3a6cc",
        "indexing_time" : "2020-06-29T16:01:25+0000",
        "keywords" : "Nachrichten, Inland, Ausland, Wirtschaft, Sport, Kultur Reportage, Bericht, News, Tagesthemen, Aktuell, Neu, Neuigkeiten, Hintergrund, Hintergrund, Information, Politik, Innenpolitik, Aussenpolitik, Video, Audio",
        "language" : "German",
        "source" : "tagesschau.de",
        "timestamp" : "2020-06-23T02:45:00+0000",
        "uri" : "https://www.tagesschau.de/inland/bilanz-warnapp-101.html"
      }
    }}, ... ]
    }
    }

### Preparing Data for INCEpTION

Using the sample code above we further process data to get a UIMA XML CAS file, which is the preferred input format for INCEpTION. In this demonstration we extract the main text, but metadata could also be added. It is important to split data into chunks if it should be distributed between multiple annotators and to simplify train, dev, and test set generation for machine learning.

_Sample code in Python to parse data into the INCEpTION compatible UIMA XML CAS format:_

    import html
    import random
    from typing import List

    def prepare(line: str, indent: int = 0) -> str:
      """Add line breaks and indentation."""
      return (' ' * indent) + line + '\n'

    def lst_to_uima_cas(lst: List[str], encoding: str = 'utf-8') -> str:
      """Parse a list of strings to UIMA XML CAS format."""
      xml = prepare(f'<?xml version=\'1.0\' encoding=\'{encoding.upper()}\'?>')
      xml += prepare('<xmi:XMI xmlns:xmi="http://www.omg.org/XMI" xmlns:cas="http:///uima/cas.ecore" xmlns:type="http:///de/tudarmstadt/ukp/dkpro/core/api/segmentation/type.ecore" xmi:version="2.0">')
      xml += prepare('<cas:NULL xmi:id="0"/>', 2)
      concat = "
      for idx, value in enumerate(lst):
        if idx > 0:
          # Add two line breaks as a separator
          concat += '\n\n'
        pos = len(concat)
        # Accumulate articles
        concat += value
        # Set spans
        xml += prepare(f'<type:Sentence xmi:id="{idx + 2}" begin="{pos}" end="{len(concat)}" sofa="1"/>', 2)
      xml += prepare(f'<cas:Sofa xmi:id="1" sofaNum="1" sofaID="_InitialView" mimeType="None" sofaString="{html.escape(concat)}"/>', 2)
      indices = ' '.join(str(i) for i in range(2, len(lst) + 2))
      xml += prepare(f'<cas:View sofa="1" members="{indices}"/>', 2)
      xml += prepare('</xmi:XMI>')
      return xml

    # Insert code from previous Python sample
    ...

    # Gather article texts into a list
    corpus = [hit['_source']['doc']['text'].replace('\n', ' ') for hit in response['hits']['hits']]
    # Shuffle data
    random.shuffle(corpus)
    encoding = 'utf-8'
    # Split data into chunks for an even distribution between annotators
    chunk_size = 10
    for i in range(0, len(corpus), chunk_size):
      xml = lst_to_uima_cas(corpus[i:i + chunk_size], encoding)
      file_name = f'news_{i / chunk_size + 1}.xmi'
      with open(file_name, 'w', encoding=encoding) as f:
        f.write(xml)

To summarize:

- We collect news data from an API endpoint hosted at UKP
- Article full text is extracted into a list
- Data is shuffled and split into chunks
- Each chunk is parsed into the UIMA XML CAS format for INCEpTION

## Twitter Data Crawling and Preparation

Twitter collection has four different modes of collecting data. We will go through each one and detail what needs to be configured.

### Obtain Twitter Credentials
The two most common modes for collection, i.e. streaming collection of live Tweets and collection of user Tweets use the official Twitter API and therefore need Twitter developer credentials. In order to obtain then, you need to:

1. Apply for an Twitter developer account at <https://developer.twitter.com/en/apply-for-access>
2. Create a new app on the Twitter Developer Portal at <https://developer.twitter.com/en/dashboard>
3. In the app you just created, click on "Keys and Tokens" and there generate an API key and and access token

After obtaining the credentials, you need to open the file `./config/credentials_template.txt` with a plain-text editor, insert the API key and access token in the appropriate field and save the file under the name `credentials.txt` in the same location.

### Configuring and Running Crawling Tasks

Before you can run one of the collection modes, you need to configure them.

#### Configure Dehydration Keys

In order to minimize memory requirements on the hard drive the crawler provides a so-called dehydration feature which strips unnecessary meta data of the Tweets. In order to configure which meta data of the Tweet is kept and which is discarded open `./config/dehydration_keys.txt` and `./config/dehydration_keys_user.txt`. There you find most meta data-keys which are contained in a Tweet and the Tweet's user respectively. For instructions on how to add new meta data-keys, see instructions in the configuration files themselves. In order to keep a specific data point leave the keyword __un-commented__, in order to discard it comment it out, i.e. the `text` keyword should probably be un-commented while the `possibly_sensitive` can probably be commented-out.

#### Streaming Live Tweets

The streaming feature collects live Tweets from Twitter and requires search terms to be specified. These terms are used to filter live Tweets and the resulting tweets will contain at least one of the tracked terms. Open the file `./config/tracked_terms.txt` and enter one term per line, i.e. for crawling COVID-19 related terms you could use terms such as "corona", "covid-19", "lockdown", and "pandemic".

In order to start the crawler with the streaming feature, open a command line prompt and enter the following command:

    python3 TwitterCrawler.py --streaming --skip-index

The `--skip-index` flag deactivates duplicate checking by skipping building the index for duplicate detection. This is not necessary if you only use the streaming feature as live Tweets can not be duplicates.

In order to exit the crawler when using the streaming feature press `CTRL-C`.

#### Historical Collection

The historical feature collects Tweets from specific dates and requires the dates and the search terms to be specified. See the section above about the streaming feature on how to configure the search terms. In order to configure which dates will be collected open the file `./config/historical_gaps.txt` with a plain-text editor. Dates need to be entered in the format "YYYY MM DD", i.e. if you would like to collect Tweets from the day that China alerted the WHO about the COVID-19 disease, you would enter "2019 12 31". One date per line, no quotes.

In order to start the crawler with the historical collection feature, open a command line prompt and enter the following command:

    python3 TwitterCrawler.py --historical

Be aware to not use the `--skip-index` flag when using the historical feature as duplicates are likely and most probably should be detected.

#### Reply Thread Collection

The reply thread feature can be used to collect reply threads which are Tweets together with all the responses and subsequent responses. Reply threads are specified by the ID of the root Tweet which needs to be provided. For this open the file `./config/thread_ids.txt` with a plain-text editor and provide one Tweet ID per line, i.e. "1238155454108151808" for a Tweet thread discussing free COVID-19 testing for all American citizens. Note: this collection is quite slow, as it takes ~4 seconds / crawled reply.


In order to start the crawler with the reply thread feature, open a command line prompt and enter the following command:

    python3 TwitterCrawler.py --reply-threads

#### User Tweets Collection

This feature can be used to collect all Tweets of a specific user. In order to collect user tweets, open the file `./config/user_names.txt` and provide one user name per line, i.e. insert "realDonaldTrump" to get Tweets from the President of the United States. Optionally you can provide a date range after each provided user name in order to filter Tweets by date, i.e. insert the line "realDonaldTrump 2019-12-31 2020-07-04" to get all Tweets by Donald Trump between the beginning of the pandemic and the Independance day of the United States.

In order to start the crawler with the user Tweet collection, open a command line prompt and enter the following command:

    python3 TwitterCrawler.py --users

#### Combining multiple collection features

All the flags used in the individual approaches can be combined into one command, i.e.

    python3 TwitterCrawler.py --historical --streaming --users --reply-threads


### Preparing Data for INCEpTION

#### Filter Dataset by Language

Before continuing to use the dataset it is sometimes useful to filter the dataset by language. In order to do so, open the command prompt and execute this command:

    python3 TwitterCrawler.py --filter en

Where "en" is the language as give by Twitter.

#### Converting the Dataset to UIMA XML CAS

The data set is in a universally-readable JSON format, see an example below. In order to convert the data set to the UIMA XML CAS format for INCEpTION open the command prompt and execute the provided helper tool:

    python3 TweetToCas.py ./data/out.jsonl

_One Tweet in the JSON format:_

    {"created_at": "Mon Sep 28 15:35:36 +0000 2020", "id": 1310604077039001600, "id_str": "1310604077039001600", "text": "RT @brianklaas: The other sinister aspect of the Trump tax story is that his finances are so bad that it raises the possibility he systemat...", "truncated": false, "user": {"id": 758406643478564864, "location": "Brooklyn, NY, United States", "description": "I'm angry. I'm angry that Democrats feel we can't be the toughest one in the room. So I'm stepping into that room. And I'm from Bklyn, so there's that.", "time_zone": null, "geo_enabled": false, "lang": null}, "geo": null, "coordinates": null, "place": null, "entities": {"hashtags": [], "urls": [], "user_mentions": [{"screen_name": "brianklaas", "name": "Brian Klaas", "id": 39279821, "id_str": "39279821", "indices": [3, 14]}], "symbols": []}, "lang": "en"}

## Annotation in INCEpTION

### Annotation Study Setup

In this section we will give a short introduction on how to use INCEpTION for your annotation study with our crawled news and Twitter data. The upcoming 3 steps describe in detail how you can create a project, invite users to your project and manage the documents which shall be available to annotate in INCEpTION. Furthermore, if you require a more detailed walk-through on all features INCEpTION has to offer, or simply require more information on INCEpTION we suggest you to follow these links:

- <https://zoidberg.ukp.informatik.tu-darmstadt.de/jenkins/job/INCEpTION%20(GitHub)%20(master)/de.tudarmstadt.ukp.inception.app$inception-app-webapp/doclinks/1/>
- <https://github.com/inception-project/inception>

The first link is INCEpTION's offical "Getting started" document, which will give you in depth information on everything INCEpTION has to offer. The second link is the open source repository for INCEpTION. The features we added are within the following three modules:

- `workload`
- `workload-dynamic`
- `workload-matrix`

### Project Structure and Setup

The first part in INCEpTION is simple and straightforward. After your first login into INCEpTION simply click the blue button in the top left corner of your screen with the caption "Create new project ...". On the following page you need to enter the projects name a click "save" on the bottom right side of your screen.

#### Annotators for a Project

After you have created your first project you can immediately invite users to it. They simply need an account on the corresponding server (either on the UKP blinky server or your own). In order to add someone to your project you need to follow those steps:

1. Select your project in the project selection menu
2. Click on "Settings" on the left hand side of your screen in the menu bar
3. Select the "Users" tab
4. In the "Add users" select the names of the users you want to add to your project
5. Hit the "Add" button

#### Import Data in INCEpTION

Importing our news and Twitter data into INCEpTION is done via uploading the files from your local machine. All of our data is provided in the UIMA XML CAS format which INCEpTION can easily handle. To add new documents:

1. Go to the projects settings
2. Select the tab "Documents"
3. Select the file you want to import
4. Do not forget to select the correct "UIMA XML CAS" format for your documents
5. Click the "Import" button

### Workload Manager

The new workload manager is our addition to INCEpTION. This extension gives project managers not only a better and faster overview, but also more control for the documents and the annotation workflow in their project.

In order to enable our new workload manager, you have to do the following steps:

1. Add to the `application.properties` file a new line: `workload.dynamic.enabled=true`. As our new feature is only for experimental use in INCEpTION it has to be enabled first in order to be usable
2. Go to the project settings
3. Select the new "Workload" tab
4. From the drop-down menu select the "Dynamic assignment"
5. Click the "Save" button

Now the project manager is able to access the new "Workload" menu item in the projects overview menu bar. Our new workload mainly consists of a huge, but easy to understand table containing all the data a manager needs from their documents. Furthermore, we added three tabs on the top right corner of the table, which give the project manager more features for his documents and users.

__Filter:__

This drop-down menu enables different filtering mechanisms for your table, e.g. filtering for documents a specific annotator is working on, or for a specific document in general.

__Annotators:__

In the annotators drop-down menu you are able to assign documents to an annotator. Furthermore, if an annotator "finished" a document unintended, you can revert the status for a document so he/she can continue working on the document.

__Settings:__

Here you are able to assign a specific document distribution mechanism as well as a "default number of annotation". Normally, each document can be annotated an infinite amount of times and annotators may select freely which document they want to annotate.

In order to have a better annotation distribution throughout all of the documents, annotators can no longer select which document they want to annotate, but rather get document after document automatically assigned. This happens either in a randomized manner throughout all documents in the project, or simply one after another from the whole list. Furthermore, each document can only be assigned a specific "default number of annotations" amount of times. This also ensures a better document distribution.

### Inter Annotator Agreement

An important and well known measure on how often multiple annotators make the same annotation decision for a specific category is the inter annotator agreement (IAA). It is required to ensure stable and consistent annotation for different categories, e.g. our sentiment analysis for Tweets. Before each annotator starts his annotation process, it is important to create an annotation guideline, that describes in detail under which circumstance a specific category has to be chosen, e.g. a Tweets sentiment is "positive", "negative" or "neutral". Then, Cohens Kappa can be calculated from the labeled Tweets. However, even with a uniform annotation guideline, annotators may label the same data differently. In order to reduce the divergence, the annotation guidelines have to be finer-granulated for each possible case, until the overall inter-rater-reliability has a reasonable value for the corresponding annotation task. An inter-rater-reliability value which is too low might be not good enough for a machine learning model to train reliably, therefore a suitable value for the annotation task must be achieved. In order to finer-granulate the annotation guidelines it is important to give more examples for each label when it has to be chosen. The higher the overall amount of provided examples for each label, the better can annotators comply with the rules and thereby achieving higher values in the inter-rater-reliability value.

INCEpTION is able to calculate Cohens Kappa by automatically. The project manager has to perform the following steps:

1. Click on "Agreement" in the projects menu bar
2. Select the feature for which Cohens Kappa shall be calculated
3. Select "Cohens Kappa" in the "Measure" drop-down
4. Click the "Calculate" button

## Machine Learning

For machine learning we provide a generic data set reader and one specific for [AllenNLP](https://allennlp.org). Both expect a folder path as input and will read every UIMA XML CAS file in there. Train, dev, and test data has to be exported from INCEpTION and put into respective folders. The data should be shuffled and common advice for train, dev, and test splits should be followed, e.g. 80 / 10 / 10. Using these data sets a machine learning model can be trained.

### Preparing INCEpTION Data for Machine Learning

Once data is split into folders, as described above, the `read()` function below can be called on each folder to get a list of tuples containing text and, in this case, a sentiment label. The label name has to be adopted to the name specified in INCEpTION.

_Sample code in Python for an generic data set reader based on INCEpTION data:_

    import os
    from typing import List, Tuple

    from cassis import load_cas_from_xmi, load_typesystem

    def read(self, path: str) -> List[Tuple[str, str]]:
      corpus = []
      with open(os.path.join(path, 'TypeSystem.xml'), 'rb') as f:
        typesystem = load_typesystem(f)
      for name in os.listdir(path):
        file_path = os.path.join(path, name)
        if os.path.splitext(file_path)[1] == 'xmi':
          with open(file_path, 'rb') as f:
            cas = load_cas_from_xmi(f, typesystem=typesystem)
          for doc in cas.select_all():
            if hasattr(doc, 'Sentiment') and doc.Sentiment is not None:
              text = doc.get_covered_text()
              sentiment = doc.Sentiment.lower()
              corpus.append((text, sentiment))
      return corpus

### Using INCEpTION Data in AllenNLP

Similar to generic machine learning the AllenNLP `INCEpTIONDatasetReader()` below can be used for each folder to get a data set reader containing tokens and, in this case, a sentiment label. The label name has to be adopted to the name specified in INCEpTION. This data set reader can be used as any other in AllenNLP.

_Sample code in Python for an AllenNLP DatasetReader based on INCEpTION data:_

    import os
    from typing import Dict, Iterable

    from allennlp.data import Instance
    from allennlp.data.dataset_readers import DatasetReader
    from allennlp.data.fields import LabelField, TextField
    from allennlp.data.token_indexers import SingleIdTokenIndexer, TokenIndexer
    from allennlp.data.tokenizers import Tokenizer
    from allennlp.data.tokenizers.spacy_tokenizer import SpacyTokenizer

    from cassis import load_cas_from_xmi, load_typesystem

    @DatasetReader.register('INCEpTION_dataset_reader')
    class INCEpTIONDatasetReader(DatasetReader):

      def __init__(self, lazy: bool = False, tokenizer: Tokenizer = None, token_indexers: Dict[str, TokenIndexer] = None, max_tokens: int = None):
        super().__init__(lazy)
        self.tokenizer = tokenizer or SpacyTokenizer()
        self.token_indexers = token_indexers or {'tokens': SingleIdTokenIndexer()}
        self.max_tokens = max_tokens

      def text_to_instance(self, tokens: str, label: str = None) -> Instance:
        if self.max_tokens:
          tokens = tokens[:self.max_tokens]
        text_field = TextField(tokens, self.token_indexers)
        fields = {'text': text_field}
        if label:
          fields['label'] = LabelField(label)
            return Instance(fields)

      def _read(self, path: str) -> Iterable[Instance]:
        with open(os.path.join(path, 'TypeSystem.xml'), 'rb') as f:
          typesystem = load_typesystem(f)
        for name in os.listdir(path):
          file_path = os.path.join(path, name)
          if os.path.splitext(file_path)[1] == 'xmi':
            with open(file_path, 'rb') as f:
              cas = load_cas_from_xmi(f, typesystem=typesystem)
            for doc in cas.select_all():
              if hasattr(doc, 'Sentiment') and doc.Sentiment is not None:
                tokens = self.tokenizer.tokenize(doc.get_covered_text())
                sentiment = doc.Sentiment.lower()
                yield self.text_to_instance(tokens, sentiment)
