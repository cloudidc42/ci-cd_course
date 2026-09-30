# Part 35: Performance Testing ใน CI/CD

## บทนำ

Performance Testing เป็นการตรวจสอบว่าระบบสามารถรองรับ load ที่คาดไว้ได้หรือไม่ และยังคงทำงานได้อย่างมีประสิทธิภาพ การทำ performance testing ใน CI/CD pipeline ช่วยให้เราตรวจจับ performance regressions ก่อนที่จะ deploy ไปยัง production ในบทนี้เราจะเรียนรู้การใช้ k6, Apache JMeter, และ Locust

## วัตถุประสงค์การเรียนรู้

- เข้าใจประเภทของ Performance Testing
- เขียน load test scenarios ด้วย k6
- ใช้ Locust สำหรับ Python-based load testing
- กำหนด performance thresholds
- ตรวจจับ performance regressions
- Integrate performance tests เข้ากับ CI pipeline

---

## 35.1 ประเภทของ Performance Testing

| ประเภท | จุดประสงค์ | ตัวอย่าง |
|--------|----------|---------|
| **Load Testing** | ทดสอบที่ expected load | 100 users พร้อมกัน |
| **Stress Testing** | ทดสอบขีดจำกัดของระบบ | เพิ่ม users จนระบบพัง |
| **Spike Testing** | ทดสอบ sudden traffic spike | เพิ่มจาก 10 เป็น 1000 users ทันที |
| **Soak Testing** | ทดสอบ sustained load ระยะยาว | 100 users เป็นเวลา 24 ชั่วโมง |
| **Volume Testing** | ทดสอบกับข้อมูลจำนวนมาก | 1 million records ในฐานข้อมูล |
| **Scalability Testing** | ทดสอบการ scale up/down | เพิ่ม servers แล้ว throughput เพิ่มไหม |

---

## 35.2 k6 - Load Testing

### การติดตั้ง k6

```bash
# macOS
brew install k6

# Ubuntu/Debian
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# Docker
docker pull grafana/k6
```

### k6 Test Basics

```javascript
// tests/performance/basic-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Counter, Trend, Gauge } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const successfulLogins = new Counter('successful_logins');
const transactionDuration = new Trend('transaction_duration');
const activeUsers = new Gauge('active_users');

// Test configuration
export const options = {
  // Scenarios
  scenarios: {
    // Gradual ramp up
    gradual_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },   // ramp up ไป 10 users ใน 2 นาที
        { duration: '5m', target: 10 },   // คงที่ 10 users เป็นเวลา 5 นาที
        { duration: '2m', target: 50 },   // ramp up ไป 50 users
        { duration: '5m', target: 50 },   // คงที่
        { duration: '2m', target: 0 },    // ramp down
      ],
    },
    
    // Constant rate
    constant_rate: {
      executor: 'constant-arrival-rate',
      rate: 100,           // 100 requests per second
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 50,
      maxVUs: 200,
    },
  },
  
  // Performance thresholds (SLOs)
  thresholds: {
    // HTTP errors ต้องน้อยกว่า 1%
    http_req_failed: ['rate<0.01'],
    
    // 95% ของ requests ต้องใช้เวลาน้อยกว่า 500ms
    http_req_duration: [
      'p(95)<500',
      'p(99)<2000',
      'avg<200',
    ],
    
    // Custom metric thresholds
    errors: ['rate<0.05'],
    
    // Transaction duration
    transaction_duration: ['p(95)<1000'],
  },
};

// Setup (รันครั้งเดียวก่อน test)
export function setup() {
  // สร้าง test data
  const response = http.post(
    `${__ENV.BASE_URL}/api/test/setup`,
    JSON.stringify({ scenario: 'load-test' }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  
  return {
    authToken: response.json('token'),
    testUserId: response.json('userId')
  };
}

// Main test function
export default function(data) {
  const { authToken } = data;
  
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${authToken}`,
  };
  
  // === Scenario: Browse Products ===
  const productsResponse = http.get(
    `${__ENV.BASE_URL}/api/products?page=1&limit=20`,
    { headers, tags: { name: 'browse_products' } }
  );
  
  check(productsResponse, {
    'products status is 200': (r) => r.status === 200,
    'products has items': (r) => r.json('items.length') > 0,
    'products response time OK': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);
  
  sleep(1);
  
  // === Scenario: View Product Detail ===
  const productId = productsResponse.json('items.0.id');
  const productResponse = http.get(
    `${__ENV.BASE_URL}/api/products/${productId}`,
    { headers, tags: { name: 'view_product' } }
  );
  
  check(productResponse, {
    'product detail status is 200': (r) => r.status === 200,
    'product has required fields': (r) => {
      const body = r.json();
      return body.id && body.name && body.price;
    },
  });
  
  sleep(0.5);
  
  // === Scenario: Add to Cart ===
  const startTime = new Date();
  
  const cartResponse = http.post(
    `${__ENV.BASE_URL}/api/cart/items`,
    JSON.stringify({
      productId: productId,
      quantity: 1
    }),
    { headers, tags: { name: 'add_to_cart' } }
  );
  
  transactionDuration.add(new Date() - startTime);
  
  check(cartResponse, {
    'add to cart status is 201': (r) => r.status === 201,
    'cart item added': (r) => r.json('items.length') >= 1,
  }) || errorRate.add(1);
  
  sleep(1);
}

