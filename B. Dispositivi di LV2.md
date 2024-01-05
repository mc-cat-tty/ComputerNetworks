# Hub
Tutti i frame vengono inoltrati a tutti i dispositivi connessi
# Switch 
Opzioni:
- VDE - Virtual Distributed Ethernet - terminale per gli switch managed. Gestione degli switch da simil-seriale.
- FSTP - Fast Spanning Tree Protocol - utile in caso di link ridondanti
- Startup configuration - per eseguire comandi all'avvio ma non a runtime

Cosa mi aspetto di vedere?

## Terminale VDE
Lista comandi disponibili:
```
help
```

Mostra la tabella porta-mac:
```
hash/print
hash/showinfo
```

## Esperimento
Abbassa a 1 secondo l'expire time dello switch mentre viene inviato una serie di pacchetti ICMP da H1 a H2 con `tcpdump` eseguito su H3.
```
hash/setexpire 0
```

Sospendi H2

Cosa succede?
- un bel po' di richieste ICMP, senza risposta (la sequenza su H1 non va avanti)
- ad un certo istante scade il TTL della tabella ARP
- iniziano a viaggiare delle ARP request e, contemporaneamente, si manifesteranno i primi *Host Unreachable*

# Virtual Bridge
```
brctl addbr BRIDGENAME
brctl addif BRIDGENAME INTERFACE
brctl show [BRIDGENAME]
```

Un bridge è un'interfaccia virtuale, gestita dal sistema operativo, 

Lancia `brctl` per la lista di verbi disponibili
Un bridge espone la stessa interfaccia di una interfaccia di rete virtuale vera e propria -> polotabile con ifconfig

Un bridge, come uno switch, si autoconfigura. Un bridge non ha indirizzi IP o simili.

```
brctl showmacs br0
```

Il bridge, in contesti virtuali, è utilizzato per mettere in comunicazione tante VM.
Può agire come interfaccia di nodo intermedio o come interfaccia di nodo terminale. Assegnare un indirizzo IP ad un bridge può essere utile nel secondo caso, se risiedono sulla stessa macchina più VM, oppure se si vuole dare un indirizzo a un dispositivo intermedio (primo caso); se così si fa.

#Ricorda `ifconfig ethX up` con X = 0, 1, 2; lo stesso vale per br0

Configurazione automatica (non necessario che `br0` esista):
```
auto br0
iface br0 inet static
 bridge_ports eth0 eth1 eth2
 address 0.0.0.0  # 0.0.0.0 indica che NON voglio specificare l'indirizzo IP
```
#Attenzione: *tra il dire e il fare c'è di mezzo Linux* -> tutte le direttive `iface` richiedono un `address` -> ultima riga, anche se strano

#Attenzione il file potrebbe non attivare tutte le interfacce fisiche associate al bridge
Me lo posso assicurare con:
```
iface eth0 inet static
 address 0.0.0.0

iface eth1 inet static
 address 0.0.0.0

iface eth2 inet static
 address 0.0.0.0
```

Il comando `ifup -a` esegue tutte le direttive del file `interfaces` -> poco consigliato per troubleshooting sulla singola interfaccia

#Attenzione Usa il comando `ifconfig ethXXX 0` per pulire lo stato dell'interfaccia 