# Part 33: Integration Testing ใน Pipeline

## บทนำ

Integration Testing คือการทดสอบที่ตรวจสอบว่า components หลายตัวทำงานร่วมกันได้อย่างถูกต้อง ต่างจาก Unit Testing ที่ทดสอบแต่ละ component แยกกัน Integration Tests ทดสอบ interaction ระหว่าง components เช่น database, external APIs, message queues ในบทนี้เราจะเรียนรู้การเขียนและ integrate integration tests เข้ากับ CI/CD pipeline

## วัตถุประสงค์การเรียนรู้

- เข้าใจความแตกต่างระหว่าง Unit, Integration, และ E2E Testing
- ใช้ Testcontainers สำหรับ database integration tests
- Mock external services ด้วย WireMock
- ทำ Contract Testing ด้วย Pact
- Integrate integration tests เข้ากับ GitHub Actions
- จัดการ test environments อย่างมีประสิทธิภาพ

---

## 33.1 Testing Pyramid

```
        /\
       /E2E\          ← จำนวนน้อย, ช้า, ราคาแพง
      /------\
     /Integr. \       ← ปานกลาง
    /----------\
   / Unit Tests \     ← จำนวนมาก, เร็ว, ราคาถูก
  /--------------\
```

### ประเภทของ Integration Tests

| ประเภท | ทดสอบอะไร | เครื่องมือ |
|--------|-----------|----------|
| Database Integration | CRUD operations, transactions | Testcontainers |
| API Integration | HTTP endpoints, contracts | WireMock, Pact |
| Message Queue | Publish/consume messages | Testcontainers (Kafka/RabbitMQ) |
| Cache Integration | Cache hit/miss, TTL | Testcontainers (Redis) |
| External Service | Third-party APIs | WireMock, Mockoon |

---

## 33.2 Testcontainers

Testcontainers เป็น library ที่ให้เรารัน Docker containers ในระหว่าง tests โดยอัตโนมัติ ช่วยให้ tests เป็น reproducible และไม่ต้องพึ่งพา shared test environments

### Python - pytest + Testcontainers

```python
# requirements-test.txt
# testcontainers==3.7.1
# pytest==7.4.3
# sqlalchemy==2.0.23
# psycopg2-binary==2.9.9
# redis==5.0.1
# pytest-asyncio==0.21.1

# conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from testcontainers.kafka import KafkaContainer
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker
import redis

# ===== Database Container =====

@pytest.fixture(scope="session")
def postgres_container():
    """รัน PostgreSQL container สำหรับทั้ง test session"""
    with PostgresContainer("postgres:15-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    """สร้าง SQLAlchemy engine"""
    engine = create_engine(
        postgres_container.get_connection_url(),
        echo=False
    )
    # Create tables
    from app.models import Base
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db_engine):
    """สร้าง database session สำหรับแต่ละ test"""
    connection = db_engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()
    
    yield session
    
    # Rollback หลังแต่ละ test (ไม่ต้องล้างข้อมูลด้วยตัวเอง)
    session.close()
    transaction.rollback()
    connection.close()

# ===== Redis Container =====

@pytest.fixture(scope="session")
def redis_container():
    """รัน Redis container"""
    with RedisContainer("redis:7-alpine") as container:
        yield container

@pytest.fixture(scope="session")
def redis_client(redis_container):
    """สร้าง Redis client"""
    client = redis.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379),
        decode_responses=True
    )
    yield client
    client.flushall()

# ===== Kafka Container =====

@pytest.fixture(scope="session")
def kafka_container():
    """รัน Kafka container"""
    with KafkaContainer("confluentinc/cp-kafka:7.5.0") as kafka:
        yield kafka
```

```python
# app/models.py
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Enum
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

Base = declarative_base()

class OrderStatus(str, enum.Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, nullable=False)
    status = Column(Enum(OrderStatus), default=OrderStatus.PENDING)
    total_amount = Column(Float, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    items = relationship("OrderItem", back_populates="order", cascade="all, delete-orphan")

class OrderItem(Base):
    __tablename__ = "order_items"
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"), nullable=False)
    product_id = Column(Integer, nullable=False)
    quantity = Column(Integer, nullable=False)
    price = Column(Float, nullable=False)
    
    order = relationship("Order", back_populates="items")
```

