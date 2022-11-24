# Hyperledger-FL

## Demo
Part 1: https://youtu.be/qrbzBjPTwKI

Part 2: https://youtu.be/ts4XNC790Hg

Part 3: https://youtu.be/zHpEhKtbNe4

Part 4: https://youtu.be/S2MvVz8JI_E

## How To Run with Docker

## Prerequisites
Make sure `docker` and `docker compose` are successfully installed. 
```
docker --version 
# Docker version 20.10.21, build baeda1f

docker compose version 
# Docker Compose version v2.6.1
```
Since everything is based on docker, there shouldn't be a problem with running the project. However, the main OS for developing the project is `Ubuntu 22.04.1 LTS`, and scripts are written in Bash. Therefore, you might need to do some steps manually if your OS does not support Bash.
```
lsb_release -a

# Distributor ID:	Ubuntu
# Description:	Ubuntu 22.04.1 LTS
# Release:	22.04
# Codename:	jammy
```

We assume that you have a base directory in `/opt/hyperledger-fl`.

## Fabric (Chaincode and Network)
First we need to clone `dist-fed-chaincode` repository. This repository contains the required chaincodes and a test network for deploying chaincodes.
```
cd /opt/hyperledger-fl
git clone https://github.com/feknall/dist-fed-chaincode
cd dist-fed-chaincode
```
You only need to use `install.sh`, `start.sh`, and `stop.sh` scripts. 
If it is the first time that you are deploying the project, you should run `install.sh` script. This script downloads required binary files and docker images.
```
chmod +x install.sh
./install.sh 
```
Next, you can use `start.sh` and `stop.sh` for deploying and undeploying.
```
chmod +x start.sh
./start.sh
```
Here is the the output of `docker ps -a`:
```
5bd0a5c1233f   dist-fed-chaincode_ccaas_image:latest   "/tini -- /docker-en…"   6 seconds ago    Up 5 seconds    9999/tcp                                                                                                                          peer0org2_dist-fed-chaincode_ccaas
5db5f2a96113   dist-fed-chaincode_ccaas_image:latest   "/tini -- /docker-en…"   6 seconds ago    Up 5 seconds    9999/tcp                                                                                                                          peer0org1_dist-fed-chaincode_ccaas
ca98a0cc11e2   hyperledger/fabric-tools:latest         "/bin/bash"              45 seconds ago   Up 44 seconds                                                                                                                                     cli
adf971f2747a   hyperledger/fabric-orderer:latest       "orderer"                46 seconds ago   Up 45 seconds   0.0.0.0:7050->7050/tcp, :::7050->7050/tcp, 0.0.0.0:7053->7053/tcp, :::7053->7053/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp   orderer.example.com
f07048137dcb   hyperledger/fabric-peer:latest          "peer node start"        46 seconds ago   Up 45 seconds   0.0.0.0:7051->7051/tcp, :::7051->7051/tcp, 0.0.0.0:9444->9444/tcp, :::9444->9444/tcp                                              peer0.org1.example.com
5963f8d83364   hyperledger/fabric-peer:latest          "peer node start"        46 seconds ago   Up 45 seconds   0.0.0.0:9051->9051/tcp, :::9051->9051/tcp, 7051/tcp, 0.0.0.0:9445->9445/tcp, :::9445->9445/tcp                                    peer0.org2.example.com
7a9577cdb68c   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   55 seconds ago   Up 54 seconds   0.0.0.0:9054->9054/tcp, :::9054->9054/tcp, 7054/tcp, 0.0.0.0:19054->19054/tcp, :::19054->19054/tcp                                ca_orderer
6d4b00b50b14   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   55 seconds ago   Up 54 seconds   0.0.0.0:7054->7054/tcp, :::7054->7054/tcp, 0.0.0.0:17054->17054/tcp, :::17054->17054/tcp                                          ca_org1
4478eef46a41   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   55 seconds ago   Up 53 seconds   0.0.0.0:8054->8054/tcp, :::8054->8054/tcp, 7054/tcp, 0.0.0.0:18054->18054/tcp, :::18054->18054/tcp                                ca_org2
```

When you are done, you can use:
```
chmod +x stop.sh
./stop.sh
```
For the next times, you do not need to run `install.sh` file again. You can just use `start.sh` and `stop.sh`.
So far, a test network is running with deployed chaincode.


