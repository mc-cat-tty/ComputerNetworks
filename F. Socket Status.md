Comando storico: `netstat`
Comando moderno: `ss` -> socket status

Non mostra solamente connessioni con host esterni, ma anche connessioni aperte tra processi dello stesso sistema operativo. Un socket viene visualizzato come un file sui fs unix.

Opzioni (valide sia per Netstat che per SS):
- `-u` UDP
- `-t` TCP
- `-n` Not resolve
- `-l` listen. Senza vengono visualizzate le connessioni attive, ovvero che generano traffico.

Quando un server si mette in ascolto su una porta può specificare anche un indirizzo da cui accettare *incoming connections*. Ha senso in due casi particolari:
- `127.0.0.1/8` loopback, in IPv6 `[::1]`
- `0.0.0.0` connessioni dal mondo esterno

Esistono notazioni più specifiche come `127.0.0.1%lo` -> che specifica l'interfaccia di loopback

`-p` process -> visualizza il processo a cui è bindata la porta

## Netcat
```bash
nc -v -l [-t|-u] -p 8080
nc [-t|-u] 127.0.0.1 8080
```

- `-v`verbose
- `-t` TCP
- `-u` UDP
- `-l` listen
- `-p` port

La scelta del protocollo deve essere consistente.

#Ricorda L'opzione `-X` di `tcpdump` visualizza i payloads

#Nota quando "apro" la connessione (che poi... è connectionless) non succede assolutamente nulla, finché non gli dico esplicitamente di mandare dei dati.

La risposta di un server UDP è da intendersi come risposta al solo livello applicativo

Si può forzare la porta sorgente del client con `-p`
## Advanced tcpdump
```
tcpdump -Xqnni lo udp and port 8080
```

La doppia `n` disabilita la risoluzione dei nomi delle porte

#Nota l'errore `connection refused`. Per una logica a maggior livello di astrazione di netcat crea un'associazione forte con un certo client. Quando questo muore, un client diverso (diverso IP o diversa porta) non può inviare pacchetti al server, che li rifiuterà. Se i pacchetti nascono dalla stessa sorgente, allora non ci saranno problemi ad accettarli.

## Socket through proc
```bash
echo "ciao!" | /proc/udp/127.0.0.1/8080
```

Attraverso il filesysem virtuale la risorsa fittizia `/127.0.0.1/8080`, normalmente non presente, viene utilizzata per inviare un messaggio UDP alla destinazione specificata.