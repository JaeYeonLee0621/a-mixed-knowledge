# Chapter 11. Payment System

## Payment System
- It is any system used to settle financial transactions through the transfer of monetary value.
- This includes the institutions, instruments, people, rules, procedures, standard, and technologies that make its exchage possible.

# Step 1. Understand the problem and establish design scope

## Functional Requirements
- `Pay-in flow` : Payment system receives money from customers on behalf of sellers
- `Pay-out flow` : Payment system sends money to sellers around the world

## Non-functional Requirements
- Reliability and Fault tolerance
- Reconciliation Process : Internal services <-> External services

## Back-of-the-envelope estimation
- 1 million transaction per day
- 10 transactions per second (TPS)
- 10 TPS is not a big number of a typical database

# Step 2. Propose High-Level Design and Get Buy-In

![KakaoTalk_Photo_2024-05-27-23-51-01 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3c701ba3-0142-455b-93ec-a0b27d1e727d)

# Pay-In Flow

![KakaoTalk_Photo_2024-05-27-23-51-01 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a1f4bd15-9840-47c0-b303-5a81950a29ac)

## Payment Service
- Accepting payment events from users and coordinates they payment process
- Risk-check, assessing for compliance

ex) Regulations (ex) AML/CFT), Money laundering, financing of terrorism

- The risk check service uses a third-party provider

## Payment executor
- Executing a single payment order via a Payment Service Provider (PSP)

## Payment Service Provider (PSP)
- Moving money from account A to accoun tB

## Card schemes
- Organizations that process credit card operations

ex) Visa, MasterCard, Discovery etc

## Ledger
- Keeping a financial record of the payment transaction
- Very important in post-payment analysis (ex) calculating the total revenue of the e-commerce website or forecasting future revenue)

## Wallet
- Keeping the account balance of the merchant
- Recording how much a given user has paid in total

+) amount field data type : "string" rather thaan "double"

1. Different protocols, software and hardward may support differnt numeric precisions in serialization and deserialization
2. The number could be extermely big or extermely small

ex) Japan's GDP ~= 5x10^14 yen / Satoshi of Bitcoin ~= 10^(-8)

##  Hosted Payment Page

Most companies prepfer not to store credit card information internally
- If they do, they have to deal with complex regulations (ex) Payment Card Industry Data Security Standard (PCI DSS))

To avoid handling credit card informaation
- Companies use hosted credit card pages provided by PSPs

1. Web site : A widget or an ifram
2. Application : A pre-built page from the payment SDK

![KakaoTalk_Photo_2024-05-27-23-51-02 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5baf2698-a852-4166-88b9-5f7d869bc413)

# Step 3. Design Deep Dive

# PSP integration

1. Using `API` : Can safely store sensitive payment information
2. `Hosted Payment Page` : Not to store sensitive payment information due to complex regulations and security concerns

<br/>

![KakaoTalk_Photo_2024-05-28-01-00-15](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d84fc89a-aa48-442d-82d6-19765ea7065b)

1. User -> Payment Service : Click the "checkout" button

2. Payment Service -> PSP : Sending payment registration request

ex) Payment information (ex) amount, currency, expiration date, redirect URL, UUID (-> Exactly-once registration))

+) UUID = nonce

3. PSP -> Payment Service  : a UUID on the PSP side that uniquely identifies the payment registration

4. Payment Service -> Database : Storing the token
5. User : Seeing the PSP-hosted payment page and filling in the payment details

5-1. PSP side token :  The PSP's javascript code uses the token to retrieve detailed information about payment request from the PSP's backend.

5-2. Redirect URL : Calling when the payment is complete

6. USer -> PSP : Executing payment
7. PSP -> User : Returning and The web page is now redirected to the redirect URL with payment status
9. PSP -> Payment Service : Asynchronously calling via a webhook

### +) Payment Process

![Toss 단건 결제](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/faed0e1f-549f-4f06-b208-9753ff6ba212)

## Reconsiliation

![KakaoTalk_Photo_2024-05-27-23-51-02 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/118bcdca-3a28-4791-a515-dd2275c01c61)

