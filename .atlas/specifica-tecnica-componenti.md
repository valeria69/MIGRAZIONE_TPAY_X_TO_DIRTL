---
unique-name: specifica-tecnica-componenti
display-name: Specifica Tecnica v4.0 — Componenti Migrazione TPAY-X → DIRTL
category: GENERAL
description: Specifica tecnica completa v4.0: 5 componenti (COMP-01→COMP-04 + tabelle), 11 domande aperte, architetture ASCII, interfacce, rischi. COMP-02 aggiornato: batch chiama servizio interno KTF, NON l'API REST pubblica.
tags: specifica-tecnica, v4-0, batch, migrazione, dirtl, tpay-x, componenti, ktf-servizio-interno, completo
---

# Specifica Tecnica — Componenti Migrazione TPAY-X → DIRTL

**Progetto:** MIGRAZIONE\_TPAY-X\_DIRTL  
**Data:** 2026-08-31  
**Versione:** 4.0 *(revisione completa da doc 31/08/2026 — campi esaustivi da* `migrazione_massiva_DIRTL_3.0.md`*)*  
**Fonti:** `migrazione_massiva_DIRTL_3.0.md`, `Migrazione forzata Tpay X - Sempre Facile.md`, `Chiusura massiva TPayX su base lista.md`

---

## ❓ DOMANDE APERTE — da risolvere prima del rilascio

> Le seguenti questioni sono irrisolte nelle fonti originali. Sono **bloccanti** salvo dove indicato.


| ID       | Domanda                                                                                                                                                                                                                          | Priorità | Fonte                                          |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------- |
| **D-01** | La tabella guida di partenza (T1) viene consegnata prima dell'avvio del batch o in corso d'opera? Se arriva dopo che lo STEP 0 è già partito, come va gestito il dato? Altrimenti la si rilegge ogni volta all'inizio del batch. | 🔴 ALTA  | `migrazione_massiva_DIRTL_3.0.md` §1           |
| **D-02** | La T1 è una vista DB? Se sì: quali sono le condizioni del filtro della vista?                                                                                                                                                    | 🔴 ALTA  | `migrazione_massiva_DIRTL_3.0.md` §1           |
| **D-03** | Dobbiamo aggiornare un flag sulla T1 per segnalare la presa in carico del record?                                                                                                                                                | 🟡 MEDIA | `migrazione_massiva_DIRTL_3.0.md` §1           |
| **D-04** | I TpayX arrivano già chiusi nella T1, oppure il batch li trova ancora aperti e deve chiuderli? Impatta sull'inserimento dello storico.                                                                                           | 🔴 ALTA  | `migrazione_massiva_DIRTL_3.0.md` §3           |
| **D-05** | Sullo storico va inserito anche il DIRFA (oltre al TpayX)?                                                                                                                                                                       | 🟡 MEDIA | `migrazione_massiva_DIRTL_3.0.md` §3           |
| **D-06** | Se sul contratto di origine sono presenti ordini in corso, il record va scartato? Con quale messaggio?                                                                                                                           | 🔴 ALTA  | `migrazione_massiva_DIRTL_3.0.md` §4.1         |
| **D-07** | La MUC (comunicazione) va inviata contestualmente allo STEP 0 oppure prima, in un processo separato?                                                                                                                             | 🔴 ALTA  | `Migrazione forzata Tpay X - Sempre Facile.md` |
| **D-08** | Nella migrazione da convenzionato a DIRFA ci sono scritture speciali su tabelle ETS/fatturazione? Verificare sul processo attuale.                                                                                               | 🟡 MEDIA | `Migrazione forzata Tpay X - Sempre Facile.md` |
| **D-09** | La migrazione dei titoli AT/AM la gestiscono i batch esistenti (`CreazioneTitoliPerMigrazione` + `MigrazioneTitoli`) in autonomia, oppure dobbiamo integrarla nel nostro processo batch?                                         | 🔴 ALTA  | `migrazione_massiva_DIRTL_3.0.md` §6           |
| **D-10** | Quali sono le precondizioni aggiuntive di migrazione che A.M. deve ancora completare (nota `A.M. Aggiungere altre precondizioni per migrazione` nel doc Sempre Facile)?                                                          | 🔴 ALTA  | `Migrazione forzata Tpay X - Sempre Facile.md` |
| **D-11** | Il nome definitivo della tabella guida batch: `TKSAMX_GUI_MIG_EVO_TL` (indicato come "provvisorio") — da confermare con il DBA.                                                                                                  | 🟡 MEDIA | `migrazione_massiva_DIRTL_3.0.md` §2 / §5      |


