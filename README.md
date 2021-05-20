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

## Setting up logstash and kibana on docker

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
## _More data.. more fun!_

Αξιοποιώντας πιο πλούσια δεδομένα μπορούμε να εκμεταλλευτούμε δυνατότητες aggregation και visualisation που μας παρέχουν τα elasticsearch και kibana.  
* Εγγραφείτε στο https://data.gov.gr/token/. (_χρησιμοποιήστε το @ionio.gr e-mail σας_)
* Μετά τη λήψη του κλειδιού σας κατεβάστε ένα dataset, πχ [δεδομένα εμβολιασμού](https://www.data.gov.gr/datasets/mdg_emvolio/) και αποθηκεύστε τα σε ένα αρχείο  
`curl --request GET 'https://data.gov.gr/api/v1/query/mdg_emvolio?date_from=2021-04-26&date_to=2021-05-03' --header 'Authorization: Token 000000000000000000000000000000000' > Vaccinations.json`  
_Αντί για `000..00` χρησιμοποιήστε το δικό σας token και επιλέξτε και τις ημερομηνίες που θέλετε._
* Αξιοποιήστε το πιο κάτω conf αρχείο για να εισαγετε τα δεδομένα σας στο elasticsearch:
```
input {
    file {
        codec => json
        path => "/app/Vaccinations.json"
        start_position => beginning
        sincedb_path => "/dev/null"
    }
}
filter {
    date {
        match => [ "referencedate", "yyyy-MM-dd'T'HH:mm:ss"]
        target => "@timestamp"
    }
}
output {
    stdout
    {
        codec => rubydebug { metadata => true }
    }
    elasticsearch {
       action => "index"
       hosts => ["es01:9200"]
       index => "vaccinations"
   }
}
```  
με χρήση της εντολής: `/usr/share/logstash/bin/logstash -f /app/Vaccinations.conf --path.data /app/vacc`.  

* Hints:
    * Αν η εισαγωγή δεν.. γίνεται, διαγράψτε το φάκελο /app/vacc, προηγούμενες απόπειρες ίσως τη μπλοκάρουν.
    * Το logstash και το kibana δε χρειάζεται να εκτελούνται ταυτόχρονα (αν έχετε ένα σύστημα με.. moderate resources ;-)  

* Είσοδος στο kibana: http://localhost:5601/
    * Για να αξιοποιήσει ένα index του elasticsearch το kibana χρειάζεται να ορίσει πρώτα ένα index pattern `Menu > Stack management > [Kibana] Index patterns`.
        * Επιλογή `vaccinations` index
        * Επιλογή time field (@timestamp)
        * Σύνοψη ποια fields είναι searchable και aggregatable
    * Αξιοποίηση Kibana Discover interface `Menu > [Analytics] Discover`.
        * Αλλάξτε το time period από last 15 minutes σε ό,τι χρειάζεται για να βλέπετε τα δεδομένα που έχετε φορτώσει στο index.
        * Από το μενού στα αριστερά επιλέξτε ποια fields θελετε να βλέπετε.
        * Δοκιμάστε αναζητήσεις που σας ενδιαφέρουν
    * Αξιοποίηση Kibana Dashboard interface `Menu > [Analytics] Dashboard`.
        * Δημιουργήστε ένα νέο Lens panel το οποίο να αναπαριστά ως Line graph τις 10 περιοχές με τα περισσότερα εμβόλια ανά ημέρα:  
        ![Vaccination line grapgh](./_img/vacc_lines.png)
        * _εάν προσθέσετε νέα δεδομένα στο elasticsearch node το line graph ενημερώνεται αυτόματα_


---
## _Not just filtering; welcome aggregating_

Εκτός από filtering (discovery και visualisation) των δεδομένων μας, μπορούμε να κάνουμε και ποικίλες πράξεις συνόλων (aggregations) πάνω στα δεδομένα, ανάλογα με αυτές που γνωρίζεται από την SQL.  

Θα χρησιμιποιήσουμε ένα πιο πλούσιο dataset το οποίο έχουμε συλλέξει από τις πηγές RSS https://www.wired.com/feed/rss και https://www.wired.co.uk/feed/rss με τη βοήθεια του Knime.

* Αξιοποιήστε το πιο κάτω conf αρχείο για να εισαγετε τα δεδομένα σας στο elasticsearch:
```
input {
    file {
        path => "/app/Wired.RSS.csv"
        start_position => beginning
    }
}
filter {
    csv {
        columns => [
                "Source",
                "Title",
                "Published",
                "Category",
                "Sub_category",
                "Creator",
                "Subject",
                "Keywords",
                "Item_Url",
                "Description"
        ]
        separator => ";"
    }
    date {
        match => [ "Published", "yyyy-MM-dd'T'HH:mm", "yyyy-MM-dd'T'HH:mm:ss"]
        target => "@timestamp"
    }
    mutate {
        split => { "Keywords" => "," }
    }
    mutate {
        strip => [ "Keywords", "Keywords" ]
    }
}
output {
    stdout
    {
        codec => rubydebug { metadata => true }
    }
    elasticsearch {
       action => "index"
       hosts => ["es01:9200"]
       index => "wired"
   }
}
```

Πλέον μπορούμε να κάνουμε αναζητήσεις μέσα σε αυτό το index.  
Ας ξεκινήσουμε απλά:
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "match": { "Title": "Covid" } } }' "localhost:9200/_search?pretty"`
_όλα καλά;_
Λίγο πιο σύνθετα (τίτλος και χρόνος δημοσίευσης):
* `curl -X GET -H "Content-Type: application/json" --data '{ "query": { "bool":{ "must":[{ "match": { "Title": "Covid" } }, {"range":{ "@timestamp": {"gte": "now-15d/d", "lt": "now-10d/d"} } } ]} } }' "localhost:9200/_search?pretty"`