### Problem
- When system components communicate asynchronously
- there is no guarantee that a message will be delivered or response will be returned

### Solution
- Periodically comparing the states among related services in order to verify that they are in agreement

1. Every night the PSP or banks send a settlement file to their clients
2. Verifying consistence of internal payment system

:finger-right: The finance team to perform manually adjustments

### Mismatch

1. Classifiable / Automatic adjustment : write a program to automate the adjustment
2. Classifiable / Unalbe to automate the adjustment : The mismatch is put into a job queue and the finance team fixed the mixmatch manually
3. Unclassifiable : Do not know how to mismatch happens and the finance team investigates it manually

## Handling payment processing data

### Some examples where a payment request could take longer than usual
1. The PSP deems a payment request high risk and requires a human to review it
2. A credit card requires extra protection like 3D Secure Authentication

### Handling these long-running payment requests

1. The PSP woudl return a pending status our client and providing a page for the customer to check the current payment status
2. The PSP tracks the pending payment on our behalf and notifies the payment service of any status update via the webhook

### Finally payment completed

1. the PSP calls the registered `webhook`
2. The Payment service `poll` the PSP for status upates

## Communication among internal services

### 1. Synchronous communication
1. Low performance
2. Poor failure isolation
3. Tight coupling
4. Hard to scale

### 2. Asynchronous communication
1. Single receiver : Once a message is processed, it gets removed from the queue

![KakaoTalk_Photo_2024-05-27-23-51-02 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/166d9e0a-65f7-4e30-be7e-ce0effe90861)

2. Multiple receivers : Each request is processed by multiple receivers

![KakaoTalk_Photo_2024-05-27-23-51-02 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/44249013-3133-4e03-bf4e-1814aa9ce4b4)

## Handling failed payments

### 1. Tracking payment state
- Deciding whether a retry or refund is needed

### 2. Retry queue and dead letter queue

![KakaoTalk_Photo_2024-05-27-23-51-02 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/30665ca1-cd9b-40b1-be89-e2af616c2ce5)

## Exactly-Once delivery
- Double charge a customer

### Exactly-Once Operation
1. It is executed at-least once
2. At the same time, it is executed at-most-once

### 1. Retry

- Providing at-least-once guarantee

![KakaoTalk_Photo_2024-05-27-23-51-02 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/609f06be-102e-419a-955a-3b43970504b7)

**Retry Strategy**

1. Immediate retry : Client immediately resends a request
2. Fixted intervals : Wait a fixed amount of time 
3. Incremental intervals : Incrementally increasing the time for subsequent retires
4. Exponential backoff : Double the waiting time between retries (ex) 2^n)
5. Cancel

**Potential Problems**

1. Client clicks the pay button twice
2. Despite successful payment, the response fails to reach our payment system, the user click the pay button again

### 2. Idempotency

- Clients can make the same call repeatedly and produce the same result
- A unique value that is generated by the client and expires after a certain period of time (ex) UUID)

**Potential Scenario**

![KakaoTalk_Photo_2024-05-27-23-51-03 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/88ddf8b4-7a71-4c24-9f1b-a59c85da032b)

- Only one request is processed
- The others receive the "429 too Many Requests"

> Database unique constraints

## Consistency

- In a distributed environment, the communication between any two services can fail, causing data inconsistency

### 1. Internal <-> External
- Using idempotency key

### 2. Replication

1. Servce both reads and writes from the primary database only
2. Ensuring all replicas are always in-sync

## Payment Security

![KakaoTalk_Photo_2024-05-27-23-55-01](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ae3e1b21-444a-4630-9416-cfeca331885c)
+) `Request/Response eavesdropping` - `Use HTTPS`

### Data tampering
- the act of deliberately modifying (destroying, manipulating, or editing) data through unauthorized channels
### PCI (Payment Card Industry Compliance)
- The technical and operational standars that businesses follow to secure and protect credit card data provided by cardholders and transmitted through card processing transactions
### PCI DSS (Payment Card Industry Data Security Standard)
- A set of rules and guidelines designed to help organizations that handle credit card information keep that information safe and secure
