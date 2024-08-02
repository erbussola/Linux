# Estendere una partizione LVM estendendo il disco virtuale già presente

### Ambiente

* VM con Linux RedHat Entreprise 8.9
* Un disco vmdk, Disk 1 (0:0) partizionato come segue:
  * /dev/sda1 per file system di boot
  * /dev/sda2 per il file system LVM

### Preparazione

Assicurati di avere un backup completo dei dati. E' importante fare attenzione quando si apportano modifiche alle partizioni, poichè gli errori posso comportare la perdita dei dati.\
Lavorando con una VM ci viene in aiuto lo snapshot.

### 1. Preparare la VM

1. Estendere l'attuale Disk 1 (0:0) (/dev/sda) con la dimensione desiderata.
2. esegui lo snapshot della VM
3. Eseguire il salvataggio di tutte le info del disco sda

	```bash
	today=`date '+%Y%m%d_%H%M%S'`
	fdisk -lu /dev/sda > info_volume_lvm_$today.txt
	pvdisplay >> info_volume_lvm_$today.txt
	vgdisplay >> info_volume_lvm_$today.txt
	lvdisplay >> info_volume_lvm_$today.txt
	df -h >> info_volume_lvm_$today.txt
	```

4. Rendere visibile alla VM lo spazio aggiunto al disco
   - Su RHEL 6-7-8 eseguire una scansione sul controller 0:0:0:0 col comando seguente

		```bash
		echo 1 > /sys/class/scsi_device/0\:0\:0\:0/device/rescan
		```

	- Su RHEL 5 verificare che host0 è il controller ed eseguire il seguente comando
		
		```bash
		echo "- - -" > /sys/class/scsi_host/host0/scan
		```

5. per verificare che lo spazio aggiunto sia visibile al sisteama usare  `fdisk`

	```bash
	root@rmftpp04:~# fdisk -lu /dev/sdb
	```

#Aggiungere lo spazio aggiuntivo alla partizione LVM 
root@rmftpp04:~# fdisk -u /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): p
Disk /dev/sdb: 400 GiB, 429496729600 bytes, 838860800 sectors
Disk model: Virtual disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfe055a8e

Device     Boot Start       End   Sectors  Size Id Type
/dev/sdb1        2048 629145599 629143552  300G 8e Linux LVM

Command (m for help): d

Selected partition 1
Partition 1 has been deleted.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-838860799, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-838860799, default 838860799):

Created a new partition 1 of type 'Linux' and of size 400 GiB.
Partition #1 contains a LVM2_member signature.

Do you want to remove the signature? [Y]es/[N]o: n

Command (m for help): t

Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Syncing disks.

#Verifcare che lo spazio aggiuntivo sia visibile sulla partizione LVM
root@rmftpp04:~# fdisk -lu /dev/sdb

#Verificare l'attuale dimensione del volume fisico LVM (in teoria dovrebbe ancora della dimensione prima dell'aggiunta alla partizione LVM)
pvs

#Estendere il volume fisico LVM
pvresize /dev/sdb1

#Verifcare il volume fisico, il gruppo logico ed il volume logico LVM
pvs
vgs
lvs

#Estendere il volume logico LVM fino al 100% della nuova dimensione
lvextend -r -l +100%FREE /dev/mapper/vg01-lvol0
#estendere lv e il fs di 250GB
lvresize --resizefs --size +10GB /dev/rhel/root
lvresize --resizefs --extents +100%FREE /dev/mapper/vg01-lvol0

#Verificare l'avvenuta espanzione del volume logico
lvs

#Verificare che il FS sia stato esteso.
df -h /data

#Per estendere un volume group esistente con un altro disco
#rescan device per trovare il disco appena aggiunto
echo "- - -" > /sys/class/scsi_host/host0/scan

#verificare la presenza del nuovo disco
lsscsi |egrep "/dev/[device]"

#creare il pv sul nuovo disco
pvcreate /dev/sdg

#estendere il vg nel quale si trova lv
vgextend JIRA /dev/sdg

#estendere lv e il fs
lvresize --resizefs --size +250GB /dev/JIRA/data
lvresize -r -L +250GB /dev/JIRA/data
lvresize -r -l +100%FREE /dev/[vg]/[lv]

##Nel caso il comando precedente non andasse, uasre il seguante
lvextend -L +250GB /dev/JIRA/data
#per fs ext4
resize4fs /dev/JIRA/data

#switch user root
sudo -i

#check disk structure
lsblk
	NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda              8:0    0   60G  0 disk
	├─sda1           8:1    0  600M  0 part /boot/efi
	├─sda2           8:2    0    1G  0 part /boot
	└─sda3           8:3    0 48.4G  0 part
	  ├─vgsys-root 253:0    0 38.4G  0 lvm  /
	  ├─vgsys-swap 253:1    0    5G  0 lvm  [SWAP]
	  └─vgsys-home 253:2    0    5G  0 lvm  /home
	sdb              8:16   0  100G  0 disk
	sr0             11:0    1  9.4G  0 rom  /dvd-rom


#estendere partizione lvm (/dev/sda3)
growpart /dev/sda 3
pvscan
pvresize /dev/sda3
lvextend -r -l +100%FREE /dev/mapper/vgsys-root
xfs_growfs /
lsblk
	NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda              8:0    0   60G  0 disk
	├─sda1           8:1    0  600M  0 part /boot/efi
	├─sda2           8:2    0    1G  0 part /boot
	└─sda3           8:3    0 58.4G  0 part
	  ├─vgsys-root 253:0    0 48.4G  0 lvm  /
	  ├─vgsys-swap 253:1    0    5G  0 lvm  [SWAP]
	  └─vgsys-home 253:2    0    5G  0 lvm  /home
	sdb              8:16   0  100G  0 disk
	sr0             11:0    1  9.4G  0 rom  /dvd-rom

#creare VG vgdata
#creare la partizione LVM sul disco /dev/sdb
fdisk -u /dev/sdb <<EOF
n
p
1


t
8e
w
EOF

#create VG vgdata
pvcreate /dev/sdb1
vgcreate vgdata /dev/sdb1
lvcreate -L +100G -n docker vgdata

#se è richiesto un FS xfs
mkfs.xfs /dev/mapper/vgdata-docker

#Se è richiesto un FS ext4
mkfs.ext4 /dev/vg_export/lv_export

#add mountpoint to /etc/fstab
mkdir -p /appli/docker
echo "/dev/mapper/vgdata-docker /appli/docker xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount /appli/docker




sudo pvcreate /dev/sdX
sudo vgcreate my_vg /dev/sdX
sudo lvcreate -l 100%FREE -n my_lv my_vg
sudo mkfs.xfs /dev/my_vg/my_lv
sudo mkdir /mnt/my_mount_point
sudo mount /dev/my_vg/my_lv /mnt/my_mount_point
UUID=$(sudo blkid -s UUID -o value /dev/my_vg/my_lv)
echo "UUID=$UUID /mnt/my_mount_point xfs defaults 0 0" | sudo tee -a /etc/fstab
