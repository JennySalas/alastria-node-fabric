# alastria-node-fabric
Alastria network using Hyperledger Fabric (the Open Source project from Linux Foundation)

# Procedure for deploying and joining an organization to the Alastria Hyperledger Fabric network

## Introduction

El siguiente procedimiento tiene cómo objetivo servir de Guía a la hora de facilitar el despliegue de nodos Hyperledger Fabric v1.4.3, los cuales serán contenedore de Docker. En este procedimiento, se plantea como realizar el despliegue de los contenedores Docker, mediante la herramienta Docker Compose.

## Requirements

### Sizing

Requerimientos después de estudio en el Grupo de Arquitectura:

- Peer: 1 CPU  5GB almacenamiento
- Couchdb: 0,5 CPU  5GB  almacenamiento
- Orderer: 0,5 CPU 1 GB RAM  5GB almacenamiento

### Prerequisites

- Ubuntu
- [Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Docker Compose](https://docs.docker.com/compose/install/)

### Networking

La máquina en la que operemos deberá contar con una serie de puertos en los que escucha para que los nodos estén accesible. En el caso del Peer, se utilzia el puerto por defecto (7051)

# Starting up a Peer and its CouchDB

Lo primero, será clonar este repositorio:
```
   git clone https://github.com/alastria/alastria-node-fabric.git
```

## Generate crypto material

Para poder operar en redes Hyperledger Fabris, se necesita que cada organización tenga una serie de claves. En la actualidad, en Alastria la apuesta es que cada socio genere su propio material criptográfico. A continuación, se explica cómo hacerlo utilizando la herramienta **cryptogen**

### Cryptogen tool

Esta herramienta nos genera el material criptográfico necesario, en base a la configuración realizada en el fichero *crypto-config.yaml* La herramienta **cryptogen** nos generará los directorios necesarios y los correspondientes certificados:

1. Será necesario que cada organización actualice el fichero *crypto-config.yaml*, indicando el nombre de la misma y el dominios específico.
   
   ```bash
   
   ## Sustituir `org`
   
   $ nano crypto-config.yaml
   
       PeerOrgs:
         - Name: `org`
       	Domain: `org`.com
       	EnableNodeOUs: true
           Template:
        	Count: 1
           Users:
               Count: 1
  
  
2. Tras ello, procederemos a ejecutar el siguiente comando, el cual nos generará todo el material criptográfico bajo el directorio crypto-config

  ```bash
  
   $ bin/cryptogen generate --config=./crypto-config.yaml
   
  ```

### TODO: Fabric CA



## Update the template

Una vez generado el material criptográfico, procederemos a modoficar el contenido de los ficheros _"docker-compose-izertis.yaml"_ y _"docker-compose-couch-izertis.yaml"_ 

* Es recomendable modificar este fichero adecuadamente con el fin representar la organización a la que va a pertenecer el nodo a desplegar. Será necesario sustituir _Izertis_ por el nombre de la _nueva organizacion_. Es recomendable cambiar también el nombre a docker-compose-`org`.yaml.
* También habrá que tener en cuenta que todas las rutas que apuntan al material criptográfico generado anteriormente, coinciden.


## Deploy the Peer and its CouchDB

El despliegue se hace mediente  **Docker-Compose** para desplegar la red docker con todos sus contenedores. Este despliegue se define en varios archivos .yaml:

- _docker-compose-izertis.yaml_: el principal, que define los nodos que se van a levantar. Este fichero  hace uso del _peer-base.yaml_ para el despliegue de los peer.
- _peer-base.yaml_ es un archivo base para el despliege de cualquier peer.
- _docker-compose-couch-izertis.yaml_: fichero que define la CouchDB asociada al Peer.

Para arrancar los nodos definidos el los ficheros mencionados anteriormente, bastará con ejecutar el siguiente comando:

```bash
  docker-compose -f docker-compose-${ORG}.yaml -f docker-compose-couch-${ORG}.yaml up -d
  # Test peer status and CLI connection:
  $ docker exec -it `org`cli /bin/bash
	$ peer node status
		status:STARTED
```


# Management of the entrance of new Organizations at the network - REQUIRES THE OPERATION OF VARIOUS ORGANIZATIONS

Cada vez que un socio quiera entrar a formar parte de la red de Hyperledger Fabric de Alastria, al ser una red permisionada, necesita ser aceptado por la mayoría de los socios que la conforman hasta ese punto (> 50%). La forma de aceptar a este nuevo socio es coger el último bloque de configuración del canal, descodificarlo, añadir la nueva información criptográfica, codificarlo, firmarlo por los distintos socios y actualizar el canal con este nuevo bloque de configuración.


## Generate the configuration file of the organization in a JSON file - NEW ORGANIZATION

1. Será necesario que cada organización actualice el fichero *configtx.yaml*, indicando el nombre de la misma y el dominios específico.:

   ```bash
   nano configtx.yaml
   
   Organizations:
       - &`org`
           Name: `org`MSP
           # ID to load the MSP definition as
           ID: `org`MSP
           MSPDir: crypto-config/peerOrganizations/`org`.com/msp
           AnchorPeers:
               - Host: peer0.`org`.com
                 Port: 7051
   ```

2. Una vez definida la configuración de la organización, procederemos a crear el fichero JSON:

   ```bash
   bin/configtxgen -printOrg `org`MSP > `org`.json
   ```

3. Por último, modificaremos el fichero JSON generado en el paso anterior, para que se defina en el mismo en Anchor Peer de nuestra organización. Habrá que añadir el correspondiente extracto dentro del fichero JSON generado previamente:

   ```json
   	"values": {
                 "AnchorPeers": {
                   "mod_policy": "Admins",
                   "value": {
                     "anchor_peers": [
                       {
                         "host": "peer0.`org`.com",
                         "port": 7051
                       }
                     ]
                   },
                   "version": "0"
                 },
   			  "MSP": {
   ```

4. Una vez generado el JSON de la Organización correctamente, se lo enviaremos a Alastria (o correspondiente socio) para su validación y aceptación en la Red H.

## Update the configuration block and the channel - ALASTRIA/VALIDATING ORGANIZATION(S)

### Update the configuration block

Lo primero, será obtener el bloque de configuración para proceder a su modificación.

```bash
# Acceder al contenedor cli de la organización
docker exec -it `org`cli /bin/bash
  # Exportar variables de entorno
  $ export CHANNEL_NAME=alastriachannel
  $ export ORDERER_TLS_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/alastria-ca-tls.pem
  # Obtener bloque de configuración
  $ peer channel fetch config config_block.pb -o orderer0:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_TLS_CA
  # Convertirlo a formato JSON
  $ configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
  # Añadir al bloque de configuración la información relativa a la nueva organización
  $ jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"`org`MSP":.[1]}}}}}' config.json ./organizations/peerOrganizations/`org`.`dominio`/`org`.json > modified_config.json
  # Convertir los bloques de configuración antiguo y el nuevo y, obtener de la comparativa de ambos el binario necesario para la nueva organzación
  $ configtxlator proto_encode --input config.json --type common.Config --output config.pb
  $ configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
  $ configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output `org`_update.pb
  # Convertir este último bloque de actualización
  $ configtxlator proto_decode --input `org`_update.pb --type common.ConfigUpdate | jq . > `org`_update.json
  $ echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat `org`_update.json)'}}}' | jq . > `org`_update_in_envelope.json
  $ configtxlator proto_encode --input `org`_update_in_envelope.json --type common.Envelope --output `org`_update_in_envelope.pb

