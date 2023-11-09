---
marp: true
---

Jérémy Lal kapouer@melix.org

# ♼ HTTP et cache

- À quoi servent les en-têtes ?

- Contrôler le cache.

- Cache et proxy: limites et extensions

---

## À quoi servent les en-têtes ?

### HTTP: requête

- un protocole (HTTP 1.0, 1.1, 2, 3)
- une méthode (GET, POST, PUT, DELETE, ...)
- un Host
- une URI
- des en-têtes
- un corps (body) optionel

---

### HTTP: réponse

- le protocole
- Status: 200, 301, 304, 404, 500...
- des en-têtes
- un corps (body) optionel

---

### Rôles

- décrire ce qui est envoyé ou reçu
- négocier le contenu
- contrôler le cache
- configurer le client

---

### Description du contenu

Description du body envoyé ou reçu

- Content-Type: son type mime (obligatoire)
- Content-Length: si présent, taille en octets
- Content-Encoding: si présent, type de compression (gzip, deflate, br)

---

### Négocier le contenu

Le client annonce ce qu'il accepte dans les headers de requête.

Le serveur choisit ce qu'il veut répondre et précise quoi dans les headers de réponse.

- Accept > Content-Type: les types mimes (text/html,application/xml;q=0.9)
- Accept-Encoding > Content-Encoding: les compressions (gzip, deflate, br)
- Accept-Language > Content-Language: les langues (fr-FR,fr;q=0.9,es-ES;q=0.8)

---

### Négocier implique Vary

Le serveur répond en précisant quels en-têtes de requête font varier les réponses:

- Vary: Accept, Accept-Language

Très important pour la gestion du cache.

---

### Exemples

Cas typique

```http
> HTTP/1.1 GET /pageweb
> Accept: text/html,application/xml;q=0.9
> Accept-Encoding: gzip, deflate, br
> Accept-Language: fr-FR,fr;q=0.9,es-ES;q=0.8,es;q=0.7,la;q=0.6,en-US;q=0.5,en;q=0.4,ja;q=0.3

< Status 200 OK
< Content-Encoding: gzip
< Content-Type: text/html; charset=utf-8
< Vary: Accept-Encoding, Accept-Language
```

---

On négocie rarement par User-Agent, mais il y a polyfill.io

```http
> HTTP GET /v3/polyfill.js
> Host: https://polyfill.io
> Accept-Encoding: gzip, deflate, br
> User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36

< Content-Type: text/javascript
< Vary: User-Agent, Accept-Encoding
```

---

## Contrôler les caches

- du navigateur
- du serveur
- et de tous les proxies qui peuvent se trouver entre les deux

Même principe que la négociation de contenu.

---

### Last-Modified et If-Modified-Since

Le serveur répond la date de dernière modification du contenu

```http
> GET /myurl

< Status: 200 OK
< Last-Modified: Thu, 09 Nov 2023 07:51:39 GMT
```

Le client peut conserver la réponse et demander si elle est encore fraîche.

```http
> GET /myurl
> If-Modified-Since: Thu, 09 Nov 2023 07:51:39 GMT

< Status: 304 Not Mofified
< PAS DE BODY
```

La réponse est 200 avec le contenu s'il a changé.

---

### ETag et If-None-Match

Le serveur donne un hash ou un numéro de version du contenu

```http
> GET /myurl

< Status: 200 OK
< ETag: "V1.10.1"
```

Le client:

```http
> GET /myurl
> If-None-Match: "V1.10.1"

< Status: 304 Not Mofified
< PAS DE BODY
```

---

### Date et Expires

Un peu moins pratique, le serveur précise quand la réponse devient périmée

```http
> GET /myurl

< Status: 200 OK
< Date: Thu, 09 Nov 2023 07:51:39 GMT
< Expires: Thu, 09 Nov 2023 10:51:39 GMT
```

Le client peut conserver la requête dans son cache avant expiration.

---

### Age, Cache-Control:max-age

Si un proxy a conservé la requête, ce dernier peut indiquer combien de temps

```http
> GET /myurl

< Status: 200 OK
< Date: Thu, 09 Nov 2023 07:51:39 GMT
< Age: 3600
< Cache-Control: max-age=7200
```

Un cache intermédiaire a conservé la réponse pendant une heure, le client sait qu'elle sera périmée dans une heure.

Age est rarement utilisé par un serveur applicatif.

---

### Options de contrôle

