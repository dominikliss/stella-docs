# Cursor rule export: `email-ai-analysis.mdc`

_Source of agent rule in theme repo: `.cursor/rules/email-ai-analysis.mdc`._

---

# E-Mail-AI-Analysen (Theme)

Vollständige Architektur: **`architecture.mdc`** → Abschnitt **„E-Mail-AI-Analysen (Ollama, Verwaltung → AI-Analyse)“**.

## Kurzreferenz

| Thema | Schreibstil | Klassifizierung |
|--------|-------------|------------|
| `analysis_type` | `writing_style` | `classification_suggestions` |
| Corpus (`dls_email`) | `direction = outbound` | `direction = inbound` |
| Optionen-Präfix | `dls_writing_style_*` | `dls_email_classification_*` |
| Route-Datei | `inc/routes/ai-writing-style.php` | `inc/routes/ai-email-classification-suggestions.php` |
| Worker | `inc/scripts/learn-writing-style.php` | `inc/scripts/learn-email-classification-suggestions.php` |
| Letzte Ausgabe | `profile` + `final_grounding` | `json` (Roh-Markdown) + `final_grounding` Option `dls_email_classification_final_grounding`; aktiv: `dls_get_active_email_classification_grounding()` |

- **Historie:** eine Tabelle `dls_writing_style_run`, Spalte **`analysis_type`**; `WritingStyleHistoryService` mit 5. Parameter bei Log-Aufrufen aus Klassifizierungs-Worker/Route.
- **Konflikt:** nur ein Job aktiv; 409 wenn die andere Analyse noch `running`.
- **UI:** `ai-analysis-actions-page.js` / `email-analysis-panel.js` — `refreshEmailAnalysisJobs`, Select **Analyse-Typ**, getrennte Historie-Listen per `analysis_type`-Query.
- **Prompt ändern:** nur im jeweiligen Worker-Skript; Klassifizierung: Erfolg wenn **`### Regelbasierte Muster`** in der Antwort vorkommt.

## Checkliste: neue Analyse-Art (3. Typ)

1. Konstante in `WritingStyleHistoryService` + `sanitize_analysis_type()`.
2. Eigene WP-Optionen + REST-Datei (oder erweiterte generische Route) + CLI-Skript unter `inc/scripts/`.
3. `functions.php` — `require` der Route.
4. Start-Handler: prüfen, ob **andere** Analyse-Jobs `running` sind (409).
5. Historie: `log_success` / `log_error` mit korrektem `analysis_type`; UI-Query `analysis_type=…`.
6. Nach DB-Spaltenänderung: `install-writing-style-history.php` Version / dbDelta.
