---
unique-name: migrazionemassivadirtl30
display-name: migrazione massiva DIRTL 3.0
category: GENERAL
description: Tabella guida con lista di codici contratto generica:
---

# Migrazione massiva DIRTL

## Regole generali

Tabella guida con lista di codici contratto generica:

- Se **DIRFA convenzionato o diretto** → migra a **DIRTL**.
- Se **DIRFA con un Tpay NON X collegato** → salta.
- Se **DIRFA con Tpay X** → chiudo **Tpay X** e migro a **DIRTL**.

---

# 1. Tabella guida - Partenza

Tabella guida di partenza per il batch *(ce la devono dare)*.  
In teoria dovrebbero esserci `codiciContratto` del **Tpay Evo (TpayX)**.

> **Nota:** la tabella guida che ci viene data dopo che abbiamo iniziato l'elaborazione con lo **STEP 0**, come viene gestito il dato?  
> Altrimenti ogni volta la rileggiamo all'inizio del batch.

### Select dalla tabella guida

```sql
SELECT *
FROM TABELLA_GUIDA
WHERE ??????;
```

> **DOMANDA:** se è una vista, quali sono le condizioni della vista?  
>
> **DOMANDA:** dobbiamo aggiornare il dato della vista per dire che l'abbiamo presa in carico in qualche modo?

## STEP preliminare - Recupero e controlli contratti

### 0. Ricavere tutti i dati dei contratti utili

Contratti interessati:

- `DIRFA`
- `TpayX`

### Controlli sul contratto base

- Contratto base sia aperto  
  → `T_MOT_SCA` → `Contratto base non aperto`

- Contratto base sia **FA convenzionato** o **DIRFA**  
  → `T_MOT_SCA = Contratto base non FA convenzionato o DIRFA`

- Contratto base ha un contratto associato && contratto Tpay == `null`  
  → `T_MOT_SCA = Dati non congruenti, contratto base con un contratto associato`  
  Riferimento: `ksaa.VKSA1B_COD_CTR`, colonna `C_CTR_ASS`

### Controlli sul contratto TpayX, se presente

- Contratto TpayX associato al contratto base relativo  
  → `T_MOT_SCA = Dati non congruenti, contratto non associato al contratto indicato`  
  Riferimento: `ksaa.VKSA1B_COD_CTR`, colonna `C_CTR_ASS`

- Contratto Tpay != Tpay Evo (`TpayX`)  
  → `T_MOT_SCA = Contratto non TpayX, non gestito dal batch`

### Controlli operativi

- Contratto Tpay == Tpay Evo (`TpayX`) && Tpay aperto  
  → chiusura del TpayX  
  → vedere servizio di chiusura TpayX massivo `ServiceBloccaPyngMassivo`

- Contratto Tpay == Tpay Evo (`TpayX`) && Tpay chiuso  
  → skip chiusura  
  → continua verso migrazione contratto base

Per verificare se il contratto Tpay è **Tpay Evo (`TpayX`)**, confrontare `C_MOD` del contratto Tpay con la lista `MODELLI_TPAY_EVO`.

A questo punto, superati tutti i controlli possibili, iniziare con la migrazione del contratto base al **DIRTL (Sempre Facile)**.

---

# 2. Tabella guida batch elaborazione

Leggere sulla nuova tabella guida batch, provvisoriamente:

`TKSAMX_GUI_MIG_EVO_TL`

e verificare se esiste già un record con quel contratto di partenza:

- se esiste → recuperarlo e capire a quale stato si trova per elaborarlo;
- se non esiste → partire con lo **STEP 0**.

### Prima query sulla nostra tabella guida

```sql
SELECT *
FROM ksaa.tksamx
WHERE c_ctr_ori = 'dirfa'
  AND c_ctr_ass = 'tpayx';
```

## STEP 0 - Invio comunicazione MUC

Operazioni:

- invio comunicazione **MUC**;
- aggiornamento `D_NEW_ELA + 60gg`.

### Primo inserimento sulla tabella guida del batch

Tabella provvisoria: `TKSAMX`

| Campo | Valore / significato |
|---|---|
| `ID_OPE_BATCH` | nuovo |
| `C_CTP_CLI` | codice cliente |
| `C_CTR_ORI` | codice contratto DIRFA |
| `C_CTR_EVO` | codice contratto Evo associato |
| `C_STA_ELA` | `0` |
| `T_STA_ELA` | `Comunicazione MUC inviata al cliente, 60gg di tempo` |
| `D_ULT_ELA` | current timestamp |
| `D_NEW_ELA` | current timestamp + 60gg |
| `C_USR` | utenza applicativa |

---

# 3. Chiusura TpayX