// Teardown
export function teardown(data) {
  // ลบ test data
  http.delete(
    `${__ENV.BASE_URL}/api/test/cleanup`,
    null,
    { headers: { 'Authorization': `Bearer ${data.authToken}` } }
  );
}
```

### Advanced k6 Scenarios

```javascript
// tests/performance/scenarios.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// โหลด test data จาก file
const users = new SharedArray('users', function() {
  return JSON.parse(open('./data/test-users.json'));
});

const products = new SharedArray('products', function() {
  return JSON.parse(open('./data/test-products.json'));
});

export const options = {
  scenarios: {
    // สถานการณ์ที่ 1: Regular users
    regular_users: {
      executor: 'constant-vus',
      vus: 50,
      duration: '10m',
      exec: 'browseAndBuy',
      tags: { user_type: 'regular' }
    },
    
    // สถานการณ์ที่ 2: Search-heavy users
    searchers: {
      executor: 'constant-vus',
      vus: 30,
      duration: '10m',
      exec: 'searchProducts',
      tags: { user_type: 'searcher' }
    },
    
    // สถานการณ์ที่ 3: Checkout flow
    checkout_users: {
      executor: 'ramping-arrival-rate',
      startRate: 5,
      timeUnit: '1m',
      preAllocatedVUs: 20,
      maxVUs: 100,
      stages: [
        { target: 10, duration: '5m' },
        { target: 20, duration: '5m' },
        { target: 5, duration: '5m' },
      ],
      exec: 'checkoutFlow',
      tags: { user_type: 'checkout' }
    },
    
    // สถานการณ์ที่ 4: Admin operations
    admin_operations: {
      executor: 'constant-vus',
      vus: 5,
      duration: '10m',
      exec: 'adminOperations',
      tags: { user_type: 'admin' }
    },
  },
  
  thresholds: {
    'http_req_duration{user_type:regular}': ['p(95)<500'],
    'http_req_duration{user_type:checkout}': ['p(95)<1000'],
    'http_req_duration{user_type:searcher}': ['p(95)<300'],
    'http_req_failed': ['rate<0.02'],
  }
};

