<img width="1891" height="975" alt="image" src="https://github.com/user-attachments/assets/27b11463-7041-4858-b98f-d06b76d087c5" />
<img width="1877" height="477" alt="image" src="https://github.com/user-attachments/assets/2d3670eb-9cb4-4af1-9154-ffc9b235d871" />


import gradio as gr
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
import os
from dotenv import load_dotenv

load_dotenv()

# ── LLM setup ────────────────────────────────────────────────────────────────
llm = ChatOpenAI(
    model_name="gpt-3.5-turbo",
    temperature=0.7,
    api_key=os.getenv("OPENAI_API_KEY"),
)

prompt = PromptTemplate(
    input_variables=["tone", "recipient", "purpose", "key_points", "sender_name"],
    template="""
You are an expert email writing assistant. Write a {tone} email based on the details below.

Recipient  : {recipient}
Purpose    : {purpose}
Key Points : {key_points}
Sender Name: {sender_name}

Guidelines:
- Match the tone exactly ({tone})
- Use a clear subject line at the very top prefixed with "Subject:"
- Structure with greeting, body paragraphs, and a professional sign-off
- Keep it concise yet complete
- Do NOT add any explanation outside the email

Write the email now:
""",
)

chain = prompt | llm | StrOutputParser()

# ── Core function ─────────────────────────────────────────────────────────────
def generate_email(tone, recipient, purpose, key_points, sender_name):
    if not recipient.strip() or not purpose.strip():
        return "⚠️ Please fill in at least the **Recipient** and **Purpose** fields."

    result = chain.invoke({
        "tone": tone,
        "recipient": recipient,
        "purpose": purpose,
        "key_points": key_points if key_points.strip() else "None specified",
        "sender_name": sender_name if sender_name.strip() else "The Sender",
    })
    return result

# ── Gradio UI ─────────────────────────────────────────────────────────────────
_theme = gr.themes.Soft(
    primary_hue="violet",
    secondary_hue="indigo",
    neutral_hue="slate",
    font=gr.themes.GoogleFont("Inter"),
)

_css = """
    #title-banner {
        text-align: center;
        padding: 24px 0 8px;
    }
    #title-banner h1 { font-size: 2.2rem; margin-bottom: 4px; }
    #title-banner p  { color: #9ca3af; font-size: 1rem; }
    #generate-btn    { font-size: 1.05rem; padding: 12px; }
    #output-box textarea {
        font-family: 'Georgia', serif;
        font-size: 0.97rem;
        line-height: 1.7;
    }
"""

with gr.Blocks(title="✉️ Smart Email Writing Assistant") as demo:

    # ── Header ────────────────────────────────────────────────────────────────
    gr.HTML("""
        <div id="title-banner">
            <h1>✉️ Smart Email Writing Assistant</h1>
            <p>Powered by GPT-3.5 · Craft perfect emails in seconds</p>
        </div>
    """)

    # ── Main layout ───────────────────────────────────────────────────────────
    with gr.Row():
        # Left column — inputs
        with gr.Column(scale=1):
            gr.Markdown("### 📝 Email Details")

            tone = gr.Dropdown(
                label="Email Tone",
                choices=[
                    "Professional",
                    "Formal",
                    "Friendly & Casual",
                    "Persuasive",
                    "Apologetic",
                    "Follow-up",
                    "Thank You",
                    "Cold Outreach",
                ],
                value="Professional",
                info="Select the style that fits your context",
            )

            recipient = gr.Textbox(
                label="Recipient Name / Role",
                placeholder="e.g. Mr. Ramesh Kumar, HR Manager",
                lines=1,
            )

            purpose = gr.Textbox(
                label="Purpose of Email",
                placeholder="e.g. Request a meeting to discuss Q2 project timelines",
                lines=2,
            )

            key_points = gr.Textbox(
                label="Key Points to Include (optional)",
                placeholder="e.g. Mention the delay in delivery, request new deadline of May 15",
                lines=3,
            )

            sender_name = gr.Textbox(
                label="Your Name",
                placeholder="e.g. Anandha Ganesh",
                lines=1,
            )

            generate_btn = gr.Button(
                "✨ Generate Email",
                variant="primary",
                elem_id="generate-btn",
            )

        # Right column — output
        with gr.Column(scale=1):
            gr.Markdown("### 📬 Generated Email")
            output = gr.Textbox(
                label="",
                lines=22,
                placeholder="Your beautifully crafted email will appear here...",
                elem_id="output-box",
            )

    # ── Examples ──────────────────────────────────────────────────────────────
    gr.Markdown("---\n### 💡 Quick Examples — click to auto-fill")
    gr.Examples(
        examples=[
            ["Professional",    "Mr. Arun Sharma, Project Manager", "Request a project status update",            "Ask for updated timeline, mention the client presentation on May 10", "Anandha Ganesh"],
            ["Cold Outreach",   "Ms. Priya Nair, CEO of TechNova",  "Introduce our AI consulting services",       "Highlight 3 key benefits, request a 20-minute call",                  "Anandha Ganesh"],
            ["Apologetic",      "Dr. Meena Krishnan",               "Apologize for missing the team meeting",     "Explain reason briefly, offer to reschedule, assure it won't repeat", "Anandha Ganesh"],
            ["Thank You",       "Mr. Vijay Rajan, Interview Panel", "Thank them after a job interview",           "Reiterate enthusiasm for the role, mention key discussion points",    "Anandha Ganesh"],
            ["Follow-up",       "Sales Team, Infosys",              "Follow up on a pending proposal",            "Proposal sent 2 weeks ago, ask for feedback or next steps",           "Anandha Ganesh"],
        ],
        inputs=[tone, recipient, purpose, key_points, sender_name],
        label="",
    )

    # ── Footer ────────────────────────────────────────────────────────────────
    gr.HTML("""
        <div style="text-align:center; padding: 20px 0 8px; color: #6b7280; font-size: 0.85rem;">
            Built with ❤️ using Gradio + LangChain + OpenAI
        </div>
    """)

    # ── Event binding ─────────────────────────────────────────────────────────
    generate_btn.click(
        fn=generate_email,
        inputs=[tone, recipient, purpose, key_points, sender_name],
        outputs=output,
    )

# ── Launch ────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    demo.launch(theme=_theme, css=_css)