## STEP 1 - Chiusura dei TpayX

### Query per recuperare la tabella guida da elaborare

```sql
SELECT *
FROM ksaa.tksamx
WHERE D_NEW_ELA <= CURRENT TIMESTAMP
  AND C_STA_ELA NOT IN ('elaborato', 'scartato');
```

### 1.a Storico del TpayX

Tenere traccia dei dati del TpayX appena chiuso tramite inserimento sullo storico.

> **DOMANDA:** i TpayX ci arrivano già chiusi o da chiudere? Da capire per inserimento dello storico.  
>
> **DOMANDA:** sullo storico va inserito anche il DIRFA che viene migrato a DIRTL?  
>
> **DOMANDA:** il DIRFA lo chiudiamo noi? Come si chiude?

### 1.b Inserimento su TETSOR

Creare una riga sulla `TETSOR` con i dati del contratto TpayX.

Riferimento:

`ksaa.vksaor_sot_ctr`

---

# 4. Migrazione DIRTL

## STEP 2 - Migrazione del contratto base

Migrazione del contratto base del TpayX (`DIRFA`) a **Sempre Facile**.

## Creazione DIRTL

### 1. Apertura del nuovo contratto DIRTL

Eseguire le operazioni effettuate nel servizio KSA che apre i contratti DIRTL:

- Classe: `ServiceNuovoContrattoPerCambioBanca.java`
- `serviceCode = "S0139"`

Operazioni:

1. **Verifica assenza ordini in corso sul contratto origine**
   - Se sono presenti ordini in corso, va scartato?
   - `T_MOT_SCA` → `Presenza ordini in corso`

2. Eseguire:

   ```java
   aperturaContratto.verificaPossibilitaMigrazioneDirTL(controller, contrattoOrigine);
   ```

   - `T_MOT_SCA` → `Non ci sono titoli migrabili`

3. Generare i nuovi codici contratto:

   ```java
   generaCodiciContratto.generaNuoviCodiciContratto(controller);
   ```

4. Eseguire:

   ```java
   aperturaContratto.aggiornaOperazioneGaranzia(
   ```

5. Eseguire / spacchettare:

   ```java
   NuovoContrattoFamilyGarantitoAzioni
       .getInstance()
       .registraNuovoContrattoFamilyGarantito(..., Modello Sempre Light)
   ```

   Da **SPACCHETTARE** per gli inserimenti necessari *(indirizzo, eventi, ecc.)*.

   Operazioni specifiche:

   - **a.** Settare l'IBAN del contratto DIRTL con quello ricavato dal contratto base.
   - **b.** Settare il codice SIA/SETIF (`C_SETIF`) a `CGBUU (SEPA)` perché già validato, poiché già cliente.
   - **c.** Inserire nella `VKSAF6_TIP_EVE_CTR` gli eventi:
     - `1A` → `OK ente garante`
     - `1C` → `attesa di firma`
     - `c_ges = 80`
     - `c_usr` dell'applicativo (`KSA`)
   - **d.** Gli indirizzi contratto dovranno essere presi dagli indirizzi contratto del contratto Family.
     - Se non presenti → bloccare il batch.
     - Cercare la tabella degli indirizzi.
     - Riferimento: `indirizzoContrattoSql.xml`
   - **f.** Il nuovo contratto DIRTL deve nascere con ente garante override `40`.
     - Riferimento: colonna `c_ctp_con` della `ksaa.VKSA27_CTR.C_MOD`

6. **Altri punti da fare**
   - **g.** NON inserire nessun consenso sulla `tets4s`.

### 2. Firma tecnica del contratto

Firmare **tecnicamente** il contratto — o i contratti nel caso si sia partiti da un convenzionato — portandolo in stato firmato:

`stato 10`

> **DOMANDA:** perché "i contratti" in caso di convenzionato?  
> Vedere `S0100 KSA firmaContratto`.

Operazioni:

- `NuovoContrattoFamilyGarantitoAzioni.registraFirmaContratto`
  - contiene inserimento evento `1D FIRMA_CA`

- `NuovoContrattoFamilyGarantitoAzioni.registraConfermaSEDA`
  - contiene inserimento evento `1F CONFERMA_SEDA`

- `NuovoContrattoFamilyGarantitoAzioni.registraControlloDocumenti`
  - contiene inserimento evento `1E CONTROLLO_DOCUMENTI`

- `NuovoContrattoFamilyGarantitoAzioni.registraRiconoscimentoCompleto`
  - contiene inserimento evento `1H RICONOSCIMENTO_COMPLETO`

