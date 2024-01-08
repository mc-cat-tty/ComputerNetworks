# Segmentazione e segregazione
>**Segmentazione**: partizionare le risorse del sistema da un POV logico/fisico

>**Segregazione**: controllare gli accessi tra risorse segmentate. Meccanismi per il controllo delle risorse. Segmentare è una condizione necessaria per segregare.

Le tecniche di segmentazione e segregazione possono essere forzate con un diverso numero di tecniche in base ai protocolli in uso.

>**Defense-in-depth**: stratificazione della sicurezza, su diversi livelli (eg. host e rete). Modello _a cipolla_.

## L2 - VLAN
Le VLAN sono una tecnica di segmentazione L2.

## L3 & L4 - Firewall
Un firewall L3 usa regole basate sul contenuto dell'header di livello 3, come indirizzi IP e protocolli.

Un firewall L4 usa regole basate su porte o protocolli di livello trasporto.

## Higher levels
>**DPI** - Deep Packet Inspection: firewall che ispezionano i segmenti ad un livello superiore. Vanno oltre il livello 4.

>**Application Layer Firewall**: ulteriore livello rispetto alla DPI. Il firewall implementa completamente la logica del protocollo applicativo.

Anche il NATting è una tecnica di segmentazione, in quanto separa lo spazio di indirizzamento delle reti. Può essere considerata anche una tecnica di segregrazione, implementando opportune regole.

# Tassonomia
## Policy di sicurezza
>Le policy di sicurezza sono definite da ACL - Access Control List.

Esistono due approcci tipici:
- **implicit negation**: blocca tutto, permetti un po'
- **implicit allow**: permetti tutto, blocca un po'

Il primo approccio favorisce la sicurezza rispetto all'usabilità. Il secondo approccio il contrario.

>Si usano regole di **block/black/deny list** e **white/allow list**

## Tassonomia dei firewall
Distribuzione:
- **hardware**: hardware dedicato, come dispositivi Cisco 
- **software**

Deplyment:
- **host**: installato sul dispositivo terminale
- **network**: installato sul dispositivo di rete

Protocolli supportati:
- L3 e L4
- application gateway

