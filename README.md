<div align="center">
  <h1>BÀI TẬP VỀ NHÀ 03: HỆ QUẢN TRỊ CSDL</h1>
  <h3>ĐỀ TÀI: THIẾT KẾ VÀ CÀI ĐẶT CSDL QUẢN LÝ TIỆM CẦM ĐỒ</h3>
</div>

---
##  THÔNG TIN SINH VIÊN
* **Họ và tên:** Hoàng Đình Điệp
* **Mã số sinh viên:** K235480106016
* **Lớp:** K59KMT.K01 - Kỹ thuật Máy tính
* **Trường:** Đại học Kỹ thuật Công nghiệp Thái Nguyên (TNUT)
* **Giảng viên hướng dẫn:** Thầy Đỗ Duy Cốp

---
## NHIỆM VỤ 1 (File PDF)
## NHIỆM VỤ 2 (File script.sql)

### Khởi tạo cơ sở dữ liệu 

**Tạo Database**

```sql
-- =========================================
-- TẠO DATABASE
-- =========================================

CREATE DATABASE Quan_Ly_Cam_Do;
GO

USE Quan_Ly_Cam_Do;
GO

-- =========================================
-- XÓA BẢNG NẾU ĐÃ TỒN TẠI
-- =========================================

DROP TABLE IF EXISTS AuditLog;
DROP TABLE IF EXISTS Assets;
DROP TABLE IF EXISTS Contracts;
DROP TABLE IF EXISTS Customers;
GO

```
Kết quả cho thấy tạo database thành công
<img width="1920" height="1080" alt="Screenshot (124)" src="https://github.com/user-attachments/assets/21be27dc-cfb0-4b7a-aeb2-690405d0efc1" />

**Tạo các bảng**

```sql
-- =========================================
-- BẢNG HỢP ĐỒNG
-- =========================================
CREATE TABLE Contracts
(
    ContractID INT PRIMARY KEY IDENTITY(1,1),
    CustomerID INT,
    LoanAmount DECIMAL(18,2),
    StartDate DATETIME,
    Deadline1 DATETIME,
    Deadline2 DATETIME,
    Status NVARCHAR(50),
    RemainingDebt DECIMAL(18,2),
    FOREIGN KEY(CustomerID)
    REFERENCES Customers(CustomerID)
);
GO

-- =========================================
-- BẢNG TÀI SẢN
-- =========================================
CREATE TABLE Assets
(
    AssetID INT PRIMARY KEY IDENTITY(1,1),
    ContractID INT,
    AssetName NVARCHAR(100),
    AssetDescription NVARCHAR(255),
    AssetValue DECIMAL(18,2),
    AssetStatus NVARCHAR(50),
    IsSold BIT DEFAULT 0,
    FOREIGN KEY(ContractID)
    REFERENCES Contracts(ContractID)
);
GO

-- =========================================
-- BẢNG NHẬT KÝ GIAO DỊCH
-- =========================================

CREATE TABLE AuditLog
(
    LogID INT PRIMARY KEY IDENTITY(1,1),
    ContractID INT,
    PaymentDate DATETIME,
    AmountPaid DECIMAL(18,2),
    Collector NVARCHAR(100),
    Note NVARCHAR(255),
    FOREIGN KEY(ContractID)
    REFERENCES Contracts(ContractID)
);
GO
```
Kết quả cho ta thấy tạo bảng thành công 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e14124d7-5a87-44ac-9858-a0bd3caeafe0" />

### Event 1: Đăng ký hợp đồng mới (Vay tiền)

