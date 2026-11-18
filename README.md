# JB-IOC-260316-MD05-Session07-Exercise05
---

## 📚 Course Information

|                 |                                                                    |
|-----------------|--------------------------------------------------------------------|
| **Institution** | RIKKEI EDUCATION                                                   |
| **Course**      | BE-105 - Devops                                                    |
| **Class**       | PTHB260316 - Java Backend                                          |
| **Session**     | Session07 - Tự động hóa quy trình CI/CD với Github Actions         |

---

## 👨‍💻 Author

|               |                                                                              |
|---------------|------------------------------------------------------------------------------|
| **Full Name** | Nguyễn Quế Phong                                                             |
|   **Email**   | [nguyenquephong13062003@gmail.com](mailto:nguyenquephong13062003@gmail.com)  |
|   **Phone**   | [0908130603](tel:+84908130603)                                               |

---

## Phân tích sự cố CI: `UnsupportedClassVersionError` trên notification-service

**Dự án:** QuickBite – `notification-service`
**Vai trò:** DevOps / Lead
**Công cụ:** GitHub Actions, Maven, `actions/setup-java`

---

### 1. Tóm tắt sự cố

Sau khi một lập trình viên push code lên `notification-service`, pipeline CI thất bại ở bước build với thông báo:

```text
UnsupportedClassVersionError: class file has been compiled by a more
recent version of the Java Runtime (class file version 65.0),
this version of the Java Runtime only recognizes class file versions up to 61.0
```

Thông tin bổ sung:

- Lập trình viên phát triển và build `notification-service` bằng **Java 21** trên máy cá nhân.
- Chưa rõ file `ci.yml` đang cấu hình phiên bản Java nào.

**Kết luận ngắn gọn:** log cho thấy có bytecode do JDK 21 tạo ra (65.0) đang được nạp bởi một JVM chỉ hỗ trợ tối đa Java 17 (61.0). Nguyên nhân gốc là **phiên bản Java dùng trên CI không đồng nhất với phiên bản Java dùng để biên dịch mã nguồn**.

---

### 2. Phân tích nguyên nhân

#### 2.1. Lỗi này xảy ra trong bối cảnh nào của pipeline CI?

Một pipeline CI Java thường có chuỗi bước: `checkout` → `setup-java` → `build/compile` → `test` → `package`.

`UnsupportedClassVersionError` là lỗi **runtime của JVM**, phát sinh khi JVM cố nạp (load) một file `.class` có định dạng mới hơn mức nó hỗ trợ. Trong CI, điều này xảy ra ở các bước có chạy JVM, ví dụ:

- Maven/Gradle chạy plugin hoặc nạp dependency đã được biên dịch bằng JDK cao hơn.
- Chạy test (`mvn test`) hoặc chạy ứng dụng với JVM của runner.

Tức là tại thời điểm lỗi, **có hai phiên bản Java cùng tham gia**:

| Khái niệm | Ý nghĩa | Trong sự cố này |
|---|---|---|
| Phiên bản biên dịch (compile-time) | JDK/`--release` tạo ra file `.class` | Java 21 (class version 65.0) |
| Phiên bản chạy (runtime) | JVM nạp và thực thi file `.class` | Java 17 (giới hạn 61.0) |

Quy tắc bắt buộc: **runtime phải ≥ phiên bản bytecode**. Ở đây 17 < 21 nên JVM từ chối nạp.

#### 2.2. Vai trò của số hiệu "class file version"

Mỗi file `.class` có phần header ghi `major_version`. JVM đọc số này trước tiên để biết file có nằm trong phạm vi nó hỗ trợ không. Nhờ vậy log lỗi tự cho biết cả hai phía:

- `class file version 65.0`: bytecode được tạo bởi JDK **21**.
- `only recognizes ... up to 61.0`: JVM hiện tại là Java **17**.

Cách suy ra nhanh: **major version = số phiên bản Java + 44** (áp dụng từ Java 5 trở đi).

| Java | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Major | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | **61** | 62 | 63 | 64 | **65** | 66 | 67 | 68 | 69 |

