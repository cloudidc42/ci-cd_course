# Part 40: Feature Flags ใน CI/CD

## บทนำ

Feature Flags (หรือที่เรียกว่า Feature Toggles หรือ Feature Switches) คือ technique ที่ช่วยให้เราสามารถ enable หรือ disable features ของ application โดยไม่ต้องทำการ deploy code ใหม่ Feature flags เป็นเครื่องมือสำคัญสำหรับ trunk-based development, gradual rollouts, A/B testing, และ kill switches ในบทนี้เราจะเรียนรู้การใช้งาน Feature Flags อย่างครบวงจร

## วัตถุประสงค์การเรียนรู้

- เข้าใจ Feature Flag patterns และ use cases
- ใช้ LaunchDarkly สำหรับ enterprise feature management
- Implement OpenFeature standard
- ตั้งค่า Flagsmith (open-source alternative)
- ทำ Trunk-Based Development ด้วย feature flags
- Gradual rollouts และ A/B testing
- จัดการ feature flag lifecycle

---

## 40.1 Feature Flag Patterns

### Types of Feature Flags

| ประเภท | อายุการใช้งาน | Use Case |
|--------|--------------|---------|
| **Release Toggle** | ชั่วคราว | ซ่อน incomplete feature |
| **Experiment Toggle** | ชั่วคราว | A/B testing |
| **Ops Toggle** | ถาวร | Kill switch, circuit breaker |
| **Permission Toggle** | ถาวร | Premium features, beta access |
| **Infrastructure Toggle** | ชั่วคราว | Migration between systems |

### Feature Flag Lifecycle

```
Create Flag → Development → Gradual Rollout → 100% → Remove Flag
    │              │              │               │         │
  disabled     enabled for    enabled for    enabled    Clean up
               developers     5% → 50%       all users  old code
```

---

## 40.2 LaunchDarkly

### SDK Setup

```javascript
// Node.js SDK
// npm install @launchdarkly/node-server-sdk

const LaunchDarkly = require('@launchdarkly/node-server-sdk');

class FeatureFlagService {
  constructor(sdkKey) {
    this.client = LaunchDarkly.init(sdkKey);
  }
  
  async initialize() {
    await this.client.waitForInitialization();
    console.log('LaunchDarkly client initialized');
  }
  
  async isEnabled(flagKey, user, defaultValue = false) {
    const context = this.buildContext(user);
    return await this.client.variation(flagKey, context, defaultValue);
  }
  
  async getVariation(flagKey, user, defaultValue) {
    const context = this.buildContext(user);
    return await this.client.variation(flagKey, context, defaultValue);
  }
  
  buildContext(user) {
    return {
      kind: 'user',
      key: user.id,
      email: user.email,
      name: user.name,
      custom: {
        plan: user.plan,
        country: user.country,
        beta_user: user.isBetaUser,
        created_at: user.createdAt
      }
    };
  }
  
  async close() {
    await this.client.close();
  }
}

// Usage
const flagService = new FeatureFlagService(process.env.LAUNCHDARKLY_SDK_KEY);
await flagService.initialize();

// ตรวจสอบ feature flag
app.get('/checkout', async (req, res) => {
  const user = req.user;
  
  const newCheckoutEnabled = await flagService.isEnabled(
    'new-checkout-flow',
    user,
    false
  );
  
  if (newCheckoutEnabled) {
    return res.render('checkout-v2');
  } else {
    return res.render('checkout-v1');
  }
});

// Multi-variant flag (A/B testing)
app.get('/pricing', async (req, res) => {
  const user = req.user;
  
  const pricingVariant = await flagService.getVariation(
    'pricing-page-variant',
    user,
    'control'
  );
  
  // Track experiment
  metrics.track('pricing_page_viewed', {
    user_id: user.id,
    variant: pricingVariant
  });
  
  return res.render(`pricing-${pricingVariant}`);
});
```

### LaunchDarkly Python SDK

