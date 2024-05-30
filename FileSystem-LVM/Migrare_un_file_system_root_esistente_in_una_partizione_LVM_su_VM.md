# Migrare un file system root esistente in una partizione LVM su di una Virtual Machine VMWare

### Ambiente

* VM con Linux RedHat Entreprise 7.9
* Un disco vmdk, Disk 1 (0:0) partizionato come segue:
  * /dev/sda1 per file system di boot
  * /dev/sda2 per il file system di root
  * /dev/sda3 per il file system di swap

### Preparazione

Assicurati di avere un backup completo dei dati. Il trasferimento di partizioni può causare perdita di dati se qualcosa va storto. Essendo una VM è possibile utilizzare uno snapshot.

### 1. Preparare la VM

1. Aggiungere un nuovo disco, con una dimensione che possa ospitare l'attuale contenuto dei file system non LVM
2. Estendi l'attuale Disk 1 (sda) con la dimensione desiderata.
3. esegui lo snapshot della VM

### 2. Preparare l'ambiente

3. **Configura i Physical Volumes (PV):**

   ```bash
   pvcreate /dev/sdb
   ```

4. **Crea un Volume Group (VG):**

   ```bash
   vgcreate vg_os /dev/sdb
   ```

5. **Crea i Logical Volume (LV) per `/`, `swap`, `/home`, `/var`, e `/var/log`:**

   ```bash
   lvcreate -L <dimensione_root>G -n root vg_os
   lvcreate -L <dimensione_swap>G -n swap vg_os
   lvcreate -L <dimensione_home>G -n home vg_os
   lvcreate -L <dimensione_var>G -n var vg_os
   lvcreate -L <dimensione_varlog>G -n varlog vg_os
   ```

### 3. Copia dei dati nei nuovi volumi logici

1. **Crea i filesystem sui nuovi volumi logici:**

   ```bash
   mkfs.xfs /dev/vg_os/root
   mkfs.xfs /dev/vg_os/home
   mkfs.xfs /dev/vg_os/var
   mkfs.xfs /dev/vg_os/varlog
   mkswap /dev/vg_os/swap
   ```

2. **Monta i nuovi volumi logici temporaneamente:**

   ```bash
   mkdir /mnt/newroot
   mount /dev/vg_os/root /mnt/newroot
   mkdir /mnt/newroot/home
   mkdir /mnt/newroot/var
   mkdir /mnt/newroot/var/log
   mount /dev/vg_os/home /mnt/newroot/home
   mount /dev/vg_os/var /mnt/newroot/var
   mount /dev/vg_os/varlog /mnt/newroot/var/log
   mkdir /mnt/newroot/boot
   mount /dev/sda1 /mnt/newroot/boot
   ```

3. **Copia i dati dalle vecchie partizioni alle nuove:**

   ```bash
   rsync -axHAX / /mnt/newroot
   rsync -axHAX /home/ /mnt/newroot/home
   rsync -axHAX /var/ /mnt/newroot/var
   rsync -axHAX /var/log/ /mnt/newroot/var/log
   ```

4. **Aggiorna `fstab`:**
   Modifica il file `/mnt/newroot/etc/fstab` per utilizzare i nuovi volumi LVM. Dovrebbe somigliare a qualcosa di simile:

   ```
   /dev/vg_os/root / xfs defaults 0 0
   /dev/sda1 /boot xfs defaults 0 0
   /dev/vg_os/swap swap swap defaults 0 0
   /dev/vg_os/home /home xfs defaults 0 0
   /dev/vg_os/var /var xfs defaults 0 0
   /dev/vg_os/varlog /var/log xfs defaults 0 0
   ```

### 4. Configurazione del bootloader

1. **Chroot nel nuovo ambiente root:**

   ```bash
   mount --bind /dev /mnt/newroot/dev
   mount --bind /proc /mnt/newroot/proc
   mount --bind /sys /mnt/newroot/sys
   chroot /mnt/newroot
   ```

2. **Rigenera l'initramfs:**

   ```bash
   dracut -f
   ```

3. **Aggiorna il bootloader (GRUB):**

   ```bash
   grub2-mkconfig -o /boot/grub2/grub.cfg
   grub2-install /dev/sda
   ```

### 5. Finalizzazione

1. **Esci dal chroot e smonta i filesystem:**

   ```bash
   exit
   umount /mnt/newroot/dev
   umount /mnt/newroot/proc
   umount /mnt/newroot/sys
   umount /mnt/newroot/home
   umount /mnt/newroot/var/log
   umount /mnt/newroot/var
   umount /mnt/newroot/boot
   umount /mnt/newroot
   ```

2. **Riavvia il sistema:**

   ```bash
   reboot
   ```

### 6. Post-riavvio

1. **Verifica che il sistema si avvii correttamente utilizzando i nuovi volumi LVM. Controlla i log di sistema e `dmesg` per assicurarti che tutto sia stato montato correttamente.**
2. **Esegui un backup completo dei dati presenti nel Volume Group (`/dev/sde`), in caso di eventuali problemi durante il trasferimento.**
3. **Creazione di un Physical Volume (PV) sul nuovo disco:**
   * Prepara il disco di sistema (`/dev/sda`) per ospitare il nuovo partizionamento LVM:
      1. elimina i due file system migrati in LVM dal disco /dev/sda, cioè /dev/sda3 e /dev/sda2.
      2. sul disco sarà presente solo partizione di /boot (/dev/sda1), estenderla fino alla dimensione desiderata, per esempio a 2GB, senza perdere il contenuto attuale.
      3. creare la partizione /dev/sda2 di tipo LVM, che ospiterà tutto quello migrato in LVM, con tutta la dimensione restante del disco.
      4. tutti i precedenti passi è possibile eseguirli, utilizzando il comando `fdisk`

        ```bash
        fdisk /dev/sda
        ```

4. **Creazione di un Physical Volume (PV) sul nuovo disco:**
   * Crea un nuovo Physical Volume su `/dev/sda2`, utilizzando il comando `pvcreate`:

        ```bash
        pvcreate /dev/sda2
        ```

5. **Aggiunta del nuovo Physical Volume al Volume Group esistente:**
   * Una volta creato il Physical Volume sulla nuova partizione, aggiungilo al Volume Group esistente utilizzando il comando `vgextend`:

        ```bash
        vgextend vg_os /dev/sda2
        ```

6. **Rimozione del Physical Volume dal disco originale:**
   * Dopo aver esteso il Volume Group con il nuovo disco, puoi rimuovere il Physical Volume dal disco originale (`/dev/sde`). Prima di farlo, assicurati che non ci siano più Logical Volume attivi su quel disco. Puoi farlo utilizzando il comando `pvmove` per spostare i dati da un Physical Volume all'altro.
   Questo comando sposterà i dati da `/dev/sde` al nuovo disco (`/dev/sda2`).

        ```bash
        pvmove /dev/sde
        ```

7. **Rimozione del disco originale dal Volume Group:**
   * Dopo aver spostato tutti i dati dal disco originale al nuovo disco, puoi rimuovere il Physical Volume originale dal Volume Group utilizzando il comando `vgreduce`:

        ```bash
        vgreduce nome_del_vg /dev/sde
        ```
