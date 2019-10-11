
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

## slide => Performance

On a l'architectuer de prometheus qui est telle qu'on se retrouve avec un processus qui scrape, qui compresse, et qui permet de faire des requêtes

Heureusement pour nous, Prometheus est très puissant, et est capable de gérer en même temps une énorme quantité de métriques en ingestion et en requêtage.

Cependant, si vous travaiallez dans une entreprise florissante, ce que je vous souhaite, il arrivera peut être un jour où vous allez atteindre un point où un seul serveur, même très puissant, ne suffira plus

Pour donner un exemple extrême, chez digital océan, ils ont 200 serveurs prometheus, qui leur permettent d'absorber 2M de métriques par secondes. Ils donnent un talk très intéressant sur comment scaler Prometheus et ils donnent plein de pistes intéressantes sur ce qu'on peut et ne peut pas faire avec Prom.

## Slide 7 => Failure domain

Même dans le cas où vous n'arrivez pas aux limites de Prom, il est très probable que vous soyez quand même obliger d'avoir plusieurs serveurs prometheus, pour une raison tout simple.

Pb 1 => Prometheus impose d'avoir un serveur par zone (failure domain)

Si on a plusieurs datacenter, on doit donc avoir plusieurs prometheus..

## slide manual sharding

Vous allez donc devoir découper votre ensemble de métriques. La question c'est comment ?

Point Captain obvious : il est déconseillé de stocker des métriques que vous souhaitez correler sur des Prometheus différents.

Dans le meilleur des cas, ça sera juste difficile à grapher dans Grafana

workaround => ajouter une valeur dans grafana pour swithcer d'une source à l'autre, voire même mettre plusieurs sources dans un même graphe

Mais dans le cas où vous souhaitez faire des requêtes complexes en PromQL permettant de correler plusieurs timeseries entre elles, vous ne pourrez pas le faire.

workaround 2 => il existe une feature appelée fédération, qui permet à un prometheus global de scrapper plusieurs prometheus/

Cependant, autant avec des prometheus dans chaque failure domaine on ne va pas avoir trop de souci de charge, autant si vous centralisez tout sur un même endroit sur lequel vous effectuez toutes vos requêtes, vous risque d'écrouler Prometheus.

CPU de tous les containers, Granularité 5secondes, sur 1 an

## SPOF

quelques slides plus tôt, j'ai dis "on se retrouve avec un processus qui scrape, qui compresse, et qui permet de faire des requêtes"

Normalement, dans votre tête quand j'ai dis ça, il devrait y avoir un panneau lumineau rouge avec SPOF qui s'allume.

Oui, par défaut, prometheus est un bon gros SPOF. 


## On va tout doubler

Donc pour pallier à ça c'est pas foufou mais la réponse des Dev de Prometheus c'est de dire, "spa grave, c'est Kube. Mettez en deux, et puis bon ils scrappent les mêmes data tant pis.

Je passe sur le fait qu'on a deux serveurs qui font exactement la même chose et qui scrappent 2 fois la même données... C'est pas le sujet.

Dans ce cas de figure, on se retrouve avec pour chaque server prometheus, 2 replicas, qui peuvent être ajouté dans Grafana. Théoriquement, les données contenues dans ces deux réplicas sont les mêmes, donc je peux passer de l'une à l'autre sans voir de différence.

Mais c'est pas hyper pratique car c'est à moi de choisir explicitement celle que je veux. Et si un dex deux serveurs Prometheus est down, c'est à moi de changer explicitement de source.

Du coup, première idée, on met un loadbalancer devant. C'est pas bête, ça permet en plus de répartir la charge sur les queries dans grafana, puisque la moitié des requêtes seront addressées sur chacun des replica.

## Théoriquement, c'est la même data

J'ai dis théoriquement, car si pour une raison ou pour un autre (un bug, une requête trop gourmande qui écroule le prometheus, un upgrade) un des replica a été down pendant quelques minutes il y aura un trou dans les métriques qu'il a scrapé.

Du coup, dans Grafana, comment je fais pour choisir lequel des deux réplicas je consulte ? Et pire, si j'ai mis un loadbalancer devant, en fonction du replica sur lequel je tombe, j'aurai (ou pas) un trou.

Donc on voit bien que c'est pas utilisable en l'état.



## Slide XXX => Thanos, un ami qui vous veut du bien

## Slide XXX => architecture de Thanos

Sidecar à côté de prom, envoyer à des querier, bucket, store, compactor

## backup Slide => Prometheus at scale, l'exemple de DO

* [https://www.youtube.com/watch?v=LmXWLNd2FTA]
DO
192 serveurs Prom
200millons time series
2 millions de samples / secondes

## backup Slide => protips

/!\ permutation de label => nouvelle TS
/!\ query max 100s timeseries
/!\ queries sur 1000s timeseries will crash prom
Work out your query in the console before graphing
Avoid high cardinality labels

## backup Slide => Les limites de Prom

Metrics will eventually get too big
Shard => pick a dimension that is a query boundary 
Don't split metrics that you want to query together
HA => faire des paires de serveurs Prom qui scrapent la même chose => loadbalancing queries
https://www.robustperception.io/which-are-my-biggest-mtrics

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

* [Thanos - Transforming Prometheus to a Global Scale in a Seven Simple Steps](https://www.youtube.com/watch?v=Iuo1EjCN5i4)
* [PromCon 2018: Thanos - Prometheus at Scale](https://www.youtube.com/watch?v=Fb_lYX01IX4)

* [Support de Thanos dans le prometheus Operator proposé par CoreOS](https://github.com/coreos/prometheus-operator)