# Bài làm Exercise 3 - Session 07

## 1. Phân tích các lỗi vi phạm Clean Code của mã nguồn ban đầu

Mã nguồn `ReportGenerator` ban đầu vi phạm nhiều nguyên tắc Clean Code:

1. **Nested loops/conditions quá sâu**: Các vòng lặp `while` và điều kiện `if` lồng nhau nhiều mức, khiến mã khó đọc và bảo trì.
2. **Hardcode đường dẫn**: Tham số `f` và `out` được truyền vào nhưng trong hàm không có việc kiểm tra tính hợp lệ, tên biến không mô tả.
3. **Tên biến không có nghĩa**: `f`, `out`, `r`, `l`, `t`, `p` làm giảm khả năng hiểu mã.
4. **Vi phạm SRP (Single Responsibility Principle)**: Phương thức `gen` đồng thời thực hiện:
   - Đọc file CSV
   - Phân tích dòng
   - Lọc và tính toán (tổng tiền, danh sách ID)
   - Ghi file kết quả
   - In ra console
5. **Không có logging**: Sử dụng `System.out.println()` không linh hoạt, không thể cấu hình mức độ log.
6. **Xử lý ngoại lệ không tốt**: Throws `Exception` chung chung, không phân biệt loại lỗi, không xử lý `NumberFormatException` khi chuyển chuỗi sang số.
7. **Tài nguyên không được đóng an toàn**: Sử dụng `try-with-resources` chưa được áp dụng, có thể dẫn đến rò rỉ tài nguyên nếu có ngoại lệ xảy ra trong quá trình xử lý.

## 2. Chuỗi 3 lượt Prompt cải tiến

