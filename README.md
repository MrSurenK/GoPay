# GoPay
An interactive payment system visualiser.(This is largely based on my learnings from ByteByteGo content regarding payments system design)
 
## Technologies Used
1. Golang (Backend)
2. Kafka -> To facilitate event driven architecture and stream data as it interacts with the various services
3. React & typescript (Frontend)  
4. Docker -> To manage the verious microservices in the payment system
5. RDB - PostgreSQL (refer below to comparison between MySQL)

## System Design:

There are two key flows in a payment system: 
    1. Pay in 
    2. Pay out

### Constraints/Reasoning of design
1. The focus will mainly on the pay in system with a simple pay out system to demonstrate the transactions taking place in this demo.
2. For simplicity, currency is limited to SGD although there is a possiblity to extend the systems capabilities to account for exchange rates and currecnies as in a more real world scenario.
3. For external payment service provider(PSP), Stripe will be used as it has the best and most comprehensive free API and sandbox ecosystem.

## Pay-in flow
 
                    Payment Event
                            |
                    Fraud check service(Simple deterministic mocking)
                            |               
 Database record <- Payment Service -> Payment Order -> Payment Executor -> Smart Routing Service -> PSP (to handle actual movement of monies between banks and card schemes)
 of payment order           |                                      |
                                                      Database record of payment event
                        /        \                \           
                Ledger Service   Wallet Service   Email Service
                       |                |           |
                       DB               DB          DB


                

Why PostgreSQL over MySQL for database? 

| Feature | Postgres | MySQL | Financial Impact | 
| ------- | -------- | ----- | ---------------- |
| Check Contraints | Strict enforcement out-of-the-box | Also supported since 8.0 | Ensures balance never drops below zero |
| Concurrency Control | True Serialization Isolation(SSI) | Repeatable Read (can suffer from write-skew) | Prevents race conditions and double spending |
| Unstructured Data | Advanced binary JSON (JSONB) with indexing using GIN (Generalized Inverted Indexing) | Standard JSON text type - slower indexing as everytime a nested property is queried the database must parse the entire JSON text string | Speeds up queries on payment gateway metadata payloads |
| Data Integrity | Strict error throwing on invalid data types | Can silently truncate or use defaults depending on mode | Prevents corrupted currency values or invalid dates | 
| Mathematical Precision | Native NUMERIC handle exact decimals flawlessly | DECIMAL works well but math functions can round up | Gurantees zero penny-dropping or rounding errors | 
| Extensibility | Supports custom types, ranges and advanced triggers | Limited extension ecosystem | Allows for powerful, automated internal auditing | 


 
