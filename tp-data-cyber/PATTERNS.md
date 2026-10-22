## Partie A

| N° | Pattern | Problème résolu | Indice dans le code |
|---|---|---|---|
| 1 | Factory Method (avec registre) | Créer le bon type de capteur (poids, température, acoustique) à partir de la configuration de la ruche, sans que l'appelant connaisse les classes concrètes | Dictionnaire `types` nom → classe, une fonction `creerCapteur(type)` qui fait le `new`, erreur « Capteur inconnu » sinon, pas de `switch` |
| 2 | Builder | Construire un rapport d'impact qui a beaucoup de paramètres (période, sites, indicateurs, format), dont certains répétables, sans un constructeur à 6 arguments | Interface fluide (chaque méthode renvoie l'objet), `indicateur()` appelé plusieurs fois, validation une seule fois dans `construire()` |
| 3 | Adapter | Le modèle de prédiction attend `temperatureExterne(site)` en °C, alors que OpenWeather a sa propre API (lat/lon, `r.main.temp`, date en secondes Unix) | La classe enveloppe un client externe et traduit : construction de l'URL, `r.main.temp` → `celsius`, `r.dt * 1000` → `Date` |
| 4 | Composite | Calculer le poids total d'un pays, d'un site ou d'une ruche avec le même appel, quelle que soit la profondeur de l'arbre (pays > site > ruches) | `Ruche` (feuille) et `Groupe` (nœud) ont la même méthode `poidsTotal()` ; le groupe contient des enfants et additionne récursivement leur `poidsTotal()` |
| 5 | Decorator | Chiffrer les coordonnées GPS des ruches avant stockage (donnée sensible) sans modifier le dépôt Postgres | Même interface que le dépôt (`enregistrer`, `lire`), reçoit un autre dépôt dans le constructeur, ajoute un traitement avant/après puis délègue |
| 6 | Proxy (virtuel) | Ne pas charger le modèle de prédiction de 300 Mo au démarrage : il n'est chargé qu'au premier appel de `predire()` | Même interface que le modèle (`predire`), champ privé `#modele` à `null`, `??=` qui crée l'objet réel seulement au premier usage |

**Les deux extraits à la structure presque identique sont les extraits 5 (Decorator) et 6 (Proxy).**
Dans les deux cas, une classe a la même interface que l'objet réel et lui délègue l'appel. C'est l'intention qui change :
- Extrait 5 : le Decorator **ajoute un comportement** (le chiffrement) à un objet qu'on lui donne déjà construit. On pourrait empiler d'autres décorateurs (cache, journal).
- Extrait 6 : le Proxy **contrôle l'accès** à l'objet réel : il décide quand le créer (chargement différé). Il ne change pas ce que fait `predire()`. D'ailleurs, il crée lui-même l'objet réel au lieu de le recevoir.



## Partie B2

Nous avons ajouté la composition C · `Retry( Log( Cache( base ) ) )` dans `ordre-decorateurs.js` :
```
C · Retry( Log( Cache( base ) ) )
  [log] GET /api/ruches – 51 ms
  [cache] HIT /api/ruches
  [log] GET /api/ruches – 0 ms
  [log] GET /api/instable – 51 ms
  [retry] tentative 1 échouée (503 Service Unavailable), on réessaie
  [log] GET /api/instable – 51 ms
  → appels réels sur le client de base : 3
```
Ici, le journal voit tout : les réponses du cache (0 ms) et chaque tentative, y compris celle qui échoue. Mais on ne peut plus distinguer, dans le journal, un vrai appel d'une réponse du cache sans regarder la durée.

1. Dans A, le cache est la couche la plus à l'extérieur. Au 2e appel à `/api/ruches`, il trouve la réponse (HIT) et la renvoie directement : il n'appelle pas l'objet qu'il enveloppe, donc ni Retry ni Log ne sont exécutés. Dans B, Log est à l'extérieur du cache : chaque appel passe d'abord par Log, qui journalise donc aussi la réponse du cache.

2. Non on ne voit pas dans les journaux que /api/instable a échoué une première fois. Dans B, Log enveloppe le cache, qui enveloppe Retry : Log ne voit qu'un seul appel, celui qui finit par réussir. L'échec de la 1re tentative est absorbé par Retry, plus bas dans la pile, et Log ne le voit jamais.

3. Nous choisirions l'ordre A · `Cache( Retry( Log( base ) ) )` :
    - **Cache à l'extérieur** : une donnée déjà reçue est servie sans rien émettre. Sur une passerelle où le temps d'émission radio est limité, c'est la meilleure économie : un HIT ne coûte ni émission, ni tentative.
    - **Retry au milieu** : la liaison est instable, donc il faut réessayer. Comme le cache est au-dessus, seule une réponse réussie est mise en cache : on ne garde jamais une erreur 503 en cache.
    - **Log au plus près du client de base** : chaque vraie émission est journalisée, y compris les échecs. On peut donc mesurer la qualité de la liaison (taux d'échec) et le temps radio réellement consommé.



## Partie C 

| Question | Réponse |
|---|---|
| **Quel contexte est propriétaire du contrat `Mesure` ?** | C'est le contexte « Santé des colonies », c'est-à-dire le consommateur. |
| **Que faire d'une ligne incomplète en production : la rejeter, la mettre en quarantaine, la corriger ?** | Selon nous, il faut la **mettre en quarantaine** comme ça on stocke la ligne brute avec le motif du rejet et la date d'import. |
| **Comment faire évoluer le contrat si un capteur d'humidité s'ajoute ?** | Il faut : - Ajouter un champ **facultatif** `humiditePct` au contrat (`null` quand la source ne le mesure pas). - Mettre à jour seulement les adaptateurs. - Versionner le contrat (ex. `Mesure` v1.1). Si un jour le changement est incompatible on pourra créer une v2 et garder la v1 le temps que tous les consommateurs migrent. |



## Partie D

## Tableau explicatifs des choix effectués

| Pattern | Problème résolu (dans QueenBee) | Alternative écartée | Pourquoi écartée |
|---|---|---|---|
| Factory Method + registre (B1) | BeeAfrica veut un SMS envoyé à toute une coopérative : on ajoute le canal `sms-groupe` sans toucher à `factory.js` | Ajouter un `case 'sms-groupe'` (ou un `if`) dans `createNotification` | Il faudrait rouvrir et retester un fichier qui marche, à chaque nouveau canal (viole OCP). Avec le registre, un canal = une classe + une ligne `registerChannel` |
| Decorator (B2) | Ajouter cache, nouvelle tentative et journalisation aux appels de la passerelle, sans modifier le client HTTP | Une sous-classe par combinaison (`ClientAvecCacheEtRetry`, `ClientAvecLogEtCache`...) | Explosion du nombre de classes (3 comportements = 7 combinaisons), et l'ordre serait figé dans le code. Les décorateurs s'empilent à la carte |
| Adapter (C-Data) | Deux sources de mesures non modifiables (trames LoRa, CSV papier) aux formats différents, alors que « Santé des colonies » attend un seul format `Mesure` | Faire les conversions dans le client `toutesLesMesures` avec des `if (source === 'lora') ... else ...` | Le client connaîtrait tous les formats : chaque nouvelle source le modifierait (viole OCP) et les conversions seraient mélangées avec la fusion des données |