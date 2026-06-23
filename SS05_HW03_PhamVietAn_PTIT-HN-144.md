# BÀI 3: Đọc hiểu & Dò lỗi qua Prompt (Quy trình Cải tiến đầu ra - Robustness & Maintainability)

## 1. Phân tích các lỗ hổng nghiêm trọng của đoạn code ban đầu
Đoạn mã nguồn do lập trình viên tập sự viết tuy có thể chạy thành công trong điều kiện lý tưởng (Happy Path), nhưng chứa đựng các lỗ hổng bảo mật và rủi ro vận hành nghiêm trọng khi đưa lên môi trường Production:

1. **Thiếu quản lý Giao dịch (No Transaction Management):**
   - Phương thức `transfer` thực hiện trừ tiền tài khoản A và cộng tiền tài khoản B thông qua hai câu lệnh `save()` độc lập. Nếu hệ thống sập nguồn, mất kết nối cơ sở dữ liệu hoặc xảy ra lỗi runtime ngay sau khi lưu tài khoản `from` nhưng trước khi lưu tài khoản `to`, tiền của người gửi sẽ biến mất còn người nhận không nhận được gì. Điều này vi phạm nghiêm trọng tính nguyên tử (Atomicity) trong ACID.
2. **Lỗi NullPointerException (NPE) tiềm ẩn:**
   - Đoạn code sử dụng `.orElse(null)` khi tìm kiếm tài khoản. Nếu truyền vào một `fromAccountId` không tồn tại, biến `from` sẽ bằng `null`. Dòng code `from.setBalance(...)` tiếp theo sẽ ném ra `NullPointerException`, làm crash luồng xử lý và trả về lỗi 500 đỏ lòm cho khách hàng.
3. **Thiếu kiểm tra số dư (Insufficient Balance Validation):**
   - Code không kiểm tra số dư hiện tại của tài khoản gửi. Nếu tài khoản gửi chỉ có 1 triệu mà chuyển đi 5 triệu, số dư sẽ bị trừ thành âm 4 triệu (`1M - 5M = -4M`). Đây là lỗi logic kinh điển trong thanh toán.
4. **Thiếu kiểm tra tính hợp lệ của số tiền chuyển (Negative Amount Validation):**
   - Tham số `amount` sử dụng kiểu `double` mà không kiểm tra số âm hay bằng không. Nếu kẻ tấn công hoặc người dùng vô tình truyền vào `-2,000,000` VND, tài khoản gửi sẽ được cộng thêm tiền (`from.getBalance() - (-2M) = +2M`) còn tài khoản nhận sẽ bị trừ tiền!
5. **Không bắt ngoại lệ (No Exception Handling) và thiếu Logging:**
   - Hệ thống không có bất kỳ dòng log nào để ghi nhận vết giao dịch (ai chuyển cho ai, bao nhiêu tiền, lúc nào). Nếu xảy ra lỗi hoặc tranh chấp, đội ngũ IT hoàn toàn mù tịt thông tin. Ngoại lệ không được đóng gói lại thành các Custom Business Exception mà để mặc lỗi hệ thống thô rỉ ra ngoài.

---

## 2. Thiết kế Chuỗi Prompt Cải tiến đầu ra gồm 3 vòng liên tiếp

### Vòng 1 (Robustness):
```text
Hãy đóng vai trò là Senior Java Developer. Nhiệm vụ của bạn là cải tiến tính bền vững (Robustness) của phương thức chuyển tiền (transfer) dưới đây bằng cách kiểm tra nghiêm ngặt dữ liệu đầu vào.

Đoạn code gốc:
public class AccountService {
    private AccountRepository accountRepository;

    public void transfer(Long fromAccountId, Long toAccountId, double amount) {
        Account from = accountRepository.findById(fromAccountId).orElse(null);
        Account to = accountRepository.findById(toAccountId).orElse(null);
       
        from.setBalance(from.getBalance() - amount);
        to.setBalance(to.getBalance() + amount);
       
        accountRepository.save(from);
        accountRepository.save(to);
    }
}

Yêu cầu cụ thể:
1. Kiểm tra null cho hai đối tượng tài khoản `from` và `to`. Nếu bất kỳ tài khoản nào không tồn tại trong Database, hãy ném ngoại lệ tùy chỉnh tương ứng: `AccountNotFoundException("Không tìm thấy tài khoản với ID: " + id)`.
2. Kiểm tra số tiền chuyển `amount`. Nếu `amount` <= 0, hãy ném ngoại lệ `IllegalArgumentException("Số tiền chuyển tiền phải lớn hơn 0")`.
3. Kiểm tra số dư tài khoản gửi. Nếu số dư hiện tại nhỏ hơn số tiền chuyển (`from.getBalance() < amount`), hãy ném ngoại lệ tự định nghĩa `InsufficientBalanceException("Tài khoản gửi không đủ số dư để thực hiện giao dịch")`.
4. Định nghĩa sẵn 2 lớp Custom Exception là `AccountNotFoundException` và `InsufficientBalanceException` kế thừa từ `RuntimeException`.
```

