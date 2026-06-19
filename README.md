<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&weight=700&size=28&duration=3000&pause=1000&color=00D4FF&center=true&vCenter=true&width=700&lines=UPI+Offline+Mesh+%E2%80%94+Payment+Without+Internet;Bluetooth+Mesh+%2B+Hybrid+Encryption;Built+with+Spring+Boot+%7C+Java+17" alt="Typing SVG" />

<br/>

<p align="center">
  <img src="https://img.shields.io/badge/Java-17+-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white"/>
  <img src="https://img.shields.io/badge/Spring_Boot-3.3.5-6DB33F?style=for-the-badge&logo=springboot&logoColor=white"/>
  <img src="https://img.shields.io/badge/Maven-Build-C71A36?style=for-the-badge&logo=apachemaven&logoColor=white"/>
  <img src="https://img.shields.io/badge/H2-In--Memory_DB-blue?style=for-the-badge&logo=h2&logoColor=white"/>
  <img src="https://img.shields.io/badge/Encryption-RSA_2048_+_AES_256_GCM-8B0000?style=for-the-badge&logo=letsencrypt&logoColor=white"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Status-Production_Ready_Demo-00C851?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Tests-Concurrency_Safe-blueviolet?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-Open_for_Learning-orange?style=for-the-badge"/>
</p>

<br/>

> **"You're in a basement with zero connectivity. You pay ₹500 to your friend. No internet. No problem."**
>
> The payment hops phone-to-phone through Bluetooth, encrypted end-to-end — until one device walks outside, gets 4G, and silently settles it with the bank. No data was readable. No amount was altered. No payment was duplicated.

<br/>

</div>

---

## 🌐 The Problem This Solves

India has **hundreds of millions** of people in areas with poor or zero internet connectivity — basements, rural zones, metro stations, disaster zones. Existing UPI requires a live internet connection for every transaction. This project proves a different model:

```
OFFLINE → ENCRYPTED → MESH-HOPPED → BRIDGE → SETTLED
```

A payment travels through **untrusted stranger phones** as an encrypted blob nobody can read, until someone with internet uploads it — and the bank settles it **exactly once**, no matter how many people delivered it.

---

## 📐 System Architecture

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                     SENDER PHONE  (completely offline)                       ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────┐   ║
║  │  PaymentInstruction {                                                 │   ║
║  │    senderVpa:   "alice@demo"    receiverVpa: "bob@demo"              │   ║
║  │    amount:      ₹500.00         pinHash:     [HMAC-SHA256]           │   ║
║  │    nonce:       [UUID-v4]       signedAt:    [epoch-ms]              │   ║
║  │  }                                                                    │   ║
║  └───────────────────────────┬──────────────────────────────────────────┘   ║
║                              │  🔐 Hybrid Encrypt                           ║
║                              │  Step 1: AES-256-GCM encrypts JSON           ║
║                              │  Step 2: RSA-2048/OAEP encrypts AES key      ║
║                              ▼                                               ║
║  ┌──────────────────────────────────────────────────────────────────────┐   ║
║  │  MeshPacket {                                                         │   ║
║  │    packetId:   [UUID]     ttl: 5    createdAt: [timestamp]           │   ║
║  │    ciphertext: [256B RSA-enc AES key | 12B IV | AES ciphertext+tag]  │   ║
║  │  }                        ← opaque to ALL intermediaries             │   ║
║  └───────────────────────────┬──────────────────────────────────────────┘   ║
╚══════════════════════════════│═════════════════════════════════════════════╝
                               │
                               │   📶 Bluetooth Gossip Mesh  (TTL decrements each hop)
                               ▼
     ┌─────────┐  hop-1   ┌─────────┐  hop-2   ┌─────────┐  hop-3   ┌──────────┐
     │  Alice  │ ────────▶│Stranger1│ ────────▶│Stranger2│ ────────▶│  Bridge  │
     │(offline)│          │(offline)│          │(offline)│          │ (4G ✅)  │
     └─────────┘          └─────────┘          └─────────┘          └────┬─────┘
                                                                          │
                                                                          │  HTTPS POST
                                                                          ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                    SPRING BOOT BACKEND  (This Project)                       ║
