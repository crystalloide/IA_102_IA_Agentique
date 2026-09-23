# Mémoire long terme avec le Store LangGraph — explication ligne à ligne

## Le code étudié

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()   # PostgresStore en production
ns = ("utilisateurs", "u42", "preferences")

store.put(ns, "langue", {"valeur": "français"})
store.put(ns, "format", {"valeur": "réponses courtes"})

items = store.search(ns)          # ou search(ns, query=...)
prefs = {i.key: i.value["valeur"] for i in items}

graph = builder.compile(checkpointer=checkpointer,
                        store=store)
```

## Contexte : deux mémoires différentes

LangGraph distingue deux niveaux de mémoire, et ce snippet montre comment les combiner :

| | Checkpointer | Store |
|---|---|---|
| Portée | **Un fil de conversation** (`thread_id`) | **Transversale** à tous les fils |
| Contenu | L'état complet du graphe à chaque étape | Des documents JSON clé/valeur, choisis par le développeur |
| Rôle | Mémoire **court terme** : reprendre une conversation, rejouer, interrompre | Mémoire **long terme** : préférences, faits appris sur l'utilisateur, connaissances |
| Exemple | L'historique des messages de la conversation n° 7 | « Stéphane préfère des réponses en français » |

Une préférence enregistrée dans le Store reste disponible quand l'utilisateur ouvre une **nouvelle** conversation — ce que le checkpointer seul ne permet pas.

---

## Explication ligne à ligne

### 1. Import

```python
from langgraph.store.memory import InMemoryStore
```

Importe l'implémentation **en mémoire vive** du Store. Toutes les implémentations de Store (mémoire, PostgreSQL, Redis…) héritent de la même interface `BaseStore` : le reste du code ne change pas quand on change de backend.

### 2. Création du store

```python
store = InMemoryStore()   # PostgresStore en production
```

- Instancie un store vide, stocké dans un dictionnaire Python.
- **Limite** : tout est perdu à l'arrêt du processus, et rien n'est partagé entre plusieurs processus ou serveurs. Idéal pour le développement, les tests et les labs.
- **En production**, on utilise un store persistant, par exemple `PostgresStore` (package `langgraph-checkpoint-postgres`), sans oublier d'appeler `store.setup()` une première fois pour créer les tables :

```python
from langgraph.store.postgres import PostgresStore

with PostgresStore.from_conn_string("postgresql://...") as store:
    store.setup()   # création des tables, à faire une fois
    ...
```

### 3. Définition de l'espace de noms (namespace)

```python
ns = ("utilisateurs", "u42", "preferences")
```

- Un **namespace** est un **tuple de chaînes**, qui fonctionne comme un chemin de dossiers : `utilisateurs/u42/preferences`.
- Il sert à **isoler** les données : les préférences de l'utilisateur `u42` ne se mélangent pas avec celles de `u43`, ni avec d'autres catégories (`("utilisateurs", "u42", "faits")`, par exemple).
- Il est **hiérarchique** : on peut ensuite rechercher sur un préfixe (`("utilisateurs", "u42")` couvre tous les sous-espaces de cet utilisateur).
- Ici `"u42"` est codé en dur pour l'exemple ; dans une vraie application, l'identifiant vient de la configuration ou du contexte d'exécution (voir plus bas).

### 4. et 5. Écriture de deux préférences

```python
store.put(ns, "langue", {"valeur": "français"})
store.put(ns, "format", {"valeur": "réponses courtes"})
```

Signature : `store.put(namespace, key, value)`.

| Argument | Valeur ici | Rôle |
|---|---|---|
| `namespace` | `ns` | Où ranger l'élément |
| `key` | `"langue"`, `"format"` | Identifiant unique **dans ce namespace** |
| `value` | `{"valeur": ...}` | Un **dictionnaire** (sérialisable en JSON) — obligatoirement un `dict`, pas une simple chaîne |

Comportement important :
- `put` est un **upsert** : si la clé existe déjà dans ce namespace, la valeur est **remplacée** (et `updated_at` mis à jour).
- Pour supprimer : `store.delete(ns, "langue")`.
- Pour lire une seule clé connue : `store.get(ns, "langue")` renvoie un `Item` (ou `None`).

### 6. Recherche

```python
items = store.search(ns)          # ou search(ns, query=...)
```

- `search` renvoie la liste des éléments dont le namespace **commence par** le préfixe fourni. Ici, on récupère les deux préférences.
- Chaque résultat est un objet `SearchItem` avec notamment : `namespace`, `key`, `value`, `created_at`, `updated_at`, et `score` (pertinence, en recherche sémantique).
- Paramètres utiles :
  - `filter={"valeur": "français"}` : filtre exact sur le contenu de `value` ;
  - `limit` / `offset` : pagination. ⚠️ **`limit` vaut 10 par défaut** : au-delà de 10 préférences, certaines seraient silencieusement ignorées si on ne l'augmente pas ;
  - `query="..."` : **recherche sémantique** en langage naturel. Elle ne fonctionne que si le store a été créé avec un index d'embeddings, sinon elle n'apporte pas de classement par pertinence :

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536}
)
items = store.search(ns, query="comment l'utilisateur veut-il qu'on lui réponde ?")
```

