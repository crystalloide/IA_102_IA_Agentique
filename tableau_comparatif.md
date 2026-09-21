# Tableau comparatif — Réseau (handoffs) vs Superviseur (sous-agents outils)

Fournisseur : `simulation` — modèle : `simulateur-ia102` — 3 requêtes de test

## Synthèse (3 requêtes)

| Critère                                | Réseau   | Superviseur   | Écart Sup. vs Rés.   | Meilleur    |
|:---------------------------------------|:---------|:--------------|:---------------------|:------------|
| Étapes de la trajectoire — total       | 13       | 29            | +123%                | Réseau      |
| Passages de main / délégations — total | 13       | 13            | +0%                  | =           |
| Appels LLM — total                     | 18       | 34            | +89%                 | Réseau      |
| Tokens d'entrée — total                | 29,101   | 36,477        | +25%                 | Réseau      |
| Tokens de sortie — total               | 3,664    | 3,906         | +7%                  | Réseau      |
| Tokens total — total                   | 32,765   | 40,383        | +23%                 | Réseau      |
| Appels d'outils de recherche — total   | 5        | 5             | +0%                  | =           |
| Durée (s) — total                      | 0.0      | 0.0           | —                    | =           |
| Couverture des points clés — moyenne   | 80%      | 80%           | +0%                  | =           |
| Sources citées (nb) — moyenne          | 2.7      | 3.0           | +13%                 | Superviseur |
| Note du LLM-juge (/5) — moyenne        | 4.00     | 4.00          | +0%                  | =           |

## Trajectoires

| Requête   | Réseau (handoffs)                                               | Superviseur                                                                                                                                         |
|:----------|:----------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------|
| R1        | chercheur → redacteur → relecteur → FIN                         | superviseur → chercheur → superviseur → redacteur → superviseur → relecteur → superviseur → FIN                                                     |
| R2        | chercheur → redacteur → relecteur → redacteur → relecteur → FIN | superviseur → chercheur → superviseur → redacteur → superviseur → relecteur → superviseur → redacteur → superviseur → relecteur → superviseur → FIN |
| R3        | chercheur → redacteur → relecteur → redacteur → relecteur → FIN | superviseur → chercheur → superviseur → redacteur → superviseur → relecteur → superviseur → redacteur → superviseur → relecteur → superviseur → FIN |

## Détail par requête

|                                                                                      | Réseau   | Superviseur   | Écart Sup. vs Rés.   | Meilleur    |
|:-------------------------------------------------------------------------------------|:---------|:--------------|:---------------------|:------------|
| ('R1 · Factuelle (1 fait, piège de version)', 'Étapes de la trajectoire')            | 3        | 7             | +133%                | Réseau      |
| ('R1 · Factuelle (1 fait, piège de version)', 'Passages de main / délégations')      | 3        | 3             | +0%                  | =           |
| ('R1 · Factuelle (1 fait, piège de version)', 'Appels LLM')                          | 4        | 8             | +100%                | Réseau      |
| ('R1 · Factuelle (1 fait, piège de version)', "Tokens d'entrée")                     | 4,129    | 5,996         | +45%                 | Réseau      |
| ('R1 · Factuelle (1 fait, piège de version)', 'Tokens de sortie')                    | 632      | 696           | +10%                 | Réseau      |
| ('R1 · Factuelle (1 fait, piège de version)', 'Tokens total')                        | 4,761    | 6,692         | +41%                 | Réseau      |
| ('R1 · Factuelle (1 fait, piège de version)', "Appels d'outils de recherche")        | 1        | 1             | +0%                  | =           |
| ('R1 · Factuelle (1 fait, piège de version)', 'Durée (s)')                           | 0.0      | 0.0           | —                    | =           |
| ('R1 · Factuelle (1 fait, piège de version)', 'Couverture des points clés')          | 100%     | 100%          | +0%                  | =           |
| ('R1 · Factuelle (1 fait, piège de version)', 'Sources citées (nb)')                 | 2.0      | 2.0           | +0%                  | =           |
| ('R1 · Factuelle (1 fait, piège de version)', 'Note du LLM-juge (/5)')               | 4.50     | 4.50          | +0%                  | =           |
| ('R2 · Synthèse multi-documents', 'Étapes de la trajectoire')                        | 5        | 11            | +120%                | Réseau      |
| ('R2 · Synthèse multi-documents', 'Passages de main / délégations')                  | 5        | 5             | +0%                  | =           |
| ('R2 · Synthèse multi-documents', 'Appels LLM')                                      | 7        | 13            | +86%                 | Réseau      |
| ('R2 · Synthèse multi-documents', "Tokens d'entrée")                                 | 12,256   | 15,207        | +24%                 | Réseau      |
| ('R2 · Synthèse multi-documents', 'Tokens de sortie')                                | 1,454    | 1,548         | +6%                  | Réseau      |
| ('R2 · Synthèse multi-documents', 'Tokens total')                                    | 13,710   | 16,755        | +22%                 | Réseau      |
| ('R2 · Synthèse multi-documents', "Appels d'outils de recherche")                    | 2        | 2             | +0%                  | =           |
| ('R2 · Synthèse multi-documents', 'Durée (s)')                                       | 0.0      | 0.0           | —                    | =           |
| ('R2 · Synthèse multi-documents', 'Couverture des points clés')                      | 60%      | 60%           | +0%                  | =           |
| ('R2 · Synthèse multi-documents', 'Sources citées (nb)')                             | 2.0      | 3.0           | +50%                 | Superviseur |
| ('R2 · Synthèse multi-documents', 'Note du LLM-juge (/5)')                           | 3.50     | 3.50          | +0%                  | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Étapes de la trajectoire')       | 5        | 11            | +120%                | Réseau      |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Passages de main / délégations') | 5        | 5             | +0%                  | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Appels LLM')                     | 7        | 13            | +86%                 | Réseau      |
| ('R3 · Aide à la décision (raisonnement chiffré)', "Tokens d'entrée")                | 12,716   | 15,274        | +20%                 | Réseau      |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Tokens de sortie')               | 1,578    | 1,662         | +5%                  | Réseau      |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Tokens total')                   | 14,294   | 16,936        | +18%                 | Réseau      |
| ('R3 · Aide à la décision (raisonnement chiffré)', "Appels d'outils de recherche")   | 2        | 2             | +0%                  | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Durée (s)')                      | 0.0      | 0.0           | —                    | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Couverture des points clés')     | 80%      | 80%           | +0%                  | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Sources citées (nb)')            | 4.0      | 4.0           | +0%                  | =           |
| ('R3 · Aide à la décision (raisonnement chiffré)', 'Note du LLM-juge (/5)')          | 4.00     | 4.00          | +0%                  | =           |
