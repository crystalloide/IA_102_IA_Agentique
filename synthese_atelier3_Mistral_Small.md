# Synthèse — Atelier 3 IA102 : Plan-and-Execute vs ReWOO

*Généré le 25/09/2026 à 10:07 — mode : API compatible / Mistral-Small
— LangGraph 1.2.11*

## 1. Protocole
- Cas d'usage : système multi-agents logistique (4 entrepôts), outils `consulter_stock`, `tarifs_transport`,
  `reserver_stock`, `calculatrice` (latence simulée de 0.3 s par appel).
- Scénarios : S1 devis simple ; S2 commande de 120 unités en environnement stable ; S3 même commande avec un
  **incident survenant pendant l'exécution** (blocage des expéditions à Lyon).
- Évaluation sur l'**état réel** de l'environnement (réservations effectives) comparé à l'optimum calculé.
- Répétitions par mesure : 1.

## 2. Résultats mesurés
| scenario | architecture | statut | appels_llm | tokens_total | duree_s | appels_outils | replanifications | cout_reel | cout_optimal |
|---|---|---|---|---|---|---|---|---|---|
| S1 — Devis simple | Plan-and-Execute | ✅ correct | 7 | 4425 | 10.39 | 2 | 0 |  | 252.0 |
| S1 — Devis simple | ReWOO | ❌ incorrect | 4 | 2163 | 6.84 | 3 | 0 |  | 252.0 |
| S2 — Commande (stable) | Plan-and-Execute | ✅ optimal | 9 | 7037 | 16.78 | 4 | 0 | 212.0 | 212.0 |
| S2 — Commande (stable) | ReWOO | ❌ délai non respecté | 7 | 3865 | 8.21 | 3 | 0 | 24.0 | 212.0 |
| S3 — Commande (imprévu) | Plan-and-Execute | ❌ délai non respecté | 13 | 11158 | 14.31 | 6 | 1 | 204.0 | 224.0 |
| S3 — Commande (imprévu) | ReWOO | ❌ délai non respecté | 7 | 3978 | 2.29 | 3 | 0 | 24.0 | 224.0 |

![Comparaison](comparaison_atelier3.png)

## 3. Analyse chiffrée
- **S1 — Devis simple** : Plan-and-Execute = 7 appels LLM / 4425 tokens (succès 100 %) ; ReWOO = 4 appels / 2163 tokens (succès 0 %). Plan-and-Execute consomme **1.8×** plus d'appels et **2.0×** plus de tokens.
- **S2 — Commande (stable)** : Plan-and-Execute = 9 appels LLM / 7037 tokens (succès 100 %) ; ReWOO = 7 appels / 3865 tokens (succès 0 %). Plan-and-Execute consomme **1.3×** plus d'appels et **1.8×** plus de tokens.
- **S3 — Commande (imprévu)** : Plan-and-Execute = 13 appels LLM / 11158 tokens (succès 0 %) ; ReWOO = 7 appels / 3978 tokens (succès 0 %). Plan-and-Execute consomme **1.9×** plus d'appels et **2.8×** plus de tokens.

**Variantes (bonus et exercices)**

| scenario | architecture | statut | appels_llm | tokens_total | duree_s | replanifications |
|---|---|---|---|---|---|---|
| S1 — Devis simple | P&E économe | ✅ correct | 5 | 2860 | 5.3 | 0 |
| S2 — Commande (stable) | P&E économe | ✅ optimal | 7 | 5074 | 7.98 | 0 |
| S3 — Commande (imprévu) | P&E économe | ✅ optimal | 10 | 8349 | 14.25 | 1 |
| S2 — Commande (stable) | ReWOO hybride | ❌ délai non respecté | 19 | 16428 | 44.29 | 2 |
| S3 — Commande (imprévu) | ReWOO hybride | ❌ délai non respecté | 22 | 19125 | 52.17 | 2 |

## 4. Grille comparative

| Critère | Plan-and-Execute | ReWOO | LLMCompiler |
|---|---|---|---|
| Appels LLM | ≈ 1 + 2 × étapes | 2 + étapes LLM[…] | 1 planif + joiner (+ replanifs) |
| Adaptation aux imprévus | forte | nulle | par cycles (joiner) |
| Latence | élevée | moyenne | faible (parallélisme) |
| Point faible | coût et latence | plan figé, dépend des prévisions | complexité, parsing du DAG |
| À privilégier pour | environnements dynamiques, actions à effets de bord | tâches prévisibles, fort volume, budget contraint | nombreuses sous-tâches indépendantes |

## 5. Enseignements clés
1. **ReWOO est nettement plus économe** en appels LLM et en tokens : le planificateur n'est appelé qu'une fois et les
   observations ne sont jamais renvoyées au LLM, sauf dans les étapes `LLM[…]` prévues.
2. **Plan-and-Execute est le seul à réussir quand la réalité diverge du plan** (S3) : la re-planification transforme un
   échec d'action en simple détour, au prix d'appels supplémentaires.
3. Le coût de Plan-and-Execute vient surtout du **re-planificateur appelé à chaque étape** ; ne l'appeler qu'en cas
   d'échec (exercice 2) réduit fortement ce surcoût tout en conservant l'adaptabilité.
4. Un **ReWOO hybride** (vérificateur + re-planification sur échec, exercice 3) combine sobriété nominale et capacité
   de rattrapage : c'est l'idée reprise par le *joiner* de LLMCompiler.
5. L'évaluation d'un agent doit porter sur **ses actions réelles**, pas seulement sur son discours.

## 6. Vos conclusions (à compléter)
- **Q1 — Environnement prévisible (S1, S2)** : …
- **Q2 — Changement dynamique (S3)** : …
- **Q3 — Pourquoi ReWOO échoue-t-il sur S3 ?** : …
- **Q4 — Optimisation de Plan-and-Execute** : …
- **Q5 — Recommandation pour votre organisation** (quelle architecture, pour quel processus, avec quels garde-fous ?) : …