### Vòng 2 (Maintainability/Clean Code):
```text
Dựa trên mã nguồn đã cải tiến ở Vòng 1, hãy tiếp tục refactor mã nguồn theo tiêu chuẩn Clean Code và khả năng bảo trì cao:

Yêu cầu cụ thể:
1. Tích hợp thư viện Lombok để loại bỏ code boilerplate (getter, setter, constructor, v.v.). Sử dụng `@RequiredArgsConstructor` cho class `AccountService` để tự động tiêm dependencies (Dependency Injection) của `AccountRepository` mà không cần viết constructor thủ công.
2. Thêm Annotation `@Transactional` (Spring Boot) lên phương thức `transfer` để quản lý giao dịch. Đảm bảo bất cứ lỗi runtime nào xảy ra thì toàn bộ các thay đổi số dư trong database sẽ tự động rollback về trạng thái ban đầu, bảo vệ tính toàn vẹn dữ liệu.
3. Tích hợp ghi log chuyên nghiệp bằng cách sử dụng `@Slf4j` của Lombok. Ghi log cấp độ INFO khi bắt đầu giao dịch, INFO khi giao dịch thành công, và ERROR (kèm thông tin chi tiết) khi giao dịch thất bại.
```

### Vòng 3 (Context-specific Tuning):
```text
Dựa trên mã nguồn đã tối ưu ở Vòng 2, hãy điều chỉnh kiểu dữ liệu trả về theo quy chuẩn API và viết bộ unit test để đảm bảo chất lượng code trước khi deploy lên Production:

Yêu cầu cụ thể:
1. Thay đổi kiểu trả về của phương thức `transfer` từ `void` thành đối tượng kết quả chuẩn hóa `TransactionResult`. Đối tượng này cần chứa các trường: `success` (boolean), `transactionId` (String - tự sinh dạng UUID), `message` (String), và `timestamp` (LocalDateTime).
2. Viết một JUnit Test Class (sử dụng JUnit 5 và Mockito) có tên là `AccountServiceTest` để kiểm thử phương thức `transfer`.
3. Trong Test Class, hãy viết một test case cụ thể kiểm tra trường hợp chuyển tiền THẤT BẠI do tài khoản nguồn không đủ số dư. Xác nhận rằng:
   - Ngoại lệ `InsufficientBalanceException` được ném ra.
   - Phương thức `accountRepository.save()` KHÔNG bao giờ được gọi cho cả hai tài khoản (để đảm bảo không có giao dịch nửa chừng nào được lưu xuống DB).
```

---

## 3. Mã nguồn Java và Test Case JUnit tối ưu hoàn chỉnh do AI sinh ra

Dưới đây là tập hợp mã nguồn hoàn chỉnh đã trải qua 3 vòng cải tiến bằng AI:

### 1. Các lớp Custom Exception (`AccountNotFoundException.java`, `InsufficientBalanceException.java`)
```java
package com.example.bank.exception;

public class AccountNotFoundException extends RuntimeException {
    public AccountNotFoundException(String message) {
        super(message);
    }
}
```

```java
package com.example.bank.exception;

public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}
```

### 2. Đối tượng kết quả giao dịch (`TransactionResult.java`)
```java
package com.example.bank.dto;

import lombok.Builder;
import lombok.Getter;
import java.time.LocalDateTime;

@Getter
@Builder
public class TransactionResult {
    private final boolean success;
    private final String transactionId;
    private final String message;
    private final LocalDateTime timestamp;
}
```

### 3. Thực thể tài khoản đơn giản (`Account.java`)
```java
package com.example.bank.entity;

import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Account {
    private Long id;
    private double balance;
}
```