```python
# pip install launchdarkly-server-sdk

import ldclient
from ldclient.config import Config
from ldclient import Context
import os
from functools import lru_cache
import logging

logger = logging.getLogger(__name__)

class FeatureFlagClient:
    """LaunchDarkly Feature Flag Client"""
    
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        if hasattr(self, '_initialized'):
            return
        
        sdk_key = os.environ.get('LAUNCHDARKLY_SDK_KEY')
        if not sdk_key:
            raise ValueError("LAUNCHDARKLY_SDK_KEY is not set")
        
        config = Config(
            sdk_key=sdk_key,
            events_enabled=True,
            connect_timeout=5,
            read_timeout=5,
            initial_reconnect_delay=1.0,
            max_reconnect_delay=30.0
        )
        
        ldclient.set_config(config)
        self.client = ldclient.get()
        
        if not self.client.is_initialized():
            raise RuntimeError("LaunchDarkly client failed to initialize")
        
        self._initialized = True
        logger.info("LaunchDarkly client initialized")
    
    def is_feature_enabled(
        self,
        flag_key: str,
        user_id: str,
        user_email: str = None,
        user_attributes: dict = None,
        default: bool = False
    ) -> bool:
        """ตรวจสอบว่า feature flag เปิดอยู่หรือไม่"""
        context = self._build_context(user_id, user_email, user_attributes)
        result = self.client.variation(flag_key, context, default)
        
        logger.debug(f"Flag '{flag_key}' for user '{user_id}': {result}")
        return result
    
    def get_string_variation(
        self,
        flag_key: str,
        user_id: str,
        default: str = "control",
        **user_attributes
    ) -> str:
        """ดึงค่า string variation"""
        context = self._build_context(user_id, **user_attributes)
        return self.client.variation(flag_key, context, default)
    
    def get_number_variation(
        self,
        flag_key: str,
        user_id: str,
        default: float = 0.0,
        **user_attributes
    ) -> float:
        """ดึงค่า number variation"""
        context = self._build_context(user_id, **user_attributes)
        return self.client.variation(flag_key, context, default)
    
    def _build_context(
        self,
        user_id: str,
        user_email: str = None,
        user_attributes: dict = None
    ) -> Context:
        """สร้าง LaunchDarkly Context"""
        builder = Context.builder(user_id).kind("user")
        
        if user_email:
            builder.set("email", user_email)
        
        if user_attributes:
            for key, value in user_attributes.items():
                builder.set(key, value)
        
        return builder.build()
    
    def close(self):
        """ปิด client"""
        self.client.close()

# Decorator สำหรับใช้งานง่ายขึ้น
def feature_flag(flag_key: str, default: bool = False):
    """Decorator สำหรับ feature flag"""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            client = FeatureFlagClient()
            request = kwargs.get('request') or (args[0] if args else None)
            
            user_id = getattr(getattr(request, 'user', None), 'id', 'anonymous')
            
            if client.is_feature_enabled(flag_key, str(user_id), default=default):
                return await func(*args, **kwargs)
            else:
                from fastapi import HTTPException
                raise HTTPException(status_code=404, detail="Feature not available")
        
        return wrapper
    return decorator

# การใช้งาน
from fastapi import FastAPI, Request, Depends

app = FastAPI()
flag_client = FeatureFlagClient()

@app.get("/api/v2/checkout")
async def new_checkout(request: Request):
    user = request.state.user
    
    if not flag_client.is_feature_enabled(
        flag_key="new-checkout-flow",
        user_id=str(user.id),
        user_email=user.email,
        user_attributes={
            "plan": user.plan,
            "country": user.country,
            "is_beta": user.is_beta_user
        }
    ):
        from fastapi.responses import RedirectResponse
        return RedirectResponse("/api/v1/checkout")
    
    # New checkout logic
    return {"version": "v2", "flow": "new"}
```

---

## 40.3 OpenFeature Standard

### OpenFeature SDK

```typescript
// npm install @openfeature/server-sdk @openfeature/flagd-provider

import { OpenFeature } from '@openfeature/server-sdk';
import { FlagdProvider } from '@openfeature/flagd-provider';

// Setup Provider
const provider = new FlagdProvider({
  host: process.env.FLAGD_HOST || 'localhost',
  port: parseInt(process.env.FLAGD_PORT || '8013'),
  tls: process.env.NODE_ENV === 'production',
});

await OpenFeature.setProviderAndWait(provider);

// Get a client
const featureClient = OpenFeature.getClient('my-app');

// Evaluation context (user data)
const context = {
  targetingKey: user.id,
  email: user.email,
  plan: user.plan,
  country: user.country,
};

// Boolean flag
const newDashboard = await featureClient.getBooleanValue(
  'new-dashboard',
  false,  // default value
  context
);

// String flag (variants)
const checkoutFlow = await featureClient.getStringValue(
  'checkout-flow',
  'v1',
  context
);

// Number flag
const maxRetries = await featureClient.getNumberValue(
  'max-api-retries',
  3,
  context
);

// Object flag (complex configuration)
const featureConfig = await featureClient.getObjectValue(
  'recommendation-engine-config',
  { algorithm: 'basic', maxItems: 10 },
  context
);

console.log(`Using checkout flow: ${checkoutFlow}`);
console.log(`Recommendation config:`, featureConfig);
```

### flagd Configuration

