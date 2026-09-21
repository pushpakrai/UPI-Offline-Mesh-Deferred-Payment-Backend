# UPI Offline Mesh Deferred Settlement Payment Backend

A Java 17 and Spring Boot backend that simulates **offline UPI-style mesh-routed payments with deferred settlement** through a bridge-ingestion API.

The system models a payment created on an offline device, propagated through a simulated device-to-device mesh, and eventually uploaded by an internet-connected bridge node. The backend securely processes the payment, prevents duplicate settlement, validates packet freshness, and performs atomic debit/credit settlement.

> **Important:** This project is a software simulation of an offline mesh payment architecture. 

## Key Features

* Offline payment packet creation and mesh propagation simulation
* Deferred settlement through an internet-connected bridge node
* Hybrid encryption using **RSA-2048 OAEP + AES-256-GCM**
* SHA-256 ciphertext hashing for packet identity and idempotency
* GCM authentication-tag validation for tamper detection
* Nonce and timestamp-based freshness validation
* Atomic duplicate protection using `ConcurrentHashMap.putIfAbsent`
* Transactional debit/credit settlement using JPA `@Transactional`
* Optimistic locking using JPA `@Version`
* Database-level unique packet-hash constraint
* Concurrent duplicate-delivery protection
* H2 in-memory database
* REST APIs for payment ingestion, mesh operations, accounts, and transactions
* Interactive browser dashboard
* Automated cryptography, tamper-resistance, and concurrency tests

## Technology Stack

| Category    | Technology                              |
| ----------- | --------------------------------------- |
| Language    | Java 17                                 |
| Framework   | Spring Boot 3.3                         |
| Build Tool  | Maven                                   |
| Persistence | Spring Data JPA                         |
| Database    | H2                                      |
| Encryption  | RSA-2048 OAEP, AES-256-GCM              |
| Hashing     | SHA-256                                 |
| Concurrency | `ConcurrentHashMap`, optimistic locking |
| Testing     | JUnit / Spring Boot Test                |
| API         | REST                                    |
| UI          | Spring Boot + HTML                      |
| Runtime     | Embedded Spring Boot server             |

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    OFFLINE SENDER DEVICE                    │
│                                                             │
│  PaymentInstruction                                         │
│  sender • receiver • amount • nonce • timestamp             │
│                         │                                   │
│                         ▼                                   │
│             RSA-2048 OAEP + AES-256-GCM                     │
│                         │                                   │
│                         ▼                                   │
│                    MeshPacket                               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          │ Device-to-device gossip
                          ▼
              ┌───────────────────────┐
              │   Virtual Mesh Nodes  │
              │                       │
              │  Device → Device      │
              │  Device → Device      │
              │  Device → Bridge      │
              └───────────┬───────────┘
                          │
                          │ Bridge obtains connectivity
                          ▼
                 POST /api/bridge/ingest
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                 SPRING BOOT BACKEND                         │
│                                                             │
│  1. SHA-256 ciphertext hash                                 │
│                 │                                           │
│  2. Atomic idempotency claim                                │
│                 │                                           │
│  3. RSA-OAEP key unwrap                                     │
│                 │                                           │
│  4. AES-256-GCM decryption + authentication                 │
│                 │                                           │
│  5. Timestamp / freshness validation                        │
│                 │                                           │
│  6. @Transactional settlement                               │
│                 │                                           │
│  7. Debit + Credit + Transaction Ledger                     │
└─────────────────────────────────────────────────────────────┘
```

## Payment Flow

### 1. Create Payment

A payment instruction contains:

* Sender
* Receiver
* Amount
* PIN hash
* Unique nonce
* Creation timestamp

A fresh AES-256 key encrypts the payment payload using AES-GCM.

The AES key is then encrypted using the server's RSA-2048 public key.

The resulting encrypted payload is placed inside a `MeshPacket`.

### 2. Propagate Through the Mesh

The packet is passed between simulated virtual devices.

Each device can carry and forward the encrypted packet without being able to read the payment details.

The mesh simulation models:

* Packet propagation
* Hop count
* TTL
* Multiple devices
* Bridge nodes

### 3. Bridge Ingestion

When a bridge node obtains internet connectivity, it sends the packet to:

```text
POST /api/bridge/ingest
```

The backend processes the packet through the following pipeline:

```text
Ciphertext
    ↓
SHA-256 Hash
    ↓
Idempotency Check
    ↓
RSA-OAEP Decryption
    ↓
AES-256-GCM Authentication + Decryption
    ↓
Freshness Validation
    ↓
Transactional Settlement
    ↓
