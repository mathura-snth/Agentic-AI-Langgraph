# Agentic AI avec LangGraph

Projet d'apprentissage progressif pour construire un **système multi-agents** avec [LangGraph](https://www.langchain.com/langgraph), en partant des bases (validation de données, graphes, outils) jusqu'à l'orchestration de plusieurs agents.

> **Objectif final : une architecture multi-agents simple** (*Simple MultiAI Agent Architecture*), combinant agents ReAct, outils externes et mémoire de conversation.

---

## Avancement

| # | Étape | Notebook | Statut |
|---|-------|----------|--------|
| 1 | Bases de Pydantic (validation de données) | `intro.ipynb` | Fait |
| 2 | Premier graphe LangGraph (State, nodes, edges conditionnels) | `simplegraph.ipynb` | Fait |
| 3 | Chatbot avec plusieurs outils (Arxiv, Wikipedia, Tavily) | `chatbotmultipletools.ipynb` | Fait |
| 4 | Agents ReAct prébuilts (`create_react_agent`) | – | À faire |
| 5 | Mémoire de conversation (`MemorySaver`) | – | À faire |
| 6 | Architecture multi-agents complète | – | À faire |

---

## Ce qui a été fait

### 1. Pydantic : validation de données (`intro.ipynb`)

Pydantic est la base de la validation des données dans l'écosystème LangChain / LangGraph. Le notebook compare d'abord une `dataclass` classique (aucune validation : `city=12` passe sans erreur) avec un `BaseModel` Pydantic (qui lève une `ValidationError`).

Notions couvertes :

- Modèles de base avec `BaseModel` et annotations de types
- Champs optionnels (`Optional`) et valeurs par défaut
- Champs de type liste (`List[str]`) et erreurs de validation sur les éléments
- Modèles imbriqués (`Customer` contenant une `Address`) et conversion automatique de types (`"75000"` → `75000`)
- Contraintes avec `Field` : `min_length`, `max_length`, `gt`, `le`, `ge`, `default`, `default_factory`, `description`
- Génération d'un schéma JSON avec `model_json_schema()`

### 2. Premier graphe LangGraph (`simplegraph.ipynb`)

Construction d'un workflow minimal pour comprendre les briques de LangGraph :

- **State** : schéma de l'état partagé, défini avec un `TypedDict` (`graph_info: str`)
- **Nodes** : fonctions `start_play`, `football`, `badminton` qui modifient l'état
- **Conditional edge** : la fonction `random_play` choisit aléatoirement entre `football` et `badminton`
- **Compilation** avec `StateGraph(...).compile()`, visualisation du graphe (Mermaid) et exécution avec `invoke`

```
START → start_play ──┬──> football  ──> END
                     └──> badminton ──> END
```

### 3. Chatbot avec plusieurs outils (`chatbotmultipletools.ipynb`)

Un chatbot capable de décider lui-même quand appeler un outil.

**Outils intégrés**

- `arxiv` : recherche d'articles scientifiques (`ArxivQueryRun`)
- `wikipedia` : recherche Wikipedia (`WikipediaQueryRun`)
- `tavily` : recherche web (`TavilySearchResults`, puis `langchain_tavily.TavilySearch`)

**LLM** : modèle hébergé sur Groq via `ChatGroq`, relié aux outils avec `bind_tools`.

**Graphe**

- État : `messages: Annotated[list[AnyMessage], add_messages]` (le *reducer* `add_messages` ajoute les messages à l'historique au lieu de l'écraser)
- Nodes : `tool_calling_llm` (le LLM) et `tools` (`ToolNode`)
- Routage avec `tools_condition` : si le LLM demande un outil → node `tools`, sinon → `END`

Deux versions du graphe ont été testées :

1. `tools → END` : l'outil est appelé, mais le résultat n'est pas réinterprété par le LLM
2. `tools → tool_calling_llm` : **boucle de type ReAct**. Le LLM peut enchaîner plusieurs appels d'outils et rédiger une réponse finale à partir des résultats

```
START → tool_calling_llm ──(tool call ?)──> tools ──┐
              ▲                                      │
              └──────────────────────────────────────┘
              │
              └──(pas d'outil)──> END
```

Exemples testés : recherche d'un article par son identifiant arXiv (`1706.03762`), conversation simple sans outil, et requête combinée (actualités IA + article récent sur le calcul quantique) déclenchant plusieurs outils en parallèle.

---

## Prochaines étapes : vers le multi-agents

D'après l'architecture visée, il reste à explorer :

- **`create_react_agent`** (`langgraph.prebuilt`) : créer des agents ReAct spécialisés sans reconstruire le graphe à la main
- **`MemorySaver`** (`langgraph.checkpoint.memory`) : donner une mémoire persistante aux conversations (checkpointing par `thread_id`)
- **Orchestration multi-agents** : plusieurs agents spécialisés (par ex. recherche web, recherche scientifique) coordonnés dans un même graphe

---

## Stack technique

- Python 3.9
- [Pydantic](https://docs.pydantic.dev/)
- [LangGraph](https://langchain-ai.github.io/langgraph/) et LangChain (`langchain_core`, `langchain_community`, `langchain_groq`, `langchain_tavily`)
- Groq (LLM) et Tavily (recherche web)
- Jupyter / VS Code

## Installation

```bash
python3 -m venv venv
source venv/bin/activate
pip install pydantic langgraph langchain-core langchain-community langchain-groq langchain-tavily \
            arxiv wikipedia python-dotenv ipykernel
```

Créer un fichier `.env` à la racine avec les clés d'API :

```env
GROQ_API_KEY=your_groq_key
TAVILY_API_KEY=your_tavily_key
```

> ⚠️ Ne jamais committer le fichier `.env` (l'ajouter au `.gitignore`).

## Notes et points d'attention

- `TavilySearchResults` est déprécié : privilégier `langchain_tavily.TavilySearch`.
- L'outil Wikipedia a renvoyé une erreur de décodage (`Expecting value: line 1 column 1`) lors du test ; à investiguer (réseau, version du package ou requête).
- Les notebooks ont été exécutés avec Python 3.9, un avertissement `urllib3`/LibreSSL apparaît sur macOS.