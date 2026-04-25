<img width="1729" height="508" alt="image" src="https://github.com/user-attachments/assets/b273907e-cae9-4e08-b4d3-47b94938ea37" />

"""
rag_app.py
==========
RAG System with Persistent Vector DB (ChromaDB)
------------------------------------------------
Compatible with:  langchain >= 1.0,  langchain-community,  langchain-openai
Features:
  ✅ Upload PDF, TXT, DOCX documents
  
  ✅ Chunks & embeds documents into ChromaDB (persisted to disk)
  
  ✅ SKIPS re-indexing if document already exists (hash-based check)


  ✅ Query via natural language using OpenAI GPT
  
  ✅ Gradio UI — three tabs: Upload | Ask | View Docs

How "no re-indexing" works:
  - Each document is tracked by filename → MD5 hash in indexed_docs.json
  - On upload we check: same file already indexed? → skip and tell the user
  - ChromaDB persists at ./chroma_db/ and survives restarts
"""

import os
import json
import shutil
import hashlib
import gradio as gr
from pathlib import Path
from dotenv import load_dotenv

# ── LangChain 1.x imports ─────────────────────────────────────────────────────
from langchain_community.document_loaders import PyPDFLoader, TextLoader, Docx2txtLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# ── Configuration ─────────────────────────────────────────────────────────────
load_dotenv(dotenv_path=Path(__file__).parent.parent / ".env")

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
CHROMA_DB_DIR  = str(Path(__file__).parent / "chroma_db")       # Persistent vector store
INDEX_TRACKER  = str(Path(__file__).parent / "indexed_docs.json")
UPLOAD_DIR     = str(Path(__file__).parent / "uploads")

os.makedirs(CHROMA_DB_DIR, exist_ok=True)
os.makedirs(UPLOAD_DIR, exist_ok=True)

# ── Model setup ───────────────────────────────────────────────────────────────
embeddings = OpenAIEmbeddings(openai_api_key=OPENAI_API_KEY, model="text-embedding-3-small")
llm        = ChatOpenAI(openai_api_key=OPENAI_API_KEY, model="gpt-3.5-turbo", temperature=0)

# ── LCEL Prompt ───────────────────────────────────────────────────────────────
SYSTEM_PROMPT = (
    "You are a helpful assistant. Use ONLY the following context from the uploaded documents "
    "to answer the question. If the answer is not found in the context, say: "
    "'I couldn't find this information in the uploaded documents.'\n\n"
    "Context:\n{context}"
)

prompt = ChatPromptTemplate.from_messages([
    ("system", SYSTEM_PROMPT),
    ("human", "{question}"),
])


# ═══════════════════════════════════════════════════════════════════════════════
#  Tracker helpers
# ═══════════════════════════════════════════════════════════════════════════════

def load_tracker() -> dict:
    """Load filename → hash/chunk-count mapping from disk."""
    if os.path.exists(INDEX_TRACKER):
        with open(INDEX_TRACKER, "r") as f:
            return json.load(f)
    return {}


def save_tracker(tracker: dict) -> None:
    """Persist tracker to disk."""
    with open(INDEX_TRACKER, "w") as f:
        json.dump(tracker, f, indent=2)


def get_file_hash(file_path: str) -> str:
    """MD5 hash of a file — used to detect identical uploads."""
    h = hashlib.md5()
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()


# ═══════════════════════════════════════════════════════════════════════════════
#  Document loading
# ═══════════════════════════════════════════════════════════════════════════════

def load_document(file_path: str):
    """Pick the right LangChain loader based on file extension."""
    ext = Path(file_path).suffix.lower()
    if ext == ".pdf":
        return PyPDFLoader(file_path).load()
    elif ext == ".txt":
        return TextLoader(file_path, encoding="utf-8").load()
    elif ext in (".docx", ".doc"):
        return Docx2txtLoader(file_path).load()
    else:
        raise ValueError(f"Unsupported type: {ext}. Use PDF, TXT, or DOCX.")


