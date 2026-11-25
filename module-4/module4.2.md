# Module 4.2: Guarded Text-to-SQL Conversational Agent (`text_to_sql.py`)

## 1. Overview & Purpose
* **What problem does it solve?** (Security operators want to ask plain-English questions like *"How many red cars breached Zone A yesterday?"* without knowing SQL or touching raw database files.)
* **What is its core job? **
Translates natural-language questions into safe, schema-accurate SQLite queries against `spectra_events.db`, executes them, and summarizes the rows back in plain English.)

## 2. Pipeline Architecture (4-Step Flow)
1. **Schema Injection:** Reads the exact table definitions (`observations`, `alerts`) and embeds them into the LLM system prompt so the model never hallucinates column names.
2. **SQL Generation:** Sends the user's question + schema to the local LLM (Ollama) to generate a structured SQL query.
3. **Pydantic Safety Gatekeeper (CRITICAL):** Validates the generated SQL *before* it ever touches SQLite—blocking destructive commands (`DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`) and enforcing `SELECT`-only + `LIMIT` bounds.
4. **Execution & Answer Synthesis:** Opens `spectra_events.db` in **read-only URI mode** (`file:spectra_events.db?mode=ro`), runs the query, and formats the result into a concise operator answer.

## 3. Security & Engineering Guardrails
* **Defense-in-Depth Read-Only Enforcement:** Why do we combine **both** Pydantic regex/AST validation *and* SQLite's OS-level `?mode=ro` flag? (Even if prompt injection tricks the LLM into outputting `DROP TABLE alerts;`, the database connection itself physically refuses writes.)
* **Row & Latency Caps:** Enforcing a maximum `LIMIT` (e.g., `LIMIT 50`) prevents a broad query like *"Show all observations"* from dumping 500,000 rows into memory.
* **Empty-Result Handling:** If a query returns `0` rows, the agent clearly reports *"No matching events found"* rather than hallucinating incidents.
