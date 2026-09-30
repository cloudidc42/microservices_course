# Part 93: Case Study: Payment System

## บทนำ

ระบบชำระเงินเป็นหัวใจสำคัญของทุก E-Commerce และ FinTech ในไทย บทนี้จะออกแบบระบบชำระเงินที่คล้ายกับ PromptPay, SCB Easy, หรือ KBank KPlus โดยครอบคลุม PCI-DSS, Transaction Processing, Reconciliation และ Fraud Detection

---

## 1. Thai Payment Ecosystem Overview

```
ระบบชำระเงินในไทย:
┌─────────────────────────────────────────────────────────────────┐
│                   Thailand Payment Landscape                      │
│                                                                   │
│  PromptPay (NITMX)  ─────────────────────────────────────────  │
│  QR Code Payment    ─── ระบบกลาง National ───────────────────  │
│                                                                   │
│  Banks:              PSPs:                Mobile Wallets:         │
│  ┌──────────────┐   ┌──────────────┐    ┌───────────────────┐  │
│  │ SCB (Easy)   │   │ Omise        │    │ TrueMoney Wallet  │  │
│  │ KBank (KPlus)│   │ 2C2P         │    │ Rabbit LINE Pay   │  │
│  │ BBL          │   │ Stripe TH    │    │ WeChat Pay TH     │  │
│  │ Krungthai    │   │ PaySolutions │    │ GrabPay           │  │
│  └──────────────┘   └──────────────┘    └───────────────────┘  │
│                                                                   │
│  Crypto (emerging):                                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Bitkub, Satang Pro, Zipmex                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Payment Platform                             │
│                                                                    │
│  ┌─────────────┐  ┌─────────────────────────────────────────┐   │
│  │   Merchant  │  │              API Gateway                 │   │
│  │    App      │──│  - mTLS Authentication                   │   │
│  └─────────────┘  │  - API Key Validation                    │   │
│                   │  - Rate Limiting (10 TPS per merchant)    │   │
│  ┌─────────────┐  │  - Request Logging (Audit)               │   │
│  │  Mobile App │──│  - IP Allowlist                          │   │
│  └─────────────┘  └──────────────────┬──────────────────────┘   │
│                                       │                           │
│  ┌────────────────────────────────────▼──────────────────────┐  │
│  │                    Core Payment Services                    │  │
│  │                                                              │  │
│  │  ┌───────────────┐  ┌────────────────┐  ┌──────────────┐  │  │
│  │  │ Payment       │  │ Transaction    │  │  Wallet      │  │  │
│  │  │ Orchestrator  │  │ Service        │  │  Service     │  │  │
│  │  └───────┬───────┘  └───────┬────────┘  └──────┬───────┘  │  │
│  │          │                  │                   │           │  │
│  │  ┌───────▼───────┐  ┌───────▼────────┐  ┌──────▼───────┐  │  │
│  │  │ Fraud         │  │ Reconciliation │  │  Notification│  │  │
│  │  │ Detection     │  │ Service        │  │  Service     │  │  │
│  │  └───────────────┘  └────────────────┘  └──────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  External Integrations                      │  │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │  │
│  │  │ PromptPay│  │SCB/KBank │  │  Omise   │  │TrueMoney  │  │  │
│  │  │  (NITMX) │  │  Direct  │  │   2C2P   │  │           │  │  │
│  │  └─────────┘  └──────────┘  └──────────┘  └───────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Database Architecture

```
Database Design สำหรับ Payment System:

┌─────────────────────────────────────────────────────────────┐
│                   Payment Database (PostgreSQL)              │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  IMPORTANT: ข้อมูลทางการเงินต้องเป็น DECIMAL ไม่ใช่ FLOAT  │
│  │  และ ทุก column sensitive ต้องเข้ารหัส (pgcrypto)      │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  Tables:                                                      │
│  - transactions (partitioned by month)                       │
│  - payment_methods (encrypted card data)                     │
│  - wallets                                                    │
│  - ledger_entries (double-entry bookkeeping)                 │
│  - reconciliation_reports                                    │
│  - fraud_events                                              │
│  - audit_logs (append-only)                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. PCI-DSS Considerations

### 3.1 PCI-DSS Requirements Overview

```
PCI-DSS Requirement Categories:

Requirement 1-2: Network Security
├── Firewall configuration
├── No default vendor passwords  
└── Protect cardholder data

Requirement 3-4: Protect Stored Data
├── Protect stored cardholder data
└── Encrypt transmission of cardholder data

Requirement 5-6: Vulnerability Management
├── Anti-virus software
└── Develop/maintain secure systems

Requirement 7-9: Access Control
├── Restrict access to cardholder data
├── Identify and authenticate access
└── Restrict physical access

Requirement 10-11: Monitor & Test
├── Track and monitor all access
└── Regularly test security systems

Requirement 12: Information Security Policy
└── Maintain information security policy
```

### 3.2 Tokenization Implementation

