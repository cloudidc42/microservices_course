# Part 93: Case Studies - Payment System

## ระบบการชำระเงิน (Payment System) - กรณีศึกษา

การสร้างระบบการชำระเงินในสถาปัตยกรรม Microservices เป็นหนึ่งในความท้าทายที่ซับซ้อนที่สุด เนื่องจากต้องการความปลอดภัยสูง ความน่าเชื่อถือ และการปฏิบัติตามมาตรฐาน PCI DSS

---

## 1. สถาปัตยกรรมระบบการชำระเงิน

```
┌─────────────────────────────────────────────────────────────────┐
│                    Payment System Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Client ──► API Gateway ──► Payment Orchestrator                 │
│                                    │                             │
│              ┌─────────────────────┼─────────────────────┐       │
│              ▼                     ▼                     ▼       │
│    Payment Gateway Service   Fraud Detection      Currency       │
│    (Stripe/PayPal)           Service              Conversion     │
│              │                     │                             │
│              ▼                     ▼                             │
│    Transaction Service       Risk Engine                         │
│              │                                                   │
│              ▼                                                   │
│    Notification Service ──► Reconciliation Service               │
│              │                                                   │
│              ▼                                                   │
│    Chargeback Service                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### PCI DSS Compliance Requirements

PCI DSS (Payment Card Industry Data Security Standard) กำหนดข้อกำหนด 12 ข้อหลักสำหรับการรักษาความปลอดภัยข้อมูลบัตรชำระเงิน:

1. **ติดตั้งและดูแล Firewall** - ป้องกันข้อมูลผู้ถือบัตร
2. **ไม่ใช้รหัสผ่านค่าเริ่มต้น** - เปลี่ยนรหัสผ่านและการตั้งค่าความปลอดภัยทั้งหมด
3. **ป้องกันข้อมูลผู้ถือบัตร** - เข้ารหัสข้อมูลที่จัดเก็บ
4. **เข้ารหัสการส่งข้อมูล** - ใช้ TLS/SSL สำหรับการส่งข้อมูลผ่านเครือข่าย
5. **ใช้ Anti-Virus** - ป้องกันมัลแวร์
6. **พัฒนาระบบที่ปลอดภัย** - ใช้ Secure SDLC
7. **จำกัดการเข้าถึงข้อมูล** - Need-to-know basis
8. **กำหนด Unique ID** - สำหรับทุกคนที่เข้าถึงระบบ
9. **จำกัดการเข้าถึงทางกายภาพ** - ป้องกันการเข้าถึงโดยตรง
10. **ติดตามและ Monitor** - บันทึกทุกการเข้าถึง
11. **ทดสอบระบบความปลอดภัย** - ทดสอบสม่ำเสมอ
12. **นโยบายความปลอดภัยข้อมูล** - สำหรับบุคลากรทั้งหมด

---

## 2. Payment Orchestrator Service

```typescript
// payment-orchestrator/src/orchestrator.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { v4 as uuidv4 } from 'uuid';

export interface PaymentRequest {
  orderId: string;
  customerId: string;
  amount: number;
  currency: string;
  paymentMethod: PaymentMethod;
  metadata?: Record<string, any>;
}

export interface PaymentMethod {
  type: 'card' | 'paypal' | 'bank_transfer';
  token?: string;
  paypalOrderId?: string;
  bankAccount?: BankAccount;
}

export interface PaymentResult {
  paymentId: string;
  status: PaymentStatus;
  transactionId?: string;
  errorCode?: string;
  errorMessage?: string;
  amount: number;
  currency: string;
  processedAt: Date;
}

export enum PaymentStatus {
  PENDING = 'PENDING',
  PROCESSING = 'PROCESSING',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
  CANCELLED = 'CANCELLED',
  REFUNDED = 'REFUNDED',
}

export interface BankAccount {
  accountNumber: string;
  routingNumber: string;
  accountType: 'checking' | 'savings';
}

@Injectable()
export class PaymentOrchestratorService {
  private readonly logger = new Logger(PaymentOrchestratorService.name);

  constructor(
    private readonly fraudDetectionService: FraudDetectionService,
    private readonly currencyConversionService: CurrencyConversionService,
    private readonly paymentGatewayService: PaymentGatewayService,
    private readonly transactionService: TransactionService,
    private readonly notificationService: NotificationService,
    private readonly eventEmitter: EventEmitter2,
    private readonly dataSource: DataSource,
  ) {}

  async processPayment(request: PaymentRequest): Promise<PaymentResult> {
    const paymentId = uuidv4();
    const idempotencyKey = `${request.orderId}-${request.customerId}`;

    this.logger.log(`Processing payment ${paymentId} for order ${request.orderId}`);

    // ตรวจสอบ Idempotency - ป้องกันการชำระซ้ำ
    const existingPayment = await this.transactionService.findByIdempotencyKey(idempotencyKey);
    if (existingPayment) {
      this.logger.log(`Duplicate payment detected for key ${idempotencyKey}`);
      return this.mapToPaymentResult(existingPayment);
    }

    // เริ่ม Database Transaction
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. บันทึก Transaction เริ่มต้นด้วยสถานะ PENDING
      const transaction = await this.transactionService.create({
        paymentId,
        idempotencyKey,
        ...request,
        status: PaymentStatus.PENDING,
      }, queryRunner);

      // 2. ตรวจสอบการฉ้อโกง (Fraud Detection)
      const fraudResult = await this.fraudDetectionService.analyze({
        customerId: request.customerId,
        amount: request.amount,
        currency: request.currency,
        paymentMethod: request.paymentMethod,
        metadata: request.metadata,
      });

      if (fraudResult.isBlocked) {
        await this.transactionService.updateStatus(
          paymentId,
          PaymentStatus.FAILED,
          'FRAUD_DETECTED',
          fraudResult.reason,
          queryRunner,
        );
        await queryRunner.commitTransaction();
        
        this.eventEmitter.emit('payment.fraud_detected', { paymentId, ...fraudResult });
        
        return {
          paymentId,
          status: PaymentStatus.FAILED,
          errorCode: 'FRAUD_DETECTED',
          errorMessage: fraudResult.reason,
          amount: request.amount,
          currency: request.currency,
          processedAt: new Date(),
        };
      }

      // 3. แปลงสกุลเงิน (ถ้าจำเป็น)
      let processAmount = request.amount;
      let processCurrency = request.currency;
      
      if (request.currency !== 'USD') {
        const conversionResult = await this.currencyConversionService.convert(
          request.amount,
          request.currency,
          'USD',
        );
        processAmount = conversionResult.convertedAmount;
        processCurrency = 'USD';
        
        await this.transactionService.updateConversionInfo(
          paymentId,
          {
            originalAmount: request.amount,
            originalCurrency: request.currency,
            convertedAmount: processAmount,
            exchangeRate: conversionResult.rate,
          },
          queryRunner,
        );
      }

      // 4. อัปเดตสถานะเป็น PROCESSING
      await this.transactionService.updateStatus(
        paymentId,
        PaymentStatus.PROCESSING,
        undefined,
        undefined,
        queryRunner,
      );

      // 5. ประมวลผลการชำระเงินผ่าน Gateway
      const gatewayResult = await this.paymentGatewayService.processPayment({
        paymentId,
        amount: processAmount,
        currency: processCurrency,
        paymentMethod: request.paymentMethod,
        metadata: {
          orderId: request.orderId,
          customerId: request.customerId,
        },
      });

      if (gatewayResult.success) {
        // 6. อัปเดตสถานะเป็น COMPLETED
        await this.transactionService.updateStatus(
          paymentId,
          PaymentStatus.COMPLETED,
          undefined,
          undefined,
          queryRunner,
          gatewayResult.transactionId,
        );

        await queryRunner.commitTransaction();

        // 7. ส่ง Event สำหรับ Notification
        this.eventEmitter.emit('payment.completed', {
          paymentId,
          orderId: request.orderId,
          customerId: request.customerId,
          amount: request.amount,
          currency: request.currency,
          transactionId: gatewayResult.transactionId,
        });

        return {
          paymentId,
          status: PaymentStatus.COMPLETED,
          transactionId: gatewayResult.transactionId,
          amount: request.amount,
          currency: request.currency,
          processedAt: new Date(),
        };
      } else {
        await this.transactionService.updateStatus(
          paymentId,
          PaymentStatus.FAILED,
          gatewayResult.errorCode,
          gatewayResult.errorMessage,
          queryRunner,
        );

        await queryRunner.commitTransaction();

        this.eventEmitter.emit('payment.failed', {
          paymentId,
          orderId: request.orderId,
          errorCode: gatewayResult.errorCode,
        });

        return {
          paymentId,
          status: PaymentStatus.FAILED,
          errorCode: gatewayResult.errorCode,
          errorMessage: gatewayResult.errorMessage,
          amount: request.amount,
          currency: request.currency,
          processedAt: new Date(),
        };
      }
    } catch (error) {
      await queryRunner.rollbackTransaction();
      this.logger.error(`Payment ${paymentId} failed with error: ${error.message}`, error.stack);
      
      throw error;
    } finally {
      await queryRunner.release();
    }
  }

  async refundPayment(paymentId: string, amount?: number, reason?: string): Promise<PaymentResult> {
    const transaction = await this.transactionService.findById(paymentId);
    
    if (!transaction) {
      throw new Error(`Payment ${paymentId} not found`);
    }

    if (transaction.status !== PaymentStatus.COMPLETED) {
      throw new Error(`Cannot refund payment with status ${transaction.status}`);
    }

    const refundAmount = amount || transaction.amount;
    
    const refundResult = await this.paymentGatewayService.refundPayment(
      transaction.transactionId,
      refundAmount,
      reason,
    );

    if (refundResult.success) {
      await this.transactionService.updateStatus(paymentId, PaymentStatus.REFUNDED);
      
      this.eventEmitter.emit('payment.refunded', {
        paymentId,
        refundAmount,
        reason,
      });
    }

    return {
      paymentId,
      status: PaymentStatus.REFUNDED,
      transactionId: refundResult.refundId,
      amount: refundAmount,
      currency: transaction.currency,
      processedAt: new Date(),
    };
  }

  private mapToPaymentResult(transaction: any): PaymentResult {
    return {
      paymentId: transaction.paymentId,
      status: transaction.status,
      transactionId: transaction.transactionId,
      amount: transaction.amount,
      currency: transaction.currency,
      processedAt: transaction.createdAt,
    };
  }
}
```

---

## 3. Payment Gateway Integration

### Stripe Integration

```typescript
// payment-gateway/src/stripe.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Stripe from 'stripe';
import { ConfigService } from '@nestjs/config';

