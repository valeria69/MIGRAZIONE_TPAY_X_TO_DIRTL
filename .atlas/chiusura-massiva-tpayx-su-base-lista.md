---
unique-name: chiusura-massiva-tpayx-su-base-lista
display-name: Chiusura massiva TPayX su base lista
category: GENERAL
description: ---
---

# Chiusura massiva TPayX su base lista

---

Documento riservato Telepass                        Versione del 30/12/2022


 

## Telepass


|  |  |
| --- | --- |
| **Link JIRA** |  |
| **Versione** | 1.0 |
| **Data Rilascio** | 10/01/2023 |
| **Responsabile Documento****Telepass****FORNITORE** | Felice Del Core |
|   |  |
| Engineering |  |


## Tavola dei Contenuti


# 1. Breve Descrizione introduttiva

E' richiesto un metodo per innescare la chiusura titoli massiva di telepass pay x ed a tal proposito è stato esposto un  nuovo endpoint su KTF che le persone di operation (utenti) invocheranno tramite client rest (es. Postman)   

# 2. Esempio di utilizzo

Il metodo da impostare per effettuare l’invocazione del servizio è il seguente:

- **Metodo HTTP:**

*POST*

Segue la lista dei parametri da impostare nell’header:

- **Parametri**

*X-Sistema-Chiamante: KTP*

*Content-Type: application/json*

Segue l’endpoint da utilizzare per invocare il servizio:

- **Endpoint:**

[*http://*](http://ktf.test.gcp.telepass.com:8080/KTF/pyng/blocca-pyng-massivo-tpayx)[*ktf.prod.gcp.telepass.com:8080*](http://ktp.prod.gcp.telepass.com:8080/KTF)[*/KTF/pyng/blocca-pyng-massivo-tpayx*](http://ktf.test.gcp.telepass.com:8080/KTF/pyng/blocca-pyng-massivo-tpayx)

Il servizio prevede in input una lista di codici utente e la matricola dell’utente che sta eseguendo l’operazione

- **Input**:

*{*  
* "codiciUtente": \[xxxxxx, xxxxxx, xxxxxx, ……\],*  
* "matricola": "xxxxx"*  
*}*

- **Output**:

Il servizio restituisci una lista dove sono contenuti gli esiti negativi e per ognuno di essi il codice utente con il relativo motivo di errore, ed una lista dove invece sono contenuti i codici utenti per i quali l’elaborazione ha avuto esito positivo.

*{*  
*"esitiNegativi": \[*  
*{*  
*"codiceUtente": 0,*  
*"esito": "string"*  
*}*  
*\],*  
*"esitiPositivi": \[*  
*{*  
*"codiceUtente": 0*  
*}*  
*\]*  
*}*

Il servizio accetta in input al massimo 200 codici utente. Nel caso in cui questo valore venga superato, verrà restituito l’errore che segue:

***Error max 200 input:***

*{*  
*"errors": \[*  
*{*  
*"code": "APP73",*  
*"cause": "01",*  
*"message": "Superati 200 input"*  
*}*  
*\]*  
*}*

# 5. Revisioni:

** **

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Versione** | **Data Rilascio** | **Responsabile** | **Note** | **Link documento** |
| 1.0 | 10/01/2023 | Felice Del Core |  |   |
|   |   |   |   |   |
|   |   |   |   |   |

 

 

---


---