- Utilizzare:

  ```java
  inserimentoDocumentoBenvenutoDaShopOnLine(
      IPersistenceController controller,
      Contratto contratto,
      String codiceModello,
      boolean migrazioneInCorso
  )
  ```

  con modello di comunicazione ad hoc fornito.

- Inserimento su `KSAA.TKSANN_OPE_ONG`
  - operazione `"fake"` già scaduta;
  - indica l'operazione effettuata.

- Inserimento sulla tabella di migrazione:
  - `KSAA.TKSAU9_CTR_MIG_TIT`
  - da elaborare;
  - stato contratto nuovo: **APERTO**.

---

# 5. Possibile tabella guida del batch

Possibile tabella da creare:

`KSAA.TKSAMX_GUI_MIG_EVO_TL`

| Campo | Descrizione |
|---|---|
| `ID_OPE_BATCH` | identificativo operazione *(timestamp millisecondi, random UUID, incrementale, ecc.)* |
| `C_CTP_CLI` | codice del cliente |
| `C_CTR_ORI` | codice contratto base/origine |
| `C_OLD_UTE_ORI` | codice utente vecchio contratto base/origine |
| `C_CTR_EVO` | codice contratto Tpay associato |
| `C_OLD_UTE_EVO` | codice utente vecchio TpayX associato |
| `C_CTR_NEW` | codice contratto nuovo Sempre Light |
| `C_OLD_UTE_NEW` | codice utente vecchio nuovo contratto Sempre Light |
| `N_TIT_MIG` | numero dei titoli da migrare |
| `C_STA_ELA` | codice stato elaborazione attuale *(ad hoc per il batch - step principali)* |
| `C_STA_TEC` | codice stato tecnico dello step principale *(es. `GUIDA_MIGRAZIONE_TITOLI_INSERITA = 11`)* |
| `T_STA_TEC` | dettaglio testuale dello stato tecnico attuale *(es. `Inserita migrazione titoli corretamente`)* |
| `D_ULT_ELA` | data ultima elaborazione |
| `D_NEW_ELA` | data ultima elaborazione + 1gg |
| `N_RETRY` | retry effettuati dal batch |
| `T_MOT_SCA` | testo motivazione scarto applicativo del record |
| `T_ECC` | testo eccezione lanciata dal batch in caso di KO |
| `C_USR` | user del batch (`KSAxx`) |

---

# 6. Gestione dei titoli di migrazione

## Processo attuale dei batch di migrazione titoli

1. In fase di inserimento contratto viene inserita su `TKSAU9` una riga con:
   - `C_STA_CTR_NEW = '09'`
   - `C_STA_OPE = ' '`

2. In fase di firma del contratto viene aggiornata la riga precedente sulla `U9` con:

   ```text
   C_STA_CTR_NEW = '10'
   ```

3. In fase di apertura contratto viene aggiornata la riga precedente sulla `U9` con:

   ```text
   C_STA_CTR_NEW = '01'
   ```

4. Il batch `CreazioneTitoliPerMigrazione` legge tutte le righe della `U9` che hanno:

   ```text
   C_STA_CTR_NEW == '01'
   C_STA_OPE == ' '
   ```

   quindi contratti aperti e migrazione da elaborare.

   Per ogni riga sulla `U9`:

   - verifica che ci siano titoli da migrare per quel contratto;
   - per ogni titolo scrive la `TKSAV0`;
   - aggiorna la riga della `U9` con:

     ```text
     C_STA_OPE = 'Y'
     ```

     cioè **ELABORATA**.

5. Il batch `MigrazioneTitoli` legge sulla `U9` i record con:

   ```text
   C_STA_OPE == 'Y'
   ```

   e, per ogni riga sulla `V0`, migra il titolo.

## Stato richiesto a fine batch

Alla fine del nostro batch, per la migrazione dei titoli, dobbiamo avere sulla `TKSAU9` un record simile al seguente:

```text
[
  {
    "C_TIP_OPE_MIG": "01",n -- A_CONTRATTO_DIRETTO
    "C_CTR_NEW": nuovo codice contratto sempre light,
    "C_UTE_NEW": nuovo codice utente sempre light,
    "C_CTR_OLD": vecchio codice contratto dirfa,
    "C_UTE_OLD": vecchio codice utente dirfa,
    "D_INI_OPE": "timestamp",
    "C_STA_CTR_NEW": "01", -- APERTO
    "D_REG": "timastemp",
    "C_STA_OPE": " " -- DA_ELABORARE
  }
]
```

In questo modo la riga viene elaborata:

1. prima dal batch `CreazioneTitoliPerMigrazione`;
2. successivamente dal batch `MigrazioneTitoli`.

> **DOMANDA:** la migrazione dei titoli la fa comunque il batch oppure dobbiamo integrarla nel nostro processo?