```go
// payment-service/internal/tokenization/tokenizer.go
package tokenization

import (
    "context"
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "io"
    
    vault "github.com/hashicorp/vault/api"
)

// Card Tokenization - ไม่เก็บ PAN (Primary Account Number) ตรงๆ
// แทนที่ด้วย Token ที่ไม่มีค่าถ้าถูก leak

type CardTokenizer struct {
    vaultClient *vault.Client
    db          TokenRepository
}

type CardData struct {
    PAN         string // 4111111111111111
    ExpiryMonth int
    ExpiryYear  int
    CVV         string // ห้ามเก็บหลัง Authorization!
    HolderName  string
}

type CardToken struct {
    Token       string // tok_xxxxxxxxxxxxxxxx
    Last4       string // 1111
    Brand       string // Visa, Mastercard
    ExpiryMonth int
    ExpiryYear  int
    HolderName  string
}

func (t *CardTokenizer) Tokenize(ctx context.Context, card CardData) (*CardToken, error) {
    // Validate card number (Luhn algorithm)
    if !luhn.Validate(card.PAN) {
        return nil, ErrInvalidCardNumber
    }
    
    // Detect card brand
    brand := detectBrand(card.PAN)
    last4 := card.PAN[len(card.PAN)-4:]
    
    // Encrypt PAN using Vault Transit (Hardware Security Module)
    encrypted, err := t.vaultClient.Logical().Write(
        "transit/encrypt/payment-key",
        map[string]interface{}{
            "plaintext": base64.StdEncoding.EncodeToString([]byte(card.PAN)),
        },
    )
    if err != nil {
        return nil, fmt.Errorf("encryption failed: %w", err)
    }
    
    // Generate non-sensitive token
    tokenID := generateToken() // tok_xxx
    
    // Store token → encrypted PAN mapping (in HSM-backed storage)
    if err := t.db.StoreToken(ctx, TokenRecord{
        Token:          tokenID,
        EncryptedPAN:   encrypted.Data["ciphertext"].(string),
        Last4:          last4,
        Brand:          brand,
        ExpiryMonth:    card.ExpiryMonth,
        ExpiryYear:     card.ExpiryYear,
        HolderName:     card.HolderName,
        // CVV ห้ามเก็บ!!
    }); err != nil {
        return nil, err
    }
    
    return &CardToken{
        Token:       tokenID,
        Last4:       last4,
        Brand:       brand,
        ExpiryMonth: card.ExpiryMonth,
        ExpiryYear:  card.ExpiryYear,
        HolderName:  card.HolderName,
    }, nil
}

// Detokenize สำหรับส่งไป Payment Processor เท่านั้น
func (t *CardTokenizer) Detokenize(ctx context.Context, token string) (*CardData, error) {
    // Audit log: ทุกครั้งที่ Detokenize ต้อง Log
    auditLog(ctx, "DETOKENIZE", token)
    
    record, err := t.db.GetToken(ctx, token)
    if err != nil {
        return nil, err
    }
    
    // Decrypt from Vault
    decrypted, err := t.vaultClient.Logical().Write(
        "transit/decrypt/payment-key",
        map[string]interface{}{
            "ciphertext": record.EncryptedPAN,
        },
    )
    if err != nil {
        return nil, err
    }
    
    panBytes, _ := base64.StdEncoding.DecodeString(decrypted.Data["plaintext"].(string))
    
    return &CardData{
        PAN:         string(panBytes),
        ExpiryMonth: record.ExpiryMonth,
        ExpiryYear:  record.ExpiryYear,
        HolderName:  record.HolderName,
    }, nil
}
```

### 3.3 Audit Logging

