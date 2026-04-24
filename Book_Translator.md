<img width="1723" height="982" alt="image" src="https://github.com/user-attachments/assets/becc261f-d4c4-4da4-9bd5-834e3b1172e3" />

<img width="1578" height="940" alt="image" src="https://github.com/user-attachments/assets/5c5fe8b7-46e6-48a0-8989-5d1e0b5ad48e" />
<img width="1762" height="950" alt="image" src="https://github.com/user-attachments/assets/2081c04f-3af5-45c2-aa83-bd07adf497f7" />


#!/usr/bin/env python3
"""📚 Book Translation Assistant — Translate entire books to Indian languages"""

import gradio as gr
import os, time, math, re, tempfile
from pathlib import Path
import requests
import fitz                          # PyMuPDF  — reads PDF
from docx import Document as DocxDoc # python-docx — reads DOCX
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from fpdf import FPDF
from dotenv import load_dotenv

load_dotenv()

# ── Constants ──────────────────────────────────────────────────────────────────
FONTS_DIR = Path(__file__).parent / "fonts"
FONTS_DIR.mkdir(exist_ok=True)

CHUNK_WORDS = 700   # words per translation chunk

LANGUAGES = [
    "Tamil (தமிழ்)",
    "Hindi (हिन्दी)",
    "Telugu (తెలుగు)",
    "Kannada (ಕನ್ನಡ)",
    "Malayalam (മലയാളം)",
    "Bengali (বাংলা)",
    "Gujarati (ગુજરાતી)",
    "Marathi (मराठी)",
    "Punjabi (ਪੰਜਾਬੀ)",
    "Odia (ଓଡ଼ିଆ)",
    "Assamese (অসমীয়া)",
    "Sanskrit (संस्कृतम्)",
    "Urdu (اردو)",
]

LANG_NAME = {l: l.split(" (")[0] for l in LANGUAGES}

# Map language → Noto font (file, CDN url, fpdf family name)
FONT_CFG = {
    "Tamil":      ("NotoSansTamil-Regular.ttf",      "NotoSansTamil/NotoSansTamil-Regular.ttf",           "Tamil"),
    "Hindi":      ("NotoSansDevanagari-Regular.ttf",  "NotoSansDevanagari/NotoSansDevanagari-Regular.ttf", "Deva"),
    "Marathi":    ("NotoSansDevanagari-Regular.ttf",  "NotoSansDevanagari/NotoSansDevanagari-Regular.ttf", "Deva"),
    "Sanskrit":   ("NotoSansDevanagari-Regular.ttf",  "NotoSansDevanagari/NotoSansDevanagari-Regular.ttf", "Deva"),
    "Telugu":     ("NotoSansTelugu-Regular.ttf",      "NotoSansTelugu/NotoSansTelugu-Regular.ttf",         "Telugu"),
    "Kannada":    ("NotoSansKannada-Regular.ttf",     "NotoSansKannada/NotoSansKannada-Regular.ttf",       "Kannada"),
    "Malayalam":  ("NotoSansMalayalam-Regular.ttf",   "NotoSansMalayalam/NotoSansMalayalam-Regular.ttf",   "Malay"),
    "Bengali":    ("NotoSansBengali-Regular.ttf",     "NotoSansBengali/NotoSansBengali-Regular.ttf",       "Bengali"),
    "Assamese":   ("NotoSansBengali-Regular.ttf",     "NotoSansBengali/NotoSansBengali-Regular.ttf",       "Bengali"),
    "Gujarati":   ("NotoSansGujarati-Regular.ttf",    "NotoSansGujarati/NotoSansGujarati-Regular.ttf",     "Gujarati"),
    "Punjabi":    ("NotoSansGurmukhi-Regular.ttf",    "NotoSansGurmukhi/NotoSansGurmukhi-Regular.ttf",     "Gurmukhi"),
    "Odia":       ("NotoSansOriya-Regular.ttf",       "NotoSansOriya/NotoSansOriya-Regular.ttf",           "Oriya"),
    "Urdu":       ("NotoNastaliqUrdu-Regular.ttf",    "NotoNastaliqUrdu/NotoNastaliqUrdu-Regular.ttf",     "Urdu"),
}
CDN_BASE = "https://cdn.jsdelivr.net/gh/googlefonts/noto-fonts@main/hinted/ttf/"

