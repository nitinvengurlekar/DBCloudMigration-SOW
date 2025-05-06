import os
import re
import requests
import streamlit as st
from bs4 import BeautifulSoup
from jinja2 import Template
from langchain_community.chat_models import ChatOpenAI
from langchain_core.messages import HumanMessage
import pdfplumber

# ─── Configuration ─────────────────────────────────────────────────────────────
os.environ['OPENAI_API_KEY'] = st.secrets['OPENAI_API_KEY']
st.title("Oracle Cloud DB Migration Agent")
st.markdown(
    "Provide inputs to generate a migration guide and SOW document. "
    "This agent references Oracle's official migration planning guide and an optional PDF."
)

# ─── Helpers ────────────────────────────────────────────────────────────────────

@st.cache_data(show_spinner=False)
def extract_pdf_sections(uploaded_pdf, section_titles):
    """Extract named sections and any tables from a PDF."""
    try:
        with pdfplumber.open(uploaded_pdf) as pdf:
            full_text = "\n".join(page.extract_text() or "" for page in pdf.pages)
            tables = []
            for page in pdf.pages:
                for table in page.extract_tables():
                    header, *rows = table
                    rows_str = "\n".join(" | ".join(cell or "" for cell in r) for r in rows)
                    tables.append(" | ".join(header) + "\n" + rows_str)

        sections = {}
        for title in section_titles:
            # match "Title\n...content...\n\nNextTitle" (case‐insensitive)
            m = re.search(rf"(?i){title}:\s*(.*?)(?=\n[A-Z][a-z]+:|\Z)", full_text, re.DOTALL)
            if m:
                sections[title] = m.group(1).strip()

        if not sections and not tables:
            return None

        parts = []
        for t, content in sections.items():
            parts.append(f"{t}:\n{content}")
        if tables:
            parts.append("-- Extracted Tables --")
            parts.extend(tables)

        return "\n\n".join(parts)

    except Exception as e:
        st.error(f"PDF extraction failed: {e}")
        return None


@st.cache_data(show_spinner=False)
def fetch_migration_guide_content(base_url, max_pages=5):
    """Scrape up to max_pages of Oracle's migration guide."""
    session = requests.Session()
    try:
        resp = session.get(base_url, timeout=10)
        resp.raise_for_status()
        soup = BeautifulSoup(resp.text, 'html.parser')

        guide_paragraphs = [p.get_text(strip=True) for p in soup.select('article p')]
        # find internal sublinks
        sublinks = {
            (a := link.get('href')) and (
                'https://www.oracle.com' + a if a.startswith('/') else a
            )
            for link in soup.select('article a[href]')
            if 'oracle.com' in link['href'] or base_url in link['href']
        }

        for idx, href in enumerate(list(sublinks)[: max_pages - 1]):
            try:
                r2 = session.get(href, timeout=5)
                r2.raise_for_status()
                sub = BeautifulSoup(r2.text, 'html.parser')
                guide_paragraphs += [p.get_text(strip=True) for p in sub.select('article p')]
            except requests.RequestException:
                continue

        return "\n".join(guide_paragraphs)[:50_000]
    except Exception as e:
        st.error(f"Could not fetch migration guide: {e}")
        return ""


def generate_migration_guide(input_dict, guide_text, pdf_text=None) -> str:
    """Call the LLM to build a 3‑part migration guide, returning Markdown."""
    prompt = "\n\n".join(
        [
            "You are an expert Oracle Cloud migration consultant. "
            "Be diligent about producing the appropriate hours for each migration task.",
            "Oracle migration planning guide content:\n" + guide_text,
            *(["Reference PDF excerpt:\n" + pdf_text] if pdf_text else []),
            (
                "Create a 3-part Oracle DB migration guide with clear task names and hours:"
                f"\n• Planning\n• Execution\n• Post-Migration Validation\n"
                f"Inputs: DB size={input_dict['database_size']}, "
                f"Downtime={input_dict['downtime_window']}, "
                f"Upgrade required={input_dict['upgrade_required']}, "
                f"Versions: {input_dict['current_version']}→{input_dict['target_version']}, "
                f"Platform={input_dict['target_platform']}, "
                f"Include non-prod={input_dict['include_nonprod']}."
            ),
        ]
    )
    try:
        return llm([HumanMessage(content=prompt)]).content
    except Exception as e:
        return f"Error generating migration guide: {e}"


def parse_total_effort(migration_markdown: str) -> int:
    """
    Scan the generated guide for lines like "... – XX hours"
    and sum up those hour values.
    """
    hours = re.findall(r"(\d+)\s*(?:hrs|hours?)", migration_markdown, flags=re.IGNORECASE)
    return sum(int(h) for h in hours) if hours else 0


