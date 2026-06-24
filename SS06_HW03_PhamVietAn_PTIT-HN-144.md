# BÀI 3: Thực hành Refactor & Nâng cấp Giao dịch (Refinement Process - Robustness & Logging)

## 1. Phân tích các lỗ hổng nghiêm trọng của đoạn code thô ban đầu

Đoạn code ban đầu chứa các lỗi rất nghiêm trọng về an toàn dữ liệu, tính toàn vẹn hệ thống và khả năng bảo trì:

1.  **Lỗi Null Pointer Exception (NPE) tiềm ẩn:**
    *   Dòng code `Product product = inventoryRepository.findById(item.getProductId()).orElse(null);` sẽ trả về `null` nếu ID sản phẩm không tồn tại trong database.
    *   Ngay dòng tiếp theo `product.setStock(...)` sẽ quăng ra `NullPointerException`, làm sập luồng xử lý của hệ thống mà không có thông điệp báo lỗi rõ ràng.

2.  **Không kiểm tra số lượng tồn kho (Out of Stock):**
    *   Hệ thống trực tiếp thực hiện phép tính trừ kho: `product.setStock(product.getStock() - item.getQuantity())` mà không kiểm tra xem số lượng mua `item.getQuantity()` có vượt quá số lượng tồn kho hiện tại `product.getStock()` hay không.
    *   Điều này dẫn đến tình trạng kho bị âm (negative stock), gây thất thoát tài chính, sai lệch kiểm kho vật lý và hư hại tính nhất quán dữ liệu.

3.  **Thiếu tính nguyên tử của giao dịch (Atomicity - Transaction Management):**
    *   Đây là lỗi nghiêm trọng nhất. Thao tác trừ kho được lưu xuống database (`inventoryRepository.save(product)`) trước khi thực hiện thanh toán qua cổng thanh toán (`paymentGateway.charge()`).
    *   Nếu khách hàng không đủ tiền hoặc cổng thanh toán bị sập đột ngột, một ngoại lệ sẽ được ném ra và luồng xử lý bị dừng. Tuy nhiên, vì phương thức `placeOrder` không có chú thích `@Transactional`, các thay đổi trừ kho đã được commit xuống database trước đó sẽ không được hoàn tác (Rollback). Kết quả: sản phẩm trong kho bị mất nhưng cửa hàng không nhận được tiền và không có đơn hàng nào được tạo.

4.  **Thiếu kiểm tra tính hợp lệ đầu vào (Input Validation):**
    *   Phương thức không kiểm tra xem đối tượng `Order` hoặc danh sách `order.getItems()` có bị `null` hoặc rỗng không. Số lượng mua của từng mặt hàng có hợp lệ (> 0) hay không.

5.  **Thiếu cơ chế Ghi log hệ thống (Logging):**
    *   Không có bất kỳ dòng log nào được ghi lại để theo dõi vết hoạt động (audit trail). Khi giao dịch thất bại, lập trình viên sẽ không có thông tin (như mã khách hàng, mã sản phẩm bị lỗi, thông điệp lỗi của Gateway) để tiến hành xử lý sự cố.

6.  **Thiếu DI (Dependency Injection) chuẩn mực:**
    *   Các thuộc tính repository và gateway được khai báo trực tiếp mà không thông qua constructor injection, gây khó khăn cho việc viết Unit Test sử dụng Mockito.

---

## 2. Chi tiết nội dung 3 lượt Prompt tương ứng với 3 vòng cải tiến

### Vòng 1: Củng cố tính bền vững (Robustness)
```markdown
[ROLE]
Bạn là một Senior Java Developer chuyên trách về xử lý an toàn dữ liệu và tối ưu hóa giao dịch.

[GOAL]
Hãy refactor phương thức `placeOrder` của lớp `OrderPlacementService` dưới đây để củng cố tính bền vững (Robustness).

[CONTEXT]
Đoạn code thô hiện tại:
```java
public class OrderPlacementService {
    private InventoryRepository inventoryRepository;
    private PaymentGateway paymentGateway;
    private OrderRepository orderRepository;

