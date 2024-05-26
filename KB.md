# KB

## 1. 보상거래

MSA(Microservices Architecture)와 같은 IT 시스템에서 보상거래를 구현하는 방식은 다음과 같은 단계를 통해 이루어질 수 있습니다. 여기서는 주로 분산 시스템에서 트랜잭션을 관리하고, 실패 시에도 시스템의 일관성을 유지하기 위해 사용하는 보상 트랜잭션(compensating transactions) 개념을 중점적으로 설명하겠습니다.

### 1. 보상 트랜잭션의 개념 이해
보상 트랜잭션은 주 트랜잭션이 실패했을 때, 그 트랜잭션에 의해 변경된 상태를 원래대로 복구하기 위한 일련의 동작입니다. 이는 분산 시스템에서 흔히 사용되는 SAGA 패턴의 일부로, SAGA 패턴은 긴 트랜잭션을 여러 개의 작은 트랜잭션으로 나누어 각각의 트랜잭션이 독립적으로 커밋될 수 있도록 합니다.

### 2. 트랜잭션 분할
주 트랜잭션을 여러 개의 독립적인 소 트랜잭션으로 분할합니다. 각 소 트랜잭션은 하나의 마이크로서비스에 의해 수행됩니다. 예를 들어, 주문 처리 시스템에서 다음과 같은 소 트랜잭션이 있을 수 있습니다:
   - 재고 감소 트랜잭션 (Inventory Service)
   - 결제 처리 트랜잭션 (Payment Service)
   - 배송 준비 트랜잭션 (Shipping Service)

### 3. 보상 트랜잭션 정의
각 소 트랜잭션에 대해 실패 시 실행할 보상 트랜잭션을 정의합니다. 예를 들어:
   - 재고 감소 트랜잭션에 대한 보상 트랜잭션은 재고 증가 (원래 상태로 복구)
   - 결제 처리 트랜잭션에 대한 보상 트랜잭션은 결제 취소
   - 배송 준비 트랜잭션에 대한 보상 트랜잭션은 배송 취소

### 4. 트랜잭션 코디네이터
MSA 환경에서는 트랜잭션을 관리하는 코디네이터가 필요합니다. 코디네이터는 각 소 트랜잭션의 상태를 추적하고, 실패 시 보상 트랜잭션을 트리거합니다. 트랜잭션 코디네이터는 다음과 같은 역할을 수행합니다:
   - 각 소 트랜잭션의 시작과 완료 여부를 추적
   - 소 트랜잭션이 실패하면 관련된 모든 보상 트랜잭션을 실행

### 5. 트랜잭션 실행 및 모니터링
트랜잭션 코디네이터는 모든 소 트랜잭션을 순차적으로 실행합니다. 모든 소 트랜잭션이 성공적으로 완료되면 주 트랜잭션을 커밋합니다. 만약 어떤 소 트랜잭션이 실패하면, 이미 완료된 소 트랜잭션에 대해 보상 트랜잭션을 실행하여 시스템의 상태를 원래대로 복구합니다.

### 6. 롤백 시나리오 처리
각 소 트랜잭션이 실패할 경우 롤백 절차를 수행합니다. 이는 보상 트랜잭션을 통해 이루어지며, 이를 통해 시스템의 일관성을 유지할 수 있습니다.

### 예시 시나리오
주문 처리 시스템의 시나리오를 통해 보상 트랜잭션을 설명해 보겠습니다.

1. **주문 생성 트랜잭션**:
   - 재고 서비스에서 재고 감소 요청
   - 결제 서비스에서 결제 처리 요청
   - 배송 서비스에서 배송 준비 요청

2. **보상 트랜잭션**:
   - 재고 서비스에서 재고 감소 실패 시 재고 복구
   - 결제 서비스에서 결제 처리 실패 시 결제 취소
   - 배송 서비스에서 배송 준비 실패 시 배송 취소

### 결론
MSA 환경에서 보상거래를 구현하는 방식은 주로 보상 트랜잭션을 통해 이루어집니다. 이는 시스템의 일관성을 유지하면서도, 각 마이크로서비스가 독립적으로 작동할 수 있도록 해줍니다. 이를 통해 분산 환경에서도 안정적이고 신뢰할 수 있는 거래 처리가 가능해집니다.


다음은 MSA 환경에서 보상 트랜잭션을 구현하는 예시 시나리오를 자바 코드로 설명하겠습니다. 여기서는 주문 처리 시스템을 예로 들어, 주문 생성 트랜잭션과 그에 대한 보상 트랜잭션을 포함한 SAGA 패턴을 구현합니다.

### 1. 서비스 인터페이스 정의

먼저, 각 마이크로서비스의 인터페이스를 정의합니다. 여기서는 재고 서비스, 결제 서비스, 배송 서비스의 인터페이스를 정의합니다.

