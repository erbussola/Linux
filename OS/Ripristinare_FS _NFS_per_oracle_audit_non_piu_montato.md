# How to per ripristinare il FS nfs /oracleaud non più montato

Gli step di seguito sono un esempio eseguito su di un server con hostname epmfldbbc06-a  
Se dovete eseguire i comandi prestate attezionzione ai nome server e NAS NFS che posso essere diversi.

1. **Visualizzare il contenuto della directory /mnt:**
   ```bash
   ll /mnt/
   ```
   - Elenca i file e le directory nella cartella `/mnt` con informazioni dettagliate (formato lungo).

2. **Verificare lo stato del montaggio NFS per oracleaud:**
   ```bash
   df -hT | grep -i oracleaud
   ```
   - Verifica che il mount point non è montato

3. **Trovare l'entry di montaggio NFS in /etc/fstab:**
   ```bash
   grep nut /etc/fstab
   ```
   - Cerca il termine "nut" nel file `/etc/fstab`. Questo serve per trovare la configurazione del montaggio NFS, indicando il server e il percorso per il punto di montaggio `oracleaud`.
   - **Output:**
     ```
     nutrn01-nases.ternaren.prv:/oracle_audit_ese/epmfldbbc06-a  /oracleaud nfs soft 0 0
     ```

4. **Arrestare il servizio rsyslog:**
   ```bash
   systemctl stop rsyslog
   ```
   - Ferma il servizio `rsyslog`, che si occupa della registrazione dei log di sistema. Questo potrebbe essere fatto per prevenire interruzioni durante la manutenzione.

5. **Montare temporaneamente la condivisione NFS:**
   ```bash
   mount -o soft nutrn01-nases.ternaren.prv:/oracle_audit_ese/epmfldbbc06-a /mnt/
   ```
   - Monta temporaneamente la condivisione NFS sulla directory `/mnt` con l'opzione `soft`, che consente al montaggio di andare in timeout se il server è irraggiungibile.

6. **Elencare i file più recenti in /mnt:**
   ```bash
   ll -trh /mnt/|tail
   ```
   - Elenca i file in `/mnt` ordinati per tempo (`-trh`), con i file più recenti alla fine. Vengono mostrati solo gli ultimi 10 file usando `tail`.
   - **Output:**
     ```
     -rw-------   1 root root  851K Jul 19 03:10 ora_audit.log-20240719.gz
     -rw-------   1 root root  553K Jul 20 03:11 ora_audit.log-20240720.gz
     -rw-------   1 root root  738K Jul 21 03:15 ora_audit.log-20240721.gz
     -rw-------   1 root root 1002K Jul 22 03:44 ora_audit.log-20240722.gz
     -rw-------   1 root root  604K Jul 23 03:10 ora_audit.log-20240723.gz
     -rw-------   1 root root  200K Jul 23 14:58 ora_audit.log-20240724.gz
     -rw-------   1 root root  386K Jul 25 03:28 ora_audit.log-20240725.gz
     -rw-------   1 root root  3.7M Jul 25 06:59 ora_audit.log
     ```

7. **Spostare i file da /oracleaud a /mnt:**
   ```bash
   mv /oracleaud/* /mnt/.
   ```
   - Sposta tutti i file da `/oracleaud` a `/mnt`. Il `.` alla fine indica la directory corrente (cioè, `/mnt/`).
   - **Output:**
     ```
     mv: overwrite ‘/mnt/./ora_audit.log’? y
     ```

8. **Elencare i file in /oracleaud dopo lo spostamento:**
   ```bash
   ll -trh /oracleaud |tail 
   ```
   - Controlla se ci sono file rimasti in `/oracleaud` dopo lo spostamento. Il risultato dovrebbe essere idealmente vuoto o minimo se lo spostamento è stato completato con successo.
   - **Output:**
     ```
     total 0
     ```

9.  **Smontare la condivisione NFS temporanea:**
    ```bash
    umount /mnt/
    ```
    - Smonta la condivisione NFS da `/mnt`.

10. **Rimontare tutti i filesystem da /etc/fstab:**
    ```bash
    mount -a
    ```
    - Monta tutti i filesystem elencati in `/etc/fstab` che non sono attualmente montati. Questo viene utilizzato per ristabilire il montaggio NFS su `/oracleaud`.

11. **Verificare il montaggio NFS su /oracleaud:**
    ```bash
    df -hT | grep -i oracleaud
    ```
    - Controlla se `/oracleaud` è stato montato correttamente e mostra i dettagli.
    - **Output:**
      ```
      10.147.146.177:/oracle_audit_ese/epmfldbbc06-a nfs4      2.0T  5.5G  2.0T   1% /oracleaud
      ```

12. **Elencare il contenuto di /oracleaud dopo il rimontaggio:**
    ```bash
    ll -trh /oracleaud |tail 
    ```
    - Elenca i file più recenti in `/oracleaud` per confermare che il montaggio e lo spostamento dei file siano stati effettuati con successo.
    - **Output:**
      ```
      -rw------- 1 root root  604K Jul 23 03:10 ora_audit.log-20240723.gz
      -rw------- 1 root root  200K Jul 23 14:58 ora_audit.log-20240724.gz
      -rw------- 1 root root  386K Jul 25 03:28 ora_audit.log-20240725.gz
      -rw------- 1 root root  844K Jul 26 03:32 ora_audit.log-20240726.gz
      -rw------- 1 root root  1.4M Jul 27 03:36 ora_audit.log-20240727.gz
      -rw------- 1 root root  2.1M Jul 28 03:08 ora_audit.log-20240728.gz
      -rw------- 1 root root  1.7M Jul 29 03:05 ora_audit.log-20240729.gz
      -rw------- 1 root root  1.3M Jul 30 03:10 ora_audit.log-20240730.gz
      -rw------- 1 root root  1.1M Jul 31 03:08 ora_audit.log-20240731.gz
      -rw------- 1 root root   17M Jul 31 15:32 ora_audit.log
      ```

13. **Avviare il servizio rsyslog:**
    ```bash
    systemctl start rsyslog
    ```
    - Riavvia il servizio `rsyslog`, riprendendo la registrazione dei log di sistema.

14. **Verificare lo stato di rsyslog:**
    ```bash
    systemctl status rsyslog
    ```
    - Controlla lo stato corrente del servizio `rsyslog` per assicurarsi che sia in esecuzione correttamente.
    - **Output:**
      ```
      ● rsyslog.service - System Logging Service
         Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled; vendor preset: enabled)
         Active: active (running) since Wed 2024-07-31 15:28:44 CEST; 9s ago
           Docs: man:rsyslogd(8)
                 http://www.rsyslog.com/doc/
       Main PID: 23877 (rsyslogd)
          Tasks: 3
         Memory: 1.1M
         CGroup: /system.slice/rsyslog.service
                 └─23877 /usr/sbin/rsyslogd -n

      Jul 31 15:28:44 epmfldbbc06-a.servizi.prv systemd[1]: Starting System Logging Service...
      Jul 31 15:28:44 epmfldbbc06-a.servizi.prv rsyslogd[23877]:  [origin software="rsyslogd" swVersion="8.24.0-57.el7_9.3" x-pid="23877" x-info="http://www.rs..."] start
      Jul 31 15:28:44 epmfldbbc06-a.servizi.prv systemd[1]: Started System Logging Service.
      ```

Questi comandi e i loro output aiutano a verificare e risolvere eventuali problemi con il montaggio NFS e garantire che il servizio `rsyslog` funzioni correttamente dopo la manutenzione.
