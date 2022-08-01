# Hyperledger-FL


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


# Process of Issuign a Credential for a Client
Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/connections/create-invitation' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
}'
```
```
{
  "connection_id": "32527d07-9f6b-4072-bf03-d0d454e6c829",
  "invitation": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/invitation",
    "@id": "528cf5bc-94e0-436f-bcce-5d38a57182bf",
    "serviceEndpoint": "http://172.17.0.1:10000",
    "recipientKeys": [
      "7T8zKWopBnS5MbkrQNhn32sJaY8LDGxvfShvyDMr2x7u"
    ],
    "label": "Aries Cloud Agent"
  },
  "invitation_url": "http://172.17.0.1:10000?c_i=eyJAdHlwZSI6ICJkaWQ6c292OkJ6Q2JzTlloTXJqSGlxWkRUVUFTSGc7c3BlYy9jb25uZWN0aW9ucy8xLjAvaW52aXRhdGlvbiIsICJAaWQiOiAiNTI4Y2Y1YmMtOTRlMC00MzZmLWJjY2UtNWQzOGE1NzE4MmJmIiwgInNlcnZpY2VFbmRwb2ludCI6ICJodHRwOi8vMTcyLjE3LjAuMToxMDAwMCIsICJyZWNpcGllbnRLZXlzIjogWyI3VDh6S1dvcEJuUzVNYmtyUU5objMyc0phWThMREd4dmZTaHZ5RE1yMng3dSJdLCAibGFiZWwiOiAiQXJpZXMgQ2xvdWQgQWdlbnQifQ=="
}
```

Client:
```
curl -X 'POST' \
  'http://127.0.0.1:5001/connections/receive-invitation' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/invitation",
    "@id": "528cf5bc-94e0-436f-bcce-5d38a57182bf",
    "serviceEndpoint": "http://172.17.0.1:10000",
    "recipientKeys": [
      "7T8zKWopBnS5MbkrQNhn32sJaY8LDGxvfShvyDMr2x7u"
    ],
    "label": "Aries Cloud Agent"
  }'
  ```
  ```
  {
  "their_label": "Aries Cloud Agent",
  "routing_state": "none",
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "updated_at": "2022-08-01T15:02:05.983601Z",
  "invitation_key": "7T8zKWopBnS5MbkrQNhn32sJaY8LDGxvfShvyDMr2x7u",
  "accept": "manual",
  "rfc23_state": "invitation-received",
  "connection_protocol": "connections/1.0",
  "state": "invitation",
  "created_at": "2022-08-01T15:02:05.983601Z",
  "invitation_mode": "once",
  "their_role": "inviter",
  "invitation_msg_id": "528cf5bc-94e0-436f-bcce-5d38a57182bf"
}
```

Client:
```
curl -X 'POST' \
  'http://127.0.0.1:5001/connections/ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89/accept-invitation' \
  -H 'accept: application/json' \
  -d ''
```
```
{
  "their_label": "Aries Cloud Agent",
  "routing_state": "none",
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "updated_at": "2022-08-01T15:02:49.035972Z",
  "invitation_key": "7T8zKWopBnS5MbkrQNhn32sJaY8LDGxvfShvyDMr2x7u",
  "accept": "manual",
  "rfc23_state": "request-sent",
  "request_id": "1e541013-9923-407d-ac77-0945d21ee73d",
  "my_did": "3HF6eMb1YDtyWxpWhyWKq7",
  "connection_protocol": "connections/1.0",
  "state": "request",
  "created_at": "2022-08-01T15:02:05.983601Z",
  "invitation_mode": "once",
  "their_role": "inviter",
  "invitation_msg_id": "528cf5bc-94e0-436f-bcce-5d38a57182bf"
}
```

Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/connections/32527d07-9f6b-4072-bf03-d0d454e6c829/accept-request' \
  -H 'accept: application/json' \
  -d ''
```
```
{
  "connection_protocol": "connections/1.0",
  "their_label": "Aries Cloud Agent",
  "updated_at": "2022-08-01T15:05:43.848183Z",
  "their_role": "invitee",
  "invitation_mode": "once",
  "invitation_key": "7T8zKWopBnS5MbkrQNhn32sJaY8LDGxvfShvyDMr2x7u",
  "their_did": "3HF6eMb1YDtyWxpWhyWKq7",
  "accept": "manual",
  "created_at": "2022-08-01T15:00:33.451843Z",
  "connection_id": "32527d07-9f6b-4072-bf03-d0d454e6c829",
  "state": "response",
  "rfc23_state": "response-sent",
  "my_did": "FRCyS4d7SYc5e4Pfxi3xCv",
  "routing_state": "none"
}
```
Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/wallet/did/create' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "method": "sov",
  "options": {
    "key_type": "ed25519"
  }
}'
```
```
{
  "result": {
    "did": "CHvMhycdjVi1JRkKQ3y7wi",
    "verkey": "79xK26SHrVUQ1kDquR7vSRgrhmtvNW4GyLzwi2nNizUF",
    "posture": "wallet_only",
    "key_type": "ed25519",
    "method": "sov"
  }
}
```
Issuer (Store these values to the Von Network).
```
did: CHvMhycdjVi1JRkKQ3y7wi
verkey: 79xK26SHrVUQ1kDquR7vSRgrhmtvNW4GyLzwi2nNizUF
role: endoser
```
Issuer
```
curl -X 'POST' \
  'http://127.0.0.1:5000/wallet/did/public?did=CHvMhycdjVi1JRkKQ3y7wi' \
  -H 'accept: application/json' \
  -d ''