```python
# app/repositories.py
from sqlalchemy.orm import Session
from sqlalchemy import func
from typing import Optional, List
from app.models import Order, OrderItem, OrderStatus

class OrderRepository:
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, user_id: int, items: list) -> Order:
        total = sum(item['price'] * item['quantity'] for item in items)
        
        order = Order(user_id=user_id, total_amount=total)
        self.session.add(order)
        self.session.flush()  # Get ID without committing
        
        for item_data in items:
            item = OrderItem(
                order_id=order.id,
                product_id=item_data['product_id'],
                quantity=item_data['quantity'],
                price=item_data['price']
            )
            self.session.add(item)
        
        self.session.commit()
        self.session.refresh(order)
        return order
    
    def get_by_id(self, order_id: int) -> Optional[Order]:
        return self.session.get(Order, order_id)
    
    def get_by_user(self, user_id: int) -> List[Order]:
        return (
            self.session.query(Order)
            .filter(Order.user_id == user_id)
            .order_by(Order.created_at.desc())
            .all()
        )
    
    def update_status(self, order_id: int, status: OrderStatus) -> Optional[Order]:
        order = self.get_by_id(order_id)
        if order:
            order.status = status
            self.session.commit()
            self.session.refresh(order)
        return order
    
    def get_revenue_summary(self, user_id: Optional[int] = None) -> dict:
        query = self.session.query(
            func.count(Order.id).label('total_orders'),
            func.sum(Order.total_amount).label('total_revenue'),
            func.avg(Order.total_amount).label('avg_order_value')
        )
        
        if user_id:
            query = query.filter(Order.user_id == user_id)
        
        result = query.first()
        return {
            'total_orders': result.total_orders or 0,
            'total_revenue': float(result.total_revenue or 0),
            'avg_order_value': float(result.avg_order_value or 0)
        }
```

```python
# tests/integration/test_order_repository.py
import pytest
from app.repositories import OrderRepository
from app.models import OrderStatus

class TestOrderRepository:
    """Integration tests สำหรับ OrderRepository"""
    
    def test_create_order(self, db_session):
        """ทดสอบการสร้าง order พร้อม items"""
        repo = OrderRepository(db_session)
        
        items = [
            {'product_id': 1, 'quantity': 2, 'price': 100.0},
            {'product_id': 2, 'quantity': 1, 'price': 250.0}
        ]
        
        order = repo.create(user_id=1, items=items)
        
        assert order.id is not None
        assert order.user_id == 1
        assert order.total_amount == 450.0
        assert len(order.items) == 2
        assert order.status == OrderStatus.PENDING
    
    def test_get_order_by_id(self, db_session):
        """ทดสอบการดึง order ด้วย ID"""
        repo = OrderRepository(db_session)
        
        # Create order
        items = [{'product_id': 1, 'quantity': 1, 'price': 500.0}]
        created = repo.create(user_id=2, items=items)
        
        # Get by ID
        found = repo.get_by_id(created.id)
        
        assert found is not None
        assert found.id == created.id
        assert found.user_id == 2
        assert found.total_amount == 500.0
    
    def test_get_orders_by_user(self, db_session):
        """ทดสอบการดึง orders ของ user"""
        repo = OrderRepository(db_session)
        
        # Create orders สำหรับ user 3
        for i in range(3):
            repo.create(user_id=3, items=[
                {'product_id': i + 1, 'quantity': 1, 'price': 100.0 * (i + 1)}
            ])
        
        # Create order สำหรับ user อื่น
        repo.create(user_id=4, items=[
            {'product_id': 1, 'quantity': 1, 'price': 100.0}
        ])
        
        orders = repo.get_by_user(user_id=3)
        
        assert len(orders) == 3
        assert all(o.user_id == 3 for o in orders)
    
    def test_update_order_status(self, db_session):
        """ทดสอบการ update status ของ order"""
        repo = OrderRepository(db_session)
        
        items = [{'product_id': 1, 'quantity': 1, 'price': 200.0}]
        order = repo.create(user_id=5, items=items)
        
        # Update to CONFIRMED
        updated = repo.update_status(order.id, OrderStatus.CONFIRMED)
        
        assert updated.status == OrderStatus.CONFIRMED
        
        # Verify in database
        from_db = repo.get_by_id(order.id)
        assert from_db.status == OrderStatus.CONFIRMED
    
    def test_get_revenue_summary(self, db_session):
        """ทดสอบการคำนวณ revenue summary"""
        repo = OrderRepository(db_session)
        
        user_id = 6
        repo.create(user_id=user_id, items=[
            {'product_id': 1, 'quantity': 1, 'price': 1000.0}
        ])
        repo.create(user_id=user_id, items=[
            {'product_id': 2, 'quantity': 2, 'price': 500.0}
        ])
        
        summary = repo.get_revenue_summary(user_id=user_id)
        
        assert summary['total_orders'] == 2
        assert summary['total_revenue'] == 2000.0
        assert summary['avg_order_value'] == 1000.0
    
    def test_create_order_transaction_rollback(self, db_session):
        """ทดสอบว่า transaction rollback ทำงานถูกต้อง"""
        repo = OrderRepository(db_session)
        
        try:
            # สร้าง order ที่จะ fail (จำลอง)
            items = []  # Empty items
            repo.create(user_id=7, items=items)
        except Exception:
            pass
        
        # ตรวจสอบว่าไม่มีข้อมูลถูก save
        orders = repo.get_by_user(user_id=7)
        assert len(orders) == 0
```

