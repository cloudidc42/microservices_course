# Part 61: Microservices Documentation

## บทนำ

Documentation เป็นสิ่งสำคัญที่มักถูกมองข้ามในการพัฒนา Microservices API ที่ไม่มี Documentation ที่ดีทำให้ทีมพัฒนาเสียเวลาในการทำความเข้าใจ API ของกันและกัน บทนี้จะครอบคลุมการสร้าง OpenAPI 3.0 Specification, การตั้งค่า Swagger UI, AsyncAPI สำหรับ Event-driven Architecture, การจัดการ API Changelog, Postman Collections และการ Generate SDK

## 1. OpenAPI 3.0 Specification

### 1.1 Complete OpenAPI Spec สำหรับ Payment Service

```yaml
# openapi/payment-service.yaml
openapi: 3.0.3
info:
  title: Payment Service API
  description: |
    API สำหรับระบบชำระเงินของ Thai E-commerce Platform
    
    ## Authentication
    API ใช้ API Key Authentication ผ่าน `X-API-Key` header
    
    ## Rate Limiting
    - Free Tier: 60 requests/minute
    - Premium Tier: 1,000 requests/minute
    
    ## Sandbox Environment
    ใช้ `https://api-sandbox.company.th` สำหรับทดสอบ
    
  version: 2.1.0
  contact:
    name: API Support
    email: api-support@company.th
    url: https://docs.company.th
  license:
    name: Proprietary
    url: https://company.th/terms
  x-logo:
    url: https://company.th/logo.png

servers:
- url: https://api.company.th/v2
  description: Production Server
- url: https://api-sandbox.company.th/v2
  description: Sandbox (สำหรับทดสอบ)
- url: http://localhost:3000/v2
  description: Local Development

tags:
- name: Payments
  description: การชำระเงินและโอนเงิน
- name: Wallets
  description: กระเป๋าเงินอิเล็กทรอนิกส์
- name: Transactions
  description: ประวัติรายการธุรกรรม
- name: PromptPay
  description: ระบบพร้อมเพย์

paths:
  /payments:
    post:
      operationId: createPayment
      tags: [Payments]
      summary: สร้างรายการชำระเงิน
      description: |
        สร้างรายการชำระเงินใหม่ รองรับการชำระผ่านช่องทางต่างๆ ได้แก่
        - บัตรเครดิต/เดบิต
        - พร้อมเพย์
        - Internet Banking
        - QR Code
      security:
      - ApiKeyAuth: []
      - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePaymentRequest'
            examples:
              creditCard:
                summary: ชำระด้วยบัตรเครดิต
                value:
                  amount: 1500.00
                  currency: THB
                  method: credit_card
                  cardToken: tok_xxx123
                  orderId: order_abc456
                  description: สินค้าจาก Shop A
              promptPay:
                summary: ชำระด้วยพร้อมเพย์
                value:
                  amount: 500.00
                  currency: THB
                  method: promptpay
                  promptPayId: "0812345678"
                  orderId: order_def789
      responses:
        '201':
          description: รายการชำระเงินสร้างสำเร็จ
          headers:
            X-Request-Id:
              description: Request ID สำหรับ Tracking
              schema:
                type: string
                format: uuid
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'
              examples:
                pending:
                  summary: รอการชำระเงิน
                  value:
                    paymentId: pay_xyz789
                    status: pending
                    amount: 1500.00
                    currency: THB
                    method: credit_card
                    createdAt: "2024-01-15T10:30:00+07:00"
                    expiresAt: "2024-01-15T11:30:00+07:00"
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /payments/{paymentId}:
    get:
      operationId: getPayment
      tags: [Payments]
      summary: ดูรายละเอียดการชำระเงิน
      security:
      - ApiKeyAuth: []
      parameters:
      - $ref: '#/components/parameters/PaymentId'
      responses:
        '200':
          description: ข้อมูลการชำระเงิน
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'
        '404':
          $ref: '#/components/responses/NotFound'

  /payments/promptpay/qr:
    post:
      operationId: generatePromptPayQR
      tags: [PromptPay]
      summary: สร้าง QR Code พร้อมเพย์
      description: สร้าง QR Code สำหรับรับชำระเงินผ่านพร้อมเพย์
      security:
      - ApiKeyAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [amount, reference]
              properties:
                amount:
                  type: number
                  minimum: 1
                  maximum: 2000000
                  example: 1500.00
                  description: จำนวนเงิน (บาท)
                reference:
  
                  type: string
                  maxLength: 20
                  example: order_abc123
                  description: Reference สำหรับอ้างอิง
                promptPayId:
                  type: string
                  description: หมายเลขพร้อมเพย์ (ถ้าไม่ระบุจะใช้ค่า Default ของร้านค้า)
                expiresInSeconds:
                  type: integer
                  default: 300
                  minimum: 60
                  maximum: 3600
                  description: อายุของ QR Code (วินาที)
      responses:
        '200':
          description: QR Code พร้อมเพย์
          content:
            application/json:
              schema:
                type: object
                properties:
                  qrCodeData:
                    type: string
                    description: ข้อมูล QR Code (EMVCo format)
                  qrCodeImage:
                    type: string
                    format: byte
                    description: รูป QR Code (Base64 PNG)
                  expiresAt:
                    type: string
                    format: date-time
                  paymentId:
                    type: string
                    description: Payment ID สำหรับ Polling status

  /wallets/{walletId}/balance:
    get:
      operationId: getWalletBalance
      tags: [Wallets]
      summary: ดูยอดเงินในกระเป๋า
      security:
      - BearerAuth: []
      parameters:
      - name: walletId
        in: path
        required: true
        schema:
          type: string
      responses:
        '200':
          description: ยอดเงินในกระเป๋า
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/WalletBalance'

  /transactions:
    get:
      operationId: listTransactions
      tags: [Transactions]
      summary: ดูประวัติธุรกรรม
      security:
      - BearerAuth: []
      parameters:
      - $ref: '#/components/parameters/Page'
      - $ref: '#/components/parameters/Limit'
      - name: startDate
        in: query
        schema:
          type: string
          format: date
        description: วันที่เริ่มต้น (YYYY-MM-DD)
      - name: endDate
        in: query
        schema:
          type: string
          format: date
        description: วันที่สิ้นสุด (YYYY-MM-DD)
      - name: type
        in: query
        schema:
          type: string
          enum: [payment, refund, transfer, topup]
      responses:
        '200':
          description: รายการธุรกรรม
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TransactionList'