---

## COMP-01 — Batch Orchestratore `MIGRAZIONE_EVO_TL`

### Obiettivo

Eseguire in modalità massiva la migrazione di contratti Family (TpayX / DIRFA / Convenzionato) verso il prodotto **DIRTL (Sempre Facile)**, operando interamente su backend senza comunicazioni standard di upselling, simulando i processi KSA già esistenti per la migrazione da convenzionato o da DIRFA.

Il processo si articola in **sei step sequenziali** per ogni contratto, governati dallo stato di elaborazione scritto sulla tabella guida batch (`TKSAMX_GUI_MIG_EVO_TL`).

---

### Architettura

```
[T1 - Tabella guida di partenza]
          │
          ▼
    STEP 0: Invio MUC
    ┌─────────────────────────────────────────────────────────┐
    │ - Invia comunicazione MUC al cliente (via Magnews)      │
    │ - Inserisce riga su TKSAMX_GUI_MIG_EVO_TL               │
    │   (C_STA_ELA=0, D_NEW_ELA = now + 60gg)                │
    └─────────────────────────────────────────────────────────┘
          │ (attesa 60gg)
          ▼
    STEP 1: Chiusura TpayX
    ┌─────────────────────────────────────────────────────────┐
    │ - Raccoglie batch di codiciUtente (max 200)             │
    │ - Chiama KTF servizio interno di chiusura TpayX         │
    │ - Esiti positivi → proseguono a STEP 2                  │
    │ - Esiti negativi → C_STA_ELA=-1, T_MOT_SCA compilato   │
    └─────────────────────────────────────────────────────────┘
          │
          ▼
    STEP 2: Creazione DIRTL  [TRANSAZIONE UNICA]
    ┌─────────────────────────────────────────────────────────┐
    │ - Invoca logica KSA apertura DIRTL scenario migrazione  │
    │ - Genera nuovi codici contratto DIRTL (+ eventuale DIRFA│
    │   se partenza da convenzionato)                         │
    │ - Imposta IBAN dal contratto family di origine          │
    │ - Imposta C_SETIF = CGBUU (SEPA)                        │
    │ - Inserisce eventi 1A, 1C (gestore 80, user=batch)      │
    │ - Indirizzi contratto presi dal contratto family        │
    │   → se non presenti: BLOCCO con T_MOT_SCA               │
    │ - Inserisce riga su TETSOR con dati contratto X         │
    │ - Ente garante override = 40 su DIRFA aperto            │
    │ - NO inserimento consensi su TETS4S                     │
    │ - Salva C_CTR_EVO + contratto associato sulla T2        │
    │ - Contratto nasce in stato 09                           │
    └─────────────────────────────────────────────────────────┘
          │
          ▼
    STEP 3: Firma tecnica  [STESSA TRANSAZIONE O SEPARATA]
    ┌─────────────────────────────────────────────────────────┐
    │ - Porta contratto in stato 10 (firmato)                 │
    │ - Inserisce evento 1D (gestore 80, user=batch)          │
    │ - Inserisce evento 1F direttamente (NO SEDA/check IBAN) │
    │ - Inserisce evento 1E direttamente (NO PEGA / TKSAA1)   │
    │ - NO scrittura su TETSXM                                │
    │ - NO comunicazioni email/sms standard                   │
    └─────────────────────────────────────────────────────────┘
          │
          ▼
    STEP 4: Apertura contratto
    ┌─────────────────────────────────────────────────────────┐
    │ - Inserisce evento 1H → contratto APERTO                │
    │ - NO welcome mail standard                              │
    │ - Invia comunicazione ad hoc via Magnews                │
    │ - Inserisce riga su TKSANN_OPE_ONG                      │
    │   (operazione "fake" già scaduta, tipo fittizio)        │
    │ - Inserisce riga su TKSAU9_CTR_MIG_TIT                  │
    │   (stato contratto nuovo = APERTO)                      │
    └─────────────────────────────────────────────────────────┘
          │
          ▼
    STEP 5: Migrazione titoli AT/AM
    ┌─────────────────────────────────────────────────────────┐
    │ - Titoli SM sul contratto di origine → CHIUSI           │
    │ - Inserisce righe su tabelle guida migrazione titoli    │
    │   (TKSAU9_CTR_MIG_TIT, TKSAV0)                         │
    │ - Batch esistenti elaborano autonomamente:              │
    │   1. CreazioneTitoliPerMigrazione                       │
    │   2. MigrazioneTitoli                                   │
    └─────────────────────────────────────────────────────────┘
```

