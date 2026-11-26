# 📦 RENDU — Projet Portfolio · Équipe Black Hat

Dépôt : <https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat>

---

## 1. Documents de conception

| Document | Lien |
|---|---|
| Architecture (modèles MongoDB + Neo4j, requêtes R1–R8, slugs pivots) | [ARCHITECTURE.md](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/blob/main/ARCHITECTURE.md) |
| Contrat d'API (routes, paramètres, réponses, erreurs) | [API_ROUTES.md](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/blob/main/API_ROUTES.md) |

---

## 2. Backend déployé

| | Lien |
|---|---|
| API (Vercel) | `https://2026-2027-rouen-portfolio-blackhat-plum.vercel.app/docs` |

- Code : FastAPI, dossier [`backend/`](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/backend), entrypoint `src.portfolio_api.main:app` (voir `[tool.vercel]` dans `backend/pyproject.toml`).
- Bases : **MongoDB Atlas** (collections `candidats`, `projets`) et **Neo4j** (nœuds `Candidat`, `Projet`, `Competence`, `Domaine`).
- Variables d'environnement configurées sur Vercel : `MONGO_URL`, `MONGO_DB`, `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD`, `CORS_ORIGINS`.

---

## 3. Valider la connexion aux bases


```bash
API=https://2026-2027-rouen-portfolio-blackhat-plum.vercel.app
```

### 3.1 Santé de l'API

```bash
curl "$API/api/health"
# {"status":"ok"}
```

### 3.2 Tests d'intégration de connexion aux bases

`/api/health` répond `ok` même si une base est inaccessible : il ne prouve donc pas la connexion. Pour vérifier directement MongoDB **et** Neo4j, utilisez les tests d'intégration de [`backend/tests/test_connexions.py`](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/blob/main/backend/tests/test_connexions.py) :

