# Spécification technique d'implémentation — Nexus Repository

Version 1.0 — 2026-09-30
Référence : dossier d'architecture v5 (décisions D1 à D7, option A)
Périmètre : configuration de la plateforme, contrats d'intégration, critères de recette. Hors périmètre : trajectoire de migration des applications, tableau de bord DORA.

## 1. Objectif et principes

Déployer Nexus Repository Pro comme catalogue unique des artefacts publiés, avec immutabilité, promotion candidat vers release, contrôle des composants tiers et alimentation de la CMDB sans saisie manuelle.

| Principe | Traduction technique |
|---|---|
| Immutabilité (D1) | Redéploiement interdit sur tous les dépôts release |
| Hash en métadonnée (D1, D7) | Manifeste de release + tag, jamais dans le chemin |
| Nom d'origine conservé (D2) | Aucune réécriture de nom à l'ingestion |
| Un dépôt par format et cycle de vie (D3) | Matrice de dépôts, section 3 |
| Identifiant d'application en tête de chemin (D3) | Contrôle de chemin, sélecteurs de contenu, section 4 |
| Un flux par nature de donnée (D5) | Publié : Nexus. Déployé : événement du pipeline |
| Contrôle automatique des tiers (D6) | Proxy + pare-feu de dépôt + décision SSI sous 1 jour ouvré |

## 2. Composants à déployer

| Composant | Rôle | Remarques |
|---|---|---|
| Nexus Repository Pro | Stockage, proxy, promotion, droits | Haute disponibilité selon l'offre Pro ; base externe et stockage de blobs partagé |
| Repository Firewall (Sonatype) | Quarantaine des composants tiers non conformes | Couverture du format raw à confirmer auprès de l'éditeur |
| Outil d'analyse Sonatype | Analyse, SBOM reconstitué pour les COTS, licences | Sert aussi à agréger les SBOM par application |
| Annuaire d'entreprise | Authentification et groupes | Groupes mappés sur les rôles Nexus |
| Chaîne de build (CI) | Publie en candidat, écrit le manifeste, produit le SBOM | Compte de service par application |
| Pipeline de déploiement | Déploie les releases, émet l'événement de déploiement | Seule source du « déployé » |
| CMDB / ITSM | Reflète publié et déployé, rattache les incidents | Lit Nexus, reçoit les événements |
| Répartiteur de charge / proxy inverse | Point d'accès unique, terminaison TLS, routage Docker | Voir 3.3 |

## 3. Dépôts

### 3.1 Matrice

| Format | Candidat | Release | Proxy | Groupe (lecture) |
|---|---|---|---|---|
| Maven | `maven-candidat` | `maven-release` | `maven-proxy-central` | `maven-all` |
| npm | `npm-candidat` | `npm-release` | `npm-proxy-npmjs` | `npm-all` |
| Docker | `docker-candidat` | `docker-release` | `docker-proxy-public` | `docker-all` |
| Raw | `raw-candidat` | `raw-release` | néant (les COTS sont déposés, pas proxifiés) | néant |

Règles :
- Les dépôts candidat et release sont de type hébergé. Les proxys ne sont jamais publiés vers.
- Les groupes exposent à la lecture l'ensemble release + proxy ; le candidat n'est jamais dans un groupe consommé par les déploiements.
- Le proxy Docker alimente uniquement la construction des images dorées (D6) ; aucune application ne tire directement depuis lui.

### 3.2 Paramétrage

| Paramètre | Candidat | Release | Proxy |
|---|---|---|---|
| Politique de redéploiement | Interdit | Interdit | Sans objet |
| Politique de version Maven | Release | Release | Release |
| Validation stricte du type de contenu | Activée | Activée | Activée |
| Blob store | Un par format et cycle de vie, avec quota souple | Idem | Idem |
| Nettoyage | Oui (3.4) | Non par défaut ; exception validée par la gouvernance | Oui (3.4) |
| Accès anonyme | Désactivé | Désactivé | Désactivé |

Le nommage évite tout suffixe de version dans le nom du dépôt. Une release n'est jamais supprimée par un nettoyage automatique.

### 3.3 Docker : point d'attention

Un dépôt Docker hébergé nécessite un point d'accès distinct (port dédié ou routage par sous-domaine, ou par chemin selon la version et le proxy inverse). Décision à figer en phase de conception détaillée : un sous-domaine par dépôt Docker, terminé sur le répartiteur de charge. Les noms d'images suivent la règle de chemin de la section 4.

### 3.4 Politiques de nettoyage

