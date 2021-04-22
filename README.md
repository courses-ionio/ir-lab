# IR-lab: Εργαστήριο ανάκτησης πληροφορίας ###

## Setting up elasticsearch and kibana on docker

Θα χρησιμοποιήσουμε το docker compose.  
Δημιουργήστε ένα yml αρχείο με όνομα `docker-compose.yml` και περιεχόμενο:  
```
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02
      - cluster.initial_master_nodes=es01,es02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01
      - cluster.initial_master_nodes=es01,es02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic

  kib01:
    image: docker.elastic.co/kibana/kibana:7.12.0
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: '["http://es01:9200","http://es02:9200"]'
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local

networks:
  elastic:
    driver: bridge
```

Δημιουργεί ένα cluster δύο elasticsearch nodes και ενός kibana node.  
Κάνει expose στο host το ένα elasticsearch node στο node 9200 και το kibana node στο port 5601.  
Όλα τα nodes επικοινωνούν μέσω του bridge network με ονομασία elastic. Εντός αυτού του δικτύου τα nodes είναι προσβάσιμα μέσω των host names: es01, es02 και kib01.
Στο host machine όμως είναι προσβάσιμα μόνο το `es01` _αλλά_ ως `localhost:9200` και το `kib01` ως `localhost:5601`.

Εκκίνηση των nodes με την εντολή: `docker-compose up` (στο φάκελο που βρίσκεται και το `docker-compose.yml`).


🕑 Μερικά λεπτά (και 🔥 με λίγη υπερθέρμανση ![overheating](./_img/overheating.png)) μετά..

```
$ curl -X GET "localhost:9200/_cat/nodes?v&pretty"
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
172.18.0.2           47          97  20    1.69    1.91     0.99 cdfhilmrstw *      es01
172.18.0.4           57          97  20    1.69    1.91     0.99 cdfhilmrstw -      es02
```

Το cluster τρέχει!  
_Σημειώνουμε για αργότερα, τερματισμός με: `docker-compose down` (ή `docker-compose down -v` για να διαγραφούν και τα αρχεία δεδομένων)._

---

Ο βασικός τρόπος αλληλεπίδρασης με το cluster είναι json δεδομένα.  

* Ας λάβουμε πληροφορίες για το cluster: `curl -X GET "localhost:9200/"`  
    ```
    {
      "name" : "es01",
      "cluster_name" : "es-docker-cluster",
      "cluster_uuid" : "laykn56KSVCP8JJ74uPJaQ",
      "version" : {
        "number" : "7.12.0",
        "build_flavor" : "default",
        "build_type" : "docker",
        "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
        "build_date" : "2021-03-18T06:17:15.410153305Z",
        "build_snapshot" : false,
        "lucene_version" : "8.8.0",
        "minimum_wire_compatibility_version" : "6.8.0",
        "minimum_index_compatibility_version" : "6.0.0-beta1"
      },
      "tagline" : "You Know, for Search"
    }
    ```

* Ας προσθέσουμε ένα έγγραφο:  
    `curl -X POST --header "Content-Type: application/json" --data '{"name" : "John", "lastname" : "Doe", "job_description" : "Systems administrator and Linux specialit"}' "localhost:9200/accounts/person/1" `
    Λαμβάνουμε response:  
    ```
    {
      "_index": "accounts",
      "_type": "person",
      "_id": "1",
      "_version": 1,
      "result": "created",
      "_shards": {
        "total": 2,
        "successful": 2,
        "failed": 0
      },
      "_seq_no": 0,
      "_primary_term": 1
    }
    ```

* Ας ανακτήσουμε το έγγραφο που προσθέσαμε:  
    `curl -X GET "localhost:9200/accounts/person/1`
    Λαμβάνουμε response:  
    `{"_index":"accounts","_type":"person","_id":"1","_version":1,"_seq_no":0,"_primary_term":1,"found":true,"_source":{"name" : "John", "lastname" : "Doe", "job_description" : "Systems administrator and Linux specialit"}}`

    * Ας ανακτήσουμε το έγγραφο που προσθέσαμε (_πιο όμορφα!_):  
        `curl -X GET "localhost:9200/accounts/person/1?pretty"`
        Λαμβάνουμε response:  
        ```
        {
          "_index" : "accounts",
          "_type" : "person",
          "_id" : "1",
          "_version" : 1,
          "_seq_no" : 0,
          "_primary_term" : 1,
          "found" : true,
          "_source" : {
            "name" : "John",
            "lastname" : "Doe",
            "job_description" : "Systems administrator and Linux specialit"
          }
        }        
        ```
    Το αποτέλεσμα περίεχει τόσο μεταδεδομένα όσο και το αρχικό κείμενο (`_source`).