components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: API Key สำหรับ Server-to-Server calls
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT Token สำหรับ User-authenticated calls

  parameters:
    PaymentId:
      name: paymentId
      in: path
      required: true
      description: Payment ID
      schema:
        type: string
        pattern: '^pay_[a-zA-Z0-9]+$'
        example: pay_xyz789
    Page:
      name: page
      in: query
      schema:
        type: integer
        minimum: 1
        default: 1
    Limit:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  schemas:
    CreatePaymentRequest:
      type: object
      required: [amount, currency, method, orderId]
      properties:
        amount:
          type: number
          format: double
          minimum: 1
          maximum: 2000000
          example: 1500.00
          description: จำนวนเงิน
        currency:
          type: string
          enum: [THB, USD]
          default: THB
          description: สกุลเงิน
        method:
          type: string
          enum: [credit_card, debit_card, promptpay, internet_banking, qr_code]
          description: วิธีการชำระเงิน
        orderId:
          type: string
          maxLength: 100
          description: Order ID จากระบบของคุณ
        description:
          type: string
          maxLength: 255
          description: รายละเอียดการชำระเงิน
        cardToken:
          type: string
          description: Token ของบัตร (สำหรับ credit_card/debit_card)
        promptPayId:
          type: string
          description: หมายเลขพร้อมเพย์ (สำหรับ promptpay)
        metadata:
          type: object
          additionalProperties:
            type: string
          description: ข้อมูลเพิ่มเติมที่ต้องการเก็บ
          
    PaymentResponse:
      type: object
      properties:
        paymentId:
          type: string
          example: pay_xyz789
        status:
          type: string
          enum: [pending, processing, completed, failed, cancelled, refunded]
        amount:
          type: number
          example: 1500.00
        currency:
          type: string
          example: THB
        method:
          type: string
        orderId:
          type: string
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
        completedAt:
          type: string
          format: date-time
          nullable: true
        failureReason:
          type: string
          nullable: true
          description: เหตุผลที่ล้มเหลว (ถ้ามี)
          
    WalletBalance:
      type: object
      properties:
        walletId:
          type: string
        balance:
          type: number
          description: ยอดเงินคงเหลือ (บาท)
        availableBalance:
          type: number
          description: ยอดเงินที่ใช้ได้ (หลังหักยอดที่อยู่ระหว่างดำเนินการ)
        currency:
          type: string
          example: THB
        lastUpdated:
          type: string
          format: date-time
          
    TransactionList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Transaction'
        pagination:
          $ref: '#/components/schemas/Pagination'
          
    Transaction:
      type: object
      properties:
        transactionId:
          type: string
        type:
          type: string
          enum: [payment, refund, transfer, topup]
        amount:
          type: number
        currency:
          type: string
        status:
          type: string
        description:
          type: string
        createdAt:
          type: string
          format: date-time
          
    Pagination:
      type: object
      properties:
        currentPage:
          type: integer
        totalPages:
          type: integer
        totalItems:
          type: integer
        itemsPerPage:
          type: integer
        hasNextPage:
          type: boolean
        hasPreviousPage:
          type: boolean

    Error:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
          example: PAYMENT_FAILED
        message:
          type: string
          example: การชำระเงินล้มเหลว
        details:
          type: object
        requestId:
          type: string
          format: uuid

  responses:
    BadRequest:
      description: ข้อมูลที่ส่งมาไม่ถูกต้อง
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: INVALID_REQUEST
            message: ข้อมูลที่ส่งมาไม่ถูกต้อง
            details:
              amount: จำนวนเงินต้องมากกว่า 0
    Unauthorized:
      description: ไม่ได้รับการยืนยันตัวตน
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    NotFound:
      description: ไม่พบข้อมูลที่ร้องขอ
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    UnprocessableEntity:
      description: ข้อมูลถูกต้องแต่ไม่สามารถดำเนินการได้
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    TooManyRequests:
      description: ส่ง Request เกินจำนวนที่กำหนด
      headers:
        Retry-After:
          schema:
            type: integer
          description: จำนวนวินาทีที่ต้องรอก่อนส่งใหม่
        X-RateLimit-Limit:
          schema:
            type: integer
        X-RateLimit-Remaining:
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    InternalServerError:
      description: ข้อผิดพลาดภายในระบบ
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