| Cible | Critère | Durée à fixer | Note |
|---|---|---|---|
| Candidats | Non promus et plus anciens que N jours | à fixer avec l'exploitation | Un candidat promu n'est pas supprimé tant que sa release existe |
| Caches proxy | Non téléchargés depuis N jours | à fixer | Évite la croissance libre du stockage |
| Images Docker candidates | Étiquettes anciennes sans référence de release | à fixer | Suivi du digest promu |
| Compactage des blobs | Tâche planifiée après nettoyage | Hebdomadaire | Libère réellement l'espace |

## 4. Convention de chemin et identifiants

### 4.1 Identifiant d'application

Format : `service.projet`, en minuscules, stable et unique (par exemple `monex.diagb`, `monex.jira`). Il est enregistré dans la CMDB, qui fait autorité. Aucune application ne publie sans identifiant enregistré.

### 4.2 Chemin par format

| Format | Emplacement de l'identifiant | Exemple de chemin résultant |
|---|---|---|
| Raw | Premier niveau du chemin | `monex.diagb/diagb/1.2/<paquet éditeur>` |
| Maven | Préfixe du groupId | groupId `monex.diagb.*`, soit chemin `monex/diagb/...` |
| npm | Portée (scope) dérivée de l'identifiant | portée `@monex-diagb` (points remplacés par un tiret, à confirmer) |
| Docker | Premier segment du nom d'image | `monex.diagb/<image>:<version>` |
| Mutualisé | Préfixe fixe `mutualise` | `mutualise/image-doree-temurin-21/<version>/` |

Règles raw :
- Chemin : `<identifiant>/<composant>/<version>/<fichier>`.
- Candidat : `<version>-<numéro de build>` pour les images dorées ; version éditeur inchangée pour les COTS (D2).
- Release : la version seule.

### 4.3 Étiquettes

| Étiquette | Contenu | Portée |
|---|---|---|
| `domaine` | Domaine métier | Recherche et tableaux de suivi |
| `equipe` | Équipe responsable | Recherche |
| `proprietaire` | Propriétaire de l'application | Recherche |
| `build` | Numéro de build de promotion | Lien candidat / release |

Le domaine, l'équipe et le propriétaire ne figurent jamais dans le chemin : ils changent, l'identifiant non.

## 5. Droits et sécurité

### 5.1 Modèle

- Authentification via l'annuaire d'entreprise (SSO). Pas de compte local hors compte d'administration de secours.
- Un sélecteur de contenu par application, sur le préfixe de son identifiant, appliqué aux dépôts hébergés du format concerné.
- Deux rôles par application : `lecteur` (lecture des releases) et `editeur` (publication en candidat). La promotion vers la release est réservée au compte de la chaîne de promotion.
- Rôle `mutualise-editeur` réservé à l'équipe qui construit les images dorées.
- Rôle `ssi-controle` : lecture transverse, gestion des quarantaines et des exceptions.
- Rôle `exploitation` : administration technique, sans droit de publication.

### 5.2 Comptes de service

- Un compte de service par application pour la chaîne de build, avec jeton d'accès ; secret stocké dans le coffre d'entreprise, rotation périodique.
- Aucun compte nominatif dans les scripts de build.
- Le pipeline de déploiement dispose d'un compte en lecture seule sur les dépôts release.

### 5.3 Routage et blocage

- Règles de routage sur les proxys pour bloquer les chemins interdits (par exemple composants hors périmètre).
- TLS obligatoire de bout en bout ; certificat géré par le répartiteur de charge.

## 6. Cycle de vie d'un artefact

### 6.1 Composant produit en interne

1. **Build** : la CI construit et publie en candidat, avec un numéro de build unique.
2. **Manifeste** : la CI publie à côté de l'artefact un manifeste de release (6.4) et le SBOM CycloneDX.
3. **Contrôles de promotion** : voir 6.3.
4. **Promotion** : l'artefact est déplacé vers le dépôt release par la chaîne de promotion ; l'étiquette `build` est posée.
5. **Immutabilité** : la release ne peut plus être remplacée. Un nouveau build du même commit donne un nouveau candidat.
6. **Publication CMDB** : la CMDB lit la release et son manifeste.

### 6.2 Composant éditeur (COTS)

1. Dépôt en raw candidat, nom d'origine conservé, avec l'empreinte fournie par l'éditeur.
2. Comparaison de l'empreinte calculée par Nexus avec celle de l'éditeur.
3. Analyse du contenu pour reconstituer le SBOM CycloneDX ; propriété `origine = reconstitué par analyse`.
4. Contrôle des licences et vulnérabilités. Cas bloqué : la SSI tranche seule, sous 1 jour ouvré.
5. Promotion en raw release. Le délai de mise en production se mesure depuis la date de réception dans le candidat.

### 6.3 Contrôles de promotion