* Ας κάνουμε μια αναθεώρηση (διόρθωση) στο κείμενο:  
    `curl -X POST --header "Content-Type: application/json" --data '{"doc":{ "job_description" : "Systems administrator and Linux specialist" }}' "localhost:9200/accounts/person/1/_update" `

    * Ας ανακτήσουμε και πάλι το έγγραφο που προσθέσαμε:  
        `curl -X GET "localhost:9200/accounts/person/1?pretty"`
        Λαμβάνουμε response:  
        ```
        {
          "_index" : "accounts",
          "_type" : "person",
          "_id" : "1",
          "_version" : 2,
          "_seq_no" : 1,
          "_primary_term" : 1,
          "found" : true,
          "_source" : {
            "name" : "John",
            "lastname" : "Doe",
            "job_description" : "Systems administrator and Linux specialist"
          }
        }
        ```
    Παρατηρήστε την αλλαγή στο `_version`.

* Ας προσθέσουμε ένα ακόμη έγγραφο:  
    `curl -X POST --header "Content-Type: application/json" --data '{"name" : "John", "lastname" : "Smith", "job_description" : "Systems administrator"}' "localhost:9200/accounts/person/2" `
    Λαμβάνουμε response:  
    ```
    {
      "_index": "accounts",
      "_type": "person",
      "_id": "2",
      "_version": 1,
      "result": "created",
      "_shards": {
        "total": 2,
        "successful": 2,
        "failed": 0
      },
      "_seq_no": 2,
      "_primary_term": 1
    }
    ```
    Παρατηρήστε το `_version` και το `_seq_no`.

* Ας κάνουμε την πρώτη αναζήτησή μας:  
    `curl -X GET "localhost:9200/_search?q=john&pretty" `
    Λαμβάνουμε response:  
    ```
    {
      "took" : 4007,
      "timed_out" : false,
      "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 2,
          "relation" : "eq"
        },
        "max_score" : 0.18232156,
        "hits" : [
          {
            "_index" : "accounts",
            "_type" : "person",
            "_id" : "1",
            "_score" : 0.18232156,
            "_source" : {
              "name" : "John",
              "lastname" : "Doe",
              "job_description" : "Systems administrator and Linux specialist"
            }
          },
          {
            "_index" : "accounts",
            "_type" : "person",
            "_id" : "2",
            "_score" : 0.18232156,
            "_source" : {
              "name" : "John",
              "lastname" : "Smith",
              "job_description" : "Systems administrator"
            }
          }
        ]
      }
    }    
    ```
    Λαμβάνουμε ως αποτέλεσμα και τα δύο documents.

* Δοκιμάστε να αναζητήσετε:
    * το string `smith`
    * το string `job_description:john`
    * το string `job_description:linux`

* Διαγραφή ενός εγγράφου:  
    `curl -X DELETE "localhost:9200/accounts/person/1?pretty"`
    Λαμβάνουμε response:  
    ```
    {
      "_index" : "accounts",
      "_type" : "person",
      "_id" : "1",
      "_version" : 3,
      "result" : "deleted",
      "_shards" : {
        "total" : 2,
        "successful" : 2,
        "failed" : 0
      },
      "_seq_no" : 3,
      "_primary_term" : 1
    }
    ```
    Ελέγξτε κάνοντας αναζήτηση: `curl -X GET "localhost:9200/_search?q=john&pretty" `

* Διαγραφή ενός ολόκληρου index:  
    `curl -X DELETE "localhost:9200/accounts?pretty"`
    Λαμβάνουμε response:  
    ```
    {
      "acknowledged" : true
    }    
    ```