## 2. Swagger UI Setup

### 2.1 Swagger UI Integration ใน Express

```typescript
// src/docs/swagger-setup.ts
import { Express } from 'express';
import swaggerUi from 'swagger-ui-express';
import YAML from 'yamljs';
import path from 'path';
import { version } from '../../package.json';

export function setupSwaggerUI(app: Express): void {
  const swaggerDocument = YAML.load(
    path.join(__dirname, '../../openapi/payment-service.yaml')
  );

  // อัพเดต Version จาก package.json
  swaggerDocument.info.version = version;

  const swaggerOptions: swaggerUi.SwaggerUiOptions = {
    explorer: true,
    swaggerOptions: {
      persistAuthorization: true,    // จำ Authorization ไว้
      tryItOutEnabled: false,        // ปิด Try It Out ใน Production
      filter: true,                  // เปิด Filter Tags
      displayRequestDuration: true,  // แสดง Request Duration
      docExpansion: 'list',         // แสดง Endpoints แบบ List
      defaultModelsExpandDepth: 2,  // Expand Schema 2 ระดับ
      syntaxHighlight: {
        activate: true,
        theme: 'agate',
      },
    },
    customCss: `
      .swagger-ui .topbar { display: none }
      .swagger-ui .info .title { color: #1a73e8 }
      .swagger-ui .info { margin-bottom: 20px }
    `,
    customSiteTitle: 'Payment Service API Documentation',
    customfavIcon: '/favicon.ico',
  };

  // Serve ใน Path ต่างๆ
  app.use(
    '/api-docs',
    swaggerUi.serve,
    swaggerUi.setup(swaggerDocument, swaggerOptions)
  );

  // Raw JSON Spec สำหรับ Tools อื่น
  app.get('/api-docs/openapi.json', (req, res) => {
    res.json(swaggerDocument);
  });

  // Raw YAML Spec
  app.get('/api-docs/openapi.yaml', (req, res) => {
    res.type('text/yaml');
    res.send(YAML.stringify(swaggerDocument, 10));
  });

  console.log('Swagger UI available at /api-docs');
}
```

### 2.2 OpenAPI Validation Middleware

```typescript
// src/middleware/openapi-validator.ts
import * as OpenApiValidator from 'express-openapi-validator';
import { Express } from 'express';
import path from 'path';

export function setupOpenAPIValidator(app: Express): void {
  app.use(
    OpenApiValidator.middleware({
      apiSpec: path.join(__dirname, '../../openapi/payment-service.yaml'),
      validateRequests: {
        allowUnknownQueryParameters: false,
        coerceTypes: true,
      },
      validateResponses: process.env.NODE_ENV !== 'production', // Validate responses ใน Dev เท่านั้น
      validateSecurity: {
        handlers: {
          ApiKeyAuth: async (req, scopes, schema) => {
            const apiKey = req.headers['x-api-key'];
            if (!apiKey) {
              throw new Error('API Key is required');
            }
            // Validate API Key
            return true;
          },
          BearerAuth: async (req, scopes, schema) => {
            const authHeader = req.headers.authorization;
            if (!authHeader?.startsWith('Bearer ')) {
              throw new Error('Bearer token is required');
            }
            return true;
          },
        },
      },
    })
  );
}
```

## 3. AsyncAPI สำหรับ Event-driven APIs

### 3.1 AsyncAPI Specification

```yaml
# asyncapi/payment-events.yaml
asyncapi: 2.6.0
info:
  title: Payment Service Events
  version: 1.0.0
  description: |
    Event Schema สำหรับ Payment Service
    รองรับการ Subscribe Events ผ่าน Kafka และ WebSocket

servers:
  kafka-production:
    url: kafka.company.th:9092
    protocol: kafka
    description: Kafka Cluster สำหรับ Production
    security:
    - saslScram: []
  
  kafka-sandbox:
    url: kafka-sandbox.company.th:9092
    protocol: kafka
    description: Kafka Cluster สำหรับทดสอบ
  
  websocket-production:
    url: wss://ws.company.th
    protocol: wss
    description: WebSocket สำหรับ Real-time Updates

channels:
  payment.completed:
    description: Event เมื่อการชำระเงินสำเร็จ
    subscribe:
      operationId: onPaymentCompleted
      summary: รับ Event เมื่อการชำระเงินสำเร็จ
      tags:
      - name: payments
      message:
        $ref: '#/components/messages/PaymentCompletedEvent'
    bindings:
      kafka:
        topic: payment.completed
        partitions: 10
        replicas: 3
        topicConfiguration:
          retentionMs: 604800000  # 7 วัน
          cleanupPolicy: delete

  payment.failed:
    description: Event เมื่อการชำระเงินล้มเหลว
    subscribe:
      operationId: onPaymentFailed
      message:
        $ref: '#/components/messages/PaymentFailedEvent'
    bindings:
      kafka:
        topic: payment.failed

  payment.refund.requested:
    description: Event เมื่อมีการขอ Refund
    publish:
      operationId: requestRefund
      summary: ส่ง Event ขอ Refund
      message:
        $ref: '#/components/messages/RefundRequestedEvent'
    bindings:
      kafka:
        topic: payment.refund.requested

  wallet.balance.updated:
    description: Event เมื่อยอดเงินในกระเป๋าเปลี่ยนแปลง
    subscribe:
      operationId: onWalletBalanceUpdated
      message:
        $ref: '#/components/messages/WalletBalanceUpdatedEvent'
    bindings:
      kafka:
        topic: wallet.balance.updated

components:
  messages:
    PaymentCompletedEvent:
      name: PaymentCompleted
      title: Payment Completed Event
      summary: Event ที่ส่งออกเมื่อการชำระเงินสำเร็จ
      contentType: application/json
      headers:
        type: object
        properties:
          eventId:
            type: string
            format: uuid
          eventVersion:
            type: string
            default: "1.0"
          correlationId:
            type: string
            description: ID สำหรับ Trace ทั้ง Transaction
          source:
            type: string
            example: payment-service
      payload:
        $ref: '#/components/schemas/PaymentCompletedPayload'
      examples:
      - name: BasicPayment
        summary: การชำระเงินทั่วไป
        payload:
          paymentId: pay_xyz789
          orderId: order_abc123
          userId: user_123
          amount: 1500.00
          currency: THB
          method: credit_card
          status: completed
          completedAt: "2024-01-15T10:30:00+07:00"
          merchantId: merchant_456

    PaymentFailedEvent:
      name: PaymentFailed
      payload:
        $ref: '#/components/schemas/PaymentFailedPayload'

    RefundRequestedEvent:
      name: RefundRequested
      payload:
        $ref: '#/components/schemas/RefundRequestedPayload'

    WalletBalanceUpdatedEvent:
      name: WalletBalanceUpdated
      payload:
        $ref: '#/components/schemas/WalletBalanceUpdatedPayload'

  schemas:
    PaymentCompletedPayload:
      type: object
      required: [paymentId, orderId, userId, amount, currency, status, completedAt]
      properties:
        paymentId:
          type: string
        orderId:
          type: string
        userId:
          type: string
        amount:
          type: number
        currency:
          type: string
        method:
          type: string
        status:
          type: string
          const: completed
        completedAt:
          type: string
          format: date-time
        merchantId:
          type: string
        fee:
          type: number
          description: ค่าธรรมเนียม
        netAmount:
          type: number
          description: ยอดสุทธิหลังหักค่าธรรมเนียม

    PaymentFailedPayload:
      type: object
      required: [paymentId, orderId, failedAt, reason]
      properties:
        paymentId:
          type: string
        orderId:
          type: string
        failedAt:
          type: string
          format: date-time
        reason:
          type: string
          enum: [insufficient_funds, card_declined, expired_card, invalid_account, timeout]
        errorCode:
          type: string
        retryable:
          type: boolean
          description: สามารถลองใหม่ได้หรือไม่

    RefundRequestedPayload:
      type: object
      required: [refundId, paymentId, amount, reason, requestedAt]
      properties:
        refundId:
          type: string
        paymentId:
          type: string
        amount:
          type: number
        reason:
          type: string
        requestedAt:
          type: string
          format: date-time
        requestedBy:
          type: string
          description: User ID ที่ขอ Refund

    WalletBalanceUpdatedPayload:
      type: object
      properties:
        walletId:
          type: string
        previousBalance:
          type: number
        newBalance:
          type: number
        change:
          type: number
          description: ยอดที่เปลี่ยนแปลง (บวก/ลบ)
        transactionId:
          type: string
        updatedAt:
          type: string
          format: date-time

  securitySchemes:
    saslScram:
      type: scramSha256
      description: SASL/SCRAM authentication สำหรับ Kafka
```

## 4. API Changelog Management

### 4.1 CHANGELOG.md Template

```markdown
# API Changelog

รูปแบบ Changelog อ้างอิงจาก [Keep a Changelog](https://keepachangelog.com/th/)

## [Unreleased]

## [2.1.0] - 2024-01-15

### Added (เพิ่มใหม่)
- เพิ่ม Endpoint `POST /payments/promptpay/qr` สำหรับสร้าง QR Code พร้อมเพย์
- เพิ่ม Support สำหรับ PromptPay แบบ Person-to-Person
- เพิ่ม Field `netAmount` ใน Payment Response (ยอดสุทธิหลังหักค่าธรรมเนียม)
- เพิ่ม Webhook Event `wallet.balance.updated`

### Changed (เปลี่ยนแปลง)
- อัพเดต Rate Limit สำหรับ Premium Tier จาก 500 เป็น 1,000 requests/minute
- เปลี่ยน Format ของ `completedAt` เป็น ISO 8601 with timezone

### Deprecated (เลิกใช้ในอนาคต)
- `POST /payments/v1/create` จะถูกลบใน v3.0.0 ให้ใช้ `POST /payments` แทน
- Field `card_number_last4` ใน Payment Response จะถูกแทนที่ด้วย `cardInfo.last4`

### Security (ความปลอดภัย)
- เพิ่ม Rate Limiting สำหรับ Payment API เพื่อป้องกัน Brute Force

## [2.0.0] - 2023-12-01

### Breaking Changes ⚠️
- เปลี่ยน Base URL จาก `/api/v1` เป็น `/v2`
- ลบ Field `card_cvv` ออกจาก Request Body (ใช้ Card Token แทน)
- เปลี่ยน Error Response Format ใหม่

### Migration Guide
ดู [Migration Guide v1 to v2](./migration-v1-to-v2.md)
```

### 4.2 Automated Changelog Generator

```typescript
// scripts/generate-changelog.ts
import { execSync } from 'child_process';
import * as fs from 'fs';

interface CommitInfo {
  hash: string;
  type: string;
  scope?: string;
  description: string;
  breaking: boolean;
  date: string;
}

function parseConventionalCommit(message: string): CommitInfo | null {
  // Conventional Commits format: type(scope): description
  const regex = /^(feat|fix|docs|style|refactor|perf|test|chore|revert|break)(\(([^)]+)\))?(!)?: (.+)$/;
  const match = message.match(regex);
  
  if (!match) return null;
  
  return {
    hash: '',
    type: match[1],
    scope: match[3],
    description: match[5],
    breaking: message.includes('BREAKING CHANGE') || match[4] === '!',
    date: new Date().toISOString().split('T')[0],
  };
}

function generateChangelog(
  fromTag: string,
  toTag: string = 'HEAD'
): string {
  const gitLog = execSync(
    `git log ${fromTag}..${toTag} --pretty=format:"%H|%s|%ai" --no-merges`
  ).toString();

  const commits: CommitInfo[] = gitLog
    .split('\n')
    .filter(Boolean)
    .map(line => {
      const [hash, message, date] = line.split('|');
      const parsed = parseConventionalCommit(message);
      
      if (!parsed) return null;
      
      return { ...parsed, hash: hash.substring(0, 7), date: date.split(' ')[0] };
    })
    .filter(Boolean) as CommitInfo[];

  const breaking = commits.filter(c => c.breaking);
  const features = commits.filter(c => c.type === 'feat' && !c.breaking);
  const fixes = commits.filter(c => c.type === 'fix');
  const perf = commits.filter(c => c.type === 'perf');
  const deprecated = commits.filter(c => c.scope === 'deprecated');

  let changelog = `## [${toTag}] - ${new Date().toISOString().split('T')[0]}\n\n`;

  if (breaking.length > 0) {
    changelog += '### Breaking Changes ⚠️\n';
    breaking.forEach(c => {
      changelog += `- ${c.description} ([${c.hash}](../../commit/${c.hash}))\n`;
    });
    changelog += '\n';
  }

  if (features.length > 0) {
    changelog += '### Added (เพิ่มใหม่)\n';
    features.forEach(c => {
      const scope = c.scope ? `**${c.scope}**: ` : '';
      changelog += `- ${scope}${c.description} ([${c.hash}](../../commit/${c.hash}))\n`;
    });
    changelog += '\n';
  }

  if (fixes.length > 0) {
    changelog += '### Fixed (แก้ไข)\n';
    fixes.forEach(c => {
      changelog += `- ${c.description} ([${c.hash}](../../commit/${c.hash}))\n`;
    });
    changelog += '\n';
  }

  if (perf.length > 0) {
    changelog += '### Performance\n';
    perf.forEach(c => {
      changelog += `- ${c.description} ([${c.hash}](../../commit/${c.hash}))\n`;
    });
    changelog += '\n';
  }

  return changelog;
}