```yaml
# flagd-config.yaml
flags:
  new-dashboard:
    state: "ENABLED"
    variants:
      "on": true
      "off": false
    defaultVariant: "off"
    targeting:
      # เปิดสำหรับ beta users เท่านั้น
      if:
        - in:
            - var: email
            - ["beta@example.com", "test@example.com"]
        - "on"
        - "off"
  
  checkout-flow:
    state: "ENABLED"
    variants:
      "v1": "v1"
      "v2": "v2"
      "v3": "v3"
    defaultVariant: "v1"
    targeting:
      # Gradual rollout ด้วย percentage
      fractional:
        - var: targetingKey
        - ["v2", 20]  # 20% ได้ v2
        - ["v3", 5]   # 5% ได้ v3
        - ["v1", 75]  # 75% ได้ v1
  
  max-api-retries:
    state: "ENABLED"
    variants:
      "low": 2
      "medium": 3
      "high": 5
    defaultVariant: "medium"
    targeting:
      if:
        - "=="
          - var: plan
          - "enterprise"
        - "high"
        - "medium"
  
  recommendation-engine-config:
    state: "ENABLED"
    variants:
      "basic": 
        algorithm: "collaborative-filtering"
        maxItems: 10
        freshness: "24h"
      "advanced":
        algorithm: "deep-learning"
        maxItems: 20
        freshness: "1h"
        model_version: "v2.1"
    defaultVariant: "basic"
    targeting:
      if:
        - "=="
          - var: plan
          - "premium"
        - "advanced"
        - "basic"
```

---

## 40.4 Flagsmith (Open Source)

### Docker Compose Setup

```yaml
# docker-compose.flagsmith.yml
version: '3.8'

services:
  flagsmith:
    image: flagsmith/flagsmith:latest
    ports:
      - "8000:8000"
    environment:
      DJANGO_ALLOWED_HOSTS: "*"
      DATABASE_URL: postgresql://postgres:password@flagsmith-db/flagsmith
      ENVIRONMENT: production
      ALLOW_REGISTRATION_WITHOUT_INVITE: "True"
      FLAGSMITH_DOMAIN: "http://localhost:8000"
    depends_on:
      flagsmith-db:
        condition: service_healthy
  
  flagsmith-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: flagsmith
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - flagsmith-db-data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U postgres
      interval: 5s
      timeout: 5s
      retries: 5
  
  flagsmith-edge-proxy:
    image: flagsmith/edge-proxy:latest
    ports:
      - "8001:8000"
    environment:
      FLAGSMITH_API_URL: "http://flagsmith:8000/api/v1/"
      FLAGSMITH_SERVER_SIDE_SDK_KEY: "srv-your-server-key"
    depends_on:
      - flagsmith

volumes:
  flagsmith-db-data:
```

### Flagsmith Python SDK

```python
# pip install flagsmith

from flagsmith import Flagsmith
from flagsmith.models import Flags
import os
import logging

logger = logging.getLogger(__name__)

class FlagsmithClient:
    """Flagsmith Feature Flag Client"""
    
    def __init__(self):
        self.client = Flagsmith(
            environment_key=os.environ.get('FLAGSMITH_SERVER_SIDE_KEY'),
            api_url=os.environ.get('FLAGSMITH_API_URL', 'https://edge.api.flagsmith.com/api/v1/'),
            enable_local_evaluation=True,  # Cache flags locally
            environment_refresh_interval_seconds=60,
        )
    
    def get_flags_for_user(self, user_id: str, user_data: dict = None) -> Flags:
        """ดึง flags สำหรับ specific user"""
        traits = user_data or {}
        return self.client.get_identity_flags(
            identifier=user_id,
            traits=traits
        )
    
    def is_enabled(
        self,
        flag_name: str,
        user_id: str = None,
        user_data: dict = None,
        default: bool = False
    ) -> bool:
        """ตรวจสอบว่า feature เปิดอยู่หรือไม่"""
        try:
            if user_id:
                flags = self.get_flags_for_user(user_id, user_data)
            else:
                flags = self.client.get_environment_flags()
            
            return flags.is_feature_enabled(flag_name)
        except Exception as e:
            logger.error(f"Error getting flag '{flag_name}': {e}")
            return default
    
    def get_value(
        self,
        flag_name: str,
        user_id: str = None,
        user_data: dict = None,
        default = None
    ):
        """ดึงค่าของ feature flag"""
        try:
            if user_id:
                flags = self.get_flags_for_user(user_id, user_data)
            else:
                flags = self.client.get_environment_flags()
            
            return flags.get_feature_value(flag_name)
        except Exception as e:
            logger.error(f"Error getting flag value '{flag_name}': {e}")
            return default

# Django Middleware
class FeatureFlagMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.flag_client = FlagsmithClient()
    
    def __call__(self, request):
        if hasattr(request, 'user') and request.user.is_authenticated:
            # Preload flags สำหรับ user
            request.feature_flags = self.flag_client.get_flags_for_user(
                user_id=str(request.user.id),
                user_data={
                    'email': request.user.email,
                    'plan': getattr(request.user, 'plan', 'free'),
                    'is_beta': getattr(request.user, 'is_beta_user', False)
                }
            )
        else:
            request.feature_flags = self.flag_client.client.get_environment_flags()
        
        response = self.get_response(request)
        return response

# Usage ใน views
def checkout_view(request):
    if request.feature_flags.is_feature_enabled('new-checkout-flow'):
        return render(request, 'checkout/v2.html')
    return render(request, 'checkout/v1.html')
```

