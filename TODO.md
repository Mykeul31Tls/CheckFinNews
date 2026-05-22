# TODO - Systeme de veille financiere a impact cours

## Vue d'ensemble

Systeme autonome de surveillance des flux d'information financiere.
Detecte en quasi-temps reel les nouvelles susceptibles d'impacter fortement le cours
d'une action et envoie une notification push via ntfy.sh.

Hebergement : conteneur Docker sur Synology NAS.
Notifications : ntfy.sh (application iOS App Store, aucun compte requis).
Scoring : Claude API (modele Haiku pour le volume, Sonnet pour les cas ambigus).
Cout estimé : < 0,05 EUR/jour (API Claude uniquement).

---

## Architecture generale

```
[Flux RSS / API publiques gratuites]
          |
          v (toutes les 5 minutes, cron interne Python)
[Collecteur RSS - feedparser]
          |
          v (deduplication SQLite par hash titre+source)
[Filtre pre-scoring rapide - mots-cles tickers surveilles]
          |
          v (uniquement les news pertinentes)
[Scoring LLM - Claude API Haiku]
          |
          v (impact_score >= seuil configurable)
[Envoi alerte - ntfy.sh HTTP POST]
          |
          v
[iPhone - app ntfy, notification push native]
```

---

## Structure du projet

```
financial-watcher/
|-- docker-compose.yml
|-- Dockerfile
|-- requirements.txt
|-- config.yml                  # configuration utilisateur (tickers, seuils, feeds)
|-- .env                        # secrets (ANTHROPIC_API_KEY, NTFY_TOPIC)
|-- src/
|   |-- main.py                 # point d'entree, boucle principale
|   |-- collector.py            # collecte et parsing des flux RSS
|   |-- deduplicator.py         # base SQLite, gestion des items deja vus
|   |-- pre_filter.py           # filtre rapide avant appel LLM
|   |-- scorer.py               # appel Claude API, parsing reponse JSON
|   |-- notifier.py             # envoi notification ntfy.sh
|   |-- logger.py               # logging structure vers fichier + stdout
|   `-- feeds.py                # definition et categorisation de tous les flux RSS
|-- data/
|   `-- seen.db                 # SQLite (genere automatiquement, persiste via volume Docker)
|-- logs/
|   `-- watcher.log             # log rotatif (genere automatiquement)
`-- tests/
    |-- test_collector.py
    |-- test_pre_filter.py
    |-- test_scorer.py
    `-- test_notifier.py
```

---

## TACHE 1 - Configuration du projet

### 1.1 - Fichier config.yml

Creer `config.yml` avec la structure suivante :

```yaml
# Tickers surveilles - le pre-filtre ne transmet au LLM que les news
# qui mentionnent au moins un de ces identifiants (insensible a la casse)
tickers:
  - IBM
  - EUTELSAT
  - ETL         # mnemonique Euronext d'Eutelsat
  - AIRBUS
  - AIR         # mnemonique Euronext d'Airbus
  - NVDA
  - MSFT
  - AAPL
  - TSLA
  # Ajouter autant que souhaite

# Mots-cles additionnels declenchant le pre-filtre independamment du ticker
# Utile pour les news sectorielles (ex: "quantique", "satellite LEO")
keywords_sectoriels:
  - quantique
  - quantum
  - satellite
  - LEO orbit
  - contrat defense
  - milliard
  - billion
  - financement gouvernemental
  - government funding
  - acquisition
  - fusion
  - merger
  - rachat

# Seuil de score minimum pour declencher une alerte (0-10)
# Recommande : 7 pour etre selectif, 6 pour etre large
seuil_alerte: 7

# Intervalle de polling en secondes (300 = 5 minutes)
polling_interval_seconds: 300

# Modele Claude a utiliser pour le scoring de masse
# Utiliser claude-haiku-4-5-20251001 pour le cout, claude-sonnet-4-6 pour la qualite
claude_model: "claude-haiku-4-5-20251001"

# Nombre maximum de tokens pour la reponse du LLM (le JSON de scoring est court)
claude_max_tokens: 400

# Langue de l'alerte ntfy ("fr" ou "en")
langue_alerte: "fr"

# Retention des items deja vus en base (en jours)
deduplication_retention_days: 7

