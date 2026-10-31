+++
title = "PART 1 - Comprendre la saturation d'une application Spring Boot avec la loi de Little"
series = ["Performance"]
date = "2026-10-31"
+++

Cet article part d'un constat simple : on peut prévoir quand un système va s'effondrer avec une seule formule. La loi de Little relie trois grandeurs qu'on manipule tous les jours — débit, latence, concurrence — et permet de raisonner sur la capacité d'une application sans sortir l'artillerie lourde.

## La loi de Little

```
L = λ × W
```

- **L** est le nombre d'éléments présents dans le système
- **λ** est le débit
- **W** est le temps de traversée du système

Traduit en termes plus familiers :
- **L** : la concurrence — le nombre de requêtes en cours de traitement à un instant donné
- **λ** : le débit, en requêtes par seconde (RPS)
- **W** : la latence

Dit autrement : **concurrence = débit × latence**.

La formule se manipule dans les trois sens. Deux valeurs connues suffisent pour retrouver la troisième :

| On cherche     | Formule     | Question                                               |
|----------------|-------------|--------------------------------------------------------|
| La concurrence | `L = λ × W` | Combien de requêtes sont traitées en même temps ?      |
| La latence     | `W = L / λ` | Combien de temps une requête peut-elle mettre ?        |
| Le débit       | `λ = L / W` | Combien de requêtes par seconde peut-on traiter ?      |

Un point clé pour la suite : **la taille d'un pool (threads, connexions…) fixe la concurrence maximale qu'il accepte**. En remplaçant L par cette taille, la loi de Little ne donne plus une valeur moyenne mais une borne : la latence maximale admissible, ou le débit maximal atteignable.

## L'exercice

Partons d'un cas concret : *notre système reçoit 2 000 requêtes par seconde. Chaque requête met 40 ms à traverser le système, dont 5 ms passées dans la base de données. Aucune requête n'attend dans une file.*

On veut déterminer à quel moment notre application Spring Web (Tomcat), qui appelle une base de données, va craquer.

Notre architecture, avec les valeurs par défaut de Spring Boot :

| Étage    | Ressource occupée                                    | Taille par défaut | Propriété                                    |
|----------|------------------------------------------------------|-------------------|----------------------------------------------|
| Tomcat   | Un thread pendant toute la requête                   | 200               | `server.tomcat.threads.max`                  |
| HikariCP | Une connexion JDBC pendant la requête SQL uniquement | 10                | `spring.datasource.hikari.maximum-pool-size` |

### Combien de requêtes en vol ?

On a le débit et la latence, on en déduit la concurrence : `L = λ × W`.

- débit : 2 000 req/s
- latence : 40 ms, soit 0,04 s

`L = 2 000 × 0,04 = 80`.

80 requêtes vivent en permanence dans le système. Avec 200 threads Tomcat, on a de la marge.

### Et le pool Hikari ?

80 requêtes en vol, mais seulement 10 connexions Hikari. Comment ça tient ?

Parce que les 40 ms représentent le temps total de bout en bout. Ce temps se répartit entre la requête SQL, le code métier Java, la sérialisation JSON, etc. Une connexion Hikari n'est mobilisée que pendant la requête SQL — 5 ms, pas 40 ms.

Notons **x** ce temps passé en base, et appliquons `L = λ × W` au pool Hikari seul :

```
L = 2 000 × 0,005 = 10
```

À 2 000 req/s avec des requêtes SQL de 5 ms, il faut 10 connexions simultanées. Ou, vu autrement, chaque connexion absorbe 200 req/s.

Notre pool en a justement 10 : il tourne à **100 %**.

## La latence passe de 40 ms à 100 ms

Un jour, sans variation de trafic, la latence grimpe à 100 ms. La loi de Little donne :

```
L = 2 000 × 0,1 = 200
```

200 requêtes en permanence dans le système. Le trafic n'a pas changé, mais la charge réelle a été multipliée par 2,5. Or Tomcat a exactement 200 threads : il est à 100 %, sans marge.

Reste à comprendre d'où sortent ces 60 ms supplémentaires. Tout dépend de x, le temps de la requête SQL.

### Cas 1 : x reste à 5 ms — le problème est dans l'application

Côté Hikari :

```
L = 2 000 × 0,005 = 10
```

Toujours 10 connexions nécessaires, et on en a 10. La base répond aussi vite qu'avant, Hikari n'y est pour rien.

Les 60 ms en plus sont consommées ailleurs dans l'application : code métier, sérialisation JSON, pause du Garbage Collector, appel HTTP vers un service tiers… C'est par là qu'il faut creuser.

