# GoPay
An interactive payment system visualiser.(This is largely based on my learnings from ByteByteGo content regarding payments system design with my own additions to futher illustrate the system design)
 
## Technologies Used
1. Golang (Backend)
2. Kafka -> To facilitate event driven architecture and stream data as it interacts with the various services
3. React & typescript (Frontend)  
4. Docker -> To manage the verious microservices in the payment system
5. RDB - PostgreSQL (refer below to comparison between MySQL)

## System Design:

There are two key flows in a payment system: 
1. Pay in 
2. Pay out (Will auto payout at a given interval to demonstrate the exchange of money)

### Constraints/Reasoning of design
1. The focus will mainly on the pay in system with a simple pay out system to demonstrate the transactions taking place in this demo.
2. For simplicity, currency is limited to SGD although there is a possiblity to extend the systems capabilities to account for exchange rates and currecnies as in a more real world scenario.
3. For external payment service provider(PSP), Stripe will be used as it has the best and most comprehensive free API and sandbox ecosystem.

## Pay-in flow
 ```
                      Payment Event
                            |
                    Fraud check service(Simple deterministic mocking)
                            |               
 Database record <- Payment Service -> Payment Order -> Smart Routing Service -> Payment Executor -> PSP (to handle actual movement of monies between banks and card schemes)
 of payment order          ||                                                           |                                        |
       Inventory Service <-||                                                    Update DB of Payment Event                   (success? yes)   
               |            |                                                                                                    |
Reserve inventory in        |                                                                                                    |   
db until successful in PSP  |                                                                                                    |   
                            |                                                                                                    |   
                            |                                                                                                    |   
---------------------------KAFKA ------------------------------------------------------------------------------------------------
           /             /           \                 \          
     Ledger Service   Wallet Service   Email Service   Fufilment Service
            |                |              |              | 
            DB               DB             DB      Update Inventory DB to permanently deduct stock 

```
> Note: Payement Service is the "brain" of this architecture and it will update the DB as the payment flows throught the different services even after the PSP has successfully processed the payment and the events are sent to kafka

Why PostgreSQL over MySQL for database? 

| Feature | Postgres | MySQL | Financial Impact | 
| ------- | -------- | ----- | ---------------- |
| Check Contraints | Strict enforcement out-of-the-box | Also supported since 8.0 | Ensures balance never drops below zero |
| Concurrency Control | True Serialization Isolation(SSI) | Repeatable Read (can suffer from write-skew) | Prevents race conditions and double spending |
| Unstructured Data | Advanced binary JSON (JSONB) with indexing using GIN (Generalized Inverted Indexing) | Standard JSON text type - slower indexing as everytime a nested property is queried the database must parse the entire JSON text string | Speeds up queries on payment gateway metadata payloads |
| Data Integrity | Strict error throwing on invalid data types | Can silently truncate or use defaults depending on mode | Prevents corrupted currency values or invalid dates | 
| Mathematical Precision | Native NUMERIC handle exact decimals flawlessly | DECIMAL works well but math functions can round up | Gurantees zero penny-dropping or rounding errors | 
| Extensibility | Supports custom types, ranges and advanced triggers | Limited extension ecosystem | Allows for powerful, automated internal auditing | 

### Resilience Measures

__Health checks for all services will be done by docker deamon itself and self healing property `restart always` will run when it detects that a service is down__

1. What if PSP failed the transaction?
> If the transaction failed at the PSP service the API would return a failure. Since monies has not moved out we can then just update the DB as Failed transaction and pass the same response back to the user in the UI with a "pls try again" later message for instance. Since our DB will be the single source of truth, a payment marked as failed will not be picked up and processed again.  

2. What if any individual service is down after PSP is successful? (Ledger service, Wallet Service or Email Service)
> Now in this situation, money has already moved and thus is it of utmost importance that the system reliably reflects this. In the event of a failure, the service will be offline or take too long to send a response to KAFKA, KAFKA will then add this particular transaction using a unique identifier such as id to a retry queue or to be processed at a later time. Since KAFKA manages all the various services independently, a single service going down will not have an effect on the others. Furthermore money has already moved and such all business logic following that must be reflected in the services that follow.  

> So what if email is sent but inventory or other services have not yet been updated? User/Customer does not need to know or wait for services to be up and running but need to reliably know that they would receive what they have paid for. As such the system must be designed to reserve inventory before the payment goes through the PSP and only release it if PSP returns a failed transaction. If PSP is sucessful here then the fufilment service will make the inventory deduction permanent. For simplicity ssake that is all the fufilment service here will do but in a real production system it should also handle other details relating to shipping/delivery to customer. 

3. What if Payment service itself is down? - The brain of the architecture
> If payment services itself is down, no transaction has been processed or gone thorugh. In which case, the front end can just throw an error to the user and request to try again anther time to gracefully handle it on the front end. From system perspective, a health check daemon will be able to detect that the service is down and spin it back up again, at which time it can begin taking in new orders and orchestrating the flow again. 

4. What if KAFKA is down? 
> Again here we rely on our DB to be our single source of truth. If KAFKA is down when it comes back up it will scan the DB and pick up the transactions that are still pending and continue from where it left off.

5. What if PSP is down?
> Money has not moved yet so we can simply handle it gracefully in the UI and allow the user to try again at a later time. We do not want to allow it similar to if KAFKA is down as it will be very bad for user experience. User or Customers will not be willing to wait long for a service to get back up and then get updated. Its better to have them just try again at a later time. 

6. What if Smart Routing Service is down?
> A _Static Circuit Breaking Fallback_ will be implemented. A default PSP will be hardcoded and if the service is down the payment executor will simply forward the payment details to the default PSP. 

7. What if the entire DB itslef is down? -> A scalable solution will be discussed here but application will not implement a full DB recovery plan. Instead a persistent volume in docker will be used to save data and simulate a DB down
> In a production environment a synchronous Primary-Replica Cluster will be implemented in different AZs say using Aurora. If the DB was truly down, the Replica cluster that is read only can be promoted to be a primary cluster to maintain data integrity and system resilience. Since the Primary-Replica cluster is synchronous, every time data is updated in the primary cluster the replica cluster will also be updated accordingly. 

8. We discussed having a retry queue but what about a dead letter queue? 
> If say we have a particular transaction that fails internal processing repeatedly, we do not want to be stuck trying it indefinitely. A dead letter queue will also be implemented so as to catch these transactions to allow for manual intervention or futher investigations. 

## Idempotency - How do we ensure that transactions are not accidentally duplicated through the system? 

The most naive answer to this would be to limit the user from spamming the checkout or payment button on the frontend. However, suppose the user has access to the endpoints on the backend and calls it directly with POSTMAN or it could be that there is a bug in the UI and user is allowed to spam the pay button. This would be a very expensive mistake and cause a huge financial headache for all parties invloved. It is thus also required to handle idempotency on the system end.  

The strategy here would be to track the transaction id within a session closely. Each transaction ID has to be uniquely generated, using say UUID generator for instance would get a globally unique transaction id. This UUID will be generated and assigned to the checkout cart. So no matter how many times the user selects pay, as long as the cart remains the same in the session it will be the same and thus blocking multiple repeat requests. The PSP would also handle idempotency via tokens on their end to ensure it does not charge the same transaction twice 