// Run
const [, , fromTag, toTag] = process.argv;
if (!fromTag) {
  console.error('Usage: ts-node generate-changelog.ts <from-tag> [to-tag]');
  process.exit(1);
}

const changelog = generateChangelog(fromTag, toTag);
console.log(changelog);

const existingChangelog = fs.existsSync('CHANGELOG.md')
  ? fs.readFileSync('CHANGELOG.md', 'utf-8')
  : '# API Changelog\n\n## [Unreleased]\n\n';

const updatedChangelog = existingChangelog.replace(
  '## [Unreleased]\n\n',
  `## [Unreleased]\n\n${changelog}`
);

fs.writeFileSync('CHANGELOG.md', updatedChangelog);
console.log('CHANGELOG.md updated successfully');
```

## 5. Postman Collections

### 5.1 Postman Collection Generator

```typescript
// scripts/generate-postman-collection.ts
import YAML from 'yamljs';
import * as fs from 'fs';
import path from 'path';

interface PostmanCollection {
  info: {
    name: string;
    description: string;
    schema: string;
    version: string;
  };
  variable: Array<{ key: string; value: string; type: string }>;
  auth: {
    type: string;
    apikey: Array<{ key: string; value: string; type: string }>;
  };
  event: Array<{
    listen: string;
    script: { type: string; exec: string[] };
  }>;
  item: any[];
}

