---
unique-name: migrazione-forzata-tpay-x-sempre-facile
display-name: Migrazione forzata Tpay X   Sempre Facile
category: GENERAL
description: Il presente documento ha lo scopo di dare le indicazioni funzionali/tecniche e non sostituisce  l’analisi tecnica del fornitore. Integrare il presente documento nel caso ci fossero informazioni funz
---

# Migrazione forzata Tpay X \-\-\> Sempre Facile

Il presente documento ha lo scopo di dare le indicazioni funzionali/tecniche e non sostituisce  l’analisi tecnica del fornitore. Integrare il presente documento nel caso ci fossero informazioni funzionali mancanti.

E' necessario realizzare un processo batch che migra, a partire da una tabella guida, una lista di contratti Family, portandoli verso il Sempre Facile (DIRTL). Il processo deve essere realizzato totalmente su backend, simulando di fatto 

- nel caso di contratto convenzionato family -\>quello che avviene sull’attuale processo di di cambio banca (migrazione da convenzionato a family) 
- nel caso di DIRFA → creando un un DIRTL e innescando la  migrazione titoli da DIRFA a DIRTL. **Eventuali titoli SM presenti sul contratto di origine NON devono essere migrati. Serve per fare in modo che le fee su strusce blu non vengano applicate ai DIRTL**

Il processo **NON **dovrà inviare le comunicazioni attuali di tutto il processo di upselling ma inviare delle comunicazioni ad hoc, invocando magnews (es non inviare email di fine funnel o welcome mail di creazione contratto). Il razionale è che a questi clienti verrà inviata una MUC quindi la percezione lato cliente dal punto di vista della comunicazione NON dovrà essere quella di un upselling fatto in modalità self.

Da capire se MUC va fatta contestuale al processo o prima (probabilmente prima)

Scrivere sulla tabella guida in modo che il cliente riceva la comunicazione ICR col codice riconoscimento (è una delle cose che KSA già fa quando si apre un contratto)

Se il contratto Family di partenza ha un tpay associato (nella TETS1B), allora non può essere migrato.

A.M. Aggiungere altre precondizioni per migrazione

Il batch dovrà eseguire le seguenti operazioni

### 1) Chiusura contratto X

La prima operazione da fare sul contratto è quella di chiuderlo tramite il processo [Chiusura massiva TPayX su base lista - Alessandro Careddu - Confluence](https://telepass.atlassian.net/wiki/spaces/~5feda7c3849d6401110f3d34/pages/2564653057/Chiusura+massiva+TPayX+su+base+lista)

Chiaramente i contratti tpayX dovranno essere inizialmente aperti, in stato 01 .

Gli esiti positivi della chiusura vanno dati in pasto al nuovo batch di migrazione descritto di segjuito.

### 2) Creazione DIRTL

STEP DA ESEGUIRE (in transazione unica): 

1. Eseguire le operazioni fatte nel servizio di KSA che apre contratti DIRTL, nello scenario di migrazione, ovvero quando si proviene da un convenzionato family oppure un DIRFA. Il contratto verrà creato in stato 09.  
Altre operazioni da effettuare:
    1. **L’iban del contratto deve essere preso dal contratto family di origine.**
    2. codici SIA/SETIF: il contratto DIRTL deve nascere con mandato SEPA, quindi   
- CGBUU sul DIRFA associato, nel caso si parte da convenzionato e quindi devo aprire anche DIRFA
    3. Sul  contratto dovranno essere inseriti gli eventi 1A e 1C con gestore 80 e user=utenza applicativa del batch
    4. Gli indirizzi contratto dovranno essere presi dagli indirizzi contratto del contratto family (se non presenti bloccare il batch)
    5. Creare una riga sulla TETSOR con i dati del contratto X
    6. I contratti devono nascere con ente garante override - **40 sull’eventuale DIRFA aperto**
    7. NON inserire nessun consenso sulla tets4s

Poiché allo step precedente il contratto tpayx è stato chiuso, valutare se salvare nella tabella guida sia il contratto tpayx che quello associato, in modo da mantenere la traccia di quello associato

1. **Firmare “tecnicamente” il contratto (o i contratti nel caso si è partiti da un convenzionato) portandolo in stato firmato (stato 10)**, eseguendo le stesse operazioni eseguite dal servizio di KSA che sta dietro la callback di Intesa. Chiaramente andrà inserito l’evento contratto 1D (con gestore 80 e user=utenza applicativa del batch)  
Prestare attenzione al fatto che non dovranno essere inviate le comunicazioni (email/sms) attuali.  
Verificare anche lato test che non venga scritta nessuna riga nella TETSXM e non venga inviata nessuna comunicazione al cliente.  
Inoltre in questa fase NON deve essere 
    1. innescato l’allineamento SEDA/check iban, ma dovrà essere inserito direttamente l’evento 1F  (con gestore 80 e user=utenza applicativa del batch)
    2. innescata l’interazione verso PEGA (tksaa1) ma dovrà essere inserito direttamente l’evento 1E di controllo documento OK (con gestore 80 e user=utenza applicativa del batch)
2. Aprire il contratto andando a creare l’evento 1H. Fare in modo chiaramente che non venga inviata la welcome mail di apertura ma una **nuova comunicazione definita ad hoc.**
3. Inserire le righe nelle tabelle guida per la migrazione titoli in modo da trasferire i titoli AT/AM  
**Se sul contratto Family di partenza ci sono dei titoli SM, vanno CHIUSI. Questo serve per evitare che il cliente paghi le fee su strisce blu sul contratto DIRTL di destinazione.**
4. Scrivere una riga sulla TKSANN con D\_SCA\_OPE=current timestamp  con un nuovo tipo operazione fittizio, per poter tracciare questi contratti

#####   
 

Verificare se ad oggni nella migrazione da convenzionato a DIRFA vengono eseguite particolari operazioni (scritture su tabelle guida per ETS/fatturazione, etc)