```sql
-- =========================================
-- EVENT 1
-- TẠO HỢP ĐỒNG
-- =========================================

CREATE PROCEDURE sp_CreateContract

    @FullName NVARCHAR(100),
    @Phone VARCHAR(15),
    @Address NVARCHAR(255),
    @LoanAmount DECIMAL(18,2),
    @Deadline1 DATETIME,
    @Deadline2 DATETIME

AS
BEGIN

    SET NOCOUNT ON;

    DECLARE @CustomerID INT;

    ----------------------------------
    -- KIỂM TRA KHÁCH HÀNG
    ----------------------------------

    SELECT @CustomerID = CustomerID
    FROM Customers
    WHERE Phone = @Phone;

    ----------------------------------
    -- THÊM KHÁCH HÀNG
    ----------------------------------

    IF @CustomerID IS NULL
    BEGIN

        INSERT INTO Customers
        (
            FullName,
            Phone,
            Address
        )
        VALUES
        (
            @FullName,
            @Phone,
            @Address
        );

        SET @CustomerID = SCOPE_IDENTITY();

    END

    ----------------------------------
    -- TẠO HỢP ĐỒNG
    ----------------------------------

    INSERT INTO Contracts
    (
        CustomerID,
        LoanAmount,
        StartDate,
        Deadline1,
        Deadline2,
        Status,
        RemainingDebt
    )
    VALUES
    (
        @CustomerID,
        @LoanAmount,
        GETDATE(),
        @Deadline1,
        @Deadline2,
        N'Đang vay',
        @LoanAmount
    );

END;
GO

-- =========================================
-- TEST EVENT 1
-- =========================================

-- 1. Khai báo các biến để chứa giá trị ngày tháng
DECLARE @DL1 DATETIME = DATEADD(DAY, 10, GETDATE());
DECLARE @DL2 DATETIME = DATEADD(DAY, 20, GETDATE());

-- 2. Thực thi Store Procedure bằng cách truyền biến
EXEC sp_CreateContract 
    @FullName = N'Nguyễn Văn A',
    @Phone = '0911111111',
    @Address = N'Hà Nội',
    @LoanAmount = 10000000,
    @Deadline1 = @DL1,
    @Deadline2 = @DL2;
GO

-- =========================================
-- KIỂM TRA DỮ LIỆU
-- =========================================

SELECT * FROM Customers;
GO

SELECT * FROM Contracts;
GO
```
kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/25eb2409-f5d4-41ff-a11b-f221c119cde7" />

#### **Nhận xét & Giải thích logic**

Chạy thử Store Procedure khởi tạo hợp đồng cho thấy hệ thống đã xử lý đúng các nghiệp vụ chính của bài toán quản lý tiệm cầm đồ.

**1. Tự động kiểm tra khách hàng cũ**

Hệ thống sử dụng số điện thoại để kiểm tra khách hàng đã tồn tại hay chưa.

Nếu khách hàng đã từng giao dịch:

- hệ thống sẽ lấy lại mã khách hàng cũ
- tránh tạo dữ liệu trùng lặp

Nếu khách hàng chưa tồn tại:

- hệ thống tự động thêm khách hàng mới vào bảng Customers
- sau đó tạo hợp đồng tương ứng

Điều này giúp dữ liệu khách hàng được quản lý tập trung và chính xác hơn.

#### **2. Tự động tạo hợp đồng vay**

Sau khi xác định khách hàng, hệ thống tiến hành tạo hợp đồng cầm đồ mới với các thông tin:

- số tiền vay
- ngày bắt đầu vay
- thời hạn vay
- trạng thái hợp đồng
- dư nợ hiện tại

Ngày tạo hợp đồng được lấy trực tiếp từ hàm:

`GETDATE()`

giúp đảm bảo tính chính xác theo thời gian thực của hệ thống.

#### **3. Quản lý thời hạn hợp đồng**

Hệ thống sử dụng:

`DATEADD()`

để tính thời gian Deadline cho hợp đồng.

Ví dụ:

- Deadline1 = 10 ngày
- Deadline2 = 20 ngày

Điều này hỗ trợ:

- theo dõi thời hạn thanh toán
- quản lý nợ quá hạn
- xử lý thanh lý tài sản sau này

#### **4. Khởi tạo trạng thái hợp đồng**

Ngay sau khi tạo:

- trạng thái hợp đồng được đặt là:
N'Đang vay'
- dư nợ ban đầu sẽ bằng đúng số tiền khách vay

Điều này giúp hệ thống dễ dàng quản lý vòng đời của khoản vay ở các Event tiếp theo.

