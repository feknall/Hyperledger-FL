# Hyperledger-FL


## Development Concerns
This project tries to integrate Hyperledger Fabric, Aries Agents, and Indy. The task is training a federated learning model. In order to acheive that
many different architectures can be used. We have selected an architecture according to the following requirements of the project:

### Amount of Customization
We all know that Indy and Aries are working together very well, and the support of Indy from Aries side is really strong. However, Fabric and Indy are completely two different ledgers that are build for different purposes. To the best of our knowledge, there are just a limited number works that try to integrate Indy and Fabric. Moreover, non of those works have considered a task like Federated Learning which has high communication and computation cost. Our approach is to use these components with least amount of customization. Therefore, we try our best to do not change source code of Hyperledger Aries Agents, Indy, and Fabric.

### Scalability
One of the main challenges of federated Learning is that it is hard to scale (e.g. 1 Million Clients) even by having central server. So, we must make sure to do not make this problem worse by introducing two different ledgers to our setup and removing the central server.


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