| Test | Ce qu'il vérifie |
|---|---|
| `test_connexion_mongo` | `ping` sur le serveur MongoDB, nom de la base (`MONGO_DB`) et lecture des collections (droits d'accès). |
| `test_connexion_neo4j` | `verify_connectivity()` du driver puis exécution de `RETURN 1 AS ok`. |

Lancement, depuis `backend/`, avec un `.env` contenant `MONGO_URL`, `MONGO_DB`, `NEO4J_URI`, `NEO4J_USER` et `NEO4J_PASSWORD` :

```bash
cd backend
uv run pytest tests/test_connexions.py -v
```

- Les deux tests `PASSED` : les deux bases sont joignables avec ces identifiants.
- Un test `FAILED` : la base concernée est inaccessible (URI, identifiants ou réseau). Par exemple, si Neo4j est arrêté ou mal configuré, `test_connexion_neo4j` échoue alors que `test_connexion_mongo` passe.
- Un test `SKIPPED` : les variables de cette base ne sont pas définies dans le `.env`.

La suite complète (`just test`, ou `uv run pytest -q` depuis `backend/`) teste aussi les routes, le seed et les requêtes du graphe.

### 3.3 Endpoints servis par MongoDB

| Req. | Endpoint | Exemple de test | Résultat attendu |
|---|---|---|---|
| R1 | `GET /api/profil` | `curl "$API/api/profil"` | Profil de `papa-daouda-ndiaye` avec expériences, formations, compétences |
| R2 | `GET /api/projets` | `curl "$API/api/projets"` | 5 projets triés par année décroissante |
| R2 (filtre) | `GET /api/projets?competence=…` | `curl "$API/api/projets?competence=python"` | Seuls les projets qui utilisent `python` |
| Détail | `GET /api/projets/{projet_slug}` | `curl "$API/api/projets/portfolio-nosql"` | Détail du projet (avec `description_longue`) |
| R8 | `GET /api/experiences?lieu=…` | `curl "$API/api/experiences?lieu=Dakar"` | Expériences situées à Dakar |
| R3 | `GET /api/stats/competences` | `curl "$API/api/stats/competences"` | Statistiques par compétence (`nb_projets`, `derniere_annee`) |
| R6 | `GET /api/stats/competences?limit=5` | `curl "$API/api/stats/competences?limit=5"` | Top 5 des compétences |

### 3.4 Endpoints servis par Neo4j

| Req. | Endpoint | Exemple de test | Résultat attendu |
|---|---|---|---|
| R4 | `GET /api/graph/projets/{projet_slug}/similaires` | `curl "$API/api/graph/projets/portfolio-nosql/similaires"` | Projets partageant des compétences, triés par `nb_communes` |
| R5 | `GET /api/graph/competences` | `curl "$API/api/graph/competences"` | `{nodes, links}` : Candidat → Projet → Compétence → Domaine |
| R7 | `GET /api/graph/domaines/{domaine_slug}/competences` | `curl "$API/api/graph/domaines/bases-de-donnees/competences"` | Compétences du domaine « Bases de données » |

Un code `404` « Candidat introuvable » ou un nœud absent indique que la base n'a pas été alimentée ; une erreur `500` indique un problème de connexion.

### 3.5 Requêtes à exécuter directement dans les bases

**MongoDB (mongosh / Atlas Data Explorer)** :

```js
// R1 — profil complet
db.candidats.findOne({ candidat_slug: "papa-daouda-ndiaye" }, { _id: 0 })

// R2 — projets utilisant une compétence
db.projets.find({ competences: "python" }, { _id: 0 }).sort({ annee: -1 })

// R3 — statistiques par compétence
db.projets.aggregate([
  { $unwind: "$competences" },
  { $group: { _id: "$competences", nb_projets: { $sum: 1 }, derniere_annee: { $max: "$annee" } } },
  { $sort: { nb_projets: -1, _id: 1 } }
])

// Index
db.candidats.getIndexes()
db.projets.getIndexes()
```

**Neo4j (Neo4j Browser / Aura)** :

```cypher
// Vérifier le contenu du graphe
MATCH (n) RETURN labels(n)[0] AS type, count(*) AS nb;

// R4 — projets similaires
MATCH (p:Projet {projet_slug: "portfolio-nosql"})-[:UTILISE]->(c:Competence)<-[:UTILISE]-(autre:Projet)
RETURN autre.projet_slug, collect(c.skill_slug) AS competences_communes, count(c) AS nb_communes
ORDER BY nb_communes DESC;

// R5 — graphe de compétences
MATCH (ca:Candidat)-[:A_REALISE]->(p:Projet)-[:UTILISE]->(c:Competence)-[:APPARTIENT_A]->(d:Domaine)
RETURN ca, p, c, d;

// R7 — compétences d'un domaine
MATCH (c:Competence)-[:APPARTIENT_A]->(d:Domaine {domaine_slug: "bases-de-donnees"})
RETURN c.skill_slug, c.nom ORDER BY c.nom;

// Contraintes d'unicité
SHOW CONSTRAINTS;
```

---

## 4. Portfolios déployés

| Membre | Portfolio | Lien déployé | Code source |
|---|---|---|---|
| Papa Daouda NDIAYE (`@PapaDaoudaNDIAYE`) | Sunset Drive — portfolio jouable (Next.js + Phaser) | <https://portfolio-papa-one.vercel.app> | [frontend-PapaDaoudaNDIAYE](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/frontends/frontend-PapaDaoudaNDIAYE) |
| Cédric Akle (`@nicktoudonou-spec`) | Portfolio Data Analyst | <https://cedric-akle-portfolio.vercel.app/#/lab> | [frontend-nicktoudonou-spec](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/frontends/frontend-nicktoudonou-spec) |
| Hilary BOCO (`@HilaryBoco06`) | Carnet de voyage — portfolio avec île 3D interactive (React + Three.js) | <https://hilary-boco-portfolio.vercel.app> | [frontend-HilaryBoco06](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/frontends/frontend-HilaryBoco06) |
| **@Bibou172** (`@Bibou172`) | Clean Girl Portfolio — portfolio interactif & scène Canvas 2D | <https://portfolioblackhatbeleive-frontend-b.vercel.app/> | [frontend-bibou172](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/frontends/frontend-bibou172) |
| Brina Achouke (`@Brina21694`) | Portfolio ACHOUKE Brina — Data Analytics & Business Intelligence (React + Vite) | <https://brina-portfolio-tau.vercel.app/> | [frontend-Brina21694](https://github.com/nosql-esigelec/2026-2027-rouen-portfolio-blackhat/tree/main/frontends/frontend-Brina21694) |
