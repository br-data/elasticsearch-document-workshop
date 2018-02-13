# Dokumenten-Suche mit Elasticsearch und Apache Tika


## Tika

[Apache Tika](https://tika.apache.org/) erkennt und extrahiert Metadaten und Text aus verschiedenen Dateitypen (wie PDF, DOC und XLS). Alle diese Dateitypen können über eine einzige Schnittstelle verarbeitet werden, wodurch Tika für die Indexierung, Inhaltsanalyse und Übersetzung und vieles mehr verwendet werden. Tika ist eine Java-Anwendung (Standalone oder Server), welche durch verschiedene Parser und Detektoren erweitert werden kann.

### Text extrahieren

mvn install

```
$ java -jar tika-app-1.17.jar -t -i ./pdf -o ./text
$ java -jar tika-app-1.17.jar -c config.xml -t -i pdf -o text
```


```xml
<dependency>
  <groupId>com.github.jai-imageio</groupId>
  <artifactId>jai-imageio-core</artifactId>
  <version>1.3.1</version>
</dependency>

<dependency>
  <groupId>com.github.jai-imageio</groupId>
  <artifactId>jai-imageio-jpeg2000</artifactId>
  <version>1.3.0</version>
</dependency>

<dependency>
  <groupId>com.levigo.jbig2</groupId>
  <artifactId>levigo-jbig2-imageio</artifactId>
  <version>2.0</version>
</dependency>
```

node-tika

https://github.com/br-data/elasticsearch-import-tools/blob/master/extract-multicore.js

## Elasticsearch

[Elasticsearch](https://www.elastic.co/) ist eine mächtige und sehr schnelle Open-Source-Suchmaschine für Textdokumente. Elasticsearch ist in Java geschrieben und verwendet Apache Lucene für die Indizierung und Suche nach Texten. Die Komplexität und der Funktionsumfang von Lucene versteckt sich hinter einer einfachen REST-API, welche die Entwicklung und Integration mit bestehenden Anwendungen erleichtert. Für die REST-API sind Clients für alle gängigen Programmiersprachen verfügbar.

Wichtige Grundbegriffe für Elasticsearch sind:

- **Cluster**: Der Verbund aller Nodes (Server) auf denen Elasticsearch läuft
- **Node**: Eine Instanz von Elasticsearch auf einem Server
- **Index**: Eine Sammlung an Dokumenten
- **Document**: Ein Datenbankeintrag

Für den skalierbaren und performaten Produktiveinsatz:

- **Shard**: Eine Instanz von Elasticsearch mit einem Teil der Dokumente
- **Replica**: Eine Instanz von Elasticsearch mit einer Kopie eines Index

In der Entwicklungsphase kommt es häufig vor, dass man der Cluster aus nur einem Node mit einem Index besteht.

### Installation

Elasticsearch kann man auf Mac OS X einfach über [Homebrew](https://brew.sh/index_de.html) zu installieren:

```
$ brew install elasticsearch@2.4
```

Service

### Status und Verwaltung

```
$ curl -XGET 'http://localhost:9200/_cluster/health?pretty=1'
```


Für die Entwicklung kann die Anzahl der Replicas auf 0 gesetzt werden, um _Unassigned shards_-Warnungen zu vermeiden. 

```
url -XPUT 'localhost:9200/_settings' -d '         
{                  
  index: {
    number_of_replicas : 0
  }
}'
```

Chrome Extension: [ElasticSearch Head](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm)

### Mapping & Analyse

as Skript bereitet einen Elasticsearch-Index für den Import von Dokumenten vor. Der alte Index wird dabei gelöscht. Um nach Begriffen mit diakritischen Zeichen suchen zu können wird eine eigener Analyzer mit [ASCII-Folding](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/asciifolding-token-filter.html) angelegt. Der Analyser ersetzt diakritische Zeichen mit den entsprechenden ASCII-Zeichen. So wird _Conceição_ im Feld **body** zu _Conceicao_ im Feld **body.folded**.

Skript starten:

```
$ node prepare.js
```

Der Analyzer kann auch manuell über das REST-Interface angelegt werden:

```
curl -XPUT localhost:9200/joram -d '
{
  settings: {
    analysis: {
      analyzer: {
        folding: {
          tokenizer: "standard",
          filter: ["lowercase", "asciifolding"]
        }
      }
    }
  }
}'
```

Der Mapper muss auch im Mapping angegeben werden:

```
curl -XPUT localhost:9200/joram/_mapping/doc -d '
{
  properties: {
    body: {
      type: "string",
      analyzer: "standard",
      fields: {
        folded: {
          type: "string",
          analyzer: "folding"
        }
      }
    }
  }
}'
```

https://github.com/br-data/elasticsearch-import-tools/blob/master/prepare.js

### Dokumente importieren

Das Skript importiert die Dokumente in Elasticsearch und ergänzt die Dokumente um ein paar Metadaten.

Datenmodell:

```javascript
body: {
  file: file,
  date: date,
  author: date,
  language: language,
  body: body
}
```

https://github.com/br-data/elasticsearch-import-tools/blob/master/import.js

### Dokumente suchen

Term vs. full text search

```
$ curl -XGET 'http://localhost:9200/_count?pretty' -d '
{
  "query": {
      "match_all": {}
  }
}'
```

### Frontend

### Zukunftsausblick