║                                                                              ║
║   POST /api/bridge/ingest                                                    ║
║        │                                                                     ║
║        ├─ [1] SHA-256(ciphertext)  →  idempotency key                       ║
║        │                                                                     ║
║        ├─ [2] IdempotencyService.claim(hash)                                 ║
║        │        ConcurrentHashMap.putIfAbsent()  ≡  Redis SETNX             ║
║        │        ✅ First caller  →  proceed                                  ║
║        │        ❌ All others   →  DUPLICATE_DROPPED  (no DB touch)         ║
║        │                                                                     ║
║        ├─ [3] HybridCryptoService.decrypt(ciphertext)                        ║
║        │        RSA-OAEP decrypts AES key  →  AES-GCM decrypts payload      ║
║        │        GCM auth tag verification  →  tamper = exception             ║
║        │                                                                     ║
║        ├─ [4] Freshness check  (signedAt must be within 24h)                 ║
║        │        → Prevents replay attacks on old captured packets            ║
║        │                                                                     ║
║        └─ [5] SettlementService.settle()   @Transactional                   ║
║                 debit sender  +  credit receiver  +  write ledger           ║
║                 @Version on Account  →  Optimistic Locking (depth-in-def)   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## ⚙️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Language** | Java 17 | `record` types, modern APIs, Spring Boot 3.x requirement |
| **Framework** | Spring Boot 3.3.5 | REST APIs, DI, JPA, Scheduling, Thymeleaf |
| **Web Layer** | Spring MVC + Embedded Tomcat | `@RestController`, `@RequestBody`, `@RequestHeader` |
| **Persistence** | Spring Data JPA + Hibernate | `@Entity`, `@Transactional`, `@Version` (optimistic lock) |
| **Database** | H2 In-Memory | Zero-setup demo; swap PostgreSQL for production |
| **Template Engine** | Thymeleaf | Server-side HTML demo dashboard at `/` |
| **Encryption** | Java Security API | RSA-2048/OAEP + AES-256-GCM + SHA-256 |
| **Concurrency** | `ConcurrentHashMap` + `parallelStream` | Thread-safe idempotency, parallel bridge simulation |
| **Scheduling** | `@Scheduled` | TTL-based cache eviction every 60 seconds |
| **Build** | Maven Wrapper (`mvnw`) | No Maven install needed |
| **Testing** | JUnit 5 + Spring Boot Test | Concurrency correctness tests |

---

## 🔐 Cryptographic Design

### Why Hybrid Encryption?

RSA-2048 can only directly encrypt ~245 bytes. A `PaymentInstruction` JSON easily exceeds that. The solution is the same pattern used by **TLS, PGP, and Signal**:

```
┌─────────────────────────────────────────────────────────┐
│              WIRE FORMAT (Base64-encoded)                │
│                                                         │
│  ┌───────────────┐ ┌──────┐ ┌───────────────────────┐  │
│  │  256 bytes    │ │  12  │ │  N bytes + 16B tag     │  │
│  │ RSA-encrypted │ │bytes │ │    AES-256-GCM         │  │
│  │   AES key     │ │  IV  │ │  encrypted payload     │  │
│  └───────────────┘ └──────┘ └───────────────────────┘  │
│       ▲                              ▲                   │
│  Only server's                 Fast + authenticated      │
│  private key can               encryption. 1-bit         │
│  unwrap this                   tamper → full reject      │
└─────────────────────────────────────────────────────────┘
```

```java
// HybridCryptoService.java — encrypt flow
byte[] iv = new byte[12];  rng.nextBytes(iv);
Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
aes.init(ENCRYPT_MODE, aesKey, new GCMParameterSpec(128, iv));
byte[] aesCiphertext = aes.doFinal(plaintext);          // authenticated!

Cipher rsa = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
rsa.init(ENCRYPT_MODE, serverPublicKey, oaepSpec);
byte[] encryptedAesKey = rsa.doFinal(aesKey.getEncoded());

// Pack: [RSA-enc AES key][IV][AES ciphertext + GCM tag]
```

