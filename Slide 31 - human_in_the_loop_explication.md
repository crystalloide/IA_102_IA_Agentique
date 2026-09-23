# Human-in-the-Loop : supervision et validation humaine avec LangGraph

Ce document explique, ligne par ligne, un exemple où un agent LangGraph doit **obtenir l'accord d'un opérateur humain** avant d'envoyer un e-mail. Le mécanisme repose sur deux éléments : `interrupt` (mettre le graphe en pause) et `Command(resume=...)` (le relancer avec la réponse de l'humain).

---

## Le code étudié

```python
from langgraph.types import interrupt, Command

@tool
def envoyer_email(dest: str, texte: str) -> str:
    """Envoie un e-mail (validation humaine requise)."""
    rep = interrupt({"action": "email", "dest": dest,
                     "texte": texte})
    if rep["ok"]:
        return envoyer(dest, rep.get("texte", texte))
    return "Envoi refusé par l'opérateur"

# reprise côté application
graph.invoke(Command(resume={"ok": True}), cfg)
```

---

## Explication ligne par ligne

### `from langgraph.types import interrupt, Command`

Importe les deux primitives du Human-in-the-Loop :

| Élément | Rôle |
|---|---|
| `interrupt` | Fonction appelée **à l'intérieur d'un nœud ou d'un outil**. Elle suspend l'exécution du graphe et transmet une valeur à l'application appelante. |
| `Command` | Objet passé à `graph.invoke(...)` **par l'application** pour relancer le graphe. Son paramètre `resume` contient la réponse de l'humain. |

> Le décorateur `@tool` n'est pas importé dans l'extrait. Il faut ajouter `from langchain_core.tools import tool`.

---

### `@tool`

Transforme la fonction Python en **outil** que le LLM peut décider d'appeler. LangChain lit :

- le **nom** de la fonction (`envoyer_email`) : le nom de l'outil ;
- les **annotations de type** (`dest: str`, `texte: str`) : le schéma des arguments que le LLM doit remplir ;
- la **docstring** : la description de l'outil présentée au modèle.

---

### `def envoyer_email(dest: str, texte: str) -> str:`

Signature de l'outil :

- `dest` : l'adresse du destinataire, fournie par le LLM ;
- `texte` : le corps du message, rédigé par le LLM ;
- `-> str` : l'outil renvoie une chaîne, qui deviendra un `ToolMessage` réinjecté dans la conversation de l'agent.

---

### `"""Envoie un e-mail (validation humaine requise)."""`

Docstring servant de description à l'outil. La mention « validation humaine requise » est purement informative pour le LLM : **ce n'est pas elle qui déclenche la validation**, c'est l'appel à `interrupt` juste en dessous.

---

### `rep = interrupt({"action": "email", "dest": dest, "texte": texte})`

C'est la ligne centrale du mécanisme. Elle se comporte différemment selon le moment de l'exécution.

**1er passage (avant validation)**

1. `interrupt` lève une exception interne (`GraphInterrupt`) qui **arrête immédiatement** l'exécution du nœud ; la suite de la fonction n'est pas exécutée.
2. LangGraph sauvegarde l'état du graphe grâce au **checkpointer**.
3. Le dictionnaire passé en argument est remonté à l'application. Avec `graph.invoke(...)`, il apparaît dans la clé `__interrupt__` du résultat :

   ```python
   {"__interrupt__": [Interrupt(value={"action": "email",
                                       "dest": "client@exemple.fr",
                                       "texte": "Bonjour ..."})]}
   ```

   L'application peut alors afficher à l'opérateur **ce que l'agent s'apprête à faire** : quelle action, à qui, avec quel contenu.

**2e passage (après validation)**

Quand l'application relance le graphe avec `Command(resume=...)`, le nœud **est ré-exécuté depuis son début**. Cette fois, `interrupt(...)` ne suspend plus rien : il **renvoie directement la valeur de `resume`**. Ici, `rep` vaut donc `{"ok": True}`.

> **Point d'attention** : puisque le nœud repart du début, tout code placé **avant** `interrupt` est exécuté deux fois. Il ne faut donc jamais y mettre d'effet de bord (écriture en base, appel API, envoi…). Dans cet exemple, rien ne précède `interrupt` : le code est sûr.