Και συνεχίζουμε με aggregations:
* _Επιτρέπουμε_ aggregations στο text field `Category`:  
`curl -X PUT -H "Content-Type: application/json" --data '{ "properties": { "Category" : {"type":"text", "fielddata":true}} }' "localhost:9200/wired/_mapping?pretty"`
* _Εκτελούμε_ ένα υπολογισμό πλήθους ανά κατηγορία:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Articles_by_categoty" : {"terms" : {"field":"Category"}}} }' "localhost:9200/wired/_search?pretty"`
* _Εκτελέστε ανάλογο σύνολο ανά Keyword_

Και aggregations σε πιο πολλά αριθμητικά δεδομένα:
*  Εκτελούμε ένα υπολογισμό πλήθους ανά περιοχή εμβολιασμού:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Total_vaccination_days_by_area" : {"terms" : {"field":"area", "size":200}}} }' "localhost:9200/vaccinations/_search?pretty"`
*  Εκτελούμε ένα υπολογισμό πλήθους και εμφωλευμένο υπολογισμό αθροίσματος ανά περιοχή εμβολιασμού:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Total_vaccinations_by_area" : {"terms" : {"field":"area", "size":200}, "aggs": {"Sum_per_area": {"sum":{"field":"daytotal"}}}}} }' "localhost:9200/vaccinations/_search?pretty"`