    public void placeOrder(Order order) {
        // Trừ kho
        for (OrderItem item : order.getItems()) {
            Product product = inventoryRepository.findById(item.getProductId()).orElse(null);
            product.setStock(product.getStock() - item.getQuantity());
            inventoryRepository.save(product);
        }
        // Thanh toán qua Gateway
        paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
        // Lưu đơn hàng
        orderRepository.save(order);
    }
}
```

[CONSTRAINTS]
Yêu cầu refactor trong vòng này:
1. Kiểm tra tính hợp lệ dữ liệu đầu vào:
   - Nếu đối tượng `order` hoặc danh sách items của order bị null hoặc rỗng, ném ra ngoại lệ `IllegalArgumentException` với thông báo phù hợp.
2. Kiểm tra tồn kho an toàn:
   - Tìm kiếm sản phẩm trong database, nếu không tìm thấy phải ném ra `ProductNotFoundException` (tạo class ngoại lệ này nếu cần).
   - Kiểm tra xem sản phẩm còn hàng không và số lượng tồn kho có đủ đáp ứng số lượng mua không. Nếu không đủ, ném ra ngoại lệ `OutOfStockException` (chứa thông tin productId và số lượng tồn kho còn lại).
3. Bắt lỗi cổng thanh toán:
   - Bao bọc lời gọi `paymentGateway.charge(...)` trong khối try-catch để phát hiện lỗi từ bên thứ ba. Nếu thanh toán thất bại, ném ra ngoại lệ tùy chỉnh `PaymentFailedException` (chứa thông điệp chi tiết của lỗi).

[FORMAT]
Trả về mã nguồn Java của lớp `OrderPlacementService` sau khi refactor cùng các class Exception tùy chỉnh đi kèm.
```

---

### Vòng 2: Áp dụng chuẩn bảo trì & Viết code sạch (Maintainability & Clean Code)
```markdown
[ROLE]
Bạn là một Technical Lead phụ trách tiêu chuẩn Clean Code và kiến trúc mã nguồn của dự án Spring Boot.

[GOAL]
Hãy tiếp tục cải tiến mã nguồn `OrderPlacementService` từ Vòng 1 để đạt tiêu chuẩn bảo trì và Clean Code của doanh nghiệp.

[CONSTRAINTS]
1. Quản lý giao dịch (Transaction Management):
   - Thêm chú thích `@Transactional` từ thư viện Spring Framework để đảm bảo tính ACID. Đặc biệt là tính nguyên tử (Atomicity): Nếu bất kỳ lỗi nào xảy ra trong quá trình trừ kho, gọi gateway thanh toán hoặc lưu đơn hàng, toàn bộ giao dịch phải được rollback hoàn toàn (bao gồm việc hoàn lại số lượng tồn kho đã trừ trước đó). Cần khai báo rõ `rollbackFor = Exception.class`.
2. Tích hợp Logging:
   - Sử dụng chú thích `@Slf4j` của thư viện Lombok để thực hiện ghi log.
   - Ghi log ở cấp độ INFO khi bắt đầu xử lý đơn hàng, khi trừ kho thành công, khi gọi thanh toán thành công và khi hoàn thành lưu đơn hàng.
   - Ghi log ở cấp độ ERROR kèm theo Stack Trace khi xảy ra các ngoại lệ (`OutOfStockException`, `PaymentFailedException`, v.v.).
3. Thay thế Field Injection bằng Constructor Injection:
   - Sử dụng chú thích `@RequiredArgsConstructor` của Lombok để Spring tự động inject các bean `InventoryRepository`, `PaymentGateway`, và `OrderRepository` thông qua constructor. Đánh dấu các trường này là `final`.

[FORMAT]
Trả về mã nguồn Java sạch đẹp của lớp `OrderPlacementService` sau khi tích hợp `@Transactional`, `@Slf4j`, `@RequiredArgsConstructor`.
```