export interface GatewayPaymentRequest {
  paymentId: string;
  amount: number;
  currency: string;
  paymentMethod: {
    type: string;
    token?: string;
  };
  metadata: Record<string, string>;
}

export interface GatewayPaymentResult {
  success: boolean;
  transactionId?: string;
  errorCode?: string;
  errorMessage?: string;
  gatewayResponse?: any;
}

@Injectable()
export class StripePaymentService {
  private readonly stripe: Stripe;
  private readonly logger = new Logger(StripePaymentService.name);

  constructor(private readonly configService: ConfigService) {
    this.stripe = new Stripe(
      this.configService.get<string>('STRIPE_SECRET_KEY'),
      { apiVersion: '2023-10-16' }
    );
  }

  async processPayment(request: GatewayPaymentRequest): Promise<GatewayPaymentResult> {
    try {
      this.logger.log(`Processing Stripe payment for ${request.paymentId}`);

      // สร้าง Payment Intent
      const paymentIntent = await this.stripe.paymentIntents.create({
        amount: Math.round(request.amount * 100), // แปลงเป็น cents
        currency: request.currency.toLowerCase(),
        payment_method: request.paymentMethod.token,
        confirm: true,
        metadata: {
          paymentId: request.paymentId,
          ...request.metadata,
        },
        // ตั้งค่า Idempotency key
        idempotencyKey: request.paymentId,
      });

      if (paymentIntent.status === 'succeeded') {
        return {
          success: true,
          transactionId: paymentIntent.id,
          gatewayResponse: paymentIntent,
        };
      } else if (paymentIntent.status === 'requires_action') {
        // จำเป็นต้องมีการยืนยัน 3D Secure
        return {
          success: false,
          errorCode: 'REQUIRES_ACTION',
          errorMessage: 'Payment requires additional authentication',
          gatewayResponse: paymentIntent,
        };
      } else {
        return {
          success: false,
          errorCode: 'PAYMENT_FAILED',
          errorMessage: `Payment failed with status: ${paymentIntent.status}`,
          gatewayResponse: paymentIntent,
        };
      }
    } catch (error) {
      if (error instanceof Stripe.errors.StripeCardError) {
        return {
          success: false,
          errorCode: error.code,
          errorMessage: error.message,
        };
      }
      
      if (error instanceof Stripe.errors.StripeInvalidRequestError) {
        return {
          success: false,
          errorCode: 'INVALID_REQUEST',
          errorMessage: error.message,
        };
      }

      throw error;
    }
  }

  async refundPayment(
    transactionId: string,
    amount: number,
    reason?: string,
  ): Promise<{ success: boolean; refundId?: string }> {
    try {
      const refund = await this.stripe.refunds.create({
        payment_intent: transactionId,
        amount: Math.round(amount * 100),
        reason: this.mapRefundReason(reason),
      });

      return {
        success: refund.status === 'succeeded',
        refundId: refund.id,
      };
    } catch (error) {
      this.logger.error(`Stripe refund failed: ${error.message}`);
      throw error;
    }
  }

  async createWebhookEvent(payload: string | Buffer, signature: string): Promise<Stripe.Event> {
    const webhookSecret = this.configService.get<string>('STRIPE_WEBHOOK_SECRET');
    return this.stripe.webhooks.constructEvent(payload, signature, webhookSecret);
  }

  async handleWebhookEvent(event: Stripe.Event): Promise<void> {
    switch (event.type) {
      case 'payment_intent.succeeded':
        const paymentIntent = event.data.object as Stripe.PaymentIntent;
        this.logger.log(`PaymentIntent succeeded: ${paymentIntent.id}`);
        break;
        
      case 'payment_intent.payment_failed':
        const failedPayment = event.data.object as Stripe.PaymentIntent;
        this.logger.warn(`PaymentIntent failed: ${failedPayment.id}`);
        break;
        
      case 'charge.dispute.created':
        const dispute = event.data.object as Stripe.Dispute;
        this.logger.warn(`Chargeback created: ${dispute.id}`);
        // ส่ง Event ไปยัง Chargeback Service
        break;
    }
  }

  private mapRefundReason(reason?: string): Stripe.RefundCreateParams.Reason | undefined {
    const reasonMap: Record<string, Stripe.RefundCreateParams.Reason> = {
      'duplicate': 'duplicate',
      'fraudulent': 'fraudulent',
      'customer_request': 'requested_by_customer',
    };
    return reason ? reasonMap[reason] : 'requested_by_customer';
  }
}
```

### PayPal Integration

```typescript
// payment-gateway/src/paypal.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance } from 'axios';

interface PayPalAccessToken {
  access_token: string;
  token_type: string;
  expires_in: number;
  expiresAt: Date;
}

@Injectable()
export class PayPalPaymentService {
  private readonly logger = new Logger(PayPalPaymentService.name);
  private readonly httpClient: AxiosInstance;
  private accessToken: PayPalAccessToken | null = null;