---

## 40.5 Trunk-Based Development กับ Feature Flags

### Feature Flag + Trunk-Based Development

```python
# สร้าง feature ใหม่โดยซ่อนไว้เบื้องหลัง flag
# ทุกคน commit ไปที่ main (trunk) แต่ feature ถูกซ่อน

# order_service.py
class OrderService:
    def __init__(self, flag_client: FlagsmithClient):
        self.flag_client = flag_client
    
    def process_order(self, order: Order, user_id: str) -> ProcessedOrder:
        """Process order - สามารถใช้ algorithm ใหม่หรือเก่าขึ้นอยู่กับ flag"""
        
        use_ml_pricing = self.flag_client.is_enabled(
            'ml-dynamic-pricing',
            user_id=user_id
        )
        
        if use_ml_pricing:
            # New: ML-based dynamic pricing
            final_price = self._calculate_ml_price(order)
        else:
            # Old: Static pricing
            final_price = self._calculate_static_price(order)
        
        # Feature flag สำหรับ fraud detection algorithm
        fraud_check_version = self.flag_client.get_value(
            'fraud-detection-version',
            user_id=user_id,
            default='v1'
        )
        
        if fraud_check_version == 'v2':
            is_fraudulent = self._check_fraud_v2(order, user_id)
        else:
            is_fraudulent = self._check_fraud_v1(order)
        
        if is_fraudulent:
            raise FraudDetectedException("Order flagged as potential fraud")
        
        return self._create_order(order, final_price)
    
    def _calculate_ml_price(self, order: Order) -> float:
        """ML-based pricing (Feature under development)"""
        # Implementation...
        pass
    
    def _calculate_static_price(self, order: Order) -> float:
        """Original static pricing"""
        return sum(item.price * item.quantity for item in order.items)
```

### Gradual Rollout Strategy