Εγκαταστήστε στον υπολογιστή σας ένα πιο εύχρηστο [REST API client](https://www.postman.com/), πχ Postman και εκτελέστε ανάλογες εισαγωγές, τροποποιήσεις, διαγραφές.

---

Για την μαζική εισαγωγή εγγράφων στο elasticsearch node μας θα αξιοποιήσουμε το logstash.  
Ξεκινάμε το elasticsearch node με χρήση του __docker-compose.yml__: `docker-compose up`.  
Ξεκινάμε και το logstash με χρήση του __logstash.yml__: `docker-compose -f logstash.yml up`
Το νέο yml αρχείο δημιουργεί και ένα logstash node:
```
ls01:
    image: docker.elastic.co/logstash/logstash:7.12.0
    container_name: ls01
    volumes:
    - ./data:/app
    networks:
      - elastic
```
Το ls01 μπορεί να επικοινωνήσει με το es01 καθώς και τα δύο χρησιμοποιούν το elastic network.  

Με τη βοήθεια το logstash μπορούμε να εισάγουμε csv αρχεία στο elasticsearch node, πχ το `data/Wired.csv`.
Για να πετύχουμε αυτό θα δημιουργήσουμε ένα κατάλληλο configuration αρχείο, το `Wired.conf`:
```
input {
    file {
        path => "/app/Wired.csv"
        start_position => beginning
    }
}
filter {
    csv {
        columns => [
                "a_link",
                "a_text"
        ]
        separator => ";"
        }
}
output {
    stdout
    {
        codec => rubydebug
    }
     elasticsearch {
        action => "index"
        hosts => ["es01:9200"]
        index => "wired"
    }
}
```

Η εκτέλεση για την εισαγωγή του csv αρχείο γίνεται από γραμμή εντολών. Εκτελούμε λοιπόν:
* Είσοδο στο logstash container: `docker exec -it ls01 /bin/bash`
* Μετάβαση στο volume που περιέχει τα δεδομένα (`.csv` αρχείο) και το configuraion (`.conf` αρχείο): `cd /app`
* Εκτέλεση εισαγωγής: `/usr/share/logstash/bin/logstash -f /app/Wired.conf --path.data /app/data`

Ελέγχουμε εάν το νέο index δημιουργήθηκε σωστά: `curl -X GET "http://localhost:9200/_cat/indices?v"`  
Μια επιτυχής εισαγωγή θα δείχνει (κάτι σαν):
```
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   wired                           pWt-upjHR4aFp_8we80oJA   1   1        108            0      115kb          115kb
```

Πλέον μπορούμε να κάνουμε αναζητήσεις μέσα σε αυτό το index.
Πχ:
* `curl -X GET "localhost:9200/_search?q=listen&pretty"`
* `curl -X GET "localhost:9200/_search?q=and&pretty"`
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "match" : { "a_text" : "vacc" } }}' "localhost:9200/_search?pretty"`
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "prefix" : { "a_text" : "vacc" } }}' "localhost:9200/_search?pretty"`
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "bool" : { "must" : { "prefix" : { "a_text" : "vacc" } }, "must_not" : { "match" : { "a_text" : "Covid-19" } } } }}' "localhost:9200/_search?pretty"`
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "bool" : { "must" : { "prefix" : { "a_text" : "vacc" } }, "must_not" : { "prefix" : { "a_text" : "covid" } } } }}' "localhost:9200/_search?pretty"`
ή ανάλογα παραδείγματα με χρήση Postman αντί για curl.


Για ακόμη καλύτερη αναζήτηση μέσα στα δεδομένα μας.. αξιοποίηση __kibana__.  
Σταματάμε το πρώτο docker-compose (αυτό με το __docker-compose.yml__) με `Ctrl+C`.  
Κάνουμε uncomment τις γραμμές του kibana container και ξεκινάμε πάλι το docker compose. (Θα πάρει λίγο χρόνο _και χώρο_).


Ανοίγουμε σε ένα browser: http://localhost:5601/ και εξερευνούμε το Discover interface.  
Μπορούμε κάνουμε αναζητήσεις με λογικές πράξεις, match/prefix mode, ... και πολλά άλλα.

---
Sources:  
* https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html
* https://www.elastic.co/blog/a-practical-introduction-to-elasticsearch
* https://www.compose.com/articles/how-scoring-works-in-elasticsearch/