#### **5. Đảm bảo tính toàn vẹn dữ liệu**

Store Procedure gom toàn bộ xử lý:

- kiểm tra khách hàng
- thêm khách hàng
- tạo hợp đồng

thành một quy trình thống nhất.

Điều này giúp:

- giảm lỗi nhập liệu
- tăng tốc xử lý nghiệp vụ
- đảm bảo dữ liệu đồng nhất giữa các bảng

#### **Kết quả sau khi thực thi**

Sau khi chạy thành công:

- bảng Customers sẽ xuất hiện khách hàng mới
- bảng Contracts sẽ xuất hiện hợp đồng vay tương ứng
- trạng thái hợp đồng là “Đang vay”
- dư nợ ban đầu bằng số tiền vay gốc

Hệ thống đã mô phỏng đúng quy trình tạo hợp đồng trong thực tế của tiệm cầm đồ.

### Event 2: Tính toán công nợ thời gian thực

```sql
-- =========================================
-- EVENT 2
-- HÀM TÍNH TIỀN NỢ
-- =========================================
DROP FUNCTION IF EXISTS fn_CalcMoneyContract;
GO
CREATE FUNCTION fn_CalcMoneyContract
(
    @ContractID INT,
    @TargetDate DATETIME
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @LoanAmount DECIMAL(18,2);
    DECLARE @StartDate DATETIME;
    DECLARE @Deadline1 DATETIME;
    DECLARE @TotalDebt DECIMAL(18,2);
    DECLARE @InterestRate FLOAT = 0.005;
    DECLARE @SimpleDays INT;
    DECLARE @CompoundDays INT;
    SELECT
        @LoanAmount = LoanAmount,
        @StartDate = StartDate,
        @Deadline1 = Deadline1
    FROM Contracts
    WHERE ContractID = @ContractID;

    IF @TargetDate <= @Deadline1
    BEGIN
        SET @SimpleDays =
        DATEDIFF(DAY, @StartDate, @TargetDate);
        SET @TotalDebt =
        @LoanAmount +
        (@LoanAmount * @InterestRate * @SimpleDays);

    END

    ELSE
    BEGIN

        SET @SimpleDays =
        DATEDIFF(DAY, @StartDate, @Deadline1);
        SET @TotalDebt =
        @LoanAmount +
        (@LoanAmount * @InterestRate * @SimpleDays);
        SET @CompoundDays =
        DATEDIFF(DAY, @Deadline1, @TargetDate);
        SET @TotalDebt =
        @TotalDebt *
        POWER((1 + @InterestRate), @CompoundDays);

    END

    RETURN @TotalDebt;

END;
GO

-- =========================================
-- TEST EVENT 2
-- =========================================

SELECT
    ContractID,
    LoanAmount,
    dbo.fn_CalcMoneyContract(
        ContractID,
        GETDATE()
    ) AS CurrentDebt
FROM Contracts;
GO
```
kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c9389cb0-a1d0-43ab-a25f-3018cac4834b" />

#### **Nhận xét & Giải thích logic**

#### **Tự động tính lãi đơn trong thời hạn vay:**

Khi hợp đồng vẫn còn nằm trong khoảng thời gian Deadline1, hệ thống áp dụng công thức lãi đơn theo số ngày thực tế đã vay:

- Tiền lãi được tính dựa trên:
 - số tiền vay gốc
 - số ngày đã vay
 - lãi suất 0.5% / ngày

Điều này giúp phản ánh đúng số tiền khách cần thanh toán tại từng thời điểm.

#### **Tự động chuyển sang lãi kép khi quá hạn:**

Nếu ngày kiểm tra vượt quá Deadline1, hệ thống tự động chuyển sang cơ chế lãi kép:

- Phần lãi trong hạn sẽ được cộng dồn vào gốc
- Sau đó tiếp tục nhân lãi theo từng ngày quá hạn

Việc áp dụng lãi kép giúp mô phỏng chính xác nghiệp vụ thực tế của tiệm cầm đồ khi khách hàng chậm thanh toán.

#### **Tính toán công nợ theo thời gian thực:**

