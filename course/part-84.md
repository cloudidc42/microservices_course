# Part 84: Data Privacy and Compliance

## บทนำ

ในยุคที่ข้อมูลส่วนบุคคลมีมูลค่าสูง การปฏิบัติตามกฎหมาย Privacy ไม่ใช่แค่เรื่องของ compliance แต่เป็นความไว้วางใจที่ users มอบให้กับ platform ของเรา บทนี้ครอบคลุม PDPA ของไทย, GDPR พื้นฐาน, และ technical implementation ที่นำไปใช้ได้จริง

---

## PDPA: พระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล (Thailand)

### ข้อมูลส่วนบุคคลคืออะไร?

```
ข้อมูลส่วนบุคคล (Personal Data) หมายถึง:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ข้อมูลที่ระบุตัวบุคคลได้โดยตรง:
├── ชื่อ-นามสกุล
├── เลขบัตรประชาชน
├── หนังสือเดินทาง
├── หมายเลขโทรศัพท์
├── Email address
├── ที่อยู่
└── ใบหน้า (biometric)

ข้อมูลที่ระบุตัวบุคคลได้โดยอ้อม:
├── IP Address
├── Cookie ID
├── Device fingerprint
├── Location data
└── Behavioral data (combined)

ข้อมูลอ่อนไหว (Sensitive Personal Data) - ต้องการความระมัดระวังพิเศษ:
├── เชื้อชาติ / ชาติพันธุ์
├── ความคิดเห็นทางการเมือง
├── ศาสนา / ความเชื่อ
├── พฤติกรรมทางเพศ
├── ประวัติอาชญากรรม
├── ข้อมูลสุขภาพ
├── ข้อมูลพันธุกรรม / ชีวมิติ
└── ข้อมูลสหภาพแรงงาน
```

### สิทธิของเจ้าของข้อมูล (PDPA + GDPR)

```
Rights ที่ต้อง implement:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Right to Access (สิทธิในการเข้าถึงข้อมูล)
   → API: GET /api/v1/privacy/my-data
   → Response time: ภายใน 30 วัน

2. Right to Rectification (สิทธิในการแก้ไขข้อมูล)
   → API: PATCH /api/v1/users/:id/profile

3. Right to Erasure / "Right to be Forgotten"
   → API: DELETE /api/v1/privacy/my-data
   → ต้องลบทุก service ที่เก็บข้อมูล

4. Right to Data Portability (สิทธิในการขอข้อมูล)
   → API: GET /api/v1/privacy/export
   → Format: JSON, CSV

5. Right to Restriction (สิทธิในการจำกัดการประมวลผล)
   → ระงับการ process แต่ยังเก็บข้อมูลไว้

6. Right to Object (สิทธิในการคัดค้าน)
   → ปฏิเสธ marketing / profiling
```

---

## Data Classification

```typescript
// shared/src/privacy/data-classification.ts

export enum DataSensitivity {
  PUBLIC = 'PUBLIC',           // ข้อมูลสาธารณะ
  INTERNAL = 'INTERNAL',       // ข้อมูลภายในองค์กร
  CONFIDENTIAL = 'CONFIDENTIAL', // ข้อมูลลับ (PII)
  RESTRICTED = 'RESTRICTED',   // ข้อมูลอ่อนไหว (Sensitive PII)
}

export const DATA_CLASSIFICATION = {
  // PUBLIC
  productName: DataSensitivity.PUBLIC,
  productPrice: DataSensitivity.PUBLIC,
  publicReviews: DataSensitivity.PUBLIC,

  // INTERNAL
  orderId: DataSensitivity.INTERNAL,
  orderStatus: DataSensitivity.INTERNAL,
  internalUserId: DataSensitivity.INTERNAL,

  // CONFIDENTIAL (PII)
  fullName: DataSensitivity.CONFIDENTIAL,
  email: DataSensitivity.CONFIDENTIAL,
  phoneNumber: DataSensitivity.CONFIDENTIAL,
  dateOfBirth: DataSensitivity.CONFIDENTIAL,
  homeAddress: DataSensitivity.CONFIDENTIAL,
  ipAddress: DataSensitivity.CONFIDENTIAL,
  cookieId: DataSensitivity.CONFIDENTIAL,
  deviceFingerprint: DataSensitivity.CONFIDENTIAL,

  // RESTRICTED (Sensitive PII)
  nationalId: DataSensitivity.RESTRICTED,
  passportNumber: DataSensitivity.RESTRICTED,
  healthData: DataSensitivity.RESTRICTED,
  biometricData: DataSensitivity.RESTRICTED,
  financialAccountData: DataSensitivity.RESTRICTED,
} as const;
```