```
```
{
  "result": {
    "did": "CHvMhycdjVi1JRkKQ3y7wi",
    "verkey": "79xK26SHrVUQ1kDquR7vSRgrhmtvNW4GyLzwi2nNizUF",
    "posture": "posted",
    "key_type": "ed25519",
    "method": "sov"
  }
}
```
Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/schemas' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "attributes": [
    "score"
  ],
  "schema_name": "prefs",
  "schema_version": "1.0"
}'
```
```
{
  "sent": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "schema": {
      "ver": "1.0",
      "id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
      "name": "prefs",
      "version": "1.0",
      "attrNames": [
        "score"
      ],
      "seqNo": 64
    }
  },
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "schema": {
    "ver": "1.0",
    "id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "name": "prefs",
    "version": "1.0",
    "attrNames": [
      "score"
    ],
    "seqNo": 64
  }
}
```

Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/credential-definitions' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "support_revocation": false,
  "tag": "default"
}'
```
```
{
  "sent": {
    "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
}
```

Client:
```
curl -X 'POST' \
  'http://127.0.0.1:5001/issue-credential/send-proposal' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "auto_remove": true,
  "comment": "string",
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
  "credential_proposal": {
    "@type": "issue-credential/1.0/credential-preview",
    "attributes": [
      {
        "name": "score",
        "value": "20"
      }
    ]
  },
  "issuer_did": "CHvMhycdjVi1JRkKQ3y7wi",
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "schema_issuer_did": "CHvMhycdjVi1JRkKQ3y7wi",
  "schema_name": "preferences",
  "schema_version": "1.0",
  "trace": true
}'
```
```
{
  "credential_proposal_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/propose-credential",
    "@id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "schema_name": "preferences",
    "schema_version": "1.0",
    "issuer_did": "CHvMhycdjVi1JRkKQ3y7wi",
    "schema_issuer_did": "CHvMhycdjVi1JRkKQ3y7wi",
    "comment": "string",
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "credential_proposal": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "20"
        }
      ]
    },
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "auto_remove": true,
  "initiator": "self",
  "state": "proposal_sent",
  "thread_id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
  "trace": true,
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "role": "holder",
  "credential_exchange_id": "4006de6f-3e86-4939-950f-85f344f67a16",
  "updated_at": "2022-08-01T16:19:04.119401Z",
  "auto_issue": false,
  "created_at": "2022-08-01T16:19:04.119401Z"
}
```

Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/issue-credential/records/6b82ab6f-844b-4bf0-a497-d40daeb98768/send-offer' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "counter_proposal": {
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "credential_proposal": {
          "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
          "attributes": [
            {
              "name": "score",
              "value": "30"
            }
          ]
        }
  }
}'
```
```
{
  "auto_remove": true,
  "credential_offer": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "nonce": "1045186427984757539800352",
    "key_correctness_proof": {
      "c": "43726574014840674561302162687127011268615439821062199243999041200241958263734",
      "xz_cap": "548848708985710322369656549936190432946001113003738991225264231774883170308310075824886472038670169513976438130352338856595784990079072916709155572608779626631120569121199428868717596685973485783149883092999430252126430724631387222259355787645915179675324853179559846618733284245417582471252617636594781458638255005945562123898980555471677468331765004191548948299542243620770363890284827980798232428848801974257473320465223350595294108861210789516511484915753957128105280796657340907216130231437578491001998465816814269739974973049293975793876220302278929385521663290243622072112120834339646023775117143241528383404512487141130807160881242823878698475065166686165407644892963357466461316580111",
      "xr_cap": [
        [
          "master_secret",
          "243248142345559471584644550712419695629315583933249319776099088887711144607594342558156422917944153451967378480628000286975620468315622437489686903164413236204977995364634402443270188417976393192772803293125842641646009737910595547144484162605130785870499946398721315752215631086207781318244696949556022169282661342234819075947481633689162989045608084209361402558752145960958377771150212011508834008996029532402209714134486858171017918690544036382825404569789922796194067881325951792443095508262535018688140173606352531279409355038894728245115948017018820171857955748374988556207183057842582473393960116666236134134534836115942287968742467714467248485655293695652005439849126154917617217288673"
        ],
        [
          "score",
          "284888659965372309736860898678522790833375315928546918145292443617302874387323590399465323627566907498452641591529855157646495197061894143852461990108898876808764169767270630504666160359038676013198879092180891349448092255513185212626220971140469651889938339465044061511132929558427562298875832867481614453795554074887838708102369269722213310804194270169674600300317317623031674907453218566388050214975539771178497536007369383762323849862388675530442427298356228445779470680899704075975923001776673140911743018193222538257615683172196713351714418533785661732829571852469026322962455005574703660776811328488633596930568754594660086794765144355257798655994236043028644633023501127657335030838824"
        ]
      ]
    }
  },
  "credential_exchange_id": "6b82ab6f-844b-4bf0-a497-d40daeb98768",
  "updated_at": "2022-08-01T16:20:01.716277Z",
  "initiator": "external",
  "created_at": "2022-08-01T16:19:04.161558Z",
  "trace": true,
  "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
  "connection_id": "32527d07-9f6b-4072-bf03-d0d454e6c829",
  "state": "offer_sent",
  "credential_proposal_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/propose-credential",
    "@id": "a937ddb5-068e-46af-b0d5-a31afc17492a",
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "credential_proposal": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    }
  },
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "thread_id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
  "credential_offer_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/offer-credential",
    "@id": "2e56ed9e-6253-4343-912d-627908bfc10b",
    "~thread": {
      "thid": "73c0e6aa-7745-42fc-b688-3b69bde860b6"
    },
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "credential_preview": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "offers~attach": [
      {
        "@id": "libindy-cred-offer-0",
        "mime-type": "application/json",
        "data": {
          "base64": "eyJzY2hlbWFfaWQiOiAiQ0h2TWh5Y2RqVmkxSlJrS1EzeTd3aToyOnByZWZzOjEuMCIsICJjcmVkX2RlZl9pZCI6ICJDSHZNaHljZGpWaTFKUmtLUTN5N3dpOjM6Q0w6NjQ6ZGVmYXVsdCIsICJrZXlfY29ycmVjdG5lc3NfcHJvb2YiOiB7ImMiOiAiNDM3MjY1NzQwMTQ4NDA2NzQ1NjEzMDIxNjI2ODcxMjcwMTEyNjg2MTU0Mzk4MjEwNjIxOTkyNDM5OTkwNDEyMDAyNDE5NTgyNjM3MzQiLCAieHpfY2FwIjogIjU0ODg0ODcwODk4NTcxMDMyMjM2OTY1NjU0OTkzNjE5MDQzMjk0NjAwMTExMzAwMzczODk5MTIyNTI2NDIzMTc3NDg4MzE3MDMwODMxMDA3NTgyNDg4NjQ3MjAzODY3MDE2OTUxMzk3NjQzODEzMDM1MjMzODg1NjU5NTc4NDk5MDA3OTA3MjkxNjcwOTE1NTU3MjYwODc3OTYyNjYzMTEyMDU2OTEyMTE5OTQyODg2ODcxNzU5NjY4NTk3MzQ4NTc4MzE0OTg4MzA5Mjk5OTQzMDI1MjEyNjQzMDcyNDYzMTM4NzIyMjI1OTM1NTc4NzY0NTkxNTE3OTY3NTMyNDg1MzE3OTU1OTg0NjYxODczMzI4NDI0NTQxNzU4MjQ3MTI1MjYxNzYzNjU5NDc4MTQ1ODYzODI1NTAwNTk0NTU2MjEyMzg5ODk4MDU1NTQ3MTY3NzQ2ODMzMTc2NTAwNDE5MTU0ODk0ODI5OTU0MjI0MzYyMDc3MDM2Mzg5MDI4NDgyNzk4MDc5ODIzMjQyODg0ODgwMTk3NDI1NzQ3MzMyMDQ2NTIyMzM1MDU5NTI5NDEwODg2MTIxMDc4OTUxNjUxMTQ4NDkxNTc1Mzk1NzEyODEwNTI4MDc5NjY1NzM0MDkwNzIxNjEzMDIzMTQzNzU3ODQ5MTAwMTk5ODQ2NTgxNjgxNDI2OTczOTk3NDk3MzA0OTI5Mzk3NTc5Mzg3NjIyMDMwMjI3ODkyOTM4NTUyMTY2MzI5MDI0MzYyMjA3MjExMjEyMDgzNDMzOTY0NjAyMzc3NTExNzE0MzI0MTUyODM4MzQwNDUxMjQ4NzE0MTEzMDgwNzE2MDg4MTI0MjgyMzg3ODY5ODQ3NTA2NTE2NjY4NjE2NTQwNzY0NDg5Mjk2MzM1NzQ2NjQ2MTMxNjU4MDExMSIsICJ4cl9jYXAiOiBbWyJtYXN0ZXJfc2VjcmV0IiwgIjI0MzI0ODE0MjM0NTU1OTQ3MTU4NDY0NDU1MDcxMjQxOTY5NTYyOTMxNTU4MzkzMzI0OTMxOTc3NjA5OTA4ODg4NzcxMTE0NDYwNzU5NDM0MjU1ODE1NjQyMjkxNzk0NDE1MzQ1MTk2NzM3ODQ4MDYyODAwMDI4Njk3NTYyMDQ2ODMxNTYyMjQzNzQ4OTY4NjkwMzE2NDQxMzIzNjIwNDk3Nzk5NTM2NDYzNDQwMjQ0MzI3MDE4ODQxNzk3NjM5MzE5Mjc3MjgwMzI5MzEyNTg0MjY0MTY0NjAwOTczNzkxMDU5NTU0NzE0NDQ4NDE2MjYwNTEzMDc4NTg3MDQ5OTk0NjM5ODcyMTMxNTc1MjIxNTYzMTA4NjIwNzc4MTMxODI0NDY5Njk0OTU1NjAyMjE2OTI4MjY2MTM0MjIzNDgxOTA3NTk0NzQ4MTYzMzY4OTE2Mjk4OTA0NTYwODA4NDIwOTM2MTQwMjU1ODc1MjE0NTk2MDk1ODM3Nzc3MTE1MDIxMjAxMTUwODgzNDAwODk5NjAyOTUzMjQwMjIwOTcxNDEzNDQ4Njg1ODE3MTAxNzkxODY5MDU0NDAzNjM4MjgyNTQwNDU2OTc4OTkyMjc5NjE5NDA2Nzg4MTMyNTk1MTc5MjQ0MzA5NTUwODI2MjUzNTAxODY4ODE0MDE3MzYwNjM1MjUzMTI3OTQwOTM1NTAzODg5NDcyODI0NTExNTk0ODAxNzAxODgyMDE3MTg1Nzk1NTc0ODM3NDk4ODU1NjIwNzE4MzA1Nzg0MjU4MjQ3MzM5Mzk2MDExNjY2NjIzNjEzNDEzNDUzNDgzNjExNTk0MjI4Nzk2ODc0MjQ2NzcxNDQ2NzI0ODQ4NTY1NTI5MzY5NTY1MjAwNTQzOTg0OTEyNjE1NDkxNzYxNzIxNzI4ODY3MyJdLCBbInNjb3JlIiwgIjI4NDg4ODY1OTk2NTM3MjMwOTczNjg2MDg5ODY3ODUyMjc5MDgzMzM3NTMxNTkyODU0NjkxODE0NTI5MjQ0MzYxNzMwMjg3NDM4NzMyMzU5MDM5OTQ2NTMyMzYyNzU2NjkwNzQ5ODQ1MjY0MTU5MTUyOTg1NTE1NzY0NjQ5NTE5NzA2MTg5NDE0Mzg1MjQ2MTk5MDEwODg5ODg3NjgwODc2NDE2OTc2NzI3MDYzMDUwNDY2NjE2MDM1OTAzODY3NjAxMzE5ODg3OTA5MjE4MDg5MTM0OTQ0ODA5MjI1NTUxMzE4NTIxMjYyNjIyMDk3MTE0MDQ2OTY1MTg4OTkzODMzOTQ2NTA0NDA2MTUxMTEzMjkyOTU1ODQyNzU2MjI5ODg3NTgzMjg2NzQ4MTYxNDQ1Mzc5NTU1NDA3NDg4NzgzODcwODEwMjM2OTI2OTcyMjIxMzMxMDgwNDE5NDI3MDE2OTY3NDYwMDMwMDMxNzMxNzYyMzAzMTY3NDkwNzQ1MzIxODU2NjM4ODA1MDIxNDk3NTUzOTc3MTE3ODQ5NzUzNjAwNzM2OTM4Mzc2MjMyMzg0OTg2MjM4ODY3NTUzMDQ0MjQyNzI5ODM1NjIyODQ0NTc3OTQ3MDY4MDg5OTcwNDA3NTk3NTkyMzAwMTc3NjY3MzE0MDkxMTc0MzAxODE5MzIyMjUzODI1NzYxNTY4MzE3MjE5NjcxMzM1MTcxNDQxODUzMzc4NTY2MTczMjgyOTU3MTg1MjQ2OTAyNjMyMjk2MjQ1NTAwNTU3NDcwMzY2MDc3NjgxMTMyODQ4ODYzMzU5NjkzMDU2ODc1NDU5NDY2MDA4Njc5NDc2NTE0NDM1NTI1Nzc5ODY1NTk5NDIzNjA0MzAyODY0NDYzMzAyMzUwMTEyNzY1NzMzNTAzMDgzODgyNCJdXX0sICJub25jZSI6ICIxMDQ1MTg2NDI3OTg0NzU3NTM5ODAwMzUyIn0="
        }
      }
    ]
  },
  "role": "issuer"
}
```
```
curl -X 'POST' \
  'http://127.0.0.1:5001/issue-credential/records/4006de6f-3e86-4939-950f-85f344f67a16/send-request' \
  -H 'accept: application/json' \
  -d ''
```
```
{
  "credential_proposal_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/propose-credential",
    "@id": "63692b6c-495d-47ea-b8ec-70c076a49476",
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "credential_proposal": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "auto_remove": true,
  "initiator": "self",
  "state": "request_sent",
  "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
  "credential_offer": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "nonce": "1045186427984757539800352",
    "key_correctness_proof": {
      "c": "43726574014840674561302162687127011268615439821062199243999041200241958263734",
      "xz_cap": "548848708985710322369656549936190432946001113003738991225264231774883170308310075824886472038670169513976438130352338856595784990079072916709155572608779626631120569121199428868717596685973485783149883092999430252126430724631387222259355787645915179675324853179559846618733284245417582471252617636594781458638255005945562123898980555471677468331765004191548948299542243620770363890284827980798232428848801974257473320465223350595294108861210789516511484915753957128105280796657340907216130231437578491001998465816814269739974973049293975793876220302278929385521663290243622072112120834339646023775117143241528383404512487141130807160881242823878698475065166686165407644892963357466461316580111",
      "xr_cap": [
        [
          "master_secret",
          "243248142345559471584644550712419695629315583933249319776099088887711144607594342558156422917944153451967378480628000286975620468315622437489686903164413236204977995364634402443270188417976393192772803293125842641646009737910595547144484162605130785870499946398721315752215631086207781318244696949556022169282661342234819075947481633689162989045608084209361402558752145960958377771150212011508834008996029532402209714134486858171017918690544036382825404569789922796194067881325951792443095508262535018688140173606352531279409355038894728245115948017018820171857955748374988556207183057842582473393960116666236134134534836115942287968742467714467248485655293695652005439849126154917617217288673"
        ],
        [
          "score",
          "284888659965372309736860898678522790833375315928546918145292443617302874387323590399465323627566907498452641591529855157646495197061894143852461990108898876808764169767270630504666160359038676013198879092180891349448092255513185212626220971140469651889938339465044061511132929558427562298875832867481614453795554074887838708102369269722213310804194270169674600300317317623031674907453218566388050214975539771178497536007369383762323849862388675530442427298356228445779470680899704075975923001776673140911743018193222538257615683172196713351714418533785661732829571852469026322962455005574703660776811328488633596930568754594660086794765144355257798655994236043028644633023501127657335030838824"
        ]
      ]
    }
  },
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "credential_request_metadata": {
    "master_secret_blinding_data": {
      "v_prime": "25317018434503908443506220369965309820692159155580851287523459204159310820981639460612483199069704104815743638826191988470469039811187530604147644179510755072431582747075466046117269694035923448179739732823320693801409173847195739828764034090597398185731439823117996872191717738328303028406415255481128525022443782091974543920955279835029945123067190365558178741633837388093932361712951086511942236522015200325944561681281394871424995288635313267721438678362813405188307392904940148176667096561514316945956831652012528807589810710493400540698569918484437551747045313558304465076043893614639070998110314840375296013435003026836665613857641554",
      "vr_prime": null
    },
    "nonce": "447104860494845077080123",
    "master_secret_name": "my-wallet"
  },
  "thread_id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
  "trace": true,
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "credential_request": {
    "prover_did": "3HF6eMb1YDtyWxpWhyWKq7",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "blinded_ms": {
      "u": "92954683989542249563639027843525089993399665645036550114949941609920586589216348348305017596551820051194055601098782644408956055887887478127309823528680684981188381103488111524985864353833143469256429274029233217219365591956732107641901620548180230498049989453697465292352708818226588672387249600436597539463473612729225310177627318361825422561137157862987610845353805325129642533641827277177597484748922887650598217337728179195124438711650597493504933070311877156127608112078423777620543943308363214373876815236311923111253880757184074729166712983223839043205081553678002645923746985652721685007054979586128536029703",
      "ur": null,
      "hidden_attributes": [
        "master_secret"
      ],
      "committed_attributes": {}
    },
    "blinded_ms_correctness_proof": {
      "c": "43078936890854380271856643003695921304894869963254120314768826096656187745145",
      "v_dash_cap": "1090630239404590831388813689403048505454041084426501824642999491061288530552626039215581057843687952065354729299829974079893000360361040590225636986941572429302438321002017102528263724766616411937005982756690799734328178728994792383429098250221357334214429307340940967708602910620209780234245122786401343442394142790245933934147099093127361188600212886355684702525528177435874729590223886585856848767083564268054084351542162883603141865330428998770514659414280984923793552486402414204586098882939142033631111998902156843135520545547087078469852624139653402578283360430735290216302788214933969554649964377462796313718397409506516126622602611136078213774029594314612315664622811988094253831599186202772062128275827889420",
      "m_caps": {
        "master_secret": "16058753194625572061446481718578606479912902391182796933987132174300575541038554191346999661992362812011708509438764114950264723323853125860387632085796424088722803963555280637443"
      },
      "r_caps": {}
    },
    "nonce": "447104860494845077080123"
  },
  "role": "holder",
  "credential_exchange_id": "4006de6f-3e86-4939-950f-85f344f67a16",
  "updated_at": "2022-08-01T16:21:47.435430Z",
  "auto_issue": false,
  "credential_offer_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/offer-credential",
    "@id": "2e56ed9e-6253-4343-912d-627908bfc10b",
    "~thread": {
      "thid": "73c0e6aa-7745-42fc-b688-3b69bde860b6"
    },
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "credential_preview": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "offers~attach": [
      {
        "@id": "libindy-cred-offer-0",
        "mime-type": "application/json",
        "data": {
          "base64": "eyJzY2hlbWFfaWQiOiAiQ0h2TWh5Y2RqVmkxSlJrS1EzeTd3aToyOnByZWZzOjEuMCIsICJjcmVkX2RlZl9pZCI6ICJDSHZNaHljZGpWaTFKUmtLUTN5N3dpOjM6Q0w6NjQ6ZGVmYXVsdCIsICJrZXlfY29ycmVjdG5lc3NfcHJvb2YiOiB7ImMiOiAiNDM3MjY1NzQwMTQ4NDA2NzQ1NjEzMDIxNjI2ODcxMjcwMTEyNjg2MTU0Mzk4MjEwNjIxOTkyNDM5OTkwNDEyMDAyNDE5NTgyNjM3MzQiLCAieHpfY2FwIjogIjU0ODg0ODcwODk4NTcxMDMyMjM2OTY1NjU0OTkzNjE5MDQzMjk0NjAwMTExMzAwMzczODk5MTIyNTI2NDIzMTc3NDg4MzE3MDMwODMxMDA3NTgyNDg4NjQ3MjAzODY3MDE2OTUxMzk3NjQzODEzMDM1MjMzODg1NjU5NTc4NDk5MDA3OTA3MjkxNjcwOTE1NTU3MjYwODc3OTYyNjYzMTEyMDU2OTEyMTE5OTQyODg2ODcxNzU5NjY4NTk3MzQ4NTc4MzE0OTg4MzA5Mjk5OTQzMDI1MjEyNjQzMDcyNDYzMTM4NzIyMjI1OTM1NTc4NzY0NTkxNTE3OTY3NTMyNDg1MzE3OTU1OTg0NjYxODczMzI4NDI0NTQxNzU4MjQ3MTI1MjYxNzYzNjU5NDc4MTQ1ODYzODI1NTAwNTk0NTU2MjEyMzg5ODk4MDU1NTQ3MTY3NzQ2ODMzMTc2NTAwNDE5MTU0ODk0ODI5OTU0MjI0MzYyMDc3MDM2Mzg5MDI4NDgyNzk4MDc5ODIzMjQyODg0ODgwMTk3NDI1NzQ3MzMyMDQ2NTIyMzM1MDU5NTI5NDEwODg2MTIxMDc4OTUxNjUxMTQ4NDkxNTc1Mzk1NzEyODEwNTI4MDc5NjY1NzM0MDkwNzIxNjEzMDIzMTQzNzU3ODQ5MTAwMTk5ODQ2NTgxNjgxNDI2OTczOTk3NDk3MzA0OTI5Mzk3NTc5Mzg3NjIyMDMwMjI3ODkyOTM4NTUyMTY2MzI5MDI0MzYyMjA3MjExMjEyMDgzNDMzOTY0NjAyMzc3NTExNzE0MzI0MTUyODM4MzQwNDUxMjQ4NzE0MTEzMDgwNzE2MDg4MTI0MjgyMzg3ODY5ODQ3NTA2NTE2NjY4NjE2NTQwNzY0NDg5Mjk2MzM1NzQ2NjQ2MTMxNjU4MDExMSIsICJ4cl9jYXAiOiBbWyJtYXN0ZXJfc2VjcmV0IiwgIjI0MzI0ODE0MjM0NTU1OTQ3MTU4NDY0NDU1MDcxMjQxOTY5NTYyOTMxNTU4MzkzMzI0OTMxOTc3NjA5OTA4ODg4NzcxMTE0NDYwNzU5NDM0MjU1ODE1NjQyMjkxNzk0NDE1MzQ1MTk2NzM3ODQ4MDYyODAwMDI4Njk3NTYyMDQ2ODMxNTYyMjQzNzQ4OTY4NjkwMzE2NDQxMzIzNjIwNDk3Nzk5NTM2NDYzNDQwMjQ0MzI3MDE4ODQxNzk3NjM5MzE5Mjc3MjgwMzI5MzEyNTg0MjY0MTY0NjAwOTczNzkxMDU5NTU0NzE0NDQ4NDE2MjYwNTEzMDc4NTg3MDQ5OTk0NjM5ODcyMTMxNTc1MjIxNTYzMTA4NjIwNzc4MTMxODI0NDY5Njk0OTU1NjAyMjE2OTI4MjY2MTM0MjIzNDgxOTA3NTk0NzQ4MTYzMzY4OTE2Mjk4OTA0NTYwODA4NDIwOTM2MTQwMjU1ODc1MjE0NTk2MDk1ODM3Nzc3MTE1MDIxMjAxMTUwODgzNDAwODk5NjAyOTUzMjQwMjIwOTcxNDEzNDQ4Njg1ODE3MTAxNzkxODY5MDU0NDAzNjM4MjgyNTQwNDU2OTc4OTkyMjc5NjE5NDA2Nzg4MTMyNTk1MTc5MjQ0MzA5NTUwODI2MjUzNTAxODY4ODE0MDE3MzYwNjM1MjUzMTI3OTQwOTM1NTAzODg5NDcyODI0NTExNTk0ODAxNzAxODgyMDE3MTg1Nzk1NTc0ODM3NDk4ODU1NjIwNzE4MzA1Nzg0MjU4MjQ3MzM5Mzk2MDExNjY2NjIzNjEzNDEzNDUzNDgzNjExNTk0MjI4Nzk2ODc0MjQ2NzcxNDQ2NzI0ODQ4NTY1NTI5MzY5NTY1MjAwNTQzOTg0OTEyNjE1NDkxNzYxNzIxNzI4ODY3MyJdLCBbInNjb3JlIiwgIjI4NDg4ODY1OTk2NTM3MjMwOTczNjg2MDg5ODY3ODUyMjc5MDgzMzM3NTMxNTkyODU0NjkxODE0NTI5MjQ0MzYxNzMwMjg3NDM4NzMyMzU5MDM5OTQ2NTMyMzYyNzU2NjkwNzQ5ODQ1MjY0MTU5MTUyOTg1NTE1NzY0NjQ5NTE5NzA2MTg5NDE0Mzg1MjQ2MTk5MDEwODg5ODg3NjgwODc2NDE2OTc2NzI3MDYzMDUwNDY2NjE2MDM1OTAzODY3NjAxMzE5ODg3OTA5MjE4MDg5MTM0OTQ0ODA5MjI1NTUxMzE4NTIxMjYyNjIyMDk3MTE0MDQ2OTY1MTg4OTkzODMzOTQ2NTA0NDA2MTUxMTEzMjkyOTU1ODQyNzU2MjI5ODg3NTgzMjg2NzQ4MTYxNDQ1Mzc5NTU1NDA3NDg4NzgzODcwODEwMjM2OTI2OTcyMjIxMzMxMDgwNDE5NDI3MDE2OTY3NDYwMDMwMDMxNzMxNzYyMzAzMTY3NDkwNzQ1MzIxODU2NjM4ODA1MDIxNDk3NTUzOTc3MTE3ODQ5NzUzNjAwNzM2OTM4Mzc2MjMyMzg0OTg2MjM4ODY3NTUzMDQ0MjQyNzI5ODM1NjIyODQ0NTc3OTQ3MDY4MDg5OTcwNDA3NTk3NTkyMzAwMTc3NjY3MzE0MDkxMTc0MzAxODE5MzIyMjUzODI1NzYxNTY4MzE3MjE5NjcxMzM1MTcxNDQxODUzMzc4NTY2MTczMjgyOTU3MTg1MjQ2OTAyNjMyMjk2MjQ1NTAwNTU3NDcwMzY2MDc3NjgxMTMyODQ4ODYzMzU5NjkzMDU2ODc1NDU5NDY2MDA4Njc5NDc2NTE0NDM1NTI1Nzc5ODY1NTk5NDIzNjA0MzAyODY0NDYzMzAyMzUwMTEyNzY1NzMzNTAzMDgzODgyNCJdXX0sICJub25jZSI6ICIxMDQ1MTg2NDI3OTg0NzU3NTM5ODAwMzUyIn0="
        }
      }
    ]
  },
  "created_at": "2022-08-01T16:19:04.119401Z"
}
```