  constructor(private readonly configService: ConfigService) {
    const baseURL = configService.get('PAYPAL_ENV') === 'production'
      ? 'https://api-m.paypal.com'
      : 'https://api-m.sandbox.paypal.com';

    this.httpClient = axios.create({
      baseURL,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  private async getAccessToken(): Promise<string> {
    if (this.accessToken && new Date() < this.accessToken.expiresAt) {
      return this.accessToken.access_token;
    }

    const clientId = this.configService.get('PAYPAL_CLIENT_ID');
    const clientSecret = this.configService.get('PAYPAL_CLIENT_SECRET');
    const credentials = Buffer.from(`${clientId}:${clientSecret}`).toString('base64');

    const response = await this.httpClient.post(
      '/v1/oauth2/token',
      'grant_type=client_credentials',
      {
        headers: {
          Authorization: `Basic ${credentials}`,
          'Content-Type': 'application/x-www-form-urlencoded',
        },
      },
    );

    this.accessToken = {
      ...response.data,
      expiresAt: new Date(Date.now() + (response.data.expires_in - 60) * 1000),
    };

    return this.accessToken.access_token;
  }

  async captureOrder(orderId: string): Promise<GatewayPaymentResult> {
    try {
      const token = await this.getAccessToken();
      
      const response = await this.httpClient.post(
        `/v2/checkout/orders/${orderId}/capture`,
        {},
        {
          headers: { Authorization: `Bearer ${token}` },
        },
      );

      if (response.data.status === 'COMPLETED') {
        const capture = response.data.purchase_units[0].payments.captures[0];
        return {
          success: true,
          transactionId: capture.id,
          gatewayResponse: response.data,
        };
      }

      return {
        success: false,
        errorCode: response.data.status,
        errorMessage: 'PayPal order not completed',
        gatewayResponse: response.data,
      };
    } catch (error) {
      this.logger.error(`PayPal capture failed: ${error.message}`);
      throw error;
    }
  }

  async refundPayment(
    captureId: string,
    amount: number,
    currency: string,
  ): Promise<{ success: boolean; refundId?: string }> {
    try {
      const token = await this.getAccessToken();
      
      const response = await this.httpClient.post(
        `/v2/payments/captures/${captureId}/refund`,
        {
          amount: {
            value: amount.toFixed(2),
            currency_code: currency,
          },
        },
        {
          headers: { Authorization: `Bearer ${token}` },
        },
      );

      return {
        success: response.data.status === 'COMPLETED',
        refundId: response.data.id,
      };
    } catch (error) {
      this.logger.error(`PayPal refund failed: ${error.message}`);
      throw error;
    }
  }
}
```

---

## 4. Fraud Detection Service

```typescript
// fraud-detection/src/fraud-detection.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface FraudAnalysisRequest {
  customerId: string;
  amount: number;
  currency: string;
  paymentMethod: {
    type: string;
    token?: string;
  };
  metadata?: Record<string, any>;
  ipAddress?: string;
  deviceFingerprint?: string;
  location?: {
    country: string;
    city: string;
    latitude: number;
    longitude: number;
  };
}

export interface FraudAnalysisResult {
  isBlocked: boolean;
  riskScore: number; // 0-100
  riskLevel: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  reason?: string;
  flags: FraudFlag[];
  requiresAdditionalVerification: boolean;
}

export interface FraudFlag {
  type: string;
  severity: 'LOW' | 'MEDIUM' | 'HIGH';
  description: string;
}

@Injectable()
export class FraudDetectionService {
  private readonly logger = new Logger(FraudDetectionService.name);

  // เกณฑ์การตรวจจับการฉ้อโกง
  private readonly VELOCITY_WINDOW_SECONDS = 3600; // 1 ชั่วโมง
  private readonly MAX_TRANSACTIONS_PER_HOUR = 10;
  private readonly MAX_AMOUNT_PER_HOUR = 10000;
  private readonly HIGH_RISK_AMOUNT_THRESHOLD = 5000;
  private readonly BLOCK_SCORE_THRESHOLD = 75;
  private readonly VERIFY_SCORE_THRESHOLD = 50;

  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly riskRuleEngine: RiskRuleEngine,
    private readonly mlFraudModel: MLFraudModelService,
    private readonly blacklistService: BlacklistService,
  ) {}

  async analyze(request: FraudAnalysisRequest): Promise<FraudAnalysisResult> {
    const flags: FraudFlag[] = [];
    let riskScore = 0;

    // 1. ตรวจสอบ Blacklist
    const blacklistCheck = await this.checkBlacklist(request);
    if (blacklistCheck.isBlacklisted) {
      return {
        isBlocked: true,
        riskScore: 100,
        riskLevel: 'CRITICAL',
        reason: `Blacklisted: ${blacklistCheck.reason}`,
        flags: [{ type: 'BLACKLIST', severity: 'HIGH', description: blacklistCheck.reason }],
        requiresAdditionalVerification: false,
      };
    }

    // 2. ตรวจสอบ Velocity (ความถี่การทำธุรกรรม)
    const velocityFlags = await this.checkVelocity(request.customerId, request.amount);
    flags.push(...velocityFlags);
    riskScore += velocityFlags.reduce((sum, f) => sum + this.getFlagScore(f), 0);

    // 3. ตรวจสอบจำนวนเงิน
    const amountFlags = this.checkAmountRules(request.amount);
    flags.push(...amountFlags);
    riskScore += amountFlags.reduce((sum, f) => sum + this.getFlagScore(f), 0);

    // 4. ตรวจสอบ Geolocation
    if (request.location) {
      const geoFlags = await this.checkGeolocation(request.customerId, request.location);
      flags.push(...geoFlags);
      riskScore += geoFlags.reduce((sum, f) => sum + this.getFlagScore(f), 0);
    }

    // 5. ใช้ Rule Engine
    const ruleFlags = await this.riskRuleEngine.evaluate(request);
    flags.push(...ruleFlags);
    riskScore += ruleFlags.reduce((sum, f) => sum + this.getFlagScore(f), 0);

    // 6. ใช้ ML Model (ถ้ามี)
    try {
      const mlScore = await this.mlFraudModel.predict({
        amount: request.amount,
        customerId: request.customerId,
        deviceFingerprint: request.deviceFingerprint,
        flags: flags.map(f => f.type),
      });
      riskScore = Math.max(riskScore, mlScore * 100);
    } catch (error) {
      this.logger.warn(`ML model prediction failed: ${error.message}`);
    }

    // จำกัด Score ไม่เกิน 100
    riskScore = Math.min(riskScore, 100);

    const riskLevel = this.getRiskLevel(riskScore);
    const isBlocked = riskScore >= this.BLOCK_SCORE_THRESHOLD;
    const requiresAdditionalVerification = 
      !isBlocked && riskScore >= this.VERIFY_SCORE_THRESHOLD;

    // บันทึกผลการตรวจสอบใน Redis เพื่อ Analytics
    await this.recordAnalysisResult(request.customerId, riskScore, isBlocked);

    return {
      isBlocked,
      riskScore,
      riskLevel,
      reason: isBlocked ? this.getBlockReason(flags) : undefined,
      flags,
      requiresAdditionalVerification,
    };
  }

  private async checkVelocity(customerId: string, amount: number): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];
    const key = `velocity:${customerId}`;
    const now = Date.now();
    const windowStart = now - this.VELOCITY_WINDOW_SECONDS * 1000;

    // ใช้ Redis Sorted Set เก็บ timestamp ของธุรกรรม
    await this.redis.zadd(key, now, `${now}`);
    await this.redis.zremrangebyscore(key, '-inf', windowStart);
    await this.redis.expire(key, this.VELOCITY_WINDOW_SECONDS * 2);

