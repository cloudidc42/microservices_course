# Part 84: Data Privacy and GDPR

## บทนำ

GDPR (General Data Protection Regulation) เป็นกฎหมายคุ้มครองข้อมูลส่วนบุคคลของสหภาพยุโรปที่มีผลบังคับใช้ตั้งแต่ปี 2018 สำหรับระบบ Microservices การปฏิบัติตาม GDPR ต้องการการออกแบบที่ระมัดระวังตั้งแต่แรก ในบทนี้เราจะเรียนรู้วิธีการนำหลัก Privacy by Design มาใช้ในระบบ Microservices

---

## 1. GDPR Compliance Architecture

### 1.1 หลักการพื้นฐาน GDPR

```typescript
// src/gdpr/principles.ts

/**
 * GDPR 7 หลักการพื้นฐาน:
 * 1. Lawfulness, Fairness, Transparency - ถูกกฎหมาย ยุติธรรม โปร่งใส
 * 2. Purpose Limitation - ใช้ข้อมูลตามวัตถุประสงค์ที่แจ้ง
 * 3. Data Minimisation - เก็บข้อมูลเท่าที่จำเป็น
 * 4. Accuracy - ข้อมูลต้องถูกต้อง
 * 5. Storage Limitation - เก็บไม่นานเกินจำเป็น
 * 6. Integrity & Confidentiality - ความปลอดภัย
 * 7. Accountability - ความรับผิดชอบ
 */

export interface GDPRConfig {
  dataRetentionDays: number;
  consentRequired: boolean;
  lawfulBasis: LawfulBasis;
  dataCategories: DataCategory[];
  processingPurposes: ProcessingPurpose[];
}

export type LawfulBasis = 
  | 'consent'
  | 'contract'
  | 'legal_obligation'
  | 'vital_interests'
  | 'public_task'
  | 'legitimate_interests';

export type DataCategory = 
  | 'basic_identity'       // ชื่อ, อีเมล
  | 'contact'              // ที่อยู่, โทรศัพท์
  | 'financial'            // ข้อมูลการเงิน
  | 'health'               // ข้อมูลสุขภาพ (sensitive)
  | 'biometric'            // ข้อมูลชีวมิติ (sensitive)
  | 'location'             // ข้อมูลตำแหน่ง
  | 'behavioral'           // พฤติกรรม
  | 'technical';           // IP, device info

export type ProcessingPurpose = 
  | 'service_delivery'
  | 'analytics'
  | 'marketing'
  | 'legal_compliance'
  | 'fraud_prevention';
```

### 1.2 Data Flow Mapping

```typescript
// src/gdpr/data-flow-map.ts
interface DataFlow {
  source: string;
  destination: string;
  dataTypes: DataCategory[];
  lawfulBasis: LawfulBasis;
  encrypted: boolean;
  crossBorder: boolean;
  retentionDays: number;
}

export const DATA_FLOW_MAP: DataFlow[] = [
  {
    source: 'UserService',
    destination: 'OrderService',
    dataTypes: ['basic_identity', 'contact'],
    lawfulBasis: 'contract',
    encrypted: true,
    crossBorder: false,
    retentionDays: 365,
  },
  {
    source: 'UserService',
    destination: 'Analytics',
    dataTypes: ['behavioral', 'technical'],
    lawfulBasis: 'consent',
    encrypted: true,
    crossBorder: true,
    retentionDays: 90,
  },
  {
    source: 'PaymentService',
    destination: 'AuditLog',
    dataTypes: ['financial'],
    lawfulBasis: 'legal_obligation',
    encrypted: true,
    crossBorder: false,
    retentionDays: 2555, // 7 years
  },
];
```

---

## 2. Data Classification and Tagging

### 2.1 Data Classification System