# ═══════════════════════════════════════════════════════════════════════════════
#  Vector store helpers
# ═══════════════════════════════════════════════════════════════════════════════

def get_vectorstore() -> Chroma:
    """Open (or create) the persisted ChromaDB collection."""
    return Chroma(
        persist_directory=CHROMA_DB_DIR,
        embedding_function=embeddings,
        collection_name="rag_documents",
    )


# ═══════════════════════════════════════════════════════════════════════════════
#  Core: Index document
# ═══════════════════════════════════════════════════════════════════════════════

def index_document(file_path: str, filename: str) -> str:
    """
    Embed a document into ChromaDB.
    Returns immediately (with a message) if the exact same file was already indexed.
    """
    tracker  = load_tracker()
    file_hash = get_file_hash(file_path)

    # ── Already indexed? ──────────────────────────────────────────────────────
    if filename in tracker and tracker[filename]["hash"] == file_hash:
        n = tracker[filename]["chunks"]
        return (
            f"[INFO] Already indexed — `{filename}` is already in the Vector DB "
            f"({n} chunks). No re-processing needed!"
        )

    # ── Load & split ──────────────────────────────────────────────────────────
    print(f"[INFO] Loading: {filename}")
    docs = load_document(file_path)

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=150,
        separators=["\n\n", "\n", ".", " ", ""],
    )
    chunks = splitter.split_documents(docs)

    if not chunks:
        return f"[ERROR] No text extracted from '{filename}'. Is it an image-only scanned PDF?"

    for chunk in chunks:
        chunk.metadata["source_file"] = filename   # tag every chunk with its origin

    # ── Store in ChromaDB ─────────────────────────────────────────────────────
    print(f"[INFO] Embedding {len(chunks)} chunks ...")
    vectorstore = get_vectorstore()
    vectorstore.add_documents(chunks)
    # Note: Chroma auto-persists in newer versions; calling persist() is safe either way.
    try:
        vectorstore.persist()
    except Exception:
        pass  # newer chromadb versions don't need explicit persist()

    # ── Update tracker ────────────────────────────────────────────────────────
    tracker[filename] = {"hash": file_hash, "chunks": len(chunks)}
    save_tracker(tracker)

    return (
        f"[OK] Indexed! '{filename}' -> {len(chunks)} chunks stored in Vector DB."
    )


# ═══════════════════════════════════════════════════════════════════════════════
#  Core: Query RAG
# ═══════════════════════════════════════════════════════════════════════════════

def format_docs(docs) -> str:
    return "\n\n".join(d.page_content for d in docs)


def query_rag(question: str) -> str:
    """LCEL RAG chain: retrieve → format → prompt → LLM → parse."""
    if not question.strip():
        return "⚠️ Please enter a question."

    vectorstore = get_vectorstore()

    # Guard: empty collection
    try:
        if vectorstore._collection.count() == 0:
            return "⚠️ No documents indexed yet. Please upload a document first."
    except Exception:
        pass

    retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 5},
    )

    # Retrieve source docs separately so we can show them
    source_docs = retriever.invoke(question)

    # Build LCEL chain
    chain = (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt
        | llm
        | StrOutputParser()
    )

    answer = chain.invoke(question)

    # Deduplicate and format sources
    sources = set()
    for doc in source_docs:
        src  = doc.metadata.get("source_file") or doc.metadata.get("source", "Unknown")
        page = doc.metadata.get("page")
        label = src + (f" (page {page + 1})" if page is not None else "")
        sources.add(label)

    src_lines = "\n".join(f"  • {s}" for s in sorted(sources))
    return f"{answer}\n\n---\n**📎 Sources:**\n{src_lines}"


# ═══════════════════════════════════════════════════════════════════════════════
#  Helpers for Gradio
# ═══════════════════════════════════════════════════════════════════════════════