// Browse and Buy scenario
export function browseAndBuy() {
  const user = users[__VU % users.length];
  const headers = getHeaders(user.token);
  
  group('Browse Products', () => {
    // หน้า listing
    http.get(`${__ENV.BASE_URL}/api/products`, { headers });
    sleep(randomBetween(1, 3));
    
    // ดู category
    const category = ['electronics', 'clothing', 'food'][Math.floor(Math.random() * 3)];
    http.get(`${__ENV.BASE_URL}/api/products?category=${category}`, { headers });
    sleep(randomBetween(1, 2));
    
    // ดู product detail
    const product = products[Math.floor(Math.random() * products.length)];
    http.get(`${__ENV.BASE_URL}/api/products/${product.id}`, { headers });
    sleep(randomBetween(2, 5));
  });
  
  // 30% โอกาสจะ add to cart
  if (Math.random() < 0.3) {
    group('Add to Cart', () => {
      const product = products[Math.floor(Math.random() * products.length)];
      
      const response = http.post(
        `${__ENV.BASE_URL}/api/cart/items`,
        JSON.stringify({ productId: product.id, quantity: 1 }),
        { headers }
      );
      
      check(response, {
        'Added to cart': (r) => r.status === 201
      });
      
      sleep(randomBetween(1, 3));
    });
  }
}

// Checkout flow scenario  
export function checkoutFlow() {
  const user = users[__VU % users.length];
  const headers = getHeaders(user.token);
  
  // Clear cart
  http.delete(`${__ENV.BASE_URL}/api/cart`, { headers });
  
  // Add items
  const itemCount = Math.floor(Math.random() * 3) + 1;
  for (let i = 0; i < itemCount; i++) {
    const product = products[Math.floor(Math.random() * products.length)];
    http.post(
      `${__ENV.BASE_URL}/api/cart/items`,
      JSON.stringify({ productId: product.id, quantity: 1 }),
      { headers }
    );
  }
  
  sleep(1);
  
  // View cart
  http.get(`${__ENV.BASE_URL}/api/cart`, { headers });
  sleep(2);
  
  // Place order
  const orderResponse = http.post(
    `${__ENV.BASE_URL}/api/orders`,
    JSON.stringify({
      shippingAddress: {
        firstName: 'Test',
        lastName: 'User',
        address: '123 Test St',
        city: 'Bangkok',
        postalCode: '10110'
      },
      paymentMethod: 'test_card'
    }),
    { headers }
  );
  
  check(orderResponse, {
    'Order placed': (r) => r.status === 201,
    'Order has ID': (r) => r.json('id') !== null
  });
}

function getHeaders(token) {
  return {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  };
}

function randomBetween(min, max) {
  return Math.random() * (max - min) + min;
}
```

### k6 Extensions และ Output

```javascript
// k6-with-influxdb-output.js
// รัน: k6 run --out influxdb=http://localhost:8086/k6 script.js

export const options = {
  // ส่ง metrics ไปยัง InfluxDB สำหรับ real-time Grafana dashboard
};

// หรือใช้ xk6-dashboard
// k6 run --out web-dashboard script.js

// หรือ output เป็น JSON
// k6 run --out json=results.json script.js
```

---

## 35.3 Locust สำหรับ Python-based Load Testing

### การติดตั้ง Locust

```bash
pip install locust
```

### Locust Test

```python
# locustfile.py
from locust import HttpUser, task, between, events, constant_throughput
from locust.contrib.fasthttp import FastHttpUser
import json
import random
import logging

logger = logging.getLogger(__name__)

