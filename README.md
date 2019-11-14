# fabric_recovery_peer

■ 참조 사이트 : https://medium.com/@kctheservant/recovery-of-a-peer-node-in-hyperledger-fabric-248af99b7ec9

< Recovery of a Peer Node in Hyperledger Fabric >
- 피어 노드의 복구 프로세스 : 실행중인 피어 노드를 중단하고 완전히 작동하는 노드로 다시 시작
- 목적 : 채널, 원장, 체인 코드 등 Hyperledger Fabric의 다양한 부분이 어떻게 작동하는지 이해.

1. Test Setup
- 예제 : Fabcar 사용
$ cd fabric-samples/fabcar
$ ./startFabric.sh

1.2 Test Network Setup 구성 (그림참조)
- One orderer
- Two organizations (Org1 and Org2)
- 각 조직(organization)에는 인증기관과 2개의 피어 노드 (피어 0 및 피어 1로 지정), 총 4개의 피어노드 구성
- Each peer node :  a couchdb + the ledger
- A channel name : mychannel
- Fabcar chaincode install
- Fabcar chaincode is instantiated in mychannel
- initLedger() : 10 sets of car record in the ledger

1.3 Command : open peer nodes
# for peer0.org1.example.com
$ docker exec -it cli bash

# for peer1.org1.example.com
$ docker exec -it -e CORE_PEER_ADDRESS=peer1.org1.example.com:8051 -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt cli bash

# for peer0.org2.example.com
$ docker exec -it -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer0.org2.example.com:9051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt cli bash

# for peer1.org2.example.com
$ docker exec -it -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer1.org2.example.com:10051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt cli bash

1.3.1 To verify things are working well ( ALL peer 안에서 )
$ peer channel getinfo -c mychannel
- Result : 2019-11-13 04:35:11.491 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
		   Blockchain info: {"height":5,"currentBlockHash":"3S53LVwq80MSbTL8l1nkftjO5ZSk+DsNBTxKrAJXXZM=","previousBlockHash":"G8Z/aoHO+H91paNN1mogtdScYKSKdRKkf16Ny6jRex4="}
$ peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryCar","CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}

1.3.2 To Verify world state in each node. (couchdb)
- http://localhost:5984/_utils. (접속 URL)
- port 5984 : peer0.org1
- port 6984 : peer1.org1
- port 7984 : peer0.org2
- port 8984 : peer1.org2

2. Shutdown a Peer Node (그림참조)
- kill peer : peer0.org2.example.com 

$ docker kill peer0.org2.example.com
$ docker rm peer0.org2.example.com
$ docker volume rm net_peer0.org2.example.com
$ docker kill couchdb2
$ docker rm couchdb2
$ docker ps -a

2.1 peer0.org2.example.com 요청을 보내고 응답이 없는 상태를 확인
$ docker exec -it cli bash
$ peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n fabcar --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["changeCarOwner", "CAR0", "KC"]}'
- Result : Error: error getting endorser client for invoke: endorser client failed to connect to peer0.org2.example.com:9051:~~~

2.2 peer1.org2.example.com은 Org2 멤버로 존재하므로 확인해봄.
$ peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n fabcar --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer1.org2.example.com:10051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt -c '{"Args":["changeCarOwner", "CAR0", "KC"]}'
- Result : Chaincode invoke successful. result: status:200

$ peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryCar","CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"KC"}

3. Recover this Peer Node into Full Service
- 3단계를 거쳐 작동중인 노드에 피어로 복구함.

3.1 Bring Up the Peer Node(peer0.org2.example.com) and CouchDB (1단계)
- docker compose files을 이용하여 복구함.
$ cd first-network
$ docker-compose -f docker-compose-cli.yaml -f docker-compose-couch.yaml up -d couchdb2 peer0.org2.example.com
$ docker ps -a

- 다음 2단계에서 채널 연결을 위해 채널이 있는지 먼저 확인. (CLI 터미널에서)
$ peer channel list

- 현재 피어(peer0.org2.example.com)는 Docker에 구동되었으나. mychannel에 연결이 안된 상태임. (그림참조 : 현재 상태의 다이어그램)
- 피어(peer0.org2.example.com)에서 couchdb의 mychannel이 비어 있음을 확인함.
http://localhost:7984/_utils/

3.2 Join the Peer Node to mychannel (2단계)
- peer0.org2.example.com가 mychannel에 연결이되면 원장이 노드에 "동기화" 됩니다.
- 위 상황에 필요한 요소는 channel genesis block (mychannel.block) 입니다.
- peer0.org2.example.com cli terminal로 들어가야함.
# for peer0.org2.example.com (cli terminal)
$ docker exec -it -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer0.org2.example.com:9051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt cli bash

$ peer channel join -b mychannel.block
$ peer channel list

- 피어(peer0.org2.example.com)에서 couchdb의 mychannel이 있음을 확인함.
http://localhost:7984/_utils/

- 블록체인 정보를 살펴보면 전체 블록 체인이 이미있는 것을 볼 수 있음 (결과를 다른 피어 노드와 비교해보면 동일값, 즉 원장 및 DB가 동기화 되었음을 확인함)
$ peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":6,"currentBlockHash":"h56/JAW5BCV6MK0EeuyyyEkCK98f1+Ju0c85+TEXDoY=","previousBlockHash":"3S53LVwq80MSbTL8l1nkftjO5ZSk+DsNBTxKrAJXXZM="}

3.3 Install Chaincode
- 체인코드 함수를 쿼리하거나 호출하기 위해서는 인스턴스화되어야 합니다.
- peer0.org2.example.com cli terminal로 들어가야함.

$ peer chaincode list --installed
- Result : Get installed chaincodes on peer:
$ peer chaincode list --instantiated -C mychannel
- Result : Get instantiated chaincodes on channel mychannel: Name: fabcar, Version: 1.0, Escc: escc, Vscc: vscc

3.3.1 peer0.org2.example.com에 chaincode install
$ peer chaincode install -n fabcar -v 1.0 -p github.com/chaincode/fabcar/go
- Result : Installed remotely response:<status:200 payload:"OK" >

3.3.2 설치확인 명령어
$ peer chaincode list --installed
- Result : Get installed chaincodes on peer: Name: fabcar, Version: 1.0, Path: github.com/chaincode/fabcar/go, Id: 16ff6630c6cbc19e265191e304ce50264028fde12466b3e845478278da58b55f

3.3.3 query the chaincode
$ peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryCar","CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"KC"}

4. Back to Normal
- 터미널 CLI peer0.org1.example.com에서 아래 명령어를 수행
$ peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n fabcar --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["changeCarOwner", "CAR0", "John"]}'
- Result : Chaincode invoke successful. result: status:200

$ peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryCar","CAR0"]}'
- Result : {"colour":"blue","make":"Toyota","model":"Prius","owner":"John"}

- peer0.org2.example.com successfully perform endorsement for this proposal (수행됨을 확인할수 있음)

5. 요약
- 피어 노드가 채널에 가입하면 네트워크에서 전체 원장을 다시 가져와 블록 체인과 월드 상태가 동기화됨.
- 원장 내용이 동일하다고해서 노드가 체인 코드 호출 또는 쿼리를 처리 할 수있는 것은 아닙니다.
- 필요한 체인 코드를이 피어에 다시 설치해야 쿼리 및 호출이 가능함.

6. Clean Up
$ cd first-network
$ ./byfn.sh down
$ docker rm $(docker ps -aq)
$ docker rmi $(docker images dev-* -q)
$ docker network prune