```typescript
// shared/src/privacy/pii-detector.ts
// ตรวจจับ PII ใน logs และ responses

import { Injectable } from '@nestjs/common';

@Injectable()
export class PIIDetector {
  private readonly patterns = [
    // Thai National ID
    {
      name: 'thai_national_id',
      pattern: /\b\d{13}\b/g,
      replacement: '[NATIONAL_ID_REDACTED]',
    },
    // Email
    {
      name: 'email',
      pattern: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
      replacement: '[EMAIL_REDACTED]',
    },
    // Thai phone number
    {
      name: 'thai_phone',
      pattern: /(\+66|0)\d{9}/g,
      replacement: '[PHONE_REDACTED]',
    },
    // Credit card number
    {
      name: 'credit_card',
      pattern: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
      replacement: '[CARD_REDACTED]',
    },
    // IP Address (IPv4)
    {
      name: 'ipv4',
      pattern: /\b(?:\d{1,3}\.){3}\d{1,3}\b/g,
      replacement: '[IP_REDACTED]',
    },
  ];

  redact(text: string): string {
    let result = text;
    for (const rule of this.patterns) {
      result = result.replace(rule.pattern, rule.replacement);
    }
    return result;
  }

  detectPII(text: string): { detected: boolean; types: string[] } {
    const types: string[] = [];
    for (const rule of this.patterns) {
      if (rule.pattern.test(text)) {
        types.push(rule.name);
      }
      // Reset lastIndex for global regex
      rule.pattern.lastIndex = 0;
    }
    return { detected: types.length > 0, types };
  }
}
```

---

## Privacy by Design Implementation

### Data Minimization Decorator

```typescript
// shared/src/privacy/decorators/pii.decorator.ts

import { applyDecorators } from '@nestjs/common';
import { Exclude, Transform } from 'class-transformer';

// Decorator สำหรับ fields ที่เป็น PII - ซ่อนใน responses โดยอัตโนมัติ
export const PIIField = () => Exclude({ toPlainOnly: true });

// Mask ข้อมูลแค่บางส่วน (เช่น email masking)
export const MaskEmail = () =>
  Transform(({ value }) => {
    if (!value) return value;
    const [local, domain] = value.split('@');
    return `${local.substring(0, 2)}***@${domain}`;
  });

export const MaskPhone = () =>
  Transform(({ value }) => {
    if (!value) return value;
    return `${value.substring(0, 3)}****${value.substring(-4)}`;
  });

// Usage ใน Entity/DTO
```

```typescript
// services/user/src/entities/user.entity.ts

import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';
import { Exclude, Expose } from 'class-transformer';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ unique: true })
  email: string;

  @Column({ nullable: true })
  phone: string;

  // PII fields - ต้อง encrypt at rest
  @Column({ name: 'national_id_encrypted', nullable: true })
  @Exclude()  // ไม่ส่งออก JSON เด็ดขาด
  nationalIdEncrypted: string;

  // Password - never expose
  @Column()
  @Exclude()
  password: string;

  // Consent tracking
  @Column({ type: 'jsonb', default: {} })
  consents: {
    marketing?: { given: boolean; timestamp: string };
    analytics?: { given: boolean; timestamp: string };
    thirdParty?: { given: boolean; timestamp: string };
  };

  // Data retention
  @Column({ nullable: true })
  requestedDeletionAt: Date;

  @Column({ nullable: true })
  anonymizedAt: Date;

  @CreateDateColumn()
  createdAt: Date;

  // Computed property - masking
  @Expose()
  get maskedEmail(): string {
    const [local, domain] = this.email.split('@');
    return `${local.substring(0, 2)}***@${domain}`;
  }
}
```

### Encryption at Rest สำหรับ Sensitive Data