Tipo di analisi:
- static/**stateless**: non conosce lo stato della connessione; comportamento ad eventi; decisioni di filtraggio prese sulla base delle informazioni contingenti al pacchetto.
- **stateful**: eg. traffico asimmetrico di una connessione TCP, sessioni HTTP mediante cookie; il pacchetto è analizzato sulla base di informazioni contenute e contesto; esiste la nozione di connessione; permette di filtrare appplicazioni con stati complessi.

# Firewall
## Deployment
I firewall abilitano alla segmentazione della rete. Possono essere posizionati strategicamente su dispositivi di rete (filter **network traffic**) o su dispositivi terminali (filter **host traffic**).

In base al dispositivo su cui il firewall è installato, il dispositivo prende il nome di:
- **screening router**: tipicamente il fw è installato sul boarder router. Implementa regole semplici di filtraggio (L3/4, no DPI) ed esegue una prima scrematura. Filtra secondo una logica stateful.
- **proxy firewall**: lavorano a livello applicativo e hanno una comprensione integrale del traffico che gira in rete. Si mette in mezzo alla comunicazione e applica regole di analisi. Esegue un'ispezione profonda del traffico.
- **stealth router**: assomiglia a uno screening router, ma è pensato per non essere rilevabile all'interno della rete, per esempio per fare _intrusion detection_. Considerato alla stregua di un dispositivo L2 (sono simili ai router, ma non sono facilmente rilevabili). Eg. non decrementa il TTL. Utile per evitare attacchi al firewall stesso.

>Il **proxy** è un dispositivo di rete che si interpone nelle comunicazioni, gestendo le connessioni verso i due capi, a livello applicativo

#Nota cifrando le comunicazioni gli strumenti di analisi intermedi (firewall) non possono ispezionare i payload dei pacchetti. Usare un proxy in questo caso può essere la soluzione.

## Network design patterns
- **single screening router**: router di bordo + screening; scala male, niente segregazione della rete interna.
- **dual-home router**: firewall dedicato con due interfacce - trusted and untrusted
- **screened-host gateway / bastion host**: estende l'architettura vista sopra aggiunendo un host per ispezionare comunicazioni tra trusted and untrusted network al livello applicativo; aggiunge un livello di sicurezza e migliora la scalabilità: lo screening router scarta il grosso dei pacchetti (just packet inspection), mentre il bastion host filtra a livello applicativo (maggiori risorse) quelli che passano la prima linea di difesa. Alcune sottoreti, come una DMZ, possono evitare il bastion host e passare solo attraverso il packet filtering.
- **screened subset**: difesa multi-strato; combina quanto visto fin'ora inserendo uno screening router al confine di una o più reti interne.

>**Bastion host**: host che implementa funzionalità di proxy firewall. Si trova a livello degli host e tutto il traffico degli utenti deve passare attraverso ad esso. Espone pochi servizi per ridurre la superficie di attacco.

>**DMZ** - De-Militarized Zone: area intermedia soggetta a meno controlli rispetto al resto della rete. Solitamente espone servizi, infatti gli host appartenenti alla DMZ possono comunicare direttamente alla rete esterna. Si noti però che sono più esposti ad attacchi.

Esempio di **defence-in-depth** e **separazione**, con un primo livello di sicurezza applicato a all'intera rete:
```
Internet <---> Router + Firewall <--> Bastion Host
					  ^                    ^
					  |                    |
					  v                    v
				     DMZ                User net
```

## Screened subnet
Un design più ottimizzato, che evita di far passare tutto il traffico attraverso il bastion host, che probabilmente porterebbe al collasso della rete: external e internal screening router. Bastion host per connessioni stringenti sulla sicurezza. DMZ che evita tutte le regole interne.

# Iptables
```
iface IN ---> logica di routing -Forwarding-> ROUTING -2-> iface OUT
                 |Input                         ^
                 v                              |
          processi locali  <-------  logica di routing
                           -Output->
```

Useremo la tabella `filter`, che ha le seguenti catene:
- `INPUT`: filtraggio traffico entrante a livello di host
- `OUTPUT`: filtraggio di traffico uscente a livello di host
- `FORWARD`: ha senso lavorare su questa catena per firewall a livello di rete (eg. sul border router)

`iptables` lavora di default sulla tabella `filter`.

## Scelta della policy
Le policy che possiamo scegliere sono (`-P`):
- `INPUT ACCEPT` -> *allow all* (default)
- `INPUT DROP` -> *deny all*

#Attenzione instaurare una connessione TCP richiede non solo pacchetti uscenti, ma anche entranti. An input policy on the firewall could affect TCP connections. UDP connections are not affected by input policies.
## Eccezioni
Possiamo aggiungere delle eccezioni alle regole di default specificando **condizione** e **target**.

Target (`-j):
- `ACCEPT`
- `REJECT`
- `DROP`

Condizioni:
- `-i <input-iface>`
- `-o <output-iface>`
- `-s <source-ip/net>`
- `-d <destination-ip/net>`
- `-p {udp, tcp} --dport <dst-port> --sport <src-port>`
- `-m state --state <states>` - permette di configurare condizioni di match basate sullo stato

Eg: `iptables ... -m state --state NEW,ESTSABLISHED -j ACCEPT`

#Vedi `RELATED` state

Altre condizioni utili:
- `-p icmp --icmp-type {echo-request, echo-reply}`
## Permanenza regole
Possiamo gestire la permanenza con `post-up` e `post-down` in *ifconfig*, ma esiste un tool apposito: `iptables-persistent`.

In particolare, le regole possono essere salvate sul file `/etc/iptables/rules.v4`.

```bash
iptables-save > destination_file
iptables-restore < source_file
```

#Consiglio puoi usarlo per creare checkpoint durante la configurazione, salvando l'output su file che non siano `rules.v4`

#Nota che `iptables` lavora su IPv4, la versione `ip6tables` lavora su IPv6