## Gateway
As `fabric-gateway` doest not have Python SDK, we have implemented a gateway using Java that receives REST calls from Python clients.

First of all, you must set this environment variable. You can set it in `~/.bashrc` file.
```
vim ~/.bashrc
```
Append this line at the end:
```
export DIST_FED_CREDENTIAL_HOME=[path-to-dist-fed-chaincode]/test-network/organizations
```
Apply changes:
```
source ~/.bashrc
```

Now, we clone `dist-fed-gateway` repository:
```
cd /opt/hyperledger-fl
git clone https://github.com/feknall/dist-fed-gateway
cd dist-fed-gateway
```
And use docker to run it:
```
chmod +x start.sh
./start.sh
```
Now, the gateway is up and running. Here is the expected `docker ps -a` result:
```
ec186c38c9ab   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     trainer01.example.com
08b240d556f4   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     aggregator.org2.example.com
6e2983736295   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     trainer02.example.com
3326ecec85a6   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     admin.org2.example.com
707bd983be7d   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     aggregator.org1.example.com
636892c57a01   hmaid/hyperledger:dist-fed-gateway      "java -jar /gateway.…"   48 seconds ago   Up 47 seconds                                                                                                                                     leadAggregator.org2.example.com
5bd0a5c1233f   dist-fed-chaincode_ccaas_image:latest   "/tini -- /docker-en…"   37 minutes ago   Up 37 minutes   9999/tcp                                                                                                                          peer0org2_dist-fed-chaincode_ccaas
5db5f2a96113   dist-fed-chaincode_ccaas_image:latest   "/tini -- /docker-en…"   37 minutes ago   Up 37 minutes   9999/tcp                                                                                                                          peer0org1_dist-fed-chaincode_ccaas
ca98a0cc11e2   hyperledger/fabric-tools:latest         "/bin/bash"              37 minutes ago   Up 37 minutes                                                                                                                                     cli
adf971f2747a   hyperledger/fabric-orderer:latest       "orderer"                37 minutes ago   Up 37 minutes   0.0.0.0:7050->7050/tcp, :::7050->7050/tcp, 0.0.0.0:7053->7053/tcp, :::7053->7053/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp   orderer.example.com
f07048137dcb   hyperledger/fabric-peer:latest          "peer node start"        37 minutes ago   Up 37 minutes   0.0.0.0:7051->7051/tcp, :::7051->7051/tcp, 0.0.0.0:9444->9444/tcp, :::9444->9444/tcp                                              peer0.org1.example.com
5963f8d83364   hyperledger/fabric-peer:latest          "peer node start"        37 minutes ago   Up 37 minutes   0.0.0.0:9051->9051/tcp, :::9051->9051/tcp, 7051/tcp, 0.0.0.0:9445->9445/tcp, :::9445->9445/tcp                                    peer0.org2.example.com
7a9577cdb68c   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   37 minutes ago   Up 37 minutes   0.0.0.0:9054->9054/tcp, :::9054->9054/tcp, 7054/tcp, 0.0.0.0:19054->19054/tcp, :::19054->19054/tcp                                ca_orderer
6d4b00b50b14   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   37 minutes ago   Up 37 minutes   0.0.0.0:7054->7054/tcp, :::7054->7054/tcp, 0.0.0.0:17054->17054/tcp, :::17054->17054/tcp                                          ca_org1
4478eef46a41   hyperledger/fabric-ca:latest            "sh -c 'fabric-ca-se…"   37 minutes ago   Up 37 minutes   0.0.0.0:8054->8054/tcp, :::8054->8054/tcp, 7054/tcp, 0.0.0.0:18054->18054/tcp, :::18054->18054/tcp                                ca_org2
```

## FedBlockchain (Federated Learning, Indy, Von Network)
### Federated Learning
By finishing previous steps, clients are ready to start. 
```
cd /opt/hyperledger-fl
git clone https://github.com/feknall/dist-fed-core-fl.git
cd dist-fed-core-fl
```
For starting the training process, run:
```
./start.sh
```
Then use
```
docker compose -f docker-compose-fl-trainer.yml logs trainer1
```
And make sure the response of checking in is 200. Then run:
```
docker compose up flAdmin
```
It trains a model for a couple of rounds and prints the accuracy of the trained model at the end.