Tra cứu từ nguồn chính thức, không cần học thuộc:

- The Java Virtual Machine Specification, mục 4.1 *The ClassFile Structure* (bảng `major_version`): <https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.1>
- Wikipedia, *Java class file*, mục Versions: <https://en.wikipedia.org/wiki/Java_class_file>
- Kiểm tra trực tiếp một file `.class`: `javap -v Foo.class | grep "major version"`

#### 2.3. Vì sao máy cá nhân không lỗi nhưng GitHub Actions thì lỗi?

| | Máy lập trình viên | GitHub Actions runner |
|---|---|---|
| Bản chất môi trường | Môi trường lâu dài, do dev tự cài đặt | VM sạch, được tạo mới cho mỗi lần chạy |
| Nguồn quyết định JDK | Dev cài sẵn (SDKMAN, IDE, installer...) | File `ci.yml` (step `actions/setup-java`), hoặc JDK mặc định của runner image nếu không khai báo |
| Trạng thái | Cùng JDK 21 cho cả compile và run | Có thể là JDK 17 hoặc bản mặc định khác |
| Kết quả | Hoạt động | Compile-time 21 và runtime 17 lệch nhau, gây lỗi |

**Sự khác biệt cốt lõi:** CI **không kế thừa** bất kỳ thứ gì từ máy dev. Phiên bản Java trên CI chỉ đến từ những gì được khai báo trong repository. Nếu `ci.yml` khai báo (hoặc ngầm dùng) Java 17 trong khi code được phát triển với Java 21, sự lệch pha sẽ chỉ lộ ra khi chạy trên CI.

#### 2.4. Các hướng chẩn đoán để xác nhận root cause

Không sửa theo cảm tính. Cần xác minh bằng bằng chứng:

1. **Xem phiên bản Java thực tế của job.** Thêm step `java -version` và `mvn -version` (hoặc xem log của step `Set up JDK`).
2. **Rà soát `ci.yml`:** `java-version` trong `actions/setup-java` đang là gì? Có khai báo hay không?
3. **Rà soát `pom.xml`:** `maven.compiler.release`, `java.version` hoặc `maven-compiler-plugin` đang đặt mức nào?
4. **Xác định nguồn của class 65.0:**
   - Mã nguồn của service được biên dịch bằng JDK 21.
   - Một thư viện nội bộ (shared library) hoặc dependency được publish từ JDK 21.
   - Thư mục `target/` hay file `.class` bị commit nhầm vào repo.
5. **Đối chiếu** với Dockerfile và môi trường deploy để chắc chắn không còn điểm lệch khác.

> Lưu ý: nếu `pom.xml` đặt `release=21` mà CI chạy JDK 17 thì thông thường `javac` sẽ báo lỗi `release version 21 not supported` ngay ở bước compile. Còn `UnsupportedClassVersionError` xuất hiện khi JVM 17 **nạp** một class đã được biên dịch ở mức 21 (từ dependency, plugin, hoặc artifact build sẵn). Dù khác dạng lỗi, gốc rễ vẫn là một: **sai lệch phiên bản Java giữa nơi tạo bytecode và nơi chạy**.

---

### 3. Đề xuất giải pháp

#### 3.1. Vị trí cần rà soát và sửa

Trong `.github/workflows/ci.yml`, tại step **`actions/setup-java`** của job build, cụ thể là tham số **`java-version`** (cùng `distribution`).

**Trước khi sửa (cấu hình gây lỗi):**

```yaml
- name: Set up JDK
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '17'      # Lệch với Java 21 dùng để phát triển
```

**Sau khi sửa:**

```yaml
name: CI - notification-service

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'     # Khớp với JDK dùng khi phát triển
          cache: maven

      - name: Verify Java version
        run: |
          java -version
          mvn -version

      - name: Build and test
        run: mvn -B clean verify
```

#### 3.2. Vì sao thay đổi này khắc phục được lỗi?