class APIUser(HttpUser):
    """จำลอง user ที่ใช้ API"""
    
    # เวลารอระหว่าง requests (1-3 วินาที)
    wait_time = between(1, 3)
    
    # Host URL
    host = "http://localhost:8080"
    
    def on_start(self):
        """รันเมื่อ user เริ่มต้น"""
        # Login
        response = self.client.post("/api/auth/login", json={
            "email": f"testuser{self.user_id}@example.com",
            "password": "TestPassword123!"
        })
        
        if response.status_code == 200:
            data = response.json()
            self.token = data['token']
            self.headers = {"Authorization": f"Bearer {self.token}"}
            logger.info(f"User {self.user_id} logged in successfully")
        else:
            logger.error(f"Login failed for user {self.user_id}")
            self.token = None
            self.headers = {}
    
    def on_stop(self):
        """รันเมื่อ user หยุด"""
        if self.token:
            self.client.post("/api/auth/logout", headers=self.headers)
    
    @task(5)
    def browse_products(self):
        """ดู product list - ทำบ่อยที่สุด (weight 5)"""
        with self.client.get(
            "/api/products",
            params={"page": random.randint(1, 10), "limit": 20},
            headers=self.headers,
            catch_response=True
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if not data.get('items'):
                    response.failure("No products returned")
            else:
                response.failure(f"Unexpected status code: {response.status_code}")
    
    @task(3)
    def view_product_detail(self):
        """ดู product detail"""
        product_id = random.randint(1, 100)
        
        with self.client.get(
            f"/api/products/{product_id}",
            headers=self.headers,
            name="/api/products/[id]",  # Group similar requests
            catch_response=True
        ) as response:
            if response.status_code == 200:
                response.success()
            elif response.status_code == 404:
                response.success()  # 404 เป็น expected behavior
            else:
                response.failure(f"Unexpected: {response.status_code}")
    
    @task(2)
    def search_products(self):
        """ค้นหา products"""
        queries = ["เสื้อ", "กางเกง", "รองเท้า", "กระเป๋า", "นาฬิกา"]
        
        with self.client.get(
            "/api/products/search",
            params={"q": random.choice(queries)},
            headers=self.headers,
            catch_response=True
        ) as response:
            if response.status_code != 200:
                response.failure(f"Search failed: {response.status_code}")
    
    @task(1)
    def add_to_cart(self):
        """เพิ่มสินค้าใน cart"""
        product_id = random.randint(1, 50)
        
        with self.client.post(
            "/api/cart/items",
            json={"productId": product_id, "quantity": 1},
            headers=self.headers,
            name="/api/cart/items",
            catch_response=True
        ) as response:
            if response.status_code not in [200, 201]:
                response.failure(f"Add to cart failed: {response.status_code}")
    
    @task(1)
    def checkout_flow(self):
        """ทำ checkout"""
        # View cart
        self.client.get("/api/cart", headers=self.headers)
        
        # Place order
        with self.client.post(
            "/api/orders",
            json={
                "shippingAddress": {
                    "firstName": "Test",
                    "lastName": "User",
                    "address": "123 Test Road",
                    "city": "Bangkok",
                    "postalCode": "10110"
                },
                "paymentMethod": "test_card"
            },
            headers=self.headers,
            name="/api/orders",
            catch_response=True
        ) as response:
            if response.status_code not in [200, 201]:
                response.failure(f"Order failed: {response.status_code}")
            else:
                order_id = response.json().get('id')
                logger.debug(f"Order created: {order_id}")

class AdminUser(HttpUser):
    """จำลอง admin ที่ทำ heavy operations"""
    
    wait_time = between(5, 10)
    weight = 1  # น้อยกว่า regular users
    
    def on_start(self):
        response = self.client.post("/api/auth/login", json={
            "email": "admin@example.com",
            "password": "AdminPassword123!"
        })
        self.headers = {"Authorization": f"Bearer {response.json().get('token', '')}"}
    
    @task
    def generate_report(self):
        """สร้าง report"""
        with self.client.get(
            "/api/admin/reports/sales",
            params={"period": "monthly"},
            headers=self.headers,
            catch_response=True,
            timeout=30
        ) as response:
            if response.status_code != 200:
                response.failure(f"Report failed: {response.status_code}")

# Custom Event Listeners
@events.request.add_listener
def on_request(request_type, name, response_time, response_length, response, 
               context, exception, start_time, url, **kwargs):
    """Log slow requests"""
    if response_time > 2000:  # > 2 วินาที
        logger.warning(f"Slow request: {name} took {response_time:.0f}ms")

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    logger.info("Load test starting")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    # สรุปผล
    stats = environment.stats
    total = stats.total
    logger.info(f"Test completed: {total.num_requests} requests, "
                f"{total.fail_ratio:.1%} failure rate, "
                f"median: {total.median_response_time:.0f}ms")
```

### Locust Configuration File

```ini
# locust.conf
headless = true
host = http://localhost:8080
users = 100
spawn-rate = 10
run-time = 10m
html = locust-report.html
csv = locust-results

# Log level
loglevel = INFO

# Stop after X failures
stop-timeout = 30
exit-code-on-error = 1
```

### รัน Locust ใน CI

```bash
# Run headless
locust -f locustfile.py \
  --headless \
  --host=http://localhost:8080 \
  --users=100 \
  --spawn-rate=10 \
  --run-time=5m \
  --html=locust-report.html \
  --csv=locust-results \
  --exit-code-on-error=1

# Run distributed (master + workers)
# Master
locust -f locustfile.py --master --expect-workers=4

# Workers (รัน 4 processes)
locust -f locustfile.py --worker --master-host=localhost &
locust -f locustfile.py --worker --master-host=localhost &
locust -f locustfile.py --worker --master-host=localhost &
locust -f locustfile.py --worker --master-host=localhost &
```

---

## 35.4 Performance Thresholds และ Gates

### Threshold Configuration

```javascript
// k6/thresholds.js - กำหนด SLO thresholds

export const THRESHOLDS = {
  // API endpoints
  api: {
    'http_req_duration{endpoint:products}': [
      { threshold: 'p(95)<200', abortOnFail: false },
      { threshold: 'p(99)<500', abortOnFail: false },
    ],
    'http_req_duration{endpoint:checkout}': [
      { threshold: 'p(95)<1000', abortOnFail: true },  // Abort test if fails
      { threshold: 'p(99)<3000', abortOnFail: false },
    ],
    'http_req_failed': [
      { threshold: 'rate<0.01', abortOnFail: true },
    ],
  },
  
  // Business metrics
  business: {
    'orders_per_second': [
      { threshold: 'rate>5', abortOnFail: false },  // Min 5 orders/sec
    ],
    'payment_success_rate': [
      { threshold: 'rate>0.95', abortOnFail: true },  // Min 95% success
    ],
  }
};

// ตรวจสอบ threshold results
export function checkThresholds(results) {
  const failures = [];
  
  for (const [metric, thresholdList] of Object.entries(results.metrics)) {
    for (const threshold of thresholdList) {
      if (!threshold.ok) {
        failures.push({
          metric,
          threshold: threshold.threshold,
          value: threshold.calculated,
          expected: threshold.expected
        });
      }
    }
  }
  
  return failures;
}
```

### Performance Regression Detection

```python
#!/usr/bin/env python3
# performance_regression.py - ตรวจสอบ performance regression

import json
import sys
from pathlib import Path
from typing import Dict, List, Optional

THRESHOLDS = {
    "p50": 1.10,   # ยอมให้เพิ่มขึ้นได้ 10%
    "p95": 1.15,   # ยอมให้เพิ่มขึ้นได้ 15%
    "p99": 1.20,   # ยอมให้เพิ่มขึ้นได้ 20%
    "error_rate": 1.05,  # ยอมให้เพิ่มขึ้นได้ 5%
}

def load_results(filepath: str) -> dict:
    """โหลด k6 results จาก JSON file"""
    with open(filepath) as f:
        return json.load(f)

def extract_metrics(results: dict) -> dict:
    """ดึง metrics ที่ต้องการ"""
    metrics = {}
    
    for metric_name, metric_data in results.get("metrics", {}).items():
        if metric_name == "http_req_duration":
            metrics["p50"] = metric_data["values"].get("p(50)", 0)
            metrics["p95"] = metric_data["values"].get("p(95)", 0)
            metrics["p99"] = metric_data["values"].get("p(99)", 0)
            metrics["avg"] = metric_data["values"].get("avg", 0)
        elif metric_name == "http_req_failed":
            metrics["error_rate"] = metric_data["values"].get("rate", 0)
    
    return metrics

def compare_with_baseline(
    current: dict,
    baseline: dict,
    thresholds: dict = THRESHOLDS
) -> List[dict]:
    """เปรียบเทียบ current results กับ baseline"""
    regressions = []
    
    for metric, threshold_multiplier in thresholds.items():
        current_value = current.get(metric, 0)
        baseline_value = baseline.get(metric, 0)
        
        if baseline_value == 0:
            continue
        
        ratio = current_value / baseline_value
        
        if ratio > threshold_multiplier:
            regressions.append({
                "metric": metric,
                "current": round(current_value, 3),
                "baseline": round(baseline_value, 3),
                "ratio": round(ratio, 3),
                "threshold": threshold_multiplier,
                "regression_pct": round((ratio - 1) * 100, 1)
            })
    
    return regressions

def generate_report(regressions: List[dict], current: dict, baseline: dict) -> str:
    """สร้าง performance report"""
    lines = []
    lines.append("## Performance Test Report")
    lines.append("")
    
    if regressions:
        lines.append("### ❌ Performance Regressions Detected!")
        lines.append("")
        lines.append("| Metric | Current | Baseline | Change | Threshold |")
        lines.append("|--------|---------|----------|--------|-----------|")
        
        for r in regressions:
            lines.append(
                f"| {r['metric']} | {r['current']}ms | {r['baseline']}ms | "
                f"+{r['regression_pct']}% | +{(r['threshold']-1)*100:.0f}% |"
            )
    else:
        lines.append("### ✅ No Performance Regressions")
    
    lines.append("")
    lines.append("### Current Metrics vs Baseline")
    lines.append("")
    lines.append("| Metric | Current | Baseline | Change |")
    lines.append("|--------|---------|----------|--------|")
    
    for metric in ["p50", "p95", "p99", "avg", "error_rate"]:
        current_val = current.get(metric, "N/A")
        baseline_val = baseline.get(metric, "N/A")
        
        if isinstance(current_val, float) and isinstance(baseline_val, float) and baseline_val > 0:
            change = ((current_val - baseline_val) / baseline_val) * 100
            change_str = f"{change:+.1f}%"
        else:
            change_str = "N/A"
        
        unit = "ms" if metric != "error_rate" else ""
        lines.append(
            f"| {metric} | {current_val}{unit} | {baseline_val}{unit} | {change_str} |"
        )
    
    return "\n".join(lines)

def main():
    if len(sys.argv) < 3:
        print("Usage: python performance_regression.py <current_results.json> <baseline_results.json>")
        sys.exit(1)
    
    current_file = sys.argv[1]
    baseline_file = sys.argv[2]
    
    current_results = load_results(current_file)
    baseline_results = load_results(baseline_file)
    
    current_metrics = extract_metrics(current_results)
    baseline_metrics = extract_metrics(baseline_results)
    
    regressions = compare_with_baseline(current_metrics, baseline_metrics)
    
    report = generate_report(regressions, current_metrics, baseline_metrics)
    print(report)
    
    # Save report
    with open("performance-report.md", "w") as f:
        f.write(report)
    
    if regressions:
        print("\n❌ Performance regressions detected!")
        sys.exit(1)
    else:
        print("\n✅ No performance regressions")
        sys.exit(0)

if __name__ == "__main__":
    main()
```

---

## 35.5 GitHub Actions Integration

```yaml
# .github/workflows/performance-tests.yml
name: Performance Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # รัน performance tests ทุกคืน
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      test_duration:
        description: 'Test duration (e.g., 5m, 10m)'
        required: false
        default: '5m'
      target_vus:
        description: 'Number of virtual users'
        required: false
        default: '50'

jobs:
  performance-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Start application stack
        run: |
          docker-compose -f docker-compose.test.yml up -d
          # รอให้ services พร้อม
          ./scripts/wait-for-services.sh
      
      - name: Setup k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 \
            --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run load tests
        run: |
          k6 run \
            --vus ${{ github.event.inputs.target_vus || '50' }} \
            --duration ${{ github.event.inputs.test_duration || '5m' }} \
            --out json=k6-results.json \
            --out influxdb=http://localhost:8086/k6 \
            tests/performance/load-test.js
        env:
          BASE_URL: http://localhost:8080
          K6_BROWSER_HEADLESS: true
        continue-on-error: true
      
      - name: Download baseline results
        uses: dawidd6/action-download-artifact@v2
        with:
          workflow: performance-tests.yml
          branch: main
          name: performance-baseline
          path: baseline/
        continue-on-error: true
      
      - name: Check performance regression
        run: |
          if [ -f baseline/k6-results.json ]; then
            python3 scripts/performance_regression.py \
              k6-results.json \
              baseline/k6-results.json
          else
            echo "No baseline found, skipping regression check"
          fi
        continue-on-error: true
      
      - name: Update baseline (main branch only)
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v3
        with:
          name: performance-baseline
          path: k6-results.json
          retention-days: 30
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: performance-test-results
          path: |
            k6-results.json
            performance-report.md
      
      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            
            let report = '## Performance Test Results\n\n';
            
            if (fs.existsSync('performance-report.md')) {
              report += fs.readFileSync('performance-report.md', 'utf8');
            } else {
              report += 'No performance report generated.';
            }
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
      
      - name: Fail if performance gate not met
        run: |
          # ตรวจสอบ k6 exit code
          if [ -f k6-exit-code ]; then
            EXIT_CODE=$(cat k6-exit-code)
            if [ "$EXIT_CODE" != "0" ]; then
              echo "Performance tests failed threshold checks!"
              exit 1
            fi
          fi
      
      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.test.yml down -v
```

---

## 35.6 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Load Test Script

สร้าง k6 load test สำหรับ Blog API:

```javascript
// แบบฝึกหัด: เติมโค้ดให้ครบ
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    // TODO: กำหนด stages ที่เหมาะสม
    // 1. Warm up: 0 -> 10 users ใน 1 นาที
    // 2. Normal load: 10 users เป็นเวลา 5 นาที
    // 3. Peak load: 10 -> 50 users ใน 2 นาที
    // 4. Ramp down: 50 -> 0 users ใน 1 นาที
  ],
  thresholds: {
    // TODO: กำหนด thresholds
    // 1. Error rate < 1%
    // 2. P95 < 500ms
    // 3. P99 < 1000ms
  }
};