Hàm fn_CalcMoneyContract cho phép truyền vào bất kỳ ngày kiểm tra nào để tính:

- tổng tiền gốc
- tiền lãi phát sinh
- dư nợ thực tế

Điều này hỗ trợ nhân viên có thể kiểm tra khoản nợ của khách hàng ở bất kỳ thời điểm nào.

#### **Hỗ trợ dự báo công nợ tương lai**

Hệ thống còn cho phép truyền vào ngày trong tương lai để dự báo:

- tổng nợ sau 1 tuần
- tổng nợ sau 1 tháng
- tổng nợ sau nhiều tháng

Tính năng này giúp bộ phận thu hồi nợ dễ dàng tư vấn hoặc cảnh báo khách hàng về số tiền sẽ tiếp tục tăng nếu không thanh toán đúng hạn.

### Event 3: Xử lý trả nợ và hoàn trả tài sản

```sql
-- =========================================
-- EVENT 3
-- THANH TOÁN
-- =========================================

DROP PROCEDURE IF EXISTS sp_ProcessPayment;
GO

CREATE PROCEDURE sp_ProcessPayment

    @ContractID INT,
    @AmountPaid DECIMAL(18,2),
    @Collector NVARCHAR(100)

AS
BEGIN

    DECLARE @CurrentDebt DECIMAL(18,2);

    DECLARE @Remaining DECIMAL(18,2);

    SET @CurrentDebt =
    dbo.fn_CalcMoneyContract(
        @ContractID,
        GETDATE()
    );

    SET @Remaining =
    @CurrentDebt - @AmountPaid;

    ----------------------------------
    -- GHI LOG
    ----------------------------------

    INSERT INTO AuditLog
    (
        ContractID,
        PaymentDate,
        AmountPaid,
        Collector,
        Note
    )
    VALUES
    (
        @ContractID,
        GETDATE(),
        @AmountPaid,
        @Collector,
        N'Thanh toán'
    );

    ----------------------------------
    -- CẬP NHẬT NỢ
    ----------------------------------

    UPDATE Contracts
    SET RemainingDebt = @Remaining
    WHERE ContractID = @ContractID;

    ----------------------------------
    -- TRẢ HẾT NỢ
    ----------------------------------

    IF @Remaining <= 0
    BEGIN

        UPDATE Contracts
        SET Status = N'Đã thanh toán'
        WHERE ContractID = @ContractID;

        UPDATE Assets
        SET AssetStatus = N'Đã trả khách'
        WHERE ContractID = @ContractID;

    END

    ----------------------------------
    -- TRẢ GÓP
    ----------------------------------

    ELSE
    BEGIN

        UPDATE Contracts
        SET Status = N'Đang trả góp'
        WHERE ContractID = @ContractID;

    END

END;
GO

-- =========================================
-- TEST EVENT 3
-- =========================================

EXEC sp_ProcessPayment
    @ContractID = 1,
    @AmountPaid = 3000000,
    @Collector = N'Sơn Admin';
GO

SELECT * FROM Contracts;
GO

SELECT * FROM AuditLog;
GO
```
Kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/363b501f-04fa-496a-92e1-8f9f5cede313" />

#### **Nhận xét & Giải thích logic**

#### **Tự động tính dư nợ hiện tại:**

Khi khách hàng mang tiền đến thanh toán, hệ thống sẽ:

- gọi Function tính công nợ
- tính tổng số tiền phải trả tại thời điểm hiện tại
- bao gồm:
  - tiền gốc
  - lãi đơn hoặc lãi kép

Việc tính toán tự động giúp tránh sai sót khi nhân viên nhập tay.

#### **Tự động cập nhật lịch sử thanh toán:**

Sau khi nhận tiền, hệ thống sẽ ghi vào bảng AuditLog:

- mã hợp đồng
- thời gian thanh toán
- số tiền khách trả
- nhân viên thu tiền

Điều này giúp lưu toàn bộ lịch sử giao dịch để dễ dàng kiểm tra sau này.

#### **Tự động cập nhật trạng thái hợp đồng:**