Πιο περίπλοκα aggregations (πχ επί όρων με συγκεκριμένες ιδιότητες):
* Εκτελούμε ένα υπολογισμό πλήθους ανά κατηγορία και ζητούμε μόνο τις κατηγορίες που έχουν τουλάχιστον **Ν** σχετιζόμενο documents:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Articles_by_categoty" : {"terms" : {"field":"Category", "min_doc_count": 5}}} }' "localhost:9200/wired/_search?pretty"`
* Εκτελούμε ένα υπολογισμό πλήθους ανά κατηγορία και εμφωλευμένα ζητούμε σημαντικά keywords για καθεμία κατηγορία:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Category" : {"terms" : {"field":"Category"}, "aggs": {"Significant_keywords": {"significant_terms": {"field": "Keywords"}}}}} }' "localhost:9200/wired/_search?pretty"`  
_Τί κάνει ένα keyword σημαντικό για μια συγκεκριμένη κατηγορία;_
    * Μελετήστε το απόσπσμα της εξόδου:
    ```
    "Category" : {
        ...
        "Significant_keywords" : {
        ...
            "key" : "culture",
            "doc_count" : 22,
            "Significant_keywords" : {
              "doc_count" : 22,
              "bg_count" : 100,
              "buckets" : [
                {
                  "key" : "games",
                  "doc_count" : 5,
                  "score" : 0.5106257378984651,
                  "bg_count" : 7
                },
                ...
    ```
    Η συχνότητα εμφάνισης του keyword `games` σε όλα τα documents είναι:  
    `buckets['games'].bg_count / Category[*].bg_count` = 7/100 = **7%**  
    Η συχνότητα εμφάνισης του keyword `games` σε όλα τα documents που ανήκουν στην κατηγορία `culture` είναι:  
    `buckets['games'].doc_count / Category['culture'].doc_count` = 5/22 = **23%**  
    Αυτή η διαφορά κάνει το συγκεκριμένο keyword σημαντικό για τη συγκεκριμένη κατηγορία.
    * Πειραματιστείτε με τη χρήση `min_doc_count` ώστε να περιορίσετε/χαλαρώσετε το κριτήριο των ελάχιστων απαιτούμενων σχετικών documents.
* Η εκτέλεση ενός significant_terms aggregation σε ένα πεδίο ελέυθερου κειμένου (όχι keywords), ενδέχεται να φέρει πολλά χαμηλής ποιότητας αποτελέσματα:  
`curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Category" : {"terms" : {"field":"Category"}, "aggs": {"Significant_keywords": {"significant_terms": {"field": "Title"}}}}} }' "localhost:9200/wired/_search?pretty"`
    * Μπορούμε να απαιτήσουμε κάθε significant keyword να εμφανίζεται σε ένα μέγιστο αριθμό documents:  
    `curl -X GET -H "Content-Type: application/json" --data '{ "aggs": {"Category" : {"terms" : {"field":"Category"}, "aggs": {"Significant_keywords": {"significant_terms": {"field": "Title"}, "aggs":{ "rare_words": { "bucket_selector":{ "buckets_path":{ "doc_count": "_count"}, "script":{ "inline": "params.doc_count < 5"}}}}}}}} }' "localhost:9200/wired/_search?pretty"`  
    _με τον κίνδυνο βέβαια να περικόψουμε κάποια αποτελέσματα._
* Μπορούμε επίσης να χρησιμοποιήσουμε ένα significant_terms aggregator και για να προτείνουμε λέξεις που ίσως ταιριάζουν σε κάποια συγκεκριμένη είσοδο χρήση:  
`curl -X GET -H "Content-Type: application/json" --data '{ "query": { "match": { "Title": "How" } }, "aggs":{"Significant_keywords": {"significant_terms": {"field": "Description", "min_doc_count":1}} } }' "localhost:9200/_search?pretty"`  
    * Πειραματιστείτε με το `min_doc_count`



---
Sources:  
* https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html
* https://www.elastic.co/blog/a-practical-introduction-to-elasticsearch
* https://www.compose.com/articles/how-scoring-works-in-elasticsearch/
* https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html
* https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-aggregations-bucket-significantterms-aggregation.html
* https://www.elastic.co/guide/en/elasticsearch/reference/master/search-aggregations-bucket-significanttext-aggregation.html
* https://qbox.io/blog/a-deep-dive-into-significant-terms-and-significant-text-bucket-aggregations-in-elasticsearch/