---

## 🛡️ The Three Hard Problems — Solved

### Problem 1 — Untrusted Intermediaries

> A stranger's phone carries your transaction. How do they not read or alter it?

**Solution:** End-to-end hybrid encryption. Intermediaries hold an opaque blob. The GCM authentication tag makes any 1-bit tampering instantly detectable on the server.

### Problem 2 — The Duplicate Storm

> 3 bridge nodes hold the same packet. They all get 4G simultaneously and POST within milliseconds. Without protection → sender debited ₹1,500 instead of ₹500.

**Solution:** Atomic idempotency gate using `ConcurrentHashMap.putIfAbsent()` — the JVM equivalent of Redis `SETNX`.

```java
// IdempotencyService.java
public boolean claim(String packetHash) {
    Instant prev = seen.putIfAbsent(packetHash, Instant.now());
    return prev == null;  // exactly ONE thread returns true, ever
}
```

The key is `SHA-256(ciphertext)` — not `packetId` (which an intermediary could forge), but the ciphertext itself, which is authenticated and byte-identical across all copies of the same packet.

**Defense in depth:** `transactions.packet_hash` carries a `UNIQUE` index. If the cache ever fails, the database rejects the duplicate insert.

### Problem 3 — Replay Attacks

> An attacker captures a valid ciphertext and replays it days later.