---

### Vòng 3: Tối ưu theo ngữ cảnh doanh nghiệp & Viết Unit Test (Context-specific Tuning)
```markdown
[ROLE]
Bạn là một Senior QA Automation Engineer kiêm Senior Spring Boot Developer.

[GOAL]
Hãy hoàn thiện lớp `OrderPlacementService` bằng cách tối ưu hóa kiểu dữ liệu trả về và viết một lớp JUnit 5 Test hoàn chỉnh sử dụng thư viện Mockito để kiểm thử các kịch bản.

[CONSTRAINTS]
1. Tối ưu hóa kết quả trả về:
   - Thay vì trả về kiểu `void`, phương thức `placeOrder` sẽ trả về đối tượng `OrderPlacementResult` chứa các thông tin: `orderId` (String), `status` (String - SUCCESS/FAILED), và `message` (String).
2. Viết JUnit Test:
   - Viết một class kiểm thử tên `OrderPlacementServiceTest` sử dụng JUnit 5 và Mockito.
   - Sử dụng `@Mock` để giả lập hành vi của `InventoryRepository`, `PaymentGateway`, và `OrderRepository`.
   - Sử dụng `@InjectMocks` để đưa các mock này vào `OrderPlacementService`.
   - Viết 2 Test Cases quan trọng:
     + Test Case 1: Đặt đơn hàng thành công (tất cả các bước chạy mượt mà, trả về kết quả SUCCESS).
     + Test Case 2: Thanh toán thất bại (Giả lập `paymentGateway.charge(...)` ném ra exception. Đảm bảo phương thức `placeOrder` ném ra `PaymentFailedException` và kiểm tra việc mock repository/gateway hoạt động đúng).
3. Đảm bảo mã nguồn hoàn chỉnh, không viết mã giả (pseudo-code), có đầy đủ import.

[FORMAT]
Hãy trình bày mã nguồn Java hoàn chỉnh của hai lớp: `OrderPlacementService` và `OrderPlacementServiceTest`.
```

---

## 3. Minh chứng chạy thực tế ở lượt chat cuối cùng

Dưới đây là toàn bộ mã nguồn Java chuẩn doanh nghiệp được tạo ra sau vòng cải tiến thứ 3:

### 1. Các ngoại lệ tùy chỉnh (Custom Exceptions):

```java
package com.speedycart.exception;

public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String message) {
        super(message);
    }
}
```

```java
package com.speedycart.exception;

public class OutOfStockException extends RuntimeException {
    public OutOfStockException(String message) {
        super(message);
    }
}
```

```java
package com.speedycart.exception;

public class PaymentFailedException extends RuntimeException {
    public PaymentFailedException(String message) {
        super(message);
    }
}
```

---

### 2. Các đối tượng Model liên quan:

```java
package com.speedycart.model;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.util.List;

@Data
@Builder
public class Order {
    private String id;
    private String customerId;
    private BigDecimal totalAmount;
    private List<OrderItem> items;
}
```

```java
package com.speedycart.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class OrderItem {
    private String productId;
    private int quantity;
}
```

```java
package com.speedycart.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class Product {
    private String id;
    private String name;
    private int stock;
}
```

```java
package com.speedycart.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class OrderPlacementResult {
    private String orderId;
    private String status;
    private String message;
}
```

---

### 3. Các Repositories và Gateway (Interfaces Mock):

```java
package com.speedycart.repository;

import com.speedycart.model.Product;
import java.util.Optional;

public interface InventoryRepository {
    Optional<Product> findById(String id);
    void save(Product product);
}
```

```java
package com.speedycart.repository;

import com.speedycart.model.Order;

public interface OrderRepository {
    void save(Order order);
}
```

