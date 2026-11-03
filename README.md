# TP 4 : Développement logiciel augmenté par des agents IA + SecureAgent

**Auteur** : Badr Eddine El Abbady, Génie Informatique, FST Settat (Université Hassan 1er)

Un système **multi-agents simulé** qui génère un projet Spring Boot (gestionnaire de tâches), puis une extension de recherche, **SecureAgent**, qui contrôle ce que chaque agent a le droit de faire grâce à des **jetons de capacité signés et délégables**.

> Tout le projet tient dans un seul notebook : `TP4_EL_ABBADY_Badr_Eddine_SecureAgent.ipynb`.
> Les agents sont **simulés** (règles et gabarits, **aucun appel à un LLM**). Le projet sécurise le *canal d'action* des agents, pas leur raisonnement.

---

## 1. Problématique

1. **Partie TP (parties 1 à 12)** : une équipe d'agents spécialisés produit-elle un meilleur résultat qu'un agent unique, et à quel coût ?
2. **Partie SecureAgent (partie 13)** : comment garantir qu'un agent autonome ne puisse faire que les actions explicitement autorisées, **même lorsqu'il délègue une partie de sa mission à un autre agent** ?

---

## 2. Architecture

```text
                    ┌──────────────────────┐
   REQUEST  ──────▶ │     Orchestrateur    │  (coordination, aucun outil)
                    └──────────┬───────────┘
        ┌──────────┬───────────┼───────────┬───────────┬───────────┬──────────────┐
  requirements  architecture   code       test       review     devops     documentation
        └──────────┴───────────┴─────┬─────┴───────────┴───────────┴──────────────┘
                                     ▼
                   call_tool(agent, tool, **args)      ◀── point de passage unique
                                     │
              ┌──────────────────────┴───────────────────────┐
              │ TP : check_permission + check_write_scope    │  (droits statiques)
              │ Partie 13 : AuthorizationEngine (jetons)     │  (capacités signées)
              └──────────────────────┬───────────────────────┘
                                     ▼
        read_file · list_files · write_file · run_tests · run_maven  (workspace/)
              + validation humaine (ask_human) + journal d'audit (LOG)
```

Principes du TP : **moindre privilège** par agent, **confinement des chemins** (anti path traversal), **validation humaine** pour les fichiers critiques (`pom.xml`, `Dockerfile`, `.github/`), **journal d'audit**, **boucle de correction** test → correction.

---

## 3. Contenu du notebook

| Partie | Contenu |
|:--:|---|
| 1 à 3 | Problème, rappels sur les agents, vocabulaire, moindre privilège, diagramme |
| 4 | Outils Python : confinement des chemins, `write_file`, tests simulés, `run_maven` |
| 5 à 10 | Requirements, Architecture, Code (+ squelette Spring Boot), Test, Review, DevOps |
| 11 | Orchestrateur multi-agents, audit du run, expérience « agent malveillant » |
| 12 | Comparaison : développement classique / agent unique / multi-agents |
| **13** | **SecureAgent : voir section 5** |
| Fin | « Pour aller plus loin » |

Les cellules « 🟧 À FAIRE » du TP sont à compléter par l'étudiant ; la partie 13 n'en contient pas et ne modifie aucune cellule fournie.

---

## 4. Installation et exécution

**Prérequis**

- Python 3 et Jupyter (Notebook, JupyterLab ou VS Code).
- Optionnel : `matplotlib` (graphiques de 13.7) et `biscuit-python` (configuration K, vrai Biscuit).
- Maven et Java ne sont **pas** nécessaires : `run_maven` fonctionne en mode simulé.

```bash
pip install jupyter matplotlib
pip install biscuit-python      # optionnel ; sans lui, la config K est ignorée proprement
jupyter notebook TP4_EL_ABBADY_Badr_Eddine_SecureAgent.ipynb
```

Puis **Exécuter toutes les cellules** (`Run All`). Testé avec Python 3.12 sous Linux ; sur une autre version de Python ou sous Windows, la disponibilité de `biscuit-python` n'a pas été vérifiée.

**Réglages en tête de notebook**

| Variable | Valeur par défaut | Effet |
|---|:--:|---|
| `AUTO_APPROVE` | `True` | Les validations humaines sont acceptées automatiquement. Mettre `False` pour répondre soi-même |
| `INJECT_BUG` | `True` | Le Code Agent simulé commet 2 erreurs à la 1re tentative (`@Entity` et `@Valid` oubliés), ce qui permet d'observer la boucle de correction |

Le notebook crée localement `workspace/` (projet généré) et `results/` (résultats SecureAgent).

---

## 5. SecureAgent : autorisation par capacités (partie 13)

### Ce qui est implémenté