```java
public interface InventoryService {
    void decreaseStock(String productId, int quantity) throws Exception;
    void increaseStock(String productId, int quantity) throws Exception;
}

public interface PaymentService {
    void processPayment(String orderId, double amount) throws Exception;
    void cancelPayment(String orderId) throws Exception;
}

public interface ShippingService {
    void prepareShipment(String orderId) throws Exception;
    void cancelShipment(String orderId) throws Exception;
}
```

### 2. 서비스 구현

다음으로, 각 서비스의 구현을 제공합니다. 실제 구현은 간단하게 예외를 던지거나 출력하는 방식으로 처리합니다.

```java
public class InventoryServiceImpl implements InventoryService {
    @Override
    public void decreaseStock(String productId, int quantity) throws Exception {
        System.out.println("Decreasing stock for product: " + productId);
        // 실제 비즈니스 로직 구현
        // 예외를 던져 실패를 시뮬레이션할 수 있습니다.
    }

    @Override
    public void increaseStock(String productId, int quantity) throws Exception {
        System.out.println("Increasing stock for product: " + productId);
        // 실제 비즈니스 로직 구현
    }
}

public class PaymentServiceImpl implements PaymentService {
    @Override
    public void processPayment(String orderId, double amount) throws Exception {
        System.out.println("Processing payment for order: " + orderId);
        // 실제 비즈니스 로직 구현
    }

    @Override
    public void cancelPayment(String orderId) throws Exception {
        System.out.println("Canceling payment for order: " + orderId);
        // 실제 비즈니스 로직 구현
    }
}

public class ShippingServiceImpl implements ShippingService {
    @Override
    public void prepareShipment(String orderId) throws Exception {
        System.out.println("Preparing shipment for order: " + orderId);
        // 실제 비즈니스 로직 구현
    }

    @Override
    public void cancelShipment(String orderId) throws Exception {
        System.out.println("Canceling shipment for order: " + orderId);
        // 실제 비즈니스 로직 구현
    }
}
```

### 3. 주문 처리 코디네이터

다음으로, 주문 처리 코디네이터를 구현합니다. 코디네이터는 각 서비스를 호출하여 트랜잭션을 관리하고, 실패 시 보상 트랜잭션을 실행합니다.

```java
public class OrderCoordinator {
    private InventoryService inventoryService;
    private PaymentService paymentService;
    private ShippingService shippingService;

    public OrderCoordinator(InventoryService inventoryService, PaymentService paymentService, ShippingService shippingService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.shippingService = shippingService;
    }

    public void createOrder(String orderId, String productId, int quantity, double amount) {
        try {
            inventoryService.decreaseStock(productId, quantity);
            paymentService.processPayment(orderId, amount);
            shippingService.prepareShipment(orderId);
            System.out.println("Order created successfully: " + orderId);
        } catch (Exception e) {
            System.err.println("Failed to create order: " + e.getMessage());
            compensateOrder(orderId, productId, quantity);
        }
    }

    private void compensateOrder(String orderId, String productId, int quantity) {
        try {
            shippingService.cancelShipment(orderId);
        } catch (Exception e) {
            System.err.println("Failed to cancel shipment: " + e.getMessage());
        }
        try {
            paymentService.cancelPayment(orderId);
        } catch (Exception e) {
            System.err.println("Failed to cancel payment: " + e.getMessage());
        }
        try {
            inventoryService.increaseStock(productId, quantity);
        } catch (Exception e) {
            System.err.println("Failed to increase stock: " + e.getMessage());
        }
        System.out.println("Order compensation completed for order: " + orderId);
    }
}
```

### 4. 실행 예제

마지막으로, 주문 처리 코디네이터를 사용하는 예제를 작성합니다.

```java
public class OrderServiceApplication {
    public static void main(String[] args) {
        InventoryService inventoryService = new InventoryServiceImpl();
        PaymentService paymentService = new PaymentServiceImpl();
        ShippingService shippingService = new ShippingServiceImpl();
        
        OrderCoordinator orderCoordinator = new OrderCoordinator(inventoryService, paymentService, shippingService);

        String orderId = "order123";
        String productId = "product456";
        int quantity = 2;
        double amount = 100.0;

        orderCoordinator.createOrder(orderId, productId, quantity, amount);
    }
}
```

이 예제는 기본적인 SAGA 패턴을 사용하여 분산 트랜잭션을 관리하는 방법을 보여줍니다. 실제 시스템에서는 더 복잡한 로직과 오류 처리, 로깅, 상태 관리 등을 포함할 수 있습니다.

