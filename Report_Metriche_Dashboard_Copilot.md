# Report dettagliato metriche dashboard GitHub Copilot

Data report: 2026-02-02

## Scopo
Questo documento descrive **tutte le metriche presenti in ogni tab della dashboard**, indicando:
- **come vengono calcolate** (logica e formule),
- **quali campi legacy** vengono usati (se presenti),
- **come ricostruirle con le nuove API** (report endpoints).

> Riferimenti al codice: streamlit dashboard in `ea-gh-copilot-dashboard-streamlit.py`.

---

## 1) Overview
### Periodo e filtri
- Default: periodo termina **ieri**. Se “escludi weekend”, seleziona **ultimi N giorni lavorativi**.
- Se periodo personalizzato: usa le date esatte con conteggio giorni lavorativi opzionale.

### Metriche Generali
1) **Licenze Totali (di sempre)**
- **Calcolo**: conteggio licenze valide (attive + pending future), escludendo licenze con `pending_cancellation_date` nel passato.
- **Legacy**: nessun campo legacy specifico.
- **Nuove API**: **non disponibile** (serve il dataset “seats/billing”).

2) **Licenze Attive (di sempre)**
- **Calcolo**: licenze valide **senza** `pending_cancellation_date`.
- **Legacy**: nessun campo legacy specifico.
- **Nuove API**: **non disponibile**.

3) **Adozione Effettiva (di sempre)**
- **Calcolo**: utenti con licenza valida che hanno `last_activity_at` valorizzato (almeno un uso storico).
- **Legacy**: nessun campo legacy specifico.
- **Nuove API**: **non disponibile**.

4) **Licenze in Sospensione (mese attuale)**
- **Calcolo**: licenze pending con `pending_cancellation_date` compreso tra oggi e fine mese.
- **Legacy**: nessun campo legacy specifico.
- **Nuove API**: **non disponibile**.

### Utenti e adozione nel periodo
5) **Utenti Attivi (Xg)**
- **Calcolo**: utenti con attività nel periodo (dedup per login) usando dataset di attività giornaliera.
- **Legacy**: `total_active_users` (daily).
- **Nuove API**: somma di `daily_active_users` nel report **organization‑1‑day** oppure dedup per periodo su **users‑1‑day**.

6) **Utenti Senza Attività (Xg)**
- **Calcolo**: `licenze_totali - utenti_attivi`. Se preset 30g: usa inattivi >30g.
- **Legacy**: `total_active_users` e `total_engaged_users` (proxy).
- **Nuove API**: `users‑1‑day` per utenti attivi + denominatore seats (non disponibile nelle API).

7) **Tasso di Adozione (Xg)**
- **Calcolo**: `utenti_attivi / licenze_totali`.
- **Legacy**: `total_active_users / licenze`.
- **Nuove API**: `daily_active_users` + seats (manca denominatore).

8) **Media di utilizzo (Xg)**
- **Calcolo**: media giornaliera di `active_users` su `daily_metrics` (weekend esclusi se impostato).
- **Legacy**: `total_active_users` giornaliero.
- **Nuove API**: media di `daily_active_users` (organization‑1‑day).

### Classifiche
9) **Team con Maggior Engagement**
- **Calcolo**: per team `attivi_team / licenze_team`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile** (manca dimensione team/hub).

10) **Linguaggi più Utilizzati (Top 3)**
- **Calcolo**: top per `total_suggestions` da `language_usage_summary`.
- **Legacy**: `total_engaged_users` per lingua.
- **Nuove API**: `totals_by_language.*.code_generation_activity_count` (organization‑1‑day).

11) **Utenti per IDE (Top 3)**
- **Calcolo**: conteggio utenti attivi per `last_activity_editor`.
- **Legacy**: `copilot_ide_code_completions` (solo aggregato).
- **Nuove API**: `totals_by_ide` (non utenti unici).

### Grafici e breakdown
12) **Attività Giornaliera**
- **Calcolo**: serie `daily_metrics.active_users`.
- **Legacy**: `total_active_users` giornaliero.
- **Nuove API**: `daily_active_users` (organization‑1‑day).

13) **Top Linguaggi Programmazione (Top 10)**
- **Calcolo**: `total_code_acceptances + total_code_lines_accepted` per lingua, filtrando linguaggi “programmazione”.
- **Legacy**: non presente come campo singolo (deriva da code completion).
- **Nuove API**: `totals_by_language.*.code_acceptance_activity_count` + `loc_accepted` (organization‑1‑day).