def list_indexed_docs() -> str:
    tracker = load_tracker()
    if not tracker:
        return "📭 No documents indexed yet."
    lines = ["**📚 Indexed Documents in Vector DB:**\n"]
    for name, info in tracker.items():
        lines.append(f"  • **{name}** — {info['chunks']} chunks")
    return "\n".join(lines)


def upload_and_index(file):
    """
    Gradio 6 upload handler.
    gr.File returns a dict like {'path': '/tmp/...'} in Gradio 6,
    a plain string in some builds, or an object with .name in older Gradio.
    """
    if file is None:
        return "No file selected."
    # Handle all Gradio versions
    if isinstance(file, dict):
        src_path = file.get("path") or file.get("name", "")
    elif isinstance(file, str):
        src_path = file
    else:
        src_path = file.name  # older Gradio
    filename  = Path(src_path).name
    dest_path = os.path.join(UPLOAD_DIR, filename)
    shutil.copy2(src_path, dest_path)
    return index_document(dest_path, filename)


def chat_handler(question, history):
    """Gradio chatbot handler — uses tuple (user, bot) format."""
    if not question.strip():
        return "", history
    answer = query_rag(question)
    history = history + [(question, answer)]
    return "", history


# ═══════════════════════════════════════════════════════════════════════════════
#  Gradio UI
# ═══════════════════════════════════════════════════════════════════════════════

CUSTOM_CSS = """
.gradio-container { max-width: 920px; margin: auto; font-family: 'Segoe UI', sans-serif; }
footer            { display: none !important; }
"""

with gr.Blocks(title="Document RAG System") as demo:

    gr.Markdown("""
    # 📄 Document RAG System
    **Upload documents → Query your data with AI — powered by ChromaDB + OpenAI**

    > ⚡ Already-indexed documents are **never re-processed**. The Vector DB persists across restarts.
    """)

    with gr.Tabs():

        # ── Tab 1: Upload ──────────────────────────────────────────────────────
        with gr.Tab("Upload Document"):
            gr.Markdown("### Select a PDF, TXT, or DOCX file to index")
            file_in = gr.File(
                label="Choose file (PDF / TXT / DOCX)",
                file_types=[".pdf", ".txt", ".docx", ".doc"],
            )
            btn_upload = gr.Button("Upload & Index", variant="primary", size="lg")
            status_out = gr.Markdown()

            btn_upload.click(fn=upload_and_index, inputs=file_in, outputs=status_out)

        # ── Tab 2: Chat ────────────────────────────────────────────────────────
        with gr.Tab("Ask Questions"):
            gr.Markdown("### Ask anything about your uploaded documents")
            chatbot   = gr.Chatbot(height=420, label="RAG Assistant")
            question  = gr.Textbox(
                placeholder="Type your question and press Enter...",
                label="Your question",
                lines=2,
            )
            btn_ask = gr.Button("Ask", variant="primary")

            question.submit(fn=chat_handler, inputs=[question, chatbot], outputs=[question, chatbot])
            btn_ask.click(fn=chat_handler,  inputs=[question, chatbot], outputs=[question, chatbot])

        # ── Tab 3: Indexed docs ────────────────────────────────────────────────
        with gr.Tab("Indexed Documents"):
            gr.Markdown("### Documents currently stored in Vector DB")
            btn_refresh = gr.Button("Refresh List")
            docs_md     = gr.Markdown(value=list_indexed_docs)

            btn_refresh.click(fn=list_indexed_docs, outputs=docs_md)


if __name__ == "__main__":
    print("Starting Document RAG System ...")
    print(f"  Vector DB  ->  {CHROMA_DB_DIR}")
    print(f"  Tracker    ->  {INDEX_TRACKER}")
    print(f"  Uploads    ->  {UPLOAD_DIR}")
    demo.launch(server_port=7860, share=False)