LRA(LRA, Long Running Actions) 코디네이터는 마이크로서비스 아키텍처(MSA) 환경에서 긴 실행 시간을 가지는 트랜잭션을 관리하기 위해 사용되는 컴포넌트입니다. LRA는 SAGA 패턴의 일종으로, 분산 트랜잭션을 더 유연하고 안정적으로 처리할 수 있도록 돕습니다. LRA 코디네이터는 이러한 트랜잭션의 전체 라이프사이클을 관리하고, 각 서비스의 상태를 추적하며, 필요할 때 보상 트랜잭션을 실행합니다.

### LRA 코디네이터의 역할

1. **트랜잭션 시작 및 종료 관리**:
   - LRA 코디네이터는 새로운 트랜잭션을 시작하고 고유한 트랜잭션 ID를 생성합니다.
   - 트랜잭션이 성공적으로 완료되면 코디네이터는 트랜잭션을 종료합니다.

2. **참여자 관리**:
   - 각 마이크로서비스는 자신이 관리하는 리소스에 대한 트랜잭션에 참여하게 됩니다. LRA 코디네이터는 이러한 참여자들을 등록하고 추적합니다.
   - 참여자들은 트랜잭션의 성공 또는 실패 여부에 따라 적절한 조치를 취하도록 코디네이터와 통신합니다.

3. **상태 추적**:
   - 코디네이터는 트랜잭션의 현재 상태를 추적합니다. 이는 각 참여자의 상태를 포함하여 트랜잭션의 전반적인 진행 상황을 관리합니다.
   - 트랜잭션이 성공적으로 완료되었는지, 실패했는지, 아니면 중단 상태에 있는지 등을 추적합니다.

4. **보상 트랜잭션 관리**:
   - 만약 트랜잭션이 실패하면, 코디네이터는 보상 트랜잭션을 실행하여 각 참여자가 자신의 상태를 원래대로 복구하도록 합니다.

### LRA 코디네이터 예시 (자바 코드)

LRA 코디네이터를 사용하여 긴 실행 시간을 가지는 트랜잭션을 관리하는 간단한 예시를 자바 코드로 설명하겠습니다. 여기서는 Narayana LRA 라이브러리를 사용합니다.

1. **의존성 추가 (Maven)**:
   
   ```xml
   <dependency>
       <groupId>org.eclipse.microprofile.lra</groupId>
       <artifactId>microprofile-lra-api</artifactId>
       <version>1.0</version>
   </dependency>
   ```

2. **LRA 코디네이터 설정**:

   ```java
   import org.eclipse.microprofile.lra.annotation.Compensate;
   import org.eclipse.microprofile.lra.annotation.LRA;
   import org.eclipse.microprofile.lra.annotation.ParticipantStatus;
   import org.eclipse.microprofile.lra.annotation.Status;

   import javax.ws.rs.POST;
   import javax.ws.rs.Path;
   import javax.ws.rs.core.Response;
   import java.net.URI;

   @Path("/order")
   public class OrderService {

       @POST
       @Path("/create")
       @LRA(value = LRA.Type.REQUIRES_NEW)
       public Response createOrder(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId) {
           try {
               // 서비스 로직 실행
               // 예: 재고 감소, 결제 처리, 배송 준비 등
               return Response.ok().build();
           } catch (Exception e) {
               return Response.status(Response.Status.INTERNAL_SERVER_ERROR).build();
           }
       }

       @Compensate
       @Path("/compensate")
       public Response compensate(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId) {
           try {
               // 보상 로직 실행
               // 예: 재고 복구, 결제 취소, 배송 취소 등
               return Response.ok(ParticipantStatus.Compensated).build();
           } catch (Exception e) {
               return Response.status(Response.Status.INTERNAL_SERVER_ERROR).build();
           }
       }

       @Status
       @Path("/status")
       public Response status(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId) {
           // 상태 확인 로직
           return Response.ok(ParticipantStatus.Completed).build();
       }
   }
   ```

### LRA 트랜잭션 실행 예시

```java
import javax.ws.rs.client.ClientBuilder;
import javax.ws.rs.client.Entity;
import javax.ws.rs.core.Response;

public class OrderClient {
    public static void main(String[] args) {
        Response response = ClientBuilder.newClient()
                .target("http://localhost:8080/order/create")
                .request()
                .post(Entity.text(""));

        if (response.getStatus() == Response.Status.OK.getStatusCode()) {
            System.out.println("Order created successfully.");
        } else {
            System.out.println("Failed to create order.");
        }
    }
}
```

### 요약

LRA 코디네이터는 MSA 환경에서 긴 실행 시간을 가지는 트랜잭션을 관리하기 위한 중요한 컴포넌트입니다. 이를 통해 각 서비스가 독립적으로 동작하면서도, 전체 트랜잭션의 일관성을 유지할 수 있습니다. 코디네이터는 트랜잭션의 상태를 추적하고, 실패 시 보상 트랜잭션을 실행하여 시스템의 안정성을 보장합니다.