# ── Font downloader ────────────────────────────────────────────────────────────
def get_font_path(lang: str) -> str | None:
    if lang not in FONT_CFG:
        return None
    fname, url_suffix, _ = FONT_CFG[lang]
    path = FONTS_DIR / fname
    if path.exists():
        return str(path)
    try:
        r = requests.get(CDN_BASE + url_suffix, timeout=120, stream=True)
        r.raise_for_status()
        with open(path, "wb") as f:
            for chunk in r.iter_content(65536):
                f.write(chunk)
        return str(path)
    except Exception as e:
        print(f"[font] download failed: {e}")
        return None

# ── Document reader ────────────────────────────────────────────────────────────
def read_document(file_path: str) -> tuple[str, str]:
    """Returns (full_text, title)"""
    ext = Path(file_path).suffix.lower()
    title = Path(file_path).stem

    if ext == ".pdf":
        doc = fitz.open(file_path)
        pages = [page.get_text() for page in doc]
        doc.close()
        return "\n\n".join(pages), title

    if ext in (".docx", ".doc"):
        doc = DocxDoc(file_path)
        paras = [p.text for p in doc.paragraphs if p.text.strip()]
        return "\n\n".join(paras), title

    if ext == ".txt":
        with open(file_path, "r", encoding="utf-8", errors="ignore") as f:
            return f.read(), title

    raise ValueError(f"Unsupported format: {ext}. Use PDF, DOCX, or TXT.")

# ── Text chunker ───────────────────────────────────────────────────────────────
def chunk_text(text: str, words: int = CHUNK_WORDS) -> list[str]:
    # Split on blank lines (paragraphs) first
    paras = [p.strip() for p in re.split(r"\n{2,}", text) if p.strip()]
    chunks, current, count = [], [], 0
    for para in paras:
        w = len(para.split())
        if count + w > words and current:
            chunks.append("\n\n".join(current))
            current, count = [], 0
        current.append(para)
        count += w
    if current:
        chunks.append("\n\n".join(current))
    return chunks

# ── LLM setup ──────────────────────────────────────────────────────────────────
def make_chain(model: str):
    llm = ChatOpenAI(
        model_name=model,
        temperature=0.1,
        max_tokens=4096,
        api_key=os.getenv("OPENAI_API_KEY"),
    )
    prompt = ChatPromptTemplate.from_messages([
        ("system",
         "You are a professional literary translator specialising in {language}. "
         "Translate the following text EXACTLY and COMPLETELY into {language}. "
         "Rules: preserve all paragraph breaks, punctuation style, and formatting. "
         "Do NOT summarise, skip, add commentary, or omit any sentence. "
         "Output ONLY the translated text, nothing else."),
        ("human", "{text}"),
    ])
    return prompt | llm | StrOutputParser()

