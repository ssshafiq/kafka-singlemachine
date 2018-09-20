
![](/images/mtbclogo.png)
# SETUP KAFKA BASE ORDERING SERVICE WITH HYPERLEDGER COMPOSER
A cookbook of Kaka base ordering service

MTBC IT-DEPARTMENT







# Introduction

The Hyperledger Fabric has introduced Kafka as it&#39;s primary consensus mechanism among the orderers. While in development, for testing purposes, a solo config orderer is used. However, in production, you need to have multiple orderer nodes set up to have fail proof systems. In the case of a hardware/software failure, this is what will rescue you from critical situations. Kafka helps implement this easily. To understand in detail how all of this works, refer to this article [TODO].

# Assumptions

I am starting this tutorial with following version of Hyperledger Composer &amp; node js. If you have version other than this it may give you different error.

## Composer version

Using Composer version 0.19.11

You can check this by typing following command
```
$ composer -v
```
![](/images/composerinstall.png)


## Composer reset server version


Using composer-reset-server 0.19.11
```
$ composer-rest-server -v
```
![](/images/restserver.png)
## Node version

Using node version v8.11.1
```
$ node -v
```
![](/images/nodeversion.png)
 

Before starting tutorial you should have same version of composer, composer-rest-server &amp; composer-playground if needed. You can install a specific version by mentioning version after @ sign

```
$ npm install -g composer-rest-server@0.19.11
```


# Network Architecture

We have the following architecture

- 3 Orderers.
- 1 Organizations.
- 2 peers
- 4 Kafka broker instances.
- 3 Zookeper instances.

# Steps to setup Kafka base ordering service

First of all download Kafka code from this link provided bellow
```
git clone https://github.com/skcript/Kafka-Fabric-Network.git
```
![](/images/gitrepo.png)

After cloning we have following files and directories

![](/images/directoriesStructure.png)
 

Make following changes in configtx.yaml file

```javascript
---

Profiles:

    TwoOrgsOrdererGenesis:

        Orderer:

            \&lt;\&lt;: \*OrdererDefaults

            Organizations:

                - \*OrdererOrg

        Consortiums:

            SampleConsortium:

                Organizations:

                    - \*Org1

    TwoOrgsChannel:

        Consortium: SampleConsortium

        Application:

            \&lt;\&lt;: \*ApplicationDefaults

            Organizations:

                - \*Org1

Organizations:

    - &amp;OrdererOrg

        Name: OrdererOrg

        ID: OrdererMSP

        MSPDir: ../crypto-config/ordererOrganizations/example.com/msp

    - &amp;Org1

        Name: Org1MSP

        ID: Org1MSP

        MSPDir: ../crypto-config/peerOrganizations/org1.example.com/msp

        AnchorPeers:

            - Host: peer0.org1.example.com

              Port: 7051

Orderer: &amp;OrdererDefaults

    OrdererType: kafka

    Addresses:

        - orderer0.example.com:7050

        - orderer1.example.com:7050

        - orderer2.example.com:7050

    BatchTimeout: 2s

    BatchSize:

        MaxMessageCount: 10

        AbsoluteMaxBytes: 99 MB

        PreferredMaxBytes: 512 KB

    Kafka:

        Brokers:

            - kafka0:9092

            - kafka1:9092

            - kafka2:9092

            - kafka3:9092

    Organizations:

Application: &amp;ApplicationDefaults

    Organizations:
```

in  crypto-config.yaml remove the org 2 so that the updated file look like this

```javascript
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Template:
      Count: 3
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    Template:
      Count: 2
    Users:
      Count: 1
 

```
Add CA in docker-compose-cli.yaml file the updated file will look like this

```javascript
version: '2'

networks:
    behave:

services:
  cli:
    container_name: cli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
      - CORE_CHAINCODE_KEEPALIVE=10
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ../chaincode/:/opt/gopath/src/github.com/chaincode
        - $GOPATH/src/github.com/hyperledger/fabric/:/opt/gopath/src/github.com/hyperledger/fabric/
        - ../crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
    networks:
        - behave
  ca.org1.example.com:
      image: hyperledger/fabric-ca:x86_64-1.1.0
      environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca.org1.example.com
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/f22de251dfdfbe2cd433d064727cf3591befa56a09de261b2e94633d0a583828_sk
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=behave #${CORE_PEER_NETWORKID}_
      ports:
      - "7054:7054"
      #command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/52dc508d562d25d8d7078f1c3fa7aaa44b82b7728c123cc50394655b5a580c75_sk -b admin:adminpw -d'
      command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
      volumes:
      - ../crypto-config/peerOrganizations/org1.example.com/ca/:/etc/hyperledger/fabric-ca-server-config
      container_name: ca.org1.example.com
      networks:
      - behave

```

