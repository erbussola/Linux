# Procedura per modificare il vlan id, il nome della connessione e il nome del device
>fonte: https://access.redhat.com/solutions/6816111

**Eseguire un backup del file ifcfg-bond0.704**
```
# cp -a /etc/sysconfig/network-scripts/ifcfg-bond0.704 /etc/sysconfig/network-scripts/BCK_ifcfg-bond0.704_BCK
```

**Configurazione corrente**
```
# nmcli connection show  | egrep "NAME|bond0.704"
NAME         UUID                                  TYPE      DEVICE
bond0.704    1fbc6657-79d2-4467-9bd5-91bd90161339  vlan      bond0.704
```

**Configurazione desiderata**
```
# nmcli connection show  | egrep "NAME|bond0.330"
NAME         UUID                                  TYPE      DEVICE
bond0.330    1fbc6657-79d2-4467-9bd5-91bd90161339  vlan      bond0.330
```

**editare il file ifcfg-bond0.704**
```
# vi /etc/sysconfig/network-scripts/ifcfg-bond0.704
```
* modificare il parametro VLAN_ID da 704 a 330, 
* modificare il parametro NAME da bond0.704 a bond0.330
* modificare il parametro DEVICE da bond0.704 a bond0.330

_configurazione prima della modifica:_
```
# egrep "VLAN_ID=|NAME=|DEVICE=" /etc/sysconfig/network-scripts/ifcfg-bond0.704
VLAN_ID=704
NAME=bond0.704
DEVICE=bond0.704
```
_configurazione dopo la modifica:_
```
# egrep "VLAN_ID=|NAME=|DEVICE=" /etc/sysconfig/network-scripts/ifcfg-bond0.704
VLAN_ID=330
NAME=bond0.330
DEVICE=bond0.330
```

**rinominare il file ifcfg-bond0.704 con ifcfg-bond0.330**
```
# mv /etc/sysconfig/network-scripts/ifcfg-bond0.704 /etc/sysconfig/network-scripts/ifcfg-bond0.330
```

**Eseguire il reload della configurazione**
```
# nmcli conn reload /etc/sysconfig/network-scripts/ifcfg-bond0.330
```

**dissattivare e riattivare la connessione bond0.330**
```
# nmcli conn down bond0.330 ;nmcli conn up bond0.330
```

**Eseguire una verificare col seguente comando**
```
# nmcli con sho
```