```python
#!/usr/bin/env python3
# rollout_manager.py

import requests
from dataclasses import dataclass
from typing import Optional
import time

@dataclass
class RolloutConfig:
    flag_key: str
    target_percentage: int
    increment: int = 10
    wait_time_minutes: int = 30
    success_rate_threshold: float = 0.99
    error_rate_threshold: float = 0.01

class RolloutManager:
    """จัดการ gradual rollout ของ feature flags"""
    
    def __init__(
        self,
        flagsmith_url: str,
        flagsmith_token: str,
        metrics_url: str,
        flag_id: str,
        environment_id: str
    ):
        self.flagsmith_url = flagsmith_url
        self.flagsmith_token = flagsmith_token
        self.metrics_url = metrics_url
        self.flag_id = flag_id
        self.environment_id = environment_id
    
    def get_current_percentage(self) -> int:
        """ดึง rollout percentage ปัจจุบัน"""
        response = requests.get(
            f"{self.flagsmith_url}/api/v1/environments/{self.environment_id}/featurestates/",
            headers={"Authorization": f"Token {self.flagsmith_token}"}
        )
        
        feature_states = response.json()['results']
        for state in feature_states:
            if state['feature'] == self.flag_id:
                return state.get('feature_segment_rollout_percentage', 0)
        
        return 0
    
    def update_rollout_percentage(self, percentage: int) -> bool:
        """อัพเดต rollout percentage"""
        response = requests.patch(
            f"{self.flagsmith_url}/api/v1/environments/{self.environment_id}/featurestates/{self.flag_id}/",
            headers={
                "Authorization": f"Token {self.flagsmith_token}",
                "Content-Type": "application/json"
            },
            json={"feature_segment_rollout_percentage": percentage}
        )
        
        return response.status_code == 200
    
    def check_metrics(self, feature_name: str) -> dict:
        """ตรวจสอบ metrics ของ feature"""
        response = requests.get(
            f"{self.metrics_url}/api/metrics",
            params={
                "feature": feature_name,
                "window": "5m"
            }
        )
        
        return response.json()
    
    def should_proceed(self, metrics: dict, config: RolloutConfig) -> bool:
        """ตัดสินใจว่าควร proceed ต่อหรือไม่"""
        error_rate = metrics.get('error_rate', 0)
        success_rate = metrics.get('success_rate', 1)
        
        if error_rate > config.error_rate_threshold:
            print(f"❌ Error rate too high: {error_rate:.1%} (threshold: {config.error_rate_threshold:.1%})")
            return False
        
        if success_rate < config.success_rate_threshold:
            print(f"❌ Success rate too low: {success_rate:.1%} (threshold: {config.success_rate_threshold:.1%})")
            return False
        
        return True
    
    def rollback(self, previous_percentage: int):
        """Rollback ไปยัง previous percentage"""
        print(f"🔄 Rolling back to {previous_percentage}%")
        self.update_rollout_percentage(previous_percentage)
    
    def execute_rollout(self, config: RolloutConfig) -> bool:
        """ดำเนิน gradual rollout"""
        current = self.get_current_percentage()
        
        print(f"Starting rollout: {current}% → {config.target_percentage}%")
        
        while current < config.target_percentage:
            next_percentage = min(
                current + config.increment,
                config.target_percentage
            )
            
            print(f"\nIncreasing rollout: {current}% → {next_percentage}%")
            self.update_rollout_percentage(next_percentage)
            
            # รอและตรวจสอบ metrics
            print(f"Waiting {config.wait_time_minutes} minutes for metrics...")
            time.sleep(config.wait_time_minutes * 60)
            
            metrics = self.check_metrics(config.flag_key)
            print(f"Metrics: {metrics}")
            
            if not self.should_proceed(metrics, config):
                self.rollback(current)
                return False
            
            current = next_percentage
            print(f"✅ Rollout at {current}% - metrics healthy")
        
        print(f"\n✅ Rollout complete: {config.flag_key} is now at {current}%")
        return True

# ตัวอย่างการใช้งาน
if __name__ == "__main__":
    manager = RolloutManager(
        flagsmith_url="https://api.flagsmith.com",
        flagsmith_token=os.environ["FLAGSMITH_TOKEN"],
        metrics_url="http://prometheus:9090",
        flag_id="new-checkout-flow",
        environment_id="production"
    )
    
    config = RolloutConfig(
        flag_key="new-checkout-flow",
        target_percentage=100,
        increment=10,
        wait_time_minutes=30,
        success_rate_threshold=0.99,
        error_rate_threshold=0.01
    )
    
    success = manager.execute_rollout(config)
    
    if not success:
        print("Rollout failed! Feature has been rolled back.")
        sys.exit(1)
```

---

## 40.6 A/B Testing

### A/B Test Framework

```typescript
// ab-test-service.ts
import { OpenFeature } from '@openfeature/server-sdk';

interface ABTestResult {
  variant: string;
  userId: string;
  flagKey: string;
  timestamp: Date;
}

interface ConversionEvent {
  userId: string;
  testKey: string;
  event: string;
  value?: number;
}

class ABTestService {
  private client = OpenFeature.getClient('ab-tests');
  private analyticsQueue: ConversionEvent[] = [];
  
  async getVariant(
    testKey: string,
    userId: string,
    userAttributes: Record<string, any> = {}
  ): Promise<string> {
    const context = {
      targetingKey: userId,
      ...userAttributes,
    };
    
    const variant = await this.client.getStringValue(
      testKey,
      'control',  // default variant
      context
    );
    
    // Track variant assignment
    this.trackVariantAssignment({
      variant,
      userId,
      flagKey: testKey,
      timestamp: new Date()
    });
    
    return variant;
  }
  
  trackConversion(
    userId: string,
    testKey: string,
    event: string,
    value?: number
  ) {
    const conversion: ConversionEvent = {
      userId,
      testKey,
      event,
      value,
    };
    
    this.analyticsQueue.push(conversion);
    
    // Flush เมื่อ queue เต็ม
    if (this.analyticsQueue.length >= 100) {
      this.flush();
    }
  }
  
  private trackVariantAssignment(result: ABTestResult) {
    // ส่งไปยัง analytics (Segment, Mixpanel, etc.)
    console.log(`AB Test: User ${result.userId} assigned to ${result.variant} for ${result.flagKey}`);
  }
  
  async flush() {
    const events = this.analyticsQueue.splice(0);
    
    if (events.length === 0) return;
    
    // Batch send to analytics
    await fetch('/api/analytics/events', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(events)
    });
  }
}

// Express middleware
const abTestService = new ABTestService();

app.use(async (req, res, next) => {
  req.abTest = abTestService;
  next();
});

// Route handler
app.get('/pricing', async (req, res) => {
  const userId = req.user?.id || req.sessionID;
  
  const pricingVariant = await req.abTest.getVariant(
    'pricing-page-test',
    userId,
    {
      plan: req.user?.plan,
      country: req.user?.country
    }
  );
  
  res.render(`pricing/${pricingVariant}`, {
    variant: pricingVariant,
    userId
  });
});

// Track conversions
app.post('/api/checkout/complete', async (req, res) => {
  const { orderId, userId, amount } = req.body;
  
  // Track conversion event
  req.abTest.trackConversion(
    userId,
    'pricing-page-test',
    'purchase_completed',
    amount
  );
  
  res.json({ success: true, orderId });
});
```

