# Introduzione
>Il NATting è una tecnica che permette di mappare più indirizzi IP privati in un unico indirizzo IP pubblico, mediante un router "di bordo". Una rete che risponde a queste caratteristiche - in cui il traffico esce mediante un unico percorso comune - è definita dall'RFC _stub network_

#Vedi RFC1631
#Nota non tutte le reti fanno NAT, soprattutto a livello enterprise

Come funziona? l'idea di base è modificare il SRC IP per i pacchetti in uscita e il DST IP per i pacchetti in in entrata -> compito della **NAT box**
## Internals
Non si riesce a fare NATting coinvolgendo solamente il livello 3.

Il NATing implica sempre anche il PATing - Port Address Translation - cosicché le trasformazioni non siano ambigue.

Ogni comunicazione è identificata da una tripla: IP - PORTA - L4 PROTO

Quando L4 non è coinvolto, come in ICMP, si potrebbero creare conflitti. Nel caso di ICMP si usa il numero di id.
#Prova a pensare come forzare conflitti. Suggestion: usa protocolli stateless come UDP o ICMP.

#Attenzione il router di bordo potrebbe modificare i checksum di header e payload.

## Binding
>Il router gestisce la corrispondenza (**binding**) tra (indirizzo interno, porta interna, protocollo) e (indirizzo esterno, porta esterna, protocollo). Questa gestione può avvenire staticamente o dinamicamente.

Binding statico: regole di binding definite dall'amministratore
Binding dinamico: regole di binding generate dal router secondo alcuni criteri per disambiguare
## SNAT
>Il **Source NATting** consente al router di bordo di modificare gli indirizzi IP sorgente in uscita dalla rete privata. In questo caso si utilizza **binding dinamico**.

## DNAT
>Il **Destination NATting** è il processo di trasformazione inverso: i pacchetti vengono originati dalla rete pubblica e devono entrare in una rete privata. Si sceglie tipicamente **binding statico**.

#Attenzione È diverso da dire che il pacchetto esce con SNAT e ritorna nella rete. Nel caso del DNAT il "mondo esterno" vuole raggiungere un dispositivo all'interno della rete.

Potrebbe diminuire la sicurezza della rete.

#Nota non si possono avere più host interni **esposti** sulla stessa porta, ma devono essere esposti su porte differenti (non standard).

Default: no-by-default.
## Contro
Distrugge la semantica della comunicazione peer-to-peer / host-to-host.

# Sperimentazione
## Hooks
Su Linux esistono due tool per lavorare con packet filtering, inspection e NAT:
- `iptables`
- `nftables` (+ moderno)

```
iface IN -x-> logica di routing -x-> ROUTING -x-> iface OUT
                 |x                    ^
                 v                     |
          processi locali  <-x->  logica di routing
```

Su ogni arco marcato con `x` possono essere impostati degli *hook*

## Chains
Ad ogni hook possono essere associate più regole, dette **catena** (chain), che vengono disambiguate mediante un'esecuzione **sequenziale** delle regole appartenenti alla catena.
## Tabelle
Le tabelle di iptables raggruppano le catene in base alla funzionalità su cui agiscono le catene. Esistono 3 tabelle:
- **Filter**: filtraggio di pacchetti, ovvero cosa accettare e cosa scartare
- **Mangle**: marking di pacchetti e modifiche ai campi ToS e TTL
- **Nat**: trasformazione di pacchetti per le funzionalità di:
	- Masquerading
	- Port forwarding
	- Transparent proxy

### NAT
La tabella *nat* definisce tre catene di default:
1. PREROUTING: DNAT
2. POSTROUTING: SNAT
3. OUTPUT: DNAT per processi locali

#Nota che la differenza tra OUTPUT e PREROUTING sta nel fatto che OUTPUT è applicata al traffico in uscita a processi locali

```
iface IN -1-> logica di routing ---> ROUTING -2-> iface OUT
                 |                     ^
                 v                     |
          processi locali  <---  logica di routing
                           -3->
```

#Nota spesso i router non generano traffico, quindi la regola di OUTPUT non viene utilizzata

#Prova ad impostare un SRC IP impropriamente sulla regola di source natting

## Debugging
Il debugging di reti a livello 4 si esegue con Netcat.

## IP Tables
### SNAT
In lettura:
```bash
iptables -L
```

Implicitamente la tabella è quella di `filter`

```bash
iptables -t nat -L [-vn]
```
- `-v` - verbose
- `-n `- non risolvere

```bash
iptables -t nat -A POSTROUTING -o eth1 -j SNAT --to-source 2.2.2.2  # Statico
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

#Nota che le opzioni non sono simmetriche. Per esempio `-i` non può essere usato per POSTROUTING perché il SO non conosce da dove proviene il pacchetto

- `-s` SOURCE_NET - posso specificare opzioni di applicazione in base alla rete di ingresso

#Vedi come entrano ed escono i pacchetti sulle due interfacce

Eliminazione:
```bash
iptables -t nat -D POSTROUTING NUM
```
Dove NUM è l'indice della regola, visibile con l'opzione `--line-numbers` in lettura (`-L`)

Replace (sovrascrive una regola già esistente - equivalente a delete+append):
```bash
iptables -t net -R POSTROUTING 1 ...
```
Rimpiazza la regola 1

Insert:
```bash
iptables -t nat -I POSTROUTING 1 ...
```
Inserisce la regola in posizione 1 -> più specifica
#Ricorda il setaccio: dalle più specifiche alle meno specifiche

### DNAT
```bash
iptables -t nat -A PREROUTING -d 2.2.2.2 -i eth1 -j DNAT --to-destination 192.168.1.1
```


#Attenzione iptables mantiene uno stato/regole finché il flusso di comunicazione non si interrompe

### DNAT L4
Obiettivo: voglio esporre particolari servizi di rete, serviti da una macchina della rete locale.

```
iptable -t nat -A PREROUTING -i eth1 -d 2.2.2.2 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.1:8080
```

```bash
ss -ntlp
```

Se provo ad aprire la porta con Netcat e non ho errori significa che il 3-way handshake è stato concluso propriamente.

```bash
nc -lp 8080
nc 2.2.2.2 80
```

#Attenzione con UDP non noterai mai errori in apertura della connessione, perché è un protocollo stateless. IP Tables usa un suo stato interno, basato su un timeout, per determinare se la sessione è ancora aperta.