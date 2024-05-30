# Utilizzare `100%FREE` nel comando `lvcreate` per creare un volume logico che occupi tutto lo spazio disponibile nel gruppo di volumi (VG). Questo è utile quando si desidera allocare tutto lo spazio rimanente nel VG a un singolo volume logico

Ecco come puoi fare:

1. **Creare un volume fisico (PV):**

   ```bash
   pvcreate /dev/sdX
   ```

2. **Creare un gruppo di volumi (VG):**

   ```bash
   vgcreate my_vg /dev/sdX
   ```

3. **Creare un volume logico (LV) che occupi tutto lo spazio disponibile:**

   ```bash
   lvcreate -l 100%FREE -n my_lv my_vg
   ```

4. **Formattare il volume logico con XFS:**

   ```bash
   mkfs.xfs /dev/my_vg/my_lv
   ```

5. **Montare il volume logico:**

   ```bash
   mkdir /mnt/my_mount_point
   mount /dev/my_vg/my_lv /mnt/my_mount_point
   ```

6. **Aggiornare `/etc/fstab` per il montaggio automatico:**
   - Usando l'UUID

   ```bash
   UUID=$(blkid -s UUID -o value /dev/my_vg/my_lv)
   echo "UUID=$UUID /mnt/my_mount_point xfs defaults 0 0" | tee -a /etc/fstab
   ```

   - Usando il device a blocchi

   ```bash
   echo "/dev/my_vg/my_lv /mnt/my_mount_point xfs defaults 0 0" | tee -a /etc/fstab
   ```

   - Eseguire un reload del systemd per confermare la nuova entry

    ```bash
    systemctl daemon-reload
    ```

7. **Verifica della configurazione:**
   - Verifica che il nuovo filesystem venga montato correttamente.

    ```bash
    mount -a
    ```

    - Verifica che sia visibile il mount

    ```bash
    df -hT /mnt/my_mount_point
    ```

    - Verifica che il contenuto del filesystem sia visibile

    ```bash
    ll -R /mnt/my_mount_point
    ```

- Ecco un esempio completo dei comandi da eseguire in sequenza:

```bash
pvcreate /dev/sdX
vgcreate my_vg /dev/sdX
lvcreate -l 100%FREE -n my_lv my_vg
mkfs.xfs /dev/my_vg/my_lv
mkdir /mnt/my_mount_point
mount /dev/my_vg/my_lv /mnt/my_mount_point
echo "/dev/my_vg/my_lv /mnt/my_mount_point xfs defaults 0 0" | tee -a /etc/fstab
systemctl daemon-reload
mount -a
df -hT /mnt/my_mount_point
ll -R /mnt/my_mount_point
```

- Utilizzando `100%FREE`, il volume logico `my_lv` occuperà tutto lo spazio disponibile nel gruppo di volumi `my_vg`