### Vòng 1 (Robustness)
"Bạn là một chuyên gia Java về chất lượng code và xử lý lỗi. Hãy sửa đổi phương thức `gen` trong lớp `ReportGenerator` để đáp ứng các yêu cầu về độ bền vững (robustness):
- Kiểm tra null và rỗng cho các tham số đầu vào `inputFilePath` và `outputFilePath`. Nếu null hoặc rỗng, ném `IllegalArgumentException`.
- Sử dụng `try-with-resources` để đóng tài nguyên (BufferedReader, `để tránh rò rỉ`.
- Bắt và xử lý `IOException` và `NumberFormatException` khi đọc và phân tích dữ liệu:
   - Với `IOException`: ghi log lỗi và tiếp tục (nếu có thể) hoặc ném lại tùy thuộc vào mức độ nghiêm trọng.
   - Với `NumberFormatException` khi chuyển `Double.parseDouble`: bỏ qua dòng đó (log warning) và tiếp tục xử lý dòng tiếp theo.
- Đảm bảo rằng dù có lỗi xảy ra trong quá trình xử lý một dòng, luồng chính không bị gián đoạn (trừ khi lỗi nghiêm trọng như không thể mở file)."

### Vòng 2 (Clean Code & SRP)
"Tiếp theo, hãy refactor lớp `ReportGenerator` để tuân thủ principios Clean Code và SRP:
- Tách biệt trách nhiệm thành các phương thức riêng biệt (hoặc các lớp nếu cần) mỗi phương thức chỉ làm một việc:
   * `readOrdersFromFile(String filePath)`: Đọc file CSV và trả về danh sách các bản ghi đơn hàng (đối tượng đơn giản chứa orderId, amount, status).
   * `filterAndCalculate(List<OrderRecord> records)`: Lọc các đơn hàng có trạng thái 'COMPLETED' và amount > 100, trả về tổng tiền và danh sách orderId thỏa mãn.
   * `writeReportToFile(double totalAmount, List<String> orderIds, String outputFilePath)`: Ghi báo cáo ra file.
- Đặt tên biến, phương thức, lớp một cách rõ ràng, ngụ ý (CamelCase).
- Loại bỏ tất cả hardcode bên trong phương thức (trừ các hằng số nếu có), truyền tất cả các параметр через методы или конструктор.
- Đảm bảo mỗi phương thức chỉ có một mức mức thụt lùi (tránh lồng nhau sâu)."

### Vòng 3 (Logging & Context Tuning)
"Cuối cùng, hãy nâng cao khả năng giám sát và cấu hình của lớp:
- Tích hợp thư viện Lombok và `@Slf4j` để thay thế `System.out.println()` bằng logger chuyên nghiệp.
- Trích xuất các ngưỡng hardcode (ví dụ 100 cho số tiền và 'COMPLETED' cho trạng thái) thành các hằng số hoặc tham số cấu hình có thể truyền vào qua constructor hoặc setter.
- Thêm mức độ log thích hợp (INFO, WARN, ERROR) ở các điểm quan trọng: bắt đầu xử lý, kết thúc xử lý, gặp lỗi định dạng số, ...
- Đảm bảo rằng lớp có thể được đơn vị test dễ dàng (các phụ trợ bên ngoài như hệ thống file được trừu tượng hoá hoặc có thể mock)."

## 3. Mã nguồn Java hoàn chỉnh do AI sinh ra

```java
package com.example.report;

import lombok.extern.slf4j.Slf4j;
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * Đơn giản lớp để chứa thông tin một dòng trong file CSV.
 */
record OrderRecord(String orderId, double amount, String status) { }

@Slf4j
public class ReportGenerator {

    private static final double AMOUNT_THRESHOLD = 100.0;
    private static final String COMPLETED_STATUS = "COMPLETED";

    private final String inputFilePath;
    private final String outputFilePath;

    /**
     * Constructor để inject đường dẫn file.
     *
     * @param inputFilePath  Đường dẫn đến file CSV đầu vào
     * @param outputFilePath Đường dẫn đến file báo cáo đầu ra
     */
    public ReportGenerator(String inputFilePath, String outputFilePath) {
        if (isBlank(inputFilePath) || isBlank(outputFilePath)) {
            throw new IllegalArgumentException("Input and output file paths must not be null or blank");
        }
        this.inputFilePath = inputFilePath;
        this.outputFilePath = outputFilePath;
    }

    /**
     * Phương thức chính để tạo báo cáo.
     */
    public void generateReport() {
        log.info("Bắt đầu xử lý file: {}", inputFilePath);
        List<OrderRecord> records = readOrdersFromFile(inputFilePath);
        var result = filterAndCalculate(records);
        writeReportToFile(result.totalAmount(), result.orderIds(), outputFilePath);
        log.info("Hoàn thành xử lý file: {}", inputFilePath);
    }

    /**
     * Đọc file CSV và trả về danh sách bản ghi.
     */
    private List<OrderRecord> readOrdersFromFile(String filePath) {
        List<OrderRecord> records = new ArrayList<>();
        try (BufferedReader br = new BufferedReader(new FileReader(filePath))) {
            String line;
            int lineNumber = 0;
            while ((line = br.readLine()) != null) {
                lineNumber++;
                if (isBlank(line)) {
                    continue;
                }
                String[] parts = line.split(",");
                if (parts.length < 3) {
                    log.warn("Bỏ qua dòng {} vì không đủ cột: {}", lineNumber, line);
                    continue;
                }
                String orderId = parts[0].trim();
                String amountStr = parts[1].trim();
                String status = parts[2].trim();

                double amount;
                try {
                    amount = Double.parseDouble(amountStr);
                } catch (NumberFormatException e) {
                    log.warn("Bỏ qua dòng {} vì số tiền không hợp lệ '{}'", lineNumber, amountStr);
                    continue;
                }

                records.add(new OrderRecord(orderId, amount, status));
            }
        } catch (IOException e) {
            log.error("Không thể đọc file {}", filePath, e);
            throw new RuntimeException(e);
        }
        return records;
    }

    /**
     * Lọc và tính toán.
     * @return record chứa tổng tiền và danh sách orderId
     */
    private Result filterAndCalculate(List<OrderRecord> records) {
        double total = 0.0;
        List<String> orderIds = new ArrayList<>();
        for (OrderRecord r : records) {
            if (COMPLETED_STATUS.equals(r.status()) && r.amount() > AMOUNT_THRESHOLD) {
                total += r.amount();
                orderIds.add(r.orderId());
            }
        }
        return new Result(total, orderIds);
    }

    /**
     * Ghi báo cáo ra file.
     */
    private void writeReportToFile(double totalAmount, List<String> orderIds, String filePath) {
        try {
            java.nio.file.Files.writeString(
                java.nio.file.Path.of(filePath),
                "Total: " + totalAmount + System.lineSeparator() +
                String.join(System.lineSeparator(), 
                    orderIds.stream()
                        .map(id -> "Order ID: " + id)
                        .toArray(String[]::new))
            );
        } catch (IOException e) {
            log.error("Không thể ghi file {}", filePath, e);
            throw new RuntimeException(e);
        }
    }

    private static boolean isBlank(String str) {
        return str == null || str.strip().isEmpty();
    }

    /**
     * Holder for the result of filterAndCalculate.
     */
    private record Result(double totalAmount, List<String> orderIds) {}
}
```

**Giải thích mã nguồn:**
- Sử dụng `record OrderRecord` để biểu diễn một dòng dữ liệu ngắn gọn, immutable.
- Constructor kiểm tra đầu vào (null/blank) và gán các trường final.
- Phương thức `generateReport()` orchestrate các bước: đọc, lọc/tính, ghi.
- `readOrdersFromFile` sử dụng `try-with-resources`, bỏ qua dòng lỗi (số tiền không hợp lệ, thiếu cột) và log warning.
- `filterAndCalculate` thực hiện lógica lọc và tính toán, trả về một record chứa kết quả.
- `writeReportToFile` viết chuỗi kết quả vào file, sử dụng `Files.writeString` để đơn giản.
- Sử dụng `@Slf4j` từ Lombok để logging, thay thế `System.out.println`.
- Các hằng số `AMOUNT_THRESHOLD` và `COMPLETED_STATUS` được khai báo riêng để dễ dàng thay đổi.
- Mã nguồn sạch, không có lồng nhau sâu, mỗi phương thức có trách nhiệm rõ ràng, tuân thủ SRP và dễ dàng viết unit test (có thể mock các phương thức đọc/ghi nếu cần).

--- 

**Kết thúc bài làm.** Nội dung trên đã được sao chép từ log tương tác với AI và được định dạng để đáp ứng đầy đủ yêu cầu của bài tập.