# Part 98: Microservices Cost Calculator

## บทนำ

หนึ่งในคำถามที่พบบ่อยที่สุดคือ "ระบบ Microservices จะเสียค่าใช้จ่ายเท่าไหร่?" บทนี้จะช่วยให้คุณประมาณต้นทุน Infrastructure ได้อย่างแม่นยำ เปรียบเทียบ AWS/GCP/Azure, คำนวณ TCO และใช้กลยุทธ์ FinOps

---

## 1. Cost Estimation Framework

### 1.1 ปัจจัยที่กำหนดต้นทุน

```
Cost Drivers ใน Microservices:

Compute (30-50% of total):
- จำนวน Services × ขนาด Instance × จำนวน Replicas
- CPU + Memory requirements
- Scale patterns (24/7 vs schedule-based)

Storage (15-25%):
- Database (Read/Write patterns)
- Object Storage (S3/GCS)
- Cache (Redis)
- Message Queue (Kafka)

Network (10-20%):
- Data Transfer (Egress แพงที่สุด!)
- Cross-AZ traffic
- CDN Usage

Observability (5-15%):
- Metrics (Prometheus/Datadog)
- Logs (Elasticsearch/Splunk)
- Traces (Jaeger/Datadog)

Tools & Licenses (5-10%):
- CI/CD
- Container Registry
- Security Scanning

People (ไม่นับใน infra แต่สำคัญมาก):
- DevOps/Platform Engineer salary
- On-call overhead
- Training
```

### 1.2 Cost Estimation Process

```
Step 1: Define Architecture
Step 2: Estimate resource requirements per service
Step 3: Apply cloud pricing
Step 4: Add overhead (monitoring, networking, etc.)
Step 5: Apply discount options (Reserved, Spot)
Step 6: Compare providers
Step 7: Calculate TCO (Total Cost of Ownership)
```

---

## 2. AWS Pricing Deep Dive

### 2.1 EC2 / EKS Compute

```
EKS Pricing Components:

1. EKS Control Plane: $0.10/hour per cluster = ~$73/month
   (ไม่ว่าจะมีกี่ Node)

2. Worker Nodes (EC2):
   ┌─────────────────────────────────────────────────────────┐
   │ Instance Type  │ vCPU │ RAM  │ On-Demand  │ 1yr Reserved│
   ├────────────────┼──────┼──────┼────────────┼─────────────┤
   │ t3.small       │   2  │  2GB │ $0.023/hr  │ $0.014/hr   │
   │ t3.medium      │   2  │  4GB │ $0.046/hr  │ $0.028/hr   │
   │ t3.large       │   2  │  8GB │ $0.093/hr  │ $0.056/hr   │
   │ m5.large       │   2  │  8GB │ $0.096/hr  │ $0.058/hr   │
   │ m5.xlarge      │   4  │ 16GB │ $0.192/hr  │ $0.116/hr   │
   │ m5.2xlarge     │   8  │ 32GB │ $0.384/hr  │ $0.232/hr   │
   │ m5.4xlarge     │  16  │ 64GB │ $0.768/hr  │ $0.464/hr   │
   │ c5.xlarge      │   4  │  8GB │ $0.170/hr  │ $0.103/hr   │
   │ c5.2xlarge     │   8  │ 16GB │ $0.340/hr  │ $0.205/hr   │
   │ r5.large       │   2  │ 16GB │ $0.126/hr  │ $0.076/hr   │
   │ r5.xlarge      │   4  │ 32GB │ $0.252/hr  │ $0.152/hr   │
   └────────────────┴──────┴──────┴────────────┴─────────────┘
   
   Note: ราคา On-Demand ใน ap-southeast-1 (Singapore)
         ราคาไทยใช้ Singapore region เป็น proxy

3. Spot Instances:
   ลด 70-90% จาก On-Demand แต่อาจถูก interrupt
   เหมาะกับ: Batch Jobs, Stateless workers, CI/CD runners
```