    const transactionCount = await this.redis.zcard(key);
    
    if (transactionCount > this.MAX_TRANSACTIONS_PER_HOUR) {
      flags.push({
        type: 'HIGH_VELOCITY',
        severity: 'HIGH',
        description: `${transactionCount} transactions in the last hour`,
      });
    } else if (transactionCount > this.MAX_TRANSACTIONS_PER_HOUR * 0.7) {
      flags.push({
        type: 'MEDIUM_VELOCITY',
        severity: 'MEDIUM',
        description: `${transactionCount} transactions approaching limit`,
      });
    }

    // ตรวจสอบยอดรวมต่อชั่วโมง
    const amountKey = `velocity:amount:${customerId}`;
    const currentHourAmount = parseFloat(await this.redis.get(amountKey) || '0');
    const newTotal = currentHourAmount + amount;
    
    await this.redis.setex(amountKey, this.VELOCITY_WINDOW_SECONDS, newTotal.toString());

    if (newTotal > this.MAX_AMOUNT_PER_HOUR) {
      flags.push({
        type: 'HIGH_AMOUNT_VELOCITY',
        severity: 'HIGH',
        description: `Total amount $${newTotal} exceeds hourly limit`,
      });
    }

    return flags;
  }

  private checkAmountRules(amount: number): FraudFlag[] {
    const flags: FraudFlag[] = [];
    
    if (amount > this.HIGH_RISK_AMOUNT_THRESHOLD) {
      flags.push({
        type: 'HIGH_AMOUNT',
        severity: 'MEDIUM',
        description: `Transaction amount $${amount} exceeds threshold`,
      });
    }

    // ตรวจสอบจำนวนเงินที่น่าสงสัย (เช่น จำนวนที่ใกล้ขีดจำกัดการรายงาน)
    if (amount === 9999 || amount === 4999) {
      flags.push({
        type: 'STRUCTURING_SUSPICION',
        severity: 'HIGH',
        description: 'Amount appears structured to avoid reporting thresholds',
      });
    }

    return flags;
  }

  private async checkGeolocation(
    customerId: string,
    location: { country: string; city: string; latitude: number; longitude: number },
  ): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];
    const lastLocationKey = `location:${customerId}`;
    const lastLocation = await this.redis.hgetall(lastLocationKey);

    if (lastLocation && lastLocation.country) {
      // ตรวจสอบ Impossible Travel
      if (lastLocation.country !== location.country) {
        const timeSinceLastTransaction = Date.now() - parseInt(lastLocation.timestamp);
        const distanceKm = this.calculateDistance(
          parseFloat(lastLocation.latitude),
          parseFloat(lastLocation.longitude),
          location.latitude,
          location.longitude,
        );
        
        const maxPossibleSpeed = 1000; // km/h (เครื่องบิน)
        const timeHours = timeSinceLastTransaction / 3600000;
        const requiredSpeed = distanceKm / timeHours;

        if (requiredSpeed > maxPossibleSpeed) {
          flags.push({
            type: 'IMPOSSIBLE_TRAVEL',
            severity: 'HIGH',
            description: `Impossible travel detected: ${distanceKm.toFixed(0)}km in ${(timeHours * 60).toFixed(0)} minutes`,
          });
        }
      }
    }

    // บันทึก Location ปัจจุบัน
    await this.redis.hset(lastLocationKey, {
      country: location.country,
      city: location.city,
      latitude: location.latitude.toString(),
      longitude: location.longitude.toString(),
      timestamp: Date.now().toString(),
    });
    await this.redis.expire(lastLocationKey, 86400); // 24 ชั่วโมง

    return flags;
  }

  private calculateDistance(lat1: number, lon1: number, lat2: number, lon2: number): number {
    const R = 6371; // รัศมีโลก (km)
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
              Math.sin(dLon/2) * Math.sin(dLon/2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return R * c;
  }

  private async checkBlacklist(request: FraudAnalysisRequest): Promise<{
    isBlacklisted: boolean;
    reason?: string;
  }> {
    const results = await Promise.all([
      this.blacklistService.checkCustomer(request.customerId),
      request.ipAddress ? this.blacklistService.checkIP(request.ipAddress) : Promise.resolve(null),
    ]);

    for (const result of results) {
      if (result?.isBlacklisted) {
        return result;
      }
    }

    return { isBlacklisted: false };
  }

  private getRiskLevel(score: number): 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL' {
    if (score < 25) return 'LOW';
    if (score < 50) return 'MEDIUM';
    if (score < 75) return 'HIGH';
    return 'CRITICAL';
  }

  private getFlagScore(flag: FraudFlag): number {
    const severityScores = { LOW: 10, MEDIUM: 25, HIGH: 40 };
    return severityScores[flag.severity] || 0;
  }

  private getBlockReason(flags: FraudFlag[]): string {
    const highSeverityFlags = flags.filter(f => f.severity === 'HIGH');
    if (highSeverityFlags.length > 0) {
      return highSeverityFlags.map(f => f.description).join('; ');
    }
    return 'Multiple risk factors detected';
  }

  private async recordAnalysisResult(
    customerId: string,
    score: number,
    isBlocked: boolean,
  ): Promise<void> {
    const key = `fraud:stats:${customerId}`;
    await this.redis.lpush(key, JSON.stringify({
      score,
      isBlocked,
      timestamp: new Date().toISOString(),
    }));
    await this.redis.ltrim(key, 0, 99); // เก็บ 100 รายการล่าสุด
    await this.redis.expire(key, 86400 * 30); // 30 วัน
  }
}
```

---

## 5. Currency Conversion Service

```typescript
// currency-conversion/src/currency.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';
import { firstValueFrom } from 'rxjs';

export interface ConversionResult {
  fromCurrency: string;
  toCurrency: string;
  originalAmount: number;
  convertedAmount: number;
  rate: number;
  rateTimestamp: Date;
}

@Injectable()
export class CurrencyConversionService {
  private readonly logger = new Logger(CurrencyConversionService.name);
  private readonly RATE_CACHE_TTL = 3600; // 1 ชั่วโมง
  private readonly RATE_CACHE_PREFIX = 'exchange_rate:';