- `java-version: '21'` khiến runner cài JDK 21 trước khi build, nên JVM chạy trong CI có mức hỗ trợ class file tối đa là 65.0.
- Bytecode 65.0 giờ nằm trong phạm vi JVM hỗ trợ, nên `UnsupportedClassVersionError` không còn xuất hiện.
- Việc chỉ định rõ `distribution: temurin` giúp bản JDK cố định, không phụ thuộc mặc định của runner image.
- Step `Verify Java version` ghi lại phiên bản thực tế vào log, giúp chẩn đoán nhanh nếu lỗi tái diễn.

> Hướng thay thế: nếu đội **chưa** sẵn sàng chạy Java 21 (ví dụ môi trường production vẫn là Java 17), thì không nâng CI mà phải hạ mức biên dịch xuống 17 (`<maven.compiler.release>17</maven.compiler.release>`) cho đồng nhất. Điểm mấu chốt là **chọn một phiên bản và áp dụng đồng bộ**, không phải mặc định luôn nâng CI.

---

### 4. Nguyên tắc phòng ngừa

1. **Một nguồn sự thật duy nhất (single source of truth).** Khai báo phiên bản Java ở một nơi rồi để mọi thứ đọc từ đó. Ví dụ dùng file `.java-version` ở thư mục gốc:

   ```text
   21
   ```

   ```yaml
   - uses: actions/setup-java@v4
     with:
       distribution: temurin
       java-version-file: .java-version
       cache: maven
   ```

2. **Dùng `release` thay vì chỉ `source`/`target` trong `pom.xml`:**

   ```xml
   <properties>
     <maven.compiler.release>21</maven.compiler.release>
   </properties>
   ```

   `--release` giúp compiler chặn việc dùng API vượt quá phiên bản đã khai báo.

3. **Đồng bộ mọi môi trường** về cùng major version: máy dev, CI, Dockerfile và môi trường deploy.

   ```dockerfile
   FROM eclipse-temurin:21-jre
   ```

4. **Luôn pin rõ `distribution` và `java-version`.** Không dựa vào JDK mặc định của runner vì image `ubuntu-latest` có thể thay đổi theo thời gian.

5. **Giữ step in phiên bản** (`java -version`, `mvn -version`) ở đầu job để dễ truy vết khi có sự cố.

6. **Nâng phiên bản Java trong một PR duy nhất.** Khi nâng Java, sửa đồng thời `pom.xml`, `ci.yml`, `.java-version` và Dockerfile, để reviewer thấy toàn bộ thay đổi liên quan. Có thể dùng Dependabot/Renovate để theo dõi cập nhật.

7. **Không commit `target/` hay file `.class`.** Kiểm tra `.gitignore` để artifact build luôn được tạo mới trong CI.

8. **Dùng `matrix` khi cần hỗ trợ nhiều phiên bản** (ví dụ thư viện dùng chung):

   ```yaml
   strategy:
     matrix:
       java: ['17', '21']
   steps:
     - uses: actions/setup-java@v4
       with:
         distribution: temurin
         java-version: ${{ matrix.java }}
   ```

---

### 5. Kết luận

- **Hiện tượng:** `UnsupportedClassVersionError` với class file version 65.0 và giới hạn 61.0.
- **Nguyên nhân gốc:** bytecode do **Java 21** tạo ra nhưng bước build trên CI chạy **Java 17**; hai phiên bản không đồng nhất vì CI không kế thừa môi trường local.
- **Cách khắc phục:** chỉnh `java-version` trong `actions/setup-java` của `ci.yml` cho khớp phiên bản phát triển (Java 21), kèm step kiểm tra phiên bản.
- **Phòng ngừa:** khai báo phiên bản Java ở một nguồn duy nhất, đồng bộ local, CI, Docker và deploy, và luôn pin phiên bản trong cấu hình.

---

### 6. Tài liệu tham khảo

- Oracle, *The Java Virtual Machine Specification*, mục 4.1: <https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.1>
- Wikipedia, *Java class file*: <https://en.wikipedia.org/wiki/Java_class_file>
- GitHub, `actions/setup-java`: <https://github.com/actions/setup-java>
- Apache Maven Compiler Plugin, tham số `release`: <https://maven.apache.org/plugins/maven-compiler-plugin/>