```

### Sign the configuration block - DEPENDING ON GOVERNANCE WILL REQUIRE ONE OR MULTIPLE SIGNATURES

Una vez tengamos el bloque de configuración modificado, habría que proceder a su firma:

```bash
# Conectarse al contenedor CLI
docker exec -it `org`cli /bin/bash
  # Exportar variables de entorno
  $ export CHANNEL_NAME=alastriachannel
  $ export ORDERER_TLS_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/alastria-ca-tls.pem
  $ peer channel signconfigtx -f `org`_update_in_envelope.pb
```

### Update the channel - IT WILL BE DONE BY THE ORGANIZATION THAT SIGNS IT LAST

```bash
# Acceder al contenedor cli de la organización
docker exec -it `org`cli /bin/bash
  $ export ORDERER_TLS_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/alastria-ca-tls.pem
  # Si eres el último socio que va a firmarlo: se firma y se actualiza el canal con el bloque
  $ peer channel update -f `org`_update_in_envelope.pb -c alastriachannel -o orderer0:7050 --tls --cafile $ORDERER_TLS_CA
```

Ahora solo faltaría que la nueva organización/socio se uniese al canal, para lo cual tendrá que ejecutar los pasos que se listan en el siguiente apartado.

## Join Alastria H network (join the channel) - NEW OEGANIZATION

Para unirse al canal, es necesario primero coger el bloque de configuración del canal o el bloque 0 (hacer un fetch). Para esto se necesita saber:

1. Nombre del canal: _alastriachannel_

2. DNS de algún Orderer y el puerto de escucha del mismo:
   - _orderer0: 22.99.11.88     puerto:7050_

3. Ruta local del peer al certificado Root del Orderer para la comunicación TLS: _"/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/alastria-ca-tls.pem"_, el cual se tendrá que copiar al contenedor `org`cli

### Configure de DNS at Peer and CLI containers

Acceder a ambos contenedores de docker y actualizar el fichero /etc/hosts

```bash
# CLI
docker exec -it `org`cli bash
  $ echo "22.99.11.88 orderer0" >> /etc/hosts
  $ exit
# PEER
docker exec -it peer0.`org`.`com` bash
  $ echo "22.99.11.88 orderer0" >> /etc/hosts
  $ exit
```

### Fetch the configuration block and Join the channel

De nuevo, desde el contenedor del CLI ejecudtar los siguientes comandos:

```bash
# Conectarse al contenedor CLI
docker exec -it `org`cli /bin/bash
  # Exportar variables de entorno
  $ export CHANNEL_NAME=alastriachannel
  $ export ORDERER_TLS_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/alastria-ca-tls.pem
  # Fetch del bloque génesis del canal
  $ peer channel fetch 0 alastria-channel.block -o orderer0:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_TLS_CA
  # Join del canal
  $ peer channel join -b alastria-channel.block
```

### Validate the configuration using the test chaincode

Para ver que todos los permisos y certificados son correctos y que el funcionamiento del canal y el peer es adecuado, se instala el chaincode y se prueba a hacer querys e invokes contra él.

```bash
# Conectarse al contenedor CLI
docker exec -it `org`cli /bin/bash
  $ cd /opt/gopath/src/github.com/chaincode
	$ peer chaincode install alastria-example-chaincode.out
	# Listar los chaincode instalados e instanciados
	$ peer chaincode list --installed
  $ peer chaincode list --instantiated -C alastriachannel
	# Query chaincode
	$ peer chaincode query -C alastriachannel -n signeCC -c '{"Args":["query","a"]}'
	# Invoke chaincode
	$ peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_TLS_CA -C $CHANNEL_NAME -n signeCC -c '{"Args":["invoke","a","b","3"]}'
```