```go
// payment-service/internal/audit/logger.go
package audit

import (
    "context"
    "time"
)

// Audit Log ต้องเป็น Append-only (ห้าม Update/Delete)
type AuditLog struct {
    ID          string    `db:"id"`
    UserID      string    `db:"user_id"`
    Action      string    `db:"action"`      // PAYMENT_INITIATED, CARD_TOKENIZED
    ResourceType string   `db:"resource_type"` // transaction, card_token
    ResourceID  string    `db:"resource_id"`
    IPAddress   string    `db:"ip_address"`
    UserAgent   string    `db:"user_agent"`
    Success     bool      `db:"success"`
    Metadata    JSONB     `db:"metadata"`
    CreatedAt   time.Time `db:"created_at"`
}

// SQL: audit_logs table ใช้ INSERT-only policy
// CREATE RULE no_update AS ON UPDATE TO audit_logs DO INSTEAD NOTHING;
// CREATE RULE no_delete AS ON DELETE TO audit_logs DO INSTEAD NOTHING;

type AuditLogger struct {
    db DB
}

func (l *AuditLogger) Log(ctx context.Context, entry AuditLog) error {
    entry.ID = generateUUID()
    entry.CreatedAt = time.Now().UTC()
    entry.IPAddress = getIPFromContext(ctx)
    entry.UserAgent = getUserAgentFromContext(ctx)
    
    _, err := l.db.ExecContext(ctx, `
        INSERT INTO audit_logs 
        (id, user_id, action, resource_type, resource_id, ip_address, user_agent, success, metadata, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
    `, entry.ID, entry.UserID, entry.Action, entry.ResourceType,
        entry.ResourceID, entry.IPAddress, entry.UserAgent,
        entry.Success, entry.Metadata, entry.CreatedAt)
    
    return err
}
```

---

## 4. Transaction Processing

### 4.1 Transaction State Machine

```
Transaction States:
┌──────────────────────────────────────────────────────────────────┐
│                     Transaction FSM                               │
│                                                                    │
│  INITIATED ──────> PROCESSING ──────> COMPLETED                  │
│      │                  │                                         │
│      │                  ├──────────> FAILED                       │
│      │                  │                                         │
│      │                  └──────────> PENDING_3DS                  │
│      │                                    │                        │
│      │                              AUTHORIZED                     │
│      │                                    │                        │
│      └──> CANCELLED              CAPTURED (final for cards)        │
│                                           │                        │
│                                    REFUND_REQUESTED                │
│                                           │                        │
│                                    REFUNDED                        │
└──────────────────────────────────────────┘
```

### 4.2 PromptPay Integration

```go
// payment-service/internal/providers/promptpay/client.go
package promptpay

import (
    "context"
    "crypto/tls"
    "encoding/xml"
    "fmt"
    "time"
)

// PromptPay ผ่าน NITMX/ITMX Standards
// อ้างอิง: Bank of Thailand Payment Standard

type PromptPayClient struct {
    bankCode    string
    merchantID  string
    privateKey  []byte
    certificate []byte
    baseURL     string
    httpClient  *http.Client
}

type QRPaymentRequest struct {
    MerchantID  string
    Amount      decimal.Decimal
    Currency    string // THB
    Reference   string // Order ID
    Description string
    ExpiresAt   time.Time
}

type QRPaymentResponse struct {
    QRString    string  // EMVCo QR string
    QRImageURL  string  // QR image URL
    TransactionID string
    ExpiresAt   time.Time
}

func (c *PromptPayClient) GenerateQR(ctx context.Context, req QRPaymentRequest) (*QRPaymentResponse, error) {
    // สร้าง EMVCo QR Code ตาม Bank of Thailand Standard
    qrData := buildEMVCoQR(
        c.merchantID,
        req.Amount,
        req.Currency,
        req.Reference,
    )
    
    // Sign with merchant certificate
    signature, err := c.signData(qrData)
    if err != nil {
        return nil, fmt.Errorf("sign QR data: %w", err)
    }
    
    // Call NITMX API
    resp, err := c.callNITMX(ctx, "/qr/generate", map[string]interface{}{
        "merchant_id":   c.merchantID,
        "qr_data":       qrData,
        "signature":     signature,
        "amount":        req.Amount.String(),
        "currency":      req.Currency,
        "reference":     req.Reference,
        "expires_at":    req.ExpiresAt.Unix(),
    })
    
    return &QRPaymentResponse{
        QRString:      resp.QRString,
        QRImageURL:    resp.QRImageURL,
        TransactionID: resp.TransactionID,
        ExpiresAt:     req.ExpiresAt,
    }, nil
}

// Webhook รับการแจ้งเตือนเมื่อมีการชำระเงิน
func (c *PromptPayClient) HandleWebhook(ctx context.Context, payload []byte, signature string) (*PaymentNotification, error) {
    // Verify signature
    if !c.verifySignature(payload, signature) {
        return nil, ErrInvalidSignature
    }
    
    var notification PaymentNotification
    if err := json.Unmarshal(payload, &notification); err != nil {
        return nil, err
    }
    
    // Idempotency check
    if processed, _ := c.isProcessed(ctx, notification.TransactionID); processed {
        return &notification, nil // Already processed
    }
    
    return &notification, nil
}

// EMVCo QR Code Builder (ตาม Thailand QR Standard)
func buildEMVCoQR(merchantID string, amount decimal.Decimal, currency, reference string) string {
    // Format: ID(length)Value
    // 00: Payload Format Indicator = 01
    // 01: Point of Initiation Method = 12 (Dynamic QR)
    // 29: Merchant Account Information (PromptPay)
    //     00: Globally Unique ID
    //     01: Merchant ID (Thai National ID/Phone/Tax ID)
    // 52: Merchant Category Code
    // 53: Transaction Currency (764 = THB)
    // 54: Transaction Amount
    // 58: Country Code = TH
    // 59: Merchant Name
    // 60: Merchant City
    // 62: Additional Data
    //     05: Reference Label
    // 63: CRC
    
    data := fmt.Sprintf("000201" +   // Payload Format
        "010212" +                    // Dynamic QR
        "2937" +                      // Merchant Account (PromptPay)
        "0016A000000677010111" +      // PromptPay AID
        "011300%s" +                  // Merchant ID (13 digits)
        "5204%s" +                    // MCC
        "5303764" +                   // THB
        "54%02d%s" +                  // Amount
        "5802TH" +                    // Country
        "...",
        merchantID,
        "5812",                       // General Merchandise
        len(amount.String()), amount.String(),
    )
    
    // Append CRC16
    crc := calculateCRC16(data + "6304")
    return data + fmt.Sprintf("6304%04X", crc)
}
```

### 4.3 Credit Card Processing (3DS 2.0)

```go
// payment-service/internal/providers/card/processor.go
package card

import (
    "context"
    "time"
)

type CardProcessor struct {
    omise    *OmiseClient
    tokenizer *Tokenizer
    db       TransactionRepository
}

type ChargeRequest struct {
    CardToken   string
    Amount      decimal.Decimal
    Currency    string
    OrderID     string
    CustomerID  string
    Description string
    ReturnURL   string // สำหรับ 3DS redirect
}

func (p *CardProcessor) Charge(ctx context.Context, req ChargeRequest) (*ChargeResponse, error) {
    // 1. Fraud check ก่อน charge
    fraudScore, err := p.fraudService.Score(ctx, FraudCheckRequest{
        CustomerID: req.CustomerID,
        Amount:     req.Amount,
        CardToken:  req.CardToken,
    })
    if err == nil && fraudScore.Score > 80 {
        return nil, ErrHighFraudRisk
    }
    
    // 2. Detokenize (เฉพาะส่วนที่จำเป็น)
    card, err := p.tokenizer.Detokenize(ctx, req.CardToken)
    if err != nil {
        return nil, fmt.Errorf("detokenize failed: %w", err)
    }
    
    // 3. Create transaction record
    txn := &Transaction{
        ID:          generateTxnID(),
        OrderID:     req.OrderID,
        CustomerID:  req.CustomerID,
        Amount:      req.Amount,
        Currency:    req.Currency,
        Status:      TxnStatusProcessing,
        CreatedAt:   time.Now(),
    }
    
    if err := p.db.Create(ctx, txn); err != nil {
        return nil, err
    }
    
    // 4. Call Payment Processor (Omise)
    chargeResult, err := p.omise.CreateCharge(ctx, &omise.ChargeParams{
        Amount:      req.Amount.Mul(decimal.NewFromInt(100)).IntPart(), // Satang
        Currency:    "THB",
        Card:        card.PANToken, // Omise token
        Description: req.Description,
        ReturnURI:   req.ReturnURL,
        Metadata:    map[string]interface{}{"order_id": req.OrderID},
        ThreeDSVersion: "2",
    })
    
    if err != nil {
        p.db.UpdateStatus(ctx, txn.ID, TxnStatusFailed, err.Error())
        return nil, err
    }
    
    // 5. Handle 3DS if required
    if chargeResult.Status == "pending" && chargeResult.AuthorizeURI != "" {
        p.db.UpdateStatus(ctx, txn.ID, TxnStatusPending3DS, "")
        return &ChargeResponse{
            Status:       "requires_action",
            RedirectURL:  chargeResult.AuthorizeURI,
            TransactionID: txn.ID,
        }, nil
    }
    
    // 6. Update transaction
    status := TxnStatusCompleted
    if chargeResult.Status == "failed" {
        status = TxnStatusFailed
    }
    p.db.UpdateStatus(ctx, txn.ID, status, chargeResult.FailureMessage)
    
    return &ChargeResponse{
        Status:        string(status),
        TransactionID: txn.ID,
        ChargeID:      chargeResult.ID,
    }, nil
}
```

---

## 5. Reconciliation System

### 5.1 Reconciliation Architecture

```
Reconciliation คือ: การตรวจสอบว่า Transaction ของเรา
ตรงกับที่ Bank/PSP บันทึกไว้หรือไม่

Daily Reconciliation Flow:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  02:00 AM: Download Settlement Files from Banks/PSPs         │
│     ↓                                                         │
│  02:30 AM: Parse and Normalize Data                          │
│     ↓                                                         │
│  03:00 AM: Match against Internal Transactions               │
│     ↓                                                         │
│  03:30 AM: Generate Discrepancy Report                       │
│     ↓                                                         │
│  04:00 AM: Alert Finance Team for manual review              │
│     ↓                                                         │
│  09:00 AM: Finance team resolves discrepancies               │
└──────────────────────────────────────────────────────────────┘

Types of Discrepancies:
1. Missing in Bank (ระบบเราบันทึก แต่ Bank ไม่มี)
2. Missing in Ours (Bank มี แต่ระบบเราไม่มี)
3. Amount Mismatch (จำนวนไม่ตรงกัน)
4. Status Mismatch (สถานะไม่ตรงกัน)
```

### 5.2 Double-Entry Bookkeeping

```go
// payment-service/internal/ledger/ledger.go
package ledger

import (
    "context"
    "database/sql"
    "time"
    
    "github.com/shopspring/decimal"
)

// Double-Entry Bookkeeping
// ทุก Transaction ต้องมี Debit = Credit

type LedgerEntry struct {
    ID            string          `db:"id"`
    TransactionID string          `db:"transaction_id"`
    AccountID     string          `db:"account_id"`
    EntryType     string          `db:"entry_type"` // DEBIT or CREDIT
    Amount        decimal.Decimal `db:"amount"`
    Currency      string          `db:"currency"`
    Description   string          `db:"description"`
    BalanceBefore decimal.Decimal `db:"balance_before"`
    BalanceAfter  decimal.Decimal `db:"balance_after"`
    CreatedAt     time.Time       `db:"created_at"`
}

// Accounts:
// CUSTOMER_WALLET_XXXX  (Asset)
// MERCHANT_WALLET_XXXX  (Liability to merchant)
// PLATFORM_FEE          (Revenue)
// PAYMENT_GATEWAY_FEE   (Expense)
// CLEARING_ACCOUNT      (Transitional)

type LedgerService struct {
    db *sql.DB
}

// Customer pays merchant: ตัวอย่าง 1,000 THB, fee 2%
func (l *LedgerService) RecordPayment(ctx context.Context, params PaymentParams) error {
    tx, err := l.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
    if err != nil {
        return err
    }
    defer tx.Rollback()

    amount := params.Amount              // 1,000
    fee := amount.Mul(decimal.NewFromFloat(0.02)) // 20
    merchantAmount := amount.Sub(fee)    // 980

    entries := []LedgerEntry{
        // Customer pays
        {
            AccountID:   "CUSTOMER_WALLET_" + params.CustomerID,
            EntryType:   "DEBIT",
            Amount:      amount,
            Description: "Payment for order " + params.OrderID,
        },
        // Goes to clearing
        {
            AccountID:   "CLEARING_ACCOUNT",
            EntryType:   "CREDIT",
            Amount:      amount,
            Description: "Payment clearing",
        },
        // Clearing to merchant
        {
            AccountID:   "CLEARING_ACCOUNT",
            EntryType:   "DEBIT",
            Amount:      merchantAmount,
            Description: "Settlement to merchant",
        },
        {
            AccountID:   "MERCHANT_WALLET_" + params.MerchantID,
            EntryType:   "CREDIT",
            Amount:      merchantAmount,
            Description: "Payment received for order " + params.OrderID,
        },
        // Platform fee
        {
            AccountID:   "CLEARING_ACCOUNT",
            EntryType:   "DEBIT",
            Amount:      fee,
            Description: "Platform fee",
        },
        {
            AccountID:   "PLATFORM_FEE_REVENUE",
            EntryType:   "CREDIT",
            Amount:      fee,
            Description: "Platform fee from order " + params.OrderID,
        },
    }

    for _, entry := range entries {
        entry.ID = generateUUID()
        entry.TransactionID = params.TransactionID
        entry.Currency = params.Currency
        entry.CreatedAt = time.Now()
        
        // Update running balance atomically
        var currentBalance decimal.Decimal
        err := tx.QueryRowContext(ctx, `
            SELECT COALESCE(balance, 0) FROM account_balances 
            WHERE account_id = $1 FOR UPDATE
        `, entry.AccountID).Scan(&currentBalance)
        
        if entry.EntryType == "DEBIT" {
            entry.BalanceBefore = currentBalance
            entry.BalanceAfter = currentBalance.Sub(entry.Amount)
        } else {
            entry.BalanceBefore = currentBalance
            entry.BalanceAfter = currentBalance.Add(entry.Amount)
        }
        
        // Insert ledger entry
        _, err = tx.ExecContext(ctx, `
            INSERT INTO ledger_entries 
            (id, transaction_id, account_id, entry_type, amount, currency, 
             description, balance_before, balance_after, created_at)
            VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10)
        `, entry.ID, entry.TransactionID, entry.AccountID, entry.EntryType,
            entry.Amount, entry.Currency, entry.Description,
            entry.BalanceBefore, entry.BalanceAfter, entry.CreatedAt)
        
        // Update balance
        _, err = tx.ExecContext(ctx, `
            INSERT INTO account_balances (account_id, balance) VALUES ($1, $2)
            ON CONFLICT (account_id) DO UPDATE SET balance = $2
        `, entry.AccountID, entry.BalanceAfter)
    }

    return tx.Commit()
}
```