### 2.2 Database Costs

```
RDS (Relational Database Service):
┌─────────────────────────────────────────────────────────────────┐
│ Instance       │ vCPU│ RAM  │ Single-AZ  │ Multi-AZ   │ Reserved│
├────────────────┼─────┼──────┼────────────┼────────────┼─────────┤
│ db.t3.medium   │  2  │  4GB │ $0.068/hr  │ $0.136/hr  │ -40%    │
│ db.t3.large    │  2  │  8GB │ $0.136/hr  │ $0.272/hr  │ -40%    │
│ db.m5.large    │  2  │  8GB │ $0.175/hr  │ $0.350/hr  │ -40%    │
│ db.m5.xlarge   │  4  │ 16GB │ $0.350/hr  │ $0.700/hr  │ -40%    │
│ db.m5.2xlarge  │  8  │ 32GB │ $0.700/hr  │ $1.400/hr  │ -40%    │
│ db.r5.large    │  2  │ 16GB │ $0.240/hr  │ $0.480/hr  │ -40%    │
│ db.r5.xlarge   │  4  │ 32GB │ $0.480/hr  │ $0.960/hr  │ -40%    │
└────────────────┴─────┴──────┴────────────┴────────────┴─────────┘

Storage: $0.115/GB/month (SSD) + $0.020/million I/Os
Backup: $0.095/GB/month (ฟรี 1x DB size)

ElastiCache (Redis):
┌──────────────────────────────────────────────────────┐
│ Node Type      │ vCPU│ RAM   │ Price/hr │ Reserved   │
├────────────────┼─────┼───────┼──────────┼────────────┤
│ cache.t3.small │  2  │  1.4GB│ $0.034/hr│ -40%       │
│ cache.t3.medium│  2  │  3.2GB│ $0.068/hr│ -40%       │
│ cache.m6g.large│  2  │  6.4GB│ $0.154/hr│ -40%       │
│ cache.r6g.large│  2  │ 13.1GB│ $0.212/hr│ -40%       │
└────────────────┴─────┴───────┴──────────┴────────────┘
```

### 2.3 Real Cost Calculator

