+++
title = "Comprendre la saturation d'une application Spring Boot avec la loi de Little"
series = ["Performance"]
date = "2026-10-31"
+++

L'objectif de cet article est de comprendre la loi de Little et de voir comment l'utiliser au quotidien pour comprendre les latences et évaluer la charge de notre système.

# La loi de Little

```
L = λ × W
```

- **L** est le nombre d'éléments présents dans le système
- **λ** est le débit
- **W** est le temps de traversée du système

En informatique, on parle plutôt de :
- **L** : la concurrence, c'est-à-dire le nombre de requêtes en cours de traitement
- **λ** : le débit, en requêtes par seconde (RPS)
- **W** : la latence

Autrement dit : **concurrence = débit × latence**.

Cette loi peut se lire dans les trois sens. Il suffit de connaître deux valeurs pour trouver la troisième :

| On cherche     | Formule     | Question à laquelle on répond                          |
|----------------|-------------|--------------------------------------------------------|
| La concurrence | `L = λ × W` | Combien de requêtes sont traitées en même temps ?      |
| La latence     | `W = L / λ` | Combien de temps une requête peut-elle mettre ?        |
| Le débit       | `λ = L / W` | Combien de requêtes par seconde peut-on traiter ?      |

Un dernier point, essentiel pour la suite : **la taille d'un pool (threads, connexions…) est la concurrence maximale qu'il accepte**. Quand on remplace L par la taille d'un pool, la loi de Little ne donne donc plus une valeur moyenne mais une limite : une latence maximale ou un débit maximal.

# L'exercice

Voici l'énoncé : *aujourd'hui, notre système reçoit 2 000 requêtes par seconde. Une requête met 40 ms à traverser le système, dont 5 ms passées dans la base de données. Aucune requête n'attend dans une file.*

On souhaite savoir à partir de quand notre application Spring Web (Tomcat), qui appelle une base de données, va s'effondrer.

Voici l'architecture de notre système, avec les valeurs par défaut de Spring Boot :

| Étage    | Ressource occupée                                    | Taille par défaut | Propriété                                    |
|----------|------------------------------------------------------|-------------------|----------------------------------------------|
| Tomcat   | Un thread pendant toute la requête                   | 200               | `server.tomcat.threads.max`                  |
| HikariCP | Une connexion JDBC pendant la requête SQL uniquement | 10                | `spring.datasource.hikari.maximum-pool-size` |

## Combien de requêtes sont traitées en même temps ?

On connaît le débit et la latence, on cherche la concurrence : `L = λ × W`.

- débit : 2 000 req/s
- latence : 40 ms, soit 0,04 s

Donc `L = 2 000 × 0,04 = 80`.

Il y a donc en permanence 80 requêtes dans notre système. Avec 200 threads Tomcat, on est large.

## Et le pool Hikari ?

Nous avons en permanence 80 requêtes dans notre système, mais seulement 10 connexions dans le pool Hikari. Comment ça tient ?

Parce que les 40 ms correspondent au temps total de traversée. Ce temps se découpe en plusieurs parties : la requête SQL, la logique métier Java, la sérialisation JSON, etc. Une requête n'occupe une connexion Hikari que pendant la requête SQL, c'est-à-dire 5 ms, pas pendant 40 ms.

Appelons **x** ce temps passé dans la base de données, et appliquons à nouveau `L = λ × W`, mais cette fois sur le pool Hikari uniquement :

```
L = λ (débit) × W (latence)
L = 2 000 × 0,005 = 10
```

Lorsqu'on recoit 2000req/s et qu'une requete met 5ms il faut donc 10 connexion en parrllèle pour les traiter. Ou autrement dit, une connexion peut gérer 200req/s

De plus, notre configuration de pool en a 10 : il est donc utilisé à **100 %**.

# La latence passe de 40 ms à 100 ms

Un jour, sans que le trafic change, la latence passe de 40 ms à 100 ms. Reprenons la loi de Little :

```
L = 2 000 × 0,1 = 200
```

Il y a maintenant en permanence 200 requêtes dans notre système. Le trafic n'a pas bougé, mais la charge réelle a été multipliée par 2,5. Et Tomcat a exactement 200 threads : il est utilisé à 100 %, sans aucune marge.

La question est maintenant : d'où viennent ces 60 ms en plus ? Pour y répondre, il faut regarder x, le temps de la requête SQL.

## Cas 1 : x reste à 5 ms, le problème vient de l'application

Calculons la concurrence sur le pool Hikari :

```
L = 2 000 × 0,005 = 10
```

On a toujours besoin de 10 connexions, et on en a 10. La base de données répond aussi vite qu'avant, le pool Hikari n'est pas en cause.

Les 60 ms en plus sont donc consommées ailleurs, côté Spring : la logique métier, la sérialisation JSON, une pause du Garbage Collector, un appel HTTP vers un autre service, etc. C'est là qu'il faut chercher.

Le système tient encore, mais Tomcat est désormais à 100 %, comme Hikari. La moindre milliseconde supplémentaire et les requêtes commenceront à attendre un thread.

