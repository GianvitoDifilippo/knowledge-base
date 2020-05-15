# Dockerizzazione di una WebApi

Procedura di dockerizzazione di una WebApi con Asp.Net Core, Docker e PostgreSQL.

## Struttura della cartella
Il progetto in esempio si chiama NetSign (versione development).
In questo caso, la cartella di progetto ha la seguente struttura.

```
home
├── Downloads
|   ├── cert
|   |   └── 4EB572F335794A1850ADA2B5FBACEDAF1E56EE23.pfx
|   └── NetSign
|       └── dev
|           ├── pgdata
|           └── res
└── repos
    └── NetSignDev
        ├── database
        |   └── netsign_dev.dump
        └── NetSign
            ├── src
            ├── .dockerignore
            ├── appsettings.json
            ├── docker-compose.yml
            ├── Dockerfile
            ├── NetSign.csproj
            └── nlog.config
var
└── log
    └── Netsign
        └── dev
```

Si notino i seguenti file:

* 4EB572F335794A1850ADA2B5FBACEDAF1E56EE23.pfx: certificato HTTPS
* netsign_dev.dump: dump del database

Nelle sezioni successive, verranno descritte le procedure di creazione dei seguenti file:

* .dockerignore
* Dockerfile
* docker-compose.yml

## .dockerignore

Contiene una lista di file e cartelle che il Dockerfile ignorerà. In questo caso è vuoto.

## Dockerfile

Il file `Dockerfile` (senza estensione) deve trovarsi nella cartella del progetto.
In questo caso, verranno esposte due porte: la porta 80 per l'HTTP e la porta 443 per l'HTTPS.
Se si desiderasse esporre solo una delle due, rimuovere una delle due linee EXPOSE.
Il Dockerfile dovrebbe essere di questo tipo:

```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
EXPOSE 80
EXPOSE 443
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "NetSign.dll"]
```

## Docker-compose

Per rendere indipendenti le varie versioni di un progetto, si dovranno creare diversi container, sia del progetto stesso che di Postgres.
Il file `docker-compose.yml` deve trovarsi nella cartella di progetto, ed è di questo tipo:

```
version: "3"

services:
    postgres:
        image: postgres
        container_name: netsigndev_postgres
        networks:
            - netsigndev
        restart: always
        volumes:
            - pgdata:/var/lib/postgresql/data
            - ./../database/netsign_dev.dump:/var/lib/postgresql/dumps/netsign_dev.dump
        environment:
            POSTGRES_USER: "postgres"
            POSTGRES_PASSWORD: "postgres"
    
    backend:
        image: netsigndev
        container_name: netsigndev_backend
        networks:
            - netsigndev
        environment: 
            - ASPNETCORE_HTTPS_PORT=5004
            - ASPNETCORE_URLS=https://+
        depends_on:
            - "postgres"
        build:
            context: .
            dockerfile: Dockerfile
        volumes:
            - res:/app/res
            - logs:/app/logs
            - /home/Downloads/cert/4EB572F335794A1850ADA2B5FBACEDAF1E56EE23.pfx:/app/4EB572F335794A1850ADA2B5FBACEDAF1E56EE23.pfx
            - ./nlog.config:/app/nlog.config
        ports:
            - "5004:443"

volumes:
    pgdata:
        driver: local
        driver_opts:
            type: none
            o: bind
            device: /home/Downloads/NetSign/dev/pgdata
    res:
        driver: local
        driver_opts:
            type: none
            o: bind
            device: /home/Downloads/NetSign/dev/res
    logs:
        driver: local
        driver_opts:
            type: none
            o: bind
            device: /var/log/NetSign/dev

networks:
    netsigndev:
        driver: bridge
```

Di seguito, il commento alle varie sezioni del file.

### Version
La versione del docker compose che si vuole utilizzare.

### Services
Contiene le impostazioni dei vari container. In questo caso ci sono due container.

postgres:

*  image: l'immagine che si vuole utilizzare (scaricherà l'ultima immagine di postgres se non è già sulla macchina).
*  container_name: il nome che verrà dato al container.
*  networks: le reti di cui questo container farà parte, in questo caso la rete chiamata default.
*  restart: da mettere in "always".
*  volumes: si specifica la cartella utilizzata per le risorse di Postgres (come i database) garantendo la persistenza dei dati anche quando il container viene fermato/rimosso. Inoltre, viene anche copiato il file dump.
*  environment: qui vengono definite le variabili di ambiente, in particolare nome utente e password di Postgres.

backend:

*  image: l'immagine che si vuole utilizzare (ricostruirà l'immagine dal dockerfile se non ne trova una già costruita).
*  container_name: il nome che verrà dato al container.
*  networks: le reti di cui questo container farà parte, in questo caso la rete chiamata default.
*  environment: qui vengono definite le variabili di ambiente, in particolare la porta e l'URL di accesso alla WebApi.
*  depends_on: specifica che il servizio dipende dal servizio "postgres".
*  build: impostazioni di compilazione del progetto, specificando il Dockerfile.
*  volumes: utilizza i volumi "res" e "logs", mappandoli rispettivamente in app/res e app/logs e copia il certificato e il file di configurazione di nlog.
*  ports: associa la porta 5004 dell'host alla porta 443 del container (non c'è la porta per l'HTTP).

### Volumes
Specifica le impostazioni dei volumi. In questo caso ci sono 3 volumi:

* pgdata: La cartella dove postgres salverà i suoi dati, in questo caso /home/Downloads/NetSign/dev/pgdata.
* res: La cartella risorse dell'applicazione, in questo caso /home/Downloads/NetSign/dev/res.
* logs: La cartella dove salvare i log, in questo caso /var/log/NetSign/dev.

### Networks
Crea la rete di default in cui ci saranno i due container.

## Stringa di connessione
La stringa di connessione deve dichiarare che si intende utilizzare il servizio postgres e non localhost:

```
"Server=postgres; Port=5432; Database=netsign_dev; Username=postgres; Password=postgres"
```

## Creare e far partire i servizi
Il seguente comando farà partire la compilazione e l'esecuzione di entrambi i servizi:

```
sudo docker-compose up -d
```

L'opzione -d specifica che l'esecuzione deve essere eseguita in background, tenendo libera la console.

## Popolazione del database

La prima volta che viene creato il servizio, occorrerà creare il database e popolarlo. I comandi sono:

```
sudo docker-compose exec postgres createdb netsign_dev -U postgres
sudo docker-compose exec -T postgres psql -U postgres -d netsign_dev -f /var/lib/postgresql/dumps/netsign_dev.dump
```

A quel punto, il database sarà creato e non ci sarà bisogno di ricrearlo se si ricompila il docker-compose.

## Fermare i servizi

Per vedere la lista dei container attivi, usare il comando:

```
sudo docker ps
```

Per fermare un container attivo, usare il comando:

```
sudo docker stop <CONTAINER ID>
```

Per eliminare un container, usare il comando:

```
sudo docker rm <CONTAINER ID>
```

## Eliminare le immagini

Per vedere la lista delle immagini, usare il comando:

```
sudo docker images
```

Per eliminare un'immagine, usare il comando:

```
sudo docker rmi <IMAGE ID>
```