function openApiToPostman(openApiSpec: any): PostmanCollection {
  const collection: PostmanCollection = {
    info: {
      name: openApiSpec.info.title,
      description: openApiSpec.info.description || '',
      schema: 'https://schema.getpostman.com/json/collection/v2.1.0/collection.json',
      version: openApiSpec.info.version,
    },
    variable: [
      {
        key: 'baseUrl',
        value: openApiSpec.servers[1]?.url ?? 'https://api-sandbox.company.th/v2',
        type: 'string',
      },
      {
        key: 'apiKey',
        value: '{{API_KEY}}',
        type: 'string',
      },
    ],
    auth: {
      type: 'apikey',
      apikey: [
        { key: 'key', value: 'X-API-Key', type: 'string' },
        { key: 'value', value: '{{apiKey}}', type: 'string' },
        { key: 'in', value: 'header', type: 'string' },
      ],
    },
    event: [
      {
        listen: 'prerequest',
        script: {
          type: 'text/javascript',
          exec: [
            '// Set Correlation ID สำหรับ Tracking',
            "pm.request.headers.add({key: 'X-Correlation-Id', value: pm.variables.replaceIn('{{$guid}}')})",
          ],
        },
      },
      {
        listen: 'test',
        script: {
          type: 'text/javascript',
          exec: [
            '// ตรวจสอบ Response Time',
            "pm.test('Response time is less than 2000ms', function() {",
            '  pm.expect(pm.response.responseTime).to.be.below(2000);',
            '});',
            '',
            '// ตรวจสอบ Content-Type',
            "pm.test('Content-Type is JSON', function() {",
            "  pm.expect(pm.response.headers.get('Content-Type')).to.include('application/json');",
            '});',
          ],
        },
      },
    ],
    item: [],
  };

  // Group endpoints by tags
  const tagGroups: Record<string, any[]> = {};
  
  for (const [pathStr, pathItem] of Object.entries(openApiSpec.paths as Record<string, any>)) {
    for (const [method, operation] of Object.entries(pathItem as Record<string, any>)) {
      if (['get', 'post', 'put', 'patch', 'delete'].includes(method)) {
        const tags = (operation as any).tags || ['Default'];
        const tag = tags[0];
        
        if (!tagGroups[tag]) {
          tagGroups[tag] = [];
        }
        
        tagGroups[tag].push(
          createPostmanRequest(pathStr, method, operation, openApiSpec)
        );
      }
    }
  }

  // สร้าง Folders ตาม Tags
  for (const [tag, requests] of Object.entries(tagGroups)) {
    collection.item.push({
      name: tag,
      item: requests,
    });
  }

  return collection;
}