| Contrôle | Composant interne | COTS | Image dorée |
|---|---|---|---|
| Empreinte de contenu conforme | Oui | Oui (comparée à l'éditeur) | Oui |
| Manifeste présent et complet | Oui | Oui (sans commit) | Oui |
| SBOM présent | Oui (généré) | Oui (reconstitué, marqué) | Oui |
| Commit présent sur branche ou étiquette protégée | Oui | Sans objet | Oui |
| Analyse licences et vulnérabilités sans blocage | Oui | Oui | Oui |
| Décision SSI sur cas bloqué | Sur exception | Sur exception | Sur exception |

Un échec bloque la promotion ; l'artefact reste en candidat et est soumis au nettoyage.

### 6.4 Manifeste de release

Fichier déposé à côté de l'artefact, nommé `<composant>-<version>.manifest.json`. Il est la source de vérité des métadonnées que Nexus ne porte pas nativement sur les assets raw.

| Champ | Description | Interne | COTS |
|---|---|---|---|
| `application` | Identifiant d'application | Oui | Oui |
| `composant` | Nom du composant | Oui | Oui |
| `version` | Version publiée | Oui | Oui |
| `build` | Numéro de build | Oui | Néant |
| `commit` | Hash complet | Oui | Néant |
| `dateCommit` | Date du commit | Oui | Néant |
| `dateReception` | Date de réception dans le candidat | Néant | Oui |
| `empreinteEditeur` | Empreinte fournie par l'éditeur | Néant | Oui |
| `origineSbom` | `genere` ou `reconstitue` | Oui | Oui |
| `imagesDorees` | Versions des images dorées consommées (si connues à la construction) | Facultatif | Facultatif |

Pour les formats porteurs (Maven, npm, Docker), le hash est aussi écrit dans leur champ natif (manifeste Maven, descripteur npm, étiquette standard de révision de l'image Docker).

### 6.5 SBOM

- Format : CycloneDX, JSON. Nom : `<composant>-<version>.cdx.json`.
- Contenu minimal : composants et versions, licences, référence au dépôt source et à la révision (commit) quand il existe.
- Rangé à côté de l'artefact, dans les mêmes dépôts candidat puis release.
- Transmis à l'outil d'analyse pour l'agrégation par application.

## 7. Images dorées

| Image | Format Nexus | Chemin |
|---|---|---|
| Temurin 21 | Raw | `mutualise/image-doree-temurin-21/<version>/` |
| MongoDB 7.0 | Raw | `mutualise/image-doree-mongodb-7.0/<version>/` |
| MariaDB 11.x | Raw | `mutualise/image-doree-mariadb-11/<version>/` |
| PostgreSQL 17 | Docker (conteneur) | `mutualise/image-doree-postgresql-17:<version>` |

Règles :
- Construites en interne à partir de sources contrôlées ; l'entrée Docker publique passe uniquement par `docker-proxy-public`.
- Chaque image porte son commit de construction, son SBOM et son manifeste.
- Une image publiée en release est immuable ; toute correction produit une nouvelle version.
- La CMDB tient la relation application vers version d'image, alimentée par l'événement de déploiement. Elle répond à la question « quelles applications une mise à jour concerne-t-elle ».

## 8. Intégration CI/CD et CMDB

### 8.1 Contrat de la chaîne de build

| Étape | Attendu |
|---|---|
| Authentification | Compte de service de l'application, jeton du coffre |
| Publication | Candidat uniquement, chemin conforme à la section 4 |
| Métadonnées | Manifeste + SBOM déposés avec l'artefact |
| Échec | Aucune publication partielle : l'ensemble artefact + manifeste + SBOM est cohérent ou absent |

### 8.2 Événement de déploiement

Émis par le pipeline à chaque déploiement, réussi ou non, vers la CMDB (D5).

| Champ | Description |
|---|---|
| `application` | Identifiant d'application |
| `environnement` | Environnement cible |
| `release` | Référence de la release Nexus (chemin, version, empreinte) |
| `imagesDorees` | Liste des images dorées et de leurs versions |
| `date` | Date et heure du déploiement |
| `resultat` | `reussi`, `echoue` ou `retour-arriere` |

Exemple :

```json
{
  "application": "monex.diagb",
  "environnement": "production",
  "release": { "composant": "diagb", "version": "1.2", "empreinte": "<empreinte>" },
  "imagesDorees": [
    { "nom": "temurin-21", "version": "<version>" },
    { "nom": "mongodb-7.0", "version": "<version>" },
    { "nom": "mariadb-11", "version": "<version>" }
  ],
  "date": "2026-09-30T08:00:00Z",
  "resultat": "reussi"
}
```

### 8.3 Alimentation de la CMDB

| Donnée | Source | Mode |
|---|---|---|
| Releases publiées | Nexus | Lecture via l'API de recherche de Nexus, par identifiant d'application |
| Déploiements | Pipeline | Événements (8.2) |
| Relation application vers image dorée | Événements | Mise à jour à chaque déploiement |
| Incidents rattachés aux releases | ITSM | Lien incident vers release, tenu dans l'ITSM |

Toute correction se fait à la source, jamais directement dans la CMDB.

## 9. Mesure de la livraison (données à disposition)

| Indicateur | Données requises | Origine |
|---|---|---|
| Fréquence de déploiement | `resultat = reussi` par application et par période | Événement |
| Délai de mise en production | `dateCommit`, date de promotion, `date` de déploiement | Manifeste, Nexus, événement |
| Taux d'échec des changements | `resultat`, incidents liés | Événement, ITSM |
| Délai de rétablissement | Ouverture et clôture de l'incident lié | ITSM |

Pour un COTS, le point de départ est `dateReception`.

## 10. Exploitation

### 10.1 Disponibilité et sauvegarde

- Haute disponibilité selon l'offre Pro retenue ; répartiteur de charge devant les nœuds.
- Sauvegarde cohérente base + stockage de blobs, avec test de restauration périodique.
- Objectifs de disponibilité, RPO et RTO : à fixer avec l'exploitation. Nexus est un composant critique de la chaîne de livraison.

### 10.2 Stockage

- Quotas souples par blob store, alerte avant saturation.
- Nettoyage puis compactage planifiés (3.4).
- Suivi de la croissance par format ; les images Docker et les images de machine sont les premiers consommateurs.

### 10.3 Supervision

| Élément | Suivi |
|---|---|
| Disponibilité du service | Sonde de santé via le répartiteur |
| Espace disque et blob stores | Seuils d'alerte |
| Latence du proxy | Comparaison avec le téléchargement direct (risque de contournement) |
| Quarantaines en attente | Délai de traitement par la SSI, cible 1 jour ouvré |
| Journal d'audit | Conservé et centralisé |

### 10.4 Montées de version

Procédure planifiée, testée sur un environnement de préproduction à l'identique, avec sauvegarde préalable.

## 11. Critères de recette

| Réf. | Critère | Vérification |
|---|---|---|
| R1 | Un redéploiement sur une release est refusé | Tentative de republication sur un chemin existant |
| R2 | Une promotion sans manifeste ou sans SBOM échoue | Promotion d'un candidat incomplet |
| R3 | Une promotion avec commit absent d'une branche protégée échoue | Candidat avec commit inconnu |
| R4 | Un éditeur d'une application ne peut publier que sous son préfixe | Publication sous le préfixe d'une autre application refusée |
| R5 | Le composant tiers non conforme est mis en quarantaine à l'entrée | Demande d'un composant à licence interdite |
| R6 | Un COTS est promu avec SBOM marqué reconstitué | Cycle complet sur un composant de test |
| R7 | L'événement de déploiement met à jour la CMDB, y compris les versions d'images dorées | Déploiement de test |
| R8 | Retrouver tous les artefacts d'une application par son identifiant | Recherche par préfixe dans chaque dépôt hébergé |
| R9 | Une mise à jour d'image dorée liste les applications concernées | Requête de relation dans la CMDB |
| R10 | Restauration réussie depuis la sauvegarde | Test de restauration sur préproduction |
| R11 | Accès anonyme refusé | Tentative de lecture sans authentification |

## 12. Points à confirmer avant réalisation

| Sujet | Question | Impact |
|---|---|---|
| Pare-feu de dépôt et raw | Quelle couverture pour le format raw ? | Niveau de contrôle des COTS et des images dorées |
| Promotion | Quels formats supportent la promotion native par étiquette dans la version retenue ? | Sinon repli par republication contrôlée par empreinte |
| Docker | Sous-domaine par dépôt ou autre routage | Configuration du répartiteur et des clients |
| npm | Règle de portée dérivée de l'identifiant | Compatibilité avec l'écosystème |
| Haute disponibilité | Offre Pro, base externe, stockage de blobs | Architecture d'infrastructure |
| Rétention | Durées de nettoyage des candidats et des caches | Coût de stockage |
| Objectifs d'exploitation | Disponibilité, RPO, RTO | Dimensionnement et sauvegarde |
| Sélecteurs de contenu | Un sélecteur par application ou par lot | Volume d'administration pour environ 60 applications |
| Hypothèses de format | Images dorées Temurin, MongoDB et MariaDB en raw ; SBOM de Jira reconstitué par nos soins | Chemins et contrôles |