  constructor(
    private readonly httpService: HttpService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async convert(
    amount: number,
    fromCurrency: string,
    toCurrency: string,
  ): Promise<ConversionResult> {
    if (fromCurrency === toCurrency) {
      return {
        fromCurrency,
        toCurrency,
        originalAmount: amount,
        convertedAmount: amount,
        rate: 1,
        rateTimestamp: new Date(),
      };
    }

    const rate = await this.getExchangeRate(fromCurrency, toCurrency);
    const convertedAmount = Math.round(amount * rate * 100) / 100;

    return {
      fromCurrency,
      toCurrency,
      originalAmount: amount,
      convertedAmount,
      rate,
      rateTimestamp: new Date(),
    };
  }

  async getExchangeRate(fromCurrency: string, toCurrency: string): Promise<number> {
    const cacheKey = `${this.RATE_CACHE_PREFIX}${fromCurrency}_${toCurrency}`;
    
    // ตรวจสอบ Cache ก่อน
    const cachedRate = await this.redis.get(cacheKey);
    if (cachedRate) {
      return parseFloat(cachedRate);
    }

    // ดึงอัตราแลกเปลี่ยนจาก External API
    try {
      const response = await firstValueFrom(
        this.httpService.get(
          `https://api.exchangeratesapi.io/v1/latest?base=${fromCurrency}&symbols=${toCurrency}&access_key=${process.env.EXCHANGE_RATE_API_KEY}`,
        ),
      );

      const rate = response.data.rates[toCurrency];
      
      if (!rate) {
        throw new Error(`Exchange rate not found for ${fromCurrency} to ${toCurrency}`);
      }

      // Cache อัตราแลกเปลี่ยน
      await this.redis.setex(cacheKey, this.RATE_CACHE_TTL, rate.toString());
      
      return rate;
    } catch (error) {
      this.logger.error(`Failed to fetch exchange rate: ${error.message}`);
      
      // ใช้ Rate จาก Fallback ที่เก็บในฐานข้อมูล
      return this.getFallbackRate(fromCurrency, toCurrency);
    }
  }

  async getSupportedCurrencies(): Promise<string[]> {
    return ['USD', 'EUR', 'GBP', 'JPY', 'CNY', 'KRW', 'THB', 'SGD', 'AUD', 'CAD'];
  }

  private async getFallbackRate(fromCurrency: string, toCurrency: string): Promise<number> {
    // อัตราแลกเปลี่ยนสำรอง (ควรอัปเดตเป็นประจำ)
    const fallbackRates: Record<string, number> = {
      'USD_THB': 35.5,
      'EUR_THB': 38.2,
      'GBP_THB': 44.8,
      'USD_EUR': 0.92,
      'USD_GBP': 0.79,
    };

    const key = `${fromCurrency}_${toCurrency}`;
    const rate = fallbackRates[key];
    
    if (!rate) {
      throw new Error(`No fallback rate available for ${fromCurrency} to ${toCurrency}`);
    }

    this.logger.warn(`Using fallback rate for ${key}: ${rate}`);
    return rate;
  }
}
```

---

## 6. Transaction Management & Idempotency

```typescript
// transaction/src/transaction.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, QueryRunner } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

import { Transaction } from './entities/transaction.entity';
import { PaymentStatus } from '../types/payment.types';

@Injectable()
export class TransactionService {
  private readonly logger = new Logger(TransactionService.name);
  private readonly IDEMPOTENCY_TTL = 86400 * 7; // 7 วัน

  constructor(
    @InjectRepository(Transaction)
    private readonly transactionRepository: Repository<Transaction>,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async create(data: Partial<Transaction>, queryRunner?: QueryRunner): Promise<Transaction> {
    const transaction = this.transactionRepository.create(data);
    
    if (queryRunner) {
      return queryRunner.manager.save(transaction);
    }
    
    return this.transactionRepository.save(transaction);
  }

  async findByIdempotencyKey(key: string): Promise<Transaction | null> {
    // ตรวจสอบจาก Redis Cache ก่อน (เร็วกว่า)
    const cacheKey = `idempotency:${key}`;
    const cachedResult = await this.redis.get(cacheKey);
    
    if (cachedResult) {
      const transactionId = cachedResult;
      return this.transactionRepository.findOne({ where: { paymentId: transactionId } });
    }

    // ถ้าไม่มีใน Cache ตรวจสอบจาก Database
    const transaction = await this.transactionRepository.findOne({
      where: { idempotencyKey: key },
    });

    if (transaction) {
      // บันทึกลง Cache
      await this.redis.setex(cacheKey, this.IDEMPOTENCY_TTL, transaction.paymentId);
    }

    return transaction;
  }

  async findById(paymentId: string): Promise<Transaction | null> {
    return this.transactionRepository.findOne({ where: { paymentId } });
  }

  async updateStatus(
    paymentId: string,
    status: PaymentStatus,
    errorCode?: string,
    errorMessage?: string,
    queryRunner?: QueryRunner,
    transactionId?: string,
  ): Promise<void> {
    const updateData: Partial<Transaction> = {
      status,
      updatedAt: new Date(),
    };

    if (errorCode) updateData.errorCode = errorCode;
    if (errorMessage) updateData.errorMessage = errorMessage;
    if (transactionId) updateData.transactionId = transactionId;
    if (status === PaymentStatus.COMPLETED) updateData.completedAt = new Date();

    if (queryRunner) {
      await queryRunner.manager.update(Transaction, { paymentId }, updateData);
    } else {
      await this.transactionRepository.update({ paymentId }, updateData);
    }
  }

  async getTransactionStats(customerId: string): Promise<{
    totalTransactions: number;
    totalAmount: number;
    successRate: number;
  }> {
    const result = await this.transactionRepository
      .createQueryBuilder('t')
      .where('t.customerId = :customerId', { customerId })
      .select([
        'COUNT(*) as totalTransactions',
        'SUM(CASE WHEN t.status = :completed THEN t.amount ELSE 0 END) as totalAmount',
        'AVG(CASE WHEN t.status = :completed THEN 1 ELSE 0 END) * 100 as successRate',
      ])
      .setParameter('completed', PaymentStatus.COMPLETED)
      .getRawOne();

    return {
      totalTransactions: parseInt(result.totalTransactions),
      totalAmount: parseFloat(result.totalAmount),
      successRate: parseFloat(result.successRate),
    };
  }
}
```

### Transaction Entity

```typescript
// transaction/src/entities/transaction.entity.ts
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  Index,
} from 'typeorm';

@Entity('transactions')
@Index(['customerId', 'createdAt'])
@Index(['status', 'createdAt'])
export class Transaction {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  @Index()
  paymentId: string;

  @Column({ unique: true })
  @Index()
  idempotencyKey: string;

  @Column()
  @Index()
  orderId: string;

  @Column()
  @Index()
  customerId: string;

  @Column('decimal', { precision: 10, scale: 2 })
  amount: number;

  @Column({ length: 3 })
  currency: string;

  @Column({
    type: 'enum',
    enum: ['PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED', 'REFUNDED'],
    default: 'PENDING',
  })
  @Index()
  status: string;

  @Column({ nullable: true })
  transactionId: string;

  @Column({ nullable: true })
  errorCode: string;

  @Column({ nullable: true })
  errorMessage: string;

  @Column('json', { nullable: true })
  conversionInfo: {
    originalAmount: number;
    originalCurrency: string;
    convertedAmount: number;
    exchangeRate: number;
  };

  @Column('json', { nullable: true })
  metadata: Record<string, any>;