def generate_sow(db_size, current_version, target_version, downtime,
                 platform, total_effort, pdf_excerpt) -> str:
    """Render the Jinja SOW template with calculated total effort."""
    template = Template(
        """
Schedule A: Statement of Work
Project: Migration and Upgrade to Oracle Cloud
Client: [Client Name]
Effective Date: [TBD]

1. Objectives & Scope
Migrate the on-premise Oracle DB ({{ db_size }}, version {{ current_version }})
to {{ platform }} ({{ target_version }}) with a maximum offline window of {{ downtime }}.

2. High-Level Tasks & Estimates

| Phase                          | Description                                              | Effort (hrs) |
|--------------------------------|----------------------------------------------------------|--------------|
| Assessment & Discovery         |  Kick-off, review exachk, finalize DBs                   | 30           |
| Environment Provisioning       |  Setup compute/network/storage/IAM                       | 40           |
| Pre-Migration Validation       |  Use CPAT & DB Advisor                                   | 40           |
| ZDM Test Migration             |  UAT migration & upgrade validation                      | 80           |
| Cutover Planning               |  Dry-run, rollback strategy                              | 40           |
| Production Migration & Upgrade |  RMAN restore, upgrade, validation                       | 16           |
| Post-Migration Support         |  Tuning, incident response, KT                           | 40           |
| Project Mgmt & Reporting       |  Status calls, issue tracking, closure                   | 32           |
| **Total**                      |                                                          | **{{ total_effort }}** |

3. Deliverables:
- Migration plan & runbooks
- Upgraded database on {{ platform }}
- Knowledge transfer documentation
- Final project closure report

{% if pdf_excerpt %}
4. Reference Document Excerpt:
> {{ pdf_excerpt | replace("\n", "\n> ") }}
{% endif %}
"""
    )
    return template.render(
        db_size=db_size,
        current_version=current_version,
        target_version=target_version,
        downtime=downtime,
        platform=platform,
        total_effort=total_effort,
        pdf_excerpt=pdf_excerpt or ""
    ).strip()


# ─── UI & Main Logic ────────────────────────────────────────────────────────────

# Optional SOW reference PDF
doc_pdf = st.file_uploader("Upload reference PDF for SOW (optional)", type=["pdf"])
pdf_excerpt = None
if doc_pdf:
    pdf_excerpt = extract_pdf_sections(doc_pdf, ["Introduction", "Objective", "Scope and Task Plan", "Assumptions"])

with st.form("migration_form"):
    db_size         = st.text_input("Database Size", "2TB")
    downtime        = st.text_input("Downtime Window", "5 hours")
    upgrade_required= st.selectbox("Is Upgrade Required?", ["Yes", "No"]) == "Yes"
    current_version = st.text_input("Current DB Version", "12.2")
    target_version  = st.text_input("Target DB Version", "19c")
    platform        = st.text_input("Target Platform", "Exadata Cloud Service")
    include_nonprod = st.selectbox("Include Non-Prod Environments?", ["Yes", "No"]) == "Yes"
    submitted       = st.form_submit_button("Generate Migration Guide and SOW")

# Initialize LLM once
llm = ChatOpenAI(model_name="gpt-4o", temperature=0.2)

if submitted:
    user_input = {
        "database_size":    db_size,
        "downtime_window":  downtime,
        "upgrade_required": upgrade_required,
        "current_version":  current_version,
        "target_version":   target_version,
        "target_platform":  platform,
        "include_nonprod":  include_nonprod,
    }

    with st.spinner("Fetching Oracle migration guide…"):
        guide_text = fetch_migration_guide_content("https://www.oracle.com/database/cloud-migration/")

    with st.spinner("Generating Migration Guide…"):
        migration_md = generate_migration_guide(user_input, guide_text, pdf_excerpt)
        total_hours = parse_total_effort(migration_md)

    st.subheader("Migration Guide")
    st.text_area("Generated Guide", migration_md, height=300)
    st.download_button("Download Migration Guide", data=migration_md, file_name="oracle_migration_guide.txt")

    with st.spinner("Rendering SOW…"):
        sow_md = generate_sow(db_size, current_version, target_version, downtime, platform, total_hours, pdf_excerpt)

    st.subheader("Statement of Work (SOW)")
    st.text_area("Generated SOW", sow_md, height=400)
    st.download_button("Download SOW", data=sow_md, file_name="oracle_migration_sow.txt")