```typescript
// shared/src/privacy/encryption.service.ts

import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyLength = 32;
  private readonly ivLength = 16;
  private readonly tagLength = 16;
  private readonly encryptionKey: Buffer;

  constructor(private readonly config: ConfigService) {
    const keyHex = this.config.get<string>('ENCRYPTION_KEY');
    if (!keyHex || keyHex.length !== 64) {
      throw new Error('ENCRYPTION_KEY must be 64 hex characters (32 bytes)');
    }
    this.encryptionKey = Buffer.from(keyHex, 'hex');
  }

  encrypt(plaintext: string): string {
    const iv = crypto.randomBytes(this.ivLength);
    const cipher = crypto.createCipheriv(this.algorithm, this.encryptionKey, iv);

    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);

    const authTag = cipher.getAuthTag();

    // Format: iv:authTag:encrypted (all in hex)
    return [
      iv.toString('hex'),
      authTag.toString('hex'),
      encrypted.toString('hex'),
    ].join(':');
  }

  decrypt(encryptedData: string): string {
    const [ivHex, authTagHex, encryptedHex] = encryptedData.split(':');

    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    const encrypted = Buffer.from(encryptedHex, 'hex');

    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.encryptionKey,
      iv
    );
    decipher.setAuthTag(authTag);

    return decipher.update(encrypted) + decipher.final('utf8');
  }

  // Deterministic encryption สำหรับ searchable fields
  // ใช้ HMAC แทน symmetric encryption
  hash(value: string): string {
    return crypto
      .createHmac('sha256', this.encryptionKey)
      .update(value.toLowerCase().trim())
      .digest('hex');
  }
}
```

```typescript
// services/user/src/subscribers/user.subscriber.ts
// Auto-encrypt PII fields ก่อน save ลง DB

import {
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
  UpdateEvent,
} from 'typeorm';
import { EncryptionService } from '../../../shared/encryption.service';
import { User } from '../entities/user.entity';

@EventSubscriber()
export class UserEncryptionSubscriber implements EntitySubscriberInterface<User> {
  constructor(private readonly encryptionService: EncryptionService) {}

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>): void {
    this.encryptPIIFields(event.entity);
  }

  beforeUpdate(event: UpdateEvent<User>): void {
    if (event.entity) {
      this.encryptPIIFields(event.entity as User);
    }
  }

  afterLoad(entity: User): void {
    this.decryptPIIFields(entity);
  }

  private encryptPIIFields(user: Partial<User>): void {
    if (user.nationalId) {
      user.nationalIdEncrypted = this.encryptionService.encrypt(user.nationalId);
      user.nationalIdHash = this.encryptionService.hash(user.nationalId);
      delete user.nationalId;
    }
  }

  private decryptPIIFields(user: User): void {
    if (user.nationalIdEncrypted) {
      (user as any).nationalId = this.encryptionService.decrypt(
        user.nationalIdEncrypted
      );
    }
  }
}
```

---

## Right to Erasure Implementation

### Data Deletion Orchestrator