### 5.3 Reconciliation Service

```go
// reconciliation-service/internal/reconciler/reconciler.go
package reconciler

import (
    "context"
    "time"
)

type ReconciliationService struct {
    internalDB  TransactionRepository
    bankClients map[string]BankClient
    reporter    ReportService
    notifier    NotificationService
}

func (r *ReconciliationService) RunDailyReconciliation(ctx context.Context, date time.Time) error {
    report := &ReconciliationReport{
        Date:      date,
        StartedAt: time.Now(),
    }

    // Get all payment methods
    paymentMethods := []string{"omise", "promptpay", "truemoney", "kbank_direct"}
    
    for _, method := range paymentMethods {
        result, err := r.reconcileMethod(ctx, date, method)
        if err != nil {
            report.Errors = append(report.Errors, ReconciliationError{
                Method: method,
                Error:  err.Error(),
            })
            continue
        }
        report.Results = append(report.Results, *result)
    }

    report.FinishedAt = time.Now()
    report.Status = "completed"
    
    // Save report
    if err := r.reporter.Save(ctx, report); err != nil {
        return err
    }
    
    // Alert if discrepancies found
    totalDiscrepancies := 0
    for _, result := range report.Results {
        totalDiscrepancies += len(result.Discrepancies)
    }
    
    if totalDiscrepancies > 0 {
        r.notifier.AlertFinanceTeam(ctx, AlertRequest{
            Subject: fmt.Sprintf("Reconciliation: %d discrepancies found for %s", 
                totalDiscrepancies, date.Format("2006-01-02")),
            ReportURL: fmt.Sprintf("/reports/reconciliation/%s", report.ID),
        })
    }

    return nil
}

func (r *ReconciliationService) reconcileMethod(ctx context.Context, date time.Time, method string) (*ReconciliationResult, error) {
    // Step 1: Get internal transactions
    internalTxns, err := r.internalDB.GetByDateAndMethod(ctx, date, method)
    if err != nil {
        return nil, err
    }
    
    // Step 2: Get bank/PSP settlement file
    client := r.bankClients[method]
    bankTxns, err := client.GetSettlement(ctx, date)
    if err != nil {
        return nil, err
    }
    
    // Step 3: Match
    internalMap := make(map[string]*Transaction)
    for _, txn := range internalTxns {
        internalMap[txn.ExternalRef] = txn
    }
    
    bankMap := make(map[string]*BankTransaction)
    for _, txn := range bankTxns {
        bankMap[txn.Reference] = txn
    }
    
    result := &ReconciliationResult{
        Method:    method,
        Date:      date,
        Internal:  len(internalTxns),
        Bank:      len(bankTxns),
    }
    
    // Find discrepancies
    for ref, internal := range internalMap {
        bank, exists := bankMap[ref]
        if !exists {
            result.Discrepancies = append(result.Discrepancies, Discrepancy{
                Type:      "MISSING_IN_BANK",
                Reference: ref,
                InternalAmount: internal.Amount,
            })
            continue
        }
        
        if !internal.Amount.Equal(bank.Amount) {
            result.Discrepancies = append(result.Discrepancies, Discrepancy{
                Type:           "AMOUNT_MISMATCH",
                Reference:      ref,
                InternalAmount: internal.Amount,
                BankAmount:     bank.Amount,
                Difference:     internal.Amount.Sub(bank.Amount),
            })
        }
    }
    
    for ref := range bankMap {
        if _, exists := internalMap[ref]; !exists {
            result.Discrepancies = append(result.Discrepancies, Discrepancy{
                Type:      "MISSING_IN_INTERNAL",
                Reference: ref,
                BankAmount: bankMap[ref].Amount,
            })
        }
    }
    
    return result, nil
}
```