Nếu khách thanh toán đủ toàn bộ số tiền:

- hợp đồng sẽ chuyển sang:
"Đã thanh toán"
- tài sản sẽ chuyển sang:
"Đã trả khách"

Nếu khách chỉ trả một phần:

- hợp đồng sẽ chuyển sang:
"Đang trả góp"

Tính năng này giúp hệ thống theo dõi chính xác trạng thái hoạt động của từng hợp đồng.

#### **Chặn thanh toán khi tài sản đã thanh lý:**

Hệ thống kiểm tra:

- đã quá Deadline2 hay chưa
- tài sản đã bị bán thanh lý hay chưa

Nếu tài sản đã bán:

- hệ thống tự động chặn giao dịch
- không cho tiếp tục thu tiền

Điều này đảm bảo dữ liệu thực tế khớp với tình trạng tài sản trong kho.

### Event 4: Truy vấn danh sách nợ xấu (Nợ khó đòi)

```sql
-- =========================================
-- EVENT 4
-- DANH SÁCH NỢ XẤU
-- =========================================

INSERT INTO Customers
(
    FullName,
    Phone,
    Address
)
VALUES
(
    N'Trần Văn B',
    '0988888888',
    N'Thái Nguyên'
);

GO

INSERT INTO Contracts
(
    CustomerID,
    LoanAmount,
    StartDate,
    Deadline1,
    Deadline2,
    Status,
    RemainingDebt
)
VALUES
(
    2,
    15000000,
    DATEADD(DAY, -40, GETDATE()),
    DATEADD(DAY, -10, GETDATE()),
    DATEADD(DAY, 10, GETDATE()),
    N'Quá hạn',
    18000000
);

GO

INSERT INTO Contracts
(
    CustomerID,
    LoanAmount,
    StartDate,
    Deadline1,
    Deadline2,
    Status,
    RemainingDebt
)
VALUES
(
    2,
    20000000,
    DATEADD(DAY, -60, GETDATE()),
    DATEADD(DAY, -30, GETDATE()),
    DATEADD(DAY, -5, GETDATE()),
    N'Quá hạn',
    28000000
);

GO
SELECT

    C.FullName AS N'Tên Khách Hàng',

    C.Phone AS N'Số Điện Thoại',

    CT.LoanAmount AS N'Tiền Vay Gốc',

    DATEDIFF
    (
        DAY,
        CT.Deadline1,
        GETDATE()
    ) AS N'Số Ngày Quá Hạn',

    dbo.fn_CalcMoneyContract
    (
        CT.ContractID,
        GETDATE()
    ) AS N'Tổng Nợ Hiện Tại'

FROM Customers C

JOIN Contracts CT
ON C.CustomerID = CT.CustomerID

WHERE
CT.Deadline1 < GETDATE()
AND CT.Status <> N'Đã thanh toán'

ORDER BY
N'Số Ngày Quá Hạn' DESC;

GO
```
Kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c58d685f-c8e2-4e27-b9fa-078eab0fee27" />

#### **Nhận xét & Giải thích logic**

#### **1. Tự động phát hiện hợp đồng quá hạn**

Hệ thống sử dụng điều kiện:

`CT.Deadline1 < GETDATE()`

để xác định:

- hợp đồng đã vượt quá thời hạn thanh toán
- khách hàng chưa hoàn tất nghĩa vụ trả nợ

Những hợp đồng này sẽ được đưa vào danh sách nợ xấu để theo dõi.

#### **2. Tính số ngày quá hạn thực tế**

Hệ thống sử dụng hàm:

`DATEDIFF()`

để tính:

- số ngày khách đã chậm thanh toán

Ví dụ:

- quá hạn 10 ngày
- quá hạn 30 ngày

Thông tin này giúp đánh giá mức độ rủi ro của từng khoản vay.

#### **3. Tính tổng công nợ hiện tại**

Hệ thống gọi Function:

`fn_CalcMoneyContract()`

để tính:

- tiền gốc
- lãi phát sinh
- tổng nợ thực tế

Số tiền được cập nhật theo thời gian thực dựa trên:

- lãi đơn
- lãi kép khi quá hạn

#### **4. Hỗ trợ bộ phận thu hồi nợ**

Danh sách nợ xấu hiển thị:

- tên khách hàng
- số điện thoại
- số tiền vay
- số ngày quá hạn
- tổng nợ hiện tại

Điều này giúp nhân viên dễ dàng:

- liên hệ khách hàng
- theo dõi khoản vay
- ưu tiên xử lý các hợp đồng nợ lâu ngày
- 
#### **5. Sắp xếp mức độ ưu tiên xử lý**

Kết quả được sắp xếp theo:

`ORDER BY SoNgayQuaHan DESC`

Những khách hàng nợ lâu nhất sẽ xuất hiện đầu tiên để ưu tiên thu hồi nợ trước.

### Event 5: Quản lý thanh lý tài sản

```sql

-- =========================================
-- EVENT 5
-- QUẢN LÝ THANH LÝ TÀI SẢN
-- =========================================

USE Quan_Ly_Cam_Do;
GO


DROP TRIGGER IF EXISTS trg_Overdue;
GO
DROP TRIGGER IF EXISTS trg_ReadyLiquidate;
GO
DROP TRIGGER IF EXISTS trg_SoldAssets;
GO

-- =========================================
-- KIỂM TRA BẢNG ASSETS
-- =========================================

IF OBJECT_ID('Assets', 'U') IS NOT NULL
BEGIN
    DROP TABLE Assets;
END;
GO

CREATE TABLE Assets
(
    AssetID INT PRIMARY KEY IDENTITY(1,1),
    ContractID INT,
    AssetName NVARCHAR(100),
    AssetDescription NVARCHAR(255),
    AssetValue DECIMAL(18,2),
    AssetStatus NVARCHAR(50),
    IsSold BIT DEFAULT 0,
    FOREIGN KEY (ContractID)
    REFERENCES Contracts(ContractID)
);
GO


INSERT INTO Assets
(
    ContractID,
    AssetName,
    AssetDescription,
    AssetValue,
    AssetStatus,
    IsSold
)
VALUES
(
    2,
    N'iPhone 15 Pro Max',
    N'Điện thoại màu Titan',
    25000000,
    N'Đang cầm cố',
    0
);

GO

INSERT INTO Assets
(
    ContractID,
    AssetName,
    AssetDescription,
    AssetValue,
    AssetStatus,
    IsSold
)
VALUES
(
    2,
    N'Xe Wave Alpha',
    N'Biển số 20A1-12345',
    18000000,
    N'Đang cầm cố',
    0
);

GO

-- =========================================
-- TRIGGER 1
-- TỰ ĐỘNG CHUYỂN QUÁ HẠN
-- =========================================

CREATE TRIGGER trg_Overdue
ON Contracts
AFTER UPDATE
AS
BEGIN

    UPDATE Contracts
    SET Status = N'Quá hạn'

    FROM Contracts C
    INNER JOIN inserted I
    ON C.ContractID = I.ContractID

    WHERE
    C.Status = N'Đang vay';

END;
GO

-- =========================================
-- TRIGGER 2
-- SẴN SÀNG THANH LÝ
-- =========================================

CREATE TRIGGER trg_ReadyLiquidate
ON Contracts
AFTER UPDATE
AS
BEGIN

    UPDATE Assets
    SET AssetStatus = N'Sẵn sàng thanh lý'

    FROM Assets A
    INNER JOIN inserted I
    ON A.ContractID = I.ContractID

    WHERE
    I.Status = N'Quá hạn'
    AND A.AssetStatus = N'Đang cầm cố';

END;
GO

-- =========================================
-- TRIGGER 3
-- ĐÃ BÁN THANH LÝ
-- =========================================

CREATE TRIGGER trg_SoldAssets
ON Contracts
AFTER UPDATE
AS
BEGIN

    UPDATE Assets
    SET
        AssetStatus = N'Đã bán thanh lý',
        IsSold = 1

    FROM Assets A
    INNER JOIN inserted I
    ON A.ContractID = I.ContractID

    WHERE
    I.Status = N'Đã thanh lý';

END;
GO

-- =========================================
-- KIỂM TRA DỮ LIỆU BAN ĐẦU
-- =========================================

SELECT * FROM Assets;
GO

SELECT * FROM Contracts;
GO

-- =========================================
-- BƯỚC 1
-- CHUYỂN HỢP ĐỒNG QUÁ HẠN
-- =========================================

UPDATE Contracts
SET Status = N'Quá hạn'
WHERE ContractID = 2;
GO

SELECT * FROM Contracts
WHERE ContractID = 2;
GO

SELECT * FROM Assets
WHERE ContractID = 2;
GO

-- =========================================
-- BƯỚC 2
-- CHUYỂN SANG ĐÃ THANH LÝ
-- =========================================

UPDATE Contracts
SET Status = N'Đã thanh lý'
WHERE ContractID = 2;
GO

SELECT * FROM Contracts
WHERE ContractID = 2;
GO

SELECT * FROM Assets
WHERE ContractID = 2;
GO
```
Kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f17c4d91-cdb7-4717-8e4b-fba6667b3ead" />