Le système tient encore, mais tout juste. Tomcat est à 100 %, Hikari aussi. La moindre milliseconde de plus et les requêtes se mettront à faire la queue pour obtenir un thread.

### Cas 2 : x passe de 5 ms à 50 ms — la base de données sature

Côté Hikari :

```
L = 2 000 × 0,05 = 100
```

100 requêtes réclament une connexion en même temps, mais il n'y en a que 10. Les 90 autres patientent.

On peut regarder la situation sous l'angle du débit. On connaît la taille du pool et la latence, on cherche le débit maximal : `λ = L / W`.

```
λ = 10 / 0,05 = 200 req/s
```

Avec des requêtes SQL de 50 ms, le pool ne laisse passer que 200 req/s. Or il en arrive 2 000.

Et pendant qu'une requête attend sa connexion Hikari, elle garde son thread Tomcat. Ce temps d'attente gonfle la latence totale, ce qui augmente la concurrence côté Tomcat. Tomcat finit par saturer lui aussi, mais c'est un symptôme : la cause première, c'est la base de données.

Contrairement au cas 1, la latence ne va pas se stabiliser à 100 ms. Ces 100 ms ne sont qu'un instantané : il arrive dix fois plus de requêtes que le pool n'en écoule, la file d'attente s'allonge en continu et la latence monte jusqu'à l'effondrement.

### Mesurer x en production

Inutile de deviner : Spring Boot Actuator et Micrometer exposent tout ce qu'il faut.

| Métrique                       | Ce qu'elle mesure                                      | Cas 1 | Cas 2                    |
|--------------------------------|--------------------------------------------------------|-------|--------------------------|
| `hikaricp.connections.usage`   | Le temps pendant lequel une connexion est utilisée (x) | 5 ms  | 50 ms                    |
| `hikaricp.connections.pending` | Le nombre de requêtes en attente d'une connexion       | 0     | des dizaines, puis plus  |
| `tomcat.threads.busy`          | Le nombre de threads Tomcat occupés                    | 200   | 200                      |

Dans les deux cas, Tomcat affiche 200 threads occupés. Ce sont `hikaricp.connections.usage` et `hikaricp.connections.pending` qui font la différence : ils orientent le diagnostic vers le code applicatif ou vers la base de données.

## À quel débit le système casse-t-il ?

Reprenons la question initiale.

Commençons par Tomcat. On connaît la taille du pool (200 threads) et le débit (2 000 req/s), on en tire la latence maximale admissible : `W = L / λ`.

```
W = 200 / 2 000 = 0,1 s
```

Tant qu'une requête met moins de 100 ms, Tomcat encaisse. C'est la limite exacte du cas 1.

Mais une requête passe par plusieurs pools, chacun avec sa propre limite. Même calcul pour Hikari :

```
W = 10 / 2 000 = 0,005 s
```

À 2 000 req/s, une requête SQL ne peut pas dépasser 5 ms. C'est pile la durée actuelle de nos requêtes : le pool Hikari fonctionne sans aucune marge. La moindre augmentation de la latence SQL fait tomber le système, bien avant que Tomcat n'atteigne sa propre limite de 100 ms.

### La loi du goulot d'étranglement

**Le débit d'un système est borné par sa ressource la plus contrainte.**

C'est la *bottleneck law* :

```
λ_max = min(capacité de chaque ressource)
```

Jusqu'ici, on est parti du débit (2 000 req/s) pour identifier ce qui allait lâcher en premier. On peut aussi raisonner dans l'autre sens : connaissant le temps passé dans chaque pool, on estime le débit maximal du système.

Pour chaque pool, `λ = L / W` donne :

| Pool     | Calcul        | Capacité max    |
|----------|---------------|-----------------|
| Tomcat   | 200 / 0,04 s  | 5 000 req/s     |
| HikariCP | 10 / 0,005 s  | **2 000 req/s** |

```
λ_max = min(5 000, 2 000) = 2 000 req/s
```

Le système plafonne à 2 000 req/s — exactement le trafic actuel. Zéro marge, confirmé par un autre angle.

Si demain le trafic passe à 3 000 req/s, Hikari ne suivra pas. Il faudrait au minimum :

```
L = 3 000 × 0,005 = 15 connexions
```

## Conclusion

On a utilisé la loi de Little dans ses trois lectures :
- débit + latence → concurrence, donc taille de pool nécessaire : `L = λ × W`
- taille du pool + débit → latence maximale : `W = L / λ`
- taille du pool + latence → débit maximal : `λ = L / W`

Et puisqu'une requête traverse plusieurs pools, il faut appliquer ce calcul à chacun : c'est toujours le plus contraint qui dicte la limite du système.
