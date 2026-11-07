# UPI Offline Mesh

A Spring Boot-based simulation of offline UPI payment routing using a Bluetooth-style mesh network.

The system allows a payment packet to be created without an internet connection, propagated across nearby devices, and eventually delivered to a backend through an internet-connected bridge node. The backend securely decrypts the payment, prevents duplicate settlement under concurrent delivery, detects tampering/replay attempts, and performs the final settlement.

> **Note:** This is a learning/portfolio implementation inspired by the [original UPI_Without_Internet project](https://github.com/perryvegehan/UPI_Without_Internet). The project uses a software-simulated mesh rather than real Bluetooth hardware.

---

## Why this project?

Traditional online payment systems assume that the sender can communicate with the payment backend at the time of payment.

This project explores a different model:

```
Sender Phone
     │
     │ Encrypted payment packet
     ▼
Offline Device
     │
     │ Mesh propagation
     ▼
Offline Device
     │
     │ Mesh propagation
     ▼
Bridge Device ────── Internet ──────► Backend
                                          │
                                          ▼
                                 Decrypt + Validate
                                          │
                                          ▼
                                Idempotent Settlement
```

The main engineering challenges are:
- Securely transporting payment data through untrusted devices
- Preventing duplicate settlement when the same payment arrives concurrently
- Detecting tampered and replayed payment packets

---

## Key Features

### 🔐 Hybrid Encryption
Payment data is encrypted using a hybrid cryptographic scheme:
- **AES-256-GCM** encrypts the payment payload.
- **RSA-OAEP** encrypts the AES session key.
- **GCM authentication** detects ciphertext tampering during decryption.
- Intermediate devices only carry encrypted data and cannot read the payment contents.

```
Payment JSON
     │
     ▼
AES-256-GCM
     │
     ├── Ciphertext
     │
AES Session Key
     │
     ▼
 RSA-OAEP
     │
     ▼
Encrypted AES Key
```

### ⚡ Idempotent Payment Settlement
The same payment may reach the backend through multiple bridge devices.

To prevent the sender from being debited multiple times:
- **SHA-256 hash** of the ciphertext is generated.
- The hash is atomically claimed using `ConcurrentHashMap.putIfAbsent()`.
- Only the first request is allowed to continue.
- Concurrent duplicate requests are rejected before decryption and settlement.

This provides exactly-once processing for the simulated delivery scenario. In a distributed production environment, this mechanism could be implemented using Redis `SET NX` with an expiration time.

### 🛡️ Tamper & Replay Protection
The encrypted payment contains:
- A unique nonce
- A timestamp

The backend:
- Rejects packets whose authenticated ciphertext has been modified.
- Rejects packets older than the allowed freshness window.
- Uses the ciphertext hash to prevent replaying the exact same payment.

### 🌐 Mesh Simulation
The project includes a software simulation of offline devices.

Each virtual device can:
- Store payment packets
- Propagate packets to neighboring devices
- Decrement packet TTL
- Act as an internet-connected bridge

This allows the complete offline-to-online flow to be demonstrated on a single machine without requiring Bluetooth hardware.

### 💳 Transaction Settlement
After successful validation:
- Sender account is debited
- Receiver account is credited
- Transaction is recorded in the ledger
- Database transaction boundaries ensure the debit/credit operation is atomic
- Optimistic locking is also used on account records as an additional concurrency safeguard.

---

## Architecture

```
OFFLINE ENVIRONMENT
┌──────────────┐
│ Sender Phone │
└──────┬───────┘
       │
       │ Encrypted MeshPacket
       ▼
┌──────────────┐
│ Offline Node │
└──────┬───────┘
       │
       │ Gossip
       ▼
┌──────────────┐
│ Offline Node │
└──────┬───────┘
       │
       │ Gossip
       ▼
┌──────────────┐
│ Bridge Node  │
│ has Internet │
└──────┬───────┘
       │
       │ HTTP POST
       ▼
┌────────────────────────────────────┐
│       Spring Boot Backend          │
│                                    │
│   SHA-256 Ciphertext Hash          │
│                ↓                   │
│         Idempotency Check          │
│                ↓                   │
│     RSA + AES-GCM Decryption       │
│                ↓                   │
│       Freshness Validation         │
│                ↓                   │
│      Transaction Settlement        │
│                ↓                   │
│        Transaction Ledger          │
└────────────────────────────────────┘
```

---

## Tech Stack

- **Java**
- **Spring Boot**
- **Spring Data JPA**
- **H2 Database**
- **Maven**
- **RSA-OAEP**
- **AES-256-GCM**
- **SHA-256**
- **Java Concurrency / ConcurrentHashMap**
- **JUnit**

---

## API

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/accounts` | Retrieve account balances |
| `GET` | `/api/transactions` | Retrieve recent transactions |
| `GET` | `/api/mesh/state` | View simulated mesh state |
| `POST` | `/api/demo/send` | Create and inject a payment |
| `POST` | `/api/mesh/gossip` | Propagate packets through the mesh |
| `POST` | `/api/mesh/flush` | Upload packets from bridge nodes |
| `POST` | `/api/mesh/reset` | Reset the simulation |
| `POST` | `/api/bridge/ingest` | Receive a payment packet from a bridge |

---

## Running the Project

### Prerequisites
- **JDK 17+**
- No external database required
- Maven Wrapper is included

### Windows
```cmd
.\mvnw.cmd spring-boot:run
```

### Linux / macOS
```bash
./mvnw spring-boot:run
```

Then open:
**http://localhost:8080**

### Run Tests

**Windows:**
```cmd
.\mvnw.cmd test
```

**Linux/macOS:**
```bash
./mvnw test
```

---

## Important Test Scenarios

The project includes tests for the core reliability and security properties:

- **Encryption round trip**  
  Verifies that encrypted payment data can be successfully decrypted.

- **Tampered ciphertext**  
  Modifies the ciphertext and verifies that authentication fails instead of allowing the transaction to settle.

- **Concurrent duplicate delivery**  
  Simulates multiple threads delivering the same payment simultaneously and verifies that:
  - 1 request → `SETTLED`
  - 2 requests → `DUPLICATE_DROPPED`
  - The sender's balance must change only once.

---

## Project Structure

```
src/
├── main/
│   ├── java/
│   │   └── ...
│   └── resources/
└── test/
    └── java/
        └── ...
```

Important components include:
- `HybridCryptoService` — encryption/decryption
- `IdempotencyService` — duplicate detection
- `BridgeIngestionService` — backend ingestion pipeline
- `SettlementService` — transaction settlement
- `MeshSimulatorService` — offline mesh simulation

---

The implementation here focuses on understanding the backend architecture, cryptography, concurrency, idempotency, and reliability challenges involved in deferred offline payment settlement.
