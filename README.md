# <p style="text-align: center;">BizShake MVP Technical Document</p>
<p style="text-align: center;">Version 0.1</p>


### General architecture

This document introduces the architecture of [BizShake](https://bizshake.io), the P2P sharing ecosystem based on BlockChain.

To this end, we have designed and developed a web application, which in its alpha release enables users to publish and share their assets via the [SmartRent](https://bizshake.io/#block-bean-business-section) business model. 

In the next releases we will implement also [SmartPawn](https://bizshake.io/#block-bean-business-section), [SmartDispute](https://bizshake.io/#block-bean-business-section), [SmartCertify](https://bizshake.io/#block-bean-business-section) and [SmartAPI](https://bizshake.io/#block-bean-business-section), as per our Roadmap.


### dApps

#### Web App

The Web App is currently developed in ASP.net and C#; its frontend is written in responsive HTML5, allowing mobile users to access the functionalities from the browsers on their devices until the Mobile Apps will be developed as well.

The functionalities implemented in this release are the following:
* Manage Users.
* Manage Assets.
* Manage Rent Offers.
* Manage Rent Deals: in this version there is no payment and wallet management. This will be implemented in a future release.
* Search function.
* Navigation of assets, offers and deals.

The WebApp allows users to represent real-world Assets and define Rent Offers. These are in draft mode and only visible to the owner, until (s)he decides to publish them. At that point, they are written to the BlockChain, and they become public.

This means that other dApps, developed using BizShake SmartAPI, will be able to search and access the published information for other purposes. In other words, the BlockChain is the master copy of all relevant data and dApps must synchronize to it.

#### Mobile Apps

The mobile apps for iOS and Android will be developed at a later stage. 


#### Database

The database schema includes entities for:
* Users
* Assets
* Offers
* Deals
* ...

To be noted that [ONT IDs](https://github.com/ontio/ontology-DID) are fully integrated in the WebApp for:

* Users' logins
* Assets' identities
* Assets' ownership certificates
* Assets' valuation certificates

As described in the BlockChain section below.


### BlockChain

The BlockChain technology of choice is [NEO](https://neo.org), and the upcoming ICO will be based on it. However, as our business logic heavily relies on the ability to certify and identify real-world assets, the [Ontology](https://ont.io) BlockChain, based on NEO Virtual Machine (NEO VM), has been chosen for this alpha release.

#### [ONT ID](https://github.com/ontio/ontology-DID)

In BizShake, ONT IDs are associated to Users and Assets.

Users log in to BizShake via an associated ONT ID, which is created on the fly during sign up (if not provided); this also implies that users' passwords are not stored in the BizShake database: the authorization for letting the user effectively login into BizShake dApps is coming from the BlockChain.

An excerpt of the code generating new ONT IDs through the Ontology SDK follows:

    static generateNewIdentity = async (privateKeyNewIdentity: PrivateKey,
                password: string, label: string, payer: any) => {
        try {
            const identity = Identity.create(privateKeyNewIdentity, password, label);
            const gasPrice = '500';
            const gasLimit = '20000';
            const tx = OntidContract.buildRegisterOntidTx(identity.ontid,
                            privateKeyNewIdentity.getPublicKey(), gasPrice, gasLimit);
            tx.payer = payer.account.address;
            TransactionBuilder.signTransaction(tx, payer.privateKey);
            TransactionBuilder.addSign(tx, privateKeyNewIdentity);
            const result = await client.sendRawTransaction(tx.serialize(), false, true);
            return result.Result.State === 0 ? null : identity;
        } catch(e) {
            console.log(e);
            return null;
        }
    };

Conversely, the code used to validate an existing ONT ID is:

    static async checkOntID(ontid: string){

        const tx = OntidContract.buildGetDDOTx(ontid);
        // tx.payer = account.address;
        // TransactionBuilder.signTransaction(tx, privateKey);
        var result = await client.sendRawTransaction(tx.serialize(), true);
        var hexResult = result.Result.Result;
        if (hexResult.length > 0) {
            result = {
                success: true,
                message: "",
                ddo: DDO.deserialize(result.Result.Result)
            };
        } else {
            result = {
                success: false,
                message: "ontid not found"
            }
        }
        return result;
    };

Furthermore, published Assets are automatically assigned an ONT ID for further identification and processing by BizShake dApps: when as Asset is published, BizShake will issue an ownership "certificate" using the [Verifiable Claim Protocol](https://ontio.github.io/documentation/claim_spec_en.html). In this way anyone will be able to verify on the BlockChain who is the ONT ID of the owner of a given Asset; BizShake is acting as *Claim Verifier*.

In the same way BizShake will implement SmartCertify, where specific Certifier Users will be able to professionally certify and evaluate users' Assets and issue a specific "certificate" on the BlockChain.

[comment]: https://github.com/ontio/ontology-DID/blob/master/docs/en/verification_provider_specification.md


#### Smart Contract

The BizShake Smart Contract supports the following actions:
* Store and retrieve Assets metadata, not including multimedia data (e.g. assets' photos and videos)
* Store and retrieve published Offers metadata
* Handle all valid states of a Deal and their transitions for both SmartRent and SmartPawn

The Smart Contract will additionally integrate the following features:
* Management of all monetary transactions related to the Deals
* Management of Assets certification for SmartCertify
* Management and storage of Disputes for SmartDispute

The current alpha version implements SmartRent, while SmartPawn will be available at a later stage.

#### ICO

The ICO/STO will be held after BizShake will get the approval from the US SEC for the sale of its Security Token (BZS). The ICO smart contract will be developed and deployed at a later stage.


### Middleware

Currently, the ONT BlockChain is very actively developed, and a number of features are either not yet available or in an early stage of development. For this reason, the current alpha release includes a middleware layer, which implements in a traditional way what will be implemented in the Smart Contract at a later stage.

The middleware layer is written as a NodeJS application, in order to exploit the TypeScript-based Ontology SDK for all BlockChain-related functionalities. 

Currently, it includes the following APIs:
* Store, retrieve, and delete entities in the internal key-value store of the Smart Contract. Data is stored in JSON format and the key is the ID in the WebApp database.
* Create, verify and manage ONTID, the Ontology identifiers for users and assets.