### Java - ใช้ Testcontainers

```java
// pom.xml dependencies
/*
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
*/

// OrderRepositoryIntegrationTest.java
import org.junit.jupiter.api.*;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
@Transactional
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderService orderService;

    @Test
    @DisplayName("ควรสร้าง order ได้สำเร็จพร้อม items")
    void shouldCreateOrderWithItems() {
        // Given
        var request = CreateOrderRequest.builder()
            .userId(1L)
            .items(List.of(
                OrderItem.builder()
                    .productId(1L)
                    .quantity(2)
                    .price(new BigDecimal("100.00"))
                    .build()
            ))
            .build();
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then
        assertThat(order.getId()).isNotNull();
        assertThat(order.getUserId()).isEqualTo(1L);
        assertThat(order.getTotalAmount()).isEqualByComparingTo("200.00");
        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
    }

    @Test
    @DisplayName("ควรอัพเดต status ของ order ได้")
    void shouldUpdateOrderStatus() {
        // Given
        Order order = createTestOrder(1L);
        
        // When
        Order updated = orderService.updateStatus(order.getId(), OrderStatus.CONFIRMED);
        
        // Then
        assertThat(updated.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        
        // Verify in DB
        Order fromDb = orderRepository.findById(order.getId()).orElseThrow();
        assertThat(fromDb.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }

    @Test
    @DisplayName("ควรส่ง event ไปยัง Kafka เมื่อ order ถูก confirm")
    void shouldPublishEventWhenOrderConfirmed() throws Exception {
        // Given
        Order order = createTestOrder(2L);
        
        var consumer = createKafkaConsumer("order-events");
        
        // When
        orderService.updateStatus(order.getId(), OrderStatus.CONFIRMED);
        
        // Then
        var records = consumer.poll(Duration.ofSeconds(5));
        assertThat(records.count()).isGreaterThan(0);
        
        var event = records.iterator().next().value();
        assertThat(event.getOrderId()).isEqualTo(order.getId());
        assertThat(event.getType()).isEqualTo("ORDER_CONFIRMED");
    }

    private Order createTestOrder(Long userId) {
        var request = CreateOrderRequest.builder()
            .userId(userId)
            .items(List.of(
                OrderItem.builder()
                    .productId(1L)
                    .quantity(1)
                    .price(new BigDecimal("500.00"))
                    .build()
            ))
            .build();
        return orderService.createOrder(request);
    }
}
```