---

### Precondizioni di scarto (STEP 0 — controlli su T1)

Ogni record della T1 viene validato prima dell'elaborazione. In caso di fallimento la riga viene marcata con `C_STA_ELA = -1` e il motivo viene scritto in `T_MOT_SCA`.


| Controllo                                                    | Riferimento DB                          | Messaggio `T_MOT_SCA`                                                |
| ------------------------------------------------------------ | --------------------------------------- | -------------------------------------------------------------------- |
| Contratto base non aperto                                    | stato contratto                         | `Contratto base non aperto`                                          |
| Contratto base non è FA convenzionato né DIRFA               | tipo prodotto                           | `Contratto base non compatibile`                                     |
| Contratto base ha contratto associato ma `C_CTR_TPAY = null` | `ksaa.VKSA1B_COD_CTR`                   | `Dati non congruenti, contratto base con un contratto associato`     |
| TpayX indicato non è associato al contratto base in T1       | `ksaa.VKSA1B_COD_CTR`, col. `C_CTR_ASS` | `Dati non congruenti, contratto non associato al contratto indicato` |
| Contratto Tpay non è TpayX (Tpay Evo)                        | tipo contratto Tpay                     | `Contratto Tpay non è TpayX`                                         |
| Contratto Family ha Tpay associato su `TETS1B`               | `TETS1B`                                | `Contratto Family con Tpay associato, non migrabile`                 |
| Ordini in corso sul contratto di origine                     | ordini attivi                           | `Presenza ordini in corso` *(⚠️ D-06 aperta)*                        |
| Nessun titolo migrabile (AT/AM)                              | titoli contratto                        | `Non ci sono titoli migrabili`                                       |


---

### Interfacce


| Direzione     | Sistema                             | Dettaglio                                                   |
| ------------- | ----------------------------------- | ----------------------------------------------------------- |
| Lettura       | T1 (tabella guida di partenza)      | `SELECT * FROM <T1> WHERE ??` *(⚠️ D-01/D-02)*              |
| Scrittura     | `KSAA.TKSAMX_GUI_MIG_EVO_TL`        | Tabella batch propria del processo                          |
| Chiamata      | KTF servizio interno chiusura TpayX | STEP 1 — chiusura TpayX                                     |
| Scrittura DB  | `KSAA.TETSOR`                       | Dati contratto X                                            |
| Scrittura DB  | `KSAA.TKSANN_OPE_ONG`               | Tracciamento operazione "fake"                              |
| Scrittura DB  | `KSAA.TKSAU9_CTR_MIG_TIT`           | Record per migrazione titoli                                |
| Chiamata      | Magnews                             | Invio MUC (STEP 0) + comunicazione ad hoc apertura (STEP 4) |
| Scrittura DB  | `KSAA.TKSAV0`                       | (tramite batch titoli)                                      |


---

### Dipendenze

- **KTF** — `ServiceBloccaPyngMassivo` (COMP-02)
- **KSA** — Servizio apertura DIRTL scenario migrazione (COMP-03)
- **Magnews** — Invio comunicazioni MUC e ad hoc
- **Batch titoli** — `CreazioneTitoliPerMigrazione` + `MigrazioneTitoli` (COMP-04)
- **DB KSAA** — Tabelle: `TKSAMX_GUI_MIG_EVO_TL`, `TETSOR`, `TETS4S`, `TETSXM`, `TETS1B`, `TKSANN_OPE_ONG`, `TKSAU9_CTR_MIG_TIT`, `TKSAV0`
- **`VKSA1B_COD_CTR`** — Vista per validazione associazione contratti
- **`indirizzoContrattoSql.xml`** — Query per recupero indirizzi contratto

---

### Rischi


