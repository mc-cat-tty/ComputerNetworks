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


Per monitorare il traffico sulla propria macchina usiamo `tmux`
Good practice per sessioni remote: lavorare su terminale `tmux`, che è un processo locale, quindi anche in caso di terminazione della connessione il processo rimarrà vivo -> **detachment** del terminale.
Modalità comandi accessibile con Ctrl-B:
- Detachment - *d*
- `tmux attach`  riprende la sessione staccata
- split orizzontale: *""*
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

# Configurazione di rete permanente
#Attenzione: disattiva la configurazione temporanea prima di iniziare quella permanente

`/etc/network/interfaces`

#Nota che importa file da `/etc/network/interfaces.d` -> per evitare di avere un unico file di configurazione monolitico

Configurazione (all'interno del file):
```
iface eth0 inet static
 address 192.168.1.1/24
```

Il file di configurazione viene riletto lanciando `ifup eth0`. Il comando `ifdown eth0` si disattiva la configurazione permanente. Se `ifup`, `ifdown` e `ifconfig` vengono usati in maniera interlacciata il comportamento corretto non è assicurato: `ifup` e `ifdown` usano file temporanei per mantenere in memoria lo stato dlel'interfaccia.

Per la lettura della configurazione a tempo di boot (automatica) serve aggiungere la riga: `auto eth0s`