---

### `if rep["ok"]:`

Teste la décision de l'opérateur. Le format `{"ok": ...}` n'est **pas imposé par LangGraph** : c'est une convention choisie par le développeur. L'application et l'outil doivent simplement s'accorder sur la structure de la réponse.

---

### `return envoyer(dest, rep.get("texte", texte))`

Si l'opérateur a validé, l'e-mail est réellement envoyé :

- `envoyer(...)` est une fonction utilitaire **supposée exister ailleurs** (client SMTP, API d'envoi…) ; elle n'est pas définie dans l'extrait.
- `rep.get("texte", texte)` permet à l'opérateur de **corriger le message** avant envoi :
  - si la réponse contient une clé `"texte"`, c'est la version modifiée par l'humain qui part ;
  - sinon, on garde le texte d'origine rédigé par le LLM.

On a donc trois décisions possibles côté humain : **approuver**, **approuver en modifiant**, ou **refuser**.

---

### `return "Envoi refusé par l'opérateur"`

Si `rep["ok"]` est faux, aucun e-mail n'est envoyé. Le message renvoyé devient le résultat de l'outil et est transmis au LLM, qui **sait ainsi que son action a été refusée** et peut adapter la suite (reformuler, demander des précisions, abandonner…).

---

### `graph.invoke(Command(resume={"ok": True}), cfg)`

Code exécuté **côté application**, une fois l'opérateur ayant pris sa décision :

- `Command(resume={"ok": True})` : au lieu de fournir une nouvelle entrée utilisateur, on transmet la réponse humaine qui deviendra la valeur de retour de `interrupt`.
- `{"ok": True}` : l'opérateur approuve sans modifier le texte. Variantes possibles :
  - `{"ok": True, "texte": "Version corrigée..."}` : approuver avec modification ;
  - `{"ok": False}` : refuser.
- `cfg` : la configuration qui identifie **la conversation à reprendre**, typiquement :

  ```python
  cfg = {"configurable": {"thread_id": "session-42"}}
  ```

  Il doit s'agir du **même `thread_id`** que lors de l'appel initial, sinon LangGraph ne retrouve pas l'état sauvegardé.

---

## Prérequis indispensables

Pour que `interrupt` fonctionne, le graphe doit être compilé avec un **checkpointer**, sinon il est impossible de mettre en pause puis de reprendre :

```python
from langgraph.checkpoint.memory import InMemorySaver

graph = builder.compile(checkpointer=InMemorySaver())
```

`InMemorySaver` convient pour les tests ; en production, on utilise un checkpointer persistant (PostgreSQL, SQLite…) pour que la pause survive à un redémarrage de l'application, l'humain pouvant répondre plusieurs heures plus tard.

---

## Déroulé complet

```
1. Utilisateur  ──► graph.invoke({"messages": [...]}, cfg)
2. LLM          ──► décide d'appeler envoyer_email(dest, texte)
3. Outil        ──► interrupt({...})  → PAUSE, état sauvegardé
4. Application  ◄── reçoit result["__interrupt__"], l'affiche à l'opérateur
5. Opérateur    ──► valide / modifie / refuse
6. Application  ──► graph.invoke(Command(resume={...}), cfg)
7. Outil        ──► ré-exécuté : interrupt(...) renvoie {...}
8. Outil        ──► envoie l'e-mail ou renvoie le refus
9. LLM          ──► reçoit le résultat et poursuit la conversation
```

---

## À retenir

- `interrupt` **suspend** le graphe et expose une proposition d'action à l'humain.
- `Command(resume=...)` **relance** le graphe ; la valeur de `resume` devient le retour de `interrupt`.
- Le nœud est **rejoué depuis le début** à la reprise : pas d'effet de bord avant `interrupt`.
- Un **checkpointer** et un **`thread_id` identique** sont obligatoires.
- Le format de la réponse (`{"ok": ..., "texte": ...}`) est une **convention libre** entre l'application et l'outil.
- Ce schéma s'applique à toute action sensible : paiement, suppression de données, publication, commande…
