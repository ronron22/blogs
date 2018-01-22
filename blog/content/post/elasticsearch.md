---
title: Aide mémoire Elasticsearch
tags:
  - elasticsearch
  - howto
  - "aide mémoire"
  - admin
  - bdd
  - nosql

date: 2017-02-02
categories:
  - admin
  - Bdd

#nomenu = "main"
#image: img/Cassandra-Logo.jpg
---

# Elasticsearch

## Littérature

* http://soat.developpez.com/articles/elasticsearch/

* http://fr.slideshare.net/dadoonet/elasticsearch-devoxx-france-2012/


## concepts

### terminologie et architecture


* noeud : une instance d’ElasticSearch
* cluster : un ensemble de noeuds
* index : un index est une collection de documents qui ont des caractéristiques similaires.
* shards : partitions ou sont stockés les documents constituant un index, un index peut s'appuyer sur plus d'un shard.
* replica : copie additionnel d'un index dans un cluster ElasticSearch

*  partition primaire (primary shard) : qui correspond à la partition
    élue principale dans l'ensemble du cluster. C'est là que se fait
    l'indexation par Lucene. Il n'y en a qu'une seule par shard dans
    l'ensemble du cluster ;
*   partition secondaire (secondary shard) : qui sont les partitions
    secondaires stockant les réplicas des partitions primaires.

Par défaut, chaque index est répartie sur 5 shards primaires et 1
réplica.

Rivières (Rivers) : permettent de "verser" des données dans ES.

Le replicat et le Shard original ne peuvent pas être sur le même
    node

### cas concret

    curl -XGET http://localhost:9200/_cat/shards 
    graylog2_8                 0 p STARTED 5013226   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 3 p STARTED 5005312   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 1 p STARTED 5001062   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 2 p STARTED 4981813   2.4gb 127.0.0.1 Aquarian 
    apache_logstash-2015.08.04 2 p STARTED    6955   2.2mb 127.0.0.1 Aquarian

Explication :

-   première colonne : nom de l'index
-   seconde : numéro du shard
-   troisième : type (primaire ou replica)
-   quatrième : statue
-   cinquième : id ??
-   sixième : taille
-   septième : adresse du nœud
-   huitième : nom du nœud

### roles et scalabilitee

Dans le cas d'un gros cluster, il est intéressant de séparer les rôles 

* Master-eligible node : qui peut devenir master et permet la création/suppression d'index et assure les fonctions de gestion du cluster 
* Data node : qui possède des données et effectue des opération CRUD dessus
* Client node : qui transmet les requêtes (et peut être les fusionnes ?)


Doc :
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html

#### Un dessin qui vaut mille mots 

https://www.elastic.co/assets/blt9ae0b9498554824c/cluster-topology.svg


### Installation et configuration

#### Installation

```bash
echo "deb http://packages.elastic.co/elasticsearch/1.6/debian stable main" > /etc/apt/sources.list.d/elastic.list && \
apt update && \
apt install elasticsearch && \
systemctl enable elasticsearch && \
systemctl start elasticsearch 
```

Par défaut, un *node name* est généré dynamiquement par Elasticsearch

Une dépendance à java est requise

```bash
apt install openjdk-8-jre-headless
```

#### Mise en cluster

-   définir un *cluster.name* sur chacun des noeuds ex

`cluster.name: kaiju`

-   définir un *node.name* sur chacun des noeuds ex

`node.name: "Godzilla" `

Attention, si vous mettez *node.master* à *true* sur le master et que le
slave tourne en standalone, ce dernier refusera les connexions si le
master est down.

### Haproxy

    frontend es_front 0.0.0.0:9200
            use_backend es_back

    backend es_back
            mode tcp
            balance leastconn
            option httpchk
            server es1 localhost:9201 check inter 5s weight 1 
            server es2 bricozor6:9201 check inter 5s weight 1 backup

### scénarios testés

```bash
  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  node 1                                              node 2                                              résultat
  --------------------------------------------------- --------------------------------------------------- ---------------------------------------------------------------------
  node offline                                        node online + /test/test/3 -d '{"key2":"value 2"}   ok après synchronisation\
                                                                                                          Pas d'actions manuelles requises

  node online + /test/test/4 -d '{"key2":"value 2"}   node offline                                        ok après synchronisation\
                                                                                                          Pas d'actions manuelles requises

  node online + /test/test/5 -d '{"key2":"value 2"}   node online + /test/test/5 -d '{"key2":"value 2"}   ok après synchronisation, il prend la modification la plus récente\
                                                                                                          Pas d'actions manuelles requises
  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

  : Conditions testées

Installation de plugins
-----------------------

### head

`cd /usr/share/elasticsearch/bin && \`\
`./plugin --install mobz/elasticsearch-head`

Accès <http://localhost:9200/_plugin/head/>

### HQ

`cd /usr/share/elasticsearch/bin && \`\
`./plugin -install royrusso/elasticsearch-HQ`

Accès <http://localhost:9200/_plugin/HQ/>

### Bigdesk


```bash
cd /usr/share/elasticsearch/bin && \
./plugin -install lukas-vlcek/bigdesk
```

Accès http://localhost:9200/_plugin/bigdesk/

## exploitation

###  Obtenir des stats approfondies

```bash
curl localhost:9200/_stats | python -m json.tool | less
```

### Arrêt du cluster

```bash
curl -XPOST `http://localhost:9200/_shutdown
```

### Status du cluster

```bash
curl "http://localhost:9200/_cluster/health?pretty"
```

### Status des nodes

```bash
curl localhost:9200/_cat/nodes
prdlamp2 10.0.95.12 71 56 0.92 d m prdlamp2 
prdlamp1 10.0.95.11 49 57 1.25 d * prdlamp1 
test1    10.0.95.90  2 60 0.02 - - test1
```