```python
#!/usr/bin/env python3
# cost_calculator.py

from dataclasses import dataclass, field
from typing import List, Dict
from enum import Enum

class CloudProvider(Enum):
    AWS = "AWS"
    GCP = "GCP"
    AZURE = "Azure"

@dataclass
class ServiceSpec:
    name: str
    cpu_millicores: int
    memory_gb: float
    replicas_min: int
    replicas_max: int
    hours_per_day: float = 24.0  # How many hours it runs
    is_spot: bool = False

@dataclass
class DatabaseSpec:
    name: str
    engine: str  # postgresql, mysql, redis, elasticsearch
    instance_class: str
    storage_gb: int
    multi_az: bool = True
    reserved: bool = True  # 1-year reserved

@dataclass
class NetworkSpec:
    egress_gb_per_month: float
    cross_az_gb_per_month: float
    cdn_gb_per_month: float = 0

class AWSCostCalculator:
    # EC2 pricing (ap-southeast-1, per hour)
    EC2_PRICES = {
        't3.small':   {'vcpu': 2,  'ram': 2,   'price': 0.023},
        't3.medium':  {'vcpu': 2,  'ram': 4,   'price': 0.046},
        't3.large':   {'vcpu': 2,  'ram': 8,   'price': 0.093},
        'm5.large':   {'vcpu': 2,  'ram': 8,   'price': 0.096},
        'm5.xlarge':  {'vcpu': 4,  'ram': 16,  'price': 0.192},
        'm5.2xlarge': {'vcpu': 8,  'ram': 32,  'price': 0.384},
        'm5.4xlarge': {'vcpu': 16, 'ram': 64,  'price': 0.768},
        'c5.xlarge':  {'vcpu': 4,  'ram': 8,   'price': 0.170},
        'c5.2xlarge': {'vcpu': 8,  'ram': 16,  'price': 0.340},
        'r5.large':   {'vcpu': 2,  'ram': 16,  'price': 0.126},
        'r5.xlarge':  {'vcpu': 4,  'ram': 32,  'price': 0.252},
    }
    
    # RDS pricing Multi-AZ (per hour)
    RDS_PRICES = {
        'db.t3.medium':  0.136,
        'db.t3.large':   0.272,
        'db.m5.large':   0.350,
        'db.m5.xlarge':  0.700,
        'db.m5.2xlarge': 1.400,
        'db.r5.large':   0.480,
        'db.r5.xlarge':  0.960,
    }
    
    RESERVED_DISCOUNT = 0.40  # 40% off for 1-year reserved
    SPOT_DISCOUNT = 0.75      # ~75% off on-demand
    
    def calculate_eks_cost(self, services: List[ServiceSpec]) -> Dict:
        costs = {}
        total_monthly = 0
        
        # EKS Control Plane
        control_plane = 73.0  # $73/month per cluster
        
        for svc in services:
            # Select appropriate instance type
            instance = self._select_instance(svc.cpu_millicores / 1000, svc.memory_gb)
            instance_price = self.EC2_PRICES[instance]['price']
            
            # Apply discounts
            if svc.is_spot:
                instance_price *= (1 - self.SPOT_DISCOUNT)
            
            # Calculate hours per month
            hours_per_month = svc.hours_per_day * 30 * svc.replicas_min
            
            # Scale variation (use avg of min and max)
            avg_replicas = (svc.replicas_min + svc.replicas_max) / 2
            actual_hours = svc.hours_per_day * 30 * avg_replicas
            
            monthly = instance_price * actual_hours
            
            costs[svc.name] = {
                'instance_type': instance,
                'avg_replicas': avg_replicas,
                'monthly_usd': round(monthly, 2),
                'monthly_thb': round(monthly * 36.5, 2),  # ~36.5 THB/USD
            }
            total_monthly += monthly
        
        return {
            'services': costs,
            'control_plane_usd': control_plane,
            'total_compute_usd': round(total_monthly + control_plane, 2),
            'total_compute_thb': round((total_monthly + control_plane) * 36.5, 2),
        }
    
    def calculate_database_cost(self, databases: List[DatabaseSpec]) -> Dict:
        costs = {}
        total_monthly = 0
        
        for db in databases:
            if db.engine == 'redis':
                base_price = self._get_elasticache_price(db.instance_class)
            else:
                base_price = self.RDS_PRICES.get(db.instance_class, 0.350)
            
            if db.reserved:
                base_price *= (1 - self.RESERVED_DISCOUNT)
            
            monthly_compute = base_price * 24 * 30
            monthly_storage = db.storage_gb * 0.115
            
            if db.engine == 'redis':
                monthly_storage = 0  # Redis doesn't have separate storage cost
            
            monthly = monthly_compute + monthly_storage
            
            costs[db.name] = {
                'instance_class': db.instance_class,
                'monthly_compute_usd': round(monthly_compute, 2),
                'monthly_storage_usd': round(monthly_storage, 2),
                'total_monthly_usd': round(monthly, 2),
                'total_monthly_thb': round(monthly * 36.5, 2),
            }
            total_monthly += monthly
        
        return {
            'databases': costs,
            'total_database_usd': round(total_monthly, 2),
            'total_database_thb': round(total_monthly * 36.5, 2),
        }
    
    def calculate_network_cost(self, network: NetworkSpec) -> Dict:
        # AWS egress pricing (ap-southeast-1)
        # First 1GB free, then $0.09/GB
        egress_cost = max(0, (network.egress_gb_per_month - 1)) * 0.09
        
        # Cross-AZ: $0.01/GB
        cross_az_cost = network.cross_az_gb_per_month * 0.01
        
        # CloudFront CDN: $0.085/GB (first 10TB)
        cdn_cost = min(network.cdn_gb_per_month, 10240) * 0.085
        if network.cdn_gb_per_month > 10240:
            cdn_cost += (network.cdn_gb_per_month - 10240) * 0.080
        
        total = egress_cost + cross_az_cost + cdn_cost
        
        return {
            'egress_usd': round(egress_cost, 2),
            'cross_az_usd': round(cross_az_cost, 2),
            'cdn_usd': round(cdn_cost, 2),
            'total_network_usd': round(total, 2),
            'total_network_thb': round(total * 36.5, 2),
        }
    
    def _select_instance(self, cpu: float, memory_gb: float) -> str:
        """Select most cost-effective instance for given requirements"""
        # Add 20% overhead for OS, monitoring, etc.
        required_cpu = cpu * 1.2
        required_memory = memory_gb * 1.2
        
        # Find cheapest instance that fits
        best = None
        best_price = float('inf')
        
        for instance, spec in self.EC2_PRICES.items():
            if spec['vcpu'] >= required_cpu and spec['ram'] >= required_memory:
                if spec['price'] < best_price:
                    best = instance
                    best_price = spec['price']
        
        return best or 'm5.2xlarge'
    
    def _get_elasticache_price(self, instance_class: str) -> float:
        prices = {
            'cache.t3.small':  0.034,
            'cache.t3.medium': 0.068,
            'cache.m6g.large': 0.154,
            'cache.r6g.large': 0.212,
        }
        return prices.get(instance_class, 0.154)


# ตัวอย่างการใช้งาน: E-Commerce Platform
def calculate_ecommerce_costs():
    calc = AWSCostCalculator()
    
    # Define Services
    services = [
        ServiceSpec("api-gateway",         cpu_millicores=500,  memory_gb=1,   replicas_min=3,  replicas_max=10),
        ServiceSpec("user-service",        cpu_millicores=500,  memory_gb=1,   replicas_min=3,  replicas_max=20),
        ServiceSpec("product-service",     cpu_millicores=1000, memory_gb=2,   replicas_min=5,  replicas_max=30),
        ServiceSpec("order-service",       cpu_millicores=1000, memory_gb=2,   replicas_min=5,  replicas_max=50),
        ServiceSpec("payment-service",     cpu_millicores=500,  memory_gb=1,   replicas_min=3,  replicas_max=20),
        ServiceSpec("inventory-service",   cpu_millicores=500,  memory_gb=1,   replicas_min=3,  replicas_max=30),
        ServiceSpec("search-service",      cpu_millicores=2000, memory_gb=4,   replicas_min=3,  replicas_max=15),
        ServiceSpec("notification-service",cpu_millicores=250,  memory_gb=0.5, replicas_min=3,  replicas_max=20),
        ServiceSpec("recommendation-svc",  cpu_millicores=2000, memory_gb=8,   replicas_min=2,  replicas_max=10, is_spot=True),
        ServiceSpec("analytics-worker",    cpu_millicores=4000, memory_gb=8,   replicas_min=2,  replicas_max=5,  is_spot=True, hours_per_day=4),
    ]
    
    # Define Databases
    databases = [
        DatabaseSpec("user-postgres",    "postgresql", "db.m5.large",   50,   multi_az=True,  reserved=True),
        DatabaseSpec("product-postgres", "postgresql", "db.m5.xlarge",  200,  multi_az=True,  reserved=True),
        DatabaseSpec("order-postgres",   "postgresql", "db.m5.xlarge",  500,  multi_az=True,  reserved=True),
        DatabaseSpec("payment-postgres", "postgresql", "db.m5.large",   100,  multi_az=True,  reserved=True),
        DatabaseSpec("redis-cluster",    "redis",      "cache.r6g.large", 0,  multi_az=False, reserved=True),
        DatabaseSpec("redis-cache",      "redis",      "cache.m6g.large", 0,  multi_az=False, reserved=False),
    ]
    
    # Network Usage
    network = NetworkSpec(
        egress_gb_per_month=50000,    # 50 TB egress
        cross_az_gb_per_month=10000,  # 10 TB cross-AZ
        cdn_gb_per_month=100000,      # 100 TB via CDN
    )
    
    compute = calc.calculate_eks_cost(services)
    database = calc.calculate_database_cost(databases)
    net = calc.calculate_network_cost(network)
    
    # Additional services
    extras = {
        "MSK (Kafka 3 brokers)":   800,
        "OpenSearch (6 nodes)":     2400,
        "ELK Stack":                1500,
        "ECR (Container Registry)": 100,
        "Route53":                  50,
        "ACM Certificates":         0,
        "CloudWatch":               500,
        "WAF":                      300,
    }
    
    total_extras = sum(extras.values())
    
    grand_total = compute['total_compute_usd'] + database['total_database_usd'] + \
                  net['total_network_usd'] + total_extras
    
    print("\n" + "="*60)
    print("E-COMMERCE PLATFORM MONTHLY COST ESTIMATE (AWS)")
    print("="*60)
    print(f"\nCOMPUTE (EKS):")
    for name, cost in compute['services'].items():
        print(f"  {name:<25} ${cost['monthly_usd']:>8.2f}/month ({cost['avg_replicas']:.1f} avg pods)")
    print(f"\n  EKS Control Plane:          ${compute['control_plane_usd']:>8.2f}/month")
    print(f"  SUBTOTAL COMPUTE:           ${compute['total_compute_usd']:>8.2f}/month")
    
    print(f"\nDATABASES:")
    for name, cost in database['databases'].items():
        print(f"  {name:<25} ${cost['total_monthly_usd']:>8.2f}/month")
    print(f"  SUBTOTAL DATABASES:         ${database['total_database_usd']:>8.2f}/month")
    
    print(f"\nNETWORK:")
    print(f"  Egress (50TB):              ${net['egress_usd']:>8.2f}/month")
    print(f"  Cross-AZ:                   ${net['cross_az_usd']:>8.2f}/month")
    print(f"  CDN (100TB):                ${net['cdn_usd']:>8.2f}/month")
    print(f"  SUBTOTAL NETWORK:           ${net['total_network_usd']:>8.2f}/month")
    
    print(f"\nADDITIONAL SERVICES:")
    for name, cost in extras.items():
        print(f"  {name:<25} ${cost:>8.2f}/month")
    print(f"  SUBTOTAL EXTRAS:            ${total_extras:>8.2f}/month")
    
    print(f"\n{'='*50}")
    print(f"TOTAL MONTHLY:              ${grand_total:>8.2f}/month")
    print(f"TOTAL MONTHLY (THB):        ฿{grand_total * 36.5:>8,.0f}/month")
    print(f"ANNUAL:                     ${grand_total * 12:>8,.2f}/year")
    print(f"{'='*50}")
    
    return grand_total

if __name__ == "__main__":
    calculate_ecommerce_costs()
```