- **Modèle de menaces** : 7 menaces (élévation de privilèges, accès non autorisé, falsification, expiration, violation de périmètre, délégation non autorisée, vol et rejeu), chacune reliée à un scénario de test.
- **Jeton de capacité atténuable** : un identifiant, une liste de restrictions (caveats) et une signature HMAC-SHA256 **chaînée**. On peut **ajouter** une restriction sans aucun secret, jamais en retirer ni en modifier une.
- **Authorization Engine** en 8 étapes : format, signature, révocation, expiration, identité, droit sur l'action, périmètre de la ressource, décision enregistrée. Le premier refus renvoie un code (`BAD_SIGNATURE`, `EXPIRED`, `WRONG_HOLDER`, `ACTION_NOT_GRANTED`, `SCOPE_VIOLATION`, `BAD_DELEGATION`…).
- **Délégation** : orchestrateur → code_agent → entity_helper, et orchestrateur → test_agent. Chaque délégation porte une preuve signée avec la clé personnelle du délégant.
- **Intégration** au workflow du TP sans réécrire les agents (`capability_mode`), avec les contrôles historiques conservés (défense en profondeur).
- **Domaine autonome** : le même moteur, sans modification, sur des actions abstraites (`get_status`, `read_resource`, `request_operation`) avec validation humaine pour les opérations sensibles.

### Protection contre le piège de la normalisation

`src/test/../main/A.java` commence par `src/test/` mais désigne `src/main/`. Le moteur normalise chaque chemin **avant** toute comparaison de préfixe.

### Les 4 configurations comparées

| Config | Mécanisme | Délégation |
|:--:|---|:--:|
| S | Permissions statiques du TP, sans jeton | non |
| H | Jeton plat JWT / HS256 | non |
| D | Jeton atténuable à chaîne HMAC (cette partie) | oui |
| K | Biscuit réel (`biscuit-python`), optionnel | oui |

### Résultats de l'exécution de référence

15 scénarios, même politique pour toutes les configurations ; « — » signifie que le scénario n'a pas de sens pour la configuration.

| Config | Accès légitimes réussis | Attaques bloquées |
|:--:|:--:|:--:|
| S | 3/3 | 3/4 |
| H | 3/3 | 8/8 |
| D | 4/4 | 11/11 |
| K | 4/4 | 8/10 |

- **S** laisse passer `src/test/../main/…` au niveau de la décision ; dans le TP, `write_file` revérifie le chemin résolu et l'arrête.
- **K (Biscuit natif)** est un jeton *porteur* : il laisse passer le rejeu d'un jeton volé et la fausse délégation. **D** les bloque grâce à la liaison au porteur et aux preuves de délégation signées. C'est un choix de conception, pas une supériorité générale sur Biscuit, qui offre un langage de politiques bien plus expressif.
- **Test de propriété** : 9000 comparaisons parent/enfant sur des chaînes de délégation aléatoires, 0 élargissement de droits. Ce n'est **pas une preuve** : cela réduit la probabilité d'un bug sans l'éliminer.
- **Coût** (médiane, une machine donnée, les valeurs varient) : environ 0,6 µs (S), 9 µs (H), 29 µs (D), 81 µs (K) ; D croît avec la longueur de la chaîne de délégation (environ 22 µs à profondeur 0, 155 µs à profondeur 8).

Les résultats sont exportés dans `results/secureagent_results.json` et `results/secureagent_results.png`.

---

## 6. Limites et menaces résiduelles

1. **Agents simulés** : aucun LLM, donc aucune mesure face à de vraies injections de prompt.
2. **Authentification de l'appelant** : le moteur croit l'argument `agent` fourni par l'orchestrateur. Sans authentification réelle, un jeton volé reste dangereux si l'attaquant contrôle cet argument.
3. **Clé racine symétrique** : celui qui vérifie peut aussi émettre. Une version asymétrique (Ed25519, comme Biscuit) sépare les deux.
4. **Révocation grossière** : on révoque un jeton entier et sa descendance, pas un droit isolé.
5. **Scénarios écrits par l'auteur des défenses** : biais de confirmation probable ; un évaluateur extérieur trouverait sans doute d'autres cas.
6. **Temps simulé** : l'expiration est testée avec une horloge injectée.
7. **Pas de domaine physique** : aucune intégration PX4, Gazebo ou ROS 2 ; la partie 13.8 n'utilise que des noms d'actions abstraits.

**Suites logiques** : signatures asymétriques, liaison au porteur par preuve de possession, politiques Datalog Biscuit (horaires, quotas), vrai LLM dans un agent pour tester des injections réelles, simulateur logiciel pour les actions autonomes.

---

## 7. Organisation conseillée du dépôt GitHub

Le projet actuel est un notebook unique. Si tu le publies, une organisation possible :

```text
secureagent/
├── README.md
├── notebooks/TP4_EL_ABBADY_Badr_Eddine_SecureAgent.ipynb
├── docs/            Threat Model, choix de conception (extraits de 13.1 et 13.2)
├── results/         secureagent_results.json / .png
└── requirements.txt jupyter, matplotlib, biscuit-python (optionnel)
```

Séparer le code du moteur (`src/`), les tests (`tests/`) et les expériences (`experiments/`) en fichiers Python est une étape supplémentaire, **non réalisée ici**.

---

## 8. Références conceptuelles

- Capabilities, atténuation et chaînes HMAC : principe des **Macaroons**.
- **Biscuit** : jetons d'autorisation à signatures chaînées et politiques Datalog (`biscuit-python`).
- **JWT / HS256** (RFC 7519) : un format de jeton, pas un modèle de capacités.
- Principe de **moindre privilège**.