### Identity
For starting the identity process, run:
```
cd /opt/hyperledger-fl
git clone https://github.com/feknall/dist-fed-core-identity.git
cd dist-fed-core-identity
```
There is 3 different scripts:
```
chmod +x start-client.sh
chmod +x start-verifier.sh
chmod +x start-issuer.sh
```
Then execute each line of this code in a separate terminal:
```
./start-client.sh
./start-verifier.sh
./start-issuer.sh
```

## Development Concerns
This project tries to integrate Hyperledger Fabric, Aries Agents, and Indy. The task is training a federated learning model. In order to acheive that
many different architectures can be used. We have selected an architecture according to the following requirements of the project:

### Amount of Customization
We all know that Indy and Aries are working together very well, and the support of Indy from Aries side is really strong. However, Fabric and Indy are completely two different ledgers that are build for different purposes. To the best of our knowledge, there are just a limited number works that try to integrate Indy and Fabric. Moreover, non of those works have considered a task like Federated Learning which has high communication and computation cost. Our approach is to use these components with least amount of customization. Therefore, we try our best to do not change source code of Hyperledger Aries Agents, Indy, and Fabric.

### Scalability
One of the main challenges of federated Learning is that it is hard to scale (e.g. 1 Million Clients) even by having central server. So, we must make sure to do not make this problem worse by introducing two different ledgers to our setup and removing the central server.

### Extensibility
Federated learning is an active research area that researchers try to find some solutions to address different kinds of concenrs. For example, some algorithms are completely focused on secure aggregation to prevent information leakage from client updates. Some other solutions try to find a way to adderss malicious activities by clients that have negative effect on the accrucay of the final model. In this work, we want to follow an architecture that is extensible to most of these recent works. Therefore, researchers can use the similar algorithm that they studied in a central server with a central identity with least amount of modification.

## Components
For this project, we are going to use many different tools and libraries. This section describes a few of them and their rule in our setup:

### Indy: Von Network
The main reason for using Indy in this project is to have a fully distributed solution that even the authentication of the users is not based on a central server. We are going to use Indy for authentication of federated learning clients (e.g. a mobile device or a personal computer). Von Network project offers and easy and ready-to-use for initating an Indy network. Moreover, it provides a web interface which is useful for tracing and debugging. We have decided to use this network by having very minor customizations. For example, containers of Von Network use their own network for communication with each other. But we decided to change it to use bridge networking, which is easier for development. Please note that all of our customizations can be reverted as the final project is ready. [Here](https://github.com/feknall/von-network) is the link to the Von Network that we use, which is a fork of the original repository. 

### Aries Agents: ACA-Py
Hyperledger Aries offers many different agent implementation. We have selected ACA-PY as it satisfy all of our requirements, has extensive amount of documentation, and is under active development. We will use ACA-PY to for creating schemas, credentials, issuing, and verification. For our requirement, each client needs an Aries agent for authentication and we need only one verifier and one issuer. The details of how the verifier is going to be used in our project will be discussed in the next section. Similar to our requirement for having some customization in Von Netowk, we have to do it for ACA-PY agents as well, which can be found in [here](https://github.com/feknall/aries-cloudagent-python).

### Fabric
In the federated learning setup with a central server, the central server is responsible for aggregation and computing the average model which is based on the responses of clients. We are going to remove the central server and replace it with a distributed ledger. We will implement some chaincode to complete this process. However, we have a problem in this part. The problem is that we cannot fully use chaincode and we need to have some process offchain, not onchain. The main reason is that it is not possible to use Python for chaincode as Fabric doesn't support it. Anyway, this is not a deal breaker. 

More specificly, we will use fabric for these tasks:
1. Selecting clients for staring the round N of training
2. Selecting committee members for evaluating client updates to prevent malicious activities
3. Calculating average of the model based on the output of committee members
4. Reading and Writing the updated model from from/to the ledger

### Fabric CA
Fabric provides an MSP which will be used for authentication/authroziation of the nodes in the network, like peers and orderes, or even clients that call chaincode. MSP is based on certificate. Therefore, if a node wants to be authenticated, it needs to have a certificate. In our case, all of our clients need to have certificate for communication with Fabric. There are different ways for generating these certificates, including using OpenSSL, Cryptogen, and Fabric-CA. Among these options, we have selected fabric-ca as it is ready-to-use and satisfies all of our requirments. The role of Fabric CA is generating some certificates for clients. Therefore, clients can send those certificates when they want to communicate with Fabric.

## Integration of Components
We will discuss the approach that we are going to implement for integrating all these components together. To do this, first we need to discuss type of entities that our solution needs. 

+ Fabric: We consider Fabric as a black-box. Therefore, whenever we say Fabric, we mean the whole network, including Peers, Orderes, and the ledger.
+ Fabric CA: Consider it as single node in the network (not Indy or Fabric Network) that can create certificates which Fabric accepts.
+ Indy: Similar to Fabric, we consider it black-box.
+ Aries Agents: Aries agents are some process that each node can run (They are not a node in the network). 
+ Client: Consider these as data owners who want to participate and train a federated learning model. Non of the other components have any kind of data that can be used for training. Therefore, only clients have data. 

Now, we discuss how we are going to integrate these components.

+ We need Fabric and Indy for our solution. It is not important for us that each of these networks how many nodes (peers, orderer, ...) have. 
+ We need at least one Fabric CA node, but our solution is extensible to have more than one Fabric CA. We will have one Aries agent in the same node that Fabric CA is running.
+ Number of clients is variable and it can be from less than 10 clients to more than thousands or even milions. For each client, we are going to consider a Aries Cloud Agent. For example, if we have 5 clients, we will have 5 aries cloud agent that will be run in the same node since they are just a process and don't need to be a separate node. Therefore, 5 clients means 5 nodes in the network.
+ We need a few (1, 2, etc.) issuers (Aries Cloud agents). Clients communicate with these issuers to receive a verifiable credential. Next, they send this credential to the Fabric CA's agent. Fabric CA's agent verifies these credential, and generates a certificate in response.

## How to generate a certificate that Fabric understands?
Fabric uses MSP that is based on certificate. Therefore, each client needs a certificate for communication with Fabric. There are two different approaches to do it, we discuss details of each approach. Next, we describe why one of them makes more sense that the other one.

### Approach 1: Fabric CA as Issuer
General Idea: Consider that the client doesn't have any kind of certificate. Therefore, it should get one. It can communicate with Fabric CA and send some of his information, like firstname, lastname, and an email address. Fabric CA uses these information to make sure this user is an honest user (e.g. by sending a verification email to him). Next, generates a certificate and send it back to the client. Now, the client has a certificate and can communicate with Fabric.

+ In this approach, Fabic CA has two roles. The first role is authenticating the user, and the second role is being a certificate authoritiy. There is no problem with being a certificate authority. But the question is that should a certificate authority be responsible for authenticating users when we can take advantages of verifiable credential? My answer is NO.
+ Moreover, this scenario doesn't need Indy at all. Indy is required for cases that an issuer want to issue a verifiable credeintiale and a verifier wants to verify it. But this scenario doesn't need a separate issuer. Actually, issuer and verifier are the same nodes. It doesn't make sense!
+ Even more, this approach uses Aries just as a socket that provides secure communication which can be replaced by many other tools.

### Approach 2: Fabric CA as Verifier
General Idea: Consider that the client doesn't have any kind of certificate. Therefore, it should get one. It can communicate with Fabric CA and send his verifiable credential (If it already doesn't have one, it should ask an issuer to issue a credential for it). Fabric CA receives the verifiable credential and verifies it. After successful verification, it generates a certificate and send it back to the client. Now, the client has a certificate and can communicate with Fabric.

+ In this approach Fabric CA is not responsible for authentication of user (as it shouldn't be). 
+ This approach needs at least another node that is an issuer. 
+ The client can use its credential that is already issued or ask for a new credential. This is exactly the main motivation of verifiable credentails.
+ This approach involves Indy.

Choosing between these two approaches is not hard at all. The first one only involves Fabric (We can add Aries as well, but it is ridicioulous). While the second approach takes all the benefits of verifiable credential and needs all capabilities of Aries and Indy. Therefore, we are going to use the second approach. 

## Why we have choosen to keep Fabric CA? We could remove it and do the authentication with direct communication between Fabric and Indy.
Will be discussed in the future...

## How to Issue a Credential by an Issuer for a Client?
There is a complete documentation for this process. Check here: [Link](https://github.com/feknall/Hyperledger-FL/blob/78c7eeb63456739e933161f1a6775074f90c3a9d/How-To-Issue-A-Credential.md)