Ledger Entry
```

### 4. Settlement

The settlement operation executes within a database transaction:

```java
@Transactional
```

The backend:

1. Loads the sender account
2. Validates available balance
3. Debits the sender
4. Credits the receiver
5. Creates a transaction ledger entry

JPA optimistic locking provides additional protection against concurrent account updates.

---

## Security Design

### Hybrid Encryption

RSA is used to protect the AES session key, while AES-GCM encrypts the payment payload.

```text
Payment JSON
     │
     ▼
AES-256-GCM
     │
     ▼
Encrypted Payload
     │
     │ AES key
     ▼
RSA-2048 OAEP
     │
     ▼
Encrypted AES Key
```

Conceptually, the packet contains:

```text
[RSA-encrypted AES key]
[12-byte GCM IV]
[AES-GCM ciphertext + authentication tag]
```

This avoids attempting to encrypt the complete payment payload directly with RSA.

### Tamper Detection

AES-GCM provides authenticated encryption.

If an intermediary modifies the encrypted payload, authentication fails during decryption and the packet is rejected instead of being settled.

### Packet Hashing

The backend calculates:

```text
SHA-256(ciphertext)
```

The resulting hash is used for idempotency and transaction identification.

The ciphertext is used instead of `packetId` because packet metadata could potentially be modified independently of the encrypted payment payload.

### Idempotency

The first ingestion step attempts to atomically claim the packet hash:

```java
seen.putIfAbsent(packetHash, timestamp);
```

Conceptually:

```text
First request
     ↓
Hash not present
     ↓
Claim hash
     ↓
Process payment
```

Duplicate request:

```text
Same hash
     ↓
Hash already claimed
     ↓
DUPLICATE_DROPPED
     ↓
No settlement
```

This protects against multiple bridge nodes submitting the same payment concurrently.

### Database-Level Protection

The transaction ledger also maintains a unique constraint on the packet hash.

This provides defense in depth if multiple requests reach the persistence layer.

### Replay Protection

Each payment contains:

* A unique nonce
* A creation timestamp

The backend rejects packets outside the configured freshness window.

A replayed copy of the same packet also produces the same ciphertext hash and is therefore rejected by the idempotency layer.

---

## Concurrency Handling

The system is designed for the case where multiple bridge nodes upload the same packet simultaneously.

Example:

```text
                 Same payment packet
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Bridge A       Bridge B       Bridge C
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                  Backend Ingestion
                         │
                  SHA-256 packet hash
                         │
                         ▼
              Atomic idempotency claim
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          First request        Duplicate requests
              │                     │
              ▼                     ▼
          Settlement          DUPLICATE_DROPPED
```

The included concurrency test verifies that simultaneous delivery of the same packet results in:

```text
1 × SETTLED
2 × DUPLICATE_DROPPED
```

and only one account debit.

---

## Project Structure

```text
UPI-Offline-Mesh---Deferred-Payment-Backend/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
│
├── src/
│   ├── main/
│   │   ├── java/com/demo/upimesh/
│   │   │
│   │   ├── UpiMeshApplication.java
│   │   │
│   │   ├── model/
│   │   │   ├── Account.java
│   │   │   ├── AccountRepository.java
│   │   │   ├── Transaction.java
│   │   │   ├── TransactionRepository.java
│   │   │   ├── MeshPacket.java
│   │   │   └── PaymentInstruction.java
│   │   │
│   │   ├── crypto/
│   │   │   ├── ServerKeyHolder.java
│   │   │   └── HybridCryptoService.java
│   │   │
│   │   ├── service/
│   │   │   ├── DemoService.java
│   │   │   ├── VirtualDevice.java
│   │   │   ├── MeshSimulatorService.java
│   │   │   ├── IdempotencyService.java
│   │   │   ├── SettlementService.java
│   │   │   └── BridgeIngestionService.java
│   │   │
│   │   ├── controller/
│   │   │   ├── ApiController.java
│   │   │   └── DashboardController.java
│   │   │
│   │   └── config/
│   │       └── AppConfig.java
│   │
│   ├── resources/
│   │   ├── application.properties
│   │   └── templates/
│   │       └── dashboard.html
│   │
│   └── test/
│       └── java/com/demo/upimesh/
│           └── IdempotencyConcurrencyTest.java
```

---

## Requirements

* JDK 17+
* Git
* Internet connection for the initial Maven dependency download

Maven does not need to be installed separately because the project includes the Maven Wrapper.

Check Java:

```bash
java -version
```

---

## Run the Application

### Windows

From the project root:

```powershell
.\mvnw.cmd spring-boot:run
```

### macOS / Linux

```bash
./mvnw spring-boot:run
```

After startup, open:

```text
http://localhost:8080
```

The application provides an interactive interface for creating payments, running mesh propagation, uploading packets through bridge nodes, and viewing account balances and transactions.

---

## Run Tests

### Windows

```powershell
.\mvnw.cmd test
```

### macOS / Linux

```bash
./mvnw test
```

The test suite covers:

* Encryption/decryption round trip
* Tampered ciphertext rejection
* Concurrent duplicate packet delivery
* Exactly-once settlement behavior

Run the concurrency test specifically:

```powershell
.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

