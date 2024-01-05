Ctrl-M per creare nuovi host
Almeno 64Mb di memoria
Usa cavi incrociati per dispositivi sullo stesso livello dello stack, cavi dritti per dispositivi dissimili.

```
ifconfig eth0 # ottiene informazioni
ifconfig eth0 192.168.1.1/24 # imposta indirizzo IP
ifconfig eth0 192.168.1.1/24 down # imposta indirizzo IP, ma la spegne a lv hw

ifconfig eth0 0.0.0.0 up # accende if fisicamente, ma no livello rete
ifconfig eth0 0 up # sopra in versione breve
```

`BROADCAST MULTICAST` -> significa implicitamente che è spenta, altrimenti ci sarebbe `UP` & `RUNNING`

Come capire cosa succede a livello MAC?
- posso vedere la ARP table: `arp`, `ip neigh` (ci dà più info: STALE, REACHABLE, DELAY)

```
ip n flush all
```

*lladdr* significa Link-layer address.
#Nota alcuni SO possono scegliere di inviare l'arp request in unicast per validare un'entry della ARP table

# Sniffing
```
tcpdump -i eth0 # if in modalità promisuca e analisi traffico
```

#Attenzione potrebbe nascondere le informazioni "meno interessanti", come quelle h2n -> `-e`

Con `-n` chiedo di non interpretare i dati di basso livello

```
tcpdump -eni eth0
```

Lettura info:
- time stamp
- mac src > mac dst (grazie a `-e`)
- ethertype (all'interno del frame header)

#Nota differenza tra [ARP] rinnovo (spesso unicast: se rispondono rinnovo, altrimenti invio broadcast) vs discovery

Transizione da modalità normale a promiscua (all'avvio di tcpdump):
```
dmesg -w
```

# Tmux
Per monitorare il traffico sulla propria macchina usiamo `tmux`
Good practice per sessioni remote: lavorare su terminale `tmux`, che è un processo locale, quindi anche in caso di terminazione della connessione il processo rimarrà vivo -> **detachment** del terminale.
Modalità comandi accessibile con Ctrl-B:
- Detachment - *d*
- `tmux attach`  riprende la sessione staccata
- split orizzontale: *"*
- split verticale: *%*
- navigazione tra split: frecce
- creazione tab - *c*
- zoom split: *z*
- navigazione tra tab: \<numero-tab\> (e.g. 0, 1, ...), oppure *p* ed *n* -> previous and next

# Arping
```
arping -i eth0 192.168.1.1
```

Forza l'utilizzo del protocollo arp per pingare la presenza di un dispositivo nella stessa rete locale. Non funziona per dispositivi su reti differenti.

È un tool di basso livello, non fa controlli su presenza del dispositivo o simili. Utile per aggirare stealth response sul livello 3.

Senza aver configurato L3: `arping -0Bi eh0`

# Configurazione di rete permanente
#Attenzione: disattiva la configurazione temporanea prima di iniziare quella permanente

`/etc/network/interfaces`

#Nota che importa file da `/etc/network/interfaces.d` -> per evitare di avere un unico file di configurazione monolitico

Configurazione (all'interno del file):
```
iface eth0 inet static
 address 192.168.1.1/24
```

Il file di configurazione viene riletto lanciando `ifup eth0`. Il comando `ifdown eth0` si disattiva la configurazione permanente. Se `ifup`, `ifdown` e `ifconfig` vengono usati in maniera interlacciata il comportamento corretto non è assicurato: `ifup` e `ifdown` usano file temporanei per mantenere in memoria lo stato dell'interfaccia.

Per la lettura della configurazione a tempo di boot (automatica) serve aggiungere la riga: `auto eth0`

#Nota: di base, nell'output di `ifconfig`, appaiono solo le interfacce attive, per questo serve specificare l'interfaccia di cui vogliamo vedere lo stato se non attiva.

# Hostnames
> Risoluzione locale degli indirizzi di rete

Inserisco in `/etc/hosts/:
```
127.0.0.1 h1
192.168.1.1 h1  # Conflitto
192.168.1.2 h2
192.168.1.3 h3 pippo pluto
```

#Nota un indirizzo può avere nomi multipli, come nel caso di *h3*
#Nota posso scoprire come viene risolto un conflitto tipo quello di *h1* sia sperimentalmente (provo ad eseguire il ping) oppure mediante il manuale

Quando viene letto `/etc/hosts`? lo scopro con `strace`

Traggo beneficio anche con l'utilizzo `tcpdump` -> si motiva l'opzione `-n` (non interpretare gli IP mediante `/etc/hosts`). `-nn` è un non-interpretare ancor più rigida

Questo metodo non è scalabile, sforzo che cresce in N^2 con il numero di host, vedremo DNS.
# Namespace Switch
File `/etc/nsswitch.conf`

Con quale priorità risolvo gli hostnames? Il file di configurazione contiene `hosts: files dns` che impone l'ordine:
1. file del sistema operativo
2. sistema DNS (normalmente rimosso nei sistemi connessi a internet)

#Prova a rimuovere la entry `dns`

# Approfondimento
![[PendingIfArp.png]]

```
root@h2:~# ip n
192.168.2.1 dev ethe lladdr 02:04:06:b1:33:62 STALE
192.168.1.1 dev etho Llador 02:04:06:b1:33:62 STALE
```
 
 *h2* riesce a pingare eth1 - anche con arp - passando per l'interfaccia *eth0*. Il "vicinato" di *h2* consiste quindi in 2 dispositivi di LV3 mappati sullo stesso indirizzo LV2 (h2n per la precisione).
## Note
- Con le due if eth0 e eth1 connesse allo stesso switch la popolazione dell'ARP cache non è consistente -> se pingo da h2 in base all'ordine di risposta (dato che rispondono entrambe le if di h1) popolo la tabella in modo diverso
- Quando eth1 è disconnessa si riesce comunque a pingare il suo IP passando per eth0. Com'è possibile? il protocollo ARP non è gestito dalla NIC - anche mediante il campo di multiplazione *type* ? la richiesta ARP viene passata al SO?? nella ARP cache sono consentiti più IP address per ogni MAC address?!

https://security.stackexchange.com/questions/112683/linux-responds-to-arp-requests-for-other-interfaces-could-this-be-a-security-v