# Taille max du log rotatif en Mo
log_max_size_mb: 10
log_backup_count: 3
```

### 1.2 - Fichier .env

```env
ANTHROPIC_API_KEY=sk-ant-...
NTFY_TOPIC=mon-alerte-bourse-xyz42
# Choisir un nom de topic long et difficile a deviner (il est public par nature sur ntfy.sh)
```

### 1.3 - Fichier requirements.txt

```
feedparser==6.0.11
requests==2.32.3
anthropic==0.40.0
pyyaml==6.0.2
schedule==1.2.2
```

---

## TACHE 2 - Module feeds.py

Definir la liste exhaustive des flux RSS surveilles, organises par categorie.

### 2.1 - Flux actions US

| Identifiant | URL | Description |
|---|---|---|
| SEC_8K_ALL | `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&type=8-K&output=atom` | Tous les depots 8-K SEC (info materielle obligatoire) |
| GLOBENEWSWIRE_TECH | `https://www.globenewswire.com/RssFeed/subjectcode/SCT-Technology/keyword/` | Communiques tech |
| GLOBENEWSWIRE_DEFENSE | `https://www.globenewswire.com/RssFeed/subjectcode/SCT-DefenseAndSecurity/` | Communiques defense |
| BUSINESSWIRE_TECH | `https://feed.businesswire.com/rss/home/?rss=G22` | Communiques corporate tech |
| PRNEWSWIRE_TECH | `https://www.prnewswire.com/rss/news-releases-list.rss` | Communiques PR |
| REUTERS_TECH | `https://feeds.reuters.com/reuters/technologyNews` | Reuters technologie |
| REUTERS_BUSINESS | `https://feeds.reuters.com/reuters/businessNews` | Reuters business |
| MARKETWATCH | `https://feeds.marketwatch.com/marketwatch/topstories/` | MarketWatch |

Note : verifier la disponibilite effective de chaque URL au moment du developpement,
certains flux Reuters/MarketWatch changent d'URL periodiquement.

### 2.2 - Flux actions europeennes / francaises

| Identifiant | URL | Description |
|---|---|---|
| AMF_DECISIONS | `https://www.amf-france.org/fr/rss/actualites` | Decisions AMF (equivalent 8-K FR) |
| EURONEXT_NEWS | `https://live.euronext.com/fr/rss/news` | Actualites Euronext |
| BOURSORAMA_ACT | `https://www.boursorama.com/bourse/actualites/rss/` | Boursorama actualites |
| ZONEBOURSE | `https://www.zonebourse.com/rss/actualites.aspx` | Zone Bourse |
| LES_ECHOS | `https://www.lesechos.fr/rss/rss_finance.xml` | Les Echos finance |
| LATRIBUNE_FINANCE | `https://www.latribune.fr/rss/rubriques/finance.html` | La Tribune finance |

### 2.3 - Flux Google News par ticker surveille (generes dynamiquement)

Pour chaque ticker de config.yml, generer une URL :
`https://news.google.com/rss/search?q={TICKER}+bourse&hl=fr&gl=FR&ceid=FR:fr`

Ces URLs sont generees au demarrage depuis la liste de tickers de config.yml,
pas hardcodees dans feeds.py.

### 2.4 - Structure de donnee d'un item collecte

```python
{
    "id": "hash_md5_titre_source",    # cle de deduplication
    "source_id": "SEC_8K_ALL",
    "source_label": "SEC EDGAR 8-K",
    "title": "...",
    "summary": "...",                  # peut etre vide selon la source
    "link": "https://...",
    "published": "2026-05-22T12:03:00Z",  # datetime ISO, ou None si absent du flux
    "collected_at": "2026-05-22T12:05:00Z"
}
```

---

## TACHE 3 - Module deduplicator.py

Gestion de la base SQLite pour eviter de scorer et alerter plusieurs fois le meme item.

### 3.1 - Schema de la table

```sql
CREATE TABLE IF NOT EXISTS seen_items (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    source_id TEXT NOT NULL,
    collected_at TEXT NOT NULL,
    scored INTEGER DEFAULT 0,
    impact_score REAL DEFAULT NULL,
    alerted INTEGER DEFAULT 0
);
```

### 3.2 - Fonctions a implementer

- `init_db(db_path)` : cree la base si elle n'existe pas, applique le schema
- `is_seen(item_id)` : retourne True si l'item est deja en base
- `mark_seen(item)` : insere l'item en base (scored=0, alerted=0)
- `mark_scored(item_id, impact_score)` : met a jour scored=1 et impact_score
- `mark_alerted(item_id)` : met a jour alerted=1
- `purge_old(retention_days)` : supprime les items plus vieux que retention_days

### 3.3 - Contrainte importante