- max-age=0: autorise la mise en cache mais immédiatement périmé
- must-revalidate: client doit vérifier que son contenu est frais (If-Modified-Since ou If-None-Match les cas échéant)
- no-cache: raccourci pour `max-age=0, must-revalidate` typiquement utilisé pour une page HTML dynamique, en conjonction avec `Last-Modified` ou `Etag`.
- no-store: pas de cache du tout
- private: cache non partagé - en pratique seul le navigateur garde en cache, pas les proxies
- stale-while-revalidate=86400: le serveur indique que le client ou un proxy peut utiliser un contenu périmé pendant un certain temps, en attendant d'avoir revalidé le contenu.

---

### Ressources immuables

Une bonne pratique consiste à donner une URL unique aux ressources immuables:

- javascript
- feuilles de style
- images

et laisser les caches les conserver longtemps sans revalidation:
`Cache-Control: max-age=31536000, immutable`.

---

## Proxy et Cache

HTTP permet d'indiquer la mise en cache, la durée, et la vérification par le client ou les proxies.

"Stateless": sans état.

Situations:

- démultiplication des URLS sans surcharger le cache
- contenus dynamiques sans surcharger le serveur applicatif
- clients authentifiés sans surcharger le cache

---

### Solutions naïves

Pour répondre à ces problèmes, un proxy cache antique comme Varnish faisait:

- récriture interne des URL: `/cat?param=1&extra=2` est déclaré comme devant être récrit `/cat?param=1`
- invalidation d'URL par préfixes: l'application utilise l'api http interne de varnish pour explicitement invalider des URL (une par une, ou par préfixes).
- variation sur un cookie précis: varier sur tous les Cookies serait contre-productif, on précise que le cache doit varier sur un nom de cookie

---

### Problèmes

- éparpillement de la configuration entre l'application et le proxy
- lourd: les appels application > varnish sont très spécifiques
- inefficace, peu de partage de cache pour les clients authentifiés

---

### Clés de cache

Une meilleure approche est d'étendre les mécanismes de la négociation HTTP aux besoins modernes.

Le protocole reste stateless, mais le proxy conserve des états sur les URL en cache.

---

### Étiquettes et purge

En-tête envoyée dans la réponse par l'applicatif.

Le proxy associe l'URL à la clé de cache trouvée dans `Cache-Key: groupKey`.

Le serveur applicatif peut invalider des groupes d'URL en faisant des HTTP PURGE vers une des URL ou vers une URL interne au proxy.

- wordpress/varnish
- polyfill.io/akamai
- cloudflare

Plus efficace, mais peu efficace (requête PURGE additionelle à chaque changement de contenu).

---

### Étiquettes et REST

- GET, HEAD: pas de modification des ressources (lecture)
- POST, PUT, DELETE: modification (écriture)

Ceci implique

- lecture: l'application ne voit pas toujours passer ces requêtes reçues par le proxy
- écriture: l'application reçoit toutes les requêtes

---

Le proxy va associer les deux étiquettes "app", "groupA" à l'URL /a

```http
> GET /a

< Cache-Control: max-age=600000, must-revalidate
< X-Cache-Tag: app, groupA
```

Le client redemande la ressource, le proxy lui renvoie sans passer par l'applicatif.

Le client fait un POST vers une autre ressource:

```http
> POST /b

< X-Cache-Tag: app, +groupA
```

L'applicatif répond un header indiquant qu'il faut invalider l'étiquette "groupA".

Le proxy invalide toutes les URL portant cette étiquette.

---

### Mapping de clés

L'applicatif indique au cache qu'il peut associer ces deux URL à une seule ressource.

```http
> GET /inexistent

< X-Cache-Map: /notfound
```

```http
> GET /anotherinexistent

< X-Cache-Map: /notfound
```

---

### Clés par permissions et authentification décentralisée

Json Web Token, Biscuit sont des jetons d'authentification sécurisés (avec rsa asymmétrique par exemple), le proxy peut donc savoir quelles permissions sont possédées par un client.

L'applicatif répond qu'une URL varie sur des permissions:

```http
> GET /secure

< X-Cache-Lock: contributor
```

Le proxy va construire la clé de cache de cette URL en y ajoutant cette permission.

Un autre client demande la même URL avec une autre permission: MISS
Un autre client demande la même URL avec la même permission: HIT.

C'est une version perfectionnée de Vary:Cookie.

---

## Sources

[Documentation pour les headers HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)

[Jetons décentralisés avec JWT](https://jwt.io/)
[Jetons décentralisés avec Biscuit](https://www.biscuitsec.org/)

---

## Bonus

[Implémentation d'un proxy avancé nginx/lua et module nodejs](https://github.com/kapouer/upcache)