Issuer:
```
curl -X 'POST' \
  'http://127.0.0.1:5000/issue-credential/records/6b82ab6f-844b-4bf0-a497-d40daeb98768/issue' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "comment": "score30"
}'
```
{
  "auto_remove": true,
  "credential_offer": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "nonce": "1045186427984757539800352",
    "key_correctness_proof": {
      "c": "43726574014840674561302162687127011268615439821062199243999041200241958263734",
      "xz_cap": "548848708985710322369656549936190432946001113003738991225264231774883170308310075824886472038670169513976438130352338856595784990079072916709155572608779626631120569121199428868717596685973485783149883092999430252126430724631387222259355787645915179675324853179559846618733284245417582471252617636594781458638255005945562123898980555471677468331765004191548948299542243620770363890284827980798232428848801974257473320465223350595294108861210789516511484915753957128105280796657340907216130231437578491001998465816814269739974973049293975793876220302278929385521663290243622072112120834339646023775117143241528383404512487141130807160881242823878698475065166686165407644892963357466461316580111",
      "xr_cap": [
        [
          "master_secret",
          "243248142345559471584644550712419695629315583933249319776099088887711144607594342558156422917944153451967378480628000286975620468315622437489686903164413236204977995364634402443270188417976393192772803293125842641646009737910595547144484162605130785870499946398721315752215631086207781318244696949556022169282661342234819075947481633689162989045608084209361402558752145960958377771150212011508834008996029532402209714134486858171017918690544036382825404569789922796194067881325951792443095508262535018688140173606352531279409355038894728245115948017018820171857955748374988556207183057842582473393960116666236134134534836115942287968742467714467248485655293695652005439849126154917617217288673"
        ],
        [
          "score",
          "284888659965372309736860898678522790833375315928546918145292443617302874387323590399465323627566907498452641591529855157646495197061894143852461990108898876808764169767270630504666160359038676013198879092180891349448092255513185212626220971140469651889938339465044061511132929558427562298875832867481614453795554074887838708102369269722213310804194270169674600300317317623031674907453218566388050214975539771178497536007369383762323849862388675530442427298356228445779470680899704075975923001776673140911743018193222538257615683172196713351714418533785661732829571852469026322962455005574703660776811328488633596930568754594660086794765144355257798655994236043028644633023501127657335030838824"
        ]
      ]
    }
  },
  "credential_exchange_id": "6b82ab6f-844b-4bf0-a497-d40daeb98768",
  "updated_at": "2022-08-01T16:23:24.347250Z",
  "initiator": "external",
  "created_at": "2022-08-01T16:19:04.161558Z",
  "trace": true,
  "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
  "connection_id": "32527d07-9f6b-4072-bf03-d0d454e6c829",
  "state": "credential_issued",
  "credential_proposal_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/propose-credential",
    "@id": "a937ddb5-068e-46af-b0d5-a31afc17492a",
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "credential_proposal": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    }
  },
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "thread_id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
  "credential": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "credential_request": {
    "prover_did": "3HF6eMb1YDtyWxpWhyWKq7",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "blinded_ms": {
      "u": "92954683989542249563639027843525089993399665645036550114949941609920586589216348348305017596551820051194055601098782644408956055887887478127309823528680684981188381103488111524985864353833143469256429274029233217219365591956732107641901620548180230498049989453697465292352708818226588672387249600436597539463473612729225310177627318361825422561137157862987610845353805325129642533641827277177597484748922887650598217337728179195124438711650597493504933070311877156127608112078423777620543943308363214373876815236311923111253880757184074729166712983223839043205081553678002645923746985652721685007054979586128536029703",
      "ur": null,
      "hidden_attributes": [
        "master_secret"
      ],
      "committed_attributes": {}
    },
    "blinded_ms_correctness_proof": {
      "c": "43078936890854380271856643003695921304894869963254120314768826096656187745145",
      "v_dash_cap": "1090630239404590831388813689403048505454041084426501824642999491061288530552626039215581057843687952065354729299829974079893000360361040590225636986941572429302438321002017102528263724766616411937005982756690799734328178728994792383429098250221357334214429307340940967708602910620209780234245122786401343442394142790245933934147099093127361188600212886355684702525528177435874729590223886585856848767083564268054084351542162883603141865330428998770514659414280984923793552486402414204586098882939142033631111998902156843135520545547087078469852624139653402578283360430735290216302788214933969554649964377462796313718397409506516126622602611136078213774029594314612315664622811988094253831599186202772062128275827889420",
      "m_caps": {
        "master_secret": "16058753194625572061446481718578606479912902391182796933987132174300575541038554191346999661992362812011708509438764114950264723323853125860387632085796424088722803963555280637443"
      },
      "r_caps": {}
    },
    "nonce": "447104860494845077080123"
  },
  "credential_offer_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/offer-credential",
    "@id": "2e56ed9e-6253-4343-912d-627908bfc10b",
    "~thread": {
      "thid": "73c0e6aa-7745-42fc-b688-3b69bde860b6"
    },
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "credential_preview": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "offers~attach": [
      {
        "@id": "libindy-cred-offer-0",
        "mime-type": "application/json",
        "data": {
          "base64": "eyJzY2hlbWFfaWQiOiAiQ0h2TWh5Y2RqVmkxSlJrS1EzeTd3aToyOnByZWZzOjEuMCIsICJjcmVkX2RlZl9pZCI6ICJDSHZNaHljZGpWaTFKUmtLUTN5N3dpOjM6Q0w6NjQ6ZGVmYXVsdCIsICJrZXlfY29ycmVjdG5lc3NfcHJvb2YiOiB7ImMiOiAiNDM3MjY1NzQwMTQ4NDA2NzQ1NjEzMDIxNjI2ODcxMjcwMTEyNjg2MTU0Mzk4MjEwNjIxOTkyNDM5OTkwNDEyMDAyNDE5NTgyNjM3MzQiLCAieHpfY2FwIjogIjU0ODg0ODcwODk4NTcxMDMyMjM2OTY1NjU0OTkzNjE5MDQzMjk0NjAwMTExMzAwMzczODk5MTIyNTI2NDIzMTc3NDg4MzE3MDMwODMxMDA3NTgyNDg4NjQ3MjAzODY3MDE2OTUxMzk3NjQzODEzMDM1MjMzODg1NjU5NTc4NDk5MDA3OTA3MjkxNjcwOTE1NTU3MjYwODc3OTYyNjYzMTEyMDU2OTEyMTE5OTQyODg2ODcxNzU5NjY4NTk3MzQ4NTc4MzE0OTg4MzA5Mjk5OTQzMDI1MjEyNjQzMDcyNDYzMTM4NzIyMjI1OTM1NTc4NzY0NTkxNTE3OTY3NTMyNDg1MzE3OTU1OTg0NjYxODczMzI4NDI0NTQxNzU4MjQ3MTI1MjYxNzYzNjU5NDc4MTQ1ODYzODI1NTAwNTk0NTU2MjEyMzg5ODk4MDU1NTQ3MTY3NzQ2ODMzMTc2NTAwNDE5MTU0ODk0ODI5OTU0MjI0MzYyMDc3MDM2Mzg5MDI4NDgyNzk4MDc5ODIzMjQyODg0ODgwMTk3NDI1NzQ3MzMyMDQ2NTIyMzM1MDU5NTI5NDEwODg2MTIxMDc4OTUxNjUxMTQ4NDkxNTc1Mzk1NzEyODEwNTI4MDc5NjY1NzM0MDkwNzIxNjEzMDIzMTQzNzU3ODQ5MTAwMTk5ODQ2NTgxNjgxNDI2OTczOTk3NDk3MzA0OTI5Mzk3NTc5Mzg3NjIyMDMwMjI3ODkyOTM4NTUyMTY2MzI5MDI0MzYyMjA3MjExMjEyMDgzNDMzOTY0NjAyMzc3NTExNzE0MzI0MTUyODM4MzQwNDUxMjQ4NzE0MTEzMDgwNzE2MDg4MTI0MjgyMzg3ODY5ODQ3NTA2NTE2NjY4NjE2NTQwNzY0NDg5Mjk2MzM1NzQ2NjQ2MTMxNjU4MDExMSIsICJ4cl9jYXAiOiBbWyJtYXN0ZXJfc2VjcmV0IiwgIjI0MzI0ODE0MjM0NTU1OTQ3MTU4NDY0NDU1MDcxMjQxOTY5NTYyOTMxNTU4MzkzMzI0OTMxOTc3NjA5OTA4ODg4NzcxMTE0NDYwNzU5NDM0MjU1ODE1NjQyMjkxNzk0NDE1MzQ1MTk2NzM3ODQ4MDYyODAwMDI4Njk3NTYyMDQ2ODMxNTYyMjQzNzQ4OTY4NjkwMzE2NDQxMzIzNjIwNDk3Nzk5NTM2NDYzNDQwMjQ0MzI3MDE4ODQxNzk3NjM5MzE5Mjc3MjgwMzI5MzEyNTg0MjY0MTY0NjAwOTczNzkxMDU5NTU0NzE0NDQ4NDE2MjYwNTEzMDc4NTg3MDQ5OTk0NjM5ODcyMTMxNTc1MjIxNTYzMTA4NjIwNzc4MTMxODI0NDY5Njk0OTU1NjAyMjE2OTI4MjY2MTM0MjIzNDgxOTA3NTk0NzQ4MTYzMzY4OTE2Mjk4OTA0NTYwODA4NDIwOTM2MTQwMjU1ODc1MjE0NTk2MDk1ODM3Nzc3MTE1MDIxMjAxMTUwODgzNDAwODk5NjAyOTUzMjQwMjIwOTcxNDEzNDQ4Njg1ODE3MTAxNzkxODY5MDU0NDAzNjM4MjgyNTQwNDU2OTc4OTkyMjc5NjE5NDA2Nzg4MTMyNTk1MTc5MjQ0MzA5NTUwODI2MjUzNTAxODY4ODE0MDE3MzYwNjM1MjUzMTI3OTQwOTM1NTAzODg5NDcyODI0NTExNTk0ODAxNzAxODgyMDE3MTg1Nzk1NTc0ODM3NDk4ODU1NjIwNzE4MzA1Nzg0MjU4MjQ3MzM5Mzk2MDExNjY2NjIzNjEzNDEzNDUzNDgzNjExNTk0MjI4Nzk2ODc0MjQ2NzcxNDQ2NzI0ODQ4NTY1NTI5MzY5NTY1MjAwNTQzOTg0OTEyNjE1NDkxNzYxNzIxNzI4ODY3MyJdLCBbInNjb3JlIiwgIjI4NDg4ODY1OTk2NTM3MjMwOTczNjg2MDg5ODY3ODUyMjc5MDgzMzM3NTMxNTkyODU0NjkxODE0NTI5MjQ0MzYxNzMwMjg3NDM4NzMyMzU5MDM5OTQ2NTMyMzYyNzU2NjkwNzQ5ODQ1MjY0MTU5MTUyOTg1NTE1NzY0NjQ5NTE5NzA2MTg5NDE0Mzg1MjQ2MTk5MDEwODg5ODg3NjgwODc2NDE2OTc2NzI3MDYzMDUwNDY2NjE2MDM1OTAzODY3NjAxMzE5ODg3OTA5MjE4MDg5MTM0OTQ0ODA5MjI1NTUxMzE4NTIxMjYyNjIyMDk3MTE0MDQ2OTY1MTg4OTkzODMzOTQ2NTA0NDA2MTUxMTEzMjkyOTU1ODQyNzU2MjI5ODg3NTgzMjg2NzQ4MTYxNDQ1Mzc5NTU1NDA3NDg4NzgzODcwODEwMjM2OTI2OTcyMjIxMzMxMDgwNDE5NDI3MDE2OTY3NDYwMDMwMDMxNzMxNzYyMzAzMTY3NDkwNzQ1MzIxODU2NjM4ODA1MDIxNDk3NTUzOTc3MTE3ODQ5NzUzNjAwNzM2OTM4Mzc2MjMyMzg0OTg2MjM4ODY3NTUzMDQ0MjQyNzI5ODM1NjIyODQ0NTc3OTQ3MDY4MDg5OTcwNDA3NTk3NTkyMzAwMTc3NjY3MzE0MDkxMTc0MzAxODE5MzIyMjUzODI1NzYxNTY4MzE3MjE5NjcxMzM1MTcxNDQxODUzMzc4NTY2MTczMjgyOTU3MTg1MjQ2OTAyNjMyMjk2MjQ1NTAwNTU3NDcwMzY2MDc3NjgxMTMyODQ4ODYzMzU5NjkzMDU2ODc1NDU5NDY2MDA4Njc5NDc2NTE0NDM1NTI1Nzc5ODY1NTk5NDIzNjA0MzAyODY0NDYzMzAyMzUwMTEyNzY1NzMzNTAzMDgzODgyNCJdXX0sICJub25jZSI6ICIxMDQ1MTg2NDI3OTg0NzU3NTM5ODAwMzUyIn0="
        }
      }
    ]
  },
  "role": "issuer"
}
```

Client:
```
curl -X 'POST' \
  'http://127.0.0.1:5001/issue-credential/records/4006de6f-3e86-4939-950f-85f344f67a16/store' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "credential_id": "score30"
}'
```
```
{
  "credential_proposal_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/propose-credential",
    "@id": "63692b6c-495d-47ea-b8ec-70c076a49476",
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "credential_proposal": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "auto_remove": true,
  "credential": {
    "referent": "score30",
    "attrs": {
      "score": "30"
    },
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default"
  },
  "initiator": "self",
  "state": "credential_acked",
  "credential_definition_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
  "credential_offer": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "nonce": "1045186427984757539800352",
    "key_correctness_proof": {
      "c": "43726574014840674561302162687127011268615439821062199243999041200241958263734",
      "xz_cap": "548848708985710322369656549936190432946001113003738991225264231774883170308310075824886472038670169513976438130352338856595784990079072916709155572608779626631120569121199428868717596685973485783149883092999430252126430724631387222259355787645915179675324853179559846618733284245417582471252617636594781458638255005945562123898980555471677468331765004191548948299542243620770363890284827980798232428848801974257473320465223350595294108861210789516511484915753957128105280796657340907216130231437578491001998465816814269739974973049293975793876220302278929385521663290243622072112120834339646023775117143241528383404512487141130807160881242823878698475065166686165407644892963357466461316580111",
      "xr_cap": [
        [
          "master_secret",
          "243248142345559471584644550712419695629315583933249319776099088887711144607594342558156422917944153451967378480628000286975620468315622437489686903164413236204977995364634402443270188417976393192772803293125842641646009737910595547144484162605130785870499946398721315752215631086207781318244696949556022169282661342234819075947481633689162989045608084209361402558752145960958377771150212011508834008996029532402209714134486858171017918690544036382825404569789922796194067881325951792443095508262535018688140173606352531279409355038894728245115948017018820171857955748374988556207183057842582473393960116666236134134534836115942287968742467714467248485655293695652005439849126154917617217288673"
        ],
        [
          "score",
          "284888659965372309736860898678522790833375315928546918145292443617302874387323590399465323627566907498452641591529855157646495197061894143852461990108898876808764169767270630504666160359038676013198879092180891349448092255513185212626220971140469651889938339465044061511132929558427562298875832867481614453795554074887838708102369269722213310804194270169674600300317317623031674907453218566388050214975539771178497536007369383762323849862388675530442427298356228445779470680899704075975923001776673140911743018193222538257615683172196713351714418533785661732829571852469026322962455005574703660776811328488633596930568754594660086794765144355257798655994236043028644633023501127657335030838824"
        ]
      ]
    }
  },
  "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
  "credential_request_metadata": {
    "master_secret_blinding_data": {
      "v_prime": "25317018434503908443506220369965309820692159155580851287523459204159310820981639460612483199069704104815743638826191988470469039811187530604147644179510755072431582747075466046117269694035923448179739732823320693801409173847195739828764034090597398185731439823117996872191717738328303028406415255481128525022443782091974543920955279835029945123067190365558178741633837388093932361712951086511942236522015200325944561681281394871424995288635313267721438678362813405188307392904940148176667096561514316945956831652012528807589810710493400540698569918484437551747045313558304465076043893614639070998110314840375296013435003026836665613857641554",
      "vr_prime": null
    },
    "nonce": "447104860494845077080123",
    "master_secret_name": "my-wallet"
  },
  "thread_id": "73c0e6aa-7745-42fc-b688-3b69bde860b6",
  "trace": true,
  "connection_id": "ec5b5f1a-94ac-4c59-a7ee-99f39ff0ff89",
  "raw_credential": {
    "schema_id": "CHvMhycdjVi1JRkKQ3y7wi:2:prefs:1.0",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "values": {
      "score": {
        "raw": "30",
        "encoded": "30"
      }
    },
    "signature": {
      "p_credential": {
        "m_2": "83539608640074017410236827333582382685932596535888218494596029890964782170966",
        "a": "103443889749567556287898513464088104925424886067330931421515902735463499912685376075816559948241667400393156183485882422100148902652586117087484628101113632589471557814283506452757423034954938176362071153492752920658004739753351300494567491990074368595884212774739026780342635462044396707327125289248184883201523072880745016656586311872388056532447878583135174966182584319349962436684419085227161144483389947207421604338555880824549108798805154355069412667984451089586073581797203459545121456194352787232659532534824783806608740869144688864457839164546036167746938261684911785946723996086950047750792275717271641117470",
        "e": "259344723055062059907025491480697571938277889515152306249728583105665800713306759149981690559193987143012367913206299323899696942213235956742929746500333318066109447376126967369971",
        "v": "9258363843849825819739293500659257337128321631734433581873513548073532521256891555467563053745528293491967586064470041692296287546874720405223386635041035127189709182448583815562339399859648610623960915324914098205944989133247142923074800000689969775840680736115481998045125568439682598160022442432545309075805825636341082525841844532136278861438638939274941435725552765857317729077977118010020119017715378371666753353263359700090863284553249726562452749710939679951325380173616312100781725599417835077504270673379363517783203217112732545995153325864931190057026500754627437993502491561170187468687618294860173071046344954738519935869403288744116848463538069750339426699954393494007050849606937040279253923457010215595780261527652467110044298791967555280991110625892099932889234428578506933898288731079528491276556052136"
      },
      "r_credential": null
    },
    "signature_correctness_proof": {
      "se": "10901269105878443315110070363413658171057903464766067914008875810540117383254825987977172154370745871301889362693404226894729591317276115528850518994937181794934093214680313302203691537438362599887039585581680315041193755212359938583751622703568538179324040431045608574347287783742320658904394315392630181341248175312131807629616207391736101398484003266828230524233282800952318511950751156190594720404027004787727833854972135186515734922585335787634560624708821183280532680416819881566912366769022783078820445993082341543197215634715466208589279061787774111028353554122972552131822467388819787024464458545210510447477",
      "c": "44333745920731412151939654681042750625369363526309257824693807387908655367709"
    }
  },
  "credential_request": {
    "prover_did": "3HF6eMb1YDtyWxpWhyWKq7",
    "cred_def_id": "CHvMhycdjVi1JRkKQ3y7wi:3:CL:64:default",
    "blinded_ms": {
      "u": "92954683989542249563639027843525089993399665645036550114949941609920586589216348348305017596551820051194055601098782644408956055887887478127309823528680684981188381103488111524985864353833143469256429274029233217219365591956732107641901620548180230498049989453697465292352708818226588672387249600436597539463473612729225310177627318361825422561137157862987610845353805325129642533641827277177597484748922887650598217337728179195124438711650597493504933070311877156127608112078423777620543943308363214373876815236311923111253880757184074729166712983223839043205081553678002645923746985652721685007054979586128536029703",
      "ur": null,
      "hidden_attributes": [
        "master_secret"
      ],
      "committed_attributes": {}
    },
    "blinded_ms_correctness_proof": {
      "c": "43078936890854380271856643003695921304894869963254120314768826096656187745145",
      "v_dash_cap": "1090630239404590831388813689403048505454041084426501824642999491061288530552626039215581057843687952065354729299829974079893000360361040590225636986941572429302438321002017102528263724766616411937005982756690799734328178728994792383429098250221357334214429307340940967708602910620209780234245122786401343442394142790245933934147099093127361188600212886355684702525528177435874729590223886585856848767083564268054084351542162883603141865330428998770514659414280984923793552486402414204586098882939142033631111998902156843135520545547087078469852624139653402578283360430735290216302788214933969554649964377462796313718397409506516126622602611136078213774029594314612315664622811988094253831599186202772062128275827889420",
      "m_caps": {
        "master_secret": "16058753194625572061446481718578606479912902391182796933987132174300575541038554191346999661992362812011708509438764114950264723323853125860387632085796424088722803963555280637443"
      },
      "r_caps": {}
    },
    "nonce": "447104860494845077080123"
  },
  "role": "holder",
  "credential_exchange_id": "4006de6f-3e86-4939-950f-85f344f67a16",
  "credential_id": "score30",
  "updated_at": "2022-08-01T16:26:34.091236Z",
  "auto_issue": false,
  "credential_offer_dict": {
    "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/offer-credential",
    "@id": "2e56ed9e-6253-4343-912d-627908bfc10b",
    "~thread": {
      "thid": "73c0e6aa-7745-42fc-b688-3b69bde860b6"
    },
    "~trace": {
      "target": "log",
      "full_thread": true,
      "trace_reports": []
    },
    "credential_preview": {
      "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/issue-credential/1.0/credential-preview",
      "attributes": [
        {
          "name": "score",
          "value": "30"
        }
      ]
    },
    "offers~attach": [
      {
        "@id": "libindy-cred-offer-0",
        "mime-type": "application/json",
        "data": {
          "base64": "eyJzY2hlbWFfaWQiOiAiQ0h2TWh5Y2RqVmkxSlJrS1EzeTd3aToyOnByZWZzOjEuMCIsICJjcmVkX2RlZl9pZCI6ICJDSHZNaHljZGpWaTFKUmtLUTN5N3dpOjM6Q0w6NjQ6ZGVmYXVsdCIsICJrZXlfY29ycmVjdG5lc3NfcHJvb2YiOiB7ImMiOiAiNDM3MjY1NzQwMTQ4NDA2NzQ1NjEzMDIxNjI2ODcxMjcwMTEyNjg2MTU0Mzk4MjEwNjIxOTkyNDM5OTkwNDEyMDAyNDE5NTgyNjM3MzQiLCAieHpfY2FwIjogIjU0ODg0ODcwODk4NTcxMDMyMjM2OTY1NjU0OTkzNjE5MDQzMjk0NjAwMTExMzAwMzczODk5MTIyNTI2NDIzMTc3NDg4MzE3MDMwODMxMDA3NTgyNDg4NjQ3MjAzODY3MDE2OTUxMzk3NjQzODEzMDM1MjMzODg1NjU5NTc4NDk5MDA3OTA3MjkxNjcwOTE1NTU3MjYwODc3OTYyNjYzMTEyMDU2OTEyMTE5OTQyODg2ODcxNzU5NjY4NTk3MzQ4NTc4MzE0OTg4MzA5Mjk5OTQzMDI1MjEyNjQzMDcyNDYzMTM4NzIyMjI1OTM1NTc4NzY0NTkxNTE3OTY3NTMyNDg1MzE3OTU1OTg0NjYxODczMzI4NDI0NTQxNzU4MjQ3MTI1MjYxNzYzNjU5NDc4MTQ1ODYzODI1NTAwNTk0NTU2MjEyMzg5ODk4MDU1NTQ3MTY3NzQ2ODMzMTc2NTAwNDE5MTU0ODk0ODI5OTU0MjI0MzYyMDc3MDM2Mzg5MDI4NDgyNzk4MDc5ODIzMjQyODg0ODgwMTk3NDI1NzQ3MzMyMDQ2NTIyMzM1MDU5NTI5NDEwODg2MTIxMDc4OTUxNjUxMTQ4NDkxNTc1Mzk1NzEyODEwNTI4MDc5NjY1NzM0MDkwNzIxNjEzMDIzMTQzNzU3ODQ5MTAwMTk5ODQ2NTgxNjgxNDI2OTczOTk3NDk3MzA0OTI5Mzk3NTc5Mzg3NjIyMDMwMjI3ODkyOTM4NTUyMTY2MzI5MDI0MzYyMjA3MjExMjEyMDgzNDMzOTY0NjAyMzc3NTExNzE0MzI0MTUyODM4MzQwNDUxMjQ4NzE0MTEzMDgwNzE2MDg4MTI0MjgyMzg3ODY5ODQ3NTA2NTE2NjY4NjE2NTQwNzY0NDg5Mjk2MzM1NzQ2NjQ2MTMxNjU4MDExMSIsICJ4cl9jYXAiOiBbWyJtYXN0ZXJfc2VjcmV0IiwgIjI0MzI0ODE0MjM0NTU1OTQ3MTU4NDY0NDU1MDcxMjQxOTY5NTYyOTMxNTU4MzkzMzI0OTMxOTc3NjA5OTA4ODg4NzcxMTE0NDYwNzU5NDM0MjU1ODE1NjQyMjkxNzk0NDE1MzQ1MTk2NzM3ODQ4MDYyODAwMDI4Njk3NTYyMDQ2ODMxNTYyMjQzNzQ4OTY4NjkwMzE2NDQxMzIzNjIwNDk3Nzk5NTM2NDYzNDQwMjQ0MzI3MDE4ODQxNzk3NjM5MzE5Mjc3MjgwMzI5MzEyNTg0MjY0MTY0NjAwOTczNzkxMDU5NTU0NzE0NDQ4NDE2MjYwNTEzMDc4NTg3MDQ5OTk0NjM5ODcyMTMxNTc1MjIxNTYzMTA4NjIwNzc4MTMxODI0NDY5Njk0OTU1NjAyMjE2OTI4MjY2MTM0MjIzNDgxOTA3NTk0NzQ4MTYzMzY4OTE2Mjk4OTA0NTYwODA4NDIwOTM2MTQwMjU1ODc1MjE0NTk2MDk1ODM3Nzc3MTE1MDIxMjAxMTUwODgzNDAwODk5NjAyOTUzMjQwMjIwOTcxNDEzNDQ4Njg1ODE3MTAxNzkxODY5MDU0NDAzNjM4MjgyNTQwNDU2OTc4OTkyMjc5NjE5NDA2Nzg4MTMyNTk1MTc5MjQ0MzA5NTUwODI2MjUzNTAxODY4ODE0MDE3MzYwNjM1MjUzMTI3OTQwOTM1NTAzODg5NDcyODI0NTExNTk0ODAxNzAxODgyMDE3MTg1Nzk1NTc0ODM3NDk4ODU1NjIwNzE4MzA1Nzg0MjU4MjQ3MzM5Mzk2MDExNjY2NjIzNjEzNDEzNDUzNDgzNjExNTk0MjI4Nzk2ODc0MjQ2NzcxNDQ2NzI0ODQ4NTY1NTI5MzY5NTY1MjAwNTQzOTg0OTEyNjE1NDkxNzYxNzIxNzI4ODY3MyJdLCBbInNjb3JlIiwgIjI4NDg4ODY1OTk2NTM3MjMwOTczNjg2MDg5ODY3ODUyMjc5MDgzMzM3NTMxNTkyODU0NjkxODE0NTI5MjQ0MzYxNzMwMjg3NDM4NzMyMzU5MDM5OTQ2NTMyMzYyNzU2NjkwNzQ5ODQ1MjY0MTU5MTUyOTg1NTE1NzY0NjQ5NTE5NzA2MTg5NDE0Mzg1MjQ2MTk5MDEwODg5ODg3NjgwODc2NDE2OTc2NzI3MDYzMDUwNDY2NjE2MDM1OTAzODY3NjAxMzE5ODg3OTA5MjE4MDg5MTM0OTQ0ODA5MjI1NTUxMzE4NTIxMjYyNjIyMDk3MTE0MDQ2OTY1MTg4OTkzODMzOTQ2NTA0NDA2MTUxMTEzMjkyOTU1ODQyNzU2MjI5ODg3NTgzMjg2NzQ4MTYxNDQ1Mzc5NTU1NDA3NDg4NzgzODcwODEwMjM2OTI2OTcyMjIxMzMxMDgwNDE5NDI3MDE2OTY3NDYwMDMwMDMxNzMxNzYyMzAzMTY3NDkwNzQ1MzIxODU2NjM4ODA1MDIxNDk3NTUzOTc3MTE3ODQ5NzUzNjAwNzM2OTM4Mzc2MjMyMzg0OTg2MjM4ODY3NTUzMDQ0MjQyNzI5ODM1NjIyODQ0NTc3OTQ3MDY4MDg5OTcwNDA3NTk3NTkyMzAwMTc3NjY3MzE0MDkxMTc0MzAxODE5MzIyMjUzODI1NzYxNTY4MzE3MjE5NjcxMzM1MTcxNDQxODUzMzc4NTY2MTczMjgyOTU3MTg1MjQ2OTAyNjMyMjk2MjQ1NTAwNTU3NDcwMzY2MDc3NjgxMTMyODQ4ODYzMzU5NjkzMDU2ODc1NDU5NDY2MDA4Njc5NDc2NTE0NDM1NTI1Nzc5ODY1NTk5NDIzNjA0MzAyODY0NDYzMzAyMzUwMTEyNzY1NzMzNTAzMDgzODgyNCJdXX0sICJub25jZSI6ICIxMDQ1MTg2NDI3OTg0NzU3NTM5ODAwMzUyIn0="
        }
      }
    ]
  },
  "created_at": "2022-08-01T16:19:04.119401Z"
}
```