Le fichier `data/seen.db` doit etre dans un repertoire monte comme volume Docker
pour persister entre les redemarrages du conteneur. Ne pas le stocker dans l'image.

---

## TACHE 4 - Module collector.py

### 4.1 - Fonction principale

```python
def collect_new_items(feeds: dict, deduplicator) -> list[dict]:
```

- Parcourir tous les flux definis dans feeds.py
- Pour chaque entree du flux, calculer le hash MD5 de `(title + source_id)`
- Ignorer si deja vu en base (appel deduplicator.is_seen)
- Sinon : construire l'item, appeler deduplicator.mark_seen, ajouter a la liste
- Retourner la liste des items nouveaux

### 4.2 - Gestion des erreurs de flux

- Timeout par flux : 10 secondes maximum
- Si un flux est inaccessible : logger un WARNING, continuer les autres flux
- Ne jamais lever d'exception fatale pour un flux indisponible
- Si un flux retourne une erreur HTTP >= 400 : logger et continuer

### 4.3 - Normalisation du champ published

Certains flux n'incluent pas de date de publication. Dans ce cas, utiliser
`collected_at` comme fallback. Normaliser toutes les dates en ISO 8601 UTC.

---

## TACHE 5 - Module pre_filter.py

Filtre rapide AVANT l'appel LLM pour eviter de payer des tokens sur des news
sans rapport avec les tickers ou secteurs surveilles.

### 5.1 - Logique de filtrage

Un item passe le pre-filtre si au moins UNE des conditions est vraie :
1. Le titre ou le summary contient au moins un ticker de config.yml (insensible a la casse)
2. Le titre ou le summary contient au moins un mot-cle sectoriel de config.yml
3. La source est SEC_8K_ALL (les 8-K doivent tous etre scores car tres cibles)

### 5.2 - Fonction principale

```python
def pre_filter(items: list[dict], tickers: list[str], keywords: list[str]) -> list[dict]:
```

Retourne uniquement les items passant le filtre.
Logger le nombre d'items recus et le nombre passant le filtre a chaque cycle.

---

## TACHE 6 - Module scorer.py

### 6.1 - Prompt systeme (a implementer exactement)

```
Tu es un analyste financier specialise dans l'evaluation de l'impact des nouvelles sur les cours boursiers.

Pour chaque news fournie, tu dois repondre UNIQUEMENT avec un objet JSON valide.
Aucun texte avant ou apres. Aucun bloc markdown. Juste le JSON brut.

Format de reponse obligatoire :
{
  "impact_score": <entier de 0 a 10>,
  "direction": "<hausse|baisse|neutre>",
  "confiance": "<haute|moyenne|faible>",
  "tickers_concernes": ["<TICKER1>", "<TICKER2>"],
  "raison": "<une seule phrase courte en francais, max 120 caracteres>"
}

Criteres de scoring :
- 9-10 : evenement exceptionnel (acquisition majeure, contrat gouvernemental massif, faillite, fusion)
- 7-8  : evenement significatif (contrat important, financement notable, partenariat strategique)
- 5-6  : evenement moderement impactant (resultats en ligne, contrat mineur, nomination)
- 3-4  : evenement faible impact (declaration standard, publication reguliere)
- 0-2  : evenement sans impact cours (communique administratif, calendrier, avis divers)

Pour tickers_concernes : utiliser les symboles boursiers standard (ex: IBM, ETL, AIR).
Si aucun ticker identifiable : tableau vide.
```

### 6.2 - Fonction principale

```python
def score_item(item: dict, config: dict) -> dict | None:
```

- Appeler l'API Claude avec le prompt systeme ci-dessus
- Message utilisateur : `Titre: {item['title']}\nResume: {item['summary']}\nSource: {item['source_label']}`
- Parser la reponse JSON (strip des backticks eventuels en securite)
- En cas d'erreur de parsing JSON : logger ERROR, retourner None
- En cas d'erreur API (rate limit, timeout) : attendre 5 secondes, reessayer une fois, sinon logger ERROR et retourner None
- Mettre a jour la base via deduplicator.mark_scored

### 6.3 - Gestion du cout

- Utiliser le modele defini dans config.yml (haiku par defaut)
- Logger le nombre de tokens utilises par appel (disponible dans la reponse API Anthropic)
- Garder un compteur quotidien de tokens dans un fichier `data/usage.json`

---

## TACHE 7 - Module notifier.py

### 7.1 - Fonction principale