function createPostmanRequest(
  pathStr: string,
  method: string,
  operation: any,
  spec: any
): any {
  const url = `{{baseUrl}}${pathStr.replace(/{([^}]+)}/g, ':$1')}`;
  
  const request: any = {
    name: operation.summary || `${method.toUpperCase()} ${pathStr}`,
    request: {
      method: method.toUpperCase(),
      header: [
        {
          key: 'Content-Type',
          value: 'application/json',
        },
        {
          key: 'X-Request-Id',
          value: '{{$guid}}',
        },
      ],
      url: {
        raw: url,
        host: ['{{baseUrl}}'],
        path: pathStr.split('/').filter(Boolean),
        variable: (pathStr.match(/{([^}]+)}/g) || []).map((param: string) => ({
          key: param.replace(/[{}]/g, ''),
          value: '',
          description: `Path parameter: ${param}`,
        })),
      },
    },
    event: [
      {
        listen: 'test',
        script: {
          type: 'text/javascript',
          exec: generateTestScript(operation),
        },
      },
    ],
  };

  // Add Request Body
  if (operation.requestBody?.content?.['application/json']) {
    const example = Object.values(
      operation.requestBody.content['application/json'].examples || {}
    )[0] as any;
    
    request.request.body = {
      mode: 'raw',
      raw: JSON.stringify(example?.value ?? {}, null, 2),
      options: { raw: { language: 'json' } },
    };
  }

  return request;
}