14) **Distribuzione Editor**
- **Calcolo**: distribuzione `last_activity_editor` sugli utenti attivi nel periodo.
- **Legacy**: `copilot_ide_code_completions` (aggregato).
- **Nuove API**: `totals_by_ide` (non utenti unici).

### Hub/Team/Chat
15) **Hub Overview (membri/licenze/attivi)**
- **Calcolo**: membership team + seats + `last_activity_at`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

16) **Utilizzo Chat IDE/Web (conversazioni totali)**
- **Calcolo**: somma `total_chats` da `ide_chat_metrics` e `dotcom_chat_metrics`.
- **Legacy**: `copilot_ide_chat`, `copilot_dotcom_chat`.
- **Nuove API**: parziale via `totals_by_feature` (chat_*), **senza** split IDE/Web.

17) **Editor più Utilizzati per Chat**
- **Calcolo**: aggregazione per `editor` su `ide_chat_metrics`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

18) **Panoramica Team (membri/licenze/privacy)**
- **Calcolo**: `teams_members`, `teams`, `seats`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

---

## 2) Team Analytics
### Licenze e adozione (per team/hub/all users)
1) **Licenze Totali / Attive / Adozione Effettiva**
- **Calcolo**: come Overview ma con perimetro team/hub.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile** (manca team/hub + seats).

2) **Utenti Attivi (Xg) / Senza Attività (Xg)**
- **Calcolo**: `last_activity_at` rispetto al periodo per i membri del team.
- **Legacy**: `total_active_users` (org‑level).
- **Nuove API**: users‑1‑day dedup per periodo, ma **non** filtrabile per team.

3) **Licenze in Sospensione**
- **Calcolo**: pending future.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

### Attività Giornaliera (Team/Org/Hub)
4) **Serie giornaliera attivi**
- **Calcolo**: `daily_metrics_team` per team, oppure `daily_metrics` per org.
- **Legacy**: `total_active_users` giornaliero.
- **Nuove API**: solo org‑level via `daily_active_users`.

### Suggerimenti & Accettazione (perimetro team/hub)
5) **Tot. Suggerimenti**
- **Calcolo**: somma `total_code_suggestions` (fallback `total_suggestions_count`).
- **Legacy**: `copilot_ide_code_completions` (solo aggregato).
- **Nuove API**: `code_generation_activity_count` (org 1‑day).

6) **Tasso Medio**
- **Calcolo**: `acceptances / suggestions`.
- **Legacy**: non presente come campo singolo.
- **Nuove API**: `code_acceptance_activity_count / code_generation_activity_count`.

7) **Tot. Linee Suggerite**
- **Calcolo**: somma `total_code_lines_suggested`.
- **Legacy**: non presente.
- **Nuove API**: `loc_suggested`.

8) **% Medio Linee Accettate**
- **Calcolo**: `total_lines_accepted / total_lines_suggested`.
- **Legacy**: non presente.
- **Nuove API**: `loc_accepted / loc_suggested`.

Trend giornaliero: stesse metriche per data.

### Metriche aggiuntive
9) **Coverage Licenze**
- **Calcolo**: `licenze_attive / licenze_totali`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

10) **Tasso Attivazione**
- **Calcolo**: `ever_used / licenze_totali`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

11) **Distribuzione Editor**
- **Calcolo**: `last_activity_editor` sugli utenti attivi.
- **Legacy**: `copilot_ide_code_completions` (aggregato).
- **Nuove API**: `totals_by_ide` (non utenti unici).

12) **Export Dati Team**
- **Calcolo**: snapshot seats + teams + date fields.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

---

## 3) Language Trends
### Top Languages Dashboard
1) **Interazioni per linguaggio (Top 10)**
- **Calcolo**: somma `total_engaged_users` per lingua su `language_usage_summary` nel periodo.
- **Legacy**: `total_engaged_users` per lingua.
- **Nuove API**: `totals_by_language.*.code_generation_activity_count` (organization‑1‑day).

### Code Completion Efficiency by Language
2) **Tasso accettazione per lingua**
- **Calcolo**: `total_code_acceptances / total_code_suggestions`.
- **Legacy**: non presente come campo singolo.
- **Nuove API**: `code_acceptance_activity_count / code_generation_activity_count` per lingua.