  @Column({ nullable: true })
  completedAt: Date;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## 7. Payment Notification Service

```typescript
// notification/src/payment-notification.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { MailService } from './mail.service';
import { SMSService } from './sms.service';
import { PushNotificationService } from './push.service';

interface PaymentCompletedEvent {
  paymentId: string;
  orderId: string;
  customerId: string;
  amount: number;
  currency: string;
  transactionId: string;
}

interface PaymentFailedEvent {
  paymentId: string;
  orderId: string;
  errorCode: string;
}

@Injectable()
export class PaymentNotificationService {
  private readonly logger = new Logger(PaymentNotificationService.name);

  constructor(
    @InjectQueue('notifications') private readonly notificationQueue: Queue,
    private readonly mailService: MailService,
    private readonly smsService: SMSService,
    private readonly pushService: PushNotificationService,
    private readonly customerService: CustomerService,
  ) {}

  @OnEvent('payment.completed')
  async handlePaymentCompleted(event: PaymentCompletedEvent): Promise<void> {
    this.logger.log(`Processing payment completed notification for ${event.paymentId}`);
    
    // เพิ่มใน Queue เพื่อ Process แบบ Async
    await this.notificationQueue.add('payment-completed', event, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 1000,
      },
    });
  }

  @OnEvent('payment.failed')
  async handlePaymentFailed(event: PaymentFailedEvent): Promise<void> {
    await this.notificationQueue.add('payment-failed', event, {
      attempts: 3,
    });
  }

  @OnEvent('payment.fraud_detected')
  async handleFraudDetected(event: any): Promise<void> {
    // ส่งการแจ้งเตือนด้านความปลอดภัยทันที
    await this.sendSecurityAlert(event);
  }

  async processPaymentCompletedNotification(event: PaymentCompletedEvent): Promise<void> {
    const customer = await this.customerService.findById(event.customerId);
    
    if (!customer) {
      this.logger.warn(`Customer ${event.customerId} not found`);
      return;
    }

    const notificationData = {
      recipientName: `${customer.firstName} ${customer.lastName}`,
      amount: event.amount,
      currency: event.currency,
      orderId: event.orderId,
      transactionId: event.transactionId,
      paymentDate: new Date().toLocaleDateString('th-TH'),
    };

    // ส่งแบบขนาน
    await Promise.allSettled([
      this.sendEmailNotification(customer.email, notificationData),
      customer.phone ? this.sendSMSNotification(customer.phone, notificationData) : Promise.resolve(),
      customer.pushToken ? this.sendPushNotification(customer.pushToken, notificationData) : Promise.resolve(),
    ]);
  }

  private async sendEmailNotification(email: string, data: any): Promise<void> {
    try {
      await this.mailService.send({
        to: email,
        subject: 'Payment Confirmation',
        template: 'payment-confirmation',
        data,
      });
    } catch (error) {
      this.logger.error(`Failed to send email to ${email}: ${error.message}`);
    }
  }

  private async sendSMSNotification(phone: string, data: any): Promise<void> {
    try {
      await this.smsService.send({
        to: phone,
        message: `Payment confirmed! Order #${data.orderId} - ${data.currency} ${data.amount}. Transaction ID: ${data.transactionId}`,
      });
    } catch (error) {
      this.logger.error(`Failed to send SMS to ${phone}: ${error.message}`);
    }
  }

  private async sendPushNotification(token: string, data: any): Promise<void> {
    try {
      await this.pushService.send({
        token,
        title: 'Payment Successful',
        body: `Your payment of ${data.currency} ${data.amount} has been processed`,
        data: { orderId: data.orderId },
      });
    } catch (error) {
      this.logger.error(`Failed to send push notification: ${error.message}`);
    }
  }

  private async sendSecurityAlert(event: any): Promise<void> {
    // ส่งการแจ้งเตือนด้านความปลอดภัยไปยัง Security Team
    await this.mailService.send({
      to: process.env.SECURITY_TEAM_EMAIL,
      subject: '🚨 Fraud Attempt Detected',
      template: 'fraud-alert',
      data: event,
    });
  }
}
```

---

## 8. Reconciliation Service

```typescript
// reconciliation/src/reconciliation.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

export interface ReconciliationReport {
  date: Date;
  totalTransactions: number;
  matchedTransactions: number;
  discrepancies: Discrepancy[];
  totalAmount: number;
  matchedAmount: number;
  discrepancyAmount: number;
}

export interface Discrepancy {
  transactionId: string;
  ourRecord: TransactionRecord;
  gatewayRecord: GatewayRecord;
  type: 'AMOUNT_MISMATCH' | 'STATUS_MISMATCH' | 'MISSING_IN_GATEWAY' | 'MISSING_IN_OUR_SYSTEM';
}

@Injectable()
export class ReconciliationService {
  private readonly logger = new Logger(ReconciliationService.name);

  constructor(
    @InjectRepository(Transaction)
    private readonly transactionRepository: Repository<Transaction>,
    private readonly stripeService: StripePaymentService,
    private readonly reportService: ReportService,
    private readonly alertService: AlertService,
  ) {}

  // รันทุกวัน เวลา 02:00 น.
  @Cron(CronExpression.EVERY_DAY_AT_2AM)
  async runDailyReconciliation(): Promise<void> {
    this.logger.log('Starting daily reconciliation...');
    
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    yesterday.setHours(0, 0, 0, 0);
    
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    try {
      const report = await this.reconcile(yesterday, today);
      
      this.logger.log(`Reconciliation complete: ${report.matchedTransactions}/${report.totalTransactions} matched`);
      
      // บันทึกรายงาน
      await this.reportService.saveReconciliationReport(report);
      
      // แจ้งเตือนถ้ามี Discrepancies
      if (report.discrepancies.length > 0) {
        await this.alertService.sendReconciliationAlert(report);
      }
    } catch (error) {
      this.logger.error(`Reconciliation failed: ${error.message}`, error.stack);
      await this.alertService.sendSystemAlert('Reconciliation Failed', error.message);
    }
  }

  async reconcile(startDate: Date, endDate: Date): Promise<ReconciliationReport> {
    // ดึงข้อมูลจาก Database ของเรา
    const ourTransactions = await this.transactionRepository
      .createQueryBuilder('t')
      .where('t.createdAt >= :startDate AND t.createdAt < :endDate', { startDate, endDate })
      .andWhere('t.status = :status', { status: 'COMPLETED' })
      .getMany();

    // ดึงข้อมูลจาก Stripe
    const gatewayTransactions = await this.fetchGatewayTransactions(startDate, endDate);
    
    const discrepancies: Discrepancy[] = [];
    let matchedTransactions = 0;
    let matchedAmount = 0;

    // เปรียบเทียบข้อมูล
    for (const ourTx of ourTransactions) {
      const gatewayTx = gatewayTransactions.find(
        gTx => gTx.id === ourTx.transactionId
      );

      if (!gatewayTx) {
        discrepancies.push({
          transactionId: ourTx.transactionId,
          ourRecord: { id: ourTx.paymentId, amount: ourTx.amount, status: ourTx.status },
          gatewayRecord: null,
          type: 'MISSING_IN_GATEWAY',
        });
        continue;
      }

      // ตรวจสอบจำนวนเงิน
      const gatewayAmount = gatewayTx.amount / 100; // แปลงจาก cents
      if (Math.abs(ourTx.amount - gatewayAmount) > 0.01) {
        discrepancies.push({
          transactionId: ourTx.transactionId,
          ourRecord: { id: ourTx.paymentId, amount: ourTx.amount, status: ourTx.status },
          gatewayRecord: { id: gatewayTx.id, amount: gatewayAmount, status: gatewayTx.status },
          type: 'AMOUNT_MISMATCH',
        });
        continue;
      }

      matchedTransactions++;
      matchedAmount += ourTx.amount;
    }

    // ตรวจสอบ Transactions ที่มีใน Gateway แต่ไม่มีในระบบเรา
    for (const gatewayTx of gatewayTransactions) {
      const ourTx = ourTransactions.find(t => t.transactionId === gatewayTx.id);
      if (!ourTx) {
        discrepancies.push({
          transactionId: gatewayTx.id,
          ourRecord: null,
          gatewayRecord: { id: gatewayTx.id, amount: gatewayTx.amount / 100, status: gatewayTx.status },
          type: 'MISSING_IN_OUR_SYSTEM',
        });
      }
    }

    const totalAmount = ourTransactions.reduce((sum, t) => sum + t.amount, 0);

    return {
      date: startDate,
      totalTransactions: ourTransactions.length,
      matchedTransactions,
      discrepancies,
      totalAmount,
      matchedAmount,
      discrepancyAmount: totalAmount - matchedAmount,
    };
  }

  private async fetchGatewayTransactions(startDate: Date, endDate: Date): Promise<any[]> {
    // ดึงข้อมูลจาก Stripe API
    const transactions = [];
    let hasMore = true;
    let startingAfter: string | undefined;

    while (hasMore) {
      const response = await this.stripeService.listCharges({
        created: {
          gte: Math.floor(startDate.getTime() / 1000),
          lt: Math.floor(endDate.getTime() / 1000),
        },
        limit: 100,
        starting_after: startingAfter,
      });

      transactions.push(...response.data);
      hasMore = response.has_more;
      
      if (hasMore) {
        startingAfter = response.data[response.data.length - 1].id;
      }
    }

    return transactions;
  }
}
```

---

## 9. Chargeback Handling Service

```typescript
// chargeback/src/chargeback.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { OnEvent } from '@nestjs/event-emitter';