```python
def send_ntfy_alert(item: dict, score: dict, config: dict, topic: str) -> bool:
```

### 7.2 - Construction du message ntfy

- **Title** (header ntfy) : `[{direction_emoji_texte}] {tickers} - Score {impact_score}/10`
  - direction_emoji_texte : "HAUSSE" si hausse, "BAISSE" si baisse, "NEUTRE" si neutre
  - Ne pas utiliser d'emoji dans le titre (contrainte projet)
- **Body** (data ntfy) : 
  ```
  {raison}
  
  Source: {source_label}
  Titre: {title}
  Confiance: {confiance}
  
  {link}
  ```
- **Priority** : 
  - impact_score >= 9 : `urgent`
  - impact_score >= 7 : `high`
  - sinon : `default`
- **Tags** (optionnel, affiche une icone dans l'app ntfy) : laisser vide pour eviter les restrictions

### 7.3 - Appel HTTP

```python
requests.post(
    f"https://ntfy.sh/{topic}",
    data=body.encode("utf-8"),
    headers={
        "Title": title,
        "Priority": priority,
        "Content-Type": "text/plain; charset=utf-8"
    },
    timeout=10
)
```

Retourner True si HTTP 200, False sinon. Logger toute erreur.

---

## TACHE 8 - Module logger.py

- Logger standard Python avec RotatingFileHandler
- Sortie simultanee vers `logs/watcher.log` ET vers stdout (pour `docker logs`)
- Format : `YYYY-MM-DD HH:MM:SS | LEVEL | module | message`
- Rotation : taille max et nombre de backups definis dans config.yml
- Niveaux utilises :
  - INFO : demarrage, cycle de collecte, items trouves, alertes envoyees
  - WARNING : flux RSS indisponible, reponse LLM inattendue
  - ERROR : erreur API Claude, erreur ntfy, erreur SQLite

---

## TACHE 9 - Module main.py

### 9.1 - Demarrage

1. Charger config.yml et les variables d'environnement (.env via python-dotenv)
2. Valider que ANTHROPIC_API_KEY et NTFY_TOPIC sont presents, sinon exit(1) avec message clair
3. Initialiser le logger
4. Initialiser la base SQLite via deduplicator.init_db
5. Construire la liste complete des flux (feeds.py + Google News dynamiques depuis config.yml)
6. Logger le recap de demarrage : nombre de tickers surveilles, nombre de flux, intervalle de polling

### 9.2 - Boucle principale

```python
def run_cycle():
    # 1. Collecte
    new_items = collector.collect_new_items(feeds, deduplicator)
    logger.info(f"Cycle : {len(new_items)} nouveaux items collectes")

    # 2. Pre-filtre
    filtered = pre_filter.pre_filter(new_items, config['tickers'], config['keywords_sectoriels'])
    logger.info(f"Pre-filtre : {len(filtered)}/{len(new_items)} items transmis au scoring")

    # 3. Scoring et alerte
    for item in filtered:
        score = scorer.score_item(item, config)
        if score is None:
            continue
        if score['impact_score'] >= config['seuil_alerte']:
            success = notifier.send_ntfy_alert(item, score, config, NTFY_TOPIC)
            if success:
                deduplicator.mark_alerted(item['id'])
                logger.info(
                    f"ALERTE envoyee | tickers={score['tickers_concernes']} "
                    f"| score={score['impact_score']} | {item['title'][:80]}"
                )

    # 4. Purge periodique (une fois par jour suffit)
    deduplicator.purge_old(config['deduplication_retention_days'])

schedule.every(config['polling_interval_seconds']).seconds.do(run_cycle)

# Executer un premier cycle immediatement au demarrage
run_cycle()

while True:
    schedule.run_pending()
    time.sleep(1)
```

---

## TACHE 10 - Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/
COPY config.yml .

# Les dossiers data/ et logs/ sont montes comme volumes Docker
# Ne pas les copier dans l'image
RUN mkdir -p data logs

CMD ["python", "src/main.py"]
```

---

## TACHE 11 - docker-compose.yml

```yaml
version: "3.9"

services:
  financial-watcher:
    build: .
    container_name: financial-watcher
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      # Persistance de la base SQLite entre redemarrages
      - ./data:/app/data
      # Persistance des logs
      - ./logs:/app/logs
      # Permettre de modifier config.yml sans rebuild
      - ./config.yml:/app/config.yml:ro
    environment:
      - TZ=Europe/Paris
    # Pas de ports exposes, le conteneur ne sert pas de trafic entrant
```

Note Synology : dans le gestionnaire Docker Synology (ou Portainer si installe),
importer ce docker-compose.yml. Les chemins de volumes seront relatifs au
repertoire du projet sur le NAS.

---

## TACHE 12 - Tests

### 12.1 - test_collector.py

- Tester la collecte sur un flux RSS public stable (ex: Reuters)
- Tester la deduplication : un meme item ne doit pas etre retourne deux fois
- Tester le comportement si un flux est inaccessible (mock timeout)

### 12.2 - test_pre_filter.py

- Tester qu'un item mentionnant "IBM" passe le filtre
- Tester qu'un item sans ticker ni mot-cle sectoriel ne passe pas
- Tester l'insensibilite a la casse ("ibm" == "IBM")
- Tester qu'un item d'une source SEC_8K_ALL passe toujours

### 12.3 - test_scorer.py

- Mocker l'appel API Anthropic (ne pas consommer de tokens en test)
- Tester le parsing JSON de la reponse
- Tester la gestion d'une reponse JSON malformee (doit retourner None sans planter)
- Tester la gestion d'une erreur API (timeout mock)

### 12.4 - test_notifier.py

- Mocker l'appel HTTP ntfy.sh
- Verifier que le header Title est bien construit
- Verifier que la priorite est correcte selon le score
- Tester le comportement si ntfy.sh retourne une erreur HTTP

---

## TACHE 13 - Mise en production sur Synology

### 13.1 - Preparation des fichiers sur le NAS

1. Creer le dossier `/volume1/docker/financial-watcher/` sur le Synology
2. Y deposer tous les fichiers du projet (via SSH, SMB ou Synology File Station)
3. Creer le fichier `.env` avec les vraies valeurs (ANTHROPIC_API_KEY, NTFY_TOPIC)
4. Verifier que les dossiers `data/` et `logs/` existent et sont accessibles en ecriture

### 13.2 - Demarrage

Via SSH sur le Synology :
```bash
cd /volume1/docker/financial-watcher
docker-compose up -d --build
docker logs -f financial-watcher
```

### 13.3 - Verification de fonctionnement

- Verifier dans les logs qu'un premier cycle s'est execute sans erreur
- Envoyer une notification de test manuelle pour valider ntfy.sh :
  ```bash
  curl -d "Test de notification - systeme operationnel" \
       -H "Title: Test veille financiere" \
       https://ntfy.sh/MON_TOPIC
  ```
- Verifier la reception sur iPhone dans l'app ntfy

### 13.4 - Maintenance

- Les logs sont dans `./logs/watcher.log`, rotatifs automatiquement
- La base SQLite est dans `./data/seen.db` (consultable avec DB Browser for SQLite)
- Pour modifier les tickers surveilles : editer `config.yml`, redemarrer le conteneur
  (`docker-compose restart`)
- Pour mettre a jour l'image apres modification du code :
  ```bash
  docker-compose up -d --build
  ```

---

## Recapitulatif des dependances

| Librairie | Version | Usage |
|---|---|---|
| feedparser | 6.0.11 | Parsing des flux RSS/Atom |
| requests | 2.32.3 | Appels HTTP (ntfy, flux non-feedparser) |
| anthropic | 0.40.0 | Client officiel Claude API |
| pyyaml | 6.0.2 | Lecture de config.yml |
| schedule | 1.2.2 | Planification du cycle de polling |
| python-dotenv | 1.0.1 | Chargement du fichier .env |

Toutes disponibles sur PyPI, aucune dependance systeme particuliere.
Python >= 3.11 requis.

---

## Points d'attention pour Claude Code

1. Ne jamais utiliser d'emoji ou de symboles speciaux dans le code, les commentaires,
   les chaines de caracteres ou les messages de log. Les accents francais sont autorises.

2. Le fichier `.env` ne doit jamais etre commite. Ajouter un `.gitignore` couvrant
   `.env`, `data/`, `logs/`.

3. La base SQLite doit etre initialisee avant le premier cycle, pas au premier acces.

4. Tous les appels reseau (RSS, Claude API, ntfy) doivent avoir un timeout explicite.

5. Le script doit survivre a l'indisponibilite d'un flux ou d'une API externe :
   aucune exception non capturee ne doit arreter la boucle principale.

6. Le champ `summary` d'un item RSS peut etre du HTML. Faire un strip des balises
   avant de l'envoyer au LLM (utiliser html.parser de la stdlib, pas de dependance externe).
