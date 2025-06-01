để sử dụng chương trình bạn cần cài python, tkinter, mysql.connector
để cài đặt tkinter, mysql.connector dùng câu lệnh trên termial:
pip install tkinter
pip install mysql-connector-python
Đảm bảo bạn có Database với 2 bảng: 
CREATE TABLE IF NOT EXISTS nguoi (
    cccd VARCHAR(20) PRIMARY KEY,
    ho_ten VARCHAR(100),
    so_dienthoai VARCHAR(15),
    tuoi INT,
    sodu0 BIGINT,
    sodu1 BIGINT,
    sodu2 BIGINT
);

CREATE TABLE IF NOT EXISTS giaodich (
    nguoi_chovay_cccd VARCHAR(20),
    nguoi_no_cccd VARCHAR(20),
    laisuat INT,
    sotien BIGINT,
    ngay_chovay DATE,
    lancuoi_capnhat DATE,
    ghichu TEXT,
    PRIMARY KEY (nguoi_chovay_cccd, nguoi_no_cccd, laisuat),
    FOREIGN KEY (nguoi_chovay_cccd) REFERENCES nguoi(cccd),
    FOREIGN KEY (nguoi_no_cccd) REFERENCES nguoi(cccd)
);

Cách sử dụng:
Đảm bảo số dư ở người và tổng các giao dịch là bằng nhau
Nếu dữ liệu của bạn chưa tối ưu thì hãy dùng chức năng tối ưu toàn bộ giao dịch trước khi thêm giao dịch mới. Vì giao dịch mới sẽ tối ưu ngay khi thêm vào
Khi nhập cccd ở giao dịch mới sẽ có 1 tab mới hiển thị và bạn sẽ tìm số mà người bạn mong muốn.


-- Nhóm 12 -- 
Hồ Quỳnh Anh
Phạm Thành Đạt
Nguyễn Quang Dương
Nguyễn Văn Việt