```typescript
// src/gdpr/classification.ts
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

export enum DataSensitivity {
  PUBLIC = 'public',
  INTERNAL = 'internal',
  CONFIDENTIAL = 'confidential',
  SENSITIVE = 'sensitive',
  SPECIAL_CATEGORY = 'special_category', // GDPR Article 9
}

export interface DataField {
  name: string;
  sensitivity: DataSensitivity;
  pii: boolean;           // Personally Identifiable Information
  encrypted: boolean;     // ต้องเข้ารหัส?
  masked: boolean;        // ต้อง mask เมื่อ log?
  retention: number;      // วันที่เก็บ
  purpose: ProcessingPurpose[];
}

// Schema สำหรับ User entity พร้อม classification
@Entity('users')
export class UserEntity {
  @Column({ comment: 'sensitivity:internal,pii:false' })
  id: string;

  @Column({ comment: 'sensitivity:confidential,pii:true,masked:true' })
  name: string;

  @Column({ comment: 'sensitivity:confidential,pii:true,masked:true' })
  email: string;

  @Column({ 
    comment: 'sensitivity:sensitive,pii:true,encrypted:true,masked:true' 
  })
  phoneNumber?: string;

  @Column({ 
    comment: 'sensitivity:special_category,pii:true,encrypted:true' 
  })
  healthData?: string;
  
  @Column({ comment: 'sensitivity:internal,pii:false' })
  createdAt: Date;
}

// Data catalog
export const USER_DATA_CATALOG: Record<string, DataField> = {
  id: {
    name: 'id',
    sensitivity: DataSensitivity.INTERNAL,
    pii: false,
    encrypted: false,
    masked: false,
    retention: 3650,
    purpose: ['service_delivery'],
  },
  name: {
    name: 'name',
    sensitivity: DataSensitivity.CONFIDENTIAL,
    pii: true,
    encrypted: false,
    masked: true,
    retention: 365,
    purpose: ['service_delivery'],
  },
  email: {
    name: 'email',
    sensitivity: DataSensitivity.CONFIDENTIAL,
    pii: true,
    encrypted: false,
    masked: true,
    retention: 365,
    purpose: ['service_delivery', 'marketing'],
  },
  phoneNumber: {
    name: 'phoneNumber',
    sensitivity: DataSensitivity.SENSITIVE,
    pii: true,
    encrypted: true,
    masked: true,
    retention: 365,
    purpose: ['service_delivery'],
  },
};
```

### 2.2 Automatic Data Tagging Middleware

```typescript
// src/gdpr/tagging-middleware.ts
import { Request, Response, NextFunction } from 'express';

export function piiTaggingMiddleware(req: Request, res: Response, next: NextFunction) {
  const originalJson = res.json.bind(res);
  
  res.json = (body: any) => {
    // Tag response headers ว่ามี PII หรือไม่
    const hasPII = containsPII(body);
    
    if (hasPII) {
      res.setHeader('X-Contains-PII', 'true');
      res.setHeader('Cache-Control', 'no-store, private');
      res.setHeader('X-Data-Classification', 'confidential');
    }
    
    return originalJson(body);
  };
  
  next();
}

function containsPII(obj: any, depth = 0): boolean {
  if (depth > 10 || !obj || typeof obj !== 'object') return false;
  
  const PII_FIELDS = ['email', 'phone', 'name', 'address', 'dateOfBirth', 'ssn'];
  
  for (const key of Object.keys(obj)) {
    if (PII_FIELDS.includes(key.toLowerCase())) return true;
    if (typeof obj[key] === 'object' && containsPII(obj[key], depth + 1)) return true;
  }
  
  return false;
}
```

---

## 3. Right to Erasure (RTBF)

### 3.1 Erasure Service Implementation