---

## 33.3 WireMock สำหรับ Service Mocking

### Python - ใช้ pytest-wiremock

```python
# conftest.py
import pytest
import requests
from wiremock.client import WireMock
from wiremock.constants import Config
from wiremock.resources.mappings import (
    MappingRequest, MappingResponse, Mapping, HttpMethods, RequestMethod
)
from testcontainers.core.container import DockerContainer

@pytest.fixture(scope="session")
def wiremock_server():
    """รัน WireMock server ใน container"""
    container = DockerContainer("wiremock/wiremock:3.3.1")
    container.with_exposed_ports(8080)
    container.start()
    
    host = container.get_container_host_ip()
    port = container.get_exposed_port(8080)
    
    Config.base_url = f"http://{host}:{port}"
    
    # รอให้ server พร้อม
    import time
    for _ in range(30):
        try:
            requests.get(f"http://{host}:{port}/__admin/")
            break
        except:
            time.sleep(1)
    
    yield WireMock()
    
    container.stop()

@pytest.fixture(autouse=True)
def reset_wiremock(wiremock_server):
    """Reset WireMock state ก่อนแต่ละ test"""
    wiremock_server.reset()
    yield
    wiremock_server.reset()
```

```python
# tests/integration/test_payment_service.py
import pytest
import json
from wiremock.resources.mappings import (
    MappingRequest, MappingResponse, Mapping, HttpMethods
)
from app.services import PaymentService

class TestPaymentServiceIntegration:
    """Integration tests สำหรับ PaymentService ที่ต้องการ external payment gateway"""
    
    def setup_payment_success_stub(self, wiremock_server, transaction_id: str):
        """สร้าง stub สำหรับ payment success"""
        mapping = Mapping(
            request=MappingRequest(
                method=HttpMethods.POST,
                url="/api/v1/payments",
                body_patterns=[{
                    "matchesJsonPath": "$.amount",
                }]
            ),
            response=MappingResponse(
                status=200,
                json_body={
                    "transaction_id": transaction_id,
                    "status": "success",
                    "gateway": "stripe",
                    "processed_at": "2024-01-15T10:00:00Z"
                }
            )
        )
        wiremock_server.create_mapping(mapping)
    
    def setup_payment_failure_stub(self, wiremock_server, error_code: str):
        """สร้าง stub สำหรับ payment failure"""
        mapping = Mapping(
            request=MappingRequest(
                method=HttpMethods.POST,
                url="/api/v1/payments"
            ),
            response=MappingResponse(
                status=402,
                json_body={
                    "error": {
                        "code": error_code,
                        "message": "Payment declined"
                    }
                }
            )
        )
        wiremock_server.create_mapping(mapping)
    
    def setup_payment_timeout_stub(self, wiremock_server):
        """สร้าง stub สำหรับ payment timeout"""
        mapping = Mapping(
            request=MappingRequest(
                method=HttpMethods.POST,
                url="/api/v1/payments"
            ),
            response=MappingResponse(
                status=200,
                fixed_delay_milliseconds=5000  # Delay 5 วินาที
            )
        )
        wiremock_server.create_mapping(mapping)
    
    def test_successful_payment(self, wiremock_server):
        """ทดสอบ payment สำเร็จ"""
        self.setup_payment_success_stub(wiremock_server, "txn-001")
        
        service = PaymentService(
            gateway_url=wiremock_server.base_url + "/api/v1/payments"
        )
        
        result = service.process_payment(
            amount=1000.0,
            currency="THB",
            card_token="tok_test_123"
        )
        
        assert result.success is True
        assert result.transaction_id == "txn-001"
        assert result.status == "success"
    
    def test_payment_declined(self, wiremock_server):
        """ทดสอบ payment ถูก decline"""
        self.setup_payment_failure_stub(wiremock_server, "CARD_DECLINED")
        
        service = PaymentService(
            gateway_url=wiremock_server.base_url + "/api/v1/payments"
        )
        
        with pytest.raises(PaymentDeclinedException) as exc_info:
            service.process_payment(
                amount=1000.0,
                currency="THB",
                card_token="tok_test_declined"
            )
        
        assert exc_info.value.error_code == "CARD_DECLINED"
    
    def test_payment_timeout_with_retry(self, wiremock_server):
        """ทดสอบ retry logic เมื่อ payment gateway timeout"""
        self.setup_payment_timeout_stub(wiremock_server)
        
        service = PaymentService(
            gateway_url=wiremock_server.base_url + "/api/v1/payments",
            timeout_seconds=2,
            max_retries=3
        )
        
        with pytest.raises(PaymentTimeoutException):
            service.process_payment(
                amount=1000.0,
                currency="THB",
                card_token="tok_test_timeout"
            )
        
        # ตรวจสอบว่า retry ถูกเรียก 3 ครั้ง
        verify = wiremock_server.get_requests_for_url("/api/v1/payments")
        assert len(verify) == 3  # 1 original + 2 retries
```