function generateTestScript(operation: any): string[] {
  const scripts: string[] = [
    `pm.test('Status code is ${Object.keys(operation.responses)[0]}', function() {`,
    `  pm.response.to.have.status(${Object.keys(operation.responses)[0]});`,
    '});',
    '',
    '// Save response variables',
  ];

  // Auto-save common response fields
  const successResponse = operation.responses['200'] || operation.responses['201'];
  if (successResponse?.content?.['application/json']?.schema?.properties) {
    const props = successResponse.content['application/json'].schema.properties;
    
    for (const [key] of Object.entries(props)) {
      if (key.includes('Id') || key.includes('Token')) {
        scripts.push(`const responseData = pm.response.json();`);
        scripts.push(`if (responseData.${key}) {`);
        scripts.push(`  pm.collectionVariables.set('${key}', responseData.${key});`);
        scripts.push('}');
        break;
      }
    }
  }

  return scripts;
}

// Main
const specPath = path.join(__dirname, '../openapi/payment-service.yaml');
const spec = YAML.load(specPath);
const collection = openApiToPostman(spec);

const outputPath = path.join(__dirname, '../postman/payment-service-collection.json');
fs.mkdirSync(path.dirname(outputPath), { recursive: true });
fs.writeFileSync(outputPath, JSON.stringify(collection, null, 2));

console.log(`Postman collection generated: ${outputPath}`);
```

## 6. SDK Generation with OpenAPI Generator

### 6.1 SDK Generation Script

```bash
#!/bin/bash
# scripts/generate-sdk.sh

set -e

SPEC_FILE="openapi/payment-service.yaml"
OUTPUT_DIR="generated-sdks"
VERSION=$(cat package.json | jq -r '.version')

echo "Generating SDKs for version $VERSION"

# TypeScript/JavaScript SDK
npx @openapitools/openapi-generator-cli generate \
  -i $SPEC_FILE \
  -g typescript-axios \
  -o $OUTPUT_DIR/typescript \
  --additional-properties=npmName=@company/payment-sdk,npmVersion=$VERSION,supportsES6=true,useSingleRequestParameter=true,withSeparateModelsAndApi=true

# Python SDK
npx @openapitools/openapi-generator-cli generate \
  -i $SPEC_FILE \
  -g python \
  -o $OUTPUT_DIR/python \
  --additional-properties=packageName=company_payment,packageVersion=$VERSION,pythonAttrNoneIfUnset=true

# PHP SDK
npx @openapitools/openapi-generator-cli generate \
  -i $SPEC_FILE \
  -g php \
  -o $OUTPUT_DIR/php \
  --additional-properties=packageName=CompanyPayment,artifactVersion=$VERSION

echo "SDK generation complete!"
```

### 6.2 TypeScript SDK Customization

```typescript
// src/sdk/payment-client.ts - Custom wrapper รอบ Generated SDK
import {
  PaymentsApi,
  WalletsApi,
  TransactionsApi,
  Configuration,
  CreatePaymentRequest,
  PaymentResponse,
} from '../generated-sdks/typescript';
import axios, { AxiosInstance } from 'axios';

interface PaymentClientConfig {
  apiKey: string;
  baseUrl?: string;
  timeout?: number;
  retries?: number;
}

export class PaymentClient {
  private paymentsApi: PaymentsApi;
  private walletsApi: WalletsApi;
  private transactionsApi: TransactionsApi;
  private axiosInstance: AxiosInstance;