# ── PDF builder ────────────────────────────────────────────────────────────────
class BookPDF(FPDF):
    def __init__(self, title: str, language: str, font_path: str, font_family: str):
        super().__init__()
        self.title_str   = title
        self.language    = language
        self.font_path   = font_path
        self.font_family = font_family
        self.add_font(font_family, "", font_path)
        self.set_auto_page_break(auto=True, margin=20)

    def header(self):
        self.set_font(self.font_family, size=9)
        self.set_text_color(120, 120, 120)
        self.cell(0, 8, f"{self.title_str}  |  {self.language} Translation", align="C")
        self.ln(4)

    def footer(self):
        self.set_y(-14)
        self.set_font(self.font_family, size=9)
        self.set_text_color(150, 150, 150)
        self.cell(0, 8, str(self.page_no()), align="C")

    def title_page(self):
        self.add_page()
        self.set_y(80)
        self.set_font(self.font_family, size=26)
        self.set_text_color(30, 30, 30)
        self.multi_cell(0, 12, self.title_str, align="C")
        self.ln(8)
        self.set_font(self.font_family, size=16)
        self.set_text_color(100, 80, 180)
        self.multi_cell(0, 10, f"{self.language} Translation", align="C")
        self.ln(6)
        self.set_font(self.font_family, size=11)
        self.set_text_color(160, 160, 160)
        self.multi_cell(0, 8, f"Generated by Book Translation Assistant", align="C")

    def add_body(self, text: str):
        self.add_page()
        self.set_font(self.font_family, size=12)
        self.set_text_color(20, 20, 20)
        for para in text.split("\n\n"):
            para = para.strip()
            if para:
                self.multi_cell(0, 7, para)
                self.ln(4)

def build_pdf(title: str, language: str, translated_text: str, out_path: str) -> str:
    lang_key = language.split(" (")[0]
    font_path = get_font_path(lang_key)
    if not font_path:
        raise RuntimeError(f"Could not load font for {language}.")

    _, _, family = FONT_CFG[lang_key]
    pdf = BookPDF(title, language, font_path, family)
    pdf.title_page()
    pdf.add_body(translated_text)
    pdf.output(out_path)
    return out_path

# ── Core translate function ────────────────────────────────────────────────────
def translate_book(file, language_choice, model_choice, progress=gr.Progress()):
    if file is None:
        return None, "⚠️ Please upload a document first."

    lang = LANG_NAME[language_choice]

    try:
        progress(0, desc="📖 Reading document…")
        text, title = read_document(file.name)
    except Exception as e:
        return None, f"❌ Failed to read file: {e}"

    if not text.strip():
        return None, "❌ Could not extract text from the document."

    chunks = chunk_text(text)
    total  = len(chunks)
    progress(0.02, desc=f"📦 Split into {total} chunks — downloading font…")

    # Pre-download font
    lang_key = lang
    font_path = get_font_path(lang_key)
    if not font_path:
        return None, f"❌ Font for {language_choice} could not be downloaded. Check internet connection."

    chain = make_chain(model_choice)
    translated_chunks = []

    for i, chunk in enumerate(chunks):
        progress((i + 1) / total * 0.90 + 0.05,
                 desc=f"🔄 Translating chunk {i+1}/{total}…")
        for attempt in range(3):
            try:
                result = chain.invoke({"language": lang, "text": chunk})
                translated_chunks.append(result)
                break
            except Exception as e:
                if attempt == 2:
                    translated_chunks.append(f"[Translation error on chunk {i+1}: {e}]")
                else:
                    time.sleep(2 ** attempt)

    full_translation = "\n\n".join(translated_chunks)

    progress(0.96, desc="📄 Generating PDF…")
    out_path = str(Path(tempfile.gettempdir()) / f"{title}_{lang}.pdf")
    try:
        build_pdf(title, language_choice, full_translation, out_path)
    except Exception as e:
        return None, f"❌ PDF generation failed: {e}"

    progress(1.0, desc="✅ Done!")
    word_count = len(full_translation.split())
    status = (f"✅ Translation complete!\n"
              f"📄 {total} chunks processed  |  "
              f"📝 ~{word_count:,} words translated  |  "
              f"🌐 Language: {language_choice}")
    return out_path, status

# ── Gradio UI ──────────────────────────────────────────────────────────────────
_theme = gr.themes.Soft(
    primary_hue="violet",
    secondary_hue="indigo",
    neutral_hue="slate",
    font=gr.themes.GoogleFont("Inter"),
)