---

## 33.4 Contract Testing ด้วย Pact

Contract Testing ช่วยให้ consumer และ provider ตกลงกันเรื่อง API contract โดยไม่ต้องรัน integration tests พร้อมกัน

### Consumer Side (Python)

```python
# tests/contract/test_order_api_consumer.py
import pytest
import json
from pact import Consumer, Provider, Like, Term, EachLike

@pytest.fixture(scope="session")
def pact():
    pact = Consumer("OrderService").has_pact_with(
        Provider("ProductService"),
        host_name="localhost",
        port=8080,
        pact_dir="./pacts"
    )
    pact.start_service()
    yield pact
    pact.stop_service()

def test_get_product_by_id(pact):
    """Consumer test: ดึงข้อมูล product"""
    
    # กำหนด expected response
    expected = {
        "id": Like(1),
        "name": Like("Test Product"),
        "price": Like(100.0),
        "available": True,
        "sku": Term(r"[A-Z]+-\d+", "SKU-001")
    }
    
    # กำหนด interaction
    (pact
     .given("product with ID 1 exists")
     .upon_receiving("a request for product 1")
     .with_request("GET", "/products/1")
     .will_respond_with(200, body=expected))
    
    with pact:
        # เรียก API จริงๆ (ผ่าน Pact mock server)
        import requests
        result = requests.get("http://localhost:8080/products/1")
        
        assert result.status_code == 200
        data = result.json()
        assert "id" in data
        assert "name" in data
        assert "price" in data
        assert data["available"] is True

def test_get_product_not_found(pact):
    """Consumer test: product ไม่มีอยู่"""
    
    (pact
     .given("product with ID 999 does not exist")
     .upon_receiving("a request for non-existent product")
     .with_request("GET", "/products/999")
     .will_respond_with(404, body={
         "error": {
             "code": "PRODUCT_NOT_FOUND",
             "message": Like("Product not found")
         }
     }))
    
    with pact:
        import requests
        result = requests.get("http://localhost:8080/products/999")
        assert result.status_code == 404

def test_create_order_validates_inventory(pact):
    """Consumer test: ตรวจสอบ inventory ก่อนสร้าง order"""
    
    (pact
     .given("product 1 has 5 units in stock")
     .upon_receiving("a request to check inventory for product 1")
     .with_request(
         "GET",
         "/products/1/inventory",
         headers={"Accept": "application/json"}
     )
     .will_respond_with(200, body={
         "product_id": 1,
         "available_quantity": Like(5),
         "reserved_quantity": Like(0),
         "can_fulfill": True
     }))
    
    with pact:
        import requests
        result = requests.get(
            "http://localhost:8080/products/1/inventory",
            headers={"Accept": "application/json"}
        )
        
        assert result.status_code == 200
        data = result.json()
        assert data["can_fulfill"] is True
```

### Provider Side Verification (Go)

