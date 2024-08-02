## Procedura all'estrazione dei log OpenShift usando un server bastion

### Introduzione
Questa guida ti aiuterà a estrarre i log da uno o più pod all'interno di un progetto specifico su un bastion OpenShift, per un determinato intervallo di date. I comandi forniti sono pensati per automatizzare il processo e rendere più efficiente la ricerca di informazioni nei log.

### Prerequisiti
* **Accesso al bastion:** Devi avere le credenziali per collegarti al bastion OpenShift.
* **Conoscenza di base di OpenShift:** È utile avere una familiarità con i concetti di pod, progetti e comandi di base di OpenShift.

### Passaggi

1. **Collegamento al bastion:**
   ```bash
   ssh user@srv_bastion
   ```
   Questo comando ti connette al bastion OpenShift specificato. Sostituisci `srv_bastion` con il nome o l'indirizzo IP corretto del tuo bastion.

2. **Login:**
   ```bash
   oc login -u admin
   ```
   Effettua il login come amministratore. Sostituisci `admin` con il tuo nome utente se diverso.  
   Puoi usare anche il *Copy login command* che trovi sulla GUI di OpenShift aprendo il menu a tendina che trovi sul nome utente.  
   Copia la stringa che trovi sotto *Log in with this token*

3. **Selezione del progetto:**
   ```bash
   oc project my-project
   ```
   Seleziona il progetto di cui vuoi estrarre i log. Sostituisci `my-project` con il nome del tuo progetto.

4. **Ricerca dei log per data:**
   ```bash
   oc get pods --no-headers | grep Running | awk '{print $1}' | xargs -I {} oc logs {} | sed -n '/2024-08-01/,/2024-08-02/p'
   ```
   * **`oc get pods --no-headers | grep Running | awk '{print $1}'`:** Questa parte del comando elenca tutti i pod in stato "Running" nel progetto corrente e ne estrae i nomi.
   * **`xargs -I {} oc logs {}`:** Per ogni nome di pod, viene eseguito il comando `oc logs` per estrarre i log.
   * **`sed -n '/2024-08-01/,/2024-08-02/p'`:** Filtra i log per le date specificate (nel formato AAAA-MM-GG).

5. **Estrazione dei log in un file:**
   ```bash
   oc get pods --no-headers| grep Running | awk '{print $1}' | xargs -I {} sh -c 'echo "##-- Logs from pod: {} --##" && oc logs {} | sed -n '/2024-08-01/,/2024-08-02/p'' > /tmp/SCTASK0082585_$(oc project | cut -d'"' -f2)_$(date '+%Y%m%d_%H%M%S').log
   ```
   * **`sh -c`:** Viene utilizzata una sotto-shell per eseguire una serie di comandi.
   * **`echo "##-- Logs from pod: {} --##"`:** Aggiunge un'intestazione a ogni sezione del file di output per identificare i log di ciascun pod.
   * **`> /tmp/SCTASK0082585_$(oc project | cut -d'"' -f2)_$(date '+%Y%m%d_%H%M%S').log`:** Reindirizza l'output in un file temporaneo con un nome univoco basato sul nome del progetto e sulla data e ora attuali.

### Opzioni aggiuntive

* **Lista di tutti i pod in un progetto:**
   ```bash
   oc get pods --no-headers -n caviinterrati-ese| grep Running | awk '{print $1}'
   ```
   Aggiungendo l'opzione `-n [nome_progetto]`, puoi specificare un progetto diverso.

* **Log di un pod specifico:**
   ```bash
   oc logs my-pod1-76f884c59-mqzxq -n my-project
   ```
   Sostituisci il nome del pod e del progetto con quelli desiderati.
   Per salvare i log in un file per un giorno specifico:
   ```bash
   oc logs my-pod1-76f884c59-mqzxq -n my-project | sed -n '/2024-08-01/,/2024-08-02/p' >> /tmp/SCTASK0082585_my-pod1_my-project_$(date '+%Y%m%d_%H%M%S').log
   ```

### Note
* **Personalizzazione:** Puoi modificare le date, i nomi dei file e i progetti per adattarli alle tue esigenze.
* **Prestazioni:** Per un gran numero di pod o grandi quantità di log, potrebbe essere necessario ottimizzare i comandi o utilizzare strumenti più specializzati.
* **Sicurezza:** Assicurati di avere i permessi necessari per eseguire questi comandi e di rispettare le politiche di sicurezza della tua organizzazione.
