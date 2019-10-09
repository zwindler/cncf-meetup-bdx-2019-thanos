
# Contenu des slides

## Slide 1 => slide d'intro

Le titre et les logos

## Slide 2

une grosse photo de Thanos

Bon j'ai du faire un gros boulot pour préparer ce meetup, j'ai du me mettre à jour et regarder tous les avengers pour savoir qui est Thanos.

## Slide 3

Ma présentation

## Slide 4

Présentation de Lectra

## Slide 5 

Table de matière ?

## Slide 6 => Previously in Prometheus

Bref rappel de Prometheus, ce que ça fait, à qui ça s'adresse

## Slide XXX => architecture de Prometheus

## Slide 7 => "C'est bien, mais pas suffisant"

Limitations de Prometheus

1 seul process qui scrape, qui compresse, et qui permet de faire des requêtes
Ca devient un problème quand on commence à récolter BEAUCOUP de métriques !!!
Scalabilité verticale
Scalabilité horizontale => découper en plusieurs prom, manual sharding
On peut pas avoir toutes les sources dans le même graphe

## Slide => Prometheus at scale, l'exemple de DO

* [https://www.youtube.com/watch?v=LmXWLNd2FTA]
DO
192 serveurs Prom
200millons time series
2 millions de samples / secondes

## Slide => protips

/!\ permutation de label => nouvelle TS
/!\ query max 100s timeseries
/!\ queries sur 1000s timeseries will crash prom
Work out your query in the console before graphing
Avoid high cardinality labels

## Slide => Les limites de Prom

Metrics will eventually get too big
Shard => pick a dimension that is a query boundary 
Don't split metrics that you want to query together
HA => faire des paires de serveurs Prom qui scrapent la même chose => loadbalancing queries
https://www.robustperception.io/which-are-my-biggest-mtrics

## Slide XXX => Thanos, un ami qui vous veut du bien

## Slide XXX => architecture de Thanos

Sidecar à côté de prom, envoyer à des querier, bucket, store, compactor

## Backup slide => Thanos vs Cortex

* [https://www.youtube.com/watch?v=iyN40FsRQEo]

*Points communs*
Réutilisent en grande partie Prometheus
Compatibles avec l'écosystème existant
Savent aggréger les données de plusieurs Prometheus
Capable de gérer de manière distincte les données récentes vs historiques
Permettent de gérer les données historiques dans des buckets
Architecture multi composants

*Différences*

Automatic versus manual sharding ???
Indexezd small chunks vs Prom TSDB blocks ???

Cortex propose de faire du query sharding pour accélerer le rendu des grosses requêtes
Cortex est nativement multi-tenant

Thanos offre la fonctionnalité de downsampling pour réaliser des requêtes extrèmement rapides sur des temps très long

## TRASH

https://thanos.io/quick-tutorial.md/#deduplicating-data-from-prometheus-ha-pairs

*[https://www.youtube.com/watch?v=Fb_lYX01IX4]

Prometheus => local storage
Scale-out => performance but not only / isolated cluster 
same failure domain

Query that aggregate ??
Prometheus federation => scrape all the leaf prom
Double scrape
Federated prometheus might explode
HA ? Doubler les replicas de prom

1st pb => metric retention !!! super old (month old)

Promtheus => big SSD or remote write
cost / usability or remote KK
Thanos same query API
Querier can merge multiple sidecar and can point to replicas that find the same target

\-/ 2h, prometheus dump les données dans un nouveau block qui contient des compressed tsdb
Contient des chunks et un index
Compact toutes les 2 heures pour gagner encore en stockage
On peut utiliser Thanos pour les envoyer dans de l'object storage

Retirer de la rétention sur prom

Le querier utilise un composant de storage temporaire qui ne contient que les métadata ce qui permet d'aller chercher que ce qu'on a besoin sur l'object storage

downsampling, @5min et @1h, sans suppression et avec des fonctions genre count, sump, min max counter

* [https://www.youtube.com/watch?v=Iuo1EjCN5i4]


* [Support de Thanos dans le prometheus Operator proposé par CoreOS](https://github.com/coreos/prometheus-operator)