```go
// product_service_test.go
package main_test

import (
    "testing"
    "fmt"
    "net/http/httptest"
    
    "github.com/pact-foundation/pact-go/v2/provider"
    "github.com/pact-foundation/pact-go/v2/models"
)

func TestProductServicePactProvider(t *testing.T) {
    // เริ่ม test server
    srv := httptest.NewServer(setupRouter())
    defer srv.Close()
    
    verifier := provider.NewVerifier()
    
    err := verifier.VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL: srv.URL,
        Provider:        "ProductService",
        
        // ดึง pacts จาก Pact Broker
        BrokerURL:            "https://your-pact-broker.example.com",
        BrokerToken:          "your-broker-token",
        PublishVerificationResults: true,
        ProviderVersion:      "1.2.3",
        
        // State handlers - setup test data ตาม state
        StateHandlers: models.StateHandlers{
            "product with ID 1 exists": func(setup bool, s models.ProviderState) (models.ProviderStateResponse, error) {
                if setup {
                    // สร้าง test product
                    testDB.InsertProduct(Product{
                        ID:        1,
                        Name:      "Test Product",
                        Price:     100.0,
                        SKU:       "SKU-001",
                        Available: true,
                    })
                } else {
                    testDB.DeleteProduct(1)
                }
                return models.ProviderStateResponse{}, nil
            },
            
            "product with ID 999 does not exist": func(setup bool, s models.ProviderState) (models.ProviderStateResponse, error) {
                if setup {
                    testDB.DeleteProduct(999)  // ตรวจสอบว่าไม่มี product นี้
                }
                return models.ProviderStateResponse{}, nil
            },
            
            "product 1 has 5 units in stock": func(setup bool, s models.ProviderState) (models.ProviderStateResponse, error) {
                if setup {
                    testDB.SetInventory(1, 5, 0)  // 5 available, 0 reserved
                } else {
                    testDB.ResetInventory(1)
                }
                return models.ProviderStateResponse{}, nil
            },
        },
    })
    
    if err != nil {
        t.Fatalf("Provider verification failed: %v", err)
    }
}
```

---

## 33.5 GitHub Actions Integration

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  integration-tests:
    runs-on: ubuntu-latest
    
    # Service containers สำหรับ tests
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-test.txt
      
      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: |
          python -m alembic upgrade head
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379/0
          WIREMOCK_URL: http://localhost:8080
        run: |
          pytest tests/integration/ \
            -v \
            --junit-xml=test-results/integration-results.xml \
            --cov=app \
            --cov-report=xml:coverage.xml \
            --cov-report=html:htmlcov \
            -m "not slow"
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: integration-test-results
          path: |
            test-results/
            htmlcov/
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml
          flags: integration

  contract-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements-test.txt
      
      - name: Run consumer contract tests
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
        run: |
          pytest tests/contract/ -v --junit-xml=test-results/contract-results.xml
      
      - name: Publish pacts to broker
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
        run: |
          pact-broker publish ./pacts \
            --consumer-app-version=${{ github.sha }} \
            --branch=${{ github.ref_name }} \
            --broker-base-url=${{ env.PACT_BROKER_URL }}

  testcontainers-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements-test.txt
      
      - name: Run Testcontainers tests
        run: |
          pytest tests/integration/testcontainers/ \
            -v \
            --junit-xml=test-results/testcontainers-results.xml \
            -p no:warnings
      
      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: "test-results/**/*.xml"
```

---

## 33.6 Test Data Management

### Database Seeding

```python
# tests/fixtures/database.py
from sqlalchemy.orm import Session
from app.models import Order, OrderItem, OrderStatus
from datetime import datetime, timedelta
import random

