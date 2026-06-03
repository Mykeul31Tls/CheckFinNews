# CLAUDE.md — Règles du projet CheckFinNews

Ce fichier est lu en premier à chaque session. Toutes les règles sont obligatoires.

---

## 1. Identité du projet

**Nom** : CheckFinNews  
**Copyright** : CheckFinNews - Copyright 2026 MBO | Mind Bolt Objects  
**Description** : Application de veille financière en temps quasi-réel.  
Surveille l'information fondamentale (communiqués, résultats, contrats, macro, géopolitique)
susceptible d'impacter le cours d'une action avant que le marché l'ait intégré.

---

## 2. Stack technique

- **Version courante** : HTML monofichier (`index.html`), sans serveur, sans build
- **Cible finale** : Backend Python + Docker sur Synology NAS
- **LLM de scoring** : `claude-haiku-4-5-20251001` via API Anthropic
- **Notifications** : ntfy.sh (push iPhone)
- **Persistance POC** : localStorage
- **Persistance finale** : SQLite

---

## 3. Charte graphique (immuable)

- Thème sombre exclusivement
- `--bg: #080b14` | `--surface1: #0d1120` | `--surface2: #161b2e`
- Accent : `#4d9fff` | Hausse : `#00e5a0` | Baisse : `#ff5757`
- Typographie : IBM Plex Sans (corps), IBM Plex Mono (données)
- Logo : "Check" en blanc, "FinNews" en `#4d9fff`
- Header : dégradé `160deg #0d1a3a -> #080b14` avec halos lumineux

---

## 4. Règles de code

- Aucun emoji ni symbole spécial dans le code source, les commentaires ou les logs
- Les caractères accentués français sont autorisés dans les commentaires et les textes visibles
- Pas de commentaires décrivant le "quoi" — uniquement le "pourquoi" si non évident
- Pas de docstrings multi-lignes
- Pas de gestion d'erreur pour des scénarios impossibles
- Ne pas ajouter de fonctionnalités au-delà du périmètre demandé

---

## 5. Sécurité

- La clé API Anthropic ne doit jamais être dans le code source
- Elle est stockée en localStorage (`cfn_cfg`) et saisie via l'interface
- Tronquer les contenus RSS avant envoi au LLM (max 400 chars de résumé)

---

## 6. Paramètres utilisateur configurables

Tous modifiables sans toucher au code :

| Paramètre | Défaut |
|---|---|
| Clé API Anthropic | — |
| Topic ntfy.sh | — |
| Tickers personnels | [] |
| Mots-clés additionnels | [] |
| Seuil de score | 7/10 |
| Intervalle de polling | 5 min |
| Âge max des articles | 6 h |
| Max alertes par cycle | 3 |

---

## 7. Pipeline de traitement

```
Flux RSS (Google News direct + sources tierces via proxy CORS)
  -> Lots de 6 en parallèle
  -> Dédup hash (titre+source) + filtre âge
  -> Limitation 5 items/flux
  -> Pré-filtre thématique (noms sociétés + mots-clés + tickers >= 4 chars)
  -> Plafond 15 items à scorer par cycle
  -> Scoring Claude Haiku (JSON structuré)
  -> Seuil d'alerte configurable
  -> Dédup sémantique (même ticker+direction dans les 2h)
  -> Plafond alertes/cycle
  -> Notification ntfy.sh + affichage dans l'app
```

---

## 8. Périmètre de surveillance

**Indices couverts** : CAC40, EuroStoxx50, DAX, NASDAQ-100, S&P500 top 100  
**Secteurs prioritaires** : technologie, défense, énergie, luxe, télécoms, pharma, finance, auto, aéronautique, spatial  
**Hors périmètre** : PME et micro-caps absentes des grands indices

---

## 9. Sources RSS

- **Google News** (`news.google.com/rss/search?q=…`) : CORS natif, appel direct
- **Autres sources** (SEC 8-K, BusinessWire, GlobeNewswire, Boursorama…) : via chaîne de proxies CORS (`corsproxy.io` → `allorigins.win` → `thingproxy` → `codetabs`)
- **Limite connue** : les proxies publics sont peu fiables sur iOS — problème résolu dans la version Docker

---

## 10. Branche de développement

- Développer sur la branche dédiée à la fonctionnalité en cours
- Ne jamais pousser directement sur `main` sans que code + tests soient fonctionnels
- Message de commit : description concise du "pourquoi", pas du "quoi"

---

## 11. Mode Session

**RÈGLE RÉGALIENNE : Chaque session couvre une seule fonctionnalité.**  
La session se termine dès que code + tests sont fonctionnels.

### Comportement obligatoire en fin de session

1. Commit et push dans la branche `main` du dépôt
2. Afficher un récapitulatif :
   - Fichiers créés ou modifiés
   - Tests ajoutés
3. Demander : **"Souhaitez-vous ouvrir une nouvelle conversation pour continuer ?"**
4. Si oui, générer le prompt complet suivant :

```
--- PROMPT NOUVELLE SESSION ---

## Contexte
[Description courte du projet et de son état actuel]
Lire CLAUDE.md en premier — il contient toutes les règles du projet.

## Déjà implémenté
- Étape 1 : [nom] -> [fichiers créés/modifiés]
- Étape 2 : [nom] -> [fichiers créés/modifiés]

## Prochaine fonctionnalité
[Description précise, critères d'acceptation, fichiers concernés]

## Instruction
Implémenter en respectant intégralement CLAUDE.md.
Arrêter dès que code + tests sont fonctionnels.

--- FIN DU PROMPT ---
```
