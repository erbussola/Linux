# Datacenter for Dummies: Le SAN

powered by di [darkpila](https://leganerd.com/author/darkpila/) 23—Lug—2014 / 10:31 AM

È vero, l’informatica è una società piramidale. Se moltissimi sono gli utenti, già lo stadio di “bravo con il computer” è molto meno densamente popolato e più si sale verso il “vertice”, meno individui si trovano.


Quindi se molti sono in grado di _giochicchiare_ con un paio di server magari in cluster Linux con _drbd_, pochi conoscono le tecnologie che permettono di creare strutture di un certo livello. Ed è un peccato perché ci sono cose molto interessanti e intrinsecamente nerd.

Si dice che la differenza tra un uomo e un bambino sta, in fondo, sul costo dei loro giocattoli. E qui c’è roba molto costosa e interessante.

Dando quindi per scontato che tutti conoscano le basi del networking e del TCP/IP, vogliamo quindi iniziare da una delle tecnologie più ostiche e interessanti dei datacenter, ovvero le SAN.

SAN sta per **Storage Area Network**, ovvero una rete di backend a cui si collegano tutti i sistemi che gestiscono lo storage dei dati. Stiamo parlando quindi di block storage (diversamente dai NAS), di Tape library, CD/DVD/Blu-ray Juke Box e molti, molti server.

Possiamo pensare ad un moderno datacenter come ad una sovrapposizione di layer.

I client parlano con i server attraverso la LAN, i server parlano con gli storage di backend attraverso la SAN.

Perché si usano le SAN? Vi sono una serie di motivi:

*   **Cluster**: che si tratti di cluster per performance (tipo HPC), distribuzione del carico (ovvero molti server con uno o più balancer a monte che provvedono a distribuire le connessioni) o fault tolerance, in tutti casi i nodi hanno necessità di prelevare e scrivere dati all’interno di una risorsa comune. In questo caso spessissimo si usa un block storage che esporti una LUN che sarà poi formattata con un cluster file system come OCFS2, Lustre, Stornext, GPFS, GFS ecc. ecc. La connessione tra i server e lo storage avviene tramite una SAN
*   **Capacità**: Difficilmente si trovano server in grado di alloggiare più di 24 dischi. Anche se per i datacenter più grandi si stanno usando sistemi di storage distribuito (tipo Nutanix, ceph, HadoopFS o GoogleFS), fino a dimensione umane (sto parlando di qualche decina di PetaByte), gli storage la fanno ancora da padrone. Uno storage come il Fujitsu Eternus DX 8700 S2 è in grado di arrivare, da solo, fino a 6 PB di dati e di servire fino a 2048 host. Inoltre è possibile collegare più storage ad una SAN potendo così scalare orizzontalmente quasi senza limiti.
*   **Velocità:** nessun bus locale può arrivare ai livelli di una SAN ben progettata. FibreChannel può arrivare a velocità di picco di 16 Gb/Sec, iSCSI a 40 Gb/sec, FCoE fino a 10 Gb/sec. Il tutto su una singola fibra. Dato che si utilizzano sempre configurazioni in _multipath_ le velocità reali possono essere superiori.

## Principali protocolli

Le SAN nascono per superare il problema dello storage condiviso. I primi cluster utilizzavano SCSI come trasporto.

Veniva collegato uno storage SCSI a due nodi, ognuno con un controller che terminava la catena. I problemi in questa configurazione potevano essere innumerevoli. Il protocollo sviluppato per queste configurazioni era conosciuto come [HIPPI](http://en.wikipedia.org/wiki/HIPPI) (High Performance Parallel Interface).

## Fibre Channel

Nel 1988 fu presentato [Fibre Channel](http://en.wikipedia.org/wiki/FibreChannel), che divenne standard [ANSI](http://en.wikipedia.org/wiki/ANSI) nel 1994. In realtà chiamarlo protocollo è riduttivo in quanto si tratta di una intera tecnologia di rete specializzata per costruzione di SAN.

Quando si parla di Fibre Channel infatti si parla di tutta una serie di concetti e tecnologie che sono legate assieme:

### Protocollo

Il protocollo Fibre Channel è costruito da uno stack di cinque differenti livelli, numerati da 0 (strato fisico) a 4 (livello di mapping).

Gli strati da 0 a 3 sono chiamati anche FC-PH dato che coinvolgono tutto lo strato fisico della soluzione.  

### Cavi

Fibre Channel utilizza fibre ottiche per il trasporto. Le più utilizzate sono le fibre ottiche multimodali OM4 con connettore LC.

Nel caso di repliche geografiche tra datacenter  è piuttosto comune trovare anche collegamenti monomodali.

### Topologia

Fibre Channel può funzionare con tre diverse topologie:

1.  **FC-P2P**: Nota anche come attacco diretto è la topologia più semplice e consiste in un collegamento punto-punto tra due dispositivi. Le porte usate per FC-P2P sono configurate come  N\_Port.
2.  **FC-AL**: Nota come “arbitrated loop” permette il collegamento fino a 7 dispositivi in una topologia a Ring e fino ad una velocità massima di 4 Gb/sec. E’ un tipo di connessione in disuso anche se è ancora utilizzata, per esempio, per il cabling degli shelf ai controller NetApp serie 20xx, 40xx, 60xx. Le porte FC-AL sono configurate come NL\_Port in caso di dispositivi collegati tra loro oppure FL\_Port nel caso di una porta su uno switch facente parte di un loop.
3.  **FC-SW**: Chiamata anche Switchted Fabric richiede l’utilizzo di uno specifico Switch e ha una topologia di tipo stellare

### Switch

Nel caso si scelga la topologia FC-SW (la più comune) sarà necessario utilizzare uno o più Switch FC. Il funzionamento è, grossomodo e a livello logico, simile a quello di uno switch Ethernet, con la differenza sostanziale che la politica di switching non è automatica ma è definita dal SAN Administrator.

Tale policy, che può essere di tipo hard o soft (hard comprende la mappatura delle porte fisiche, soft quella dei WWPN), è nota come Zoning. Un insieme di uno o più switch FC collegati tra loro prende il nome di Fabric. Le porte dello switch sono configurate come F\_Port, mentre quelle dei  nodi come N\_Port.

### Trunk/ISL

Più switch Fibre Channel possono essere collegati tra loro configurando le rispettive porte come _E\_Port_. Vari produttori usano protocolli diversi per la creazione di trunk.

È altresì vero che il 90% del mercato è in mano a Brocade quindi ci si riferisce ai trunk usando direttamente il nome ISL che è il loro protocollo.

### Multipath

È buona norma non creare mai dei “single point of failure” all’interno di un datacenter e questo si riflette anche nel progetto di una SAN.

Questo vuol dire che, per esempio, si useranno sempre almeno due Fabric in parallelo e che ogni singolo componente (storage o server che sia) sarà collegato ad entrambe.

Questo porta ad avere più percorsi (o path) per raggiungere un qualunque sistema collegato alla SAN. I path possono essere utilizzati in round-robin o in fault tolerance.

Dato che ogni singola LUN pubblicata su una SAN sarà normalmente trovata dal sistema operativo un volta per ogni percorso (ogni path viene visto come un diverso bus SCSI), lo stessa  risulterà “duplicata” più volte.

Per ovviare a questo problema i sistemi operativi dispongono di driver di multipath, che in questo caso devono essere abilitati, che permettono di eliminare i dischi duplicati e di gestire correttamente le policy d’uso degli stessi path.  

### HBA

Non è null’altro che il controller Fibre Channel inserito all’interno di un server o di uno storage.

### WWN e WWPN

È l’identificativo (solitamente cablato in hardware) di un sistema con HBA Fibre Channel. WWN sta per World Wide Name e WWPN per World Wide Port Name.

Nel caso di HBA con singola porta WWN e WWPN coincideranno, mentre nel caso di HBA multiporta ci sarà un WWN che identifica l’HBA e un WWPN per ogni singola porta di quel controller.

### Velocità

Attualmente Fibre Channel può raggiungere la velocità massima di 16 Gb/sec. Le due velocità più comuni sono 4 e 8 Gb/sec.  I 2 Gb/sec non sono praticamente più utilizzati.

### MTU

Fibre Channel utilizza normalmente frame minime da 2148 byte (questo è da ricordare nel momento in cui si parlerà di FCoE).

Lo standard supporta, al pari dell’ethernet, anche Jumbo Frame fino a 9036 byte, anche se non sono spesso utilizzati.

## iSCSI

Nato come alternativa a basso costo rispetto a Fibre Channel, iSCSI utilizza i tradizionali comandi SCSI utilizzando IP come trasporto.

Il protocollo ha impiegato un certo tempo per affermarsi. Il motivo è da ricercarsi nel fatto che la maggior parte delle implementazioni sono state create totalmente in software e, specialmente all’inizio, hanno manifestato non pochi problemi di compatibilità. Inoltre le performance su reti a 100 Mbit/sec erano decisamente poco appetibili.

iSCSI si è quindi evoluto nel corso del tempo. Sono uscite schede di rete in grado di gestire il protocollo in hardware e di fare boot direttamente da LUN remote iSCSI, così come il supporto a velocità comprese tra 1 e 40 Gbit/sec. Anche le implementazioni software sono drasticamente migliorate eliminando colli di bottiglia e i problemi di compatibilità.

iSCSI si pone ora come un protocollo che possa adattarsi sia alle implementazioni consumer/PMI sia come protocollo mainstream per i grossi datacenter. Ha una velocità di punta superiore a FC anche se paga comunque sia una latenza superiore sia un utilizzo della CPU più elevato (pur nelle implementazioni in hardware).

La fortuna di iSCSI in parte sono stati tutti i sistemi NAS, sia casalinghi sia di fascia più elevata, che lo hanno implementato velocemente side-by-side con i protocolli NAS e CIFS in modo da ottenere degli Unified Storage.

Tutti i prodotti QNAP e Synology implementano ora iSCSI, così come prodotti di fascia più elevata come EMC VNXe o NetApp. Esistono anche block storage che utilizzano esclusivamente iSCSI come, ad esempio gli HP StoreVirtual (ex LeftHand) e Dell Equallogic.

iSCSI permette quindi sia di costruirsi una propria SAN con uno switch di fascia medio/bassa (purché supporti i Jumbo Frame) e un NAS, sia datacenter di un certo livello usando o switch dedicati o VLAN appositamente configurate.

Inoltre iSCSI, viaggiando su IP, permette di effettuare repliche remote utilizzando connettività IP piuttosto che dark fiber come spesso si è costretti a fare con Fibre Channel.

Alcuni storage, come la linea Eternus DX di Fujitsu, permettono di avere anche configurazioni ibride usando, per esempio, Fibre Channel per la connetività locale e iSCSI per la replica remota.

I concetti e le tecnologie da sapere su iSCSI sono:

*   **Initiator**: è il software che si trova sulle macchine client della rete e che permette di effettuare il mounting delle LUN. Tutti i principali sistemi operativi client e server dispongono di un initiator software. Alcune schede di rete di tipo CNA dispongono anche di initiator su firmware e sono quindi in grado di effettuare il boot del sistema operativo direttamente da una LUN iSCSI
*   **Target**: è il software presente sui sistemi iSCSI (server o storage) che devono esportare delle LUN.
*   **iqn**: è un identificativo univoco che rappresenta un nodo sulla rete iSCSI. La sua naming convention è definita dalle RFC 3720 e 3721. iSCSI supporta anche gli indirizzamenti EUI e NAA anche se sono molto meno utilizzati
*   **IP**: iSCSI utilizza IP come trasporto. Quindi ogni nodo deve avere un indirizzo IP per ogni singola porta che possiede. Le porte TCP usate sono solitamente le 860 e 3260. Per chiarire il concetto uno storage con due controller e quattro porte ethernet configurate con iSCSI disporrà, solitamente dato che esistono eccezioni, di quattro indirizzi IP differenti e di uno o due (a seconda delle implementazioni) identificatori iqn.
*   **iSNS**: è l’equivalente del DNS per il mondo iSCSI
*   **LUN**: come in Fibre Channel, le LUN sono mappate attraverso gli iqn. Sul client si inserirà l’iqn del del target. A quel punto l’initiator effettuerà una scansione delle risorse condivise dal target (leggi LUN) e darà all’amministratore la possibilità di mapparle come dischi locali. Sul target solitamente si assegna a ogni LUN l’iqn degli initiator che dovranno avere accesso a quelle LUN.
*   **CHAP**: Oltre al meccanismo di riconoscimento basato sulla verifica dei rispettivi iqn da parte di target e initiator, è possibile abilitare il protocollo CHAP come ulteriore meccanismo di sicurezza.
*   **Multipath**: per aumentare la banda a disposizione o per gestire il failover si può usare o l’aggregazione dei canali tramite trunk LACP/etherchannel oppure il multipath. Il primo meccanismo, per quanto supportato, non è solitamente consigliato. iSCSI gestisce efficamente il multipath e molti produttori di storage ne consigliano espressamente l’adozione specialmente per una corretta gestione della parte di fault tolerance.
*   **Jumbo Frame**: Lo standard Ethernet permette l’utilizzo di frame fino a 9000 byte, al posto dei più tradizionali 1500 byte. Per un efficace utilizzo del mezzo è però indispensabile che tutti i nodi (e lo switch) siano configurati per usare i jumbo frame. Questo è uno dei motivi (oltre ad un banale discorso di congestione) che impongono l’utilizzo di una rete Ethernet fisicamente separata (o in una VLAN differente se gli switch sono sufficientemente veloci) per il trasporto dell’iSCSI. I jumbo frame dovrebbero essere abilitati per tutte le velocità a partire 1 Gb/sec (sui 100 Mbit non fanno praticamente differenza)
*   **Schede CNA**: Sono delle schede specializzate in grado di gestire iSCSI in hardware. Nella maggior parte dei casi sono visti dal sistema operativo come dei controller SCSI. In questo modo sarà possibile effettuare non solo un offload di tutta la parte iSCSI (e evitare di subissare la CPU di interrupt) ma sarà anche possibile lavorare senza utilizzare l’initiator software del sistema operativo installato.

## FCoE – Fibre Channel over Ethernet

Con l’avvento dei 10 Gbit/sec Ethernet i produttori hanno pensato che sarebbe una idea “ficherrima” utilizzare un cablaggio unificato per LAN e SAN.

![Storage_FCoE](https://leganerd.com/wp-content/uploads/2014/07/Storage_FCoE.png?width=800&height=450&quality=75)

Fregandosene altamente del protocollo iSCSI (ritenuto troppo “consumer” e limitato alla sola gestione di dischi) hanno pensato quindi di unificare ethernet e Fibre Channel e di permettere a quest’ultimo di utilizzare Ethernet in fibra come trasporto.

Purtroppo vi sono alcuni problemi:

*   **Consegna**: Fibre Channel è un protocollo a consegna garantita, mentre Ethernet è, a tutti gli effetti, un “best effort”
*   **MTU**: Fibre Channel utilizza una trama minima di 2048 byte che non può essere frammentata

Per ovviare a questi problemi è stato creato lo standard CEE che permette di estendere lo standard ethernet con le caratteristiche di cui sopra. L’idea sulla carta non è male.

I server dispongono di un’unica scheda di rete in grado di fungere sia da scheda LAN sia da HBA Fibre Channel, mentre gli switch possono avere sia porte FCoE (che possono fungere sia da porte LAN sia SAN) sia sole porte Ethernet sia sole porte Fibre Channel (per colelgare gli apparecchi legacy che non supportano FCoE).

Si ottiene quindi sia un risparmio in termini di costi (dovuto al fatto che non è più necessario avere distinti apparati per LAN e SAN, sia al fatto che dispositivi come tape library FC possono essere integrate senza problemi) sia in termini di semplificazione del comparto server (schede uniche) sia di cablaggio del back-end.

Nel mondo reale è un’altra cosa. FCoE evolve lentamente, non permette di collegare più fabric in cascata e i produttori pur supportandolo ufficialmente, tentano in tutti modi di NON vedere soluzioni FCoE. Oramai gli unici che ancora spingono attivamente FCoE sono HP (per la parte server) e NetAPP (per la parte storage).

[Datacenter for Dummies](https://leganerd.com/tag/datacenter-for-dummies/) – Questo articolo è parte di una serie:

1.  [Datacenter for Dummies: Le SAN](https://leganerd.com/2014/07/23/datacenter-for-dummies-san/)
2.  [Datacenter for Dummies: Gli Storage](rmalink:%20https://leganerd.com/2014/07/29/datacenter-for-dummies-storage/)
3.  [Datacenter for Dummies: Networking](https://leganerd.com/2014/08/24/datacenter-for-dummies-networking/)

### Approfondisci su Wikipedia:

*   [Storage Area Network](http://en.wikipedia.org/wiki/Storage_area_network) (wikipedia.org)
*   [Fibre Channel](http://en.wikipedia.org/wiki/Fibre_Channel) (wikipedia.org)
*   [iSCSI](http://en.wikipedia.org/wiki/ISCSI) (wikipedia.org)
*   [FCoE](http://en.wikipedia.org/wiki/FCoE) (wikipedia.org)