---

## 40.7 Kill Switches

### Operation Kill Switch

```python
# kill_switches.py
from enum import Enum
from dataclasses import dataclass
import redis
import json
import time
import logging

logger = logging.getLogger(__name__)

class KillSwitchAction(Enum):
    DISABLE = "disable"
    RATE_LIMIT = "rate_limit"
    MAINTENANCE_MODE = "maintenance_mode"
    FEATURE_DISABLE = "feature_disable"

@dataclass
class KillSwitchConfig:
    name: str
    action: KillSwitchAction
    reason: str
    activated_by: str
    activated_at: float
    expires_at: float = None

class KillSwitchManager:
    """จัดการ operational kill switches"""
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.prefix = "killswitch:"
    
    def activate(
        self,
        switch_name: str,
        action: KillSwitchAction,
        reason: str,
        activated_by: str,
        duration_seconds: int = None
    ) -> bool:
        """เปิด kill switch"""
        config = KillSwitchConfig(
            name=switch_name,
            action=action,
            reason=reason,
            activated_by=activated_by,
            activated_at=time.time(),
            expires_at=time.time() + duration_seconds if duration_seconds else None
        )
        
        key = f"{self.prefix}{switch_name}"
        value = json.dumps({
            'action': action.value,
            'reason': reason,
            'activated_by': activated_by,
            'activated_at': config.activated_at,
            'expires_at': config.expires_at
        })
        
        if duration_seconds:
            self.redis.setex(key, duration_seconds, value)
        else:
            self.redis.set(key, value)
        
        logger.warning(
            f"KILL SWITCH ACTIVATED: {switch_name} | "
            f"Action: {action.value} | Reason: {reason} | By: {activated_by}"
        )
        
        return True
    
    def deactivate(self, switch_name: str, deactivated_by: str) -> bool:
        """ปิด kill switch"""
        key = f"{self.prefix}{switch_name}"
        result = self.redis.delete(key)
        
        if result:
            logger.info(f"Kill switch deactivated: {switch_name} by {deactivated_by}")
        
        return bool(result)
    
    def is_active(self, switch_name: str) -> bool:
        """ตรวจสอบว่า kill switch เปิดอยู่หรือไม่"""
        key = f"{self.prefix}{switch_name}"
        return bool(self.redis.exists(key))
    
    def get_config(self, switch_name: str) -> KillSwitchConfig:
        """ดึง configuration ของ kill switch"""
        key = f"{self.prefix}{switch_name}"
        data = self.redis.get(key)
        
        if not data:
            return None
        
        config_data = json.loads(data)
        return KillSwitchConfig(
            name=switch_name,
            action=KillSwitchAction(config_data['action']),
            reason=config_data['reason'],
            activated_by=config_data['activated_by'],
            activated_at=config_data['activated_at'],
            expires_at=config_data.get('expires_at')
        )
    
    def list_active(self) -> list:
        """แสดง kill switches ทั้งหมดที่ active"""
        keys = self.redis.keys(f"{self.prefix}*")
        active = []
        
        for key in keys:
            name = key.decode().replace(self.prefix, '')
            config = self.get_config(name)
            if config:
                active.append(config)
        
        return active

# FastAPI Kill Switch Middleware
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse

class KillSwitchMiddleware:
    def __init__(self, app, kill_switch_manager: KillSwitchManager):
        self.app = app
        self.ks = kill_switch_manager
    
    async def __call__(self, scope, receive, send):
        if scope['type'] == 'http':
            request = Request(scope, receive)
            
            # ตรวจสอบ maintenance mode
            if self.ks.is_active('maintenance_mode'):
                config = self.ks.get_config('maintenance_mode')
                response = JSONResponse(
                    status_code=503,
                    content={
                        'error': 'Service Unavailable',
                        'message': config.reason,
                        'retry_after': 300
                    },
                    headers={
                        'Retry-After': '300',
                        'X-Maintenance-Mode': 'true'
                    }
                )
                await response(scope, receive, send)
                return
            
            # ตรวจสอบ API kill switch
            if self.ks.is_active('api_disabled'):
                response = JSONResponse(
                    status_code=503,
                    content={'error': 'API is temporarily disabled'}
                )
                await response(scope, receive, send)
                return
        
        await self.app(scope, receive, send)
```