---

## 6. Fraud Detection System

### 6.1 Rule-Based + ML Fraud Detection

```python
# fraud-detection/app/services/fraud_scorer.py
from dataclasses import dataclass
from typing import List, Dict
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
import redis
import json

@dataclass
class FraudCheckRequest:
    transaction_id: str
    customer_id: str
    amount: float
    currency: str
    card_token: str
    ip_address: str
    device_fingerprint: str
    merchant_id: str
    timestamp: float

@dataclass  
class FraudScore:
    score: float          # 0-100
    decision: str         # APPROVE, REVIEW, REJECT
    reasons: List[str]
    rules_triggered: List[str]

class FraudDetectionService:
    def __init__(self, redis_client: redis.Redis, model_path: str):
        self.redis = redis_client
        self.model = self._load_model(model_path)
        
        # Rule thresholds
        self.REVIEW_THRESHOLD = 60
        self.REJECT_THRESHOLD = 80
        
    def score(self, req: FraudCheckRequest) -> FraudScore:
        rules_triggered = []
        base_score = 0
        
        # Rule 1: Velocity check - ธุรกรรมมากเกินไปใน 1 ชั่วโมง
        hourly_count = self._get_hourly_count(req.customer_id)
        if hourly_count > 20:
            base_score += 30
            rules_triggered.append(f"HIGH_VELOCITY: {hourly_count} txns/hour")
        elif hourly_count > 10:
            base_score += 15
            rules_triggered.append(f"MEDIUM_VELOCITY: {hourly_count} txns/hour")
        
        # Rule 2: Amount anomaly
        avg_amount = self._get_customer_avg_amount(req.customer_id)
        if avg_amount > 0 and req.amount > avg_amount * 5:
            base_score += 25
            rules_triggered.append(f"UNUSUAL_AMOUNT: {req.amount} vs avg {avg_amount}")
        
        # Rule 3: IP reputation
        ip_risk = self._check_ip_reputation(req.ip_address)
        if ip_risk == "HIGH":
            base_score += 30
            rules_triggered.append(f"HIGH_RISK_IP: {req.ip_address}")
        elif ip_risk == "MEDIUM":
            base_score += 15
            rules_triggered.append(f"MEDIUM_RISK_IP: {req.ip_address}")
        
        # Rule 4: Device fingerprint check
        if self._is_new_device(req.customer_id, req.device_fingerprint):
            base_score += 10
            rules_triggered.append("NEW_DEVICE")
        
        # Rule 5: Geographic anomaly
        if self._is_geographic_anomaly(req.customer_id, req.ip_address):
            base_score += 20
            rules_triggered.append("GEOGRAPHIC_ANOMALY")
        
        # Rule 6: Card testing (small amounts after new token)
        if req.amount < 10 and self._is_new_card_token(req.card_token):
            base_score += 25
            rules_triggered.append("POTENTIAL_CARD_TESTING")
        
        # ML Model Score (ใช้ gradient boosting)
        features = self._extract_features(req)
        ml_score = self.model.predict_proba([features])[0][1] * 100
        
        # Combine rule-based + ML (weighted average)
        final_score = (base_score * 0.4) + (ml_score * 0.6)
        final_score = min(100, final_score)
        
        # Decision
        if final_score >= self.REJECT_THRESHOLD:
            decision = "REJECT"
        elif final_score >= self.REVIEW_THRESHOLD:
            decision = "REVIEW"
        else:
            decision = "APPROVE"
        
        # Update velocity counters
        self._increment_velocity(req.customer_id)
        
        return FraudScore(
            score=round(final_score, 2),
            decision=decision,
            reasons=self._get_human_reasons(rules_triggered),
            rules_triggered=rules_triggered
        )
    
    def _extract_features(self, req: FraudCheckRequest) -> List[float]:
        """Extract ML features from transaction"""
        customer_history = self._get_customer_history(req.customer_id)
        
        return [
            req.amount,
            req.amount / max(customer_history.get('avg_amount', req.amount), 1),
            customer_history.get('total_transactions', 0),
            customer_history.get('failed_transactions', 0),
            customer_history.get('hourly_count', 0),
            customer_history.get('daily_count', 0),
            1 if self._is_new_device(req.customer_id, req.device_fingerprint) else 0,
            1 if self._is_geographic_anomaly(req.customer_id, req.ip_address) else 0,
            self._get_merchant_risk_score(req.merchant_id),
            self._get_time_features(req.timestamp),
        ]
    
    def _get_hourly_count(self, customer_id: str) -> int:
        key = f"fraud:velocity:hourly:{customer_id}"
        count = self.redis.get(key)
        return int(count) if count else 0
    
    def _increment_velocity(self, customer_id: str):
        hour_key = f"fraud:velocity:hourly:{customer_id}"
        day_key = f"fraud:velocity:daily:{customer_id}"
        
        pipe = self.redis.pipeline()
        pipe.incr(hour_key)
        pipe.expire(hour_key, 3600)  # 1 hour
        pipe.incr(day_key)
        pipe.expire(day_key, 86400)  # 24 hours
        pipe.execute()
```