```typescript
// services/user/src/privacy/data-deletion.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class DataDeletionService {
  private readonly logger = new Logger(DataDeletionService.name);

  constructor(
    @InjectRepository(User)
    private readonly userRepo: Repository<User>,
    @InjectRepository(DeletionRequest)
    private readonly deletionRequestRepo: Repository<DeletionRequest>,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async requestDeletion(
    userId: string,
    reason: string,
  ): Promise<DeletionRequest> {
    // ตรวจสอบว่ามี active orders หรือไม่
    const hasActiveOrders = await this.checkActiveOrders(userId);
    if (hasActiveOrders) {
      throw new ConflictException(
        'Cannot delete account with active orders. ' +
        'Please wait for all orders to complete.'
      );
    }

    const deletionRequest = await this.deletionRequestRepo.save({
      userId,
      reason,
      requestedAt: new Date(),
      status: 'PENDING',
      scheduledAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
    });

    // Notify all services that store user data
    await this.eventEmitter.emit('privacy.deletion.requested', {
      userId,
      deletionRequestId: deletionRequest.id,
      scheduledAt: deletionRequest.scheduledAt,
    });

    // Send confirmation email to user
    await this.eventEmitter.emit('notification.email.send', {
      to: await this.getUserEmail(userId),
      template: 'data-deletion-confirmation',
      data: {
        scheduledDate: deletionRequest.scheduledAt,
        cancellationDeadline: new Date(
          deletionRequest.scheduledAt.getTime() - 7 * 24 * 60 * 60 * 1000
        ),
      },
    });

    return deletionRequest;
  }

  async executeDeletion(deletionRequestId: string): Promise<void> {
    const request = await this.deletionRequestRepo.findOne({
      where: { id: deletionRequestId },
    });

    if (!request || request.status !== 'SCHEDULED') {
      throw new BadRequestException('Invalid deletion request');
    }

    try {
      // 1. Anonymize user record (เก็บ aggregated data แต่ลบ PII)
      await this.anonymizeUser(request.userId);

      // 2. Emit event ให้ทุก services ลบข้อมูล
      await this.eventEmitter.emit('privacy.deletion.execute', {
        userId: request.userId,
        deletionRequestId: request.id,
      });

      // 3. Mark as completed
      await this.deletionRequestRepo.update(request.id, {
        status: 'COMPLETED',
        completedAt: new Date(),
      });

      this.logger.log(
        `Data deletion completed for user ${request.userId}`
      );
    } catch (error) {
      await this.deletionRequestRepo.update(request.id, {
        status: 'FAILED',
        failureReason: error.message,
      });
      throw error;
    }
  }

  private async anonymizeUser(userId: string): Promise<void> {
    const anonymizedEmail = `deleted-${userId}@anonymized.invalid`;
    
    await this.userRepo.update(userId, {
      firstName: 'Deleted',
      lastName: 'User',
      email: anonymizedEmail,
      phone: null,
      nationalIdEncrypted: null,
      nationalIdHash: null,
      homeAddress: null,
      dateOfBirth: null,
      profilePhoto: null,
      anonymizedAt: new Date(),
      isAnonymized: true,
    });
  }
}
```

```typescript
// services/order/src/listeners/privacy.listener.ts
// Order Service รับ event และ anonymize order data ที่เกี่ยวข้อง

import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class PrivacyListener {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepo: Repository<Order>,
  ) {}

  @OnEvent('privacy.deletion.execute')
  async handleDeletion(event: PrivacyDeletionEvent): Promise<void> {
    // Anonymize PII ใน orders แต่เก็บ business data ไว้
    await this.orderRepo.update(
      { userId: event.userId },
      {
        userName: 'Deleted User',
        userEmail: `deleted-${event.userId}@anonymized.invalid`,
        shippingName: 'Deleted User',
        shippingPhone: null,
        // ไม่ลบ: orderId, amount, items, timestamps
        // เพราะต้องการสำหรับ financial records
      }
    );

    // Acknowledge deletion
    await this.eventEmitter.emit('privacy.deletion.ack', {
      deletionRequestId: event.deletionRequestId,
      service: 'order-service',
      recordsAnonymized: count,
    });
  }
}
```

---

## Consent Management

```typescript
// services/user/src/privacy/consent.service.ts

export enum ConsentType {
  MARKETING = 'marketing',
  ANALYTICS = 'analytics',
  THIRD_PARTY_SHARING = 'third_party_sharing',
  PERSONALIZATION = 'personalization',
}

@Injectable()
export class ConsentService {
  constructor(
    @InjectRepository(ConsentRecord)
    private readonly consentRepo: Repository<ConsentRecord>,
  ) {}

  async updateConsent(
    userId: string,
    consentType: ConsentType,
    given: boolean,
    ipAddress: string,
    userAgent: string,
  ): Promise<ConsentRecord> {
    // บันทึกประวัติ consent ทุกครั้ง (สำหรับ audit)
    const record = await this.consentRepo.save({
      userId,
      consentType,
      given,
      givenAt: given ? new Date() : null,
      withdrawnAt: !given ? new Date() : null,
      // บันทึก context สำหรับ proof
      metadata: {
        ipAddress,
        userAgent,
        consentVersion: '2.0', // version ของ consent form
        source: 'user_settings',
      },
    });

    // Emit event สำหรับ services ที่ต้องปรับพฤติกรรมตาม consent
    await this.eventEmitter.emit('consent.updated', {
      userId,
      consentType,
      given,
      recordId: record.id,
    });

    return record;
  }

  async getUserConsents(userId: string): Promise<UserConsentSummary> {
    // ดึง latest consent สำหรับแต่ละ type
    const consents = await this.consentRepo
      .createQueryBuilder('c')
      .where('c.userId = :userId', { userId })
      .distinctOn(['c.consentType'])
      .orderBy('c.consentType')
      .addOrderBy('c.createdAt', 'DESC')
      .getMany();

    return {
      userId,
      consents: consents.reduce((acc, c) => {
        acc[c.consentType] = {
          given: c.given,
          updatedAt: (c.given ? c.givenAt : c.withdrawnAt) || c.createdAt,
        };
        return acc;
      }, {} as Record<string, { given: boolean; updatedAt: Date }>),
    };
  }
}
```