export default function() {
  // TODO: เพิ่ม test scenarios
  // 1. GET /api/posts (list posts)
  // 2. GET /api/posts/:id (view post)
  // 3. POST /api/posts (create post - ต้อง auth)
  // 4. POST /api/posts/:id/comments (add comment)
}
```

### แบบฝึกหัดที่ 2: Baseline Comparison

1. รัน load test บน main branch และบันทึก baseline
2. สร้าง PR ที่มี code change ที่ทำให้ช้าลง
3. รัน load test อีกครั้งและเปรียบเทียบกับ baseline
4. ดู regression report

### แบบฝึกหัดที่ 3: Locust Distributed Testing

ตั้งค่า distributed Locust test:
1. Master node รัน Locust master
2. 3 Worker nodes
3. รัน 500 concurrent users
4. วิเคราะห์ bottlenecks จาก results

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **ประเภทของ Performance Testing**: Load, Stress, Spike, Soak testing
- **k6**: Load testing tool ที่เขียนด้วย JavaScript, เร็วและ efficient
- **Locust**: Python-based load testing ที่ยืดหยุ่นสูง
- **Performance Thresholds**: กำหนด SLO-based performance gates
- **Regression Detection**: เปรียบเทียบกับ baseline อัตโนมัติ
- **CI Integration**: รัน performance tests ใน GitHub Actions

บทต่อไป (Part 36) เราจะเรียนรู้เรื่อง **Security Testing ใน Pipeline**