  constructor(config: PaymentClientConfig) {
    this.axiosInstance = axios.create({
      timeout: config.timeout ?? 30000,
    });

    // Retry Interceptor
    this.setupRetryInterceptor(config.retries ?? 3);

    // Request Interceptor - เพิ่ม Correlation ID
    this.axiosInstance.interceptors.request.use((request) => {
      request.headers['X-Correlation-Id'] = crypto.randomUUID();
      request.headers['X-SDK-Version'] = '2.1.0';
      request.headers['X-SDK-Language'] = 'typescript';
      return request;
    });

    const configuration = new Configuration({
      basePath: config.baseUrl ?? 'https://api.company.th/v2',
      apiKey: config.apiKey,
    });

    this.paymentsApi = new PaymentsApi(configuration, undefined, this.axiosInstance);
    this.walletsApi = new WalletsApi(configuration, undefined, this.axiosInstance);
    this.transactionsApi = new TransactionsApi(configuration, undefined, this.axiosInstance);
  }

  async createPayment(request: CreatePaymentRequest): Promise<PaymentResponse> {
    const response = await this.paymentsApi.createPayment(request);
    return response.data;
  }

  async getPayment(paymentId: string): Promise<PaymentResponse> {
    const response = await this.paymentsApi.getPayment(paymentId);
    return response.data;
  }

  async waitForPayment(
    paymentId: string,
    options: { timeoutMs?: number; pollingIntervalMs?: number } = {}
  ): Promise<PaymentResponse> {
    const { timeoutMs = 300000, pollingIntervalMs = 3000 } = options;
    const startTime = Date.now();

    while (Date.now() - startTime < timeoutMs) {
      const payment = await this.getPayment(paymentId);

      if (['completed', 'failed', 'cancelled'].includes(payment.status ?? '')) {
        return payment;
      }

      await this.sleep(pollingIntervalMs);
    }

    throw new Error(`Payment ${paymentId} did not complete within ${timeoutMs}ms`);
  }

  private setupRetryInterceptor(maxRetries: number): void {
    this.axiosInstance.interceptors.response.use(
      response => response,
      async error => {
        const config = error.config;

        if (!config._retryCount) {
          config._retryCount = 0;
        }

        const isRetryable =
          error.response?.status >= 500 ||
          error.code === 'ECONNABORTED' ||
          error.code === 'ETIMEDOUT';

        if (isRetryable && config._retryCount < maxRetries) {
          config._retryCount++;
          const delay = Math.pow(2, config._retryCount) * 1000;
          await this.sleep(delay);
          return this.axiosInstance(config);
        }

        return Promise.reject(error);
      }
    );
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ตัวอย่างการใช้งาน SDK ในระบบ Thai E-commerce
async function exampleUsage() {
  const client = new PaymentClient({
    apiKey: process.env.PAYMENT_API_KEY!,
    baseUrl: 'https://api-sandbox.company.th/v2',
  });

  // สร้างการชำระเงินด้วย PromptPay
  const payment = await client.createPayment({
    amount: 1500.00,
    currency: 'THB',
    method: 'promptpay',
    orderId: 'order_abc123',
    description: 'ซื้อสินค้าจาก Thai Shop',
  });

  console.log('Payment created:', payment.paymentId);

  // รอให้ชำระเงิน
  const completedPayment = await client.waitForPayment(payment.paymentId!, {
    timeoutMs: 300000,    // รอสูงสุด 5 นาที
    pollingIntervalMs: 5000, // Poll ทุก 5 วินาที
  });

  if (completedPayment.status === 'completed') {
    console.log('Payment successful!');
  } else {
    console.error('Payment failed:', completedPayment.failureReason);
  }
}
```

## สรุป

| เครื่องมือ | วัตถุประสงค์ | ประโยชน์หลัก | ความยากในการ Setup |
|-----------|------------|------------|-----------------|
| OpenAPI 3.0 | Spec ของ REST API | Standard format, เป็น Single Source of Truth | ปานกลาง |
| Swagger UI | UI สำหรับดูและทดสอบ API | Developer Experience ดีขึ้น | ง่าย |
| AsyncAPI | Spec ของ Event-driven API | Document Kafka/WebSocket Events | ปานกลาง |
| CHANGELOG | บันทึกการเปลี่ยนแปลง | Transparency, Migration ง่ายขึ้น | ง่าย |
| Postman Collection | Test และ Share API Examples | Team Collaboration | ง่าย |
| OpenAPI Generator | Auto-generate SDK | ลด Boilerplate Code มาก | ปานกลาง |

การทำ API Documentation ที่ดีช่วยลดเวลาในการ Integrate ระหว่างทีม ลด Bug จาก Misunderstanding และทำให้ Developer Experience ดีขึ้น สำหรับบริษัทไทยที่ให้บริการ Payment API ควรมี Documentation ทั้งภาษาไทยและอังกฤษ เพื่อรองรับนักพัฒนาในประเทศและต่างประเทศ