| ID   | Rischio                                                                                                                                 | Impatto  | Mitigazione                                                                       |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------------------------- |
| R-01 | Batch non idempotente: se il processo si interrompe a metà STEP 2 (transazione), il contratto DIRTL potrebbe essere creato parzialmente | 🔴 ALTO  | Gestione esplicita di rollback + marcatura stato su T2 prima di ogni operazione   |
| R-02 | Limite 200 TpayX per chiamata a KTF (STEP 1): lotti grandi richiedono più chiamate sequenziali                                          | 🟡 MEDIO | Chunking della lista in batch da 200 con gestione esiti parziali                  |
| R-03 | Bypass di SEDA/PEGA richiede modifiche chirurgiche al codice KSA esistente (STEP 3)                                                     | 🔴 ALTO  | Test su ambiente non-prod per verificare assenza scrittura TETSXM e comunicazioni |
| R-04 | Titoli SM non migrati: se la chiusura SM fallisce, il cliente paga fee su strisce blu su DIRTL                                          | 🔴 ALTO  | Verifica esplicita chiusura SM prima di procedere con migrazione titoli AT/AM     |
| R-05 | Temporizzazione MUC vs. STEP 0: se MUC non viene inviata prima del batch, il cliente non riceve comunicazione (⚠️ D-07)                 | 🟡 MEDIO | Chiarire timing con business prima del rilascio                                   |
| R-06 | La T1 (tabella partenza) non ha struttura definita: il batch non può essere completato senza di essa (⚠️ D-01/D-02/D-11)                | 🔴 ALTO  | Bloccare sviluppo STEP 0 finché struttura T1 non è definita                       |


---

## COMP-02 — KTF `ServiceBloccaPyngMassivo`

### Obiettivo

Il **batch orchestratore (COMP-01)** allo STEP 1 richiama il **servizio KTF di chiusura** per chiudere massivamente i contratti TpayX (Telepass Pay X / Tpay Evo). Non viene utilizzata l'API REST pubblica `/KTF/pyng/blocca-pyng-massivo-tpayx`, bensì il servizio interno di chiusura contratti.

---

### Architettura

```
COMP-01 (Batch) 
    │
    │ Chiama servizio interno KTF di chiusura
    │ (Non API REST /blocca-pyng-massivo-tpayx)
    │
    ▼
KTF – Servizio Chiusura Contratti
    │
    ├── Per ogni codiceUtente/C_CTR_ASS: chiude il contratto TpayX
    ├── Registra esito su tabella TETSOR (storico chiusure)
    └── Restituisce:
        ├── esitiPositivi: [{ codiceUtente }]
        └── esitiNegativi: [{ codiceUtente, esito (messaggio errore) }]
```

---

### Dipendenze

- **KTF** — microservizio interno Telepass Pay X
- **DB TpayX** — tabelle stato contratto Tpay

---

### Rischi


| ID   | Rischio                                                                                                                                                     | Impatto  | Mitigazione                                                            |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------- |
| R-07 | I TpayX devono essere in stato `01` (aperto) al momento della chiamata: se arrivano già chiusi il batch deve gestire l'esito come "già chiuso" e proseguire | 🟡 MEDIO | ⚠️ D-04: chiarire con business se i TpayX arrivano già chiusi nella T1 |


---

## COMP-03 — KSA Apertura DIRTL (scenario migrazione)

### Obiettivo

Creare il contratto **DIRTL (Sempre Facile)** a partire da un contratto Family (convenzionato o DIRFA), simulando — lato backend — il processo di cambio banca/migrazione già implementato in KSA. Il comportamento si differenzia in due path:

- **Path A — da convenzionato Family:** apre anche un DIRFA intermedio, poi il DIRTL.
- **Path B — da DIRFA:** apre direttamente il DIRTL e innesca migrazione titoli.

Il servizio opera in **transazione unica** per la sequenza creazione → firma → apertura.

---

### Architettura

