Lezione interlacciata con [[7. ICMP]]

```bash
traceroute 8.8.8.8
```

#Nota che l'indirizzo IP di alcuni router non è disponibile. Vengono indicati con `* * *`
# Implementazione
`traceroute` forgia pacchetti IP ad arte, senza passare per il sistema operativo.

In particolare, genera pacchetti con TTL progressivamente crescente, in modo che "muoiano" su router man mano più lontani dalla sorgente. Questi router si palesano inviando un Time Exceeded.

#Attenzione i router sono *tenuti* ma non *obbligati* a segnalare TTL Time Exceeded; spesso per rendere la rete più stealth. Traceroute implementa una logica di timeout, tale per cui un router che non invia Time Exceeded viene comunque individuato.

Cosa posso usare per il probing?
- `-T` TCP SYN
- `-I, --icmp` ICMP ECHO
- Di default vengono inviati datagrammi UDP rivolti a porte improbabili (dalla 33434 in poi) e si attende un *destination port unreachable* dalla destinazione. 

#Attenzione si compie una assunzione potenzialmente sbagliata: *il percorso seguito dai pacchetti in ogni istante di tempo (per ogni TTL) è lo stesso. Quindi due pacchetti successivi condividono un percorso prefisso comune.* -> può non essere vero dato che i protocolli di routing utilizzati su Internet sono dinamici. Si assume che la velocità di esecuzione di traceroute sia tale per cui le routes sia la stessa.