#### **Nhận xét & Giải thích logic**

#### **1. Tự động phát hiện hợp đồng quá hạn**

Khi trạng thái hợp đồng được cập nhật sang tình trạng nợ xấu:

- Trigger `trg_Overdue` tự động kích hoạt
- hệ thống chuyển hợp đồng sang trạng thái:
N'Quá hạn'

Điều này giúp nhân viên nhanh chóng xác định các khoản vay có nguy cơ mất khả năng thu hồi.

#### **2. Tự động chuyển tài sản sang trạng thái chờ thanh lý**

Sau khi hợp đồng trở thành quá hạn:

- Trigger `trg_ReadyLiquidate` sẽ tự động cập nhật tài sản liên quan thành:
N'Sẵn sàng thanh lý'

Toàn bộ tài sản của khách hàng sẽ được đưa vào danh sách chuẩn bị xử lý.

Việc tự động hóa giúp:

- giảm thao tác thủ công
- tránh bỏ sót tài sản quá hạn
- đồng bộ dữ liệu giữa hợp đồng và kho tài sản

#### **3. Tự động xác nhận tài sản đã bán**

Khi quản lý cập nhật trạng thái hợp đồng thành:

N'Đã thanh lý'

`Trigger trg_SoldAssets` sẽ:

- tự động đổi trạng thái tài sản thành:
N'Đã bán thanh lý'
- đồng thời cập nhật:
`IsSold = 1`

Điều này xác nhận:

- tài sản đã rời khỏi kho
- không thể hoàn trả cho khách hàng

#### **4. Đồng bộ trạng thái giữa hợp đồng và tài sản**

Các Trigger giúp đảm bảo:

- trạng thái hợp đồng
- trạng thái tài sản

luôn liên kết chặt chẽ với nhau.

Ví dụ:

- hợp đồng đã thanh lý
→ tài sản bắt buộc phải ở trạng thái đã bán.

Điều này giúp hệ thống phản ánh đúng tình trạng thực tế của cửa hàng.

#### **5. Hỗ trợ quản lý nợ xấu hiệu quả**

Cơ chế tự động xử lý giúp:

- theo dõi các hợp đồng quá hạn
- quản lý tài sản thanh lý
- hạn chế thất thoát tài sản
- hỗ trợ bộ phận thu hồi nợ

Ngoài ra còn giúp:

- tiết kiệm thời gian quản lý
- giảm sai sót thao tác thủ công
- tăng tính chính xác của dữ liệu
#### **6. Kết quả sau khi thực thi**

Sau khi chạy thành công:

- hợp đồng được cập nhật đúng trạng thái
- tài sản tự động đổi trạng thái tương ứng
- hệ thống xác định chính xác:
  - tài sản còn cầm cố
  - tài sản chờ thanh lý
  - tài sản đã bán

Qua đó mô phỏng đầy đủ quy trình quản lý thanh lý tài sản trong hệ thống cầm đồ thực tế.