### 6.2 Real-time Fraud Rules Engine

```go
// fraud-detection/internal/rules/engine.go
package rules

import (
    "context"
    "time"
)

type Rule struct {
    ID          string
    Name        string
    Description string
    Score       float64
    Condition   func(ctx context.Context, txn Transaction, history CustomerHistory) bool
}

type RulesEngine struct {
    rules  []Rule
    redis  *redis.Client
}

func NewRulesEngine(redis *redis.Client) *RulesEngine {
    engine := &RulesEngine{redis: redis}
    
    engine.rules = []Rule{
        {
            ID:          "R001",
            Name:        "BLOCKED_CARD",
            Description: "Card is in blocklist",
            Score:       100,
            Condition: func(ctx context.Context, txn Transaction, _ CustomerHistory) bool {
                blocked, _ := engine.isCardBlocked(ctx, txn.CardToken)
                return blocked
            },
        },
        {
            ID:          "R002",
            Name:        "BLOCKED_IP",
            Description: "IP address is in blocklist",
            Score:       80,
            Condition: func(ctx context.Context, txn Transaction, _ CustomerHistory) bool {
                blocked, _ := engine.isIPBlocked(ctx, txn.IPAddress)
                return blocked
            },
        },
        {
            ID:          "R003",
            Name:        "CARD_COUNTRY_MISMATCH",
            Description: "Card issuing country doesn't match transaction country",
            Score:       30,
            Condition: func(ctx context.Context, txn Transaction, _ CustomerHistory) bool {
                return txn.CardCountry != "TH" && txn.MerchantCountry == "TH"
            },
        },
        {
            ID:          "R004",
            Name:        "RAPID_RETRIES",
            Description: "Multiple failed attempts in short time",
            Score:       50,
            Condition: func(ctx context.Context, txn Transaction, history CustomerHistory) bool {
                return history.FailedLastHour >= 3
            },
        },
        {
            ID:          "R005",
            Name:        "NIGHT_LARGE_TRANSACTION",
            Description: "Large transaction during unusual hours",
            Score:       20,
            Condition: func(ctx context.Context, txn Transaction, _ CustomerHistory) bool {
                hour := time.Now().In(bangkokTZ).Hour()
                return (hour >= 1 && hour <= 5) && txn.Amount > 50000 // > 50,000 THB
            },
        },
    }
    
    return engine
}

func (e *RulesEngine) Evaluate(ctx context.Context, txn Transaction, history CustomerHistory) []RuleResult {
    var results []RuleResult
    
    for _, rule := range e.rules {
        if rule.Condition(ctx, txn, history) {
            results = append(results, RuleResult{
                RuleID:  rule.ID,
                Name:    rule.Name,
                Score:   rule.Score,
                Matched: true,
            })
        }
    }
    
    return results
}
```