export enum ChargebackStatus {
  RECEIVED = 'RECEIVED',
  UNDER_REVIEW = 'UNDER_REVIEW',
  EVIDENCE_SUBMITTED = 'EVIDENCE_SUBMITTED',
  WON = 'WON',
  LOST = 'LOST',
  EXPIRED = 'EXPIRED',
}

export interface ChargebackEvidence {
  type: 'RECEIPT' | 'SHIPPING_PROOF' | 'CUSTOMER_COMMUNICATION' | 'REFUND_POLICY' | 'OTHER';
  description: string;
  fileUrl?: string;
}

@Injectable()
export class ChargebackService {
  private readonly logger = new Logger(ChargebackService.name);

  constructor(
    @InjectRepository(Chargeback)
    private readonly chargebackRepository: Repository<Chargeback>,
    private readonly stripeService: StripePaymentService,
    private readonly transactionService: TransactionService,
    private readonly notificationService: PaymentNotificationService,
  ) {}

  @OnEvent('stripe.charge.dispute.created')
  async handleChargebackCreated(disputeData: any): Promise<void> {
    this.logger.log(`Chargeback received for dispute: ${disputeData.id}`);
    
    const transaction = await this.transactionService.findByTransactionId(
      disputeData.charge
    );

    if (!transaction) {
      this.logger.warn(`Transaction not found for charge: ${disputeData.charge}`);
      return;
    }

    const chargeback = await this.chargebackRepository.save({
      disputeId: disputeData.id,
      paymentId: transaction.paymentId,
      amount: disputeData.amount / 100,
      currency: disputeData.currency.toUpperCase(),
      reason: disputeData.reason,
      status: ChargebackStatus.RECEIVED,
      responseDeadline: new Date(disputeData.evidence_details.due_by * 1000),
      rawData: disputeData,
    });

    // แจ้งทีม Finance
    await this.notificationService.sendChargebackAlert(chargeback);

    // เริ่มกระบวนการตรวจสอบอัตโนมัติ
    await this.autoReviewChargeback(chargeback);
  }

  async submitEvidence(
    chargebackId: string,
    evidence: ChargebackEvidence[],
  ): Promise<void> {
    const chargeback = await this.chargebackRepository.findOne({
      where: { id: chargebackId }
    });

    if (!chargeback) {
      throw new Error(`Chargeback ${chargebackId} not found`);
    }

    if (chargeback.status !== ChargebackStatus.UNDER_REVIEW) {
      throw new Error(`Cannot submit evidence for chargeback with status ${chargeback.status}`);
    }

    // ส่ง Evidence ไปยัง Stripe
    const evidenceData = this.buildStripeEvidence(evidence, chargeback);
    
    await this.stripeService.updateDispute(chargeback.disputeId, {
      evidence: evidenceData,
      submit: true,
    });

    await this.chargebackRepository.update(chargebackId, {
      status: ChargebackStatus.EVIDENCE_SUBMITTED,
      evidence,
      evidenceSubmittedAt: new Date(),
    });

    this.logger.log(`Evidence submitted for chargeback ${chargebackId}`);
  }

  private async autoReviewChargeback(chargeback: Chargeback): Promise<void> {
    const transaction = await this.transactionService.findById(chargeback.paymentId);
    
    // ตรวจสอบว่าคุ้มค่าที่จะต่อสู้หรือไม่
    const isFightable = await this.assessFightability(chargeback, transaction);
    
    if (!isFightable) {
      this.logger.log(`Accepting chargeback ${chargeback.id} - not worth fighting`);
      await this.chargebackRepository.update(chargeback.id, {
        status: ChargebackStatus.LOST,
        notes: 'Accepted - not cost effective to fight',
      });
      return;
    }

    // เก็บ Evidence อัตโนมัติ
    const autoEvidence = await this.collectAutoEvidence(transaction);
    
    await this.chargebackRepository.update(chargeback.id, {
      status: ChargebackStatus.UNDER_REVIEW,
      autoCollectedEvidence: autoEvidence,
    });
  }

  private async assessFightability(chargeback: Chargeback, transaction: any): Promise<boolean> {
    // ต่อสู้ถ้า:
    // 1. จำนวนเงินสูงกว่า $50
    // 2. มี Evidence เพียงพอ
    // 3. ประเภทของ Chargeback ไม่ใช่ "fraud" ที่ชัดเจน
    
    if (chargeback.amount < 50) return false;
    if (chargeback.reason === 'fraudulent' && !transaction.hasDeliveryProof) return false;
    
    return true;
  }

  private async collectAutoEvidence(transaction: any): Promise<ChargebackEvidence[]> {
    const evidence: ChargebackEvidence[] = [];

    // เพิ่ม Receipt
    evidence.push({
      type: 'RECEIPT',
      description: `Transaction receipt for order ${transaction.orderId}`,
    });

    // เพิ่ม Customer Communication (ถ้ามี)
    if (transaction.customerCommunication) {
      evidence.push({
        type: 'CUSTOMER_COMMUNICATION',
        description: 'Email communication with customer',
        fileUrl: transaction.customerCommunication,
      });
    }

    return evidence;
  }

  private buildStripeEvidence(evidence: ChargebackEvidence[], chargeback: Chargeback): any {
    const stripeEvidence: any = {
      billing_address: chargeback.billingAddress,
      product_description: chargeback.productDescription,
    };

    for (const item of evidence) {
      switch (item.type) {
        case 'RECEIPT':
          stripeEvidence.receipt = item.fileUrl;
          break;
        case 'SHIPPING_PROOF':
          stripeEvidence.shipping_documentation = item.fileUrl;
          break;
        case 'CUSTOMER_COMMUNICATION':
          stripeEvidence.customer_communication = item.fileUrl;
          break;
        case 'REFUND_POLICY':
          stripeEvidence.refund_policy = item.description;
          break;
      }
    }

    return stripeEvidence;
  }
}
```

---

## 10. Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  payment-orchestrator:
    build: ./payment-orchestrator
    ports:
      - "3001:3000"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/payments
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092
      FRAUD_SERVICE_URL: http://fraud-detection:3000
      CURRENCY_SERVICE_URL: http://currency-conversion:3000
    depends_on:
      - postgres
      - redis
      - kafka
    networks:
      - payment-network

  fraud-detection:
    build: ./fraud-detection
    ports:
      - "3002:3000"
    environment:
      REDIS_URL: redis://redis:6379
      ML_MODEL_URL: http://ml-model:8000
    depends_on:
      - redis
    networks:
      - payment-network

  currency-conversion:
    build: ./currency-conversion
    ports:
      - "3003:3000"
    environment:
      REDIS_URL: redis://redis:6379
      EXCHANGE_RATE_API_KEY: ${EXCHANGE_RATE_API_KEY}
    networks:
      - payment-network

  notification:
    build: ./notification
    ports:
      - "3004:3000"
    environment:
      KAFKA_BROKERS: kafka:9092
      SMTP_HOST: ${SMTP_HOST}
      SMTP_PORT: ${SMTP_PORT}
      TWILIO_ACCOUNT_SID: ${TWILIO_ACCOUNT_SID}
      TWILIO_AUTH_TOKEN: ${TWILIO_AUTH_TOKEN}
    networks:
      - payment-network

  reconciliation:
    build: ./reconciliation
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/payments
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    networks:
      - payment-network

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: payments
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    networks:
      - payment-network

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass redispassword
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - payment-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    networks:
      - payment-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - payment-network

networks:
  payment-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

---

## 11. Kubernetes Deployment

```yaml
# k8s/payment-orchestrator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-orchestrator
  namespace: payment-system
  labels:
    app: payment-orchestrator
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-orchestrator
  template:
    metadata:
      labels:
        app: payment-orchestrator
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: payment-sa
      containers:
        - name: payment-orchestrator
          image: your-registry/payment-orchestrator:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: redis-url
            - name: STRIPE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: stripe-secret-key
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - payment-orchestrator
                topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Service