---

## 3. Cloud Provider Comparison

### 3.1 AWS vs GCP vs Azure

```
Cloud Provider Comparison (Asia Pacific):

┌─────────────────────────────────────────────────────────────────────┐
│ Category              │ AWS                │ GCP         │ Azure     │
├───────────────────────┼────────────────────┼─────────────┼───────────┤
│ Thailand Region       │ Singapore (SEA-1)  │ Singapore   │ Singapore │
│                       │                    │             │           │
│ Kubernetes            │ EKS ($73/cluster)  │ GKE (free!) │ AKS(free!)│
│                       │                    │             │           │
│ Compute (m5.xlarge    │ $0.192/hr          │$0.190/hr    │$0.194/hr  │
│  equiv.)              │                    │             │           │
│                       │                    │             │           │
│ Managed Postgres      │ RDS ($0.350/hr     │Cloud SQL    │Azure DB   │
│ (4vCPU,16GB Multi-AZ) │  Multi-AZ)         │($0.315/hr)  │($0.330/hr)│
│                       │                    │             │           │
│ Redis                 │ ElastiCache        │ Memorystore │ Azure Cache│
│                       │ $0.154/hr          │$0.175/hr    │$0.162/hr  │
│                       │                    │             │           │
│ Object Storage        │ S3 $0.023/GB       │GCS $0.020/GB│Blob $0.018│
│                       │                    │             │           │
│ CDN                   │ CloudFront         │ Cloud CDN   │ Azure CDN │
│                       │ $0.085/GB          │$0.080/GB    │$0.087/GB  │
│                       │                    │             │           │
│ Egress                │ $0.09/GB           │$0.08/GB     │$0.087/GB  │
│                       │                    │             │           │
│ Support               │ Best (most mature) │Good         │Good       │
│                       │                    │             │           │
│ Thai Market Presence  │ Strongest          │Growing      │Growing    │
│                       │                    │             │           │
│ Free Tier             │ 12 months          │ Always free │12 months  │
│                       │                    │ tier available│         │
└───────────────────────┴────────────────────┴─────────────┴───────────┘

Overall Cost Comparison (same workload):
AWS:   $38,000/month (baseline)
GCP:   $35,000/month (~8% cheaper)
Azure: $37,000/month (~3% cheaper)

Note: ราคาต่างกัน ~5-15% แต่ Ecosystem ความสมบูรณ์ AWS ยังนำอยู่
```