### 4. Lớp xử lý nghiệp vụ tối ưu (`AccountService.java`)
```java
package com.example.bank.service;

import com.example.bank.dto.TransactionResult;
import com.example.bank.entity.Account;
import com.example.bank.exception.AccountNotFoundException;
import com.example.bank.exception.InsufficientBalanceException;
import com.example.bank.repository.AccountRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;

    /**
     * Thực hiện chuyển tiền giữa 2 tài khoản ngân hàng
     *
     * @param fromAccountId ID tài khoản chuyển
     * @param toAccountId   ID tài khoản nhận
     * @param amount        Số tiền chuyển
     * @return Kết quả giao dịch TransactionResult
     */
    @Transactional(rollbackFor = Exception.class)
    public TransactionResult transfer(Long fromAccountId, Long toAccountId, double amount) {
        String txId = UUID.randomUUID().toString();
        log.info("[TxId: {}] Bắt đầu yêu cầu chuyển tiền từ tài khoản {} sang {} với số tiền: {}", 
                txId, fromAccountId, toAccountId, amount);

        // 1. Kiểm tra số tiền hợp lệ
        if (amount <= 0) {
            log.error("[TxId: {}] Giao dịch thất bại: Số tiền chuyển phải lớn hơn 0 (amount = {})", txId, amount);
            throw new IllegalArgumentException("Số tiền chuyển tiền phải lớn hơn 0");
        }

        // 2. Tìm kiếm tài khoản gửi
        Account from = accountRepository.findById(fromAccountId)
                .orElseThrow(() -> {
                    log.error("[TxId: {}] Giao dịch thất bại: Không tìm thấy tài khoản gửi ID {}", txId, fromAccountId);
                    return new AccountNotFoundException("Không tìm thấy tài khoản với ID: " + fromAccountId);
                });

        // 3. Tìm kiếm tài khoản nhận
        Account to = accountRepository.findById(toAccountId)
                .orElseThrow(() -> {
                    log.error("[TxId: {}] Giao dịch thất bại: Không tìm thấy tài khoản nhận ID {}", txId, toAccountId);
                    return new AccountNotFoundException("Không tìm thấy tài khoản với ID: " + toAccountId);
                });

        // 4. Kiểm tra số dư tài khoản gửi
        if (from.getBalance() < amount) {
            log.error("[TxId: {}] Giao dịch thất bại: Tài khoản {} không đủ số dư. Hiện tại: {}, Yêu cầu: {}", 
                    txId, fromAccountId, from.getBalance(), amount);
            throw new InsufficientBalanceException("Tài khoản gửi không đủ số dư để thực hiện giao dịch");
        }

        // 5. Khấu trừ và cộng tiền
        from.setBalance(from.getBalance() - amount);
        to.setBalance(to.getBalance() + amount);

        // 6. Lưu thay đổi vào DB
        accountRepository.save(from);
        accountRepository.save(to);

        log.info("[TxId: {}] Giao dịch thành công. Số dư mới: Tài khoản {} = {}, Tài khoản {} = {}", 
                txId, fromAccountId, from.getBalance(), toAccountId, to.getBalance());

        return TransactionResult.builder()
                .success(true)
                .transactionId(txId)
                .message("Chuyển tiền thành công")
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```

### 5. Mã nguồn kiểm thử JUnit 5 (`AccountServiceTest.java`)
```java
package com.example.bank.service;

import com.example.bank.entity.Account;
import com.example.bank.exception.InsufficientBalanceException;
import com.example.bank.repository.AccountRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AccountServiceTest {

    @Mock
    private AccountRepository accountRepository;

    @InjectMocks
    private AccountService accountService;

    private Account senderAccount;
    private Account receiverAccount;

    @BeforeEach
    void setUp() {
        // Khởi tạo các tài khoản giả lập cho test case
        senderAccount = new Account(1L, 5000.0); // Tài khoản gửi có 5,000 VND
        receiverAccount = new Account(2L, 1000.0); // Tài khoản nhận có 1,000 VND
    }

    @Test
    @DisplayName("Ném ngoại lệ InsufficientBalanceException và không lưu DB khi tài khoản gửi không đủ tiền")
    void transfer_ShouldThrowInsufficientBalanceException_WhenBalanceIsLessThanAmount() {
        // Arrange (Chuẩn bị)
        Long fromId = 1L;
        Long toId = 2L;
        double transferAmount = 6000.0; // Yêu cầu chuyển 6,000 VND (> 5,000 VND hiện có)

        // Mock hành vi tìm kiếm tài khoản từ Repository
        when(accountRepository.findById(fromId)).thenReturn(Optional.of(senderAccount));
        when(accountRepository.findById(toId)).thenReturn(Optional.of(receiverAccount));

        // Act & Assert (Thực hiện & Kiểm chứng)
        // 1. Xác thực xem ngoại lệ InsufficientBalanceException có được ném ra đúng như mong muốn không
        assertThrows(InsufficientBalanceException.class, () -> {
            accountService.transfer(fromId, toId, transferAmount);
        });

        // 2. Xác thực xem số dư của các tài khoản có được giữ nguyên (không bị trừ) không
        assertEquals(5000.0, senderAccount.getBalance());
        assertEquals(1000.0, receiverAccount.getBalance());

        // 3. RÀNG BUỘC QUAN TRỌNG: Đảm bảo phương thức save() KHÔNG BAO GIỜ được gọi
        // Điều này chứng minh transaction không thực thi lưu dữ liệu sai lệch xuống DB
        verify(accountRepository, never()).save(any(Account.class));
    }
}
```
```