```typescript
// src/gdpr/erasure.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';

interface ErasureRequest {
  userId: string;
  requestedBy: string;
  reason: string;
  verificationToken: string;
}

interface ErasureResult {
  userId: string;
  erasedAt: Date;
  services: ServiceErasureResult[];
  auditId: string;
}

interface ServiceErasureResult {
  service: string;
  status: 'success' | 'failed' | 'partial' | 'not_applicable';
  recordsDeleted?: number;
  recordsAnonymized?: number;
  error?: string;
}

@Injectable()
export class ErasureService {
  constructor(
    private dataSource: DataSource,
    private eventEmitter: EventEmitter2,
  ) {}

  async processErasureRequest(request: ErasureRequest): Promise<ErasureResult> {
    const auditId = crypto.randomUUID();
    const results: ServiceErasureResult[] = [];
    
    // ตรวจสอบว่ามีข้อมูลค้างอยู่ไหม (เช่น active subscription)
    await this.validateErasureEligibility(request.userId);
    
    // Start transaction สำหรับ local data
    await this.dataSource.transaction(async (manager) => {
      // 1. Anonymize user record (ไม่ลบ เพื่อ referential integrity)
      await manager.query(`
        UPDATE users SET
          name = 'Deleted User',
          email = CONCAT('deleted_', id, '@removed.invalid'),
          phone_number = NULL,
          date_of_birth = NULL,
          address = NULL,
          profile_image = NULL,
          deleted_at = NOW(),
          is_deleted = true,
          deletion_audit_id = $2
        WHERE id = $1
      `, [request.userId, auditId]);
      
      results.push({
        service: 'UserService',
        status: 'success',
        recordsAnonymized: 1,
      });
      
      // 2. Delete session data
      const sessionResult = await manager.query(`
        DELETE FROM user_sessions WHERE user_id = $1
      `, [request.userId]);
      
      // 3. Delete OAuth tokens
      await manager.query(`
        DELETE FROM oauth_tokens WHERE user_id = $1
      `, [request.userId]);
      
      // 4. Anonymize audit logs (เก็บ action แต่ลบ PII)
      await manager.query(`
        UPDATE audit_logs SET
          user_email = NULL,
          user_name = NULL,
          ip_address = '0.0.0.0',
          user_agent = NULL
        WHERE user_id = $1
      `, [request.userId]);
    });
    
    // Publish event สำหรับ other services
    this.eventEmitter.emit('gdpr.erasure.requested', {
      userId: request.userId,
      auditId,
      requestedAt: new Date(),
    });
    
    return {
      userId: request.userId,
      erasedAt: new Date(),
      services: results,
      auditId,
    };
  }

  private async validateErasureEligibility(userId: string): Promise<void> {
    // ตรวจสอบ active contracts/subscriptions
    const activeContracts = await this.dataSource.query(`
      SELECT COUNT(*) as count FROM orders 
      WHERE user_id = $1 AND status IN ('pending', 'processing')
    `, [userId]);
    
    if (parseInt(activeContracts[0].count) > 0) {
      throw new Error('ERASURE_BLOCKED: User has active orders');
    }
    
    // ตรวจสอบ legal hold
    const legalHold = await this.dataSource.query(`
      SELECT COUNT(*) as count FROM legal_holds 
      WHERE user_id = $1 AND expires_at > NOW()
    `, [userId]);
    
    if (parseInt(legalHold[0].count) > 0) {
      throw new Error('ERASURE_BLOCKED: User is under legal hold');
    }
  }
}

// Event handlers ใน other services
@Injectable()
export class OrderServiceErasureHandler {
  @OnEvent('gdpr.erasure.requested')
  async handleErasure(event: { userId: string; auditId: string }) {
    // Anonymize order data
    await this.orderRepository.query(`
      UPDATE orders SET
        shipping_name = 'Deleted User',
        shipping_address = '*** REMOVED ***',
        billing_name = 'Deleted User',
        billing_address = '*** REMOVED ***'
      WHERE user_id = $1
    `, [event.userId]);
  }
}
```

---

## 4. Data Portability (Data Export)

### 4.1 Data Export Service

```typescript
// src/gdpr/export.service.ts
import { Injectable } from '@nestjs/common';
import archiver from 'archiver';
import { createWriteStream } from 'fs';
import { join } from 'path';

interface ExportRequest {
  userId: string;
  format: 'json' | 'csv' | 'xml';
  includeCategories: DataCategory[];
}

interface ExportResult {
  downloadUrl: string;
  expiresAt: Date;
  fileSize: number;
  recordCount: number;
}

@Injectable()
export class DataExportService {
  async exportUserData(request: ExportRequest): Promise<ExportResult> {
    const exportId = crypto.randomUUID();
    const exportDir = `/tmp/exports/${exportId}`;
    
    // Collect data from all services
    const userData = await this.collectUserData(request.userId, request.includeCategories);
    
    // Create archive
    const archivePath = await this.createArchive(exportDir, userData, request.format);
    
    // Upload to secure storage
    const downloadUrl = await this.uploadToStorage(archivePath, exportId);
    
    // Log export for audit
    await this.auditLog.record({
      action: 'DATA_EXPORT',
      userId: request.userId,
      exportId,
      categories: request.includeCategories,
    });
    
    return {
      downloadUrl,
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
      fileSize: 0, // จะอัปเดตจาก archive
      recordCount: this.countRecords(userData),
    };
  }

  private async collectUserData(userId: string, categories: DataCategory[]) {
    const data: Record<string, any> = {};
    
    if (categories.includes('basic_identity') || categories.includes('contact')) {
      data.profile = await this.userRepository.findOne({
        where: { id: userId },
        select: ['id', 'name', 'email', 'phone', 'createdAt'],
      });
    }
    
    if (categories.includes('behavioral')) {
      data.orders = await this.orderRepository.find({
        where: { userId },
        select: ['id', 'items', 'total', 'status', 'createdAt'],
      });
      
      data.activityLog = await this.activityRepository.find({
        where: { userId },
        take: 1000,
        order: { createdAt: 'DESC' },
      });
    }
    
    if (categories.includes('financial')) {
      data.payments = await this.paymentRepository.find({
        where: { userId },
        select: ['id', 'amount', 'currency', 'status', 'createdAt'],
      });
    }
    
    return data;
  }

  private async createArchive(
    dir: string, 
    data: Record<string, any>, 
    format: 'json' | 'csv' | 'xml'
  ): Promise<string> {
    const archivePath = `${dir}.zip`;
    const output = createWriteStream(archivePath);
    const archive = archiver('zip', { zlib: { level: 9 } });
    
    archive.pipe(output);
    
    for (const [key, value] of Object.entries(data)) {
      if (format === 'json') {
        archive.append(
          JSON.stringify(value, null, 2), 
          { name: `${key}.json` }
        );
      } else if (format === 'csv') {
        const csv = this.convertToCSV(value);
        archive.append(csv, { name: `${key}.csv` });
      }
    }
    
    // Add README
    archive.append(
      this.generateReadme(data),
      { name: 'README.txt' }
    );
    
    await archive.finalize();
    
    return archivePath;
  }

  private generateReadme(data: Record<string, any>): string {
    return `
