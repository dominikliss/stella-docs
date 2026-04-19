# Cursor rule export: `email-ai-analysis.mdc`

_Source of agent rule in theme repo: `.cursor/rules/email-ai-analysis.mdc`._

---

# E-Mail-AI-Analysen (Theme)

Vollständige Architektur: **`architecture.md`** → Abschnitt **„E-Mail-AI-Analysen (Ollama, Verwaltung → AI-Profile)“**.

## Kurzreferenz

| Thema | Schreibstil | Klassifizierung |
|--------|-------------|------------|
| Profil `name` | `writing_style` | `classification` |
| `source_type` | `email` | `email` |
| Corpus (`dls_mail_message`) | `direction = outbound` | `direction = inbound` |
| Primäre Tabelle | `dls_ai_profile` + `dls_ai_profile_run` | gleich |
| REST (neu) | `GET/PATCH /dls/v1/ai/profiles`, `POST …/run`, promote, worker/stop | gleich |
| Legacy kompatibel | `ai-writing-style.php` (Profil-gestützt) | `ai-email-classification-suggestions.php` |
| Worker | `run-ai-profile.php` + `ai-profile-run-execute.php` | gleich |
| Aktives Grounding | `dls_ai_profile.grounding` zuerst | gleich; `dls_get_active_email_classification_grounding()` |

- **Historie:** weiterhin `dls_writing_style_run` via `WritingStyleHistoryService` (Worker protokolliert parallel); zusätzlich `dls_ai_profile_run` als kanonische Run-Historie.
- **Konflikt:** global nur ein aktiver Run; 409 `analysis_busy`, außer derselbe Profil-Lauf läuft schon (200).
- **UI:** `ai-profiles-page.js` auf `/verwaltung/ai-profiles/`.
- **Prompt ändern:** UI pflegt nur **`user_prompt`** (Analyse-Prompt); Speichern setzt **`system_prompt`** leer. Platzhalter: `{{corpus}}`, `{{count}}`, `{{model}}`. Worker sendet an Ollama nur eine **System**-Nachricht, wenn `system_prompt` in der DB noch nicht leer ist (Legacy), sonst nur **User** mit `user_prompt`. Klassifizierung: Worker prüft `require_substring` (Standard `### Regelbasierte Muster`).
- **Maschinenlesbare Felder:** In `dls_email` (Layer-2) unverändert englische Tokens; Taxonomie-Markdown bleibt deutschsprachig wo vorgesehen.

## Checkliste: neues Profil / neuer Quelltyp

1. Optional `WritingStyleHistoryService`-Konstante nur wenn die Legacy-Historie einen neuen `analysis_type` braucht.
2. `dls_ai_profile`-Zeile (name + source_type) + Filter-JSON; bei neuem `source_type` Builder in `AiCorpusBuilderRegistry` registrieren.
3. `functions.php` — Route/Install nur wenn neue Endpoints oder Tabellen.
4. `dls_ai_profile_begin_run_for_profile_id` / Mutex beibehalten (ein globaler Worker).