```java
package com.speedycart.gateway;

import java.math.BigDecimal;

public interface PaymentGateway {
    void charge(String customerId, BigDecimal amount);
}
```

---

### 4. Mã nguồn Java `OrderPlacementService.java` tối ưu hoàn chỉnh:

```java
package com.speedycart.service;

import com.speedycart.exception.*;
import com.speedycart.gateway.PaymentGateway;
import com.speedycart.model.*;
import com.speedycart.repository.InventoryRepository;
import com.speedycart.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderPlacementService {

    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    /**
     * Thực hiện đặt đơn hàng của SpeedyCart.
     * Transactional đảm bảo tính Atomicity: Nếu lỗi xảy ra, việc trừ kho và lưu đơn sẽ tự động rollback.
     */
    @Transactional(rollbackFor = Exception.class)
    public OrderPlacementResult placeOrder(Order order) {
        log.info("Bắt đầu xử lý đặt đơn hàng cho khách hàng: {}", order != null ? order.getCustomerId() : "N/A");

        // 1. Kiểm tra dữ liệu đầu vào (Validation)
        if (order == null) {
            log.error("Thông tin đơn hàng (Order) bị null");
            throw new IllegalArgumentException("Thông tin đơn hàng không hợp lệ (null).");
        }
        if (order.getItems() == null || order.getItems().isEmpty()) {
            log.error("Giỏ hàng của đơn hàng: {} trống rỗng", order.getId());
            throw new IllegalArgumentException("Giỏ hàng không có sản phẩm.");
        }

        try {
            // 2. Trừ tồn kho (Inventory update)
            for (OrderItem item : order.getItems()) {
                Product product = inventoryRepository.findById(item.getProductId())
                        .orElseThrow(() -> {
                            log.error("Không tìm thấy sản phẩm với ID: {} trong kho", item.getProductId());
                            return new ProductNotFoundException("Sản phẩm " + item.getProductId() + " không tồn tại.");
                        });

                if (product.getStock() < item.getQuantity()) {
                    log.error("Sản phẩm {} hết hàng hoặc không đủ. Tồn kho hiện tại: {}, Số lượng mua: {}",
                            product.getId(), product.getStock(), item.getQuantity());
                    throw new OutOfStockException("Sản phẩm " + product.getName() + " không đủ tồn kho. Tồn: " + product.getStock());
                }

                int newStock = product.getStock() - item.getQuantity();
                product.setStock(newStock);
                inventoryRepository.save(product);
                log.info("Cập nhật kho thành công cho sản phẩm {}. Tồn kho mới: {}", product.getId(), newStock);
            }

            // 3. Tiến hành thanh toán qua Gateway
            log.info("Yêu cầu cổng thanh toán trừ số tiền: {} cho khách hàng: {}", order.getTotalAmount(), order.getCustomerId());
            try {
                paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
                log.info("Thanh toán thành công qua cổng thanh toán cho đơn hàng: {}", order.getId());
            } catch (Exception e) {
                log.error("Lỗi xảy ra trong quá trình gọi cổng thanh toán cho đơn hàng: {}. Lỗi: {}", order.getId(), e.getMessage());
                throw new PaymentFailedException("Giao dịch thanh toán thất bại: " + e.getMessage());
            }

            // 4. Lưu thông tin đơn hàng
            orderRepository.save(order);
            log.info("Lưu đơn hàng thành công vào Database. Đơn hàng ID: {}", order.getId());

            return OrderPlacementResult.builder()
                    .orderId(order.getId())
                    .status("SUCCESS")
                    .message("Đặt đơn hàng thành công.")
                    .build();

        } catch (Exception e) {
            log.error("Đặt đơn hàng thất bại. Đơn hàng ID: {}. Lý do: {}", order.getId(), e.getMessage());
            // Việc ném RuntimeException ở đây sẽ kích hoạt cơ chế rollback của Spring @Transactional
            throw e;
        }
    }
}
```