Add couch db for peer 1 add following code above peer 0 in docker-composer-kafa.yaml file


```javascript
    couchdb01:
        extends:
            file: docker-compose-base.yml
            service: couchdb
        container_name: couchdb01
        # Comment/Uncomment the port mapping if you want to hide/expose the CouchDB service,
        # for example map it to utilize Fauxton User Interface in dev environments.
        ports:
        - "5984:5984"
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
```

In peer 0 orgniztion 1 add following lines under environment tag

```javascript
            - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
            - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb01:5984
```

now add follwing configration under depends-on tag of peer0 org1 

```javascript
            - couchdb01
```
finally peer 0 org1 will look like this

```javascript
    peer0.org1.example.com:
        extends:
            file: docker-compose-base.yml
            service: peer
        container_name: peer0.org1.example.com
        environment:
            - CORE_PEER_CHAINCODELISTENADDRESS=peer0.org1.example.com:7052
            - CORE_PEER_ID=peer0.org1.example.com
            - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
            - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:7051
            - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
            - CORE_PEER_GOSSIP_ORGLEADER=${CORE_PEER_GOSSIP_ORGLEADER_PEER0_ORG1}
            - CORE_PEER_GOSSIP_USELEADERELECTION=${CORE_PEER_GOSSIP_USELEADERELECTION_PEER0_ORG1}
            - CORE_PEER_LOCALMSPID=Org1MSP
            - CORE_PEER_TLS_CLIENTROOTCAS_FILES=/var/hyperledger/users/Admin@org1.example.com/tls/ca.crt
            - CORE_PEER_TLS_CLIENTCERT_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.crt
            - CORE_PEER_TLS_CLIENTKEY_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.key
            - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
            - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb01:5984
        volumes:
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/var/hyperledger/msp
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/var/hyperledger/tls
            - ../crypto-config/peerOrganizations/org1.example.com/users:/var/hyperledger/users
            - ../config/:/var/hyperledger/configs

        depends_on:
            - orderer0.example.com
            - orderer1.example.com
            - orderer2.example.com
            - couchdb01
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
```


Do same for peer1 org 1 finally couch db and peer 0 org1 will look like this. There will be
few changes as we perform in peer 0 orginization 1

```javascript
   couchdb11:
        extends:
            file: docker-compose-base.yml
            service: couchdb
        container_name: couchdb11
        # Comment/Uncomment the port mapping if you want to hide/expose the CouchDB service,
        # for example map it to utilize Fauxton User Interface in dev environments.
        ports:
        - "7984:5984"
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
                
    peer1.org1.example.com:
        extends:
            file: docker-compose-base.yml
            service: peer
        container_name: peer1.org1.example.com
        environment:
            - CORE_PEER_CHAINCODELISTENADDRESS=peer1.org1.example.com:7052
            - CORE_PEER_ID=peer1.org1.example.com
            - CORE_PEER_ADDRESS=peer1.org1.example.com:7051
            - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
            - CORE_PEER_GOSSIP_ORGLEADER=${CORE_PEER_GOSSIP_ORGLEADER_PEER1_ORG1}
            - CORE_PEER_GOSSIP_USELEADERELECTION=${CORE_PEER_GOSSIP_USELEADERELECTION_PEER1_ORG1}
            - CORE_PEER_LOCALMSPID=Org1MSP
            - CORE_PEER_TLS_CLIENTROOTCAS_FILES=/var/hyperledger/users/Admin@org1.example.com/tls/ca.crt
            - CORE_PEER_TLS_CLIENTCERT_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.crt
            - CORE_PEER_TLS_CLIENTKEY_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.key
            - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
            - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb11:5984
        volumes:
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/var/hyperledger/msp
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/var/hyperledger/tls
            - ../crypto-config/peerOrganizations/org1.example.com/users:/var/hyperledger/users
            - ../config/:/var/hyperledger/configs
        depends_on:
            - orderer0.example.com
            - orderer1.example.com
            - orderer2.example.com
            - peer0.org1.example.com
            - couchdb11
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
        ports:
          - 8051:7051
          - 8053:7053
```

fianlly this docker-compose-kafka.yaml file will look like this. If you have made any mistake please refer to this file