---

## 7. Refund System

```go
// payment-service/internal/refund/processor.go
package refund

type RefundProcessor struct {
    txnRepo      TransactionRepository
    providerMap  map[string]PaymentProvider
    ledger       LedgerService
    notifier     NotificationService
}

func (p *RefundProcessor) ProcessRefund(ctx context.Context, req RefundRequest) (*Refund, error) {
    // 1. Get original transaction
    originalTxn, err := p.txnRepo.GetByID(ctx, req.TransactionID)
    if err != nil {
        return nil, ErrTransactionNotFound
    }
    
    // 2. Validate
    if originalTxn.Status != "completed" {
        return nil, ErrInvalidTransactionStatus
    }
    
    totalRefunded, _ := p.txnRepo.GetTotalRefunded(ctx, req.TransactionID)
    if totalRefunded.Add(req.Amount).GreaterThan(originalTxn.Amount) {
        return nil, ErrRefundExceedsOriginal
    }
    
    // 3. Check refund window (ตาม Business Rule)
    daysSinceCharge := time.Since(originalTxn.CreatedAt).Hours() / 24
    if daysSinceCharge > 90 {
        return nil, ErrRefundWindowExpired
    }
    
    // 4. Create refund record
    refund := &Refund{
        ID:              generateUUID(),
        TransactionID:   req.TransactionID,
        Amount:          req.Amount,
        Reason:          req.Reason,
        Status:          RefundStatusProcessing,
        RequestedByID:   req.RequestedByID,
        RequestedAt:     time.Now(),
    }
    
    if err := p.txnRepo.CreateRefund(ctx, refund); err != nil {
        return nil, err
    }
    
    // 5. Call payment provider
    provider := p.providerMap[originalTxn.Provider]
    result, err := provider.Refund(ctx, ProviderRefundRequest{
        OriginalChargeID: originalTxn.ExternalID,
        Amount:          req.Amount,
        Reason:          req.Reason,
    })
    
    if err != nil {
        p.txnRepo.UpdateRefundStatus(ctx, refund.ID, RefundStatusFailed, err.Error())
        return nil, fmt.Errorf("provider refund failed: %w", err)
    }
    
    // 6. Update ledger
    p.ledger.RecordRefund(ctx, LedgerRefundParams{
        OriginalTransactionID: req.TransactionID,
        RefundID:             refund.ID,
        Amount:               req.Amount,
        CustomerID:           originalTxn.CustomerID,
        MerchantID:           originalTxn.MerchantID,
    })
    
    // 7. Update status
    p.txnRepo.UpdateRefundStatus(ctx, refund.ID, RefundStatusCompleted, "")
    
    // 8. Notify customer
    p.notifier.SendRefundNotification(ctx, NotifyRequest{
        CustomerID: originalTxn.CustomerID,
        Amount:     req.Amount,
        OrderID:    originalTxn.OrderID,
    })
    
    return refund, nil
}
```

---

## สรุป

ระบบชำระเงินในไทยต้องคำนึงถึง:

1. **PCI-DSS Compliance** - Tokenization, Audit Logging, Access Control
2. **Local Payment Methods** - PromptPay, TrueMoney, Mobile Banking integration
3. **Double-Entry Bookkeeping** - ความถูกต้องทางการเงิน 100%
4. **Reconciliation** - ตรวจสอบกับ Bank ทุกวัน
5. **Fraud Detection** - Rule-based + ML สำหรับ Real-time scoring
6. **Idempotency** - ป้องกัน Double charge
7. **Saga Pattern** - จัดการ Distributed Transactions

> "In payment systems, correctness is not optional — it's the foundation"

---

*ถัดไป: Part 94 - Case Study: Food Delivery*