```
COMP-01 invoca la logica KSA (non via REST API pubblica, ma 
tramite chiamata interna al servizio KSA)

Path A (da convenzionato):
  contratto convenzionato → nuovo DIRFA → nuovo DIRTL

Path B (da DIRFA):
  contratto DIRFA → nuovo DIRTL

In entrambi i path, la sequenza è:

  [1] Creazione contratto (stato 09)
      ├── IBAN dal contratto family di origine
      ├── C_SETIF = CGBUU (SEPA)
      │   Path A: applica CGBUU anche sul DIRFA associato
      ├── Evento 1A (gestore 80, user=utenza batch)
      ├── Evento 1C (gestore 80, user=utenza batch)
      ├── Indirizzi da contratto family → se assenti: BLOCCO
      ├── Riga su TETSOR con dati contratto TpayX
      ├── Ente garante override = 40 su DIRFA aperto
      └── NO consensi su TETS4S

  [2] Firma tecnica (stato 10)
      ├── Evento 1D (gestore 80, user=utenza batch)
      ├── Evento 1F (inserito direttamente — NO SEDA/check IBAN)
      ├── Evento 1E (inserito direttamente — NO PEGA / TKSAA1)
      ├── NO comunicazioni email/sms
      └── NO scrittura TETSXM

  [3] Apertura contratto
      ├── Evento 1H
      ├── NO welcome mail standard
      ├── Comunicazione ad hoc via Magnews (modello da definire)
      ├── Riga su TKSANN_OPE_ONG
      │   (tipo operazione fittizio "fake" già scaduta, D_SCA_OPE=now)
      └── Riga su TKSAU9_CTR_MIG_TIT (stato contratto nuovo = APERTO)
```

---

### Interfacce


| Operazione           | Tabella / Sistema             | Tipo             |
| -------------------- | ----------------------------- | ---------------- |
| Creazione contratto  | KSA interno                   | Scrittura DB     |
| IBAN                 | Contratto family origine      | Lettura          |
| Indirizzi contratto  | `indirizzoContrattoSql.xml`   | Lettura          |
| `TETSOR`             | Dati contratto TpayX          | Scrittura        |
| `TETS4S`             | Consensi (NON scrivere)       | —                |
| `TETSXM`             | Comunicazioni (NON scrivere)  | —                |
| `TKSANN_OPE_ONG`     | Tracciamento operazione       | Scrittura        |
| `TKSAU9_CTR_MIG_TIT` | Migrazione titoli             | Scrittura        |
| Magnews              | Comunicazione ad hoc apertura | Chiamata esterna |


---

### Dipendenze

- **KSA** — servizio di creazione contratti, callback Intesa
- **DB KSAA** — tutte le tabelle sopra elencate
- **Intesa** — backend firma contratto (da simulare, non invocare)
- **PEGA / TKSAA1** — allineamento documenti (da bypassare con evento 1E diretto)
- **SEDA** — check IBAN (da bypassare con evento 1F diretto)
- **Magnews** — comunicazioni ad hoc

---

### Rischi


| ID   | Rischio                                                                                                                   | Impatto  | Mitigazione                                                                    |
| ---- | ------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------ |
| R-10 | Il bypass di SEDA e PEGA richiede modifiche al codice esistente del servizio KSA: rischio di regressione su altri flussi  | 🔴 ALTO  | Isolare il flag "modalità batch" e testare estensivamente su ambiente non-prod |
| R-11 | La comunicazione ad hoc apertura non è ancora definita (modello Magnews da creare)                                        | 🟡 MEDIO | Bloccare STEP 4 finché il modello non è approvato da business                  |
| R-12 | Path A (da convenzionato): operazioni su tabelle ETS/fatturazione non documentate (⚠️ D-08)                               | 🟡 MEDIO | Analisi codice esistente del processo cambio banca convenzionato → DIRFA       |
| R-13 | Assenza indirizzi contratto sul family di origine → BLOCCO dell'intera transazione                                        | 🟡 MEDIO | Prefiltrare i contratti senza indirizzi durante lo STEP 0 (precondizioni)      |
| R-14 | Verifica assenza scrittura TETSXM deve essere validata su ambiente di test: rischio invio email accidentale in produzione | 🔴 ALTO  | Aggiungere test case esplicito per assenza record TETSXM dopo STEP 3           |


---

## COMP-04 — Gestione Titoli (`CreazioneTitoliPerMigrazione` + `MigrazioneTitoli`)

### Obiettivo

Trasferire i titoli **AT/AM** (Autovelox/Autostrade Manutenzione) dal contratto Family di origine al nuovo contratto DIRTL. I titoli **SM (Strisce Blu)** sul contratto di origine vanno **chiusi** (non migrati) per evitare l'applicazione delle fee su strisce blu sul nuovo DIRTL.

---