### 7. Transformation en dictionnaire simple

```python
prefs = {i.key: i.value["valeur"] for i in items}
```

Compréhension de dictionnaire qui, pour chaque élément trouvé :
- prend sa **clé** (`i.key`) comme clé du dictionnaire ;
- extrait le champ `"valeur"` de son contenu (`i.value["valeur"]`).

Résultat :

```python
{"langue": "français", "format": "réponses courtes"}
```

Ce format compact est pratique pour l'injecter dans un prompt système. Point de vigilance : si un élément du namespace n'a pas de champ `"valeur"`, on obtient une `KeyError` ; `i.value.get("valeur")` est plus robuste.

### 8. et 9. Compilation du graphe avec les deux mémoires

```python
graph = builder.compile(checkpointer=checkpointer,
                        store=store)
```

- `builder` est un `StateGraph` défini auparavant (nœuds et arêtes), et `checkpointer` un saver défini auparavant (`InMemorySaver`, `PostgresSaver`…). Ils ne sont pas créés dans ce snippet.
- `compile()` transforme la définition en graphe exécutable.
- `checkpointer=` active la **mémoire court terme** (état par `thread_id`).
- `store=` rend le store **accessible depuis chaque nœud** du graphe : c'est ce qui permet aux nœuds de lire et d'écrire la mémoire long terme pendant l'exécution.

---

## Pour aller plus loin : utiliser le store dans un nœud

Dans le snippet, les lectures/écritures se font **hors** du graphe. En pratique, on les fait **dans les nœuds**, en récupérant l'identifiant utilisateur depuis la configuration plutôt que de le coder en dur :

```python
from langgraph.store.base import BaseStore
from langchain_core.runnables import RunnableConfig

def repondre(state, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    ns = ("utilisateurs", user_id, "preferences")
    prefs = {i.key: i.value["valeur"] for i in store.search(ns)}

    systeme = f"Réponds en {prefs.get('langue', 'français')}, format : {prefs.get('format', 'libre')}."
    ...

# Appel : même utilisateur, nouvelle conversation → les préférences sont retrouvées
graph.invoke(
    {"messages": [...]},
    {"configurable": {"thread_id": "conv-2", "user_id": "u42"}},
)
```

LangGraph **injecte automatiquement** le store passé à `compile()` dans le paramètre `store` du nœud. Selon la version, il est aussi accessible via `runtime.store` (paramètre `runtime: Runtime`) ou `get_store()`.

## À retenir

- **Checkpointer** = mémoire d'**une** conversation ; **Store** = mémoire **entre** conversations.
- Les données sont rangées par **namespace** (tuple hiérarchique) + **clé**, avec une **valeur `dict`**.
- `put` écrit ou remplace, `get` lit une clé, `search` liste par préfixe (10 résultats max par défaut), `delete` supprime.
- La recherche sémantique (`query=`) exige un **index d'embeddings** configuré sur le store.
- `InMemoryStore` pour développer, un store persistant (`PostgresStore`…) en production.
