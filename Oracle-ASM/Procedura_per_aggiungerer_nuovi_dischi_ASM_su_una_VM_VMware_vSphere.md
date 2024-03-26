## Procedura per aggiungerer nuovi dischi ASM su una VM VMware vSphere

1. **Dopo aver aggiunto i nuovi dischi sulla VM da vSphere, appuntarsi gli SCSI ID**
   
   Nell'esempio sono stati aggiunti i seguenti ID dischi:
   
   ```
   scsi(3:4)
   scsi(3:5)
   scsi(3:6)
   scsi(3:8)
   scsi(3:9)
   scsi(3:10)
   scsi(3:11)
   scsi(3:12)
   scsi(3:13)
   ```

2. **Verifica che l'OS Abbia Visto i Nuovi Dischi Appena Aggiunti**
   
   - Su RHEL 6-7-8:
     ```
     dmesg -T | grep "$(date "+%a %b %e")" | grep "Attached SCSI disk" | uniq | tail
     ```
   - Su RHEL 5:
     ```
     grep "$(date "+%b %e")" /var/log/messages | grep -i "Attached SCSI disk" | uniq | tail
     ```
   - Alternativamente, senza selezionare una data:
     ```
     dmesg | grep "Attached SCSI disk" | uniq | tail
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
   
   Utilizza uno dei seguenti comandi per trovare e associare i dischi:
   ```
   dmesg | grep "Attached SCSI disk" | awk '{print $3 $4}' | grep ^1:0 | egrep -v "Direct-Access|Attached" | uniq
   ```

5. **Verifica i Dispositivi Associati agli ID SCSI**
   
   ```
   lsscsi | egrep "sdb[e-m]"
   ```

6. **Verifica le Dimensioni dei Dischi Aggiunti**
   
   ```
   lsblk | grep sdb[e-m] | awk '{print $1" " $4}'
   ```

7. **Visualizza gli ID_WWN dei Nuovi Dischi**
   
   ```
   for i in {e..m}; do ID_WWN=$(udevadm info --query=all --name=/dev/sdb$i | grep -i "ID_WWN="|awk -F= '{print $2}'); echo "sdb$i = $ID_WWN";done
   ```

8. **Aggiungi i Dischi al File di Configurazione Udev per ASM**
   
   Verifica che il SYMLINK non sia già presente, altrimenti aggiungi il suffisso _00.
   
   ```
   grep [NOME_ALIAS_DISCO} /etc/udev/rules.d/99-oracle-asmdevices.rules
   ```

9. **Aggiungi la Nuova Configurazione del Disco al File 99-oracle-asmdevices.rules**
   
   ```
   vi /etc/udev/rules.d/99-oracle-asmdevices.rules
   ```

   Aggiungi la seguente riga:
   ```
   ACTION=="add|change", ENV{ID_WWN}=="0x6000c297d4ae6fe0", SYMLINK+="oracleasmdisks/DATA_WEBWAMSES_03", GROUP="asmadmin", OWNER="grid", MODE="0660"
   ```

10. **Rendi Visibili ad Oracle ASM i Dischi Configurati**
    
    ```
    udevadm control --reload-rules && udevadm trigger --type=devices --action=change && sleep 5; ls -ltrh /dev/oracleasmdisks/ | grep CON05ES
    ```