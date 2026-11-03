# FI Payments Ecosystem – Current State Architecture

> Based on the described scope. Systems named by you are used as stated; roles of IMM vs. SAP and the control/dashboard components are inferred and should be validated.

## 1. Logical architecture

```mermaid
flowchart LR
  subgraph PAYERS["Payers - 10M+ members"]
    MEM["Members (Web / Mobile / Contact Center)"]
    MBANK["Member Banks (RDFI)"]
    MCHK["Mailed Checks"]
  end

  subgraph POL["Policy / Billing Systems"]
    PC["P&C Policy System"]
    HAL["Life Co: HAL"]
    ALIP["Life Co: ALIP"]
    BEN["Life Co: Benekiva"]
  end

  subgraph IN["Premium Collection (Inbound)"]
    IMM["IMM - Internal Money Movement"]
    STRIPE["Stripe (Card)"]
    LBX["FI Lockbox"]
    BT["Banktec (check capture)"]
    DMI["DMI (check processing)"]
  end

  subgraph DISB["Disbursement Sources"]
    BCC["Bank CC Refunds"]
    ESC["Escheatment"]
    DDA["DD&A Account Closure Payments"]
    GW["Guidewire (P&C Claims)"]
    LCC["Life Co Claims (ALIP, HAL)"]
    ORA["Oracle (Corporate Invoice Approval)"]
  end

  subgraph INT["Integration Layer"]
    API["APIs"]
    KAFKA["Kafka Topics / Event-Driven Architecture"]
    IL["Integration / Transformation Layer"]
  end

  subgraph PROC["Payment Processing"]
    SAP["SAP - Payments Processor"]
    FRAUD["Actimize (Fraud) and LexisNexis (OFAC)"]
    ARIS["ARIS (Check Disbursement)"]
    VEND["3rd Party Check Print Vendor"]
  end

  subgraph BANK["Bank / Networks"]
    JPMC["JPMC (ODFI - ACH, Wires, Check Clearing)"]
    ACH["ACH Network"]
    WIRE["Fedwire / SWIFT"]
  end

  subgraph SHIP["Check Delivery"]
    CARR["FedEx / UPS / USPS"]
  end

  subgraph OBS["Run-to-Run Controls and Monitoring"]
    CTRL["Run-to-Run Controls (count / amount balancing at every hop)"]
    DASH["Real-Time Dashboard: Tracking, Failure Analysis, Queue Depth"]
  end

  GL["Finance: GL / Reconciliation / Reporting"]

  %% Premium inbound
  MEM -->|"Card"| STRIPE --> IMM
  MEM -->|"ACH"| IMM
  IMM -->|"ACH debit file"| JPMC --> ACH --> MBANK
  MBANK -.->|"Returns / NOC"| JPMC -.-> IMM
  MCHK --> LBX --> BT --> DMI -->|"Deposit"| JPMC
  IMM -->|"Payment posting"| IL
  DMI -->|"Payment posting"| IL
  IL --> PC
  IL --> HAL
  IL --> ALIP
  IL --> BEN

  %% Disbursement
  GW -->|"Claim payments"| API
  LCC -->|"Claim payments"| API
  BCC --> API
  ESC --> API
  DDA --> API
  API --> KAFKA --> IL --> SAP
  ORA -->|"Approved invoices"| SAP
  SAP <-->|"Screening"| FRAUD
  SAP -->|"ACH / Wire instructions"| JPMC
  JPMC --> ACH
  JPMC --> WIRE
  SAP -->|"Check requests"| ARIS --> VEND --> CARR --> MCHK
  JPMC -.->|"Bank statements / check clearing"| GL
  SAP -->|"Status / postings"| GL
  SAP -.->|"Payment status events"| KAFKA

  %% Controls
  CTRL -.-> IMM
  CTRL -.-> IL
  CTRL -.-> SAP
  CTRL -.-> ARIS
  CTRL -.-> JPMC
  CTRL --> DASH
  KAFKA -.->|"Queue depth / lag"| DASH
```

## 2. Flow summary

| Flow | Path |
|---|---|
| Premium - Card | Member → Stripe → IMM → Integration layer → P&C / HAL / ALIP / Benekiva |
| Premium - ACH | Member → IMM → JPMC (ODFI) → ACH network → member bank; returns back via JPMC |
| Premium - Check | Member → FI lockbox → Banktec → DMI → clear via JPMC → posting to policy systems |
| Disbursement - ACH / Wire | Source system → APIs / Kafka → Integration layer → SAP → Actimize / LexisNexis screening → JPMC |
| Corporate AP | Oracle approval → SAP → screening → JPMC (ACH / wire) or ARIS (check) |
| Disbursement - Check | SAP → ARIS → 3rd party print vendor → FedEx / UPS / USPS |
| Controls | Run-to-run balancing at every hop; events and queue metrics feed a real-time dashboard |