### Architettura

```
COMP-01 (STEP 5) scrive su TKSAU9_CTR_MIG_TIT:
  C_TIP_OPE_MIG = "01"  -- A_CONTRATTO_DIRETTO
  C_CTR_NEW     = nuovo codice contratto DIRTL
  C_UTE_NEW     = nuovo codice utente
  C_STA_CTR_NEW = 'APERTO'

          │
          ▼
Batch: CreazioneTitoliPerMigrazione
  - Legge TKSAU9 con C_STA_OPE = 'da elaborare'
  - Verifica titoli da migrare per ogni contratto
  - Per ogni titolo scrive su TKSAV0
  - Aggiorna TKSAU9: C_STA_OPE = 'Y'

          │
          ▼
Batch: MigrazioneTitoli
  - Legge TKSAV0 con C_STA_OPE == 'Y'
  - Per ogni riga su V0: migra il titolo

          │
          ▼
Titoli AT/AM presenti sul nuovo contratto DIRTL
```

**Aggiornamento T2 per fase di firma e apertura:**

Nella fase di inserimento contratto (STEP 2) viene inserita su `TKSAU9` una riga con:

- `C_STA_CTR_NEW = '09'`
- `C_STA_OPE = 'da elaborare'`

Nella fase di firma (STEP 3) viene aggiornata la riga con:

- `C_STA_CTR_NEW = '10'`

Nella fase di apertura (STEP 4) viene aggiornata la riga con:

- `C_STA_CTR_NEW = 'APERTO'`

Solo a questo punto i batch esistenti elaborano i titoli.

---

### Interfacce


| Tabella                   | Operazione                              | Chi scrive                     | Chi legge                      |
| ------------------------- | --------------------------------------- | ------------------------------ | ------------------------------ |
| `KSAA.TKSAU9_CTR_MIG_TIT` | Inserimento + aggiornamento progressivo | COMP-01 (STEP 2/3/4)           | `CreazioneTitoliPerMigrazione` |
| `KSAA.TKSAV0`             | Scrittura titoli da creare              | `CreazioneTitoliPerMigrazione` | `MigrazioneTitoli`             |


**Struttura richiesta su `TKSAU9` a fine STEP 4 (state target)**

```json
[
  {
    "C_TIP_OPE_MIG": "01",
    "C_CTR_NEW": "<nuovo codice contratto DIRTL>",
    "C_UTE_NEW": "<nuovo codice utente>",
    "C_STA_CTR_NEW": "APERTO"
  }
]
```

---

### Dipendenze

- **Batch `CreazioneTitoliPerMigrazione`** — batch esistente, non da modificare (⚠️ D-09: confermare)
- **Batch `MigrazioneTitoli`** — batch esistente, non da modificare (⚠️ D-09: confermare)
- **DB KSAA** — `TKSAU9_CTR_MIG_TIT`, `TKSAV0`

---

### Rischi


| ID   | Rischio                                                                                                                                    | Impatto  | Mitigazione                                                                  |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------ | -------- | ---------------------------------------------------------------------------- |
| R-15 | Titoli SM non chiusi prima della migrazione: il cliente paga fee su strisce blu su DIRTL                                                   | 🔴 ALTO  | Chiusura SM obbligatoria nello STEP 5 prima di qualunque scrittura su TKSAU9 |
| R-16 | Domanda aperta D-09: se i batch esistenti NON gestiscono autonomamente questa migrazione, tutto lo STEP 5 è da sviluppare ex-novo          | 🔴 ALTO  | Chiarire con team batch prima di pianificare                                 |
| R-17 | La sequenza `C_STA_CTR_NEW = 09 → 10 → APERTO` deve avvenire in ordine stretto; una discrepanza causa mancata elaborazione da batch titoli | 🟡 MEDIO | Lock ottimistico / controllo stato prima di ogni UPDATE                      |


---

## Tabelle Guida — Schema Completo

### T1 — Tabella guida di partenza (fornita da business)

> ⚠️ La struttura non è ancora definita. La tabella / vista viene fornita da business o da Operations e contiene la lista dei contratti da migrare.

**Contenuto atteso (da fonti):**

- `codiciContratto` del Tpay Evo (TpayX) — uno per riga
- In teoria presenti contratti DIRFA base + TpayX associato

**Query di lettura (da completare):**

