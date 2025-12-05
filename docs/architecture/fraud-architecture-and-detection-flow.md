# Fraud Architecture & Detection Flow

Dokumentasi ini menjelaskan area di mana fraud terjadi, alur deteksi anti-fraud, dan real case yang terjadi pada sistem login, transaksi, dan abuse.

---

# 1. Area Terjadinya Fraud & Arsitektur Anti-Fraud

```mermaid
flowchart TD
  A["Client Layer
Login • Register • Transaksi"]
  B["API Gateway
(IP Fraud, BOT, Rate Abuse)"]
  C["Auth / Transaction API
(Abnormal Usage, Anomaly)"]
  D["Message Bus
Kafka • RMQ • NATS"]
  E["Fraud Engine
Rules • ML • Device FP • Graph DB • Velocity"]
  F["Decision Engine
Allow / Block / Challenge (OTP)"]
  G["Alert & Forensic System"]

  A -->|Fraud Login| B
  A -->|Fraud Transaksi| B
  B -->|PREVENT FRAUD| C
  C -->|DETECT FRAUD| D
  D -->|STREAM EVENT| E
  E -->|JUDGE FRAUD| F
  F --> G

```

---

# 2. Real Case – Credential Stuffing (Login Fraud)

Penyerang menggunakan 50.000 email/password bocor untuk mencoba login secara otomatis.

## Diagram (Login Fraud)

```mermaid
sequenceDiagram
    participant Attacker
    participant Client
    participant APIGW as API Gateway
    participant AuthAPI as Auth Service
    participant FraudE as Fraud Engine
    participant Decision as Decision Engine

    Attacker->>Client: Sends mass login requests (50,000)
    Client->>APIGW: Login attempt
    APIGW-->>APIGW: Rate limit & IP reputation check
    APIGW->>AuthAPI: Forward allowed attempts
    AuthAPI-->>AuthAPI: Detect failed login spikes
    AuthAPI->>FraudE: Send login anomaly event
    FraudE-->>FraudE: ML Score: High Risk
    FraudE->>Decision: High risk login
    Decision-->>APIGW: BLOCK login
    APIGW->>Client: Login rejected
```

### Dimana Fraud Terjadi:

* Client (login form)
* API Gateway (IP/bot spikes)
* Auth API (failed login abnormal)
* Fraud Engine (risk scoring)

---

# 3. Real Case – Account Takeover (Transaksi Tidak Wajar)

Akun pengguna dibajak, lalu penipu melakukan transfer besar dari device baru di luar negeri.

## Diagram (Transaction Fraud)

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant APIGW
    participant TxAPI as Transaction API
    participant FraudE as Fraud Engine
    participant Decision as Decision Engine

    User->>Client: Perform large transfer on new device
    Client->>APIGW: Transaction request
    APIGW-->>APIGW: Check IP (foreign), device mismatch
    APIGW->>TxAPI: Forward transaction
    TxAPI-->>TxAPI: Detect unusual amount
    TxAPI->>FraudE: Send transaction event
    FraudE-->>FraudE: Behavioral ML detects anomaly
    FraudE->>Decision: High risk transaction
    Decision-->>APIGW: CHALLENGE (OTP)
    APIGW->>Client: Request OTP verification
```

### Dimana Fraud Terjadi:

* Client → transaksi tidak wajar
* Gateway → IP luar negeri
* Transaction API → jumlah abnormal
* Fraud Engine → ML anomaly detection

---

# 4. Real Case – Bonus Abuse / Multi-Account Fraud

Pelaku membuat puluhan akun palsu untuk mendapatkan welcome bonus berkali-kali.

## Diagram (Abuse)

```mermaid
flowchart LR
    A[User creates many accounts<br/>Same device/IP] --> B[API Gateway]
    B -->|Device/IP Duplication| C[User Service]
    C -->|Suspicious profile pattern| D[Message Bus]
    D --> E[Fraud Engine<br/>Graph DB detects relationships]
    E --> F[Decision Engine<br/>Block Signup]
```

### Dimana Fraud Terjadi:

* Client → pendaftaran akun banyak
* Gateway → device/ip duplication
* User API → profil duplikasi
* Fraud Engine → Graph DB (fraud ring)

---

# 5. Ringkasan Fraud & Solusi

| Real Fraud Case         | Dimana Terjadi           | Solusi Anti-Fraud                          |
| ----------------------- | ------------------------ | ------------------------------------------ |
| **Credential stuffing** | Login, Gateway, Auth API | Rate limit, IP scoring, MFA, bot detection |
| **Account takeover**    | Login + Transaksi        | Device fingerprint, GeoIP, OTP             |
| **Bonus abuse**         | Register                 | Graph DB, device duplication check         |
| **Payment fraud**       | Payment API              | Card BIN check, 3DS, rules engine          |

---