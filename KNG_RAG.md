# 👟 Marathon Shoe Picker — Knowledge Graph vs Traditional RAG

## 🧠 What did I Build

A Python + Neo4j app that:
1. **Stores shoe data** as a connected graph in Neo4j (nodes + relationships)
2. **Answers the question**: *"Which shoe should I buy for my marathon?"*
3. **Shows the difference** between a dumb keyword search (Traditional RAG) and a smart graph-aware search (Knowledge Graph RAG)

---

## 📁 Files as aprerequisite — Total: **6 files**

```
KNA_RAG/
├── 1. requirements.txt         ← Python packages to install
├── 2. .env                     ← Your Neo4j login secrets
├── 3. data/
│   └── shoe_catalogue.json     ← The shoe data (your "knowledge base")
├── 4. build_graph.py           ← Loads shoes into Neo4j as a graph
├── 5. query_rag.py             ← Traditional RAG: keyword search
└── 6. query_kg_rag.py          ← Knowledge Graph RAG: smart graph search
```


### STEP 1 — `requirements.txt`

- `neo4j` → talks to your graph database
- `python-dotenv` → reads your secret passwords from a file
- `openai` → (optional) use GPT to generate answers from results

---

### STEP 2 — `.env`
> *" Neo4j login — like a username/password sticky note"*

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password123
```

---

### STEP 3 — `data/shoe_catalogue.json`
> *"The actual shoe catalog — 5 marathon shoes with their features"*

Each shoe has:
- **specs**: brand, model, weight, stack height, drop, price
- **rated_for**: what terrain and distance it's good for
- **mitigates**: what injuries it helps prevent


### STEP 4 — `build_graph.py`
> *"Draws the map in Neo4j — creates all the dots (nodes) and lines (relationships)"*

What it does:
1. Opens Neo4j
2. Clears old data
3. For each shoe → creates a **Shoe node**
4. For each condition → creates a **Condition node** + links it with `RATED_FOR`
5. For each injury → creates an **InjuryRisk node** + links it with `MITIGATES`

---

### STEP 5 — `query_rag.py` (Traditional RAG)
> *"The dumb ctrl+F search — just looks for keywords"*

What it does:
1. Takes  question: *"road marathon, worried about shin splints"*
2. Searches for shoes that CONTAIN the words "road" and "shin_splints"
3. Returns whatever matches — no smarts, no connections

**The problem**: If you ask "best shoe for 42km on pavement", it might miss results because "pavement" ≠ "road".

---

### STEP 6 — `query_kg_rag.py` (Knowledge Graph RAG)
> *"The smart map-walker — follows connections to find the perfect shoe"*

What it does:
1. Takes your question: *"road marathon, worried about shin splints"*
2. Walks the graph: Shoe → RATED_FOR → Condition(road, 42km) AND Shoe → MITIGATES → InjuryRisk(shin_splints)
3. Returns shoes that match **both** conditions
4. Ranks them by injury mitigation count + weight

**The win**: Finds the exact right shoe because it understands relationships, not just keywords.

---


Now in Neo4j Browser

 `http://localhost:7474` 

Ran the below querty

MATCH (s:Shoe)-[r]->(n) RETURN s, r, n LIMIT 50


we can see:
- 🟡 **Shoe nodes** (Nike Vaporfly, ASICS Gel, etc.)
- 🟢 **Condition nodes** (road/42km, trail/21km)
- 🔴 **InjuryRisk nodes** (shin_splints, plantar_fasciitis)
- **Lines (relationships)** connecting everything

---

## 🆚 Side-by-Side Comparison

| Feature | Traditional RAG | Knowledge Graph RAG |
|---|---|---|
| How it searches | Keyword matching | Graph traversal (follows connections) |
| Understands relationships? | ❌ No | ✅ Yes |
| Can rank by multiple factors? | ❌ No | ✅ Yes |
| Easy to explain? | ✅ Yes | ✅ Yes (it's a map!) |
| Best for complex questions? | ❌ No | ✅ Yes |

---

## ✅ Pre-requisites (Before You Start)

1. **Neo4j Desktop** installed and running (free at neo4j.com)
2. **Python 3.8+** installed
3. **Cursor** as your code editor
4. A Neo4j database created with a password you remember



<img width="1012" height="498" alt="visualisation" src="https://github.com/user-attachments/assets/717b89e1-6fba-496c-a1ad-a02bc539c77d" />