### 3.2 Cost Optimization Strategies

```python
# FinOps Best Practices

class CostOptimizationStrategies:
    """
    กลยุทธ์ลด Cloud Cost ใน Microservices
    """
    
    STRATEGIES = {
        "Right-sizing": {
            "description": "ปรับขนาด Instance ให้เหมาะสม",
            "savings": "10-30%",
            "steps": [
                "ดู CPU/Memory utilization จาก Prometheus",
                "ถ้า avg CPU < 30%: ลด instance size",
                "ถ้า avg Memory < 50%: ลด memory limit",
                "ทำทุก 2-4 สัปดาห์",
            ],
            "tools": ["AWS Compute Optimizer", "GCP Rightsizing Recommendations"],
        },
        
        "Spot/Preemptible": {
            "description": "ใช้ Spot Instances สำหรับ Fault-tolerant workloads",
            "savings": "60-90%",
            "eligible_workloads": [
                "Batch data processing",
                "ML model training",
                "CI/CD build agents",
                "Non-critical microservices",
                "Analytics workers",
            ],
            "not_eligible": [
                "Stateful services (databases)",
                "Payment processing",
                "Real-time critical paths",
            ],
        },
        
        "Reserved Instances": {
            "description": "Pre-commit สำหรับ Stable workloads",
            "savings": "40-60%",
            "tiers": {
                "1-year no upfront": "30% savings",
                "1-year all upfront": "40% savings",
                "3-year no upfront": "45% savings",
                "3-year all upfront": "60% savings",
            },
            "when_to_buy": "เมื่อ workload stable > 3 เดือน",
        },
        
        "Autoscaling Optimization": {
            "description": "Scale down aggressively during low traffic",
            "savings": "15-40%",
            "tactics": [
                "Schedule-based scaling: ลด replicas ช่วงกลางคืน",
                "Scale-to-zero: Knative สำหรับ non-critical",
                "Aggressive scale-down policy",
                "Peak/off-peak HPA configurations",
            ],
        },
        
        "Data Transfer Optimization": {
            "description": "ลด Network Egress (แพงมาก!)",
            "savings": "30-50% on networking",
            "tactics": [
                "ใช้ CDN สำหรับ Static assets (ลด Origin cost)",
                "Enable compression (gzip/brotli)",
                "Use Regional caches (ลด cross-AZ)",
                "Batch API calls (ลดจำนวน requests)",
                "Use S3 Transfer Acceleration แทน download direct",
            ],
        },
        
        "Storage Tiering": {
            "description": "ย้าย cold data ไป Cheaper storage",
            "savings": "40-80% on storage",
            "implementation": {
                "S3 Lifecycle Rules": {
                    "Hot (0-30 days)": "Standard ($0.023/GB)",
                    "Warm (30-90 days)": "Standard-IA ($0.0125/GB)",
                    "Cold (90+ days)": "Glacier ($0.004/GB)",
                    "Archive (1yr+)": "Glacier Deep Archive ($0.00099/GB)",
                }
            },
        },
        
        "Database Optimization": {
            "description": "ปรับ Database ให้ Cost-effective",
            "tactics": [
                "Aurora Serverless v2: Pay per ACU (ดีสำหรับ Variable load)",
                "Read Replicas ในแทน Vertical scaling",
                "RDS Proxy: ลด connection overhead",
                "Multi-region replication เฉพาะที่จำเป็น",
                "Audit Query performance: ลด RDS CPU = ลด cost",
            ],
        },
    }


# Schedule-based autoscaling
def generate_scheduled_scaling():
    """
    Thai Market Traffic Pattern
    Business hours: 9AM-10PM peak
    Night: 12AM-7AM low
    """
    
    schedules = [
        {
            "name": "business-hours-scale-up",
            "cron": "0 9 * * 1-5",  # Weekday 9AM Bangkok
            "min_replicas": 10,
            "max_replicas": 100,
        },
        {
            "name": "evening-peak-scale-up", 
            "cron": "0 17 * * *",  # 5PM everyday
            "min_replicas": 15,
            "max_replicas": 150,
        },
        {
            "name": "night-scale-down",
            "cron": "0 23 * * *",  # 11PM everyday
            "min_replicas": 2,
            "max_replicas": 20,
        },
        {
            "name": "weekend-base",
            "cron": "0 0 * * 6,0",  # Sat/Sun midnight
            "min_replicas": 5,
            "max_replicas": 50,
        },
    ]
    
    return schedules
```