```javascript
   couchdb11:
        extends:
            file: docker-compose-base.yml
            service: couchdb
        container_name: couchdb11
        # Comment/Uncomment the port mapping if you want to hide/expose the CouchDB service,
        # for example map it to utilize Fauxton User Interface in dev environments.
        ports:
        - "7984:5984"
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
                
    peer1.org1.example.com:
        extends:
            file: docker-compose-base.yml
            service: peer
        container_name: peer1.org1.example.com
        environment:
            - CORE_PEER_CHAINCODELISTENADDRESS=peer1.org1.example.com:7052
            - CORE_PEER_ID=peer1.org1.example.com
            - CORE_PEER_ADDRESS=peer1.org1.example.com:7051
            - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
            - CORE_PEER_GOSSIP_ORGLEADER=${CORE_PEER_GOSSIP_ORGLEADER_PEER1_ORG1}
            - CORE_PEER_GOSSIP_USELEADERELECTION=${CORE_PEER_GOSSIP_USELEADERELECTION_PEER1_ORG1}
            - CORE_PEER_LOCALMSPID=Org1MSP
            - CORE_PEER_TLS_CLIENTROOTCAS_FILES=/var/hyperledger/users/Admin@org1.example.com/tls/ca.crt
            - CORE_PEER_TLS_CLIENTCERT_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.crt
            - CORE_PEER_TLS_CLIENTKEY_FILE=/var/hyperledger/users/Admin@org1.example.com/tls/client.key
            - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
            - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb11:5984
        volumes:
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/var/hyperledger/msp
            - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/var/hyperledger/tls
            - ../crypto-config/peerOrganizations/org1.example.com/users:/var/hyperledger/users
            - ../config/:/var/hyperledger/configs
        depends_on:
            - orderer0.example.com
            - orderer1.example.com
            - orderer2.example.com
            - peer0.org1.example.com
            - couchdb11
        networks:
          behave:
             aliases:
               - ${CORE_PEER_NETWORKID}
        ports:
          - 8051:7051
          - 8053:7053
```

## genrate.sh
in genrate.sh file remove following lines

```javascript
# generate anchor peer transaction
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./config/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
if [ "$?" -ne 0 ]; then
  echo "Failed to generate anchor peer update for Org1MSP..."
  exit 1
fi

# generate anchor peer transaction
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./config/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
if [ "$?" -ne 0 ]; then
  echo "Failed to generate anchor peer update for Org2MSP..."
  exit 1
fi

```

genrate.sh updated file will look like this

```javascript

#!/bin/sh
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
export PATH=$GOPATH/src/github.com/hyperledger/fabric/build/bin:${PWD}/bin:${PWD}:$PATH
export FABRIC_CFG_PATH=${PWD}/network-config
CHANNEL_NAME=mychannel

# create folders
mkdir -p config
mkdir -p crypto-config

# remove previous crypto material and config transactions
rm -fr ./config/*
rm -fr ./crypto-config/*


# generate crypto material
cryptogen generate --config=./network-config/crypto-config.yaml
if [ "$?" -ne 0 ]; then
  echo "Failed to generate crypto material..."
  exit 1
fi

# generate genesis block for orderer
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./config/orderer.block
if [ "$?" -ne 0 ]; then
  echo "Failed to generate orderer genesis block..."
  exit 1
fi

# generate channel configuration transaction
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./config/channel.tx -channelID $CHANNEL_NAME
if [ "$?" -ne 0 ]; then
  echo "Failed to generate channel configuration transaction..."
  exit 1
fi

```

## start.sh

in start.sh add two peers and join them with channel
the updated start.sh file will look like this

```javascript

#!/bin/bash
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
# Exit on first error, print all commands.
set -ev

# don't rewrite paths for Windows Git Bash users
export MSYS_NO_PATHCONV=1

docker-compose -f ./network-config/docker-compose-kafka.yml down
docker-compose -f ./network-config/docker-compose-cli.yml down


docker-compose -f ./network-config/docker-compose-kafka.yml up -d
docker-compose -f ./network-config/docker-compose-cli.yml up -d

#docker stack deploy --compose-file ./network-config/docker-compose-kafka.yml 
#docker stack deploy --compose-file ./network-config/docker-compose-cli.yml


# wait for Hyperledger Fabric to start
# incase of errors when running later commands, issue export FABRIC_START_TIMEOUT=<larger number>
export FABRIC_START_TIMEOUT=30
echo ${FABRIC_START_TIMEOUT}
sleep ${FABRIC_START_TIMEOUT}

# # # Create the channel
docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel create -o orderer0.example.com:7050 -c mychannel -f /var/hyperledger/configs/channel.tx
# # # Join peer0.org1.example.com to the channel.
docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel join -b mychannel.block

# Fetch Channel details
docker exec  -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer1.org1.example.com peer channel fetch config -o orderer0.example.com:7050 -c mychannel

# Join peer1.org1.example.com to the channel.
docker exec  -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer1.org1.example.com peer channel join -b mychannel_config.block


```