---

### 5. Lớp kiểm thử JUnit 5 Mockito `OrderPlacementServiceTest.java`:

```java
package com.speedycart.service;

import com.speedycart.exception.PaymentFailedException;
import com.speedycart.exception.OutOfStockException;
import com.speedycart.gateway.PaymentGateway;
import com.speedycart.model.*;
import com.speedycart.repository.InventoryRepository;
import com.speedycart.repository.OrderRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Collections;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
public class OrderPlacementServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderPlacementService orderPlacementService;

    private Order validOrder;
    private Product validProduct;

    @BeforeEach
    public void setUp() {
        OrderItem item = OrderItem.builder()
                .productId("PROD01")
                .quantity(2)
                .build();

        validOrder = Order.builder()
                .id("ORD100")
                .customerId("CUST99")
                .totalAmount(new BigDecimal("250000"))
                .items(Collections.singletonList(item))
                .build();

        validProduct = Product.builder()
                .id("PROD01")
                .name("Sản phẩm Demo")
                .stock(10)
                .build();
    }

    @Test
    public void testPlaceOrder_Success() {
        // Given
        when(inventoryRepository.findById("PROD01")).thenReturn(Optional.of(validProduct));
        doNothing().when(paymentGateway).charge("CUST99", new BigDecimal("250000"));
        doNothing().when(orderRepository).save(any(Order.class));

        // When
        OrderPlacementResult result = orderPlacementService.placeOrder(validOrder);

        // Then
        assertNotNull(result);
        assertEquals("SUCCESS", result.getStatus());
        assertEquals("ORD100", result.getOrderId());
        assertEquals(8, validProduct.getStock()); // Tồn kho giảm từ 10 xuống 8

        verify(inventoryRepository, times(1)).findById("PROD01");
        verify(inventoryRepository, times(1)).save(validProduct);
        verify(paymentGateway, times(1)).charge("CUST99", new BigDecimal("250000"));
        verify(orderRepository, times(1)).save(validOrder);
    }

    @Test
    public void testPlaceOrder_PaymentFailed_ShouldThrowException() {
        // Given
        when(inventoryRepository.findById("PROD01")).thenReturn(Optional.of(validProduct));
        // Giả lập cổng thanh toán ném ra ngoại lệ Runtime
        doThrow(new RuntimeException("Kết nối timeout")).when(paymentGateway)
                .charge("CUST99", new BigDecimal("250000"));

        // When & Then
        PaymentFailedException exception = assertThrows(PaymentFailedException.class, () -> {
            orderPlacementService.placeOrder(validOrder);
        });

        assertTrue(exception.getMessage().contains("Giao dịch thanh toán thất bại"));
        assertEquals(8, validProduct.getStock()); // Tồn kho của instance product vẫn bị giảm trong bộ nhớ (nhưng DB sẽ được rollback nhờ @Transactional)
        
        // Xác minh lưu đơn hàng CHƯA từng được gọi do gặp lỗi thanh toán trước
        verify(orderRepository, never()).save(any(Order.class));
    }
    
    @Test
    public void testPlaceOrder_OutOfStock_ShouldThrowException() {
        // Given
        validProduct.setStock(1); // Set tồn kho là 1, trong khi khách mua 2
        when(inventoryRepository.findById("PROD01")).thenReturn(Optional.of(validProduct));

        // When & Then
        OutOfStockException exception = assertThrows(OutOfStockException.class, () -> {
            orderPlacementService.placeOrder(validOrder);
        });

        assertTrue(exception.getMessage().contains("không đủ tồn kho"));
        assertEquals(1, validProduct.getStock()); // Tồn kho không đổi

        verify(paymentGateway, never()).charge(anyString(), any(BigDecimal.class));
        verify(orderRepository, never()).save(any(Order.class));
    }
}
```
```
