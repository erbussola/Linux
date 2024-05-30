# Utilizzare `100%FREE` nel comando `lvcreate` per creare un volume logico che occupi tutto lo spazio disponibile nel gruppo di volumi (VG). Questo è utile quando si desidera allocare tutto lo spazio rimanente nel VG a un singolo volume logico

Ecco come puoi fare:

1. **Creare un volume fisico (PV):**

   ```bash
   sudo pvcreate /dev/sdX
   ```

2. **Creare un gruppo di volumi (VG):**

   ```bash
   sudo vgcreate my_vg /dev/sdX
   ```

3. **Creare un volume logico (LV) che occupi tutto lo spazio disponibile:**

   ```bash
   sudo lvcreate -l 100%FREE -n my_lv my_vg
   ```

4. **Formattare il volume logico con XFS:**

   ```bash
   sudo mkfs.xfs /dev/my_vg/my_lv
   ```

5. **Montare il volume logico:**

   ```bash
   sudo mkdir /mnt/my_mount_point
   sudo mount /dev/my_vg/my_lv /mnt/my_mount_point
   ```

6. **Aggiornare `/etc/fstab` per il montaggio automatico:**

   ```bash
   UUID=$(sudo blkid -s UUID -o value /dev/my_vg/my_lv)
   echo "UUID=$UUID /mnt/my_mount_point xfs defaults 0 0" | sudo tee -a /etc/fstab
   ```

Ecco un esempio completo dei comandi da eseguire in sequenza:

```bash
sudo pvcreate /dev/sdX
sudo vgcreate my_vg /dev/sdX
sudo lvcreate -l 100%FREE -n my_lv my_vg
sudo mkfs.xfs /dev/my_vg/my_lv
sudo mkdir /mnt/my_mount_point
sudo mount /dev/my_vg/my_lv /mnt/my_mount_point
UUID=$(sudo blkid -s UUID -o value /dev/my_vg/my_lv)
echo "UUID=$UUID /mnt/my_mount_point xfs defaults 0 0" | sudo tee -a /etc/fstab
```

- Utilizzando `100%FREE`, il volume logico `my_lv` occuperà tutto lo spazio disponibile nel gruppo di volumi `my_vg`
