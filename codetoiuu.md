from tkinter import *
from tkinter import messagebox
import mysql.connector
from mysql.connector import Error
from datetime import date, datetime


# CHÚ Ý VỀ CÁCH ĐẶT TÊN:
# Các hàm được gọi trong các button sẽ được đặt tên có chữ "hàm" ở đầu.
# Ví dụ hàm thêm người dùng sẽ có tên là: ham_them_nguoi_dung
# Các form hiển thị giao diện sẽ được đặt tên có chữ "form" ở đầu.
# Ví dụ form thêm người dùng sẽ có tên là: form_them_nguoi_dung


# Tạo cửa sổ chính của ứng dụng giao diện
root = Tk()


# Biến toàn cục
global lst_Nguoi, lst_0, lst_1, lst_2
lst_Nguoi = []
lst_0 = []
lst_1 = []
lst_2 = []


# Xoá các widget cũ
def Destroy():
    for widget in frame_main.winfo_children():
        widget.destroy()


# --- Lớp Nguoi ---
# Định nghĩa lớp Nguoi để lưu thông tin người dùng (họ tên, CCCD, số điện thoại, tuổi, số dư theo lãi suất).
# Có property sodu để tính tổng số dư và phương thức __str__ để in thông tin.
class Nguoi:
    def __init__(self, ho_ten, cccd, so_dienthoai, tuoi, sodu0=0, sodu1=0, sodu2=0):
        self.ho_ten = ho_ten
        self.cccd = cccd
        self.so_dienthoai = so_dienthoai
        self.tuoi = tuoi
        self.sodu0 = sodu0
        self.sodu1 = sodu1
        self.sodu2 = sodu2


    @property
    def sodu(self):
        return self.sodu0 + self.sodu1 + self.sodu2


    def __str__(self):
        return f"{self.ho_ten} - CCCD: {self.cccd}, Tuổi: {self.tuoi}, SĐT: {self.so_dienthoai}, Số dư: {self.sodu}"


# --- Lớp GiaoDich ---
# Định nghĩa lớp GiaoDich để quản lý giao dịch (người cho vay, người nợ, số tiền, lãi suất, ngày vay, cập nhật cuối).
# Tự động cập nhật số dư khi khởi tạo, tính lãi suất theo thời gian với phương thức Lai_suat.
# Phương thức Thuc_hien để hủy giao dịch và cập nhật số dư ngược lại.
class GiaoDich:
    def __init__(self, nguoi_chovay, nguoi_no, sotien, laisuat, ngay_chovay, lancuoi_capnhat=None, ghichu=""):
        self.nguoi_chovay = nguoi_chovay
        self.nguoi_no = nguoi_no
        self.sotien = sotien
        self.laisuat = laisuat
        self.ngay_chovay = ngay_chovay
        self.ghichu = ghichu
        if lancuoi_capnhat is None:
            self.lancuoi_capnhat = ngay_chovay
        else:
            self.lancuoi_capnhat = lancuoi_capnhat
        if self.laisuat == 1:
            self.nguoi_chovay.sodu1 = self.nguoi_chovay.sodu1 + self.sotien
            self.nguoi_no.sodu1 = self.nguoi_no.sodu1 - self.sotien
        elif self.laisuat == 2:
            self.nguoi_chovay.sodu2 = self.nguoi_chovay.sodu2 + self.sotien
            self.nguoi_no.sodu2 = self.nguoi_no.sodu2 - self.sotien
        else:
            self.nguoi_chovay.sodu0 = self.nguoi_chovay.sodu0 + self.sotien
            self.nguoi_no.sodu0 = self.nguoi_no.sodu0 - self.sotien

    def Lai_suat(self, now):
        months_diff = (now.year - self.lancuoi_capnhat.year) * 12 + (now.month - self.lancuoi_capnhat.month)
        if self.laisuat == 1:
            self.nguoi_chovay.sodu1 = self.nguoi_chovay.sodu1 - self.sotien + (self.sotien * (1.01 ** months_diff)) // 1
            self.nguoi_no.sodu1 = self.nguoi_no.sodu1 + self.sotien - (self.sotien * (1.01 ** months_diff)) // 1
            self.sotien = (self.sotien * (1.01 ** months_diff)) // 1
        elif self.laisuat == 2:
            self.nguoi_chovay.sodu2 = self.nguoi_chovay.sodu2 - self.sotien + (self.sotien * (1.02 ** months_diff)) // 1
            self.nguoi_no.sodu2 = self.nguoi_no.sodu2 + self.sotien - (self.sotien * (1.02 ** months_diff)) // 1
            self.sotien = (self.sotien * (1.02 ** months_diff)) // 1
        self.lancuoi_capnhat = now

    def Thuc_hien(self):
        if self.laisuat == 1:
            self.nguoi_chovay.sodu1 = self.nguoi_chovay.sodu1 - self.sotien
            self.nguoi_no.sodu1 = self.nguoi_no.sodu1 + self.sotien
        elif self.laisuat == 2:
            self.nguoi_chovay.sodu2 = self.nguoi_chovay.sodu2 - self.sotien
            self.nguoi_no.sodu2 = self.nguoi_no.sodu2 + self.sotien
        else:
            self.nguoi_chovay.sodu0 = self.nguoi_chovay.sodu0 - self.sotien
            self.nguoi_no.sodu0 = self.nguoi_no.sodu0 + self.sotien




    
# ========= Hàm MySQL =========
# --- Hàm kết nối và quản lý MySQL ---
# Doc_diachisql: Đọc thông tin kết nối MySQL từ file address_mysql.txt, thử kết nối và trả về connection.
# Ket_noisql: Cho phép nhập mới hoặc kiểm tra thông tin kết nối MySQL, lưu vào file nếu thành công.
def Doc_diachisql():
    try:
        with open("address_mysql.txt", "a+") as file:
            file.seek(0)
            lines = [line.strip() for line in file.readlines()]
            if len(lines) < 5:
                print("Lỗi: File address_mysql.txt không đủ thông tin kết nối.")
                return 1
            host, user, password, database, port = lines[:5]
            if not all([host, user, database, port]):
                print("Lỗi: Một hoặc nhiều thông tin kết nối bị rỗng.")
                return 1
            try:
                conn = mysql.connector.connect(
                    host=host,
                    user=user,
                    password=password,
                    database=database,
                    port=port
                )
                if conn.is_connected():
                    return conn
            except Error as e:
                print(f"Lỗi kết nối MySQL: {e}")
                return 1
    except Exception as e:
        print(f"Lỗi khi đọc file: {e}")
        return 1


def Ket_noisql():
    file = open("address_mysql.txt", "a+")
    file.seek(0)
    noidung = file.read()
    if noidung:
        print("Đã được kết nối với MySQL. Thông tin kết nối hiện tại:")
        file.seek(0)
        lines = file.readlines()
        if len(lines) >= 5:
            host, user, password, database, port = [line.strip() for line in lines[:5]]
            print(f"Host: {host}, User: {user}, Password: {'*' * len(password)}, Database: {database}, Port: {port}")
            try:
                connection = mysql.connector.connect(
                    host=host,
                    user=user,
                    password=password,
                    database=database,
                    port=port
                )
                if connection.is_connected():
                    print("Kết nối đến MySQL vẫn hoạt động!")
                    cursor = connection.cursor()
                    cursor.execute("SELECT DATABASE();")
                    db_name = cursor.fetchone()
                    print(f"Đang kết nối đến database: {db_name[0]}")
            except Error as e:
                print(f"Lỗi kết nối đến MySQL: {e}")
                file.close()
                return 1
            finally:
                if 'connection' in locals() and connection.is_connected():
                    cursor.close()
                    connection.close()
            try:
                sign = int(input("Ấn phím 1 nếu muốn thay đổi kết nối, 0 để giữ nguyên: "))
                if sign != 1:
                    file.close()
                    return 0
            except ValueError:
                print("Vui lòng nhập số hợp lệ (0 hoặc 1)!")
                file.close()
                return 1
    host = input("Nhập host: ")
    user = input("Nhập tên user: ")
    password = input("Nhập mật khẩu: ")
    database = input("Nhập tên database: ")
    port = input("Nhập port (mặc định 3306): ") or "3306"
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database,
            port=port
        )
        if connection.is_connected():
            print("Kết nối đến MySQL thành công!")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()
            print(f"Đang kết nối đến database: {db_name[0]}")
    except Error as e:
        print(f"Lỗi kết nối đến MySQL: {e}")
        file.close()
        return 1
    finally:
        if 'connection' in locals() and connection.is_connected():
            cursor.close()
            connection.close()
    file.seek(0)
    file.truncate()
    print(host, file=file)
    print(user, file=file)
    print(password, file=file)
    print(database, file=file)
    print(port, file=file, end="")
    file.close()
    return 0