DATA EXPORT - ${new Date().toISOString()}
=============================================

This archive contains your personal data as required by GDPR Article 20.

CONTENTS:
${Object.keys(data).map(k => `- ${k}.json: Your ${k} data`).join('\n')}

DATA RETENTION:
This export link expires in 24 hours.

CONTACT:
If you have questions about your data, contact: privacy@example.com
`;
  }

  private convertToCSV(data: any[]): string {
    if (!Array.isArray(data) || data.length === 0) return '';
    
    const headers = Object.keys(data[0]);
    const rows = data.map(row => 
      headers.map(h => {
        const val = row[h];
        if (typeof val === 'string' && val.includes(',')) {
          return `"${val.replace(/"/g, '""')}"`;
        }
        return val ?? '';
      }).join(',')
    );
    
    return [headers.join(','), ...rows].join('\n');
  }

  private countRecords(data: Record<string, any>): number {
    return Object.values(data).reduce((sum, v) => {
      return sum + (Array.isArray(v) ? v.length : 1);
    }, 0);
  }
}
```

---

## 5. Consent Management Service

### 5.1 Consent Service

```typescript
// src/gdpr/consent.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, Index } from 'typeorm';

@Entity('consent_records')
@Index(['userId', 'purpose'])
export class ConsentRecord {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  @Index()
  userId: string;

  @Column()
  purpose: string;

  @Column()
  version: string;     // version ของ consent form

  @Column()
  granted: boolean;

  @Column({ type: 'jsonb', nullable: true })
  context: {
    ipAddress: string;
    userAgent: string;
    consentSource: string;  // 'registration', 'settings', 'popup'
  };

  @CreateDateColumn()
  createdAt: Date;

  @Column({ nullable: true })
  revokedAt?: Date;

  @Column({ nullable: true })
  expiresAt?: Date;
}

@Injectable()
export class ConsentService {
  constructor(
    @InjectRepository(ConsentRecord)
    private consentRepo: Repository<ConsentRecord>,
  ) {}

  async grantConsent(
    userId: string,
    purpose: ProcessingPurpose,
    context: { ipAddress: string; userAgent: string; source: string }
  ): Promise<ConsentRecord> {
    // ยกเลิก consent เดิมก่อน
    await this.consentRepo.update(
      { userId, purpose, revokedAt: null as any },
      { revokedAt: new Date() }
    );
    
    const record = this.consentRepo.create({
      userId,
      purpose,
      version: await this.getCurrentConsentVersion(purpose),
      granted: true,
      context: {
        ipAddress: context.ipAddress,
        userAgent: context.userAgent,
        consentSource: context.source,
      },
      expiresAt: this.calculateExpiry(purpose),
    });
    
    return this.consentRepo.save(record);
  }

  async revokeConsent(userId: string, purpose: ProcessingPurpose): Promise<void> {
    await this.consentRepo.update(
      { userId, purpose, revokedAt: null as any },
      { revokedAt: new Date(), granted: false }
    );
    
    // Trigger downstream effects
    await this.handleConsentRevocation(userId, purpose);
  }

  async hasConsent(userId: string, purpose: ProcessingPurpose): Promise<boolean> {
    const record = await this.consentRepo.findOne({
      where: { userId, purpose, granted: true, revokedAt: null as any },
      order: { createdAt: 'DESC' },
    });
    
    if (!record) return false;
    
    // ตรวจสอบ expiry
    if (record.expiresAt && record.expiresAt < new Date()) {
      return false;
    }
    
    // ตรวจสอบว่า consent version ยังล่าสุดหรือไม่
    const currentVersion = await this.getCurrentConsentVersion(purpose);
    if (record.version !== currentVersion) {
      return false;
    }
    
    return true;
  }