```sql
SELECT *
FROM <TABELLA_GUIDA>  -- nome da confermare
WHERE ??????;         -- condizioni da definire (D-01 / D-02)
```

**Domande bloccanti:** D-01, D-02, D-03, D-11.

---

### T2 — `KSAA.TKSAMX_GUI_MIG_EVO_TL` — Tabella guida batch (da creare)

> Nome provvisorio — da confermare con DBA (⚠️ D-11).

Questa tabella è **proprietà del nostro batch** e viene creata per tracciare lo stato di ogni contratto durante l'elaborazione.


| Campo           | Tipo (suggerito) | NOT NULL    | Descrizione                                                                                         |
| --------------- | ---------------- | ----------- | --------------------------------------------------------------------------------------------------- |
| `ID_OPE_BATCH`  | `VARCHAR(64)`    | ✅ PK        | Identificativo univoco operazione batch *(timestamp ms, random UUID o incrementale — da scegliere)* |
| `C_CTP_CLI`     | `VARCHAR(20)`    | ✅           | Codice del cliente                                                                                  |
| `C_CTR_ORI`     | `VARCHAR(20)`    | ✅           | Codice contratto di origine (DIRFA o convenzionato)                                                 |
| `C_OLD_CTR_EVO` | `VARCHAR(20)`    | ❌           | Codice contratto TpayX (Evo) associato al contratto base                                            |
| `C_OLD_UTE_EVO` | `VARCHAR(20)`    | ❌           | Codice utente vecchio TpayX associato                                                               |
| `C_CTR_NEW`     | `VARCHAR(20)`    | ❌           | Codice contratto nuovo DIRTL (Sempre Facile) — valorizzato al STEP 2                                |
| `C_STA_ELA`     | `CHAR(2)`        | ✅           | Codice stato elaborazione (es. `0`=MUC inviata, `1`=TpayX chiuso, … `-1`=scartato)                  |
| `T_STA_ELA`     | `VARCHAR(500)`   | ❌           | Testo descrittivo stato elaborazione (es. *"Comunicazione MUC inviata al cliente, 60gg di attesa"*) |
| `D_INS`         | `TIMESTAMP`      | ✅           | Data/ora inserimento record nel batch                                                               |
| `D_NEW_ELA`     | `TIMESTAMP`      | ❌           | Data prossima elaborazione pianificata (es. `D_INS + 60gg` dopo STEP 0)                             |
| `N_RETRY`       | `INTEGER`        | ✅ DEFAULT 0 | Numero di retry effettuati dal batch sul record corrente                                            |
| `T_MOT_SCA`     | `VARCHAR(1000)`  | ❌           | Testo motivazione scarto applicativo (valorizzato se `C_STA_ELA = -1`)                              |
| `T_ECC`         | `VARCHAR(4000)`  | ❌           | Testo eccezione tecnica lanciata dal batch (per diagnostica)                                        |


**Valori `C_STA_ELA` (da confermare con team):**


| Valore | Significato                           |
| ------ | ------------------------------------- |
| `0`    | MUC inviata — in attesa 60 giorni     |
| `1`    | TpayX chiuso (STEP 1 OK)              |
| `2`    | DIRTL creato (STEP 2 OK)              |
| `3`    | Contratto firmato (STEP 3 OK)         |
| `4`    | Contratto aperto (STEP 4 OK)          |
| `5`    | Migrazione titoli avviata (STEP 5 OK) |
| `-1`   | Scartato — vedere `T_MOT_SCA`         |
| `-2`   | Errore tecnico — vedere `T_ECC`       |


**Primo inserimento sulla T2 (STEP 0):**


| Campo           | Valore                                                 |
| --------------- | ------------------------------------------------------ |
| `ID_OPE_BATCH`  | nuovo (generato)                                       |
| `C_CTP_CLI`     | codice cliente                                         |
| `C_CTR_ORI`     | codice contratto DIRFA                                 |
| `C_OLD_CTR_EVO` | codice contratto Evo associato                         |
| `C_STA_ELA`     | `0`                                                    |
| `T_STA_ELA`     | `Comunicazione MUC inviata al cliente, 60gg di attesa` |
| `D_INS`         | `current_timestamp`                                    |
| `D_NEW_ELA`     | `current_timestamp + 60 giorni`                        |


**Query recupero record da elaborare:**