## Genrating artifacts

now Kafka-Fabric-Network directory run the following command

```javascript
./genrate.sh
```

it will produce the following output
![](/images/genrate.png)


## Ca key 

Go inside the following directory and find ca updated key 
```
crypto-config/peerOrganizations/org1.example.com/ca
```
My key look like this
![](/images/cakey.png)

copy this key ending with _sk and replace with at the end of following line in docker-compose-cli.yaml

```javascirpt
       - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/[newkey]
```

the updated Ca look like this with new key

```javascript
  ca.org1.example.com:
     image: hyperledger/fabric-ca:x86_64-1.1.0
     environment:
       - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
       - FABRIC_CA_SERVER_CA_NAME=ca.org1.example.com
       - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem
       - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/4a309917302acbbea311081e5d91671e4a63ac82fb85ff6801a6adb3ee97b2ee_sk
       - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=behave #${CORE_PEER_NETWORKID}_
     ports:
       - "7054:7054"
     #command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/52dc508d562d25d8d7078f1c3fa7aaa44b82b7728c123cc50394655b5a580c75_sk -b admin:adminpw -d'
     command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
     volumes:
       - ../crypto-config/peerOrganizations/org1.example.com/ca/:/etc/hyperledger/fabric-ca-server-config
     container_name: ca.org1.example.com
     networks:
       - behave

```

## createPeerAdminCard.sh

create a new file in home directory of this project with following name createPeerAdminCard.sh

and add follwoing lines of code in it. It contains information about structure of our orginzaiton. If you have more peer or any new resource you have to make changes in this file in this  part cat << EOF > DevServer_connection.json {}


createPeerAdminCard.sh file will look like this

```javascript
#!/bin/bash

# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
# http://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

Usage() {
	echo ""
	echo "Usage: ./createPeerAdminCard.sh [-h host] [-n]"
	echo ""
	echo "Options:"
	echo -e "\t-h or --host:\t\t(Optional) name of the host to specify in the connection profile"
	echo -e "\t-n or --noimport:\t(Optional) don't import into card store"
	echo ""
	echo "Example: ./createPeerAdminCard.sh"
	echo ""
	exit 1
}

Parse_Arguments() {
	while [ $# -gt 0 ]; do
		case $1 in
			--help)
				HELPINFO=true
				;;
			--host | -h)
                shift
				HOST="$1"
				;;
            --noimport | -n)
				NOIMPORT=true
				;;
		esac
		shift
	done
}

HOST=localhost
R1=10.10.30.134
R2=10.10.30.134
Parse_Arguments $@

if [ "${HELPINFO}" == "true" ]; then
    Usage
fi

# Grab the current directory
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

if [ -z "${HL_COMPOSER_CLI}" ]; then
  HL_COMPOSER_CLI=$(which composer)
fi

echo
# check that the composer command exists at a version >v0.16
COMPOSER_VERSION=$("${HL_COMPOSER_CLI}" --version 2>/dev/null)
COMPOSER_RC=$?

if [ $COMPOSER_RC -eq 0 ]; then
    AWKRET=$(echo $COMPOSER_VERSION | awk -F. '{if ($2<19) print "1"; else print "0";}')
    if [ $AWKRET -eq 1 ]; then
        echo Cannot use $COMPOSER_VERSION version of composer with fabric 1.1, v0.19 or higher is required
        exit 1
    else
        echo Using composer-cli at $COMPOSER_VERSION
    fi
else
    echo 'No version of composer-cli has been detected, you need to install composer-cli at v0.19 or higher'
    exit 1
fi

cat << EOF > DevServer_connection.json
{
    "name": "hlfv1",
    "x-type": "hlfv1",
    "x-commitTimeout": 300,
    "version": "1.0.0",
    "client": {
        "organization": "Org1",
        "connection": {
            "timeout": {
                "peer": {
                    "endorser": "300",
                    "eventHub": "300",
                    "eventReg": "300"
                },
                "orderer": "300"
            }
        }
    },
    "channels": {
        "mychannel": {
            "orderers": [
                "orderer0.example.com",
                "orderer1.example.com",
                "orderer2.example.com"
            ],
            "peers": {
                "peer0.org1.example.com": {
                    "endorsingPeer": true,
                    "chaincodeQuery": true,
                    "eventSource": true
		},
		"peer1.org1.example.com": {
                    "endorsingPeer": true,
                    "chaincodeQuery": true,
                    "eventSource": true
		}
            }
        }
    },
    "organizations": {
        "Org1": {
            "mspid": "Org1MSP",
            "peers": [
                "peer0.org1.example.com",
		"peer1.org1.example.com"
            ],
            "certificateAuthorities": [
                "ca.org1.example.com"
            ]
        }

    },
    "orderers": {
        "orderer0.example.com": {
            "url": "grpc://${HOST}:7050"
        },
        "orderer1.example.com": {
            "url": "grpc://${R2}:8050"
        },
        "orderer2.example.com": {
            "url": "grpc://${R1}:9050"
        }
    },
    "peers": {
        "peer0.org1.example.com": {
            "url": "grpc://${HOST}:7051",
            "eventUrl": "grpc://${HOST}:7053"
        },
        "peer1.org1.example.com": {
            "url": "grpc://${HOST}:8051",
            "eventUrl": "grpc://${HOST}:8053"
        }

    },
    "certificateAuthorities": {
        "ca.org1.example.com": {
            "url": "http://${HOST}:7054",
            "caName": "ca.org1.example.com"
        }
    }
}
EOF

PRIVATE_KEY="${DIR}"/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/0cfb1b4e0c986120d13ec11ede610fac64155ad86385c20e08a38791086c3e2c_sk
CERT="${DIR}"/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem

if [ "${NOIMPORT}" != "true" ]; then
    CARDOUTPUT=/tmp/PeerAdmin@hlfv1.card
else
    CARDOUTPUT=PeerAdmin@hlfv1.card
fi

"${HL_COMPOSER_CLI}"  card create -p DevServer_connection.json -u PeerAdmin -c "${CERT}" -k "${PRIVATE_KEY}" -r PeerAdmin -r ChannelAdmin --file $CARDOUTPUT

if [ "${NOIMPORT}" != "true" ]; then
    if "${HL_COMPOSER_CLI}"  card list -c PeerAdmin@hlfv1 > /dev/null; then
        "${HL_COMPOSER_CLI}"  card delete -c PeerAdmin@hlfv1
    fi

    "${HL_COMPOSER_CLI}"  card import --file /tmp/PeerAdmin@hlfv1.card 
    "${HL_COMPOSER_CLI}"  card list
    echo "Hyperledger Composer PeerAdmin card has been imported, host of fabric specified as '${HOST}'"
    rm /tmp/PeerAdmin@hlfv1.card
else
    echo "Hyperledger Composer PeerAdmin card has been created, host of fabric specified as '${HOST}'"
fi


```