---

## 40.8 Feature Flag Testing

```python
# tests/test_feature_flags.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from app.services import OrderService
from app.feature_flags import FlagsmithClient

class TestFeatureFlags:
    """ทดสอบ feature flag integration"""
    
    def test_new_checkout_enabled_for_beta_users(self):
        """Beta users ควรเห็น new checkout"""
        mock_client = Mock(spec=FlagsmithClient)
        mock_client.is_enabled.return_value = True
        
        service = OrderService(flag_client=mock_client)
        result = service.get_checkout_version(user_id="beta-user-1")
        
        mock_client.is_enabled.assert_called_once_with(
            'new-checkout-flow',
            user_id='beta-user-1'
        )
        assert result == 'v2'
    
    def test_new_checkout_disabled_for_regular_users(self):
        """Regular users ควรเห็น old checkout"""
        mock_client = Mock(spec=FlagsmithClient)
        mock_client.is_enabled.return_value = False
        
        service = OrderService(flag_client=mock_client)
        result = service.get_checkout_version(user_id="regular-user-1")
        
        assert result == 'v1'
    
    def test_graceful_degradation_on_flag_error(self):
        """เมื่อ flag service error ควรใช้ default value"""
        mock_client = Mock(spec=FlagsmithClient)
        mock_client.is_enabled.side_effect = ConnectionError("Flag service unavailable")
        
        service = OrderService(flag_client=mock_client)
        
        # ควรไม่ throw exception แต่ใช้ default
        result = service.get_checkout_version(user_id="user-1")
        assert result == 'v1'  # Default

class TestKillSwitch:
    """ทดสอบ kill switches"""
    
    @pytest.fixture
    def redis_mock(self):
        mock = MagicMock()
        mock.exists.return_value = False
        mock.get.return_value = None
        return mock
    
    def test_maintenance_mode_returns_503(self, client, redis_mock):
        """Maintenance mode ควร return 503"""
        # เปิด maintenance mode
        redis_mock.exists.return_value = True
        redis_mock.get.return_value = json.dumps({
            'action': 'maintenance_mode',
            'reason': 'Scheduled maintenance',
            'activated_by': 'admin',
            'activated_at': time.time()
        }).encode()
        
        with patch('app.kill_switch.redis_client', redis_mock):
            response = client.get('/api/orders')
            
            assert response.status_code == 503
            assert 'Scheduled maintenance' in response.json()['message']
```

---

## 40.9 GitHub Actions Integration

```yaml
# .github/workflows/feature-flag-deploy.yml
name: Deploy with Feature Flags

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy application
        run: |
          # Deploy code
          kubectl set image deployment/order-service \
            order-service=registry.example.com/order-service:${{ github.sha }}
          
          kubectl rollout status deployment/order-service --timeout=5m
      
      - name: Enable feature flag for canary
        if: success()
        run: |
          # เปิด feature flag สำหรับ 10% ก่อน
          curl -X PATCH \
            "https://api.flagsmith.com/api/v1/environments/${{ secrets.FLAGSMITH_ENV_ID }}/featurestates/${{ secrets.FEATURE_STATE_ID }}/" \
            -H "Authorization: Token ${{ secrets.FLAGSMITH_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"feature_segment_rollout_percentage": 10}'
          
          echo "Feature flag enabled for 10% of users"
      
      - name: Monitor and auto-rollout
        run: |
          # รอและตรวจสอบ metrics
          for i in 1 2 3; do
            sleep 300  # รอ 5 นาที
            
            # ตรวจสอบ error rate
            ERROR_RATE=$(curl -s "${{ secrets.PROMETHEUS_URL }}/api/v1/query?query=sum(rate(http_requests_total{status_code%3D~%225..%22}[5m]))/sum(rate(http_requests_total[5m]))" | jq '.data.result[0].value[1]' | tr -d '"')
            
            echo "Current error rate: $ERROR_RATE"
            
            # ถ้า error rate > 1% ให้ stop
            if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
              echo "::error::Error rate too high! Rolling back feature flag"
              
              # ปิด feature flag
              curl -X PATCH \
                "https://api.flagsmith.com/api/v1/environments/${{ secrets.FLAGSMITH_ENV_ID }}/featurestates/${{ secrets.FEATURE_STATE_ID }}/" \
                -H "Authorization: Token ${{ secrets.FLAGSMITH_TOKEN }}" \
                -H "Content-Type: application/json" \
                -d '{"feature_segment_rollout_percentage": 0}'
              
              exit 1
            fi
            
            # เพิ่ม percentage
            NEW_PCT=$((($i * 30)))
            curl -X PATCH \
              "https://api.flagsmith.com/api/v1/environments/${{ secrets.FLAGSMITH_ENV_ID }}/featurestates/${{ secrets.FEATURE_STATE_ID }}/" \
              -H "Authorization: Token ${{ secrets.FLAGSMITH_TOKEN }}" \
              -H "Content-Type: application/json" \
              -d "{\"feature_segment_rollout_percentage\": $NEW_PCT}"
            
            echo "Feature flag increased to $NEW_PCT%"
          done
          
          # เปิดให้ทุกคน
          curl -X PATCH \
            "https://api.flagsmith.com/api/v1/environments/${{ secrets.FLAGSMITH_ENV_ID }}/featurestates/${{ secrets.FEATURE_STATE_ID }}/" \
            -H "Authorization: Token ${{ secrets.FLAGSMITH_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"feature_segment_rollout_percentage": 100}'
          
          echo "✅ Feature flag rolled out to 100%"
```

