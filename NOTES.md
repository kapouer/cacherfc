# RFC process

<https://authors.ietf.org/>

# Reference implementation

https://github.com/kapouer/upcache

# Useful Links

https://www.rfc-editor.org/rfc/rfc9213.html

# Quick ideas about HTTP caches

## Application, Proxy, Client

## Response headers

- Cache-Control: qui cache quoi et combien de temps
- ETag: hash pour négocier la validité de la ressource grâce à If-Modified-Since

## Request headers

- If-Modified-Since: envoie le Etag, le serveur répond soit 200 avec le contenu si ça ne correpsond pas, soit 304 Not Modified si ça correspond -> évite de transférer des données inutilement.
Ce mécanisme là s'applique bien aux pages html modifiées régulièrement, ou aux points d'API json.

- En revanche le Cache-COntrol s'appliquent bien aux ressources "immuables" comme les images, les bundles css/js, les polices de caractères, etc..

## html and immutable resources

Le html change, les liens vers les ressources statiques sont immuables:
si on veut changer la ressource, on change le lien, et pas la ressource.