**Solution:** Two-layer protection inside the encrypted payload:
- `signedAt` timestamp — server rejects packets older than 24 hours (attacker can't modify this without breaking the GCM tag)
- `nonce` (UUID) — every legitimate payment from Alice to Bob gets a unique nonce, so two real payments don't collide, but a *replay* of the same ciphertext is identical and caught by the idempotency cache

---

## 🗂️ Project Structure

```
upi-offline-mesh/
├── pom.xml                              Spring Boot 3.3.5 parent, Java 17, all deps
├── mvnw / mvnw.cmd                      Maven wrapper — no install needed
│
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties   H2 config, port 8080, idempotency TTLs
    │   │   └── templates/
    │   │       └── dashboard.html       Interactive demo UI (Thymeleaf)
    │   │
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java          ← Spring Boot entry point
    │       │
    │       ├── config/
    │       │   └── AppConfig.java               @EnableScheduling
    │       │
    │       ├── crypto/                          ── Cryptography Layer
    │       │   ├── ServerKeyHolder.java         RSA-2048 keypair, @PostConstruct
    │       │   └── HybridCryptoService.java     Encrypt / Decrypt / Hash
    │       │
    │       ├── model/                           ── Domain Layer
    │       │   ├── Account.java                 @Entity, @Version (optimistic lock)
    │       │   ├── AccountRepository.java       JpaRepository<Account, String>
    │       │   ├── Transaction.java             @Index unique on packetHash
    │       │   ├── TransactionRepository.java   JpaRepository<Transaction, Long>
    │       │   ├── MeshPacket.java              Wire format (opaque ciphertext)
    │       │   └── PaymentInstruction.java      Decrypted inner payload
    │       │
    │       ├── service/                         ── Business Logic Layer
    │       │   ├── DemoService.java             Seeds accounts, simulates sender phone
    │       │   ├── VirtualDevice.java           One simulated phone in the mesh
    │       │   ├── MeshSimulatorService.java    Gossip protocol across 5 virtual devices
    │       │   ├── IdempotencyService.java      ConcurrentHashMap ≡ Redis SETNX + TTL
    │       │   ├── BridgeIngestionService.java  THE pipeline: hash→claim→decrypt→settle
    │       │   └── SettlementService.java       @Transactional debit + credit + ledger
    │       │
    │       └── controller/                      ── HTTP Layer
    │           ├── ApiController.java           All REST endpoints
    │           └── DashboardController.java     Serves dashboard at /
    │
    └── test/
        └── java/com/demo/upimesh/
            └── IdempotencyConcurrencyTest.java  3-threads, 1-packet, 1-settlement proof
```

---

## 🚀 Quick Start

### Prerequisites

- **JDK 17+** installed (`java -version` to check). That's literally it — no Maven, no database, no Redis.

### Run (Windows)

```cmd
mvnw.cmd spring-boot:run
```

### Run (Mac / Linux)

```bash
./mvnw spring-boot:run
```

> First run downloads Maven + dependencies (~90 MB, ~2 min). All subsequent runs start in **under 5 seconds**.

### Open the Dashboard

```
http://localhost:8080
```

### Browse the live database

```
http://localhost:8080/h2-console
```

| Field | Value |
|---|---|
| JDBC URL | `jdbc:h2:mem:upimesh` |
| Username | `sa` |
| Password | *(leave blank)* |

---

## 🎬 Demo Walkthrough

The dashboard has four steps — click them in order:

```
┌──────────────────────────────────────────────────────────────┐
│  Step 1 ─ Inject into Mesh                                   │
│           Choose Alice → Bob, ₹500, PIN                      │
│           Server encrypts (hybrid) → phone-alice holds it    │
├──────────────────────────────────────────────────────────────┤
│  Step 2 ─ Run Gossip Rounds  (click 2×)                      │
│           All 5 virtual devices now hold the packet          │
│           TTL decrements each hop                            │
├──────────────────────────────────────────────────────────────┤
│  Step 3 ─ Bridges Upload                                     │
│           phone-bridge gets 4G, POSTs to /api/bridge/ingest  │
│           Watch balances change + ledger row appear          │
├──────────────────────────────────────────────────────────────┤
│  Step 4 ─ Run the Concurrency Test                           │
│           3 threads deliver the same packet simultaneously   │
│           Result: 1 SETTLED, 2 DUPLICATE_DROPPED             │
└──────────────────────────────────────────────────────────────┘
```

---

## 📡 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Interactive demo dashboard (Thymeleaf) |
| `GET` | `/api/server-key` | Server's RSA-2048 public key (Base64) |
| `GET` | `/api/accounts` | All account balances |
| `GET` | `/api/transactions` | Last 20 settled transactions |
| `GET` | `/api/mesh/state` | State of all 5 virtual devices + packet counts |
| `POST` | `/api/demo/send` | Simulate sender: encrypt + inject into mesh |
| `POST` | `/api/mesh/gossip` | One round of gossip (spread packets) |
| `POST` | `/api/mesh/flush` | Bridge nodes upload to backend (runs in parallel) |
| `POST` | `/api/mesh/reset` | Clear mesh + idempotency cache |
| `POST` | `/api/bridge/ingest` | **Production endpoint** — real bridge nodes POST here |
| `GET` | `/h2-console` | Live database browser |

### POST `/api/bridge/ingest` — Request & Response

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{
  "packetId":   "550e8400-e29b-41d4-a716-446655440000",
  "ttl":        2,
  "createdAt":  1730000000000,
  "ciphertext": "<base64-encoded RSA+AES blob>"
}
```

```json
// SETTLED
{ "outcome": "SETTLED", "packetHash": "a3f8c9...", "reason": null, "transactionId": 42 }

// Duplicate from second bridge
{ "outcome": "DUPLICATE_DROPPED", "packetHash": "a3f8c9...", "reason": null, "transactionId": null }

// Tampered or stale
{ "outcome": "INVALID", "packetHash": "a3f8c9...", "reason": "stale_packet", "transactionId": null }
```

---

## 🧪 Tests

```bash
# Run all tests
mvnw.cmd test

# Run only the headline concurrency test
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

| Test | What It Proves |
|---|---|
| `encryptDecryptRoundTrip` | Hybrid encryption is correctly symmetric |
| `tamperedCiphertextIsRejected` | A single bit-flip → `INVALID`, never `SETTLED` |
| `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` | **3 concurrent threads → exactly 1 SETTLED, 2 DUPLICATE_DROPPED, sender debited once** |

---

## 🏭 Demo → Production Gap

This is a portfolio demo. Here's what a production version looks like:

| Demo (this repo) | Production equivalent |
|---|---|
| H2 in-memory database | PostgreSQL / MySQL with read replicas |
| `ConcurrentHashMap` idempotency | **Redis `SET NX EX 86400`** (distributed across instances) |
| RSA keypair generated on startup | Private key in **HSM / AWS KMS / HashiCorp Vault** |
| `DemoService` simulates sender phone | Same crypto code ported to **Android / Kotlin** |
| `MeshSimulatorService` (software mesh) | Real **BLE GATT** or **Wi-Fi Direct** between devices |
| One service owns the ledger | Integration with **NPCI / bank core banking** |
| No auth on `/api/bridge/ingest` | **Mutual TLS** or signed bridge-node certificates |
| H2 console exposed | **Disabled** in production |
| No rate limiting | Per-bridge-node rate limit, per-sender velocity checks |
| Console logging | Structured logs to **SIEM**, alerts on `INVALID` spikes |

> The cryptographic design and idempotency logic are **production-shaped**. The infrastructure shell around them is what a real deployment would upgrade.

---

## ⚠️ Honest Limitations of the Concept

For any reviewer who looks deeper:

1. **No offline balance proof** — The receiver sees "₹500 sent" but this is an IOU. If Alice's account is empty when the packet reaches the backend, settlement is `REJECTED`. (Real-world fix: UPI Lite uses a pre-funded hardware-backed wallet that can cryptographically prove available funds offline.)

2. **Double-spend risk offline** — Alice with ₹500 could send to Bob in basement A, then send to Carol in basement B. Whichever packet arrives at the server first wins. Same root cause as #1.

3. **Android BLE constraints** — Background Bluetooth LE on Android 8+ is heavily throttled. iOS peripheral mode is restricted. Real-world mesh adoption would require OS-level or regulatory support.

4. **Privacy metadata** — Intermediaries can't read the payload, but they know *a payment packet exists* and *when*. A production deployment needs regulatory disclosure around this.

The honest framing for this project: **"mesh-routed deferred settlement"** — and the cryptography and idempotency engineering here is the real, demonstrable value.

---

## 🔧 Troubleshooting

| Error | Fix |
|---|---|
| `java: command not found` | Install JDK 17+. Windows: `winget install EclipseAdoptium.Temurin.17.JDK` |
| Port 8080 in use | Set `server.port=8081` in `application.properties` |
| `mvnw.cmd` not recognized (PowerShell) | Use `.\\mvnw.cmd spring-boot:run` |
| First run takes forever | Normal — downloading Maven + 90MB deps. Subsequent runs: ~5 sec |
| Concurrency test flakes | Run 3× — if consistently failing, share the full failure output |
| STS import fails | File → Import → Maven → Existing Maven Projects → select the folder with `pom.xml` |

---

## 👤 About This Project

Built to demonstrate real-world backend engineering concepts in a single runnable demo:

- **Hybrid cryptography** (RSA-OAEP + AES-256-GCM) — the same scheme as TLS
- **Concurrent idempotency** with atomic compare-and-set
- **Optimistic locking** via JPA `@Version`
- **Gossip protocol** simulation
- **ACID-compliant ledger settlement** with `@Transactional`
- **Replay attack mitigation** with timestamp + nonce

All of this runs on a single `java -jar` with zero external dependencies.

---

<div align="center">

**⭐ Star this repo if it helped you understand offline payment architecture.**

<br/>

<img src="https://img.shields.io/badge/Made_with-Spring_Boot_+_Java-6DB33F?style=for-the-badge&logo=spring&logoColor=white"/>
<img src="https://img.shields.io/badge/Concept-Mesh_Routed_Deferred_Settlement-00D4FF?style=for-the-badge"/>

</div>
