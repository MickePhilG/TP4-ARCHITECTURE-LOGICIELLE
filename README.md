# Architecture Logicielle – Séance 4 · Patterns créationnels et structuraux

M1 Data Engineer · INGETIS

TP 4 réalisé par **Oliver GBAGUIDI**, **Daniel NKOT** et **Namory SOUMAHORO** (parcours Data).

## Contenu

| Dossier | Contenu |
|---|---|
| `creationnels/` | Exemples du cours : Factory Method, Abstract Factory, Builder, Prototype, Singleton |
| `structuraux/` | Exemples du cours : Adapter, Bridge, Composite, Decorator, Facade, Proxy |
| `tp-data-cyber/` | Notre TP 4 : reconnaître, étendre et appliquer des patterns à QueenBee (kit fourni, complété sur place comme le demande la fiche) |

Dans le dossier du TP :

- `extension-canal.js` : partie B1, ajout du canal `sms-groupe` (un SMS envoyé à toute une coopérative) avec la classe `SmsGroupeNotification`, enregistrée par `registerChannel` sans modifier `factory.js` (Factory Method + registre, principe ouvert/fermé) ;
- `ordre-decorateurs.js` : partie B2, ajout d'une composition C · `Retry( Log( Cache( base ) ) )` pour comparer l'effet de l'ordre des décorateurs ;
- `sources-mesures.js` : partie C Data, deux adaptateurs vers le contrat `Mesure` du contexte « Santé des colonies » :
    - `LoRaAdapter` : renommage des champs des trames capteurs et date au format ISO 8601 ;
    - `ReleveManuelAdapter` : découpage des lignes CSV des relevés papier, conversion de la date `JJ/MM/AAAA HH:MM` en ISO 8601 UTC, des grammes en kilogrammes et des °F en °C, lignes incomplètes écartées ;
- `PATTERNS.md` :
    - partie A : tableau des 6 extraits QueenBee (Factory Method, Builder, Adapter, Composite, Decorator, Proxy) et différence d'intention entre les extraits 5 (Decorator) et 6 (Proxy) ;
    - partie B2 : sortie de la composition C et réponses aux 3 questions (choix de l'ordre `Cache( Retry( Log( base ) ) )` pour la passerelle QueenBee) ;
    - partie C : propriétaire du contrat `Mesure`, traitement d'une ligne incomplète (quarantaine), évolution du contrat pour un capteur d'humidité ;
    - partie D : tableau des choix effectués (Factory Method, Decorator, Adapter), avec le problème résolu et l'alternative écartée.

Les fichiers fournis non modifiables (`creationnels/factory.js`, les sources de mesures) n'ont pas été modifiés.

## Lancer les tests

Prérequis : Node.js 20 ou plus récent, aucune dépendance à installer. Commande à lancer depuis ce dossier.

Les tests du parcours Data (B1 et C-Data) :

```bash
node --test tp-data-cyber/extension-canal.spec.js tp-data-cyber/sources-mesures.spec.js
```

Résultat attendu : `pass 6`, `fail 0`.

Avec `npm run tp4` (depuis la racine du projet), les 5 tests « C-Cyber » échouent : ils concernent le parcours Cyber et sont ignorés par un binôme Data.

## Voir l'effet de l'ordre des décorateurs

```bash
node tp-data-cyber/ordre-decorateurs.js
```
