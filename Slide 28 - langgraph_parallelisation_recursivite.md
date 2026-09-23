# LangGraph — Parallélisation et contrôle de la récursivité

Explication ligne à ligne de deux mécanismes de LangGraph :

1. la **parallélisation de type map-reduce** avec `Send` ;
2. le **contrôle de la récursivité** avec `recursion_limit` et `GraphRecursionError`.

---

## 1°) Parallélisation (map-reduce)

### Le code

```python
from langgraph.types import Send
import operator

class State(TypedDict):
    sujets: list[str]
    resultats: Annotated[list, operator.add]

def repartir(state):
    return [Send("worker", {"sujet": s})
            for s in state["sujets"]]

g.add_conditional_edges("planner", repartir,
                        ["worker"])
```

### Explication ligne à ligne

```python
from langgraph.types import Send
```
Importe la classe `Send`. Un objet `Send` est un **message d'envoi dynamique** : il indique à LangGraph « exécute tel nœud avec tel état d'entrée ». C'est l'outil qui permet de créer, à l'exécution, un nombre de branches parallèles que l'on ne connaît pas à l'avance (phase *map*).

```python
import operator
```
Importe le module standard `operator`, qui expose les opérateurs Python sous forme de fonctions. On l'utilise pour `operator.add`, c'est-à-dire l'opérateur `+` (qui, appliqué à deux listes, les concatène).

```python
class State(TypedDict):
```
Définit le **schéma de l'état global** du graphe sous forme de `TypedDict` : un dictionnaire dont les clés et les types sont déclarés. Chaque nœud lit cet état et renvoie une mise à jour partielle de celui-ci.

```python
    sujets: list[str]
```
Première clé de l'état : la liste des sujets à traiter (par exemple `["Python", "SQL", "Docker"]`). Elle n'a pas de *reducer* : si un nœud renvoie `sujets`, la nouvelle valeur **remplace** l'ancienne.

```python
    resultats: Annotated[list, operator.add]
```
Deuxième clé : la liste des résultats. L'annotation `Annotated[list, operator.add]` associe à cette clé un **reducer** : quand un nœud renvoie `{"resultats": [...]}`, LangGraph ne remplace pas la valeur existante, il fait `ancienne_liste + nouvelle_liste`.
C'est indispensable ici : plusieurs workers écrivent **en même temps** dans `resultats`. Sans reducer, LangGraph lèverait une erreur (`InvalidUpdateError`) car il ne saurait pas quelle mise à jour garder. Avec `operator.add`, toutes les contributions sont **concaténées** : c'est la phase *reduce*.