_css = """
#app-header { text-align:center; padding: 28px 0 10px; }
#app-header h1 { font-size: 2.3rem; margin-bottom:6px; }
#app-header p  { color:#9ca3af; font-size:1rem; }
#translate-btn { font-size:1.1rem; }
#status-box textarea { font-size:0.95rem; line-height:1.6; }
"""

with gr.Blocks(title="📚 Book Translation Assistant") as demo:
    gr.HTML("""
        <div id="app-header">
            <h1>📚 Book Translation Assistant</h1>
            <p>Translate entire books into Indian languages — PDF output with native fonts</p>
        </div>
    """)

    with gr.Row():
        # ── Left panel ─────────────────────────────────────────────────────────
        with gr.Column(scale=1):
            gr.Markdown("### 📂 Upload Document")
            file_input = gr.File(
                label="Upload PDF / DOCX / TXT (up to 1500 pages)",
                file_types=[".pdf", ".docx", ".doc", ".txt"],
            )

            gr.Markdown("### ⚙️ Settings")
            lang_dropdown = gr.Dropdown(
                label="Target Language",
                choices=LANGUAGES,
                value="Tamil (தமிழ்)",
                info="Choose the Indian language to translate into",
            )
            model_dropdown = gr.Dropdown(
                label="AI Model",
                choices=[
                    "gpt-4o",
                    "gpt-4-turbo",
                    "gpt-3.5-turbo",
                ],
                value="gpt-4o",
                info="GPT-4o gives the most accurate translations",
            )

            translate_btn = gr.Button("🚀 Start Translation", variant="primary",
                                      elem_id="translate-btn")

            gr.Markdown("""
> **Tips**
> - GPT-4o is recommended for literary accuracy
> - Large books (1000+ pages) may take 20-40 min
> - Font is auto-downloaded on first use per language
""")

        # ── Right panel ────────────────────────────────────────────────────────
        with gr.Column(scale=1):
            gr.Markdown("### 📊 Progress & Output")
            status_box = gr.Textbox(
                label="Status",
                lines=4,
                interactive=False,
                placeholder="Status will appear here after translation starts…",
                elem_id="status-box",
            )
            output_file = gr.File(
                label="📥 Download Translated PDF",
                interactive=False,
            )

    # ── Examples ───────────────────────────────────────────────────────────────
    gr.Markdown("---\n### 🌐 Supported Indian Languages")
    gr.DataFrame(
        value=[
            ["Tamil", "தமிழ்", "Dravidian"],
            ["Hindi", "हिन्दी", "Indo-Aryan"],
            ["Telugu", "తెలుగు", "Dravidian"],
            ["Kannada", "ಕನ್ನಡ", "Dravidian"],
            ["Malayalam", "മലയാളം", "Dravidian"],
            ["Bengali", "বাংলা", "Indo-Aryan"],
            ["Gujarati", "ગુજરાતી", "Indo-Aryan"],
            ["Marathi", "मराठी", "Indo-Aryan"],
            ["Punjabi", "ਪੰਜਾਬੀ", "Indo-Aryan"],
            ["Odia", "ଓଡ଼ିଆ", "Indo-Aryan"],
            ["Assamese", "অসমীয়া", "Indo-Aryan"],
            ["Sanskrit", "संस्कृतम्", "Indo-Aryan"],
            ["Urdu", "اردو", "Indo-Aryan"],
        ],
        headers=["Language", "Native Script", "Family"],
        interactive=False,
    )

    gr.HTML("""
        <div style="text-align:center;padding:20px 0 8px;color:#6b7280;font-size:.85rem;">
            Built with ❤️ using Gradio · LangChain · OpenAI · PyMuPDF · fpdf2
        </div>
    """)

    translate_btn.click(
        fn=translate_book,
        inputs=[file_input, lang_dropdown, model_dropdown],
        outputs=[output_file, status_box],
    )

if __name__ == "__main__":
    demo.launch(theme=_theme, css=_css, server_port=7861)