---

## 4. TCO (Total Cost of Ownership)

### 4.1 คำนวณ TCO แบบสมบูรณ์

```
TCO = Infrastructure + People + Tooling + Opportunity Cost

Infrastructure (ต่อปี):
- AWS/Cloud costs:         $456,000  (38K/month × 12)
- Software Licenses:       $24,000   (Datadog, PagerDuty, etc.)
- Training:                $15,000
- TOTAL INFRASTRUCTURE:    $495,000

People (ต่อปี) - สำหรับ Microservices ที่จริงจัง:
- 3x Backend Engineers:    ฿5.4M   (~฿1.8M/คน)
- 1x DevOps/Platform:      ฿1.8M
- 0.5x DBA:                ฿0.9M
- 0.5x Security Engineer:  ฿0.9M
- On-call overhead (20%):  ฿1.8M
- TOTAL PEOPLE:            ฿10.8M/year

Tooling:
- GitHub Enterprise:       ฿180,000/year
- Jira/Confluence:         ฿120,000/year
- Monitoring stack:        ฿240,000/year
- TOTAL TOOLING:           ฿540,000/year

GRAND TCO:
- Infrastructure:          ฿18,072,000/year (at 36.5 THB/USD)
- People:                  ฿10,800,000/year
- Tooling:                 ฿540,000/year
- TOTAL:                   ฿29,412,000/year (≈ ฿2.45M/month)

ROI Consideration:
ถ้าระบบ generate revenue ฿50M/year
TCO = 59% of revenue → Not great

เป้าหมาย: Infrastructure ไม่เกิน 10% of revenue
```

