# Guida completa alle metriche della dashboard GitHub Copilot

Data guida: 2026-02-02

## 0) Obiettivo e come usare questo documento
Questa guida è una **mappa operativa** di tutte le metriche presenti in ogni tab della dashboard. Per ciascuna metrica trovi:
1. **Definizione chiara**: cosa misura realmente.
2. **Formula/Logica**: come viene calcolata in pratica.
3. **Dataset usati oggi**: collection Mongo o campi legacy.
4. **EquivalentI nelle nuove API**: come ricostruirla dai report ufficiali Copilot.
5. **Limitazioni e gap**: dove non è possibile ricostruire esattamente la metrica con le nuove API.

Questa guida è pensata per essere **una vera procedura di migrazione**, non solo un elenco di campi.

> Codice sorgente dashboard: `ea-gh-copilot-dashboard-streamlit.py`.

---

## 1) Panorama dei dataset usati oggi
### 1.1 Collections principali (MongoDB)
- `billing_seats` / `seats`: base licenze (stato licenza, last_activity, editor, creazione seat).
- `daily_metrics`: utenti attivi giornalieri (org).
- `daily_metrics_team`: utenti attivi giornalieri (team).
- `code_completion_metrics` / `code_completion_metrics_team`: suggerimenti, accettazioni, linee.
- `language_usage_summary`: interazioni per linguaggio.
- `ide_chat_metrics`: conversazioni chat in IDE.
- `dotcom_chat_metrics`: conversazioni chat su GitHub.com.
- `teams` e `teams_members`: membership e privacy dei team.

### 1.2 Nuove API Copilot (report endpoints)
#### Organization 1-day
Contiene dati **giornalieri** aggregati per organizzazione:
- `daily_active_users`
- `user_initiated_interaction_count`
- `code_generation_activity_count`
- `code_acceptance_activity_count`
- `loc_suggested`, `loc_accepted`
- `totals_by_ide`
- `totals_by_feature`
- `totals_by_language`
- `totals_by_model`

#### Users 1-day
Contiene dati giornalieri **per utente**:
- stesse metriche dell’org, più: `user_id`, `user_login`, `used_chat`, `used_agent`.

#### Organization 28-day latest
Snapshot “rolling 28 giorni”:
- `day_totals` (totali 28g)
- `report_start_day`, `report_end_day`

#### Users 28-day latest
Totali per utente sugli ultimi 28 giorni.

---

## 2) Overview — guida dettagliata metriche
### 2.1 Periodo di analisi
- Default: termina **ieri**.
- Se “escludi weekend” è attivo, la finestra è sugli **ultimi N giorni lavorativi**.
- Se selezione personalizzata, la finestra è esattamente l’intervallo scelto.

### 2.2 Metriche Generali (licenze e adozione storica)
#### 2.2.1 Licenze Totali (di sempre)
- **Definizione**: numero complessivo di licenze valide (attive + pending future).
- **Formula**: count(seats) dove `pending_cancellation_date` è assente oppure nel futuro.
- **Dataset attuale**: `billing_seats`/`seats`.
- **Legacy**: nessun campo legacy specifico.
- **Nuove API**: **non disponibile**.
- **Note**: è metrica “licensing”, non ricostruibile dai report di utilizzo.

#### 2.2.2 Licenze Attive (di sempre)
- **Definizione**: licenze operative senza cancellazione pendente.
- **Formula**: count(seats) dove `pending_cancellation_date` è nullo.
- **Dataset attuale**: `billing_seats`/`seats`.
- **Nuove API**: **non disponibile**.

#### 2.2.3 Adozione Effettiva (di sempre)
- **Definizione**: utenti che hanno **mai** usato Copilot.
- **Formula**: count(seats validi) con `last_activity_at` valorizzato.
- **Dataset attuale**: `billing_seats`/`seats`.
- **Nuove API**: non disponibile (le nuove API non hanno “ultimo uso” per utente).

#### 2.2.4 Licenze in Sospensione (mese attuale)
- **Definizione**: licenze con cancellazione programmata entro fine mese.
- **Formula**: count(seats) con `pending_cancellation_date` tra oggi e ultimo giorno del mese.
- **Nuove API**: non disponibile.

### 2.3 Metriche periodo (adozione operativa)
#### 2.3.1 Utenti Attivi (Xg)
- **Definizione**: utenti che hanno utilizzato Copilot almeno una volta nel periodo.
- **Formula**:
	- con Mongo: dedup logins in `daily_metrics` per date nel periodo;
	- con nuove API: somma `daily_active_users` (org 1-day) oppure dedup `user_id` in users 1-day.
