# Dokumenten-Workshop

Dokumenten aus öffentlichen Quellen oder Leaks mit Apache Tika aufbereiten und mit Elasticsearch durchsuchbar machen.

*“Documents are the new data.”*

## Tika

[Apache Tika](https://tika.apache.org/) erkennt und extrahiert Metadaten und Text aus verschiedenen Dateitypen (wie PDF, DOC und XLS). Alle diese Dateitypen können über eine einzige Schnittstelle verarbeitet werden, wodurch Tika für die Indexierung, Inhaltsanalyse und Übersetzung und vieles mehr verwendet werden. Tika ist eine Java-Anwendung (Standalone oder Server), welche durch verschiedene Parser und Detektoren erweitert werden kann.

### Installation

Tika kann über [Homebrew](https://brew.sh/index_de.html) installiert oder einfach von der [Homepage](https://tika.apache.org/download.html) heruntergeladen werden.

```
$ brew install tika
```

Es gibt verschiedene JAR-Releases von Tika:

- **Source**: Source Code zum selbst kompilieren und modifizieren
- **App**: Standalone-Anwendung mit (fast) allen Abhängigkeiten 
- **Server**: Server-Anwendung mit REST-Interface
- **Eval**: Tool zum Vergleichen verschiedener Tools und Versionen

In machen Fällen bietet es sich an, Tika aus dem Source Code selbst zu bauen. Dieses Vorgehen ermöglicht eine bessere Kontrolle über die verwendeten Abhängigkeiten und die Konfiguration.

### Text extrahieren

Die häufigste Form von Dokumenten im Bereich Journalismus sind PDFs. Wurden diese direkt aus Word oder eine vergleichbaren Textverarbeitungssoftware erstellt, lässt sich der Text verhältnismäßig einfach extrahieren. Etwas schwieriger wird es bei Scans von Dokumenten. Um den Text aus Bildern zu extrahieren verwendet der Tika PDF-Parser die Open-Source-OCR-Lösung [Tesseract](https://github.com/tesseract-ocr/tesseract).

```
brew install tesseract --with-all-languages --with-serial-num-pack
```
Wenn Tesseract auf dem System installiert ist, versucht Tika automatisch Tesseract für die Extraktion von Bildern aufzurufen. 

```
$ java -jar tika-app-1.17.jar -t -i ./pdf -o ./text
```

Standardgemäß klappt das aber nicht perfekt, weil Abhängigkeiten zum Verarbeiten von TIFF, JPEG2000 und BigJPEG-Bildern fehlen. 

In diesem Fall muss man Tika aus dem Source Code neu bauen. Dazu müssen die fehlenden Abhängigkeiten im [Maven](https://maven.apache.org/)-Projekt hinzugefügt werden. Hier ein Auszug aus der `pom.xml` im Ordner `app`:

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

Sind die fehlenden Abhängigkeiten hinzugefügt, muss das Projekt neu gebaut werden:

```
$ mvn install
```

Sollte man Tesseract nicht verwenden wollen, kann man den [PDF Parser](https://wiki.apache.org/tika/PDFParser%20%28Apache%20PDFBox%29) in einer Konfigurationsdatei `config.xml` entsprechen anpassen:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<properties>
  <parsers>
    <parser class="org.apache.tika.parser.DefaultParser">
      <parser-exclude class="org.apache.tika.parser.ocr.TesseractOCRParser"/>
    </parser>
  </parsers>
</properties>
```

Eine eigene Konfigurationsdatei kann man mit dem Parameter -c laden:

```
$ java -jar tika-app-1.17.jar -c config.xml -t -i pdf -o text
```

Für die Verwendung in Node.js gibt es eine Brücke [node-tika](https://github.com/ICIJ/node-tika). Diese wurde aber schon länger nicht mehr aktualisiert und ist sehr fehleranfällig. node-tika wird auch in den BR Data [elasticsearch-import-tools](https://github.com/br-data/elasticsearch-import-tools/blob/master/extract-multicore.js) verwendet.

Mehr Infos zur Verwendung von Tika in Verbindung mit Tesseract finden sich im [Tika Wiki](https://wiki.apache.org/tika/TikaOCR).

## Elasticsearch

[Elasticsearch](https://www.elastic.co/) ist eine mächtige und sehr schnelle Open-Source-Suchmaschine für Textdokumente. Elasticsearch ist in Java geschrieben und verwendet Apache Lucene für die Indizierung und Suche nach Texten. Die Komplexität und der Funktionsumfang von Lucene versteckt sich hinter einer einfachen REST-API, welche die Entwicklung und Integration mit bestehenden Anwendungen erleichtert. Für die REST-API sind Clients für alle gängigen Programmiersprachen verfügbar.

Wichtige Grundbegriffe für Elasticsearch sind:

- **Cluster**: Der Verbund aller Nodes (Server) auf denen Elasticsearch läuft
- **Node**: Eine Instanz von Elasticsearch auf einem Server oder dem eigenen Rechner
- **Index**: Eine Sammlung an Dokumenten und damit verbundene Einstellungen (Analyzer, Mapping)
- **Document**: Ein Datenbankeintrag für jedes Dokument (JSON-Objekt mit Text im Feld `body`)

Für den skalierbaren Produktiveinsatz:

- **Shard**: Eine Instanz von Elasticsearch mit einem Teil der Dokumente
- **Replica**: Eine Instanz von Elasticsearch mit einer Kopie eines Index

In der Entwicklungsphase kommt es häufig vor, dass der Cluster nur aus einem Node mit einem Index besteht.

Zum Einarbeiten in Elasticsearch empfiehlt sich der [Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/index.html).

### Installation

Elasticsearch kann man auf Mac OS X einfach über [Homebrew](https://brew.sh/index_de.html) zu installieren. Wir arbeiten momentan noch mit der Version 2.4 von Elasticsearch. Ein Update ist aber geplant.

```
$ brew install elasticsearch@2.4
```

Elasticsearch kann man entweder direkt ausführen...

```
$ elasticsearch@2.4
```

... oder als Service starten:

```
$ brew services start elasticsearch@2.4
```

Für die ordnungsgmäße Ausführung braucht es eine gültige [Konfiguration](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/setup-configuration.html#settings):

Um herauszufinden, wo auf dem System die Konfiguration liegt, kann man `brew info` aufrufen:

```
brew info elasticsearch@2.4
```

Linux-Benutzer können Elasticsearch mit [systemd](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/setup-service.html#using-systemd) oder init(https://www.elastic.co/guide/en/elasticsearch/reference/2.4/setup-service.html#_using_chkconfig) starten.

### Status und Verwaltung

Den aktuellen Status des Elasticsearch-Clusters kann man über das REST-Interface abfragen:

```
$ curl -XGET 'http://localhost:9200/_cluster/health?pretty=1'
```

Viel komfortabler lassen sich die verschiedenen Nodes und Indizes aber mit der Chrome Extension [ElasticSearch Head](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm) verwalten.


Falls es _Unassigned shards_-Warnungen gibt, können für die Entwicklung, die Anzahl der Replicas auf 0 gesetzt werden: 

```
$ curl -XPUT 'localhost:9200/_settings' -d '         
{                  
  index: {
    number_of_replicas : 0
  }
}'
```

Im Idealfall ist der Cluster-Status jetzt grün.

### Index erstellen

Leeren Index mit dem Namen `my-index` anlegen:

```
$ curl -XPUT localhost:9200/my-index -d '
{
  settings: {
    index: {
      number_of_replicas: 0
    }
  }
}'
```

Elasticsearch bestätigt gelungene Anfragen immer mit `{"acknowledged":true}`. Falsche oder nicht durchführbare Anfragen werden mit einer Fehlermeldung quittiert.

Eine Liste aller Indizes bekommt man mit:

```
$ curl -XGET  'localhost:9200/_cat/indices?v'
```

Einen Index kann man auch sehr schnell wieder löschen:

```
$ curl -XDELETE localhost:9200/my-index
```

### Analyzer und Mapping

Analyzer sind Funktionen, die bestimmen, wie mit importierten Texten umgegangen werden soll. Das sind zum Beispiel:

- **Tokenizer**: Eingabetext auf Wortgrenzen aufteilen
- **Token-Filter**: Vom Tokenizer ausgegebenen Token aufräumen
- **Lowercase-Token-Filter**: Alle Token in Kleinbuchstaben konvertieren
- **Stop-Token-Filter**: Stoppwörter (der, die, das, ein, eine ...) entfernen

Um nach Begriffen mit diakritischen Zeichen suchen zu können, legen wir einen eigenen Analyzer mit [ASCII-Folding](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/asciifolding-token-filter.html) an. Der Analyser ersetzt diakritische Zeichen mit den entsprechenden ASCII-Zeichen. So wird das portugiesische _Conceição_ in _Conceicao_ umgewandelt.

```
$ curl -XPUT localhost:9200/my-index -d '
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

Definierte Analyzer können dann im Mapping verwendet werden. Im Mapping wird definiert, welcher Analyzer auf welches Feld angewandt werden soll. In unserem Beispiel wollen wir, dass _Conceição_ im Feld **body** zu _Conceicao_ im Feld **body.folded** wird:

```
$ curl -XPUT localhost:9200/my-index/_mapping/doc -d '
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

Im Mapping kann man auch die Datentypen von bestimmten Feldern festlegen, wenn man zum Beispiel mit Datumsfeldern oder Zahlen arbeitet:

```
$ curl -XPUT localhost:9200/my-index/_mapping/doc -d '
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
    },
    meta: {
      properties: {
        timestamp: {
          type: "date",
          format: "strict_date_optional_time"
        }
      }
    }
  }
}'
```

Wird kein Analyzer oder Mapping angegeben, verwendet Elasticsearch (meist sinnvolle) Standardeinstellungen. Mehr Infos zu Analyzern und Mapping gibt es in der [Elasticsearch Dokumentation](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/configuring-analyzers.html).

In den [elasticsearch-import-tools](https://github.com/br-data/elasticsearch-import-tools) gibt es eine [Skript](https://github.com/br-data/elasticsearch-import-tools/blob/master/prepare.js), welches einen Elasticsearch-Index für den Import von Dokumenten vorbereitet und einem viel Arbeit abnimmt. Der alte Index wird dabei gelöscht:

```
$ node prepare.js localhost:9200 my-index doc
```

**Wichtig:** Wenn man Analyzer oder Mapping eines bestehenden Index' ändert, sollte man die Daten danach [reindizieren](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/reindex.html).  

### Dokumente importieren

Einzelne Daten lassen sich über die REST-API importieren:

```
$ curl -XPUT 'localhost:9200/my-index/doc/1?pretty' -d '
{
  "fileName": "text/my-textfile.txt",
  "date": "2018-02-12",
  "author": "John Doe",
  "body": "Lorem ipsum dolores sit amet."
}'
```

Um viele Dokumente zu importieren, empfiehlt sich die Verwendung einer Library, wie [elasticsearch.js](https://github.com/elastic/elasticsearch-js) für Node.js. Ein einfaches Beispiel:

```javascript
const elastic = require('elasticsearch');
const client = new elastic.Client({ host: 'localhost:9200' });

client.create({
  'my-index',
  'doc',
  body: {
    fileName: 'text/my-textfile.txt',
    date: '2018-02-12',
    author: 'John Doe',
    body: 'Lorem ipsum dolores sit amet.'
  }
},
error => {
  if (error) {
    console.error(error)
  } else {
    console.log(`Inserted document ${filePath} to ElasticSearch`);
  }
});
```

Auch für den Dateimport gibt es ein fertiges [Skript](https://github.com/br-data/elasticsearch-import-tools/blob/master/import.js) aus der BR-Data-Werkzeugkiste:

```
$ node import.js ./text localhost:9200 my-index doc
```

### Dokumente suchen

```
$ curl -XGET 'http://localhost:9200/my-index/doc/_search' -d '{
  "query" : {
    "match" : {
      "author" : "John Doe"
    }
  }
}'
```

Es gibt verschiedene Möglichkeiten eine Suchanfrage zu stellen. Grundsätzlich wird zwischen Volltextsuche (full text) und Begriffssuche (term) unterschieden. Hier die vier wichtigsten Suchmethoden:

- **Multi Match Query**: Findet exakte Suchausdrücke wie **John Doe** [(Dokumentation)](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html).
- **Simple Query DSL**: Findet alle Wörter eines Suchausdrucks: **"John" AND "Doe"**. Unterstützt außerdem Wildcards und einfache Suchoperatoren [(Dokumentation)](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html).
- **Fuzzy Query**: Fuzzy-Suche, welche einzelne Begriffe findet, auch wenn sie Buchstabendreher enthalten **Jhon** [(Dokumentation)](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html).
- **Regexp Query**: Regex-Suche für einzelne Begriffe **J.hn*** [(Dokumentation)](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html).

Weitere Informationen zur Suche in Elasticsearch gibt es der [Dokumentation](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_the_search_api.html). Grundsätzliche Optionen, um die Ergebnisse einer Suche zu filtern, finde sich [hier](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/common-options.html).

Ein einfaches Implementierungsbeispiel um mit [elasticsearch.js](https://github.com/elastic/elasticsearch-js) Dokumente im Index `my-index` nach `John Doe` zu durchsuchen:

```javascript
const elastic = require('elasticsearch');
const client = new elastic.Client({ host: 'localhost:9200' });

client.search({
  'my-index',,
  size: 500, // Limit number of results
  body: {
    query: {
      simple_query_string: {
        'John Doe',
        fields: ['body','body.folded'],
        default_operator: 'and',
        analyze_wildcard: true
      }
    },
    {
      excludes: ['body*'] // Exclude document body from response
    },
    fields: {
      body: {} // Add highlighted matches to response
    }
  }
}, (error, result) => {
  if (error) {
    console.error(error)
  } else {
    console.log(result);
  }
});
```

Die [elasticsearch-import-tools](https://github.com/br-data/elasticsearch-import-tools/) beinhalten einen kleinen, vorkonfigurierten Express-Server ([server.js](https://github.com/br-data/elasticsearch-import-tools/blob/master/server.js)), welcher die wichtigsten Suchfunktionen in einer einfache REST-API verpackt. Einmal gestartet kann man Suchanfragen vereinfacht stellen: 

```
$ curl http://localhost:3003/match/John%20Doe
```

### Frontend

Die Dokumentation für das BR-Data Elasticsearch Frontend sich hier: [elasticsearch-frontend](https://github.com/br-data/elasticsearch-frontend).

### Zukunftsausblick

- Autovervollständigung
- Named Entity Recognition
- Dokumenten-Import
- Zentralisierter Data Store