3) **Tot. Suggerimenti**
- **Calcolo**: somma `total_code_suggestions`.
- **Legacy**: non presente.
- **Nuove API**: `code_generation_activity_count`.

4) **Tot. Linee Suggerite**
- **Calcolo**: somma `total_code_lines_suggested`.
- **Legacy**: non presente.
- **Nuove API**: `loc_suggested`.

5) **% Medio Linee Accettate**
- **Calcolo**: `total_code_lines_accepted / total_code_lines_suggested`.
- **Legacy**: non presente.
- **Nuove API**: `loc_accepted / loc_suggested`.

Trend giornaliero: aggregazioni per data delle stesse metriche.

---

## 4) Licenze & Billing
1) **Licenze Totali / Attive / % Attive**
- **Calcolo**: conteggio da seats con pending futuro valido.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

2) **Inattivi >30g**
- **Calcolo**: `last_activity_at` rispetto a cutoff 30g.
- **Legacy**: `total_active_users` (org‑level) solo come proxy.
- **Nuove API**: users‑1‑day dedup 30g (manca seats per denominatore).

3) **Licenze Aggiunte nel Periodo**
- **Calcolo**: `seat_created_at` nel periodo (dedup).
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

4) **Analisi Costi**
- **Calcolo**: pricing per `plan_type` × licenze valide.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

5) **Costo Inattivi >30g**
- **Calcolo**: `num_inactive_30d × costo_medio`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

---

## 5) User Deep Dive
1) **Giorni con Copilot**
- **Calcolo**: `now - seat_created_at`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

2) **Ultimo uso**
- **Calcolo**: massimo tra `last_activity_at` e ultime date su `ide_chat_metrics` / `dotcom_chat_metrics`.
- **Legacy**: `copilot_ide_chat`, `copilot_dotcom_chat` solo aggregati.
- **Nuove API**: users‑28‑day/latest fornisce totali 28g, **non** timestamp “ultimo uso”.

3) **Ultimo Editor Utilizzato**
- **Calcolo**: `last_activity_editor`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

4) **Stato Licenza**
- **Calcolo**: pending/attiva basato su `pending_cancellation_date`.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: **non disponibile**.

5) **Report Riepilogativo (export)**
- **Calcolo**: mix di `last_activity_at`, editor, stato licenza.
- **Legacy**: nessun campo legacy diretto.
- **Nuove API**: parziale con users‑1‑day/28‑day per conteggi (mancano editor/licenza).

---

## 6) Admin Utilities
Metriche interne (SSO, data quality, system health, audit) basate su DB interno dashboard e Graph/Token service.
- **Legacy / Nuove API**: non applicabili.

---

## Mappatura rapida verso le nuove API (report endpoints)
### Organization 1‑day
- `daily_active_users`
- `user_initiated_interaction_count`
- `code_generation_activity_count`
- `code_acceptance_activity_count`
- `loc_suggested`, `loc_accepted`
- `totals_by_ide` (breakdown IDE)
- `totals_by_feature` (chat/code completion/agent)
- `totals_by_language`
- `totals_by_model`

### Users 1‑day
- Stesse metriche per utente con `user_id`, `user_login`, `used_chat`, `used_agent`.

### Organization 28‑day latest
- `day_totals` + range `report_start_day` / `report_end_day`.

### Users 28‑day latest
- Totali per utente sui 28 giorni.

---

## Gap principali rispetto alle nuove API
- **Licensing/seats**: assenti (necessario dataset seats/billing).
- **Team/Hub**: assenti nelle nuove API.
- **Editor “ultimo uso”**: assente.
- **Chat IDE vs Web**: le nuove API non distinguono la superficie.
- **Utenti unici per IDE**: disponibili solo come conteggi per IDE, non utenti unici.

---

## Raccomandazioni operative
1) Continuare a usare le collections `billing_seats`, `teams`, `teams_members` per tutte le metriche licensing/team.
2) Usare **organization‑1‑day** per tutte le serie giornaliere e breakdown per feature/language/IDE.
3) Usare **users‑1‑day** per dedup utenti attivi nel periodo (quando serve utenti unici).
4) Usare **28‑day latest** per snapshot mensili, con attenzione a sovrapposizioni di periodo.

---

Fine report.