---

## REST API

| Method | Endpoint             | Purpose                            |
| ------ | -------------------- | ---------------------------------- |
| `GET`  | `/`                  | Application dashboard              |
| `GET`  | `/api/server-key`    | Retrieve server RSA public key     |
| `GET`  | `/api/accounts`      | Retrieve account balances          |
| `GET`  | `/api/transactions`  | Retrieve recent transactions       |
| `GET`  | `/api/mesh/state`    | Retrieve virtual mesh state        |
| `POST` | `/api/demo/send`     | Create and inject a payment packet |
| `POST` | `/api/mesh/gossip`   | Execute a mesh gossip round        |
| `POST` | `/api/mesh/flush`    | Upload bridge packets to backend   |
| `POST` | `/api/mesh/reset`    | Reset mesh state                   |
| `POST` | `/api/bridge/ingest` | Process an incoming payment packet |
| `GET`  | `/h2-console`        | Access H2 database console         |

### Bridge Ingestion Request

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-encrypted-payload"
}
```

### Response

Successful settlement:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Duplicate:

```json
{
  "outcome": "DUPLICATE_DROPPED",
  "packetHash": "a3f8c9...",
  "reason": "Packet already processed",
  "transactionId": null
}
```

Invalid packet:

```json
{
  "outcome": "INVALID",
  "packetHash": "a3f8c9...",
  "reason": "Invalid or tampered ciphertext",
  "transactionId": null
}
```

---

## H2 Database

The application uses an in-memory H2 database for local execution.

H2 console:

```text
http://localhost:8080/h2-console
```

Default JDBC URL:

```text
jdbc:h2:mem:upimesh
```

Username:

```text
sa
```

Password:

```text
```

The database is intended for local development and testing.

---

## Production Considerations

This repository intentionally uses lightweight infrastructure so the complete architecture can run locally.

A production implementation would require additional infrastructure and security controls.

| Current implementation          | Production consideration                                      |
| ------------------------------- | ------------------------------------------------------------- |
| H2 in-memory database           | PostgreSQL / MySQL with appropriate replication               |
| `ConcurrentHashMap` idempotency | Redis `SET NX EX` or equivalent distributed store             |
| Startup-generated RSA key pair  | Managed key storage / HSM / KMS                               |
| Simulated virtual devices       | Android / Kotlin client implementation                        |
| Software mesh simulation        | BLE GATT / Wi-Fi Direct / appropriate transport               |
| Local settlement ledger         | Integration with banking/payment infrastructure               |
| Basic bridge ingestion          | Mutual TLS, signed bridge identities, authentication          |
| Seeded demo accounts            | Proper customer/account infrastructure                        |
| Local H2 console                | Disabled in production                                        |
| Basic application logging       | Structured logging, monitoring, alerting and SIEM integration |
| No production rate limiting     | Per-node and per-account rate limiting                        |

---

## Design Limitations

This project models **deferred settlement**, not guaranteed offline finality.

### Balance Verification

Because settlement occurs after connectivity is restored, the receiver cannot obtain authoritative server-side balance confirmation at the moment the payment is created offline.

A payment may therefore be rejected later if the sender does not have sufficient funds when settlement occurs.

### Offline Double Spending

A malicious sender could potentially create multiple offline payment instructions before any of them reach the settlement backend.

The backend can ensure that each packet is processed once, but preventing offline double spending requires additional mechanisms such as a pre-funded secure wallet or hardware-backed offline authorization model.

### Mesh Connectivity

The mesh is software-simulated.

Real-world phone-to-phone communication introduces additional challenges involving:

* BLE background restrictions
* Device discovery
* Connection reliability
* Battery consumption
* Platform restrictions
* Packet forwarding reliability
* Node trust
* Privacy

These concerns are outside the scope of this backend implementation.

### Payment Network Integration

This project does not connect to:

* NPCI
* UPI production infrastructure
* Real bank accounts
* Real VPAs
* Real payment gateways

It models the backend architecture and settlement mechanics using local accounts and an H2 database.

---