# --- Hàm đọc/ghi dữ liệu MySQL ---
# Doc_nguoi_tu_mysql: Lấy danh sách người từ bảng nguoi, tạo đối tượng Nguoi và lưu vào lst_Nguoi.
# Ghi_nguoi_vaomysql: Thêm một người mới vào bảng nguoi.
# Ghi_sodu_nguoi_vaomysql: Cập nhật số dư của tất cả người trong bảng nguoi.
# Doc_giaodich_tu_db: Lấy danh sách giao dịch từ bảng giaodich, tạo đối tượng GiaoDich và phân loại theo lãi suất.
# Ghi_giaodich_vaomysql: Ghi tất cả giao dịch từ lst_0, lst_1, lst_2 vào bảng giaodich.
# Xoa_toanbo_giaodich: Xóa toàn bộ giao dịch trong bảng giaodich.
# Xoa_nguoi: Xóa một người khỏi bảng nguoi, cập nhật số dư liên quan nếu cần.
# Xoa_Giaodich_Capnhat_Sodu: Xóa một giao dịch và cập nhật số dư của người liên quan.
def Doc_nguoi_tu_mysql(conn):
    global lst_Nguoi
    lst_Nguoi.clear()
    cursor = conn.cursor()
    cursor.execute("SELECT ho_ten, cccd, so_dienthoai, tuoi, sodu0, sodu1, sodu2 FROM nguoi")
    results = cursor.fetchall()
    for row in results:
        ho_ten, cccd, so_dienthoai, tuoi, sodu0, sodu1, sodu2 = row
        n = Nguoi(ho_ten, cccd, so_dienthoai, tuoi, sodu0, sodu1, sodu2)
        lst_Nguoi.append(n)
    cursor.close()
    return lst_Nguoi


# --- Hàm hỗ trợ tối ưu giao dịch ---
# Timkiem_khoanno: Tìm các khoản nợ mà một người cho vay, trả về danh sách [số tiền, giao dịch].
# Timkiem_khoanvay: Tìm các khoản vay mà một người nợ, trả về danh sách [số tiền, giao dịch].
# lay_to_hop_n: Tạo các tổ hợp n phần tử từ danh sách, dùng để kiểm tra tổng số tiền giao dịch.
# Timkiem_nguoicungno: Tìm các cặp giao dịch có cùng người nợ giữa hai danh sách.
def Timkiem_khoanno(lst_giaodich,nguoi): # hàm sẽ trả list người nợ mình với các phần từ có dạng [Số tiền, giao dịch]
    lst = []
    for i in lst_giaodich:
        if i.nguoi_chovay == nguoi:
            tmp=[]
            tmp.append(i.sotien)
            tmp.append(i)
            lst.append(tmp)
    return lst


def Timkiem_khoanvay(lst_giaodich,nguoi): # hàm sẽ trả list người cho mình vay với các phần từ có dạng [Số tiền, giao dịch]
    lst = []
    for i in lst_giaodich:
        if i.nguoi_no == nguoi:
            tmp=[]
            tmp.append(i.sotien)
            tmp.append(i)
            lst.append(tmp)
    return lst


def lay_to_hop_n(danh_sach, so_phan_tu):
    def quay_lui(bat_dau, to_hop_hien_tai):
        # Nếu tổ hợp hiện tại đạt độ dài mong muốn, thêm vào kết quả
        if len(to_hop_hien_tai) == so_phan_tu:
            ket_qua.append(to_hop_hien_tai[:])
            return
       
        # Thử thêm các phần tử từ vị trí bắt đầu trở đi
        for i in range(bat_dau, len(danh_sach)):
            to_hop_hien_tai.append(danh_sach[i])
            quay_lui(i + 1, to_hop_hien_tai)
            to_hop_hien_tai.pop()
    ket_qua = []
    # Kiểm tra điều kiện đầu vào
    if so_phan_tu < 0 or so_phan_tu > len(danh_sach):
        return ket_qua
    quay_lui(0, [])
    return ket_qua


def Timkiem_nguoicungno(lst_nguoi1,lstnguoi2):
    lst = []
    for i in lst_nguoi1:
        for j in lstnguoi2:
            if i[1].nguoi_no ==j[1].nguoi_no:
                lst.append([i,j])
    return lst