Explication :

* première colonne : nom du noeud
* seconde : adresse IP
* troisième : ??
* quatrième : ??
* cinquième : ??
* sixième : ??
* septième : "m" pour master elligible, "*" pour master effectif et - ??

### état des index (indices)

```bash
curl 'localhost:9200/_cat/indices?v'
health status index                 pri rep docs.count docs.deleted store.size pri.store.size 
green  open   village              1   1    2541204       364644    196.7mb         97.3mb 
green  open   preprod_village      1   1     905310            0     94.1mb           47mb 
green  open   moderation           1   1       6183            5     16.1mb            8mb 
green  open   preprod_tags         1   1       2178            0    578.8kb        289.4kb 
green  open   log                  1   1    1117393            0    156.5mb         78.2mb 
green  open   categories           1   1        344            0    226.8kb        113.4kb 
green  open   preprod_moderation   1   1          2            0     28.4kb         14.2kb 
green  open   ags                 1   1       2601            0    733.1kb        357.1kb 
green  open   preprod_log          1   1     709760            0     85.1mb         42.5mb 
yellow open   classified           1   1    2294880       172341      1.3gb          1.3gb 
green  open   preprod_classified   1   1      18749         3597     28.9mb         14.4mb 
green  open   preprod_categories   1   1        344            0    221.8kb        110.9kb
```

Ici, on peut voir que l'index "classified" est en anomalie (Yellow status)

Pour information :
 * store.size : store size of primaries & replicas 
 * pri.store.size : store size of primaries

### Status des shards (et replicas)

```bash
curl http://localhost:9200/_cat/shards
```

### vérifier le mapping
Mapping is the process of defining how a document, and the fields it contains, are stored and indexed

```bash
curl localhost:92009.collection/_mapping?pretty
```

### obtenir la version 

```bash
curl localhost:9200
```

## Troubleshooting

### Elastichsearch est en rouge

Si le serveur Elastic tourne en standalone, je pense que les replicas
shards ne servent à rien (pas d'autres noeuds), il est donc possible de
les désactiver :

    curl -XPUT 'localhost:9200/mon_index_de_la_mort/_settings' -d '
    {

        "index" : {
            "number_of_replicas" : 0
        }
    }'

Sinon

1\) vérifier si des shards sont orphelins ("unassigned\_shards" : 2,)

    curl http://localhost:9200/_cluster/'health?pretty'          
    {
      "cluster_name" : "nom du cluster",
      "status" : "red",
      "timed_out" : false,
      "number_of_nodes" : 1,
      "number_of_data_nodes" : 1,
      "active_primary_shards" : 77,
      "active_shards" : 77,
      "relocating_shards" : 0,
      "initializing_shards" : 0,
      "unassigned_shards" : 2,
      "number_of_pending_tasks" : 0,
      "number_of_in_flight_fetch" : 0
    }

2\) si unassigned\_shards est différent de 0, récupérer la liste des
shards "UNASSIGNED" et le nom du node avec un

    curl http://localhost:9200/_cat/shards 
    graylog2_8                 0 p STARTED 5013226   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 3 p STARTED 5005312   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 1 p STARTED 5001062   2.4gb 127.0.0.1 Aquarian 
    graylog2_8                 2 p STARTED 4981813   2.4gb 127.0.0.1 Aquarian 
    apache_logstash-2015.08.04 2 p STARTED    6955   2.2mb 127.0.0.1 Aquarian

Explication :

-   première colonne : nom de l'index
-   seconde : numéro du shard
-   troisième : type (primaire ou replica)
-   quatrième : statue
-   cinquième : id ??
-   sixième : taille
-   septième : adresse du nœud
-   huitième : nom du nœud

2.1) récupérer le numéro du shard non assigné (seconde colonne)

2.2) récupérer le nom de l'index non assigné (première colonne)

2.3) récupérer le nom du node (dernière colonne)

3) enfin, avec tous les éléments récupérés, réassocier le shard orphelin
au noeud(-noeud) :

    curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
            "commands" : [ {
                  "allocate" : {
                      "index" : "nom de l'index", 
                      "shard" : numéro du shard, 
                      "node" : "nom du node", 
                      "allow_primary" : true
                  }
                }
            ]
        }'; 

Si le serveur Elastic tourne en standalone, je pense que les replicas
shards ne servent à rien (pas d'autres noeuds), il est donc possible de
les désactiver :
```
index.number_of_replicas: 0
```
### performance
Données de performances :

`curl "http://localhost:9200/_nodes/stats/jvm?pretty"̀


# Measure

## anchors-in-markdown

## Sauvegarde

récupérer le nom du dépôt de backup avec :

`curl -s 'http://localhost:9200/_cluster/state?pretty' | less`

Puis, récupérer la liste des snapshot

`curl -s 'http://localhost:9200/_snapshot/es_backup/_all?pretty' | less`

## would have more than the allowed 10% free disk threshold

```
curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.disk.threshold_enabled" : false
    }
}'
```

## cluster.routing.allocation.disk.watermark 

https://www.elastic.co/guide/en/elasticsearch/reference/current/disk-allocator.html


## supprimer d'anciennes donnés

`curator delete --prefix packetbeat- --older-than 3̀

## This can result in part of the JVM being swapped out. Increase RLIMIT_MEMLOCK 

Ajout MAX_LOCKED_MEMORY=unlimited dans /etc/init.d/elasticsearch

### where expected to be resolved to a single node

message 
```
where expected to be resolved to a single node
```

Vérifier qu'il n'y a pas plusieurs instances en cours sur le node


Réglage pour un standalone :
* http://blog.lavoie.sl/2012/09/configure-elasticsearch-on-a-single-host.html


