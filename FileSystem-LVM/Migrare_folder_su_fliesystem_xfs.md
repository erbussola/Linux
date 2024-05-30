# Migrare a caldo la cartella `/u01` in un filesystem separato con un volume LVM

1. **Creazione del volume LVM:**
   - Crea un volume logico (LV) da utilizzare per `/u01`. Supponiamo che tu abbia già un volume group (VG) esistente, ad esempio `rhel`.

    ```bash
    lvcreate -L 400G -n u01 rhel
    ```

   - Formatta il volume logico con un filesystem, ad esempio xfs.

    ```bash
    mkfs.xfs /dev/rhel/u01
    ```

2. **Preparazione del mount point temporaneo:**
   - Crea un punto di mount temporaneo.

    ```bash
    mkdir /mnt/u01
    ```

3. **Montare il nuovo volume logico:**
   - Monta il nuovo volume logico nel punto di mount temporaneo.

    ```bash
    mount /dev/rhel/u01 /mnt/u01
    ```

4. **Sincronizzazione dei dati:**
   - Copia i dati dalla vecchia `/u01` alla nuova posizione usando `rsync` per mantenere le proprietà dei file.

    ```bash
    rsync -avx /u01/ /mnt/u01/
    ```

5. **Verifica della copia:**
   - Assicurati che tutti i dati siano stati copiati correttamente.

    ```bash
    diff -r /u01 /mnt/u01
    ```

6. **Spostamento della vecchia `/u01`:**
   - Rinomina la vecchia `/u01` per tenerla come backup temporaneo.

    ```bash
    mv /u01 /u01_old
    ```

   - Crea un nuovo mount point per `/u01`.

    ```bash
    mkdir /u01
    ```

7. **Montare il nuovo volume logico su `/u01`:**
   - Smontare il mount di appoggio `/mnt/u01`

    ```bash
    umount /mnt/u01
    ```

   - Monta il nuovo volume logico direttamente su `/u01`.

    ```bash
    mount /dev/rhel/u01 /u01
    ```

8. **Aggiornamento di `/etc/fstab`:**
   - Aggiungi una voce in `/etc/fstab` per montare il volume logico su `/u01` automaticamente al boot.

    ```bash
    echo '/dev/rhel/u01 /u01 xfs defaults 0 2' >> /etc/fstab
    ```

    - Eseguire un reload del systemd per confermare la nuova entry

    ```bash
    systemctl daemon-reload
    ```

9. **Verifica della configurazione:**
   - Verifica che il nuovo filesystem sia montato correttamente.

    ```bash
    mount -a
    ```

10. **Pulizia:**
    - Se tutto funziona correttamente, puoi rimuovere il vecchio `/u01` di backup.

    ```bash
    rm -rf /u01_old
    ```

- Ecco i comandi da eseguire in sequenza:

    ```bash
    lvcreate -L 400G -n u01 rhel
    mkfs.xfs /dev/rhel/u01
    mkdir /mnt/u01
    mount /dev/rhel/u01 /mnt/u01
    rsync -avx /u01/ /mnt/u01/
    diff -r /u01 /mnt/u01
    mv /u01 /u01_old
    mkdir /u01
    umount /mnt/u01
    mount /dev/rhel/u01 /u01
    echo '/dev/rhel/u01 /u01 xfs defaults 0 2' >> /etc/fstab
    mount -a
    rm -rf /u01_old
    ```

- Questi passaggi dovrebbero consentirti di spostare a caldo la cartella `/u01` su un nuovo filesystem con volume LVM senza riavviare il sistema. Tuttavia, è sempre consigliabile eseguire un backup dei dati importanti e testare queste operazioni su un ambiente di staging prima di eseguirle in produzione.