```python
def repartir(state):
```
Définit la **fonction de routage** (on l'appellera sur une arête conditionnelle). Elle reçoit l'état courant du graphe, tel qu'il est après l'exécution du nœud `planner`.

```python
    return [Send("worker", {"sujet": s})
            for s in state["sujets"]]
```
Compréhension de liste qui fabrique **un `Send` par sujet** :
- `"worker"` : nom du nœud cible ;
- `{"sujet": s}` : l'état d'entrée **propre à cette instance** du worker. Chaque worker reçoit uniquement ce petit dictionnaire, et non l'état global complet.

Avec 3 sujets, la fonction renvoie 3 `Send` : LangGraph lance alors **3 exécutions du nœud `worker` en parallèle**, dans la même étape (*super-step*).

```python
g.add_conditional_edges("planner", repartir,
                        ["worker"])
```
Ajoute au constructeur de graphe `g` (un `StateGraph`) une **arête conditionnelle** partant du nœud `planner` :
- `"planner"` : nœud source ; l'arête est évaluée après son exécution ;
- `repartir` : la fonction qui décide où aller ; comme elle renvoie une liste de `Send`, elle déclenche un *fan-out* ;
- `["worker"]` : la liste des **destinations possibles** (*path map*). Elle ne modifie pas le comportement mais permet à LangGraph de connaître la structure du graphe (validation, visualisation avec `draw_mermaid()`).

### Déroulé à l'exécution

```
            ┌──► worker(sujet="Python") ──┐
planner ────┼──► worker(sujet="SQL")    ──┼──► resultats = [r1] + [r2] + [r3]
            └──► worker(sujet="Docker") ──┘
   (map : Send)        (en parallèle)          (reduce : operator.add)
```

> ⚠️ L'ordre des éléments dans `resultats` n'est pas garanti comme reflétant l'ordre de `sujets` : si l'ordre compte, faites renvoyer au worker le sujet avec son résultat.

### Ce que l'extrait ne montre pas

L'extrait suppose plusieurs éléments définis ailleurs :
- les imports `from typing import TypedDict, Annotated` et `from langgraph.graph import StateGraph, START, END` ;
- la création du graphe : `g = StateGraph(State)` ;
- les nœuds `planner` et `worker` ajoutés avec `g.add_node(...)` ;
- le worker doit renvoyer sa contribution **sous forme de liste** (`{"resultats": [...]}`) pour que `operator.add` puisse concaténer.

### Exemple complet exécutable

```python
import operator
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class State(TypedDict):
    sujets: list[str]
    resultats: Annotated[list, operator.add]

class WorkerState(TypedDict):
    sujet: str

def planner(state: State):
    # Ici on pourrait générer les sujets avec un LLM
    return {"sujets": ["Python", "SQL", "Docker"]}

def repartir(state: State):
    return [Send("worker", {"sujet": s}) for s in state["sujets"]]

def worker(state: WorkerState):
    # Chaque instance ne voit que son sujet
    return {"resultats": [f"Résumé de {state['sujet']}"]}

def synthese(state: State):
    print("Résultats agrégés :", state["resultats"])
    return {}

g = StateGraph(State)
g.add_node("planner", planner)
g.add_node("worker", worker)
g.add_node("synthese", synthese)

g.add_edge(START, "planner")
g.add_conditional_edges("planner", repartir, ["worker"])
g.add_edge("worker", "synthese")   # synthese attend la fin de tous les workers
g.add_edge("synthese", END)

graph = g.compile()
graph.invoke({"sujets": [], "resultats": []})
```

---

## 2°) Récursivité

### Le code

```python
from langgraph.errors import GraphRecursionError
try:
    graph.invoke(inputs, {"recursion_limit": 30})
except GraphRecursionError:
    ...  # sortie contrôlée
```

### Explication ligne à ligne

```python
from langgraph.errors import GraphRecursionError
```
Importe l'exception levée par LangGraph lorsqu'un graphe **dépasse le nombre maximal d'étapes autorisé** sans atteindre `END`. Elle protège contre les boucles infinies, fréquentes dans les graphes cycliques (agent qui rappelle un outil indéfiniment, boucle « générer → critiquer → corriger » qui ne converge jamais…).

```python
try:
```
Ouvre un bloc protégé : si l'exécution du graphe lève une exception gérée plus bas, le programme ne plante pas.

```python
    graph.invoke(inputs, {"recursion_limit": 30})
```
Exécute le graphe compilé :
- `inputs` : l'état initial (par exemple `{"messages": [...]}`) ;
- `{"recursion_limit": 30}` : le **second argument est la configuration d'exécution** (`config`). La clé `recursion_limit` fixe le nombre maximal de **super-steps** autorisés, ici 30.

Un *super-step* correspond à un tour d'exécution du graphe : tous les nœuds déclenchés en même temps comptent pour **une seule** étape. Ainsi, dans la partie 1, les 3 workers lancés par `Send` ne consomment qu'un super-step, pas trois.

Si la limite n'est pas précisée, LangGraph applique une valeur par défaut (historiquement 25). L'augmenter est utile pour des agents légitimement longs ; la baisser permet de borner le coût (appels LLM) et la durée.

```python
except GraphRecursionError:
```
Intercepte spécifiquement l'exception de dépassement de limite. Les autres erreurs (erreur d'un outil, d'API…) ne sont pas capturées ici et remontent normalement.

```python
    ...  # sortie contrôlée
```
`...` (*Ellipsis*) est un simple **emplacement réservé** : à remplacer par la logique de repli. Exemples de « sortie contrôlée » :
- journaliser l'incident (`logger.warning("Limite de récursion atteinte")`) ;
- renvoyer un message par défaut à l'utilisateur (« Je n'ai pas pu terminer la tâche ») ;
- récupérer le dernier état connu, si le graphe a été compilé avec un *checkpointer* et appelé avec un `thread_id` : `graph.get_state(config).values` ;
- relancer avec d'autres paramètres ou escalader vers un humain.

### Exemple de sortie contrôlée

```python
import logging
from langgraph.errors import GraphRecursionError

config = {"recursion_limit": 30, "configurable": {"thread_id": "session-42"}}

try:
    resultat = graph.invoke(inputs, config)
except GraphRecursionError:
    logging.warning("Limite de 30 étapes atteinte : arrêt de la boucle.")
    # Nécessite un checkpointer (ex. MemorySaver) passé à g.compile(checkpointer=...)
    resultat = graph.get_state(config).values
    resultat["statut"] = "interrompu"
```

### Pour aller plus loin : anticiper plutôt que subir

L'exception est un **filet de sécurité** externe. Pour que le graphe s'arrête proprement de lui-même avant la limite, LangGraph propose la valeur gérée `RemainingSteps` : on l'ajoute à l'état, et une arête conditionnelle peut router vers `END` (ou vers un nœud de synthèse) quand il reste peu d'étapes.

```python
from langgraph.managed import RemainingSteps

class State(TypedDict):
    messages: list
    remaining_steps: RemainingSteps   # rempli automatiquement par LangGraph

def continuer_ou_arreter(state: State):
    if state["remaining_steps"] <= 2:
        return "synthese"             # sortie anticipée et propre
    return "agent"
```

---

## Récapitulatif

| Élément | Rôle |
|---|---|
| `Send("worker", {...})` | Lance dynamiquement une instance d'un nœud avec son propre état (map) |
| Fonction de routage renvoyant une liste de `Send` | Crée un *fan-out* : N exécutions parallèles dans un même super-step |
| `Annotated[list, operator.add]` | Reducer : concatène les contributions parallèles au lieu de les écraser (reduce) |
| `add_conditional_edges(src, fn, [dest])` | Branche la fonction de routage après `src` et déclare les destinations possibles |
| `{"recursion_limit": N}` | Nombre maximal de super-steps avant arrêt forcé |
| `GraphRecursionError` | Exception levée au dépassement ; à capturer pour une sortie contrôlée |
| `RemainingSteps` | Permet au graphe d'anticiper la limite et de s'arrêter proprement |
