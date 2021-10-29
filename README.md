# Deploying Hyperledger Fabric on Kubernetes 2.3+ with a Kubernetes Operator

## Pre requisites

- Kubernetes cluster 1.15+
- Kubectl HLF plugin
- Helm

## Nice to haves
- Lens (https://github.com/lensapp/lens)

# Installing the Kubectl HLF Plugin#
```bash
kubectl krew install hlf 
```


## Provisioning a cluster
If you don't have an existing cluster, you can provision one with [KiND](https://github.com/kubernetes-sigs/kind).):

```bash
kind create cluster --image=kindest/node:v1.22.2
```


Configure kubectl:
```bash
kind get kubeconfig > ~/.kube/hlf-kind
export KUBECONFIG=~/.kube/hlf-kind
gedit ~/.kube/hlf-kind
```
Check:
```bash
kubectl get nodes
kubectl get pods
```

## Installing the HLF operator

Add helm repository, source code here: https://github.com/kfsoftware/hlf-helm-charts 
```bash
helm repo add kfs https://kfsoftware.github.io/hlf-helm-charts --force-update 
```

Install chart
```bash
helm install hlf-operator -f hlf-operator.yaml kfs/hlf-operator
```
In case you change the values, you can upgrade:
```bash
helm upgrade hlf-operator -f ./hlf-operator.yaml kfs/hlf-operator
```

Checks

```bash
# check CRDs
kubectl get crds

# check pods until ready
kubectl get pods -w

# check logs if there's a problem
kubectl logs hlf-operator-controller-manager-75dbc94f58-rvnnl -c manager -f
```

## Blockchain network

Create initial folders:
```bash
mkdir -p resources/org1
mkdir -p resources/org2
mkdir -p resources/orderer/ordererorg1
```



## Peer organization

Generate CA manifest:
```
kubectl hlf ca create --storage-class=standard --capacity=2Gi --name=org1-ca \
    --enroll-id=enroll --enroll-pw=enrollpw  --output > resources/org1/ca.yaml
```
Create CA
```bash
kubectl apply -f ./resources/org1/ca.yaml
```

Checks

```bash
kubectl get fabriccas.hlf.kungfusoftware.es -A
# check pods until ready
kubectl get pods -w
```

Register user for the peers

```bash
kubectl hlf ca register --name=org1-ca --user=peer \
  --secret=peerpw --type=peer \
 --enroll-id enroll --enroll-secret=enrollpw \
 --mspid Org1MSP
```

```bash
kubectl hlf peer create --storage-class=standard \
    --enroll-id=peer --mspid=Org1MSP \
    --enroll-pw=peerpw --capacity=5Gi \
    --name=org1-peer0 --ca-name=org1-ca.default \
    --output > resources/org1/peer1.yaml
```

Create Peer
```bash
kubectl apply -f ./resources/org1/peer1.yaml
```

Checks

```bash
kubectl get fabricpeers.hlf.kungfusoftware.es -A
# check pods until ready
kubectl get pods -w
```

## Prepare network config for org1

Register user
```bash
kubectl hlf ca register --name=org1-ca --user=admin \
 --secret=adminpw --type=admin \
 --enroll-id enroll --enroll-secret=enrollpw \
 --mspid Org1MSP  
```
Enroll user
```bash
kubectl hlf ca enroll --name=org1-ca \
   --user=admin --secret=adminpw --mspid Org1MSP \
   --ca-name ca  --output peer-org1.yaml
```
Get connection config yaml
```bash
kubectl hlf inspect --output org1.yaml -o Org1MSP
```
Add user key and cert to org1.yaml from peer-org1.yaml
```bash
kubectl hlf utils adduser --userPath=peer-org1.yaml \
    --config=org1.yaml --username=admin --mspid=Org1MSP
```

Install chaincode
```bash
kubectl hlf chaincode install --path=./chaincodes/fabcar/go \
    --config=org1.yaml --language=golang --label=fabcar --user=admin --peer=org1-peer0.default
```

Check pods:
```bash
kubectl get pods -w
```

## Orderer organization

Create orderer org folder
```bash
mkdir -p resources/orderer/ordererorg1 
```
Generate CA manifest
```bash
kubectl hlf ca create --storage-class=standard \
   --capacity=2Gi --name=ordererorg1-ca \
   --enroll-id=enroll --enroll-pw=enrollpw  \
   --output > resources/orderer/ordererorg1/ca.yaml
```

Create CA
```bash
kubectl apply -f ./resources/orderer/ordererorg1/ca.yaml
```

Register user for the orderer
```bash
kubectl hlf ca register --name=ordererorg1-ca \
   --user=orderer --secret=ordererpw \
   --type=orderer --enroll-id enroll \
   --enroll-secret=enrollpw --mspid=OrdererMSP
```

Generate Orderer manifest
```bash
kubectl hlf ordnode create  --storage-class=standard \
   --enroll-id=orderer --mspid=OrdererMSP \
   --enroll-pw=ordererpw --capacity=2Gi \
   --name=ordnode-1 --ca-name=ordererorg1-ca.default \
   --output > resources/orderer/ordererorg1/orderer.yaml
```


Create Ordering Service
```bash
kubectl apply -f ./resources/orderer/ordererorg1/orderer.yaml
```

Check pods:
```bash
kubectl get pods -w
```

## Create a channel
Create connection config yaml
```bash
kubectl hlf inspect --output ordservice.yaml \
 -o OrdererMSP
```
Register user
```bash
kubectl hlf ca register --name=ordererorg1-ca \
   --user=admin --secret=adminpw \
   --type=admin --enroll-id enroll \
   --enroll-secret=enrollpw --mspid=OrdererMSP
```
Enroll user to submit the transaction
```bash
kubectl hlf ca enroll --name=ordererorg1-ca \
  --user=admin --secret=adminpw --mspid OrdererMSP \
     --ca-name ca  --output admin-ordservice.yaml
``` 
Add user from admin-ordservice.yaml to ordservice.yaml
```bash
kubectl hlf utils adduser --userPath=admin-ordservice.yaml --config=ordservice.yaml --username=admin --mspid=OrdererMSP
```
Generate channel block
```bash
kubectl hlf channel generate \
  --output=demo.block --name=demo \
  --organizations Org1MSP \
  --ordererOrganizations OrdererMSP

```

Visualize channel:
```bash
configtxlator proto_decode --input demo.block \
  --type common.Block --output block.json
```

Enroll user to submit the transaction

```bash
# enroll using the TLS CA
kubectl hlf ca enroll --name=ordererorg1-ca \
   --namespace=default --user=admin \
   --secret=adminpw --mspid OrdererMSP \
   --ca-name tlsca  \
   --output admin-tls-ordservice.yaml 
```

Join the orderer node with the channel block
```bash
kubectl hlf ordnode join --block=demo.block \
  --name=ordnode-1 --namespace=default \
  --identity=admin-tls-ordservice.yaml
```


## Join peer to channel

Register user
```bash
kubectl hlf ca register --name=org1-ca \
  --user=admin --secret=adminpw --type=admin \
 --enroll-id enroll --enroll-secret=enrollpw \
  --mspid Org1MSP  
```
Enroll user
```bash
kubectl hlf ca enroll --name=org1-ca --user=admin --secret=adminpw --mspid Org1MSP \
        --ca-name ca  --output peer-org1.yaml
```
Get connection config yaml
```bash
kubectl hlf inspect --output org1.yaml -o Org1MSP -o OrdererMSP
```
Add user key and cert to org1.yaml from peer-org1.yaml
```bash
kubectl hlf utils adduser --userPath=peer-org1.yaml --config=org1.yaml --username=admin --mspid=Org1MSP
```

Join peer to channel
```bash
kubectl hlf channel join --name=demo \
   --config=org1.yaml \
    --user=admin -p=org1-peer0.default
```

Add anchor peer for Org1MSP
```bash
kubectl hlf channel addanchorpeer \
   --channel=demo --config=org1.yaml \
   --user=admin --peer=org1-peer0.default
```

Inspect peer heights
```bash
kubectl hlf channel top --channel=demo \
   --config=org1.yaml \
   --user=admin -p=org1-peer0.default
```

Install chaincode
```bash
kubectl hlf chaincode install --path=./chaincodes/fabcar/go \
    --config=org1.yaml --language=golang --label=fabcar --user=admin --peer=org1-peer0.default

# this can take 3-4 minutes
```

Query approved chaincodes:
```bash
kubectl hlf chaincode queryinstalled --config=org1.yaml --user=admin --peer=org1-peer0.default
```

Approve chaincode
```bash
PACKAGE_ID=fabcar:0c616be7eebace4b3c2aa0890944875f695653dbf80bef7d95f3eed6667b5f40 # replace it with the package id of your chaincode
kubectl hlf chaincode approveformyorg --config=org1.yaml --user=admin --peer=org1-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "1.0" --sequence 1 --name=fabcar \
    --policy="OR('Org1MSP.member')" --channel=demo
```

Commit chaincode
```bash
kubectl hlf chaincode commit --config=org1.yaml --mspid=Org1MSP --user=admin \
    --version "1.0" --sequence 1 --name=fabcar \
    --policy="OR('Org1MSP.member')" --channel=demo
```

Test chaincode
```bash
kubectl hlf chaincode invoke --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=fabcar --channel=demo \
    --fcn=initLedger -a '[]'

```

Query all cars:
```bash
kubectl hlf chaincode query --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=fabcar --channel=demo \
    --fcn=QueryAllCars -a '[]'
```


## Update the channel

Get original channel config
```bash
kubectl hlf channel inspect --channel=demo --config=org1.yaml \
   --user=admin -p=org1-peer0.default > demo_original.json
```

Modify the channel
```bash
export CH_NAME=demo
configtxlator proto_encode --input demo_original.json --type common.Config --output config.pb

configtxlator proto_encode --input demo_update.json --type common.Config --output modified_config.pb

configtxlator compute_update --channel_id $CH_NAME --original config.pb --updated modified_config.pb --output config_update.pb


configtxlator proto_decode --input config_update.pb --type common.ConfigUpdate --output config_update.json


echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CH_NAME'", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' | jq . > config_update_in_envelope.json

configtxlator proto_encode --input config_update_in_envelope.json --type common.Envelope --output config_update_in_envelope.pb

# kubectl hlf channel signupdate --channel=demo -f config_update_in_envelope.pb --user=admin --config=org1.yaml --mspid=Org1MSP --output org1-demo-update-sign.pb
# kubectl hlf channel signupdate --channel=demo -f config_update_in_envelope.pb --user=admin --config=org2.yaml --mspid=Org2MSP --output org2-demo-update-sign.pb

kubectl hlf channel update --channel=demo -f config_update_in_envelope.pb \
   --config=org1.yaml --user=admin --mspid=Org1MSP

#  -s org1-demo-update-sign.pb -s org2-demo-update-sign.pb

```



## Create second organizations

Prepare folder:
```bash
mkdir -p resources/org2
```

Generate CA manifest:
```
kubectl hlf ca create --storage-class=standard --capacity=2Gi --name=org2-ca \
    --enroll-id=enroll --enroll-pw=enrollpw  --output > resources/org2/ca.yaml
```

Create CA
```bash
kubectl apply -f ./resources/org2/ca.yaml
```

Register user for the peers

```bash
kubectl hlf ca register --name=org2-ca --user=peer --secret=peerpw --type=peer \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org2MSP
```

Generate Peer manifests:
```bash
kubectl hlf peer create --storage-class=standard --enroll-id=peer --mspid=Org2MSP \
       --enroll-pw=peerpw --capacity=5Gi --name=org2-peer0 --ca-name=org2-ca.default --output > resources/org2/peer1.yaml
```

Create Peer
```bash
kubectl apply -f ./resources/org2/peer1.yaml
```

## Add second organization to the channel

Get JSON config for organization
```bash
kubectl hlf org inspect -o Org2MSP --output-path=crypto-config
```

Add organization
```bash
kubectl hlf channel addorg --peer=org1-peer0.default --name=demo \
    --config=org1.yaml --user=admin --msp-id=Org2MSP --org-config=./configtx.yaml
``` 

Check channel config

```bash
kubectl hlf channel inspect --channel=demo --config=org1.yaml \
   --user=admin -p=org1-peer0.default > demo.json
```


## Join peer of second organization to channel

Register user
```bash
kubectl hlf ca register --name=org2-ca --user=admin --secret=adminpw --type=admin \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org2MSP  
```
Enroll user
```bash
kubectl hlf ca enroll --name=org2-ca --user=admin --secret=adminpw --mspid Org2MSP \
        --ca-name ca  --output peer-org2.yaml
```
Get connection config yaml
```bash
kubectl hlf inspect --output org2.yaml  -o Org1MSP -o Org2MSP -o OrdererMSP
```
Add user key and cert to org2.yaml from peer-org2.yaml
```bash
kubectl hlf utils adduser --userPath=peer-org2.yaml --config=org2.yaml --username=admin --mspid=Org2MSP
```

Join peer to channel
```bash
kubectl hlf channel join --name=demo --config=org2.yaml \
    --user=admin -p=org2-peer0.default
```

## Approve & Commit chaincode again


Install chaincode
```bash
kubectl hlf chaincode install --path=./chaincodes/fabcar/go \
    --config=org2.yaml --language=golang --label=fabcar --user=admin --peer=org2-peer0.default

# this can take 3-4 minutes
```

Query approved chaincodes:
```bash
kubectl hlf chaincode queryinstalled --config=org2.yaml --user=admin --peer=org2-peer0.default
```

Approve chaincode
```bash
PACKAGE_ID=fabcar:0c616be7eebace4b3c2aa0890944875f695653dbf80bef7d95f3eed6667b5f40 # replace it with the package id of your chaincode
kubectl hlf chaincode approveformyorg --config=org1.yaml --user=admin --peer=org1-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "1.0" --sequence 2 --name=fabcar \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo

PACKAGE_ID=fabcar:0c616be7eebace4b3c2aa0890944875f695653dbf80bef7d95f3eed6667b5f40 # replace it with the package id of your chaincode
kubectl hlf chaincode approveformyorg --config=org2.yaml --user=admin --peer=org2-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "1.0" --sequence 2 --name=fabcar \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo

```

Commit chaincode
```bash
kubectl hlf chaincode commit --config=org1.yaml --mspid=Org1MSP --user=admin \
    --version "1.0" --sequence 2 --name=fabcar \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo
```

Test chaincode
```bash
kubectl hlf chaincode invoke --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=fabcar --channel=demo \
    --fcn=initLedger -a '[]'

```

Query all cars:
```bash
kubectl hlf chaincode query --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=fabcar --channel=demo \
    --fcn=QueryAllCars -a '[]'
```


MISSING!!!!
- Approve/Commit with the new organization
- Test that it works
- Add more consenters to the channel using the channel participation API


## Add more orderers

Generate Orderer manifest for second orderer
```bash
kubectl hlf ordnode create  --storage-class=standard --enroll-id=orderer --mspid=OrdererMSP \
    --enroll-pw=ordererpw --capacity=2Gi --name=ordnode-2 --ca-name=ordererorg1-ca.default \
    --output > resources/orderer/ordererorg1/orderer2.yaml
```

Create orderer2
```bash
kubectl apply -f ./resources/orderer/ordererorg1/orderer2.yaml
```


Generate Orderer manifest for third orderer
```bash
kubectl hlf ordnode create  --storage-class=standard --enroll-id=orderer --mspid=OrdererMSP \
    --enroll-pw=ordererpw --capacity=2Gi --name=ordnode-3 --ca-name=ordererorg1-ca.default \
    --output > resources/orderer/ordererorg1/orderer3.yaml
```

Create orderer3
```bash
kubectl apply -f ./resources/orderer/ordererorg1/orderer3.yaml
```
Join orderer2 and orderer3
```bash
kubectl hlf ordnode join --block=demo.block --name=ordnode-2 --namespace=default --identity=admin-tls-ordservice.yaml
kubectl hlf ordnode join --block=demo.block --name=ordnode-3 --namespace=default --identity=admin-tls-ordservice.yaml
```

Fetch channel config
```bash
kubectl hlf channel inspect --channel=demo --config=org1.yaml \
   --user=admin -p=org1-peer0.default > demo.json
```