---

## Audit Logging for Compliance

```typescript
// shared/src/audit/audit.service.ts

export enum AuditAction {
  // Data access
  DATA_ACCESS = 'DATA_ACCESS',
  DATA_EXPORT = 'DATA_EXPORT',
  
  // Data modification
  DATA_CREATE = 'DATA_CREATE',
  DATA_UPDATE = 'DATA_UPDATE',
  DATA_DELETE = 'DATA_DELETE',
  DATA_ANONYMIZE = 'DATA_ANONYMIZE',
  
  // Authentication
  AUTH_LOGIN = 'AUTH_LOGIN',
  AUTH_LOGOUT = 'AUTH_LOGOUT',
  AUTH_FAILED = 'AUTH_FAILED',
  AUTH_MFA = 'AUTH_MFA',
  
  // Administrative
  ADMIN_USER_ACCESS = 'ADMIN_USER_ACCESS',
  PERMISSION_CHANGE = 'PERMISSION_CHANGE',
  
  // Privacy
  CONSENT_UPDATE = 'CONSENT_UPDATE',
  DELETION_REQUEST = 'DELETION_REQUEST',
  DATA_BREACH_REPORT = 'DATA_BREACH_REPORT',
}

@Injectable()
export class AuditService {
  constructor(
    @InjectRepository(AuditLog)
    private readonly auditRepo: Repository<AuditLog>,
  ) {}

  async log(params: AuditLogParams): Promise<void> {
    await this.auditRepo.save({
      action: params.action,
      actorId: params.actorId,
      actorType: params.actorType, // 'user' | 'service' | 'admin'
      targetId: params.targetId,
      targetType: params.targetType, // 'user' | 'order' | 'payment'
      targetDataClassification: params.dataClassification,
      
      // Request context
      ipAddress: params.ipAddress,
      userAgent: params.userAgent,
      sessionId: params.sessionId,
      requestId: params.requestId,
      
      // Changes (for UPDATE actions)
      changedFields: params.changedFields,
      
      // Outcome
      success: params.success,
      failureReason: params.failureReason,
      
      // Compliance metadata
      legalBasis: params.legalBasis, // 'consent' | 'contract' | 'legal_obligation'
      dataResidency: params.dataResidency, // 'TH' | 'SG' | 'US'
      
      createdAt: new Date(),
    });
  }
}
```

```typescript
// shared/src/audit/audit.interceptor.ts

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class AuditInterceptor implements NestInterceptor {
  constructor(
    private readonly auditService: AuditService,
    private readonly reflector: Reflector,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const auditConfig = this.reflector.getAllAndOverride<AuditConfig>(
      'audit',
      [context.getHandler(), context.getClass()]
    );

    if (!auditConfig) return next.handle();

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const startTime = Date.now();

    return next.handle().pipe(
      tap({
        next: (result) => {
          this.auditService.log({
            action: auditConfig.action,
            actorId: user?.id,
            actorType: user ? 'user' : 'anonymous',
            targetId: request.params?.id || result?.id,
            targetType: auditConfig.targetType,
            dataClassification: auditConfig.dataClassification,
            ipAddress: request.ip,
            userAgent: request.headers['user-agent'],
            requestId: request.headers['x-request-id'],
            success: true,
            legalBasis: auditConfig.legalBasis,
          });
        },
        error: (error) => {
          this.auditService.log({
            action: auditConfig.action,
            actorId: user?.id,
            actorType: user ? 'user' : 'anonymous',
            targetType: auditConfig.targetType,
            ipAddress: request.ip,
            requestId: request.headers['x-request-id'],
            success: false,
            failureReason: error.message,
          });
        },
      })
    );
  }
}

// Decorator สำหรับใช้ใน controllers
export const Audit = (config: AuditConfig) =>
  SetMetadata('audit', config);

// Usage:
// @Audit({ action: AuditAction.DATA_ACCESS, targetType: 'user', dataClassification: 'CONFIDENTIAL' })
// @Get(':id')
// async getUser(...) {}
```