- **Legacy**: `total_active_users`.
- **Nuove API**: `daily_active_users` (org 1-day) o dedup in users 1-day.

#### 2.3.2 Utenti Senza Attività (Xg)
- **Definizione**: licenze valide che non hanno uso nel periodo.
- **Formula**: `licenze_totali - utenti_attivi` (eccezione: preset 30g usa inattivi >30g).
- **Legacy**: nessun campo singolo, proxy con `total_active_users`.
- **Nuove API**: necessita licenze totali (non disponibili in API).

#### 2.3.3 Tasso di Adozione (Xg)
- **Definizione**: quota licenze che risultano attive nel periodo.
- **Formula**: `utenti_attivi / licenze_totali`.
- **Legacy**: proxy con `total_active_users`.
- **Nuove API**: richiede licenze totali (mancano).

#### 2.3.4 Media di utilizzo (Xg)
- **Definizione**: media giornaliera utenti attivi nel periodo.
- **Formula**: media(`daily_metrics.active_users`) nel periodo (weekend esclusi se impostato).
- **Legacy**: `total_active_users` giornaliero.
- **Nuove API**: media di `daily_active_users`.

### 2.4 Classifiche e confronto
#### 2.4.1 Team con Maggior Engagement
- **Definizione**: team con maggiore quota di attivi rispetto alle licenze del team.
- **Formula**: `team_active_users / team_total_licenses`.
- **Dataset attuale**: `teams_members`, `seats`, logica `last_activity_at`.
- **Nuove API**: **non disponibile** (manca dimensione team/hub).

#### 2.4.2 Linguaggi più Utilizzati (Top 3)
- **Definizione**: linguaggi con maggiore volume di suggerimenti.
- **Formula**: top per `total_suggestions` in `language_usage_summary`.
- **Legacy**: `total_engaged_users` per lingua.
- **Nuove API**: `totals_by_language.*.code_generation_activity_count`.

#### 2.4.3 Utenti per IDE (Top 3)
- **Definizione**: stima utenti attivi per editor.
- **Formula**: conteggio utenti con `last_activity_editor` nel periodo.
- **Legacy**: `copilot_ide_code_completions` (aggregato, non utenti unici).
- **Nuove API**: `totals_by_ide` (non utenti unici).

### 2.5 Grafici e breakdown
#### 2.5.1 Attività Giornaliera
- **Definizione**: utenti attivi per giorno.
- **Formula**: serie `daily_metrics.active_users`.
- **Legacy**: `total_active_users` giornaliero.
- **Nuove API**: `daily_active_users`.

#### 2.5.2 Top Linguaggi Programmazione (Top 10)
- **Definizione**: linguaggi con più interazioni “code acceptance/lines”.
- **Formula**: per lingua `total_code_acceptances + total_code_lines_accepted`.
- **Nuove API**: `totals_by_language` + `loc_accepted`.

#### 2.5.3 Distribuzione Editor
- **Definizione**: quota utenti attivi per editor.
- **Formula**: distribuzione `last_activity_editor` sugli attivi.
- **Nuove API**: `totals_by_ide` (non utenti unici).

### 2.6 Hub/Team/Chat
#### 2.6.1 Hub Overview
- **Definizione**: licenze e attivi per hub geografico.
- **Formula**: membership hub + seats + `last_activity_at`.
- **Nuove API**: **non disponibile**.

#### 2.6.2 Utilizzo Chat IDE/Web
- **Definizione**: conversazioni chat avviate in IDE e web.
- **Formula**: `total_chats` su `ide_chat_metrics` e `dotcom_chat_metrics`.
- **Legacy**: `copilot_ide_chat`, `copilot_dotcom_chat`.
- **Nuove API**: parziale via `totals_by_feature` (chat_*), senza split IDE/Web.

#### 2.6.3 Editor più Utilizzati per Chat
- **Definizione**: editor con più conversazioni.
- **Formula**: group by `editor` in `ide_chat_metrics`.
- **Nuove API**: **non disponibile**.

#### 2.6.4 Panoramica Team (membri/licenze/privacy)
- **Definizione**: profilo dei team.
- **Formula**: joins `teams` + `teams_members` + seats.
- **Nuove API**: **non disponibile**.

---