Now go inside the following directories and find keyfile 

```
crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/

```

my key look like this

```
49b6df69f057c0f656d5ce8ef09b0e47c850f5bd5f231e86a89c162e9f915e1c_sk

```
now replace this key with private_key variable 

the updated private_key will look like this

```javascript
PRIVATE_KEY="${DIR}"/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/49b6df69f057c0f656d5ce8ef09b0e47c850f5bd5f231e86a89c162e9f915e1c_sk
```

give sudo permission to createPeerAdmin.sh

```javascript
 sudo chmod 777 createPeerAdmin.sh
```
## starting network

exucute the following command in home directory of this project
```
./start.sh
```
if it show  Error: got unexpected status: SERVICE_UNAVAILABLE -- will not enqueue, consenter for this channel hasn't started yet

then gradully increase FABRICS-TIMEOUT to 40 in start.sh

check your docker logs of CA is any error like this 

``` 
Error: Failed to find private key for certificate in '/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem': Could not find matching private key for SKI: Failed getting key for SKI [[74 48 153 23 48 42 203 190 163 17 8 30 93 145 103 30 74 99 172 130 251 133 255 104 1 166 173 179 238 151 178 238]]: Key with SKI 4a309917302acbbea311081e5d91671e4a63ac82fb85ff6801a6adb3ee97b2ee not found in /etc/hyperledger/fabric-ca-server/msp/keystore


```
then repeform  Ca key step

if your network started successfully then it will show you output like this

![](succussfulstart.png)

## excuting createPeeradmin.sh

now excute createpeeradim file using following command

```
./createPeerAdmin.sh
```
if command not exuted give follwoing permission to file

```
chmod 777 createPeerAdmin.sh
chmdo +x  createPeerAdmin.sh

```

after successfully stating following output will be displayed
![](/images/succussfulcreatePeerAdmin.png)


## Installing composer bna file

if you have ready composer bna file you can now install on this network and start
and start your playgound and reset server it will make transations onit