  async getConsentHistory(userId: string): Promise<ConsentRecord[]> {
    return this.consentRepo.find({
      where: { userId },
      order: { createdAt: 'DESC' },
    });
  }

  private async handleConsentRevocation(
    userId: string, 
    purpose: ProcessingPurpose
  ): Promise<void> {
    switch (purpose) {
      case 'marketing':
        // Unsubscribe from mailing list
        await this.marketingService.unsubscribe(userId);
        break;
      case 'analytics':
        // Delete analytics data
        await this.analyticsService.deleteUserData(userId);
        break;
    }
  }

  private calculateExpiry(purpose: ProcessingPurpose): Date | undefined {
    const CONSENT_DURATIONS: Partial<Record<ProcessingPurpose, number>> = {
      marketing: 365,  // 1 year
      analytics: 180,  // 6 months
    };
    
    const days = CONSENT_DURATIONS[purpose];
    if (!days) return undefined;
    
    return new Date(Date.now() + days * 24 * 60 * 60 * 1000);
  }

  private async getCurrentConsentVersion(purpose: string): Promise<string> {
    // ดึง version ล่าสุดจาก config
    return process.env[`CONSENT_VERSION_${purpose.toUpperCase()}`] || '1.0';
  }
}
```

---

## 6. Data Masking and Anonymization

### 6.1 Masking Functions

```typescript
// src/gdpr/masking.ts

export class DataMasker {
  // Mask email: john@example.com -> j***@e******.com
  static maskEmail(email: string): string {
    const [local, domain] = email.split('@');
    const [domainName, ...tlds] = domain.split('.');
    
    const maskedLocal = local[0] + '*'.repeat(Math.max(local.length - 1, 3));
    const maskedDomain = domainName[0] + '*'.repeat(Math.max(domainName.length - 1, 5));
    
    return `${maskedLocal}@${maskedDomain}.${tlds.join('.')}`;
  }

  // Mask phone: +66812345678 -> +668****678
  static maskPhone(phone: string): string {
    if (phone.length <= 4) return '****';
    return phone.slice(0, 4) + '*'.repeat(phone.length - 7) + phone.slice(-3);
  }

  // Mask name: John Doe -> J*** D**
  static maskName(name: string): string {
    return name.split(' ')
      .map(part => part[0] + '*'.repeat(Math.max(part.length - 1, 2)))
      .join(' ');
  }

  // Mask credit card: 4111111111111111 -> ****-****-****-1111
  static maskCreditCard(card: string): string {
    const cleaned = card.replace(/\D/g, '');
    return `****-****-****-${cleaned.slice(-4)}`;
  }

  // Mask IP address: 192.168.1.100 -> 192.168.1.xxx
  static maskIP(ip: string): string {
    const parts = ip.split('.');
    if (parts.length === 4) {
      return `${parts[0]}.${parts[1]}.${parts[2]}.xxx`;
    }
    // IPv6
    return ip.split(':').slice(0, 4).join(':') + ':xxxx:xxxx:xxxx:xxxx';
  }

