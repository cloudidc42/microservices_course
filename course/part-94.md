# Part 94: Case Studies - Food Delivery Platform

## แพลตฟอร์มส่งอาหาร (Food Delivery Platform) - กรณีศึกษา

ในบทนี้เราจะออกแบบและสร้างแพลตฟอร์มส่งอาหารในรูปแบบ Microservices คล้ายกับ Grab Food หรือ Foodpanda

---

## 1. สถาปัตยกรรมภาพรวม

```
┌──────────────────────────────────────────────────────────────────┐
│                  Food Delivery Platform Architecture              │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Mobile App / Web ──► API Gateway (Kong)                          │
│                              │                                     │
│         ┌────────────────────┼────────────────────┐               │
│         ▼                    ▼                    ▼               │
│   Restaurant Service    Order Service       User Service          │
│         │                    │                    │               │
│         ▼                    ▼                    ▼               │
│   Menu Service         Payment Service      Driver Service        │
│                              │                    │               │
│                              ▼                    ▼               │
│                     Delivery Assignment    Location Service       │
│                              │                    │               │
│                              └────────────────────┘               │
│                                        │                          │
│                              Tracking Service                     │
│                                        │                          │
│                          Rating & Review Service                  │
│                                        │                          │
│                          Push Notification Service                │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### เทคโนโลยีหลักที่ใช้

| Service | Technology |
|---------|-----------|
| API Gateway | Kong + Rate Limiting |
| Order Service | NestJS + PostgreSQL |
| Restaurant Service | NestJS + MongoDB |
| Driver Location | NestJS + Redis Geo + PostGIS |
| Real-time Tracking | WebSocket + Socket.io |
| Push Notifications | Firebase Cloud Messaging |
| Message Queue | Apache Kafka |
| Search | Elasticsearch |

---

## 2. Restaurant Service

```typescript
// restaurant-service/src/restaurant.service.ts
import { Injectable, Logger, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface Restaurant {
  id: string;
  name: string;
  description: string;
  cuisine: string[];
  address: Address;
  location: GeoPoint;
  openingHours: OpeningHours[];
  rating: number;
  reviewCount: number;
  minimumOrder: number;
  deliveryFee: number;
  estimatedDeliveryTime: number; // นาที
  isOpen: boolean;
  images: string[];
  metadata: Record<string, any>;
}

export interface Address {
  street: string;
  district: string;
  city: string;
  province: string;
  postalCode: string;
  country: string;
}

export interface GeoPoint {
  type: 'Point';
  coordinates: [number, number]; // [longitude, latitude]
}

export interface OpeningHours {
  dayOfWeek: number; // 0-6 (Sunday-Saturday)
  openTime: string; // HH:MM
  closeTime: string; // HH:MM
}

@Injectable()
export class RestaurantService {
  private readonly logger = new Logger(RestaurantService.name);
  private readonly CACHE_TTL = 300; // 5 นาที

  constructor(
    @InjectModel('Restaurant') private readonly restaurantModel: Model<any>,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async findNearby(
    latitude: number,
    longitude: number,
    radiusKm: number = 5,
    cuisine?: string[],
    isOpen?: boolean,
  ): Promise<Restaurant[]> {
    const cacheKey = `nearby:${latitude}:${longitude}:${radiusKm}:${cuisine?.join(',')}:${isOpen}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const query: any = {
      location: {
        $near: {
          $geometry: { type: 'Point', coordinates: [longitude, latitude] },
          $maxDistance: radiusKm * 1000, // แปลงเป็นเมตร
        },
      },
      isActive: true,
    };

    if (cuisine && cuisine.length > 0) {
      query.cuisine = { $in: cuisine };
    }

    if (isOpen !== undefined) {
      query.isOpen = isOpen;
    }

    const restaurants = await this.restaurantModel
      .find(query)
      .limit(50)
      .lean()
      .exec();

    const result = restaurants.map(r => this.mapToRestaurant(r));
    
    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(result));
    
    return result;
  }

  async checkIsOpen(restaurantId: string): Promise<boolean> {
    const restaurant = await this.restaurantModel
      .findById(restaurantId)
      .select('openingHours timezone')
      .lean();

    if (!restaurant) {
      throw new NotFoundException(`Restaurant ${restaurantId} not found`);
    }

    const now = new Date();
    const dayOfWeek = now.getDay();
    const currentTime = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`;

    const todayHours = restaurant.openingHours.find(
      (h: OpeningHours) => h.dayOfWeek === dayOfWeek
    );

    if (!todayHours) return false;

    return currentTime >= todayHours.openTime && currentTime <= todayHours.closeTime;
  }

  async updateRating(restaurantId: string, newRating: number): Promise<void> {
    await this.restaurantModel.findByIdAndUpdate(restaurantId, {
      $inc: { reviewCount: 1 },
      $set: { rating: newRating },
    });

    // ลบ Cache
    const pattern = `nearby:*`;
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  private mapToRestaurant(doc: any): Restaurant {
    return {
      id: doc._id.toString(),
      name: doc.name,
      description: doc.description,
      cuisine: doc.cuisine,
      address: doc.address,
      location: doc.location,
      openingHours: doc.openingHours,
      rating: doc.rating,
      reviewCount: doc.reviewCount,
      minimumOrder: doc.minimumOrder,
      deliveryFee: doc.deliveryFee,
      estimatedDeliveryTime: doc.estimatedDeliveryTime,
      isOpen: doc.isOpen,
      images: doc.images,
      metadata: doc.metadata,
    };
  }
}
```

---

## 3. Menu Service

```typescript
// menu-service/src/menu.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface MenuItem {
  id: string;
  restaurantId: string;
  categoryId: string;
  name: string;
  nameEn?: string;
  description: string;
  price: number;
  discountedPrice?: number;
  images: string[];
  tags: string[];
  allergens: string[];
  nutritionInfo?: NutritionInfo;
  customizations: Customization[];
  isAvailable: boolean;
  preparationTime: number; // นาที
  calories?: number;
}

export interface NutritionInfo {
  calories: number;
  protein: number;
  carbohydrates: number;
  fat: number;
  fiber: number;
}

export interface Customization {
  id: string;
  name: string;
  type: 'radio' | 'checkbox';
  isRequired: boolean;
  options: CustomizationOption[];
}

export interface CustomizationOption {
  id: string;
  name: string;
  additionalPrice: number;
  isDefault: boolean;
}

export interface MenuCategory {
  id: string;
  restaurantId: string;
  name: string;
  nameEn?: string;
  description?: string;
  sortOrder: number;
  items: MenuItem[];
}

@Injectable()
export class MenuService {
  private readonly logger = new Logger(MenuService.name);
  private readonly MENU_CACHE_TTL = 600; // 10 นาที

  constructor(
    @InjectModel('Menu') private readonly menuModel: Model<any>,
    @InjectModel('MenuItem') private readonly menuItemModel: Model<any>,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async getRestaurantMenu(restaurantId: string): Promise<MenuCategory[]> {
    const cacheKey = `menu:${restaurantId}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const categories = await this.menuModel
      .find({ restaurantId, isActive: true })
      .sort({ sortOrder: 1 })
      .populate({
        path: 'items',
        match: { isAvailable: true },
        options: { sort: { sortOrder: 1 } },
      })
      .lean();

    const result = categories.map(cat => this.mapToMenuCategory(cat));
    
    await this.redis.setex(cacheKey, this.MENU_CACHE_TTL, JSON.stringify(result));
    
    return result;
  }

  async validateOrderItems(
    restaurantId: string,
    items: OrderItem[],
  ): Promise<{ isValid: boolean; errors: string[]; totalPrice: number }> {
    const errors: string[] = [];
    let totalPrice = 0;

    for (const item of items) {
      const menuItem = await this.menuItemModel.findOne({
        _id: item.menuItemId,
        restaurantId,
        isAvailable: true,
      }).lean();

      if (!menuItem) {
        errors.push(`Menu item ${item.menuItemId} not found or unavailable`);
        continue;
      }

      let itemPrice = menuItem.discountedPrice || menuItem.price;

      // ตรวจสอบและคำนวณราคา Customizations
      for (const customization of item.customizations || []) {
        const menuCustomization = menuItem.customizations.find(
          (c: Customization) => c.id === customization.customizationId
        );

        if (!menuCustomization) {
          errors.push(`Customization ${customization.customizationId} not found`);
          continue;
        }

        if (menuCustomization.isRequired && (!customization.selectedOptions || customization.selectedOptions.length === 0)) {
          errors.push(`Customization "${menuCustomization.name}" is required`);
          continue;
        }

        for (const optionId of customization.selectedOptions || []) {
          const option = menuCustomization.options.find(
            (o: CustomizationOption) => o.id === optionId
          );
          if (option) {
            itemPrice += option.additionalPrice;
          }
        }
      }

      totalPrice += itemPrice * item.quantity;
    }

    return {
      isValid: errors.length === 0,
      errors,
      totalPrice,
    };
  }

  async updateItemAvailability(
    menuItemId: string,
    isAvailable: boolean,
  ): Promise<void> {
    const item = await this.menuItemModel.findByIdAndUpdate(
      menuItemId,
      { isAvailable },
      { new: true },
    );

    if (item) {
      // ลบ Cache ของร้านอาหารนั้น
      await this.redis.del(`menu:${item.restaurantId}`);
    }
  }

  private mapToMenuCategory(doc: any): MenuCategory {
    return {
      id: doc._id.toString(),
      restaurantId: doc.restaurantId,
      name: doc.name,
      nameEn: doc.nameEn,
      description: doc.description,
      sortOrder: doc.sortOrder,
      items: (doc.items || []).map((item: any) => this.mapToMenuItem(item)),
    };
  }

  private mapToMenuItem(doc: any): MenuItem {
    return {
      id: doc._id.toString(),
      restaurantId: doc.restaurantId,
      categoryId: doc.categoryId,
      name: doc.name,
      nameEn: doc.nameEn,
      description: doc.description,
      price: doc.price,
      discountedPrice: doc.discountedPrice,
      images: doc.images,
      tags: doc.tags,
      allergens: doc.allergens,
      nutritionInfo: doc.nutritionInfo,
      customizations: doc.customizations,
      isAvailable: doc.isAvailable,
      preparationTime: doc.preparationTime,
      calories: doc.calories,
    };
  }
}

interface OrderItem {
  menuItemId: string;
  quantity: number;
  customizations?: {
    customizationId: string;
    selectedOptions: string[];
  }[];
  specialInstructions?: string;
}
```

---

## 4. Order Service

```typescript
// order-service/src/order.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

export enum OrderStatus {
  PENDING = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  PREPARING = 'PREPARING',
  READY_FOR_PICKUP = 'READY_FOR_PICKUP',
  PICKED_UP = 'PICKED_UP',
  DELIVERING = 'DELIVERING',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
}

export interface CreateOrderRequest {
  customerId: string;
  restaurantId: string;
  items: OrderItemRequest[];
  deliveryAddress: DeliveryAddress;
  paymentMethodId: string;
  specialInstructions?: string;
  promoCode?: string;
}

export interface OrderItemRequest {
  menuItemId: string;
  quantity: number;
  customizations?: {
    customizationId: string;
    selectedOptions: string[];
  }[];
  specialInstructions?: string;
}

export interface DeliveryAddress {
  street: string;
  district: string;
  city: string;
  province: string;
  postalCode: string;
  latitude: number;
  longitude: number;
  notes?: string;
}

@Injectable()
export class OrderService {
  private readonly logger = new Logger(OrderService.name);

  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly menuService: MenuService,
    private readonly restaurantService: RestaurantService,
    private readonly paymentService: PaymentService,
    private readonly deliveryAssignmentService: DeliveryAssignmentService,
    private readonly etaService: ETAService,
    private readonly eventEmitter: EventEmitter2,
    @InjectQueue('order-processing') private readonly orderQueue: Queue,
    private readonly dataSource: DataSource,
  ) {}

  async createOrder(request: CreateOrderRequest): Promise<Order> {
    // 1. ตรวจสอบว่าร้านอาหารเปิดอยู่
    const isOpen = await this.restaurantService.checkIsOpen(request.restaurantId);
    if (!isOpen) {
      throw new Error('Restaurant is currently closed');
    }

    // 2. ตรวจสอบและคำนวณราคาสินค้า
    const validation = await this.menuService.validateOrderItems(
      request.restaurantId,
      request.items,
    );

    if (!validation.isValid) {
      throw new Error(`Invalid order items: ${validation.errors.join(', ')}`);
    }

    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 3. คำนวณค่าจัดส่ง
      const restaurant = await this.restaurantService.findById(request.restaurantId);
      const deliveryFee = await this.calculateDeliveryFee(
        restaurant.location,
        request.deliveryAddress,
      );

      // 4. คำนวณ ETA
      const eta = await this.etaService.calculate(
        restaurant.location,
        { latitude: request.deliveryAddress.latitude, longitude: request.deliveryAddress.longitude },
        restaurant.estimatedDeliveryTime,
      );

      // 5. คำนวณส่วนลด (ถ้ามี promo code)
      let discount = 0;
      if (request.promoCode) {
        discount = await this.calculateDiscount(
          request.promoCode,
          validation.totalPrice,
          request.customerId,
        );
      }

      const totalAmount = validation.totalPrice + deliveryFee - discount;

      // 6. สร้าง Order
      const order = queryRunner.manager.create(Order, {
        customerId: request.customerId,
        restaurantId: request.restaurantId,
        items: request.items.map(item => ({
          menuItemId: item.menuItemId,
          quantity: item.quantity,
          customizations: item.customizations,
          specialInstructions: item.specialInstructions,
        })),
        deliveryAddress: request.deliveryAddress,
        subtotal: validation.totalPrice,
        deliveryFee,
        discount,
        totalAmount,
        status: OrderStatus.PENDING,
        estimatedDeliveryTime: eta.estimatedMinutes,
        estimatedDeliveryAt: new Date(Date.now() + eta.estimatedMinutes * 60000),
        specialInstructions: request.specialInstructions,
      });

      const savedOrder = await queryRunner.manager.save(order);

      // 7. ประมวลผลการชำระเงิน
      const paymentResult = await this.paymentService.processPayment({
        orderId: savedOrder.id,
        customerId: request.customerId,
        amount: totalAmount,
        currency: 'THB',
        paymentMethodId: request.paymentMethodId,
      });

      if (!paymentResult.success) {
        throw new Error(`Payment failed: ${paymentResult.errorMessage}`);
      }

      // 8. อัปเดตสถานะเป็น CONFIRMED
      savedOrder.status = OrderStatus.CONFIRMED;
      savedOrder.paymentId = paymentResult.paymentId;
      await queryRunner.manager.save(savedOrder);

      await queryRunner.commitTransaction();

      // 9. ส่ง Events
      this.eventEmitter.emit('order.created', savedOrder);
      this.eventEmitter.emit('order.confirmed', savedOrder);

      // 10. เพิ่มใน Queue สำหรับ Assign Driver
      await this.orderQueue.add('assign-driver', {
        orderId: savedOrder.id,
        restaurantId: request.restaurantId,
        deliveryAddress: request.deliveryAddress,
      }, {
        delay: 5000, // รอ 5 วินาทีก่อน assign
      });

      return savedOrder;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      this.logger.error(`Order creation failed: ${error.message}`, error.stack);
      throw error;
    } finally {
      await queryRunner.release();
    }
  }

  async updateOrderStatus(
    orderId: string,
    status: OrderStatus,
    updatedBy: string,
  ): Promise<Order> {
    const order = await this.orderRepository.findOne({ where: { id: orderId } });
    
    if (!order) {
      throw new Error(`Order ${orderId} not found`);
    }

    // ตรวจสอบ State Transition ที่ถูกต้อง
    if (!this.isValidTransition(order.status, status)) {
      throw new Error(`Invalid status transition from ${order.status} to ${status}`);
    }

    const previousStatus = order.status;
    order.status = status;
    order.statusHistory = [
      ...(order.statusHistory || []),
      { status, timestamp: new Date(), updatedBy },
    ];

    const updatedOrder = await this.orderRepository.save(order);

    // ส่ง Event ตามสถานะ
    this.eventEmitter.emit(`order.${status.toLowerCase()}`, updatedOrder);

    // ถ้า Driver รับงานแล้ว
    if (status === OrderStatus.PICKED_UP) {
      this.eventEmitter.emit('order.driver_picked_up', updatedOrder);
    }

    return updatedOrder;
  }

  async cancelOrder(orderId: string, reason: string, cancelledBy: string): Promise<Order> {
    const order = await this.orderRepository.findOne({ where: { id: orderId } });
    
    if (!order) {
      throw new Error(`Order ${orderId} not found`);
    }

    const cancellableStatuses = [
      OrderStatus.PENDING,
      OrderStatus.CONFIRMED,
    ];

    if (!cancellableStatuses.includes(order.status)) {
      throw new Error(`Cannot cancel order with status ${order.status}`);
    }

    order.status = OrderStatus.CANCELLED;
    order.cancellationReason = reason;
    order.cancelledBy = cancelledBy;
    order.cancelledAt = new Date();

    const updatedOrder = await this.orderRepository.save(order);

    // คืนเงินถ้าชำระแล้ว
    if (order.paymentId) {
      await this.paymentService.refund(order.paymentId, order.totalAmount, reason);
    }

    this.eventEmitter.emit('order.cancelled', updatedOrder);

    return updatedOrder;
  }

  private isValidTransition(from: OrderStatus, to: OrderStatus): boolean {
    const validTransitions: Record<OrderStatus, OrderStatus[]> = {
      [OrderStatus.PENDING]: [OrderStatus.CONFIRMED, OrderStatus.CANCELLED],
      [OrderStatus.CONFIRMED]: [OrderStatus.PREPARING, OrderStatus.CANCELLED],
      [OrderStatus.PREPARING]: [OrderStatus.READY_FOR_PICKUP],
      [OrderStatus.READY_FOR_PICKUP]: [OrderStatus.PICKED_UP],
      [OrderStatus.PICKED_UP]: [OrderStatus.DELIVERING],
      [OrderStatus.DELIVERING]: [OrderStatus.DELIVERED],
      [OrderStatus.DELIVERED]: [],
      [OrderStatus.CANCELLED]: [],
    };

    return validTransitions[from]?.includes(to) || false;
  }

  private async calculateDeliveryFee(
    restaurantLocation: GeoPoint,
    deliveryAddress: DeliveryAddress,
  ): Promise<number> {
    const distanceKm = this.calculateDistance(
      restaurantLocation.coordinates[1],
      restaurantLocation.coordinates[0],
      deliveryAddress.latitude,
      deliveryAddress.longitude,
    );

    // ค่าจัดส่งขั้นต้น + ตามระยะทาง
    const baseFee = 20; // 20 บาท
    const perKmFee = 5; // 5 บาท/กม.
    const maxFee = 100; // สูงสุด 100 บาท

    return Math.min(baseFee + Math.ceil(distanceKm) * perKmFee, maxFee);
  }

  private calculateDistance(
    lat1: number, lon1: number,
    lat2: number, lon2: number,
  ): number {
    const R = 6371;
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
              Math.sin(dLon/2) * Math.sin(dLon/2);
    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  }

  private async calculateDiscount(
    promoCode: string,
    subtotal: number,
    customerId: string,
  ): Promise<number> {
    // ตรวจสอบ Promo Code
    const promo = await this.promoService.validate(promoCode, customerId, subtotal);
    return promo ? promo.discountAmount : 0;
  }
}
```

---

## 5. Real-time Order Tracking (WebSocket)

```typescript
// tracking-service/src/tracking.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  ConnectedSocket,
  MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { JwtService } from '@nestjs/jwt';

interface OrderUpdate {
  orderId: string;
  status: string;
  driverLocation?: {
    latitude: number;
    longitude: number;
    heading?: number;
    speed?: number;
  };
  estimatedArrival?: Date;
  message?: string;
}

@WebSocketGateway({
  namespace: 'tracking',
  cors: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['*'],
    credentials: true,
  },
})
export class TrackingGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  private server: Server;

  private readonly logger = new Logger(TrackingGateway.name);
  
  // Map ระหว่าง Socket ID กับ User/Driver ID
  private readonly socketToUser = new Map<string, string>();
  private readonly socketToDriver = new Map<string, string>();
  
  // Map ระหว่าง Order ID กับ Socket IDs ของลูกค้า
  private readonly orderRooms = new Map<string, Set<string>>();

  constructor(
    private readonly jwtService: JwtService,
    private readonly trackingService: TrackingService,
    private readonly driverLocationService: DriverLocationService,
  ) {}

  async handleConnection(client: Socket): Promise<void> {
    try {
      const token = client.handshake.auth.token || 
                    client.handshake.headers.authorization?.replace('Bearer ', '');
      
      if (!token) {
        client.disconnect();
        return;
      }

      const payload = this.jwtService.verify(token);
      
      if (payload.role === 'driver') {
        this.socketToDriver.set(client.id, payload.sub);
        this.logger.log(`Driver ${payload.sub} connected (socket: ${client.id})`);
      } else {
        this.socketToUser.set(client.id, payload.sub);
        this.logger.log(`User ${payload.sub} connected (socket: ${client.id})`);
      }
    } catch (error) {
      this.logger.warn(`Invalid token for socket ${client.id}`);
      client.disconnect();
    }
  }

  handleDisconnect(client: Socket): void {
    const userId = this.socketToUser.get(client.id);
    const driverId = this.socketToDriver.get(client.id);
    
    if (userId) {
      this.socketToUser.delete(client.id);
      this.logger.log(`User ${userId} disconnected`);
    }
    
    if (driverId) {
      this.socketToDriver.delete(client.id);
      this.logger.log(`Driver ${driverId} disconnected`);
    }

    // ออกจาก Order Rooms ทั้งหมด
    for (const [orderId, sockets] of this.orderRooms) {
      sockets.delete(client.id);
      if (sockets.size === 0) {
        this.orderRooms.delete(orderId);
      }
    }
  }

  @SubscribeMessage('subscribe_order')
  async handleSubscribeOrder(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { orderId: string },
  ): Promise<void> {
    const userId = this.socketToUser.get(client.id);
    
    if (!userId) {
      client.emit('error', { message: 'Not authenticated' });
      return;
    }

    // ตรวจสอบว่า User มีสิทธิ์ติดตาม Order นี้
    const hasAccess = await this.trackingService.verifyOrderAccess(data.orderId, userId);
    
    if (!hasAccess) {
      client.emit('error', { message: 'Access denied' });
      return;
    }

    // เข้า Room สำหรับ Order นี้
    client.join(`order:${data.orderId}`);
    
    if (!this.orderRooms.has(data.orderId)) {
      this.orderRooms.set(data.orderId, new Set());
    }
    this.orderRooms.get(data.orderId)!.add(client.id);

    // ส่งสถานะปัจจุบันทันที
    const currentStatus = await this.trackingService.getOrderStatus(data.orderId);
    client.emit('order_update', currentStatus);

    this.logger.log(`User ${userId} subscribed to order ${data.orderId}`);
  }

  @SubscribeMessage('driver_location_update')
  async handleDriverLocationUpdate(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: {
      latitude: number;
      longitude: number;
      heading?: number;
      speed?: number;
      orderId?: string;
    },
  ): Promise<void> {
    const driverId = this.socketToDriver.get(client.id);
    
    if (!driverId) {
      client.emit('error', { message: 'Not authenticated as driver' });
      return;
    }

    // บันทึก Location ของ Driver
    await this.driverLocationService.updateLocation(driverId, {
      latitude: data.latitude,
      longitude: data.longitude,
      heading: data.heading,
      speed: data.speed,
    });

    // ถ้ากำลังส่ง Order อยู่ ส่ง Location update ไปยังลูกค้า
    if (data.orderId) {
      const update: OrderUpdate = {
        orderId: data.orderId,
        status: 'DELIVERING',
        driverLocation: {
          latitude: data.latitude,
          longitude: data.longitude,
          heading: data.heading,
          speed: data.speed,
        },
      };

      // คำนวณ ETA ใหม่
      const order = await this.trackingService.getOrderDetails(data.orderId);
      if (order) {
        const eta = await this.etaService.calculateFromDriverLocation(
          { latitude: data.latitude, longitude: data.longitude },
          order.deliveryAddress,
        );
        update.estimatedArrival = new Date(Date.now() + eta * 60000);
      }

      this.server.to(`order:${data.orderId}`).emit('order_update', update);
    }
  }

  @OnEvent('order.status_changed')
  handleOrderStatusChanged(event: { orderId: string; status: string; message?: string }): void {
    const update: OrderUpdate = {
      orderId: event.orderId,
      status: event.status,
      message: event.message,
    };

    this.server.to(`order:${event.orderId}`).emit('order_update', update);
    this.logger.log(`Order ${event.orderId} status updated to ${event.status}`);
  }
}
```

---

## 6. Driver Location Service (Geospatial)

```typescript
// driver-location/src/driver-location.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

export interface DriverLocation {
  driverId: string;
  latitude: number;
  longitude: number;
  heading?: number;
  speed?: number;
  updatedAt: Date;
  status: 'AVAILABLE' | 'BUSY' | 'OFFLINE';
}

export interface NearbyDriver {
  driverId: string;
  distance: number; // เมตร
  location: { latitude: number; longitude: number };
  rating: number;
  vehicleType: string;
  estimatedArrival: number; // นาที
}

@Injectable()
export class DriverLocationService {
  private readonly logger = new Logger(DriverLocationService.name);
  private readonly LOCATION_GEO_KEY = 'driver:locations';
  private readonly LOCATION_TTL = 300; // 5 นาที (ถ้าไม่ update ถือว่า Offline)
  private readonly LOCATION_DATA_PREFIX = 'driver:location:data:';

  constructor(
    @InjectRedis() private readonly redis: Redis,
    @InjectRepository(Driver)
    private readonly driverRepository: Repository<Driver>,
  ) {}

  async updateLocation(
    driverId: string,
    location: {
      latitude: number;
      longitude: number;
      heading?: number;
      speed?: number;
    },
  ): Promise<void> {
    // อัปเดต GEO Index ใน Redis
    await this.redis.geoadd(
      this.LOCATION_GEO_KEY,
      location.longitude,
      location.latitude,
      driverId,
    );

    // บันทึก Location Data เพิ่มเติม
    const locationData: DriverLocation = {
      driverId,
      latitude: location.latitude,
      longitude: location.longitude,
      heading: location.heading,
      speed: location.speed,
      updatedAt: new Date(),
      status: 'AVAILABLE', // จะ Update จาก Order Service
    };

    await this.redis.setex(
      `${this.LOCATION_DATA_PREFIX}${driverId}`,
      this.LOCATION_TTL,
      JSON.stringify(locationData),
    );
  }

  async findNearbyAvailableDrivers(
    latitude: number,
    longitude: number,
    radiusKm: number = 5,
    limit: number = 10,
  ): Promise<NearbyDriver[]> {
    // ค้นหา Drivers ในรัศมีที่กำหนด
    const nearbyDrivers = await this.redis.georadius(
      this.LOCATION_GEO_KEY,
      longitude,
      latitude,
      radiusKm,
      'km',
      'WITHCOORD',
      'WITHDIST',
      'ASC',
      'COUNT',
      limit * 2, // ดึงมากขึ้นเพื่อกรองที่ไม่พร้อม
    ) as any[];

    const results: NearbyDriver[] = [];

    for (const driverData of nearbyDrivers) {
      const driverId = driverData[0];
      const distance = parseFloat(driverData[1]) * 1000; // แปลงเป็นเมตร
      const coords = driverData[2];

      // ตรวจสอบว่า Driver พร้อมรับงาน
      const locationData = await this.getDriverLocation(driverId);
      
      if (!locationData || locationData.status !== 'AVAILABLE') {
        continue;
      }

      // ดึงข้อมูล Driver
      const driver = await this.driverRepository.findOne({
        where: { id: driverId, isActive: true },
      });

      if (!driver) continue;

      // คำนวณเวลาถึงโดยประมาณ (ความเร็วเฉลี่ย 30 km/h ในเมือง)
      const estimatedArrival = Math.ceil((distance / 1000) / 30 * 60);

      results.push({
        driverId,
        distance,
        location: {
          latitude: parseFloat(coords[1]),
          longitude: parseFloat(coords[0]),
        },
        rating: driver.rating,
        vehicleType: driver.vehicleType,
        estimatedArrival,
      });

      if (results.length >= limit) break;
    }

    return results;
  }

  async getDriverLocation(driverId: string): Promise<DriverLocation | null> {
    const data = await this.redis.get(`${this.LOCATION_DATA_PREFIX}${driverId}`);
    return data ? JSON.parse(data) : null;
  }

  async setDriverStatus(
    driverId: string,
    status: 'AVAILABLE' | 'BUSY' | 'OFFLINE',
  ): Promise<void> {
    const locationData = await this.getDriverLocation(driverId);
    
    if (locationData) {
      locationData.status = status;
      
      if (status === 'OFFLINE') {
        // ลบออกจาก GEO Index
        await this.redis.zrem(this.LOCATION_GEO_KEY, driverId);
        await this.redis.del(`${this.LOCATION_DATA_PREFIX}${driverId}`);
      } else {
        await this.redis.setex(
          `${this.LOCATION_DATA_PREFIX}${driverId}`,
          this.LOCATION_TTL,
          JSON.stringify(locationData),
        );
      }
    }
  }

  async getDriverRoute(
    driverId: string,
    orderId: string,
  ): Promise<{ waypoints: { lat: number; lng: number }[] }> {
    const driverLocation = await this.getDriverLocation(driverId);
    if (!driverLocation) {
      throw new Error(`Driver ${driverId} location not found`);
    }

    const order = await this.orderService.findById(orderId);
    const restaurant = await this.restaurantService.findById(order.restaurantId);

    // ใช้ Google Maps Directions API หรือ OSRM สำหรับ Route Planning
    const route = await this.mapsService.getRoute([
      { lat: driverLocation.latitude, lng: driverLocation.longitude },
      { lat: restaurant.location.coordinates[1], lng: restaurant.location.coordinates[0] },
      { lat: order.deliveryAddress.latitude, lng: order.deliveryAddress.longitude },
    ]);

    return route;
  }
}
```

---

## 7. Delivery Assignment Algorithm

```typescript
// delivery-assignment/src/assignment.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Process, Processor } from '@nestjs/bull';
import { Job } from 'bull';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

interface AssignmentJob {
  orderId: string;
  restaurantId: string;
  deliveryAddress: {
    latitude: number;
    longitude: number;
  };
}

@Processor('order-processing')
@Injectable()
export class DeliveryAssignmentService {
  private readonly logger = new Logger(DeliveryAssignmentService.name);
  private readonly MAX_RETRIES = 5;
  private readonly RETRY_INTERVAL = 30000; // 30 วินาที

  constructor(
    private readonly driverLocationService: DriverLocationService,
    private readonly driverService: DriverService,
    private readonly orderService: OrderService,
    private readonly notificationService: NotificationService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  @Process('assign-driver')
  async assignDriver(job: Job<AssignmentJob>): Promise<void> {
    const { orderId, restaurantId, deliveryAddress } = job.data;
    
    this.logger.log(`Attempting to assign driver for order ${orderId} (attempt ${job.attemptsMade + 1})`);

    const restaurant = await this.restaurantService.findById(restaurantId);
    
    if (!restaurant) {
      this.logger.error(`Restaurant ${restaurantId} not found`);
      return;
    }

    // ค้นหา Driver ที่พร้อมใกล้ร้านอาหาร
    const availableDrivers = await this.driverLocationService.findNearbyAvailableDrivers(
      restaurant.location.coordinates[1],
      restaurant.location.coordinates[0],
      10, // รัศมี 10 km
      5,  // ดึงมา 5 คน
    );

    if (availableDrivers.length === 0) {
      this.logger.warn(`No available drivers for order ${orderId}`);
      
      if (job.attemptsMade < this.MAX_RETRIES - 1) {
        throw new Error('No available drivers'); // Bull จะ Retry อัตโนมัติ
      } else {
        // แจ้งลูกค้าและร้านอาหาร
        await this.notificationService.sendNoDriverAlert(orderId);
        return;
      }
    }

    // เลือก Driver ที่ดีที่สุด
    const selectedDriver = this.selectBestDriver(availableDrivers, restaurant);
    
    // ส่งคำขอให้ Driver
    const accepted = await this.offerOrderToDriver(selectedDriver.driverId, orderId);
    
    if (!accepted) {
      // ลอง Driver คนต่อไป
      for (let i = 1; i < availableDrivers.length; i++) {
        const accepted = await this.offerOrderToDriver(availableDrivers[i].driverId, orderId);
        if (accepted) {
          await this.finalizeAssignment(orderId, availableDrivers[i].driverId);
          return;
        }
      }
      
      // ไม่มี Driver รับงาน ลองใหม่
      throw new Error('All drivers declined');
    }

    await this.finalizeAssignment(orderId, selectedDriver.driverId);
  }

  private selectBestDriver(
    drivers: NearbyDriver[],
    restaurant: Restaurant,
  ): NearbyDriver {
    // คะแนนจาก:
    // - ระยะทาง (60% weight)
    // - Rating (30% weight)
    // - จำนวนงานวันนี้ (10% weight - น้อยกว่าดีกว่า)
    
    return drivers.reduce((best, driver) => {
      const distanceScore = 1 - (driver.distance / 10000); // Normalize 0-1
      const ratingScore = driver.rating / 5;
      const score = distanceScore * 0.6 + ratingScore * 0.3;
      
      if (!best || score > best.score) {
        return { ...driver, score };
      }
      return best;
    }, null as any);
  }

  private async offerOrderToDriver(
    driverId: string,
    orderId: string,
  ): Promise<boolean> {
    // ส่ง Push Notification ให้ Driver
    await this.notificationService.sendOrderOffer(driverId, orderId);
    
    // รอการตอบรับ 30 วินาที
    const responseKey = `driver:order:response:${driverId}:${orderId}`;
    
    return new Promise((resolve) => {
      const timeout = setTimeout(() => {
        this.redis.del(responseKey);
        resolve(false);
      }, 30000);

      this.redis.subscribe(`driver:response:${driverId}`, (message) => {
        const data = JSON.parse(message);
        if (data.orderId === orderId) {
          clearTimeout(timeout);
          this.redis.unsubscribe(`driver:response:${driverId}`);
          resolve(data.accepted);
        }
      });
    });
  }

  private async finalizeAssignment(orderId: string, driverId: string): Promise<void> {
    // อัปเดต Order
    await this.orderService.assignDriver(orderId, driverId);
    
    // อัปเดต Driver Status
    await this.driverLocationService.setDriverStatus(driverId, 'BUSY');
    
    // ส่ง Event
    this.eventEmitter.emit('order.driver_assigned', { orderId, driverId });
    
    this.logger.log(`Driver ${driverId} assigned to order ${orderId}`);
  }
}
```

---

## 8. ETA Calculation Service

```typescript
// eta-service/src/eta.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';
import { firstValueFrom } from 'rxjs';

export interface ETAResult {
  estimatedMinutes: number;
  preparationMinutes: number;
  travelMinutes: number;
  confidence: 'HIGH' | 'MEDIUM' | 'LOW';
}

@Injectable()
export class ETAService {
  private readonly logger = new Logger(ETAService.name);
  private readonly AVERAGE_SPEED_KMH = 25; // ความเร็วเฉลี่ยในเมือง
  private readonly TRAFFIC_FACTOR = 1.3; // ปรับตามสภาพการจราจร

  constructor(
    private readonly httpService: HttpService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async calculate(
    restaurantLocation: { latitude: number; longitude: number },
    deliveryLocation: { latitude: number; longitude: number },
    preparationTimeMinutes: number,
  ): Promise<ETAResult> {
    // คำนวณเวลาเดินทาง
    const travelMinutes = await this.calculateTravelTime(
      restaurantLocation,
      deliveryLocation,
    );

    const totalMinutes = preparationTimeMinutes + travelMinutes;

    return {
      estimatedMinutes: totalMinutes,
      preparationMinutes,
      travelMinutes,
      confidence: 'MEDIUM',
    };
  }

  async calculateFromDriverLocation(
    driverLocation: { latitude: number; longitude: number },
    deliveryLocation: { latitude: number; longitude: number },
  ): Promise<number> {
    return this.calculateTravelTime(driverLocation, deliveryLocation);
  }

  private async calculateTravelTime(
    from: { latitude: number; longitude: number },
    to: { latitude: number; longitude: number },
  ): Promise<number> {
    const cacheKey = `eta:${from.latitude.toFixed(3)}:${from.longitude.toFixed(3)}:${to.latitude.toFixed(3)}:${to.longitude.toFixed(3)}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return parseInt(cached);
    }

    try {
      // ใช้ Google Maps Distance Matrix API
      const response = await firstValueFrom(
        this.httpService.get(
          `https://maps.googleapis.com/maps/api/distancematrix/json`,
          {
            params: {
              origins: `${from.latitude},${from.longitude}`,
              destinations: `${to.latitude},${to.longitude}`,
              mode: 'driving',
              traffic_model: 'best_guess',
              departure_time: 'now',
              key: process.env.GOOGLE_MAPS_API_KEY,
            },
          },
        ),
      );

      const element = response.data.rows[0].elements[0];
      
      if (element.status === 'OK') {
        const travelMinutes = Math.ceil(
          (element.duration_in_traffic?.value || element.duration.value) / 60
        );
        
        // Cache เป็น 5 นาที
        await this.redis.setex(cacheKey, 300, travelMinutes.toString());
        
        return travelMinutes;
      }
    } catch (error) {
      this.logger.warn(`Google Maps API failed, using fallback: ${error.message}`);
    }

    // Fallback: คำนวณจากระยะทางตรง
    const distanceKm = this.calculateDistance(from, to);
    const travelMinutes = Math.ceil((distanceKm / this.AVERAGE_SPEED_KMH) * 60 * this.TRAFFIC_FACTOR);
    
    return travelMinutes;
  }

  private calculateDistance(
    from: { latitude: number; longitude: number },
    to: { latitude: number; longitude: number },
  ): number {
    const R = 6371;
    const dLat = (to.latitude - from.latitude) * Math.PI / 180;
    const dLon = (to.longitude - from.longitude) * Math.PI / 180;
    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(from.latitude * Math.PI / 180) *
      Math.cos(to.latitude * Math.PI / 180) *
      Math.sin(dLon / 2) * Math.sin(dLon / 2);
    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  }
}
```

---

## 9. Rating & Review Service

```typescript
// rating-service/src/rating.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { OnEvent } from '@nestjs/event-emitter';

export interface ReviewRequest {
  orderId: string;
  customerId: string;
  restaurantRating: number; // 1-5
  restaurantReview?: string;
  driverRating?: number; // 1-5
  driverReview?: string;
  foodRating: number; // 1-5
  deliverySpeedRating?: number; // 1-5
  images?: string[];
}

@Injectable()
export class RatingService {
  private readonly logger = new Logger(RatingService.name);

  constructor(
    @InjectRepository(Review)
    private readonly reviewRepository: Repository<Review>,
    private readonly restaurantService: RestaurantService,
    private readonly driverService: DriverService,
    private readonly orderService: OrderService,
  ) {}

  async submitReview(request: ReviewRequest): Promise<Review> {
    // ตรวจสอบว่า Order Delivered แล้ว
    const order = await this.orderService.findById(request.orderId);
    
    if (!order || order.status !== 'DELIVERED') {
      throw new Error('Order must be delivered before review');
    }

    if (order.customerId !== request.customerId) {
      throw new Error('You can only review your own orders');
    }

    // ตรวจสอบว่ารีวิวแล้วยัง
    const existingReview = await this.reviewRepository.findOne({
      where: { orderId: request.orderId },
    });

    if (existingReview) {
      throw new Error('Order already reviewed');
    }

    // ตรวจสอบค่าคะแนน
    if (request.restaurantRating < 1 || request.restaurantRating > 5) {
      throw new Error('Rating must be between 1 and 5');
    }

    const review = await this.reviewRepository.save({
      orderId: request.orderId,
      customerId: request.customerId,
      restaurantId: order.restaurantId,
      driverId: order.driverId,
      restaurantRating: request.restaurantRating,
      restaurantReview: request.restaurantReview,
      driverRating: request.driverRating,
      driverReview: request.driverReview,
      foodRating: request.foodRating,
      deliverySpeedRating: request.deliverySpeedRating,
      images: request.images,
    });

    // อัปเดต Rating ของร้านอาหาร
    await this.updateRestaurantRating(order.restaurantId);

    // อัปเดต Rating ของ Driver (ถ้ามี)
    if (request.driverRating && order.driverId) {
      await this.updateDriverRating(order.driverId);
    }

    return review;
  }

  async getRestaurantReviews(
    restaurantId: string,
    page: number = 1,
    limit: number = 20,
  ): Promise<{ reviews: Review[]; total: number; averageRating: number }> {
    const [reviews, total] = await this.reviewRepository.findAndCount({
      where: { restaurantId },
      order: { createdAt: 'DESC' },
      skip: (page - 1) * limit,
      take: limit,
    });

    const stats = await this.reviewRepository
      .createQueryBuilder('r')
      .where('r.restaurantId = :restaurantId', { restaurantId })
      .select('AVG(r.restaurantRating)', 'avgRating')
      .getRawOne();

    return {
      reviews,
      total,
      averageRating: parseFloat(stats.avgRating) || 0,
    };
  }

  private async updateRestaurantRating(restaurantId: string): Promise<void> {
    const stats = await this.reviewRepository
      .createQueryBuilder('r')
      .where('r.restaurantId = :restaurantId', { restaurantId })
      .select([
        'AVG(r.restaurantRating) as avgRating',
        'COUNT(*) as reviewCount',
      ])
      .getRawOne();

    await this.restaurantService.updateRating(
      restaurantId,
      parseFloat(stats.avgRating),
    );
  }

  private async updateDriverRating(driverId: string): Promise<void> {
    const stats = await this.reviewRepository
      .createQueryBuilder('r')
      .where('r.driverId = :driverId', { driverId })
      .andWhere('r.driverRating IS NOT NULL')
      .select('AVG(r.driverRating)', 'avgRating')
      .getRawOne();

    await this.driverService.updateRating(
      driverId,
      parseFloat(stats.avgRating),
    );
  }
}
```

---

## 10. Push Notification Service

```typescript
// notification-service/src/push-notification.service.ts
import { Injectable, Logger } from '@nestjs/common';
import * as admin from 'firebase-admin';
import { OnEvent } from '@nestjs/event-emitter';

interface PushNotificationPayload {
  token: string;
  title: string;
  body: string;
  data?: Record<string, string>;
  imageUrl?: string;
}

@Injectable()
export class PushNotificationService {
  private readonly logger = new Logger(PushNotificationService.name);
  private readonly firebaseApp: admin.app.App;

  constructor(private readonly deviceTokenService: DeviceTokenService) {
    this.firebaseApp = admin.initializeApp({
      credential: admin.credential.cert({
        projectId: process.env.FIREBASE_PROJECT_ID,
        privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
        clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      }),
    });
  }

  @OnEvent('order.confirmed')
  async handleOrderConfirmed(order: any): Promise<void> {
    await this.sendToUser(order.customerId, {
      title: 'ยืนยันออร์เดอร์แล้ว ✅',
      body: `ออร์เดอร์ #${order.id.slice(-8)} ได้รับการยืนยันแล้ว ร้านกำลังเตรียมอาหาร`,
      data: {
        type: 'ORDER_CONFIRMED',
        orderId: order.id,
      },
    });
  }

  @OnEvent('order.driver_assigned')
  async handleDriverAssigned(event: { orderId: string; driverId: string }): Promise<void> {
    const order = await this.orderService.findById(event.orderId);
    const driver = await this.driverService.findById(event.driverId);
    
    await this.sendToUser(order.customerId, {
      title: 'ไรเดอร์รับออร์เดอร์แล้ว 🛵',
      body: `${driver.name} กำลังมารับอาหาร`,
      data: {
        type: 'DRIVER_ASSIGNED',
        orderId: event.orderId,
        driverId: event.driverId,
      },
    });
  }

  @OnEvent('order.picked_up')
  async handleOrderPickedUp(order: any): Promise<void> {
    await this.sendToUser(order.customerId, {
      title: 'ออร์เดอร์ถูกรับแล้ว 🎉',
      body: 'ไรเดอร์กำลังเดินทางมาส่งอาหารให้คุณ',
      data: {
        type: 'ORDER_PICKED_UP',
        orderId: order.id,
      },
    });
  }

  @OnEvent('order.delivered')
  async handleOrderDelivered(order: any): Promise<void> {
    await this.sendToUser(order.customerId, {
      title: 'ส่งอาหารแล้ว 🍔',
      body: 'อาหารของคุณถูกส่งถึงแล้ว! รีวิวออร์เดอร์เพื่อรับคะแนนพิเศษ',
      data: {
        type: 'ORDER_DELIVERED',
        orderId: order.id,
      },
    });
  }

  async sendToUser(userId: string, payload: Omit<PushNotificationPayload, 'token'>): Promise<void> {
    const tokens = await this.deviceTokenService.getUserTokens(userId);
    
    if (tokens.length === 0) {
      this.logger.warn(`No device tokens for user ${userId}`);
      return;
    }

    await this.sendBatch(tokens, payload);
  }

  async sendToDriver(driverId: string, payload: Omit<PushNotificationPayload, 'token'>): Promise<void> {
    const tokens = await this.deviceTokenService.getDriverTokens(driverId);
    await this.sendBatch(tokens, payload);
  }

  async sendOrderOffer(driverId: string, orderId: string): Promise<void> {
    const order = await this.orderService.findById(orderId);
    
    await this.sendToDriver(driverId, {
      title: 'มีออร์เดอร์ใหม่! 📦',
      body: `ออร์เดอร์ห่างจากคุณ ${order.estimatedDriverDistance} กม. - ฿${order.totalAmount}`,
      data: {
        type: 'ORDER_OFFER',
        orderId,
        timeout: '30', // วินาที
      },
    });
  }

  private async sendBatch(
    tokens: string[],
    payload: Omit<PushNotificationPayload, 'token'>,
  ): Promise<void> {
    const messages: admin.messaging.Message[] = tokens.map(token => ({
      token,
      notification: {
        title: payload.title,
        body: payload.body,
        imageUrl: payload.imageUrl,
      },
      data: payload.data,
      android: {
        priority: 'high',
        notification: {
          sound: 'default',
          channelId: 'food-delivery',
        },
      },
      apns: {
        payload: {
          aps: {
            sound: 'default',
            badge: 1,
          },
        },
      },
    }));

    try {
      const response = await this.firebaseApp.messaging().sendEach(messages);
      
      // ลบ Token ที่ไม่ถูกต้อง
      const invalidTokens = response.responses
        .map((r, i) => r.error?.code === 'messaging/registration-token-not-registered' ? tokens[i] : null)
        .filter(Boolean) as string[];
      
      if (invalidTokens.length > 0) {
        await this.deviceTokenService.removeTokens(invalidTokens);
      }

      this.logger.log(`Sent ${response.successCount}/${messages.length} push notifications`);
    } catch (error) {
      this.logger.error(`Failed to send push notifications: ${error.message}`);
    }
  }
}
```

---

## 11. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  api-gateway:
    image: kong:3.4
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /etc/kong/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
    volumes:
      - ./kong.yml:/etc/kong/kong.yml
    ports:
      - "8000:8000"
    networks:
      - food-delivery

  restaurant-service:
    build: ./restaurant-service
    ports:
      - "3001:3000"
    environment:
      MONGODB_URI: mongodb://mongo:27017/restaurants
      REDIS_URL: redis://redis:6379
    depends_on:
      - mongo
      - redis
    networks:
      - food-delivery

  order-service:
    build: ./order-service
    ports:
      - "3002:3000"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/orders
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092
    depends_on:
      - postgres
      - redis
      - kafka
    networks:
      - food-delivery

  tracking-service:
    build: ./tracking-service
    ports:
      - "3003:3000"
      - "3004:3001" # WebSocket port
    environment:
      REDIS_URL: redis://redis:6379
    networks:
      - food-delivery

  driver-location-service:
    build: ./driver-location
    ports:
      - "3005:3000"
    environment:
      REDIS_URL: redis://redis:6379
      DATABASE_URL: postgresql://postgres:password@postgres:5432/drivers
      GOOGLE_MAPS_API_KEY: ${GOOGLE_MAPS_API_KEY}
    networks:
      - food-delivery

  notification-service:
    build: ./notification-service
    environment:
      KAFKA_BROKERS: kafka:9092
      FIREBASE_PROJECT_ID: ${FIREBASE_PROJECT_ID}
      FIREBASE_PRIVATE_KEY: ${FIREBASE_PRIVATE_KEY}
      FIREBASE_CLIENT_EMAIL: ${FIREBASE_CLIENT_EMAIL}
    networks:
      - food-delivery

  mongo:
    image: mongo:6
    volumes:
      - mongo_data:/data/db
    networks:
      - food-delivery

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: food_delivery
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - food-delivery

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - food-delivery

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
    networks:
      - food-delivery

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - food-delivery

networks:
  food-delivery:
    driver: bridge

volumes:
  mongo_data:
  postgres_data:
  redis_data:
```

---

## 12. Kong API Gateway Configuration

```yaml
# kong.yml
_format_version: "3.0"

services:
  - name: restaurant-service
    url: http://restaurant-service:3000
    routes:
      - name: restaurant-routes
        paths:
          - /api/v1/restaurants
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local

  - name: order-service
    url: http://order-service:3000
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 60
          policy: local

  - name: tracking-ws
    url: http://tracking-service:3001
    routes:
      - name: tracking-websocket
        paths:
          - /ws/tracking
        protocols:
          - ws
          - wss

plugins:
  - name: cors
    config:
      origins:
        - "https://app.fooddelivery.com"
        - "https://admin.fooddelivery.com"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      headers:
        - Authorization
        - Content-Type
      max_age: 3600

  - name: prometheus
    config:
      per_consumer: true
```

---

## สรุปบทที่ 94

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **สถาปัตยกรรม** | Restaurant + Menu + Order + Driver + Tracking + Rating + Notification |
| **Real-time Tracking** | WebSocket + Socket.io + Redis Pub/Sub |
| **Geospatial** | Redis GEO Commands สำหรับค้นหา Driver ใกล้เคียง |
| **Driver Assignment** | Bull Queue + Scoring Algorithm (Distance + Rating) |
| **ETA Calculation** | Google Maps API + Haversine Fallback |
| **Push Notifications** | Firebase Cloud Messaging (FCM) |
| **Order Flow** | PENDING → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERING → DELIVERED |
| **Rating System** | Restaurant + Driver + Food + Speed Ratings |
| **Kong Gateway** | Rate Limiting + JWT Auth + CORS + Prometheus |
| **Database** | MongoDB (Restaurant/Menu) + PostgreSQL (Order/Driver) + Redis (Location/Cache) |
