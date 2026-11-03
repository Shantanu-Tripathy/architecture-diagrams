# FI Payments Ecosystem – Target State Architecture

> IMM (legacy premium rail) is retired. SAP becomes the single payments processor for premium collections and disbursements, behind the common integration layer. Placement of Stripe and the lockbox flows under SAP is inferred; validate.

```mermaid
flowchart LR
  subgraph PAYERS["Payers - 10M+ members"]
    MEM["Members (Web / Mobile / Contact Center)"]
    MBANK["Member Banks (RDFI)"]
    MCHK["Mailed Checks"]
  end

  subgraph POL["Policy Systems"]
    PC["P&C Policy System"]
    HAL["HAL"]
    ALIP["ALIP"]
    BEN["Benekiva"]
  end

  subgraph SRC["Disbursement Sources"]
    BCC["Bank CC Refunds"]
    ESC["Escheatment"]
    DDA["DD&A Account Closure"]
    GW["Guidewire (P&C Claims)"]
    LCC["Life Co Claims (ALIP, HAL)"]
    ORA["Oracle (Invoice Approval)"]
  end

  subgraph INT["Integration Layer"]
    API["APIs"]
    KAFKA["Kafka / Event-Driven"]
    IL["Integration / Transformation"]
  end

  SAP["SAP - Unified Payments Processor (Premium and Disbursements)"]
  FRAUD["Actimize (Fraud) and LexisNexis (OFAC)"]
  STRIPE["Stripe (Card)"]

  subgraph CHK["Check Handling"]
    LBX["FI Lockbox"]
    BT["Banktec"]
    DMI["DMI"]
    ARIS["ARIS (Check Disbursement)"]
    VEND["3rd Party Print Vendor"]
    CARR["FedEx / UPS / USPS"]
  end

  subgraph BANK["Bank / Networks"]
    JPMC["JPMC (ODFI - ACH, Wires, Check Clearing)"]
    ACH["ACH Network"]
    WIRE["Fedwire / SWIFT"]
  end

  subgraph OBS["Controls and Monitoring"]
    CTRL["Run-to-Run Controls at every hop"]
    DASH["Real-Time Dashboard: Tracking, Failures, Queue Depth"]
  end

  GL["Finance: GL / Reconciliation / Reporting"]

  %% Premium
  MEM -->|"Card"| STRIPE --> API
  MEM -->|"ACH authorization"| API
  PC & HAL & ALIP & BEN -->|"Premium due / billing"| API
  MCHK --> LBX --> BT --> DMI -->|"Check remittance"| IL
  DMI -->|"Deposit"| JPMC
  API --> KAFKA --> IL --> SAP
  SAP -->|"ACH debit"| JPMC
  JPMC --> ACH --> MBANK
  MBANK -.->|"Returns / NOC"| JPMC -.-> SAP
  SAP -.->|"Payment posted events"| KAFKA
  IL -->|"Posting"| PC & HAL & ALIP & BEN

  %% Disbursements
  GW & LCC & BCC & ESC & DDA --> API
  ORA -->|"Approved invoices"| SAP
  SAP <-->|"Screening"| FRAUD
  SAP -->|"ACH / Wire"| JPMC
  JPMC --> WIRE
  SAP -->|"Check requests"| ARIS --> VEND --> CARR --> MCHK

  %% Finance and controls
  SAP --> GL
  JPMC -.->|"Statements / clearing"| GL
  CTRL -.-> IL
  CTRL -.-> SAP
  CTRL -.-> ARIS
  CTRL -.-> JPMC
  CTRL --> DASH
  KAFKA -.->|"Queue depth / lag"| DASH
```

## Current to target changes

| Area | Current | Target |
|---|---|---|
| Premium processing | IMM | SAP via integration layer |
| Disbursements | SAP | SAP (unchanged) |
| Card | Stripe → IMM | Stripe → integration layer → SAP |
| ACH premium | IMM → JPMC | SAP → JPMC |
| Screening | Disbursements only | Applied to all SAP payments (assumed) |
| Controls / dashboard | Per hop incl. IMM | Per hop; IMM removed |