### 4.2 FinOps Dashboard

```yaml
# grafana/finops-dashboard.json
# Real-time cost visibility

Dashboard Panels:
1. Daily Cloud Spend (bar chart, per service)
2. Cost Anomalies (alert when +20% vs rolling 7-day avg)
3. Reserved vs On-demand ratio
4. Spot instance savings
5. Top 10 expensive resources
6. Cost per transaction / per API call
7. Budget alerts (80%, 90%, 100% of monthly budget)
8. Savings recommendations

Key Metrics:
- Unit Economics: Cost per Order, Cost per User, Cost per API Call
- Efficiency: CPU utilization, Memory utilization
- Waste: Unattached EBS volumes, Old snapshots, Unused Reserved instances
```

---

## 5. Cost Alert System

```go
// finops-service/internal/alerter/cost_alerter.go
package alerter

import (
    "context"
    "time"
)

type CostAlerter struct {
    awsCostExplorer CostExplorerClient
    slack           SlackClient
    pagerduty       PagerDutyClient
    db              CostRepository
}

type CostAlert struct {
    ServiceName     string
    CurrentCost     float64
    ExpectedCost    float64
    ChangePercent   float64
    Severity        string // WARNING, CRITICAL
    Recommendation  string
}

func (a *CostAlerter) CheckDailySpend(ctx context.Context) error {
    today := time.Now().Format("2006-01-02")
    yesterday := time.Now().AddDate(0, 0, -1).Format("2006-01-02")
    
    // Get today's spend by service tag
    todaySpend, err := a.awsCostExplorer.GetCostAndUsage(ctx, CostRequest{
        Start:       today,
        End:         time.Now().Format("2006-01-02T15:04:05Z"),
        Granularity: "HOURLY",
        GroupBy:     "aws:cloudformation:stack-name",
    })
    
    // Get 7-day rolling average
    rollingAvg, _ := a.db.GetRollingAverage(ctx, 7)
    
    for service, cost := range todaySpend {
        avg := rollingAvg[service]
        if avg == 0 {
            continue
        }
        
        changePercent := ((cost - avg) / avg) * 100
        
        if changePercent > 50 {
            // CRITICAL: spend is 50%+ above average
            a.slack.PostMessage(ctx, "#cost-alerts", formatCriticalAlert(service, cost, avg, changePercent))
            a.pagerduty.CreateIncident(ctx, fmt.Sprintf("Cost anomaly: %s +%.0f%%", service, changePercent))
        } else if changePercent > 20 {
            // WARNING
            a.slack.PostMessage(ctx, "#cost-alerts", formatWarningAlert(service, cost, avg, changePercent))
        }
        
        // Store daily cost for trending
        a.db.StoreDailyCost(ctx, DailyCost{
            Service: service,
            Cost:    cost,
            Date:    today,
        })
    }
    
    return nil
}

func formatCriticalAlert(service string, cost, avg, change float64) string {
    return fmt.Sprintf(
        "🚨 *COST CRITICAL* - %s\n"+
        "Today's spend: $%.2f\n"+
        "7-day average: $%.2f\n"+
        "Change: +%.0f%%\n"+
        "Investigate: https://console.aws.amazon.com/cost-management/",
        service, cost, avg, change,
    )
}
```

---

## สรุป

การจัดการค่าใช้จ่ายใน Microservices ต้องทำอย่างเป็นระบบ:

1. **ประมาณการก่อน Build** - ใช้ Cost Calculator เพื่อ Validate Business Case
2. **Right-sizing** - ปรับขนาด Instance ตาม Actual usage ไม่ใช่คาด
3. **Spot + Reserved** - ผสมกลยุทธ์ลด Compute cost 40-60%
4. **Network Optimization** - CDN + Compression ช่วยลด Egress cost มาก
5. **Storage Tiering** - ย้าย Cold data → Glacier ประหยัด 80%
6. **Cost Visibility** - FinOps Dashboard ให้ทุกทีมเห็นค่าใช้จ่ายของตัวเอง
7. **Unit Economics** - วัด Cost per Business Transaction ไม่ใช่แค่ Monthly bill

> "You can't optimize what you can't measure — make cost a first-class metric alongside latency and error rate"

---

*ถัดไป: Part 99 - Final Project: World-Class Microservices System*