```sql
SELECT *
FROM KSAA.TKSAMX_GUI_MIG_EVO_TL
WHERE C_STA_ELA = <stato_corrente>
  AND D_NEW_ELA <= current_timestamp
ORDER BY D_INS ASC;
```

---

### T3 — `KSAA.TKSAU9_CTR_MIG_TIT` — Tabella migrazione titoli (esistente)

Tabella gestita dai batch `CreazioneTitoliPerMigrazione` e `MigrazioneTitoli`. Il batch MIGRAZIONE\_EVO\_TL scrive qui il record trigger che attiva la catena.


| Campo           | Tipo          | Descrizione                                                     |
| --------------- | ------------- | --------------------------------------------------------------- |
| `C_TIP_OPE_MIG` | `VARCHAR(2)`  | Tipo operazione migrazione (`01` = A\_CONTRATTO\_DIRETTO)       |
| `C_CTR_NEW`     | `VARCHAR(20)` | Codice contratto nuovo DIRTL                                    |
| `C_UTE_NEW`     | `VARCHAR(20)` | Codice utente nuovo                                             |
| `C_STA_CTR_NEW` | `VARCHAR(20)` | Stato contratto nuovo (`09` → `10` → `APERTO`)                  |
| `C_STA_OPE`     | `CHAR(1)`     | Stato operazione per batch titoli (`Y` = pronto per migrazione) |


---

### T4 — `KSAA.TKSAV0` — Tabella titoli da migrare (esistente)

Tabella scritta da `CreazioneTitoliPerMigrazione` e letta da `MigrazioneTitoli`.


| Campo                                             | Tipo      | Descrizione                                          |
| ------------------------------------------------- | --------- | ---------------------------------------------------- |
| `C_STA_OPE`                                       | `CHAR(1)` | Stato operazione (`Y` = pronto per MigrazioneTitoli) |
| *(altri campi da documentare con il team titoli)* | —         | ⚠️ Schema completo da ricavare dal codice esistente  |


---

### T5 — `KSAA.TKSANN_OPE_ONG` — Tracciamento operazioni (esistente)

Usata allo STEP 4 per tracciare l'operazione di apertura contratto.


| Campo               | Descrizione                                                                          |
| ------------------- | ------------------------------------------------------------------------------------ |
| `D_SCA_OPE`         | Timestamp operazione (valorizzato a `current_timestamp`)                             |
| *(tipo operazione)* | Tipo fittizio da creare ad hoc per tracciare questi contratti *(valore da definire)* |


---

## Matrice di tracciabilità STEP → Tabelle


| STEP   | Tabella letta | Tabella scritta                     | Sistema chiamato                                         |
| ------ | ------------- | ----------------------------------- | -------------------------------------------------------- |
| STEP 0 | T1            | T2 (`TKSAMX`)                       | Magnews (MUC)                                            |
| STEP 1 | T2            | T2 (aggiornamento stato)            | KTF (servizio interno chiusura TpayX)                    |
| STEP 2 | T2            | T2, `TETSOR`                        | KSA (apertura DIRTL)                                     |
| STEP 3 | T2            | T2                                  | — (logica KSA interna)                                   |
| STEP 4 | T2            | T2, `TKSANN_OPE_ONG`, T3 (`TKSAU9`) | Magnews (comunicazione ad hoc)                           |
| STEP 5 | T2, T3        | T3, T4 (`TKSAV0`)                   | Batch `CreazioneTitoliPerMigrazione`, `MigrazioneTitoli` |


---

## Riepilogo domande aperte prioritizzate

```
🔴 BLOCCANTI (devono essere risolte prima del kick-off sviluppo):
  D-01 — Struttura e timing T1
  D-02 — T1 è una vista? Filtri?
  D-04 — TpayX arrivano chiusi o aperti?
  D-06 — Ordini in corso → scarto?
  D-07 — Timing MUC vs STEP 0
  D-09 — Batch titoli esistenti gestiscono autonomamente?
  D-10 — Precondizioni aggiuntive A.M.

🟡 IMPORTANTI (risolvere entro fine sprint 1):
  D-03 — Aggiornare flag T1 su presa in carico?
  D-05 — Storico include DIRFA?
  D-08 — Operazioni ETS/fatturazione su path convenzionato?
  D-11 — Nome definitivo T2
```