class DatabaseFixtures:
    """จัดการ test data"""
    
    def __init__(self, session: Session):
        self.session = session
    
    def create_order(
        self,
        user_id: int = 1,
        status: OrderStatus = OrderStatus.PENDING,
        total_amount: float = None,
        items_count: int = 1
    ) -> Order:
        """สร้าง order สำหรับ testing"""
        
        items = []
        total = 0
        
        for i in range(items_count):
            price = round(random.uniform(50, 500), 2)
            quantity = random.randint(1, 5)
            total += price * quantity
            items.append({
                'product_id': i + 1,
                'quantity': quantity,
                'price': price
            })
        
        if total_amount is not None:
            total = total_amount
        
        order = Order(
            user_id=user_id,
            status=status,
            total_amount=total,
            created_at=datetime.utcnow() - timedelta(days=random.randint(0, 30))
        )
        self.session.add(order)
        self.session.flush()
        
        for item_data in items:
            item = OrderItem(
                order_id=order.id,
                **item_data
            )
            self.session.add(item)
        
        self.session.commit()
        return order
    
    def create_order_batch(self, count: int, **kwargs) -> list:
        """สร้าง orders หลายอัน"""
        return [self.create_order(**kwargs) for _ in range(count)]
    
    def create_user_with_orders(self, user_id: int, order_count: int = 5) -> dict:
        """สร้าง user พร้อม orders"""
        orders = self.create_order_batch(order_count, user_id=user_id)
        return {
            'user_id': user_id,
            'orders': orders,
            'total_spent': sum(o.total_amount for o in orders)
        }

# ใช้ใน tests
@pytest.fixture
def sample_orders(db_session):
    fixtures = DatabaseFixtures(db_session)
    return fixtures.create_order_batch(5, user_id=1)
```

---

## 33.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Database Integration Tests

สร้าง integration tests สำหรับ UserRepository:

```python
# แบบฝึกหัด
class UserRepository:
    def create_user(self, email: str, name: str) -> User: ...
    def find_by_email(self, email: str) -> Optional[User]: ...
    def update_profile(self, user_id: int, updates: dict) -> User: ...
    def delete_user(self, user_id: int) -> bool: ...
    def find_active_users(self) -> List[User]: ...

# TODO: เขียน integration tests ที่ครอบคลุม:
# 1. Create user successfully
# 2. Find user by email (found and not found)
# 3. Update user profile
# 4. Soft delete user
# 5. List active users
# 6. Email uniqueness constraint
# ใช้ Testcontainers สำหรับ PostgreSQL
```

### แบบฝึกหัดที่ 2: WireMock Integration

สร้าง integration tests สำหรับ ShippingService ที่ต้อง call external shipping API:

```python
# แบบฝึกหัด
class ShippingService:
    def calculate_shipping(self, origin_postal: str, dest_postal: str, weight_kg: float) -> ShippingRate: ...
    def create_shipment(self, order_id: str, address: Address) -> Shipment: ...
    def track_shipment(self, tracking_number: str) -> TrackingInfo: ...

# TODO: ใช้ WireMock เพื่อ stub shipping API:
# 1. Successful shipping rate calculation
# 2. Create shipment with label
# 3. Track shipment status
# 4. Handle API errors gracefully
# 5. Test retry on timeout
```

### แบบฝึกหัดที่ 3: Contract Testing

สร้าง Pact consumer test สำหรับ NotificationService:

```python
# แบบฝึกหัด
# Consumer: OrderService
# Provider: NotificationService

# TODO: เขียน consumer tests สำหรับ:
# 1. POST /notifications/email - ส่ง email notification
# 2. POST /notifications/sms - ส่ง SMS
# 3. GET /notifications/{id}/status - ดู status
# 4. รับ 4xx เมื่อ payload ไม่ถูกต้อง
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Testcontainers**: รัน real databases และ services ใน Docker containers ระหว่าง tests
- **WireMock**: Mock external HTTP services สำหรับ deterministic testing
- **Contract Testing (Pact)**: ตรวจสอบ API contracts ระหว่าง consumer และ provider
- **GitHub Actions**: Integrate integration tests เข้ากับ CI pipeline
- **Test Data Management**: จัดการ test fixtures อย่างมีประสิทธิภาพ

บทต่อไป (Part 34) เราจะเรียนรู้เรื่อง **End-to-End Testing อัตโนมัติ**