---

## Data Residency

```yaml
# kubernetes/configs/data-residency.yaml
# กำหนด region สำหรับแต่ละประเภทข้อมูล

apiVersion: v1
kind: ConfigMap
metadata:
  name: data-residency-config
data:
  # Thai users' PII ต้องเก็บใน Thailand (PDPA requirement)
  thai_pii_region: "ap-southeast-7"  # AWS ap-southeast-7 = Thailand
  thai_pii_db: "mysql-th-primary.company.internal"
  
  # Analytics data สามารถเก็บ globally
  analytics_region: "ap-southeast-1"  # Singapore
  
  # Backup regions ต้องอยู่ใน approved list
  thai_pii_backup_regions: "ap-southeast-7-b,ap-southeast-7-c"
```

```typescript
// shared/src/residency/data-residency.service.ts

@Injectable()
export class DataResidencyService {
  // กำหนด database connection ตาม user's country
  async getConnectionForUser(userId: string): Promise<DataSource> {
    const userCountry = await this.getUserCountry(userId);
    
    switch (userCountry) {
      case 'TH':
        // Thai users - ข้อมูลต้องอยู่ใน Thailand
        return this.thaiDataSource;
      
      case 'EU':
        // EU users - GDPR compliance
        return this.euDataSource;
      
      default:
        return this.defaultDataSource;
    }
  }
}
```

---

## Data Retention Policies

```typescript
// shared/src/retention/retention-policy.ts

export const RETENTION_POLICIES = {
  // Financial data - 7 years (กฎหมายภาษีไทย)
  transactions: { years: 7, anonymizeAfter: false },
  invoices: { years: 7, anonymizeAfter: false },
  
  // Order data - 3 years
  orders: { years: 3, anonymizeAfter: true },
  
  // User profiles - จนกว่าจะขอลบ + 30 days
  userProfiles: { days: 30, afterDeletionRequest: true },
  
  // Analytics - 2 years (anonymized)
  analyticsEvents: { years: 2, anonymizeAfter: true },
  
  // Audit logs - 3 years (compliance)
  auditLogs: { years: 3, anonymizeAfter: false },
  
  // Application logs - 90 days
  applicationLogs: { days: 90, anonymizeAfter: true },
  
  // Session data - 24 hours
  sessions: { hours: 24, anonymizeAfter: false },
};

// Cron job สำหรับ cleanup
@Injectable()
export class DataRetentionJob {
  @Cron('0 2 * * *') // Run at 2 AM daily
  async enforceRetentionPolicies(): Promise<void> {
    for (const [dataType, policy] of Object.entries(RETENTION_POLICIES)) {
      await this.processRetentionPolicy(dataType, policy);
    }
  }

  private async processRetentionPolicy(
    dataType: string,
    policy: RetentionPolicy,
  ): Promise<void> {
    const cutoffDate = this.calculateCutoffDate(policy);
    
    if (policy.anonymizeAfter) {
      await this.anonymizeExpiredData(dataType, cutoffDate);
    } else {
      await this.deleteExpiredData(dataType, cutoffDate);
    }
  }
}
```

---

## Breach Notification Procedure