## 3) Team Analytics — guida dettagliata
### 3.1 Licenze e adozione per team/hub
- **Licenze Totali / Attive / Adozione Effettiva**: stesse logiche di Overview, ma applicate al perimetro selezionato.
- **Utenti Attivi / Senza Attività**: si basa su `last_activity_at` dei membri.
- **Licenze in Sospensione**: pending future.
- **Nuove API**: nessuna di queste è calcolabile per team/hub (mancano chiavi team).

### 3.2 Attività Giornaliera
- **Definizione**: utenti attivi giorno per giorno nel perimetro.
- **Formula**: `daily_metrics_team` o `daily_metrics`.
- **Nuove API**: solo org-level; non per team/hub.

### 3.3 Suggerimenti & Accettazione
- **Tot. Suggerimenti**: somma `total_code_suggestions`.
- **Tasso Medio**: `acceptances / suggestions`.
- **Tot. Linee Suggerite**: somma `total_code_lines_suggested`.
- **% Linee Accettate**: `total_lines_accepted / total_lines_suggested`.
- **Nuove API**: disponibili solo a livello organizzazione via `code_generation_activity_count`, `code_acceptance_activity_count`, `loc_suggested`, `loc_accepted`.

### 3.4 Metriche aggiuntive
- **Coverage Licenze**: `licenze_attive / licenze_totali`.
- **Tasso Attivazione**: `ever_used / licenze_totali`.
- **Distribuzione Editor**: `last_activity_editor`.
- **Export**: snapshot seat con team.
- **Nuove API**: non disponibili.

---

## 4) Language Trends — guida dettagliata
### 4.1 Top Languages Dashboard
- **Interazioni per linguaggio**: somma `total_engaged_users` per lingua.
- **Nuove API**: `totals_by_language.*.code_generation_activity_count`.

### 4.2 Code Completion Efficiency by Language
- **Tasso accettazione**: `total_code_acceptances / total_code_suggestions`.
- **Tot. Suggerimenti**: somma `total_code_suggestions`.
- **Tot. Linee Suggerite**: somma `total_code_lines_suggested`.
- **% Linee Accettate**: `total_code_lines_accepted / total_code_lines_suggested`.
- **Trend giornaliero**: aggregazioni per data con gli stessi campi.
- **Nuove API**: equivalenze dirette in `code_generation_activity_count`, `code_acceptance_activity_count`, `loc_suggested`, `loc_accepted` per lingua.

---

## 5) Licenze & Billing — guida dettagliata
### 5.1 Stato licenze
- **Licenze Totali / Attive / % Attive**: basate su `pending_cancellation_date`.
- **Inattivi >30g**: `last_activity_at` < cutoff.
- **Licenze aggiunte**: `seat_created_at` nel periodo.
- **Nuove API**: non disponibili (manca dataset licenze).

### 5.2 Analisi costi
- **Costo mensile/annuale**: licenze valide × prezzo piano.
- **Costo inattivi**: inattivi >30g × costo medio.
- **Nuove API**: non disponibili.

---

## 6) User Deep Dive — guida dettagliata
- **Giorni con Copilot**: `now - seat_created_at`.
- **Ultimo uso**: max tra `last_activity_at` e ultime date chat.
- **Ultimo editor**: `last_activity_editor`.
- **Stato licenza**: pending o attiva.
- **Nuove API**: disponibili solo totali (28g), non “ultimo uso” o editor.

---

## 7) Admin Utilities
Metriche di sistema (SSO, data quality, audit) basate su DB interno e Graph.
- **Legacy / Nuove API**: non applicabili.

---

## 8) Linee guida pratiche di migrazione (step-by-step)
1. **Sostituisci i grafici giornalieri** con `organization‑1‑day`.
2. **Per utenti unici nel periodo** usa `users‑1‑day` e dedup per `user_id`.
3. **Per breakdown IDE/Feature/Language/Model** usa `totals_by_*` di org 1‑day.
4. **Mantieni le metriche licensing** su `billing_seats` finché le nuove API non esporranno i seats.
5. **Per team/hub** continua con dataset interni (mancano nelle nuove API).
6. **Per Chat IDE/Web** conserva le collections chat; nuove API non distinguono la superficie.

---

## 9) Gap principali (da comunicare chiaramente)
- Nessuna API per licenze/seats.
- Nessuna dimensione team/hub.
- Nessun “ultimo editor” per utente.
- Chat IDE vs Web non distinguibile.
- Utenti unici per IDE non disponibili.

---

Fine guida.