---

## 40.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Feature Flag System

สร้าง simple feature flag system โดยใช้ Redis:

```python
# แบบฝึกหัด: implement FeatureFlagService
class FeatureFlagService:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def create_flag(self, flag_key: str, default_enabled: bool) -> bool:
        """สร้าง feature flag ใหม่"""
        # TODO: implement
        pass
    
    def is_enabled(self, flag_key: str, user_id: str = None) -> bool:
        """ตรวจสอบว่า flag เปิดอยู่ไหม"""
        # TODO: implement with user targeting
        # - ถ้า user_id อยู่ใน allowed_users list → enabled
        # - ถ้า rollout_percentage > random → enabled
        # - default: disabled
        pass
    
    def set_rollout_percentage(self, flag_key: str, percentage: int) -> bool:
        """ตั้ง rollout percentage"""
        # TODO: implement
        pass
    
    def add_allowed_user(self, flag_key: str, user_id: str) -> bool:
        """เพิ่ม user ที่จะเห็น feature"""
        # TODO: implement
        pass
```

### แบบฝึกหัดที่ 2: A/B Testing Pipeline

สร้าง A/B test pipeline ที่:
1. Assign users ไปยัง control/treatment group
2. Track conversions ด้วย event tracking
3. คำนวณ statistical significance
4. แสดงผลลัพธ์ใน dashboard

### แบบฝึกหัดที่ 3: Kill Switch Implementation

สร้าง kill switch system ที่:
1. มี maintenance mode toggle
2. มี per-feature kill switches
3. มี rate limiting kill switch
4. ส่ง Slack notifications เมื่อ activate/deactivate

---

## สรุปบทที่ 31-40

ในส่วนนี้ (บทที่ 31-40) เราได้เรียนรู้เนื้อหาขั้นสูงของ CI/CD:

| บท | หัวข้อ | เครื่องมือหลัก |
|----|--------|--------------|
| 31 | Logging & Log Aggregation | ELK Stack, Fluentd, Loki |
| 32 | Metrics & Alerting | Prometheus, Grafana, Alertmanager |
| 33 | Integration Testing | Testcontainers, WireMock, Pact |
| 34 | E2E Testing | Playwright, Cypress |
| 35 | Performance Testing | k6, Locust |
| 36 | Security Testing | Semgrep, OWASP ZAP, Gitleaks |
| 37 | Container Security | Trivy, Falco, Pod Security |
| 38 | Dependency Scanning | Snyk, Dependabot, Syft |
| 39 | Database Migrations | Flyway, Liquibase, Alembic |
| 40 | Feature Flags | LaunchDarkly, OpenFeature, Flagsmith |

### สิ่งที่ได้เรียนรู้

1. **Observability**: Logs + Metrics + Traces คือ pillars ของ production monitoring
2. **Testing**: ทำ testing pyramid ให้ครบ: Unit → Integration → E2E
3. **Security**: "Shift Left" security เข้าไปใน pipeline ตั้งแต่ต้น
4. **Reliability**: Feature flags ช่วยให้ deploy ได้บ่อยขึ้นโดยไม่เสี่ยง

บทต่อไป (Part 41+) จะครอบคลุม Infrastructure as Code, Kubernetes Advanced, GitOps, และ Platform Engineering

---

*ขอแสดงความยินดี! คุณได้เรียนจบ Part 31-40 แล้ว ซึ่งครอบคลุม Observability, Testing, Security, และ Feature Management*