## Cas 2 : x passe de 5 ms à 50 ms, la base de données sature

Calculons à nouveau la concurrence sur le pool Hikari :

```
L = 2 000 × 0,05 = 100
```

100 requêtes ont besoin d'une connexion au même moment, mais il n'y en a que 10. Les 90 autres attendent leur tour.

On peut aussi le voir sous l'angle du débit. Cette fois, on connaît la concurrence (la taille du pool) et la latence, on cherche le débit : `λ = L / W`.

```
λ = 10 / 0,05 = 200 req/s
```

Avec des requêtes SQL de 50 ms, le pool ne peut traiter que 200 req/s. Or il en arrive 2 000.

Le problème, c'est qu'une requête qui attend une connexion Hikari garde son thread Tomcat. Le temps d'attente s'ajoute donc à la latence totale, ce qui augmente encore la concurrence côté Tomcat. Tomcat sature à son tour, mais ce n'est qu'une conséquence : le vrai coupable est la base de données.

Contrairement au cas 1, la latence ne va d'ailleurs pas rester à 100 ms. Les 100 ms mesurées ne sont qu'une photo à un instant donné : comme il arrive dix fois plus de requêtes que le pool ne peut en traiter, la file d'attente grossit sans arrêt et la latence continue de monter jusqu'à l'effondrement.

## Comment connaître x en production ?

Pas besoin de deviner : Spring Boot Actuator expose déjà les bonnes métriques avec Micrometer.

| Métrique                       | Ce qu'elle mesure                                      | Cas 1 | Cas 2                    |
|--------------------------------|--------------------------------------------------------|-------|--------------------------|
| `hikaricp.connections.usage`   | Le temps pendant lequel une connexion est utilisée (x) | 5 ms  | 50 ms                    |
| `hikaricp.connections.pending` | Le nombre de requêtes qui attendent une connexion      | 0     | des dizaines, puis plus  |
| `tomcat.threads.busy`          | Le nombre de threads Tomcat occupés                    | 200   | 200                      |

Dans les deux cas, Tomcat est à 200 threads. C'est donc `hikaricp.connections.usage` et `hikaricp.connections.pending` qui permettent de savoir où chercher : dans le code de l'application, ou dans la base de données.

# À partir de quand le système s'effondre-t-il ?

Revenons à notre question de départ.

Une première réponse consiste à regarder Tomcat. On connaît la taille du pool (200 threads) et le débit (2 000 req/s), on cherche la latence maximale : `W = L / λ`.

```
W = 200 / 2 000 = 0,1 s
```

Tant qu'une requête traverse le système en moins de 100 ms, Tomcat tient la charge. C'est exactement la limite atteinte dans le cas 1.

Mais une requête traverse plusieurs pools, et chacun a sa propre limite. Il faut donc faire le même calcul pour le pool Hikari :

```
W = 10 / 2 000 = 0,005 s
```

Avec 2 000 req/s, une requête SQL doit donc durer au maximum 5 ms. C'est exactement le temps actuel de nos requêtes SQL : le pool Hikari n'a aucune marge. La moindre augmentation de la latence SQL suffit à faire tomber le système, bien avant que Tomcat n'atteigne ses 100 ms.

## La loi du goulot d'étranglement

**Le débit maximum d'un système est limité par sa ressource la plus contraignante.**

C'est ce qu'on appelle la loi du goulot d'étranglement (*bottleneck law*) :

```
λ_max = min(capacité de chaque ressource)
```

Jusqu'ici, nous sommes partis du débit (2 000 req/s) pour trouver l'élément qui allait casser en premier. Mais on peut aussi faire l'inverse : en connaissant le temps passé dans chaque pool, on peut estimer le débit maximum de notre système.

Pour chaque pool, la loi de Little donne `λ = L / W` :

| Pool     | Calcul        | Capacité max    |
|----------|---------------|-----------------|
| Tomcat   | 200 / 0,04 s  | 5 000 req/s     |
| HikariCP | 10 / 0,005 s  | **2 000 req/s** |

```
λ_max = min(5 000, 2 000) = 2 000 req/s
```

Notre système ne peut donc pas dépasser 2 000 req/s, c'est-à-dire exactement le trafic qu'il reçoit. C'est une autre façon de voir qu'il n'a aucune marge.

Ainsi, si demain nous recevons 3 000 req/s, notre pool HikariCP ne tiendra pas la charge. Pour la tenir, il faudrait au moins :

```
L = λ × W = 3 000 × 0,005 = 15 connexions
```

# Conclusion

La loi de Little se lit dans les trois sens, et nous les avons tous utilisés :
- avec le débit et la latence, on trouve la concurrence, et donc la taille de pool nécessaire : `L = λ × W`
- avec la taille d'un pool et le débit, on trouve la latence maximale : `W = L / λ`
- avec la taille d'un pool et la latence, on trouve le débit maximal : `λ = L / W`

Et comme une requête traverse plusieurs pools, il faut faire le calcul pour chacun d'eux : c'est toujours le plus contraignant qui fixe la limite du système.