  // Anonymize date (keep year and month only)
  static anonymizeDate(date: Date): string {
    return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`;
  }

  // Generalize location
  static generalizeLocation(lat: number, lng: number, precision: number = 1): {
    lat: number;
    lng: number;
  } {
    const factor = Math.pow(10, precision);
    return {
      lat: Math.round(lat * factor) / factor,
      lng: Math.round(lng * factor) / factor,
    };
  }

  // k-anonymity: generalize age to range
  static generalizeAge(age: number, bucketSize: number = 5): string {
    const bucket = Math.floor(age / bucketSize) * bucketSize;
    return `${bucket}-${bucket + bucketSize - 1}`;
  }
}

// Log masking middleware
export function createLogMasker() {
  const sensitiveFields = ['password', 'email', 'phone', 'creditCard', 'ssn', 'token'];
  
  return {
    mask: (obj: Record<string, any>): Record<string, any> => {
      const masked = { ...obj };
      
      for (const key of Object.keys(masked)) {
        const lowerKey = key.toLowerCase();
        
        if (sensitiveFields.some(f => lowerKey.includes(f))) {
          masked[key] = '[REDACTED]';
        } else if (lowerKey === 'email' && typeof masked[key] === 'string') {
          masked[key] = DataMasker.maskEmail(masked[key]);
        } else if (typeof masked[key] === 'object' && masked[key] !== null) {
          masked[key] = this.mask(masked[key]);
        }
      }
      
      return masked;
    }
  };
}
```

---

## 7. PII Detection and Handling

### 7.1 PII Detector

```typescript
// src/gdpr/pii-detector.ts

interface PIIDetectionResult {
  hasPII: boolean;
  findings: PIIFinding[];
  riskScore: number;  // 0-100
}

interface PIIFinding {
  type: string;
  value: string;
  maskedValue: string;
  confidence: number;
  location: string;  // JSON path
}

export class PIIDetector {
  private patterns: Record<string, RegExp> = {
    email: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
    phone_th: /(?:\+66|0)[0-9]{8,9}/g,
    phone_intl: /\+[1-9]\d{7,14}/g,
    credit_card: /\b(?:\d{4}[-\s]?){3}\d{4}\b/g,
    thai_id: /[0-9]{13}/g,
    passport: /[A-Z]{1,2}[0-9]{6,9}/g,
    ip_address: /\b(?:\d{1,3}\.){3}\d{1,3}\b/g,
    date_of_birth: /\b\d{4}[-\/]\d{2}[-\/]\d{2}\b/g,
  };

  detect(text: string, path: string = 'root'): PIIDetectionResult {
    const findings: PIIFinding[] = [];
    
    for (const [type, pattern] of Object.entries(this.patterns)) {
      const matches = text.matchAll(pattern);
      
      for (const match of matches) {
        const value = match[0];
        
        // Skip short numbers (likely not real PII)
        if (type === 'thai_id' && !this.validateThaiID(value)) continue;
        
        findings.push({
          type,
          value,
          maskedValue: this.maskValue(type, value),
          confidence: this.calculateConfidence(type, value),
          location: path,
        });
      }
    }
    
    const riskScore = this.calculateRiskScore(findings);
    
    return {
      hasPII: findings.length > 0,
      findings,
      riskScore,
    };
  }

  detectInObject(obj: any, path: string = 'root'): PIIDetectionResult {
    const allFindings: PIIFinding[] = [];
    
    const traverse = (value: any, currentPath: string) => {
      if (typeof value === 'string') {
        const result = this.detect(value, currentPath);
        allFindings.push(...result.findings);
      } else if (Array.isArray(value)) {
        value.forEach((item, i) => traverse(item, `${currentPath}[${i}]`));
      } else if (typeof value === 'object' && value !== null) {
        for (const [key, val] of Object.entries(value)) {
          traverse(val, `${currentPath}.${key}`);
        }
      }
    };
    
    traverse(obj, path);
    
    return {
      hasPII: allFindings.length > 0,
      findings: allFindings,
      riskScore: this.calculateRiskScore(allFindings),
    };
  }

  private validateThaiID(id: string): boolean {
    if (id.length !== 13) return false;
    
    let sum = 0;
    for (let i = 0; i < 12; i++) {
      sum += parseInt(id[i]) * (13 - i);
    }
    
    const checkDigit = (11 - (sum % 11)) % 10;
    return checkDigit === parseInt(id[12]);
  }

  private maskValue(type: string, value: string): string {
    switch (type) {
      case 'email': return DataMasker.maskEmail(value);
      case 'phone_th':
      case 'phone_intl': return DataMasker.maskPhone(value);
      case 'credit_card': return DataMasker.maskCreditCard(value);
      default: return '*'.repeat(value.length);
    }
  }

  private calculateConfidence(type: string, value: string): number {
    const baseConfidence: Record<string, number> = {
      email: 0.95,
      credit_card: 0.90,
      thai_id: 0.85,
      phone_th: 0.80,
      ip_address: 0.70,
    };
    return baseConfidence[type] || 0.50;
  }

  private calculateRiskScore(findings: PIIFinding[]): number {
    if (findings.length === 0) return 0;
    
    const weights: Record<string, number> = {
      credit_card: 30,
      thai_id: 25,
      email: 20,
      phone_th: 15,
      ip_address: 10,
    };
    
    const score = findings.reduce((sum, f) => {
      return sum + (weights[f.type] || 5) * f.confidence;
    }, 0);
    
    return Math.min(100, score);
  }
}
```

---

## 8. Audit Logging for Compliance

```typescript
// src/gdpr/audit.service.ts
import { Injectable } from '@nestjs/common';

interface AuditEvent {
  id: string;
  timestamp: Date;
  action: AuditAction;
  userId?: string;
  actorId: string;        // ผู้ทำ action
  actorType: 'user' | 'system' | 'admin';
  resourceType: string;
  resourceId: string;
  changes?: {
    before: Record<string, any>;
    after: Record<string, any>;
  };
  metadata: {
    ipAddress: string;
    userAgent: string;
    requestId: string;
  };
  legalBasis?: LawfulBasis;
}

type AuditAction = 
  | 'DATA_ACCESS'
  | 'DATA_CREATE'
  | 'DATA_UPDATE'
  | 'DATA_DELETE'
  | 'DATA_EXPORT'
  | 'CONSENT_GRANTED'
  | 'CONSENT_REVOKED'
  | 'ERASURE_REQUESTED'
  | 'ERASURE_COMPLETED'
  | 'DATA_BREACH_DETECTED'
  | 'LOGIN'
  | 'LOGOUT'
  | 'PERMISSION_CHANGE';

@Injectable()
export class GDPRAuditService {
  async log(event: Omit<AuditEvent, 'id' | 'timestamp'>): Promise<void> {
    const auditRecord: AuditEvent = {
      ...event,
      id: crypto.randomUUID(),
      timestamp: new Date(),
    };
    
    // Store in append-only audit log (ห้าม update หรือ delete)
    await this.auditRepository.insert(auditRecord);
    
    // Send to SIEM system
    await this.siemClient.send(auditRecord);
    
    // Alert on high-risk actions
    if (this.isHighRisk(event.action)) {
      await this.alertService.notify({
        type: 'HIGH_RISK_ACTION',
        details: auditRecord,
      });
    }
  }

  async queryAuditLog(filters: {
    userId?: string;
    action?: AuditAction;
    fromDate?: Date;
    toDate?: Date;
    resourceType?: string;
  }): Promise<AuditEvent[]> {
    // สำหรับ Data Subject Access Request (DSAR)
    return this.auditRepository.find({
      where: {
        userId: filters.userId,
        action: filters.action,
      },
      order: { timestamp: 'DESC' },
      take: 1000,
    });
  }

  private isHighRisk(action: AuditAction): boolean {
    const HIGH_RISK_ACTIONS: AuditAction[] = [
      'DATA_DELETE',
      'ERASURE_COMPLETED',
      'DATA_BREACH_DETECTED',
      'PERMISSION_CHANGE',
    ];
    return HIGH_RISK_ACTIONS.includes(action);
  }
}
```

---

## 9. Data Retention Policies

### 9.1 Retention Policy Engine

```typescript
// src/gdpr/retention.ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

interface RetentionPolicy {
  entityName: string;
  retentionDays: number;
  condition?: string;       // SQL WHERE condition
  action: 'delete' | 'anonymize' | 'archive';
  legalBasis?: string;
}

const RETENTION_POLICIES: RetentionPolicy[] = [
  {
    entityName: 'user_sessions',
    retentionDays: 30,
    action: 'delete',
  },
  {
    entityName: 'activity_logs',
    retentionDays: 90,
    action: 'delete',
    condition: 'sensitivity = \'low\'',
  },
  {
    entityName: 'payment_records',
    retentionDays: 2555, // 7 years (legal requirement)
    action: 'archive',
    legalBasis: 'Thai Revenue Code requires 7 year retention',
  },
  {
    entityName: 'marketing_consents',
    retentionDays: 365,
    action: 'delete',
    condition: 'granted = false',
  },
  {
    entityName: 'deleted_users',
    retentionDays: 30,
    action: 'delete',
    condition: 'is_deleted = true',
  },
];

@Injectable()
export class RetentionService {
  // รัน daily ตอน midnight
  @Cron('0 2 * * *') // 2 AM ทุกวัน
  async enforceRetentionPolicies(): Promise<void> {
    console.log('Starting retention policy enforcement...');
    
    for (const policy of RETENTION_POLICIES) {
      try {
        await this.enforcePolicy(policy);
      } catch (error) {
        console.error(`Failed to enforce policy for ${policy.entityName}:`, error);
        await this.auditService.log({
          action: 'DATA_DELETE',
          actorId: 'system',
          actorType: 'system',
          resourceType: policy.entityName,
          resourceId: 'bulk',
          metadata: {
            ipAddress: '127.0.0.1',
            userAgent: 'RetentionService',
            requestId: crypto.randomUUID(),
          },
        });
      }
    }
  }

  private async enforcePolicy(policy: RetentionPolicy): Promise<void> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - policy.retentionDays);
    
    let query = `
      SELECT COUNT(*) as count FROM ${policy.entityName}
      WHERE created_at < $1
    `;
    const params: any[] = [cutoffDate];
    
    if (policy.condition) {
      query += ` AND ${policy.condition}`;
    }
    
    const [{ count }] = await this.dataSource.query(query, params);
    
    if (parseInt(count) === 0) return;
    
    console.log(`Processing ${count} records from ${policy.entityName}`);
    
    switch (policy.action) {
      case 'delete':
        await this.deleteExpiredRecords(policy, cutoffDate);
        break;
      case 'anonymize':
        await this.anonymizeExpiredRecords(policy, cutoffDate);
        break;
      case 'archive':
        await this.archiveExpiredRecords(policy, cutoffDate);
        break;
    }
  }

  private async deleteExpiredRecords(
    policy: RetentionPolicy, 
    cutoffDate: Date
  ): Promise<number> {
    let query = `
      DELETE FROM ${policy.entityName}
      WHERE created_at < $1
    `;
    const params: any[] = [cutoffDate];
    
    if (policy.condition) {
      query += ` AND ${policy.condition}`;
    }
    
    const result = await this.dataSource.query(query, params);
    return result.affected || 0;
  }
}
```

---

## 10. Privacy by Design Principles

### 10.1 Privacy-First Architecture

```typescript
// src/gdpr/privacy-decorator.ts

// Decorator สำหรับ mark methods ที่เข้าถึง PII
function RequiresConsent(purpose: ProcessingPurpose) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const userId = args[0]; // assume first arg is userId
      
      const hasConsent = await this.consentService.hasConsent(userId, purpose);
      if (!hasConsent) {
        throw new Error(`CONSENT_REQUIRED: ${purpose}`);
      }
      
      return originalMethod.apply(this, args);
    };
    
    return descriptor;
  };
}

function AuditDataAccess(action: AuditAction) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const result = await originalMethod.apply(this, args);
      
      await this.auditService.log({
        action,
        actorId: 'system',
        actorType: 'system',
        resourceType: target.constructor.name,
        resourceId: args[0],
        metadata: {
          ipAddress: '127.0.0.1',
          userAgent: 'System',
          requestId: crypto.randomUUID(),
        },
      });
      
      return result;
    };
    
    return descriptor;
  };
}

// ตัวอย่างการใช้งาน
class AnalyticsService {
  @RequiresConsent('analytics')
  @AuditDataAccess('DATA_ACCESS')
  async getUserBehaviorData(userId: string) {
    return this.analyticsRepository.find({ where: { userId } });
  }
}
```

### 10.2 Privacy Configuration

```yaml
# config/privacy.yaml
privacy:
  dataRetention:
    userProfiles: 365    # days
    sessionData: 30
    activityLogs: 90
    paymentRecords: 2555  # 7 years
    marketingData: 365
    
  dataMinimization:
    # Fields ที่ไม่เก็บถ้าไม่จำเป็น
    optional:
      - phoneNumber
      - dateOfBirth
      - address
    
  encryption:
    atRest: true
    algorithm: AES-256-GCM
    fields:
      - phoneNumber
      - nationalId
      - healthData
      
  masking:
    logs:
      - email
      - password
      - creditCard
      - phone
    
  consent:
    required:
      - marketing
      - analytics
      - thirdPartySharing
    optional:
      - serviceImprovement
    
  breachNotification:
    autorityNotificationHours: 72
    subjectNotificationThreshold: "high_risk"
    
  dpia:  # Data Protection Impact Assessment
    required:
      - biometric
      - health
      - financial
      - location_tracking
```

---

## สรุปท้ายบท

| หัวข้อ | กฎ GDPR | Implementation |
|--------|---------|----------------|
| Data Classification | Article 4 | Data catalog, sensitivity tags |
| Right to Erasure | Article 17 | Anonymization + cascade events |
| Data Portability | Article 20 | JSON/CSV export with archive |
| Consent Management | Article 7 | Versioned consent records |
| Data Masking | Article 25 | PII masking in logs/responses |
| PII Detection | Article 25 | Pattern matching + ML |
| Audit Logging | Article 30 | Immutable append-only logs |
| Data Retention | Article 5 | Policy engine + automated cleanup |
| Privacy by Design | Article 25 | Decorators, middleware, config |
| Breach Notification | Article 33 | 72-hour authority notification |

### Checklist GDPR Compliance

- [ ] Data inventory และ flow mapping ครบถ้วน
- [ ] Consent mechanism สำหรับทุก processing purpose
- [ ] RTBF (Right to be Forgotten) ทำงานได้ครบทุก service
- [ ] Data export ใน machine-readable format
- [ ] Audit log แบบ immutable
- [ ] Data retention policies บังคับใช้อัตโนมัติ
- [ ] Encryption at rest สำหรับ sensitive fields
- [ ] PII masking ใน logs
- [ ] DPA (Data Processing Agreement) กับ third parties
- [ ] Privacy policy อัปเดตและเข้าถึงได้