```typescript
// services/security/src/breach/breach-notification.service.ts

@Injectable()
export class BreachNotificationService {
  // PDPA กำหนด: ต้องแจ้ง PDPC ภายใน 72 ชั่วโมง
  // GDPR กำหนด: ต้องแจ้ง DPA ภายใน 72 ชั่วโมง

  async reportBreach(breach: BreachReport): Promise<void> {
    // 1. Log breach ทันที
    await this.auditService.log({
      action: AuditAction.DATA_BREACH_REPORT,
      actorId: breach.reportedBy,
      targetType: 'breach',
      success: true,
    });

    // 2. Alert security team ทันที
    await this.alertSecurityTeam(breach);

    // 3. Schedule regulatory notifications
    const notificationDeadline = new Date(
      breach.discoveredAt.getTime() + 72 * 60 * 60 * 1000 // 72 hours
    );

    await this.scheduleRegulatoryNotification({
      breach,
      deadline: notificationDeadline,
      authorities: breach.affectedRegions.includes('TH') 
        ? ['PDPC_THAILAND'] 
        : [],
    });

    // 4. ถ้ามีความเสี่ยงสูง - แจ้ง users ที่ได้รับผลกระทบ
    if (breach.riskLevel === 'HIGH') {
      await this.notifyAffectedUsers(breach);
    }
  }

  private async notifyAffectedUsers(breach: BreachReport): Promise<void> {
    const affectedUserIds = await this.identifyAffectedUsers(breach);
    
    for (const userId of affectedUserIds) {
      await this.emailService.send({
        to: await this.getUserEmail(userId),
        subject: 'Important Security Notice Regarding Your Account',
        template: 'breach-notification',
        data: {
          breachDate: breach.discoveredAt,
          affectedData: breach.dataTypesAffected,
          recommendedActions: [
            'Change your password immediately',
            'Enable two-factor authentication',
            'Monitor your account for suspicious activity',
          ],
        },
      });
    }
  }
}
```

---

## PDPA Compliance Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PDPA COMPLIANCE CHECKLIST                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Legal Basis                                                        │
│  [ ] ระบุ legal basis สำหรับการเก็บข้อมูลแต่ละประเภท              │
│  [ ] มี consent mechanism ที่ชัดเจน granular                       │
│  [ ] Consent เป็น freely given, specific, informed, unambiguous    │
│  [ ] มีวิธี withdraw consent ที่ง่ายพอๆ กับการให้ consent           │
│                                                                     │
│  Data Subject Rights                                                │
│  [ ] Right to Access - API พร้อม                                   │
│  [ ] Right to Rectification - ผู้ใช้แก้ข้อมูลได้                   │
│  [ ] Right to Erasure - deletion flow ทำงาน                        │
│  [ ] Right to Portability - export ได้ใน 30 วัน                    │
│  [ ] Right to Restriction - สามารถ pause processing ได้             │
│  [ ] Right to Object - opt-out marketing ได้                       │
│                                                                     │
│  Technical Measures                                                 │
│  [ ] Encryption at rest (AES-256)                                  │
│  [ ] Encryption in transit (TLS 1.3)                               │
│  [ ] PII redacted ใน logs                                          │
│  [ ] Data minimization implemented                                  │
│  [ ] Retention policies enforced                                   │
│  [ ] Audit logging สำหรับ PII access                               │
│                                                                     │
│  Organizational                                                     │
│  [ ] มี DPO (Data Protection Officer) หรือ ผู้รับผิดชอบ           │
│  [ ] Privacy Policy อัพเดทแล้ว                                     │
│  [ ] DPIA (Data Protection Impact Assessment) สำหรับ high-risk     │
│  [ ] Vendor assessment สำหรับ third-party processors               │
│  [ ] Breach response procedure พร้อม                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## สรุป

Privacy by Design ไม่ใช่แค่ feature เพิ่มเติม แต่เป็น foundation ของระบบที่ดี

| ด้าน | Implementation |
|------|----------------|
| Data Classification | Label ทุก field ตาม sensitivity |
| Encryption | AES-256-GCM for PII, TLS 1.3 in transit |
| Consent | Granular, withdrawable, audited |
| Right to Erasure | Anonymization workflow + all services |
| Audit Logging | ทุก PII access ต้อง log |
| Data Retention | Auto-cleanup policies |
| Breach Response | 72-hour notification procedure |

สิ่งสำคัญที่สุดคือ **Privacy by Design** — ออกแบบ privacy ตั้งแต่ต้น ไม่ใช่ add-on ทีหลัง เพราะการแก้ไขภายหลังมีต้นทุนสูงกว่ามาก ทั้งในแง่ technical debt และความเสี่ยงทาง legal
