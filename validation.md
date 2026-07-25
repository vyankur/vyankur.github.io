# Portfolio Governance & Architectural Validation Rules

This document serves as the single source of truth for mandatory validation rules, file roles, and pre-commit governance checks for **vyankur.github.io**.

---

## 1. Grid Display Rules (3-Item Maximum Rule)

> [!IMPORTANT]
> **Progressive Disclosure Constraint**: No grid section on the landing page (Featured Insights, Knowledge Hub initial view, Testimonials grid) may display more than **3 items directly in the initial view** without user interaction.

*   **Featured Insights (`#featured-insights`)**: Must display exactly **3 standout cards** in the initial view:
    1. Card 1: `tabpy-integration-tableau-python` (Bringing Python Into Tableau: TabPy Integration)
    2. Card 2: `tableau-server-repository` (Querying Tableau Server Repository Metadata)
    3. Card 3: `ai-in-tableau` (AI in Tableau: Bedrock & LangChain Agents)
*   **Knowledge Hub (`#insights`)**: Must render exactly **3 initial cards** (`featured.slice(0, 3)`). Clicking *"View Knowledge Roadmap"* toggles progressive disclosure for the full roadmap grid.
*   **Testimonials Grid**: Must display 3 featured cards with swipe carousel navigation on smaller screens.

---

## 2. File Roles & Architectural Purpose

*   **`index.html`**: The **PRIMARY, LIVE SINGLE-PAGE APPLICATION (SPA)** served by GitHub Pages at `https://vyankur.github.io/`. Contains all active HTML, CSS design tokens, dynamic JavaScript models, and single-page hash routing logic (`#/insights/<slug>`, `#/projects/<id>`).
*   **`index_v4.html`**: A **standalone local backup / staging mirror snapshot** kept in sync for experimental changes and local testing.
*   **`content/insights/`**: Contains technical markdown (`.md`) articles and `articles.json` index catalog. **NO `.md` files belong in `content/projects/`**.
*   **`content/projects/`**: Stores `.json` metadata files for project case studies (`calendar.json`, `ai-bi.json`, `workbook-sentinel.json`, etc.).
*   **`assets/tabpy-screenshots/`**: Stores sequential screenshots (`1.png`, `1.1.png`, `2.png`, `3.png`, `4.png`, `5.png`, `6.png`) referenced inside the TabPy Integration guide.

---

## 3. Mandatory Quality & Validation Rules

### A. Parity & Catalog Indexing
- Every local `.md` article inside `content/insights/` MUST have a matching catalog entry inside `content/insights/articles.json` AND inside the `knowledgeArticles` Javascript array in both `index.html` and `index_v4.html`.

### B. JavaScript Runtime Integrity
- **Single Script Block Execution**: Keep core application logic inside unified `<script>` blocks. Never append logic inside `<script type="application/ld+json">` blocks.
- **Zero Syntax Errors**: Run `validate_js.js` to compile all script blocks via Node.js before committing.

### C. Editorial & Typography Standards
- **Zero Em-Dashes**: Do not use em-dashes (`—`) or en-dashes (`–`) in public UI copy or articles. Replace with commas, colons, or clean parentheses to maintain a natural human tone.
- **Mermaid Diagram Formatting**: Flowcharts must use `flowchart TD` with explicit subgraphs and HTML labels (`<br/><b>...</b>`) so font sizes render clearly across devices.

### D. Chatbot Intelligence Engine
- The chatbot `processUserMessage(query)` must perform multi-source keyword aggregation across Articles (`knowledgeArticles`), Project Case Studies (`projectCaseStudies`), Work History, and Core Skills.
- Chatbot responses must format links as clickable markdown `[Title](#/insights/slug)` and `[Title](#/projects/id)`.

---

## 4. Automated Pre-Commit Validation Checklist

Execute the following verification scripts before any commit:
1. `python -c "import json; json.load(open('content/insights/articles.json'))"` (validates `articles.json` JSON syntax)
2. `node validate_js.js` (validates JS compilation of script blocks)
3. `python -c "import glob; files = ['index.html', 'index_v4.html', 'content/insights/articles.json'] + glob.glob('content/insights/*.md'); found = [print(f, i+1, line.strip()) for f in files for i, line in enumerate(open(f, encoding='utf-8')) if '—' in line or '–' in line]; assert len(found) == 0"` (validates zero em-dashes)
