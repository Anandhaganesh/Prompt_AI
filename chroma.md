import chromadb
from sentence_transformers import SentenceTransformer

# Initialize ChromaDB with local persistence (saves to ./chroma_db folder)
client = chromadb.PersistentClient(path="./chroma_db")

# Create or get a collection
collection = client.get_or_create_collection(
    name="my_documents",
    metadata={"hnsw:space": "cosine"}  # use cosine similarity
)

print("ChromaDB initialized ✓")

model = SentenceTransformer("all-MiniLM-L6-v2")

print("Model loaded ✓")

# Your documents
documents = [
    "Python is a high-level programming language known for its simplicity.",
    "ChromaDB is an open-source vector database for AI applications.",
    "HuggingFace provides pre-trained models for NLP tasks.",
    "Vector embeddings convert text into numerical representations.",
    "Semantic search finds results based on meaning, not keywords."
]

# Generate embeddings using HuggingFace model
embeddings = model.encode(documents).tolist()

# Add to ChromaDB collection
collection.add(
    documents=documents,
    embeddings=embeddings,
    ids=[f"doc_{i}" for i in range(len(documents))]  # unique IDs required
)

print(f"Added {len(documents)} documents ✓")

def semantic_search(query_text, top_k=3):
    # Embed the query using the same model
    query_embedding = model.encode([query_text]).tolist()

    # Query ChromaDB
    results = collection.query(
        query_embeddings=query_embedding,
        n_results=top_k
    )

    print(f"\nQuery: '{query_text}'")
    print("-" * 50)
    for i, (doc, distance) in enumerate(
        zip(results["documents"][0], results["distances"][0])
    ):
        score = round(1 - distance, 4)  # convert distance → similarity
        print(f"[{i+1}] Score: {score:.4f} | {doc}")

# Test it
semantic_search("What is a vector database?")
semantic_search("How does language processing work?")



Query: 'What is a vector database?'
--------------------------------------------------
[1] Score: 0.5821 | ChromaDB is an open-source vector database for AI applications.
[2] Score: 0.4266 | Vector embeddings convert text into numerical representations.
[3] Score: 0.1850 | Semantic search finds results based on meaning, not keywords.

Query: 'How does language processing work?'
--------------------------------------------------
[1] Score: 0.3175 | HuggingFace provides pre-trained models for NLP tasks.
[2] Score: 0.3056 | Python is a high-level programming language known for its simplicity.
[3] Score: 0.2652 | Vector embeddings convert text into numerical representations.