# --- Hàm xử lý và tối ưu giao dịch ---
# Chuyen_giaodich: Chuyển giao dịch từ người nợ sang người cho vay, ưu tiên các khoản nợ chung, cập nhật ghi chú.
# Toi_uu: Tối ưu danh sách giao dịch bằng cách hợp nhất hoặc xóa các giao dịch trùng lặp.
# Xu_ly: Xử lý một giao dịch mới, kiểm tra trùng lặp, tính lãi suất, và gọi Chuyen_giaodich hoặc Toi_uu nếu cần.
# In_giaodich, In_toanbogiaodich: In thông tin một hoặc tất cả giao dịch (dùng cho debug hoặc hiển thị).
# Capnhat_laisuat: Cập nhật lãi suất cho danh sách giao dịch dựa trên thời gian hiện tại.
# Kiemtra_giaodich_trung: Kiểm tra giao dịch mới có trùng với giao dịch cũ, hợp nhất hoặc xóa nếu cần.
# Toi_uu_toan_bo: Tối ưu toàn bộ giao dịch trong cơ sở dữ liệu, lặp lại đến khi không còn thay đổi.
def Chuyen_giaodich(lst_giaodich, giaodich_moi, sodu_no):
    lst_nguoino_nguoino = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_no)
    lst_nguoino_nguoivay = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_chovay)
    lst_nguoicungno = Timkiem_nguoicungno(lst_nguoino_nguoivay, lst_nguoino_nguoino)
    sotien_moi = giaodich_moi.sotien
    if sodu_no > 0:  # Người cho nợ có số dư lớn hơn 0 nên có thể chuyển khoản vay của mình cho thg vay
        if lst_nguoicungno != []:  # Kiểm tra xem 2 thg cho vay với nợ thì có thằng nào nợ chung 2 thg này ko
            sum = 0
            for i in lst_nguoicungno:
                if sum + i[1][0] <= sotien_moi:
                    sum += i[1][0]
                    i[0][1].sotien += i[1][0]
                    if i[0][1].ghichu is None:
                        i[0][1].ghichu = ""
                    i[0][1].ghichu += " " + "Đã chuyển thêm " + str(i[1][0]) + " do người vay có cùng khoản nợ với " + i[0][1].nguoi_chovay.ho_ten + " và " + i[1][1].nguoi_chovay.ho_ten + "\n"
                    lst_giaodich.remove(i[1][1])
                elif sum < sotien_moi:
                    i[1][1].sotien = i[1][1].sotien - sotien_moi + sum
                    if i[1][1].ghichu is None:
                        i[1][1].ghichu = ""
                    i[1][1].ghichu += " " + "Đã chuyển bớt " + str(sotien_moi - sum) + " sang " + i[0][1].nguoi_chovay.ho_ten + " \n"
                    i[0][1].sotien += sotien_moi - sum
                    i[0][1].nguoi_no = i[1][1].nguoi_no
                    if i[0][1].ghichu is None:
                        i[0][1].ghichu = ""
                    i[0][1].ghichu += " " + "Đã chuyển giao dịch từ " + i[0][1].nguoi_no.ho_ten + " sang\n"
                    sum = sotien_moi
                    break
            sotien_moi = sotien_moi - sum
        if sotien_moi > 0:  # Chuyển từng giao dịch của người nợ cho người vay
            lst_nguoino_nguoino = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_no)
            to_hop1 = lay_to_hop_n(lst_nguoino_nguoino, 1)
            for i in to_hop1:  # Kiểm tra xem số tiền nợ có thể bằng tổng của 1 giao dịch của thg nợ ko
                if sotien_moi == i[0][0]:
                    i[0][1].nguoi_chovay = giaodich_moi.nguoi_chovay
                    if i[0][1].ghichu is None:
                        i[0][1].ghichu = ""
                    i[0][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
                    return
            to_hop2 = lay_to_hop_n(lst_nguoino_nguoino, 2)
            for i in to_hop2:  # Kiểm tra xem số tiền nợ có thể bằng tổng của 2 giao dịch của thg nợ ko
                sum = i[0][0] + i[1][0]
                if sotien_moi == sum:
                    i[0][1].nguoi_chovay = giaodich_moi.nguoi_chovay
                    if i[0][1].ghichu is None:
                        i[0][1].ghichu = ""
                    i[0][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
                    i[1][1].nguoi_chovay = giaodich_moi.nguoi_chovay
                    if i[1][1].ghichu is None:
                        i[1][1].ghichu = ""
                    i[1][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
                    return
            sum = 0
            for i in lst_nguoino_nguoino:  # Nếu ko thì chuyển lần lượt số tiền đến khi đủ
                if sum + i[0] <= sotien_moi:
                    sum += i[0]
                    i[1].nguoi_chovay = giaodich_moi.nguoi_chovay
                    if i[1].ghichu is None:
                        i[1].ghichu = ""
                    i[1].ghichu += " " + "Đã chuyển giao dịch từ " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
                elif sum < sotien_moi:
                    i[1].sotien = i[1].sotien - sotien_moi + sum
                    if i[1].ghichu is None:
                        i[1].ghichu = ""
                    i[1].ghichu += " " + "Đã chuyển bớt " + str(sotien_moi - sum) + " sang " + giaodich_moi.nguoi_chovay.ho_ten + " \n"
                    giaodich_moi.sotien = sotien_moi - sum
                    giaodich_moi.nguoi_no = i[1].nguoi_no
                    if giaodich_moi.ghichu is None:
                        giaodich_moi.ghichu = ""
                    giaodich_moi.ghichu += " " + "Đã chuyển giao dịch từ " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
                    lst_giaodich.append(giaodich_moi)
                    break
    elif sodu_no == 0:  # Trường hợp số dư thg nợ bằng 0 thì phải chuyển toàn bộ tài khoản của mình
        if lst_nguoicungno != []:  # Kiểm tra xem 2 thg cho vay với nợ thì có thằng nào nợ chung 2 thg này ko
            sum = 0
            for i in lst_nguoicungno:
                sum += i[1][0]
                i[0][1].sotien += i[1][0]
                if i[0][1].ghichu is None:
                    i[0][1].ghichu = ""
                i[0][1].ghichu += " " + "Đã chuyển thêm " + str(i[1][0]) + " do người vay có cùng khoản nợ với " + i[0][1].nguoi_chovay.ho_ten + " và " + i[1][1].nguoi_chovay.ho_ten + "\n"
                lst_giaodich.remove(i[1][1])
            sotien_moi = sotien_moi - sum
        lst_nguoino_nguoino = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_no)
        for i in lst_nguoino_nguoino:
            i[1].nguoi_chovay = giaodich_moi.nguoi_chovay
            if i[1].ghichu is None:
                i[1].ghichu = ""
            i[1].ghichu += " " + "Đã chuyển giao dịch từ " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
    else:
        if lst_nguoicungno != []:  # Kiểm tra xem 2 thg cho vay với nợ thì có thằng nào nợ chung 2 thg này ko
            sum = 0
            for i in lst_nguoicungno:
                sum += i[1][0]
                i[0][1].sotien += i[1][0]
                if i[0][1].ghichu is None:
                    i[0][1].ghichu = ""
                i[0][1].ghichu += " " + "Đã chuyển thêm " + str(i[1][0]) + " do người vay có cùng khoản nợ với " + i[0][1].nguoi_chovay.ho_ten + " và " + i[1][1].nguoi_chovay.ho_ten + "\n"
                lst_giaodich.remove(i[1][1])
            sotien_moi = sotien_moi - sum
        lst_nguoino_nguoino = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_no)
        sum = 0
        for i in lst_nguoino_nguoino:
            sum += i[1].sotien
            i[1].nguoi_chovay = giaodich_moi.nguoi_chovay
            if i[1].ghichu is None:
                i[1].ghichu = ""
            i[1].ghichu += " " + "Đã chuyển giao dịch từ " + giaodich_moi.nguoi_no.ho_ten + " sang\n"
        # Không đặt lại ghichu thành "" mà giữ nguyên giá trị hiện tại
        giaodich_moi.sotien = sotien_moi - sum
        if giaodich_moi.ghichu is None:
            giaodich_moi.ghichu = ""
        lst_giaodich.append(giaodich_moi)




def Toi_uu(lst_giaodich, giaodich_moi):
    lst_khoanvay = Timkiem_khoanvay(lst_giaodich, giaodich_moi.nguoi_chovay)
    lst_khoanno = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_chovay)
    if lst_khoanvay == []:
        return
    else:
        for i in lst_khoanno[:]:
            lst_khoanvay = Timkiem_khoanvay(lst_giaodich, giaodich_moi.nguoi_chovay)
            for j in lst_khoanvay[:]:
                tmp = Timkiem_khoanno(lst_giaodich, j[1].nguoi_chovay)
                for z in tmp:
                    if z[1].nguoi_no == i[1].nguoi_no:
                        if z[1].ghichu is None:
                            z[1].ghichu = ""
                        if j[1].sotien > i[1].sotien:
                            z[1].sotien += i[1].sotien
                            if i[1] in lst_giaodich:
                                lst_giaodich.remove(i[1])
                            j[1].sotien = j[1].sotien - i[1].sotien
                            z[1].ghichu += "Đã cộng thêm tiền do trùng lặp \n"
                        elif j[1].sotien == i[1].sotien:
                            z[1].sotien += i[1].sotien
                            if i[1] in lst_giaodich:
                                lst_giaodich.remove(i[1])
                            if j[1] in lst_giaodich:
                                lst_giaodich.remove(j[1])
                            z[1].ghichu += "Đã cộng thêm tiền do trùng lặp \n"
                        else:
                            z[1].sotien += j[1].sotien
                            i[1].sotien -= j[1].sotien
                            if j[1] in lst_giaodich:
                                lst_giaodich.remove(j[1])
                            z[1].ghichu += "Đã cộng thêm tiền do trùng lặp \n"
        lst_khoanno = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_chovay)
        tmp_lst = []
        for i in lst_khoanno[:]:
            lst_khoanvay = Timkiem_khoanvay(lst_giaodich, giaodich_moi.nguoi_chovay)
            to_hop2 = lay_to_hop_n(lst_khoanvay, 2)
            for j in to_hop2:
                sum = j[0][0] + j[1][0]
                if i[0] == sum:
                    if j[0][1].ghichu is None:
                        j[0][1].ghichu = ""
                    if j[1][1].ghichu is None:
                        j[1][1].ghichu = ""
                    j[0][1].nguoi_no = i[1].nguoi_no
                    j[0][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    j[1][1].nguoi_no = i[1].nguoi_no
                    j[1][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    tmp_lst.append(i[1])
                    break
        for i in tmp_lst:
            if i in lst_giaodich:
                lst_giaodich.remove(i)
        tmp_lst = []
        lst_khoanvay = Timkiem_khoanvay(lst_giaodich, giaodich_moi.nguoi_chovay)
        for i in lst_khoanvay[:]:
            lst_khoanno = Timkiem_khoanno(lst_giaodich, giaodich_moi.nguoi_chovay)
            to_hop1 = lay_to_hop_n(lst_khoanno, 1)
            for j in to_hop1:
                if i[0] == j[0][0]:
                    if j[0][1].ghichu is None:
                        j[0][1].ghichu = ""
                    j[0][1].nguoi_chovay = i[1].nguoi_chovay
                    j[0][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    tmp_lst.append(i[1])
                    break
            to_hop2 = lay_to_hop_n(lst_khoanno, 2)
            for j in to_hop2:
                sum = j[0][0] + j[1][0]
                if i[0] == sum:
                    if j[0][1].ghichu is None:
                        j[0][1].ghichu = ""
                    if j[1][1].ghichu is None:
                        j[1][1].ghichu = ""
                    j[0][1].nguoi_chovay = i[1].nguoi_chovay
                    j[0][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    j[1][1].nguoi_chovay = i[1].nguoi_chovay
                    j[1][1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    tmp_lst.append(i[1])
                    break
            for j in lst_khoanno[:]:
                if i[1].sotien > j[1].sotien:
                    if j[1].ghichu is None:
                        j[1].ghichu = ""
                    j[1].nguoi_chovay = i[1].nguoi_chovay
                    i[1].sotien = i[1].sotien - j[1].sotien
                    j[1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                elif i[1].sotien == j[1].sotien:
                    if j[1].ghichu is None:
                        j[1].ghichu = ""
                    j[1].nguoi_chovay = i[1].nguoi_chovay
                    tmp_lst.append(i[1])
                    j[1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    break
                else:
                    if i[1].ghichu is None:
                        i[1].ghichu = ""
                    j[1].sotien = j[1].sotien - i[1].sotien
                    i[1].nguoi_no = j[1].nguoi_no
                    i[1].ghichu += " " + "Đã chuyển giao dịch của " + giaodich_moi.nguoi_chovay.ho_ten + " sang\n"
                    break
        for i in tmp_lst:
            if i in lst_giaodich:
                lst_giaodich.remove(i)




def Xu_ly(lst_0,lst_1,lst_2, giaodich_moi):
    laisuat = giaodich_moi.laisuat
    if laisuat == 1:
        sodu_no = giaodich_moi.nguoi_no.sodu1
        lst_giaodich = lst_1
    elif laisuat == 2:
        sodu_no = giaodich_moi.nguoi_no.sodu2
        lst_giaodich = lst_2
    else:
        sodu_no = giaodich_moi.nguoi_no.sodu0
        lst_giaodich = lst_0
    Capnhat_laisuat(lst_giaodich) # Tính só dư hiện tại của các giao dịch trước đó
    now = datetime.now()
    giaodich_moi.Lai_suat(now) # Để tính số dư hiện tại vì 2 khoản vay có tính lãi không thể tương tác nếu ko cùng ngày cập nhật
    if Kiemtra_giaodich_trung(lst_giaodich, giaodich_moi):
        None
    else:
        Chuyen_giaodich(lst_giaodich,giaodich_moi,sodu_no)
        Toi_uu(lst_giaodich, giaodich_moi)


def Ghi_nguoi_vaomysql(nguoi, conn):
    cursor = conn.cursor()
    sql = """
    INSERT INTO nguoi (cccd, ho_ten, so_dienthoai, tuoi, sodu0, sodu1, sodu2)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    """
    try:
        cursor.execute(sql, (
            nguoi.cccd,
            nguoi.ho_ten,
            nguoi.so_dienthoai,
            nguoi.tuoi,
            nguoi.sodu0,
            nguoi.sodu1,
            nguoi.sodu2
        ))
        conn.commit()
        return 0
    except mysql.connector.Error as err:
        print(f"Lỗi khi thêm người {nguoi.ho_ten}: {err}")
        return 1
    finally:
        cursor.close()






def Doc_giaodich_tu_db(conn, lst_nguoi):
    global lst_0, lst_1, lst_2
    lst_0, lst_1, lst_2 = [], [], []
    cursor = conn.cursor()
    nguoi_dict = {n.cccd: n for n in lst_nguoi}
    cursor.execute("SELECT nguoi_chovay_cccd, nguoi_no_cccd, sotien, laisuat, ngay_chovay, lancuoi_capnhat, ghichu FROM giaodich")
    results = cursor.fetchall()
    for row in results:
        nguoi_chovay_cccd, nguoi_no_cccd, sotien, laisuat, ngay_chovay, lancuoi_capnhat, ghichu = row
        nguoi_chovay = nguoi_dict.get(nguoi_chovay_cccd)
        nguoi_no = nguoi_dict.get(nguoi_no_cccd)
        if nguoi_chovay and nguoi_no:
            gd = GiaoDich(
                nguoi_chovay=nguoi_chovay,
                nguoi_no=nguoi_no,
                sotien=sotien,
                laisuat=laisuat,
                ngay_chovay=ngay_chovay,
                lancuoi_capnhat=lancuoi_capnhat,
                ghichu=ghichu
            )
            if laisuat == 0:
                lst_0.append(gd)
            elif laisuat == 1:
                lst_1.append(gd)
            elif laisuat == 2:
                lst_2.append(gd)
    cursor.close()
    return lst_0, lst_1, lst_2


def Ghi_giaodich_vaomysql(lst_0, lst_1, lst_2, conn):
    lst = lst_0 + lst_1 + lst_2
    cursor = conn.cursor()
    sql = """
    INSERT INTO giaodich (nguoi_chovay_cccd, nguoi_no_cccd, laisuat, sotien, ngay_chovay, lancuoi_capnhat, ghichu)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    """
    for i in lst:
        try:
            cursor.execute(sql, (
                i.nguoi_chovay.cccd,
                i.nguoi_no.cccd,
                i.laisuat,
                i.sotien,
                i.ngay_chovay,
                i.lancuoi_capnhat,
                i.ghichu
            ))
            conn.commit()
        except mysql.connector.Error as err:
            print(f"Lỗi khi thêm giao dịch: {err}")
    cursor.close()


def Xoa_toanbo_giaodich(conn):
    cursor = conn.cursor()
    try:
        cursor.execute("SET FOREIGN_KEY_CHECKS = 0")
        cursor.execute("SET SQL_SAFE_UPDATES = 0")
        cursor.execute("DELETE FROM giaodich")
        cursor.execute("SET SQL_SAFE_UPDATES = 1")
        cursor.execute("SET FOREIGN_KEY_CHECKS = 1")
        conn.commit()
    except mysql.connector.Error as err:
        print(f"Lỗi: {err}")
        conn.rollback()
    finally:
        cursor.close()


def Xoa_nguoi(conn, cccd_can_xoa):
    cursor = conn.cursor()
    cursor.execute("""
        SELECT sodu0, sodu1, sodu2 FROM nguoi WHERE cccd = %s
    """, (cccd_can_xoa,))
    row = cursor.fetchone()
    if row is None:
        cursor.close()
        return 1
    elif all(sodu == 0 for sodu in row):
        cursor.execute("DELETE FROM nguoi WHERE cccd = %s", (cccd_can_xoa,))
        conn.commit()
        cursor.close()
        return 0
    else:
        cursor.execute("""
            SELECT nguoi_chovay_cccd, nguoi_no_cccd, laisuat, sotien
            FROM giaodich
            WHERE nguoi_chovay_cccd = %s OR nguoi_no_cccd = %s
        """, (cccd_can_xoa, cccd_can_xoa))
        giao_dichs = cursor.fetchall()
        for chovay_cccd, no_cccd, laisuat, sotien in giao_dichs:
            sodu_col = f"sodu{laisuat}"
            if chovay_cccd == cccd_can_xoa:
                cursor.execute(f"""
                    UPDATE nguoi SET {sodu_col} = {sodu_col} + %s WHERE cccd = %s
                """, (sotien, no_cccd))
            elif no_cccd == cccd_can_xoa:
                cursor.execute(f"""
                    UPDATE nguoi SET {sodu_col} = {sodu_col} - %s WHERE cccd = %s
                """, (sotien, chovay_cccd))
        cursor.execute("""
            DELETE FROM giaodich WHERE nguoi_chovay_cccd = %s OR nguoi_no_cccd = %s
        """, (cccd_can_xoa, cccd_can_xoa))
        cursor.execute("DELETE FROM nguoi WHERE cccd = %s", (cccd_can_xoa,))
        conn.commit()
        cursor.close()
        return 0


def Xoa_Giaodich_Capnhat_Sodu(conn, nguoi_chovay_cccd, nguoi_no_cccd, laisuat):
    cursor = conn.cursor()
    try:
        cursor.execute("""
            SELECT sotien FROM giaodich
            WHERE nguoi_chovay_cccd = %s AND nguoi_no_cccd = %s AND laisuat = %s
        """, (nguoi_chovay_cccd, nguoi_no_cccd, laisuat))
        result = cursor.fetchone()
        if result is None:
            print("Không tìm thấy giao dịch để xóa.")
            return 1
        sotien = result[0]
        sodu_col = f"sodu{laisuat}"
        cursor.execute(f"""
            UPDATE nguoi SET {sodu_col} = {sodu_col} - %s WHERE cccd = %s
        """, (sotien, nguoi_chovay_cccd))
        cursor.execute(f"""
            UPDATE nguoi SET {sodu_col} = {sodu_col} + %s WHERE cccd = %s
        """, (sotien, nguoi_no_cccd))
        cursor.execute("""
            DELETE FROM giaodich
            WHERE nguoi_chovay_cccd = %s AND nguoi_no_cccd = %s AND laisuat = %s
        """, (nguoi_chovay_cccd, nguoi_no_cccd, laisuat))
        conn.commit()
        return 0
    except mysql.connector.Error as err:
        print(f"Lỗi MySQL: {err}")
        conn.rollback()
        return 1
    finally:
        cursor.close()


def In_giaodich(giaodich):
    print("Người cho vay:", giaodich.nguoi_chovay.ho_ten)
    print("Người nợ:", giaodich.nguoi_no.ho_ten)
    print("Số tiền:", giaodich.sotien, "với lãi suất", giaodich.laisuat, "%/tháng:")
    print("Ghi chú:", giaodich.ghichu)
    print()


def In_toanbogiaodich(lst_0, lst_1, lst_2):
    for i in lst_0:
        In_giaodich(i)
    for i in lst_1:
        In_giaodich(i)
    for i in lst_2:
        In_giaodich(i)


def Capnhat_laisuat(lst_giaodich):
    now = datetime.now()
    for i in lst_giaodich:
        i.Lai_suat(now)


def Kiemtra_giaodich_trung(lst_giaodich, giaodich_moi):
    for i in lst_giaodich:
        if (giaodich_moi.nguoi_chovay == i.nguoi_chovay) and (giaodich_moi.nguoi_no == i.nguoi_no):
            i.sotien += giaodich_moi.sotien
            i.ghichu += "Đã cộng thêm " + str(giaodich_moi.sotien) + " do khoản vay trùng lặp\n"
            return True
        if (giaodich_moi.nguoi_chovay == i.nguoi_no) and (giaodich_moi.nguoi_no == i.nguoi_chovay):
            if i.sotien == giaodich_moi.sotien:
                lst_giaodich.remove(i)
                print("Khoản vay mới và cũ đã đối nghịch nhau. Xóa bỏ khoản vay trước đó")
                return True
            elif i.sotien > giaodich_moi.sotien:
                i.sotien -= giaodich_moi.sotien
                i.ghichu += "Đã bỏ khoản vay mới và bỏ " + str(giaodich_moi.sotien) + " ở khoản vay cũ do khoản vay trùng lặp\n"
                return True
            else:
                lst_giaodich.remove(i)
                giaodich_moi.sotien = giaodich_moi.sotien - i.sotien
                return False
    return False


def Toi_uu_toan_bo(conn):
    global lst_0, lst_1, lst_2, lst_Nguoi
    
    # Bước 1: Đọc danh sách người từ MySQL
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    nguoi_dict = {n.cccd: n for n in lst_Nguoi}


    # Bước 2: Đọc toàn bộ giao dịch từ MySQL
    cursor = conn.cursor()
    cursor.execute("SELECT nguoi_chovay_cccd, nguoi_no_cccd, sotien, laisuat, ngay_chovay, lancuoi_capnhat, ghichu FROM giaodich")
    results = cursor.fetchall()


    # Bước 3: Khởi tạo danh sách giao dịch và xử lý từng giao dịch
    lst_0, lst_1, lst_2 = [], [], []
    for row in results:
        nguoi_chovay_cccd, nguoi_no_cccd, sotien, laisuat, ngay_chovay, lancuoi_capnhat, ghichu = row
        nguoi_chovay = nguoi_dict.get(nguoi_chovay_cccd)
        nguoi_no = nguoi_dict.get(nguoi_no_cccd)


        if nguoi_chovay and nguoi_no:
            gd = GiaoDich(
                nguoi_chovay=nguoi_chovay,
                nguoi_no=nguoi_no,
                sotien=sotien,
                laisuat=laisuat,
                ngay_chovay=ngay_chovay,
                lancuoi_capnhat=lancuoi_capnhat,
                ghichu=ghichu if ghichu is not None else ""
            )
            Xu_ly(lst_0, lst_1, lst_2, gd)


    # Bước 4: Xóa toàn bộ giao dịch trong cơ sở dữ liệu
    Xoa_toanbo_giaodich(conn)


    # Bước 5: Ghi lại các giao dịch đã xử lý vào MySQL
    Ghi_giaodich_vaomysql(lst_0, lst_1, lst_2, conn)


    # Bước 6: Đóng con trỏ
    cursor.close()


    
def form_ket_noi_sql():
    def ghi_va_ket_noi():
        host = host_var.get().strip()
        user = user_var.get().strip()
        password = pass_var.get().strip()
        database = db_var.get().strip()
        port = port_var.get().strip() or "3306"
        if not all([host, user, database, port]):
            messagebox.showerror("Lỗi", "Vui lòng nhập đầy đủ thông tin (trừ mật khẩu có thể để trống).")
            return
        try:
            conn = mysql.connector.connect(
                host=host,
                user=user,
                password=password,
                database=database,
                port=port
            )
            if conn.is_connected():
                with open("address_mysql.txt", "w") as file:
                    print(host, file=file)
                    print(user, file=file)
                    print(password, file=file)
                    print(database, file=file)
                    print(port, file=file, end="")
                messagebox.showinfo("Thành công", "Kết nối thành công và đã lưu vào file.")
                form.destroy()
        except Error as e:
            messagebox.showerror("Lỗi kết nối", f"Kết nối thất bại:\n{e}")
        finally:
            if 'conn' in locals() and conn.is_connected():
                conn.close()


    form = Toplevel(root)
    form.title("Thiết lập kết nối MySQL")
    form.geometry("400x250")
    form.grab_set()
    Label(form, text="Host:").grid(row=0, column=0, sticky=W, padx=10, pady=5)
    host_var = StringVar()
    Entry(form, textvariable=host_var).grid(row=0, column=1, padx=10, pady=5)
    
    Label(form, text="User:").grid(row=1, column=0, sticky=W, padx=10, pady=5)
    user_var = StringVar()
    Entry(form, textvariable=user_var).grid(row=1, column=1, padx=10, pady=5)
    
    Label(form, text="Password:").grid(row=2, column=0, sticky=W, padx=10, pady=5)
    pass_var = StringVar()
    Entry(form, textvariable=pass_var, show="*").grid(row=2, column=1, padx=10, pady=5)
    
    Label(form, text="Database:").grid(row=3, column=0, sticky=W, padx=10, pady=5)
    db_var = StringVar()
    Entry(form, textvariable=db_var).grid(row=3, column=1, padx=10, pady=5)
    
    Label(form, text="Port:").grid(row=4, column=0, sticky=W, padx=10, pady=5)
    port_var = StringVar(value="3306")
    Entry(form, textvariable=port_var).grid(row=4, column=1, padx=10, pady=5)
    
    Button(form, text="Lưu và kết nối", command=ghi_va_ket_noi).grid(row=5, column=0, columnspan=2, pady=15)


def form_them_nguoi():
    Destroy()
    Label(frame_main, text="Nhập thông tin người muốn thêm:", font=("Segoe UI", 14)).grid(row=0, column=0, columnspan=2, padx=5, pady=5)
    Label(frame_main, font=("Segoe UI", 11), text="Họ tên:").grid(row=1, column=0, sticky=E, padx=5, pady=5)
    ho_ten_var = StringVar()
    
    Entry(frame_main, font=("Segoe UI", 11), textvariable=ho_ten_var).grid(row=1, column=1, padx=5, pady=5)
    Label(frame_main, font=("Segoe UI", 11), text="CCCD:").grid(row=2, column=0, sticky=E, padx=5, pady=5)
    cccd_var = StringVar()
    
    Entry(frame_main, font=("Segoe UI", 11), textvariable=cccd_var).grid(row=2, column=1, padx=5, pady=5)
    Label(frame_main, font=("Segoe UI", 11), text="Tuổi:").grid(row=3, column=0, sticky=E, padx=5, pady=5)
    tuoi_var = StringVar()
    
    Entry(frame_main, font=("Segoe UI", 11), textvariable=tuoi_var).grid(row=3, column=1, padx=5, pady=5)
    Label(frame_main, font=("Segoe UI", 11), text="Số điện thoại:").grid(row=4, column=0, sticky=E, padx=5, pady=5)
    sdt_var = StringVar()
    
    Entry(frame_main, font=("Segoe UI", 11), textvariable=sdt_var).grid(row=4, column=1, padx=5, pady=5)


    def ham_them_nguoi_dung():
        ho_ten = ho_ten_var.get().strip()
        cccd = cccd_var.get().strip()
        tuoi = tuoi_var.get().strip()
        sdt = sdt_var.get().strip()
        if not ho_ten or not cccd:
            messagebox.showwarning("Thiếu thông tin", "Vui lòng nhập đầy đủ họ tên và CCCD")
            return
        try:
            tuoi = int(tuoi) if tuoi else 0
            conn = Doc_diachisql()
            if conn == 1:
                messagebox.showerror("Lỗi", "Không thể kết nối đến MySQL")
                return
            nguoi = Nguoi(ho_ten, cccd, sdt, tuoi, 0, 0, 0)
            result = Ghi_nguoi_vaomysql(nguoi, conn)
            if result == 0:
                global lst_Nguoi
                lst_Nguoi.append(nguoi)
                messagebox.showinfo("Thành công", "Người dùng đã được thêm vào MySQL!")
                ho_ten_var.set("")
                cccd_var.set("")
                tuoi_var.set("")
                sdt_var.set("")
            else:
                messagebox.showerror("Lỗi", "Thêm người thất bại. Vui lòng kiểm tra lại dữ liệu.")
            conn.close()
        except ValueError:
            messagebox.showerror("Lỗi", "Tuổi phải là số nguyên!")
        except Exception as e:
            messagebox.showerror("Lỗi", f"Đã xảy ra lỗi: {e}")


    Button(frame_main, font=("Segoe UI", 11), text="Thêm người dùng", command=ham_them_nguoi_dung).grid(row=5, column=1, pady=10)
    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_nguoi).grid(row=5, column=0, pady=10)


def form_xoa_nguoi():
    Destroy()
    conn = Doc_diachisql()
    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    global lst_Nguoi, lst_0, lst_1, lst_2
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
    Label(frame_main, text="Nhập căn cước người muốn xóa:", font=("Segoe UI", 14)).grid(row=0, column=0, columnspan=2, padx=5, pady=5, sticky="nsew")
    Label(frame_main, text="CCCD:", font=("Segoe UI", 11)).grid(row=1, column=0, padx=5, pady=5, sticky="e")
    cccd_var = StringVar()
    Entry(frame_main, font=("Segoe UI", 11), textvariable=cccd_var).grid(row=1, column=1, padx=5, pady=5)


    def ham_xoa_nguoi_dung():
        cccd_nhap = cccd_var.get().strip()
        if not cccd_nhap:
            messagebox.showerror("Lỗi", "Vui lòng nhập CCCD!")
            return
        cursor = conn.cursor()
        cursor.execute("""
            SELECT ho_ten, sodu0, sodu1, sodu2 FROM nguoi WHERE cccd = %s
        """, (cccd_nhap,))
        row = cursor.fetchone()
        if row is None:
            messagebox.showerror("Không tìm thấy", f"Không có người dùng nào với CCCD: {cccd_nhap}")
            cccd_var.set("")
            cursor.close()
            conn.close()
            return
        ho_ten, sodu0, sodu1, sodu2 = row
        if all(sodu == 0 for sodu in (sodu0, sodu1, sodu2)):
            result = Xoa_nguoi(conn, cccd_nhap)
            if result == 0:
                lst_Nguoi[:] = [nguoi for nguoi in lst_Nguoi if nguoi.cccd != cccd_nhap]
                lst_0[:] = [gd for gd in lst_0 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                lst_1[:] = [gd for gd in lst_1 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                lst_2[:] = [gd for gd in lst_2 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                messagebox.showinfo("Thành công", f"Đã xóa người dùng: {ho_ten}")
            else:
                messagebox.showerror("Lỗi", "Không thể xóa người dùng")
        else:
            sodu_text = f"Số dư: 0%: {sodu0} VNĐ, 1%: {sodu1} VNĐ, 2%: {sodu2} VNĐ"
            confirm = messagebox.askyesno(
                "Xác nhận xóa",
                f"Người dùng {ho_ten} vẫn còn số dư:\n{sodu_text}\nBạn có muốn xóa và cập nhật số dư liên quan không?"
            )
            if confirm:
                result = Xoa_nguoi(conn, cccd_nhap)
                if result == 0:
                    lst_Nguoi[:] = [nguoi for nguoi in lst_Nguoi if nguoi.cccd != cccd_nhap]
                    lst_0[:] = [gd for gd in lst_0 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                    lst_1[:] = [gd for gd in lst_1 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                    lst_2[:] = [gd for gd in lst_2 if gd.nguoi_chovay.cccd != cccd_nhap and gd.nguoi_no.cccd != cccd_nhap]
                    messagebox.showinfo("Thành công", f"Đã xóa người dùng: {ho_ten} và cập nhật số dư liên quan")
                else:
                    messagebox.showerror("Lỗi", "Không thể xóa người dùng")
            else:
                messagebox.showinfo("Hủy", "Đã hủy xóa người dùng")
        cccd_var.set("")
        cursor.close()
        conn.close()


    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_nguoi).grid(row=5, column=0, pady=10)
    Button(frame_main, font=("Segoe UI", 11), text="Xóa người dùng", command=ham_xoa_nguoi_dung).grid(row=5, column=1, pady=10)




def form_tra_cuu_thong_tin():
    Destroy()
    conn = Doc_diachisql()
    
    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=10)
        return
    
    try:
        global lst_Nguoi, lst_0, lst_1, lst_2
        lst_Nguoi = Doc_nguoi_tu_mysql(conn)
        lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
        
        if not lst_Nguoi:
            Label(frame_main, text="(Chưa có người dùng nào)", font=("Segoe UI", 12), fg="gray").pack(pady=10)
            messagebox.showinfo("Thông báo", "Hiện không có người dùng nào để tra cứu.")
            return
        
        # Tạo Frame con để chứa nội dung của form này
        content_frame = Frame(frame_main)
        content_frame.pack(fill="both", expand=True, padx=10, pady=10)
        
        # Cấu hình content_frame để mở rộng
        content_frame.grid_rowconfigure(3, weight=1)
        content_frame.grid_columnconfigure(0, weight=1)
        
        # Label và Entry nhập CCCD
        Label(content_frame, font=("Segoe UI", 13), text="Hãy nhập số căn cước công dân của người bạn muốn tra cứu:").grid(row=0, column=0, padx=10, pady=(0, 5), sticky="w")
        cccd_var = StringVar()
        Entry(content_frame, font=("Segoe UI", 13), textvariable=cccd_var, width=30).grid(row=1, column=0, padx=10, pady=5, sticky="w")
        
        # Tạo frame chứa kết quả tra cứu với thanh cuộn
        result_frame = Frame(content_frame)
        result_frame.grid(row=3, column=0, padx=0, pady=5, sticky="nsew")
        result_frame.grid_rowconfigure(0, weight=1)
        result_frame.grid_columnconfigure(0, weight=1)
        
        canvas = Canvas(result_frame)
        scrollbar = Scrollbar(result_frame, orient="vertical", command=canvas.yview)
        scrollable_frame = Frame(canvas)
        
        scrollable_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
        )
        
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        def resize_scroll_frame(event):
            canvas.itemconfig(canvas.create_window((0, 0), window=scrollable_frame, anchor="nw"), width=event.width)
        
        canvas.bind("<Configure>", resize_scroll_frame)


        def ham_tra_cuu_thong_tin():
            # Xóa nội dung cũ trong scrollable_frame
            for widget in scrollable_frame.winfo_children():
                widget.destroy()
            
            cccd_nhap = cccd_var.get().strip()
            nguoi_tim_thay = None
            for nguoi in lst_Nguoi:
                if str(nguoi.cccd).strip() == cccd_nhap:
                    nguoi_tim_thay = nguoi
                    break
            
            if nguoi_tim_thay is None:
                Label(scrollable_frame, text=f"Không có người dùng nào với CCCD: {cccd_nhap}", font=("Segoe UI", 11), fg="red", wraplength=600).pack(anchor="w", padx=15, pady=10)
                cccd_var.set("")
                return
            
            # Hiển thị thông tin cơ bản của người dùng (không hiển thị số dư)
            Label(scrollable_frame, text="Thông tin người dùng:", font=("Segoe UI", 13, "bold")).pack(anchor="w", padx=15, pady=(10, 5))
            Label(scrollable_frame, text=f"Họ tên: {nguoi_tim_thay.ho_ten}", font=("Segoe UI", 11), wraplength=600).pack(anchor="w", padx=25, pady=2)
            Label(scrollable_frame, text=f"CCCD: {nguoi_tim_thay.cccd}", font=("Segoe UI", 11), wraplength=600).pack(anchor="w", padx=25, pady=2)
            Label(scrollable_frame, text=f"Tuổi: {nguoi_tim_thay.tuoi}", font=("Segoe UI", 11), wraplength=600).pack(anchor="w", padx=25, pady=2)
            Label(scrollable_frame, text=f"SĐT: {nguoi_tim_thay.so_dienthoai}", font=("Segoe UI", 11), wraplength=600).pack(anchor="w", padx=25, pady=2)
            
            # Tìm và hiển thị các giao dịch liên quan
            Label(scrollable_frame, text="Các khoản cho vay:", font=("Segoe UI", 12, "bold")).pack(anchor="w", padx=15, pady=(15, 5))
            co_giao_dich = False
            all_giaodich = lst_0 + lst_1 + lst_2
            for gd in all_giaodich:
                if gd.nguoi_chovay.cccd == cccd_nhap:
                    co_giao_dich = True
                    Label(scrollable_frame, text=f"- Người nợ: {gd.nguoi_no.ho_ten}, Số tiền: {gd.sotien} VNĐ, Lãi suất: {gd.laisuat}%/tháng", font=("Segoe UI", 11), wraplength=550).pack(anchor="w", padx=35, pady=2)
                    if gd.ghichu:
                        Label(scrollable_frame, text=f"  Ghi chú: {gd.ghichu}", font=("Segoe UI", 11, "italic"), wraplength=500).pack(anchor="w", padx=45, pady=1)
            
            if not co_giao_dich:
                Label(scrollable_frame, text="Không có khoản cho vay.", font=("Segoe UI", 11, "italic"), wraplength=600).pack(anchor="w", padx=35, pady=5)
            
            Label(scrollable_frame, text="Các khoản nợ:", font=("Segoe UI", 12, "bold")).pack(anchor="w", padx=15, pady=(15, 5))
            co_giao_dich = False
            for gd in all_giaodich:
                if gd.nguoi_no.cccd == cccd_nhap:
                    co_giao_dich = True
                    Label(scrollable_frame, text=f"- Người cho vay: {gd.nguoi_chovay.ho_ten}, Số tiền: {gd.sotien} VNĐ, Lãi suất: {gd.laisuat}%/tháng", font=("Segoe UI", 11), wraplength=550).pack(anchor="w", padx=35, pady=2)
                    if gd.ghichu:
                        Label(scrollable_frame, text=f"  Ghi chú: {gd.ghichu}", font=("Segoe UI", 11, "italic"), wraplength=500).pack(anchor="w", padx=45, pady=1)
            
            if not co_giao_dich:
                Label(scrollable_frame, text="Không có khoản nợ.", font=("Segoe UI", 11, "italic"), wraplength=600).pack(anchor="w", padx=35, pady=5)
            
            cccd_var.set("")


        # Nút Tra cứu và Trở về
        button_frame = Frame(content_frame)
        button_frame.grid(row=2, column=0, pady=10)
        Button(button_frame, text="Tra cứu", font=("Segoe UI", 11), command=ham_tra_cuu_thong_tin, width=10).pack(side="left", padx=5)
        Button(button_frame, text="Trở về", font=("Segoe UI", 11), command=form_menu_quan_ly_nguoi, width=10).pack(side="left", padx=5)


    finally:
        if conn.is_connected():
            conn.close()


def form_in_danh_sach_ng_dung():
    Destroy()
    conn = Doc_diachisql()
    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    global lst_Nguoi
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    conn.close()
    Label(frame_main, text="Danh sách người dùng:", font=("Segoe UI", 14, "bold")).pack(pady=10)
    
    if not lst_Nguoi:
        Label(frame_main, text="(Chưa có người dùng nào)", font=("Segoe UI", 12), fg="gray").pack(pady=5)
        return
    
    canvas = Canvas(frame_main)
    scrollbar = Scrollbar(frame_main, orient="vertical", command=canvas.yview)
    canvas.configure(yscrollcommand=scrollbar.set)
    canvas.pack(side=LEFT, fill=BOTH, expand=True)
    scrollbar.pack(side=RIGHT, fill=Y)
    scroll_frame = Frame(canvas)
    window = canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
    scroll_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
    
    def resize_scroll_frame(event):
        canvas.itemconfig(window, width=event.width)
    
    canvas.bind("<Configure>", resize_scroll_frame)
    
    for i, nguoi in enumerate(lst_Nguoi):
        Label(scroll_frame,
              text=f"{i}. Họ tên: {nguoi.ho_ten}, CCCD: {nguoi.cccd}, SĐT: {nguoi.so_dienthoai}, Tuổi: {nguoi.tuoi}",
              font=("Segoe UI", 11), anchor="w", justify=LEFT).pack(anchor="w", padx=10, pady=3)
    
    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_nguoi).pack(pady=10)


# Hàm này tạo ra để hiển thị danh sách người dùng => thuận tiện cho việc thêm giao dịch
def form_show_danh_sach_nguoi_toplevel():
    """
    Hiển thị danh sách người dùng trong một cửa sổ Toplevel mới.
    Tái sử dụng logic hiển thị từ form_in_danh_sach_ng_dung().
    """
    # Tạo cửa sổ Toplevel
    danh_sach_window = Toplevel(root)
    danh_sach_window.title("Danh sách người dùng")
    danh_sach_window.geometry("600x400")
    


    # Kết nối MySQL để lấy danh sách người dùng
    conn = Doc_diachisql()
    if conn == 1:
        Label(danh_sach_window, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    
    global lst_Nguoi
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    conn.close()


    # Tiêu đề
    Label(danh_sach_window, text="Danh sách người dùng:", font=("Segoe UI", 14, "bold")).pack(pady=10)
    
    if not lst_Nguoi:
        Label(danh_sach_window, text="(Chưa có người dùng nào)", font=("Segoe UI", 12), fg="gray").pack(pady=5)
        Button(danh_sach_window, font=("Segoe UI", 11), text="Đóng", command=danh_sach_window.destroy).pack(pady=10)
        return
    
    # Tạo Canvas và Scrollbar để hiển thị danh sách
    canvas = Canvas(danh_sach_window)
    scrollbar = Scrollbar(danh_sach_window, orient="vertical", command=canvas.yview)
    canvas.configure(yscrollcommand=scrollbar.set)
    canvas.pack(side=LEFT, fill=BOTH, expand=True)
    scrollbar.pack(side=RIGHT, fill=Y)
    
    scroll_frame = Frame(canvas)
    window = canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
    scroll_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
    
    def resize_scroll_frame(event):
        canvas.itemconfig(window, width=event.width)
    
    canvas.bind("<Configure>", resize_scroll_frame)
    
    # Hiển thị danh sách người dùng
    for i, nguoi in enumerate(lst_Nguoi):
        Label(scroll_frame,
              text=f"{i}. Họ tên: {nguoi.ho_ten}, CCCD: {nguoi.cccd}, SĐT: {nguoi.so_dienthoai}, Tuổi: {nguoi.tuoi}",
              font=("Segoe UI", 11), anchor="w", justify=LEFT).pack(anchor="w", padx=10, pady=3)
    
    # Nút đóng cửa sổ
    Button(danh_sach_window, font=("Segoe UI", 11), text="Đóng", command=danh_sach_window.destroy).pack(pady=10)


def form_them_giao_dich():
    Destroy()  
    conn = Doc_diachisql()
    form_show_danh_sach_nguoi_toplevel() # Hiển thị danh sách người dùng thuận tiện cho việc thêm giao dịch


    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    
    global lst_Nguoi, lst_0, lst_1, lst_2
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
    
    Label(frame_main, text="Nhập thông tin giao dịch (nhập STT trong danh sách người dùng)", font=("Segoe UI", 16, "bold")).grid(row=0, column=0, columnspan=4, pady=10)
    Label(frame_main, text="Người cho vay", font=("Segoe UI", 12)).grid(row=1, column=0, sticky=E)
    entry_nguoi_chovay = Entry(frame_main, font=("Segoe UI", 11))
    entry_nguoi_chovay.grid(row=1, column=1)
    Label(frame_main, text="Người nợ", font=("Segoe UI", 12)).grid(row=2, column=0, sticky=E)
    entry_nguoi_no = Entry(frame_main, font=("Segoe UI", 11))
    entry_nguoi_no.grid(row=2, column=1)
    Label(frame_main, text="Số tiền", font=("Segoe UI", 12)).grid(row=3, column=0, sticky=E)
    entry_sotien = Entry(frame_main, font=("Segoe UI", 11))
    entry_sotien.grid(row=3, column=1)
    Label(frame_main, text="Lãi suất (%)", font=("Segoe UI", 12)).grid(row=4, column=0, sticky=E)
    entry_laisuat = Entry(frame_main, font=("Segoe UI", 11))
    entry_laisuat.grid(row=4, column=1)
    Label(frame_main, text="Ngày cho vay (YYYY/MM/DD)", font=("Segoe UI", 12)).grid(row=5, column=0, sticky=E)
    entry_ngaychovay = Entry(frame_main, font=("Segoe UI", 11))
    entry_ngaychovay.grid(row=5, column=1)
    Label(frame_main, text="Ngày cập nhật cuối (có thể bỏ qua)", font=("Segoe UI", 12)).grid(row=6, column=0, sticky=E)
    entry_lancuoi = Entry(frame_main, font=("Segoe UI", 11))
    entry_lancuoi.grid(row=6, column=1)
    Label(frame_main, text="Ghi chú (nếu có)", font=("Segoe UI", 12)).grid(row=7, column=0, sticky=E)
    entry_ghichu = Entry(frame_main, font=("Segoe UI", 11))
    entry_ghichu.grid(row=7, column=1)
    
    def thuc_hien_them_giao_dich():
        try:
            nguoi_chovay_idx = int(entry_nguoi_chovay.get())
            nguoi_no_idx = int(entry_nguoi_no.get())
            if nguoi_chovay_idx == nguoi_no_idx:
                messagebox.showerror("Lỗi", "Người cho vay và người nợ không được trùng!")
                return
            if nguoi_chovay_idx < 0 or nguoi_chovay_idx >= len(lst_Nguoi) or nguoi_no_idx < 0 or nguoi_no_idx >= len(lst_Nguoi):
                messagebox.showerror("Lỗi", "STT người không hợp lệ!")
                return
            nguoi_chovay = lst_Nguoi[nguoi_chovay_idx]
            nguoi_no = lst_Nguoi[nguoi_no_idx]
            sotien = int(entry_sotien.get())
            if sotien <= 0:
                messagebox.showerror("Lỗi", "Số tiền phải lớn hơn 0!")
                return
            laisuat = int(entry_laisuat.get())
            if laisuat not in (0, 1, 2):
                messagebox.showerror("Lỗi", "Lãi suất phải là 0, 1 hoặc 2!")
                return
            ngay_chovay = entry_ngaychovay.get().strip()
            lancuoi = entry_lancuoi.get().strip()
            ghichu = entry_ghichu.get().strip()
            try:
                if ngay_chovay:
                    y, m, d = map(int, ngay_chovay.split("/"))
                    ngay_chovay = date(y, m, d)
                else:
                    messagebox.showwarning("Thiếu ngày", "Vui lòng nhập ngày cho vay!")
                    return
                lancuoi = date(*map(int, lancuoi.split("/"))) if lancuoi else None
            except ValueError:
                messagebox.showerror("Lỗi", "Định dạng ngày không hợp lệ (YYYY/MM/DD)!")
                return
            gd = GiaoDich(nguoi_chovay, nguoi_no, sotien, laisuat, ngay_chovay, lancuoi, ghichu)
            Xu_ly(lst_0, lst_1, lst_2, gd)  # Process transaction with algorithm logic
            Xoa_toanbo_giaodich(conn)  # Clear database
            Ghi_giaodich_vaomysql(lst_0, lst_1, lst_2, conn)  # Write updated transactions
            conn.commit()
            messagebox.showinfo("Thành công", "Đã thêm giao dịch thành công!")
            # Clear input fields
            entry_nguoi_chovay.delete(0, END)
            entry_nguoi_no.delete(0, END)
            entry_sotien.delete(0, END)
            entry_laisuat.delete(0, END)
            entry_ngaychovay.delete(0, END)
            entry_lancuoi.delete(0, END)
            entry_ghichu.delete(0, END)
            # Re-optimize and display transactions
            form_toi_uu_giao_dich()  # Re-run global optimization
            form_in_giao_dich()  # Display updated transactions
        except ValueError as e:
            messagebox.showerror("Lỗi", f"Dữ liệu không hợp lệ: {e}")
        except Exception as e:
            messagebox.showerror("Lỗi", f"Lỗi khi thêm giao dịch: {e}")
            conn.rollback()
        finally:
            if conn.is_connected():
                conn.close()                


    Button(frame_main, text="Thêm giao dịch", font=("Segoe UI", 11), command=thuc_hien_them_giao_dich).grid(row=8, column=1, pady=10)
    Button(frame_main, text="Trở về", font=("Segoe UI", 11), command=form_menu_quan_ly_giao_dich).grid(row=8, column=0, pady=10)


def form_xoa_giao_dich():
    Destroy()
    conn = Doc_diachisql()
    
    form_show_danh_sach_nguoi_toplevel() # Hiển thị danh sách người dùng thuận tiện cho việc thêm giao dịch


    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    global lst_Nguoi, lst_0, lst_1, lst_2
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
    Label(frame_main, text="Nhập thông tin giao dịch muốn xóa:", font=("Segoe UI", 14)).grid(row=0, column=0, columnspan=2, padx=5, pady=5, sticky="nsew")
    Label(frame_main, text="CCCD người cho vay:", font=("Segoe UI", 11)).grid(row=1, column=0, padx=5, pady=5, sticky="e")
    Label(frame_main, text="CCCD người nợ:", font=("Segoe UI", 11)).grid(row=2, column=0, padx=5, pady=5, sticky="e")
    Label(frame_main, text="Lãi suất (0/1/2):", font=("Segoe UI", 11)).grid(row=3, column=0, padx=5, pady=5, sticky="e")
    
    chovay_var = StringVar()
    no_var = StringVar()
    laisuat_var = StringVar()
    
    Entry(frame_main, font=("Segoe UI", 11), textvariable=chovay_var).grid(row=1, column=1, padx=5, pady=5)
    Entry(frame_main, font=("Segoe UI", 11), textvariable=no_var).grid(row=2, column=1, padx=5, pady=5)
    Entry(frame_main, font=("Segoe UI", 11), textvariable=laisuat_var).grid(row=3, column=1, padx=5, pady=5)


    def ham_xoa_giao_dich():
        chovay_cccd = chovay_var.get().strip()
        no_cccd = no_var.get().strip()
        try:
            laisuat = int(laisuat_var.get().strip())
            if laisuat not in (0, 1, 2):
                raise ValueError("Lãi suất phải là 0, 1 hoặc 2")
        except ValueError as e:
            messagebox.showerror("Lỗi", f"Lãi suất không hợp lệ: {e}")
            return
        if not chovay_cccd or not no_cccd:
            messagebox.showerror("Lỗi", "Vui lòng nhập đầy đủ CCCD người cho vay và người nợ!")
            return
        result = Xoa_Giaodich_Capnhat_Sodu(conn, chovay_cccd, no_cccd, laisuat)
        if result == 0:
            if laisuat == 0:
                lst_0[:] = [gd for gd in lst_0 if not (gd.nguoi_chovay.cccd == chovay_cccd and gd.nguoi_no.cccd == no_cccd)]
            elif laisuat == 1:
                lst_1[:] = [gd for gd in lst_1 if not (gd.nguoi_chovay.cccd == chovay_cccd and gd.nguoi_no.cccd == no_cccd)]
            elif laisuat == 2:
                lst_2[:] = [gd for gd in lst_2 if not (gd.nguoi_chovay.cccd == chovay_cccd and gd.nguoi_no.cccd == no_cccd)]
            messagebox.showinfo("Thành công", "Đã xóa giao dịch và cập nhật số dư thành công!")
        else:
            messagebox.showerror("Lỗi", "Không tìm thấy giao dịch để xóa!")
        chovay_var.set("")
        no_var.set("")
        laisuat_var.set("")
        conn.close()


    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_giao_dich).grid(row=5, column=0, pady=10)
    Button(frame_main, font=("Segoe UI", 11), text="Xóa giao dịch", command=ham_xoa_giao_dich).grid(row=5, column=1, pady=10)


def form_in_giao_dich():
    Destroy()
    conn = Doc_diachisql()
    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    
    global lst_Nguoi, lst_0, lst_1, lst_2
    try:
        lst_Nguoi = Doc_nguoi_tu_mysql(conn)
        lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
        
        Label(frame_main, text="DANH SÁCH GIAO DỊCH", font=("Segoe UI", 16, "bold")).pack(pady=10)
        canvas = Canvas(frame_main)
        scrollbar = Scrollbar(frame_main, orient=VERTICAL, command=canvas.yview)
        scroll_frame = Frame(canvas)
        scroll_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(
                scrollregion=canvas.bbox("all")
            )
        )
        canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.pack(side=LEFT, fill=BOTH, expand=True)
        scrollbar.pack(side=RIGHT, fill=Y)


        def hien_thi_giaodich(giaodich, frame):
            Label(frame, text=f"Người cho vay: {giaodich.nguoi_chovay.ho_ten}", font=("Segoe UI", 11, "bold")).pack(anchor="w", padx=10)
            Label(frame, text=f"Người nợ: {giaodich.nguoi_no.ho_ten}", font=("Segoe UI", 11)).pack(anchor="w", padx=20)
            Label(frame, text=f"Số tiền: {giaodich.sotien} VNĐ | Lãi suất: {giaodich.laisuat}%/tháng", font=("Segoe UI", 11)).pack(anchor="w", padx=20)
            if giaodich.ghichu:
                Label(frame, text=f"Ghi chú: {giaodich.ghichu}", font=("Segoe UI", 11, "italic")).pack(anchor="w", padx=20)
            Label(frame, text="-" * 60).pack(pady=5)


        if not (lst_0 or lst_1 or lst_2):
            Label(scroll_frame, text="Không có giao dịch nào.", font=("Segoe UI", 11, "italic")).pack(pady=10)
            messagebox.showinfo("Thông báo", "Hiện không có giao dịch nào để hiển thị.")
        else:
            for gd in lst_0 + lst_1 + lst_2:
                hien_thi_giaodich(gd, scroll_frame)
    except Exception as e:
        messagebox.showerror("Lỗi", f"Lỗi khi hiển thị giao dịch: {e}")
        conn.rollback()
    finally:
        if conn.is_connected():
            conn.close()
    
    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_giao_dich).pack(pady=10)


def form_toi_uu_giao_dich():
    Destroy()
    conn = Doc_diachisql()
    if conn == 1:
        Label(frame_main, text="Không thể kết nối đến MySQL", font=("Segoe UI", 12), fg="red").pack(pady=5)
        return
    
    global lst_Nguoi, lst_0, lst_1, lst_2
    lst_Nguoi = Doc_nguoi_tu_mysql(conn)
    lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
    
    try:
        Toi_uu_toan_bo(conn)
        lst_Nguoi = Doc_nguoi_tu_mysql(conn)
        lst_0, lst_1, lst_2 = Doc_giaodich_tu_db(conn, lst_Nguoi)
        conn.commit()
        messagebox.showinfo("Thành công", "Đã tối ưu giao dịch thành công!")
    except Exception as e:
        messagebox.showerror("Lỗi", f"Lỗi khi tối ưu giao dịch: {e}")
        conn.rollback()
    finally:
        if conn.is_connected():
            conn.close()
    
    Button(frame_main, font=("Segoe UI", 11), text="Trở về", command=form_menu_quan_ly_giao_dich).pack(pady=10)


def form_menu_quan_ly_nguoi():
    Destroy()
    Button(frame_main, font=("Segoe UI", 11), text="1. Thêm người", command=form_them_nguoi).grid(row=0, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="2. Xoá người", command=form_xoa_nguoi).grid(row=1, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="3. Tra cứu thông tin", command=form_tra_cuu_thong_tin).grid(row=2, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="4. In danh sách người dùng", command=form_in_danh_sach_ng_dung).grid(row=3, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="5. Trở về MENU", command=form_Menu_chinh).grid(row=4, column=0, padx=5, pady=5, sticky=W)


def form_menu_quan_ly_giao_dich():
    Destroy()
    Button(frame_main, font=("Segoe UI", 11), text="1. Thêm giao dịch", command=form_them_giao_dich).grid(row=0, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="2. Xoá giao dịch", command=form_xoa_giao_dich).grid(row=1, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="3. Hiển thị toàn bộ giao dịch sau khi tối ưu (Chọn chức năng tối ưu trước!)", command=form_in_giao_dich).grid(row=2, column=0, padx=5, pady=5, sticky=W)
    Button(frame_main, font=("Segoe UI", 11), text="4. Trở về MENU", command=form_Menu_chinh).grid(row=3, column=0, padx=5, pady=5, sticky=W)


def form_Menu_chinh():
    Destroy()
    Label(frame_main, text="Chào mừng bạn đến với chương trình \n Tối ưu dòng tiền", font=("Segoe UI", 16)).pack(pady=10)


# === Cài đặt cửa sổ ===
def Window():
    window_width = 960
    window_height = 540
    screen_width = root.winfo_screenwidth()
    screen_height = root.winfo_screenheight()
    x = int((screen_width - window_width) / 2)
    y = int((screen_height - window_height) / 2)
    root.geometry(f"{window_width}x{window_height}+{x}+{y}")
    root.title("Quản lý dòng tiền")
Window()


frame_menu = Frame(root, bg="lightgray", width=200)
frame_menu.pack(side=LEFT, fill=Y)
frame_main = Frame(root)
frame_main.pack(side=LEFT, fill=BOTH, expand=True)


Button(frame_menu, text="MENU", command=form_Menu_chinh, font=("Segoe UI", 11)).pack(pady=5, fill=X)
Button(frame_menu, text="1. Quản lý người", command=form_menu_quan_ly_nguoi, font=("Segoe UI", 11)).pack(pady=5, fill=X)
Button(frame_menu, text="2. Quản lý giao dịch", command=form_menu_quan_ly_giao_dich, font=("Segoe UI", 11)).pack(pady=5, fill=X)
Button(frame_menu, text="3. Tối ưu giao dịch", command=form_toi_uu_giao_dich, font=("Segoe UI", 11)).pack(pady=5, fill=X)
Button(frame_menu, text="4. Kết nối với SQL", command=form_ket_noi_sql, font=("Segoe UI", 11)).pack(pady=5, fill=X)
Button(frame_menu, text="5. Thoát chương trình", command=root.quit, font=("Segoe UI", 11)).pack(pady=5, fill=X)


form_Menu_chinh()
root.mainloop()