metadata:
  name: payment-orchestrator-svc
  namespace: payment-system
spec:
  selector:
    app: payment-orchestrator
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-orchestrator-hpa
  namespace: payment-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-orchestrator
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 12. PCI DSS Network Policy

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-network-policy
  namespace: payment-system
spec:
  podSelector:
    matchLabels:
      app: payment-orchestrator
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # อนุญาตให้รับจาก API Gateway เท่านั้น
    - from:
        - namespaceSelector:
            matchLabels:
              name: api-gateway
          podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 3000
  egress:
    # อนุญาตให้เชื่อมต่อ Database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # อนุญาตให้เชื่อมต่อ Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # อนุญาตให้เชื่อมต่อ Kafka
    - to:
        - podSelector:
            matchLabels:
              app: kafka
      ports:
        - protocol: TCP
          port: 9092
    # อนุญาตให้เชื่อมต่อ External APIs (Stripe, PayPal)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

---

## 13. Monitoring & Alerting

```typescript
// monitoring/src/payment-metrics.ts
import { Injectable } from '@nestjs/common';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { Counter, Histogram, Gauge } from 'prom-client';

@Injectable()
export class PaymentMetricsService {
  constructor(
    @InjectMetric('payment_transactions_total')
    private readonly transactionCounter: Counter<string>,
    
    @InjectMetric('payment_processing_duration_seconds')
    private readonly processingDuration: Histogram<string>,
    
    @InjectMetric('payment_fraud_score')
    private readonly fraudScoreHistogram: Histogram<string>,
    
    @InjectMetric('payment_active_transactions')
    private readonly activeTransactions: Gauge<string>,
    
    @InjectMetric('payment_amount_total')
    private readonly totalAmountCounter: Counter<string>,
  ) {}

  recordTransaction(
    status: string,
    method: string,
    currency: string,
    amount: number,
  ): void {
    this.transactionCounter.labels(status, method, currency).inc();
    this.totalAmountCounter.labels(currency).inc(amount);
  }

  recordProcessingTime(durationSeconds: number, method: string): void {
    this.processingDuration.labels(method).observe(durationSeconds);
  }

  recordFraudScore(score: number, blocked: string): void {
    this.fraudScoreHistogram.labels(blocked).observe(score);
  }

  setActiveTransactions(count: number): void {
    this.activeTransactions.set(count);
  }
}
```

```yaml
# prometheus/payment-alerts.yaml
groups:
  - name: payment_alerts
    rules:
      - alert: HighPaymentFailureRate
        expr: |
          sum(rate(payment_transactions_total{status="FAILED"}[5m])) /
          sum(rate(payment_transactions_total[5m])) > 0.1
        for: 2m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "High payment failure rate detected"
          description: "Payment failure rate is {{ $value | humanizePercentage }} (threshold: 10%)"

      - alert: FraudDetectionAnomaly
        expr: |
          sum(rate(payment_transactions_total{status="FRAUD_DETECTED"}[5m])) > 10
        for: 1m
        labels:
          severity: warning
          team: security
        annotations:
          summary: "Unusual fraud activity detected"
          description: "{{ $value }} fraud attempts in the last 5 minutes"

      - alert: PaymentProcessingLatency
        expr: |
          histogram_quantile(0.95, payment_processing_duration_seconds_bucket) > 5
        for: 5m
        labels:
          severity: warning
          team: payments
        annotations:
          summary: "High payment processing latency"
          description: "95th percentile latency is {{ $value }}s"

      - alert: ReconciliationDiscrepancy
        expr: reconciliation_discrepancies_total > 0
        labels:
          severity: critical
          team: finance
        annotations:
          summary: "Payment reconciliation discrepancy found"
          description: "{{ $value }} discrepancies found in reconciliation"
```

---

## 14. Integration Tests

```typescript
// tests/payment-integration.test.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Payment System Integration Tests', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();

    // ดึง Auth Token
    const authResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password' });
    
    authToken = authResponse.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe('Payment Processing', () => {
    it('should process a valid payment successfully', async () => {
      const response = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          orderId: 'ORDER-001',
          customerId: 'CUSTOMER-001',
          amount: 100.00,
          currency: 'USD',
          paymentMethod: {
            type: 'card',
            token: 'tok_visa', // Stripe Test Token
          },
        })
        .expect(201);

      expect(response.body.status).toBe('COMPLETED');
      expect(response.body.paymentId).toBeDefined();
      expect(response.body.transactionId).toBeDefined();
    });

    it('should return same result for duplicate payment (idempotency)', async () => {
      const paymentData = {
        orderId: 'ORDER-002',
        customerId: 'CUSTOMER-001',
        amount: 200.00,
        currency: 'USD',
        paymentMethod: { type: 'card', token: 'tok_visa' },
      };

      const firstResponse = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send(paymentData)
        .expect(201);

      const secondResponse = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send(paymentData)
        .expect(201);

      expect(firstResponse.body.paymentId).toBe(secondResponse.body.paymentId);
    });

    it('should reject payment with high fraud risk', async () => {
      const response = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          orderId: 'ORDER-003',
          customerId: 'HIGH-RISK-CUSTOMER',
          amount: 9999.00, // ใกล้ Structuring Threshold
          currency: 'USD',
          paymentMethod: { type: 'card', token: 'tok_visa' },
        })
        .expect(201);

      expect(response.body.status).toBe('FAILED');
      expect(response.body.errorCode).toBe('FRAUD_DETECTED');
    });

    it('should convert currency correctly', async () => {
      const response = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          orderId: 'ORDER-004',
          customerId: 'CUSTOMER-001',
          amount: 3550.00,
          currency: 'THB', // Thai Baht
          paymentMethod: { type: 'card', token: 'tok_visa' },
        })
        .expect(201);

      expect(response.body.status).toBe('COMPLETED');
    });
  });

  describe('Refunds', () => {
    it('should process a full refund', async () => {
      // ทำการชำระเงินก่อน
      const paymentResponse = await request(app.getHttpServer())
        .post('/payments')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          orderId: 'ORDER-REFUND-001',
          customerId: 'CUSTOMER-001',
          amount: 50.00,
          currency: 'USD',
          paymentMethod: { type: 'card', token: 'tok_visa' },
        })
        .expect(201);

      const paymentId = paymentResponse.body.paymentId;

      // ทำการ Refund
      const refundResponse = await request(app.getHttpServer())
        .post(`/payments/${paymentId}/refund`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ reason: 'customer_request' })
        .expect(200);

      expect(refundResponse.body.status).toBe('REFUNDED');
    });
  });
});
```

---

## สรุปบทที่ 93

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **สถาปัตยกรรม** | Payment Orchestrator + Fraud Detection + Currency Conversion + Notification + Reconciliation + Chargeback |
| **PCI DSS** | ข้อกำหนด 12 ข้อสำหรับความปลอดภัยข้อมูลบัตรชำระเงิน |
| **Payment Gateways** | Stripe (Credit Card) + PayPal (Digital Wallet) |
| **Idempotency** | ใช้ Idempotency Key ป้องกันการชำระซ้ำ + Redis Cache |
| **Fraud Detection** | Velocity Check + Geolocation + ML Model + Blacklist |
| **Currency Conversion** | External API + Redis Cache + Fallback Rates |
| **Reconciliation** | รันทุกวัน เปรียบเทียบข้อมูลกับ Gateway |
| **Chargeback** | รับ Dispute → Auto Review → Submit Evidence |
| **Monitoring** | Prometheus Metrics + Alert Rules |
| **Security** | Network Policy + Secret Management + TLS |
| **Testing** | Integration Tests ครอบคลุม Happy Path + Edge Cases |
