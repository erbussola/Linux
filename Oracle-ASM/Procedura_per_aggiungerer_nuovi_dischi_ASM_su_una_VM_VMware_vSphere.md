## Procedura per aggiungerer nuovi dischi ASM su una VM VMware vSphere

1. **Dopo aver aggiunto i nuovi dischi sulla VM da vSphere, appuntarsi gli SCSI ID**
   
   Nell'esempio sono stati aggiunti i seguenti ID dischi:
   
   ```
   scsi(1:8)
   scsi(1:9)
   scsi(1:10)
   ```

2. **Verifica che l'OS Abbia Visto i Nuovi Dischi Appena Aggiunti**
   
   - Su RHEL 6-7-8:
     ```
     dmesg -T | grep "$(date "+%a %b %e")" | grep "Attached SCSI disk" | uniq | tail
     [Wed Jan 31 14:37:40 2024] sd 1:0:8:0: [sdf] Attached SCSI disk
     [Wed Jan 31 14:37:40 2024] sd 1:0:10:0: [sdh] Attached SCSI disk
     [Wed Jan 31 14:37:40 2024] sd 1:0:9:0: [sdg] Attached SCSI disk
     ```
   - Su RHEL 5:
     ```
     grep "$(date "+%b %e")" /var/log/messages | grep -i "Attached SCSI disk" | uniq | tail
     Jan 31 14:37:40 covrnld13-a kernel: sd 1:0:8:0: [sdf] Attached SCSI disk
     Jan 31 14:37:40 covrnld13-a kernel: sd 1:0:10:0: [sdh] Attached SCSI disk
     Jan 31 14:37:40 covrnld13-a kernel: sd 1:0:9:0: [sdg] Attached SCSI disk
     ```
   - Alternativamente, senza selezionare una data:
     ```
     dmesg | grep "Attached SCSI disk" | uniq | tail
     [    1.813093] sd 1:0:8:0: [sdf] Attached SCSI disk
     [    1.815900] sd 1:0:10:0: [sdh] Attached SCSI disk
     [    1.838322] sd 1:0:9:0: [sdg] Attached SCSI disk
     ```

3. **Esegui una Scansione se Necessario**
   
   - RHEL 5:
     ```
     echo "- - -" > /sys/class/scsi_host/host0/scan
     ```
   - RHEL 6-7-8:
     ```
     echo 1 > /sys/class/scsi_device/0:0:1:0/device/rescan
     ```

4. **Associare i Dischi ai Rispettivi SCSI ID**
   
   Utilizza il seguente comando per trovare e associare i dischi:
   ```
   dmesg | grep "Attached SCSI disk" | awk '{print $3 $4}' | grep ^1:0 | egrep -v "Direct-Access|Attached" | uniq
   1:0:8:0:[sdf]
   1:0:10:0:[sdh]
   1:0:9:0:[sdg]

   ```

5. **Verifica i Dispositivi Associati agli ID SCSI**
   
   ```
   lsscsi | egrep "sd[f-h]"
   [1:0:8:0]    disk    VMware   Virtual disk     2.0   /dev/sdf
   [1:0:9:0]    disk    VMware   Virtual disk     2.0   /dev/sdg
   [1:0:10:0]    disk    VMware   Virtual disk     2.0   /dev/sdh
   ```

6. **Verifica le Dimensioni dei Dischi Aggiunti**
   
   ```
   lsblk | grep sdb[e-m] | awk '{print $1" " $4}'
   sdf 20G
   sdg 20G
   sdh 20G
   ```

7. **Visualizza gli ID_WWN dei Nuovi Dischi**
   
   ```
   for i in {f..h}; do ID_WWN=$(udevadm info --query=all --name=/dev/sd$i | grep -i "ID_WWN="|awk -F= '{print $2}'); echo "sd$i = $ID_WWN";done
   sdf = 0x6000c29c14cb66c5
   sdg = 0x6000c294a946cc4f
   sdh = 0x6000c293d646c727
   ```

8. **Aggiungi i Dischi al File di Configurazione Udev per ASM**
   
   Verifica che il SYMLINK non sia già presente, se presente aggiungi il suffisso _## continuando la numerazione trovata.
   
   ```
   grep [NOME_ALIAS_DISCO} /etc/udev/rules.d/99-oracle-asmdevices.rules
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c2957a119904", SYMLINK+="oracleasmdisks/DATA_DB105ES_00", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c29c4675e13c", SYMLINK+="oracleasmdisks/DATA_DB105ES_01", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ```

   Aggiungi la Nuova Configurazione del Disco editando il file 99-oracle-asmdevices.rules 
   
   ```
   vi /etc/udev/rules.d/99-oracle-asmdevices.rules
   ```

   Aggiungi le seguente righe:
   ```
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c29c14cb66c5", SYMLINK+="oracleasmdisks/DATA_DB105ES_02", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c294a946cc4f", SYMLINK+="oracleasmdisks/DATA_DB105ES_02", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c293d646c727", SYMLINK+="oracleasmdisks/DATA_DB105ES_02", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ```

9. **Rendi Visibili ad Oracle ASM i Dischi Configurati**
    
    ```
    udevadm control --reload-rules && udevadm trigger --type=devices --action=change
    ```

10. **Verifica che sia stata recepita la configurazione da udev e dal OS**

    ```
    ls -ltrh /dev/oracleasmdisks/ | grep DB105ES
    lrwxrwxrwx 1 root root 6 Mar 27 10:44 DATA_DB105ES_01 -> ../sdf
    lrwxrwxrwx 1 root root 6 Mar 27 10:44 DATA_DB105ES_02 -> ../sdg
    lrwxrwxrwx 1 root root 6 Mar 27 10:44 DATA_DB105ES_03 -> ../sdh
    ```
