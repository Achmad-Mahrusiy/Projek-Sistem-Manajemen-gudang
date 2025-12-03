import psycopg2
from psycopg2 import Error
import os
from datetime import datetime, timedelta
import sys

# =====================================================
# DATABASE CONNECTION
# =====================================================

def get_db_connection():
    """Membuat koneksi ke database PostgreSQL"""
    try:
        conn = psycopg2.connect(
            host="localhost",
            database="Projek_kelompok 3",
            user="postgres",
            password="1234"
        )
        return conn
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def close_connection(conn, cursor=None):
    """Menutup koneksi database"""
    if cursor:
        cursor.close()
    if conn:
        conn.close()

# =====================================================
# UTILITY FUNCTIONS
# =====================================================

def clear_screen():
    """Membersihkan layar terminal"""
    os.system('cls' if os.name == 'nt' else 'clear')

def pause():
    """Menunggu input user sebelum melanjutkan"""
    input("\nTekan Enter untuk melanjutkan...")

def print_header(title):
    """Menampilkan header"""
    clear_screen()
    print("=" * 80)
    print(f"{title.center(80)}")
    print("=" * 80)
    print()

def format_currency(amount):
    """Format mata uang"""
    return f"Rp {amount:,.2f}"

def validate_float_input(prompt):
    """Validasi input float"""
    while True:
        try:
            value = float(input(prompt).strip())
            if value < 0:
                print("Nilai tidak boleh negatif!")
                continue
            return value
        except ValueError:
            print("Input harus berupa angka!")

def validate_int_input(prompt):
    """Validasi input integer"""
    while True:
        try:
            value = int(input(prompt).strip())
            if value < 0:
                print("Nilai tidak boleh negatif!")
                continue
            return value
        except ValueError:
            print("Input harus berupa bilangan bulat!")

# =====================================================
# CATALOG SYSTEM
# =====================================================

class CatalogSystem:
    def __init__(self):
        pass
    
    def view_product_catalog(self, filter_tersedia=True):
        """Menampilkan katalog produk dengan status stok"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return
        
        try:
            with conn.cursor() as cur:
                if filter_tersedia:
                    query = """
                        SELECT 
                            k.id_katalog,
                            p.nama_produk,
                            p.kategori,
                            p.satuan,
                            k.stok_tersedia,
                            k.harga_jual,
                            k.status,
                            CASE 
                                WHEN k.stok_tersedia = 0 THEN 'HABIS'
                                WHEN k.stok_tersedia <= 5 THEN 'MENIPIS'
                                ELSE 'TERSEDIA'
                            END as status_stok
                        FROM katalog k
                        JOIN produk p ON k.id_produk = p.id_produk
                        WHERE p.status = 'aktif' AND k.stok_tersedia > 0
                        ORDER BY p.kategori, p.nama_produk
                    """
                else:
                    query = """
                        SELECT 
                            k.id_katalog,
                            p.nama_produk,
                            p.kategori,
                            p.satuan,
                            k.stok_tersedia,
                            k.harga_jual,
                            k.status,
                            CASE 
                                WHEN k.stok_tersedia = 0 THEN 'HABIS'
                                WHEN k.stok_tersedia <= 5 THEN 'MENIPIS'
                                ELSE 'TERSEDIA'
                            END as status_stok
                        FROM katalog k
                        JOIN produk p ON k.id_produk = p.id_produk
                        WHERE p.status = 'aktif'
                        ORDER BY p.kategori, p.nama_produk
                    """
                
                cur.execute(query)
                catalog = cur.fetchall()
                return catalog
                
        except Error as e:
            print(f"Error mengambil data katalog: {e}")
            return None
        finally:
            conn.close()
    
    def display_catalog(self, filter_tersedia=True):
        """Menampilkan katalog produk dalam format tabel"""
        catalog = self.view_product_catalog(filter_tersedia)
        
        if not catalog:
            print("Tidak ada data katalog yang ditemukan!")
            return
        
        print_header("KATALOG PRODUK TOKO")
        
        total_produk = len(catalog)
        produk_tersedia = len([p for p in catalog if p[4] > 0])
        produk_menipis = len([p for p in catalog if p[4] <= 5 and p[4] > 0])
        produk_habis = len([p for p in catalog if p[4] == 0])
        
        print(f"STATISTIK KATALOG:")
        print(f"  Total Produk    : {total_produk}")
        print(f"  Tersedia        : {produk_tersedia}")
        print(f"  Stok Menipis    : {produk_menipis}")
        print(f"  Stok Habis      : {produk_habis}")
        print()
        
        categories = {}
        for item in catalog:
            kategori = item[2]
            if kategori not in categories:
                categories[kategori] = []
            categories[kategori].append(item)
        
        for kategori, items in categories.items():
            print(f"\n{'='*80}")
            print(f"KATEGORI: {kategori.upper()}")
            print(f"{'='*80}")
            print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12} {'Status':<15}")
            print(f"{'-'*80}")
            
            for item in items:
                id_katalog, nama_produk, _, satuan, stok, harga, _, status_stok = item
                print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_currency(harga):<12} {status_stok:<15}")
        
        print(f"\n{'='*80}")
        return catalog
    
    def view_stock_summary(self):
        """Menampilkan ringkasan stok"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        p.kategori,
                        COUNT(*) as total_produk,
                        SUM(k.stok_tersedia) as total_stok,
                        COUNT(CASE WHEN k.stok_tersedia = 0 THEN 1 END) as habis,
                        COUNT(CASE WHEN k.stok_tersedia <= 5 AND k.stok_tersedia > 0 THEN 1 END) as menipis,
                        SUM(k.stok_tersedia * k.harga_jual) as nilai_stok
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE p.status = 'aktif'
                    GROUP BY p.kategori
                    ORDER BY p.kategori
                """
                cur.execute(query)
                stock_summary = cur.fetchall()
                
                query_low_stock = """
                    SELECT 
                        p.nama_produk,
                        k.stok_tersedia,
                        p.satuan,
                        k.harga_jual
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia <= 5 AND k.stok_tersedia > 0
                    AND p.status = 'aktif'
                    ORDER BY k.stok_tersedia ASC
                    LIMIT 10
                """
                cur.execute(query_low_stock)
                low_stock = cur.fetchall()
                
                query_out_of_stock = """
                    SELECT 
                        p.nama_produk,
                        p.satuan,
                        k.harga_jual
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia = 0
                    AND p.status = 'aktif'
                    ORDER BY p.nama_produk
                    LIMIT 10
                """
                cur.execute(query_out_of_stock)
                out_of_stock = cur.fetchall()
                
                return stock_summary, low_stock, out_of_stock
                
        except Error as e:
            print(f"Error mengambil ringkasan stok: {e}")
            return None, None, None
        finally:
            conn.close()
    
    def display_stock_summary(self):
        """Menampilkan ringkasan stok"""
        stock_summary, low_stock, out_of_stock = self.view_stock_summary()
        
        print_header("RINGKASAN STOK TOKO")
        
        if not stock_summary:
            print("Tidak ada data stok yang ditemukan!")
            return
        
        print("RINGKASAN STOK PER KATEGORI:")
        print(f"{'='*80}")
        print(f"{'Kategori':<15} {'Total Produk':<12} {'Total Stok':<12} {'Habis':<8} {'Menipis':<10} {'Nilai Stok':<15}")
        print(f"{'-'*80}")
        
        total_nilai_stok = 0
        for kategori in stock_summary:
            kategori_name, total_produk, total_stok, habis, menipis, nilai_stok = kategori
            total_nilai_stok += nilai_stok if nilai_stok else 0
            print(f"{kategori_name:<15} {total_produk:<12} {total_stok:<12.2f} {habis:<8} {menipis:<10} {format_currency(nilai_stok) if nilai_stok else 'Rp 0':<15}")
        
        print(f"{'='*80}")
        print(f"TOTAL NILAI STOK: {format_currency(total_nilai_stok)}")
        print()
        
        print("PRODUK DENGAN STOK MENIPIS (<= 5):")
        if low_stock:
            print(f"{'-'*60}")
            print(f"{'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12}")
            print(f"{'-'*60}")
            for product in low_stock:
                print(f"{product[0]:<20} {product[1]:<10} {product[2]:<8} {format_currency(product[3]):<12}")
        else:
            print("  Tidak ada produk dengan stok menipis")
        print()
        
        print("PRODUK HABIS:")
        if out_of_stock:
            print(f"{'-'*50}")
            print(f"{'Nama Produk':<20} {'Satuan':<8} {'Harga':<12}")
            print(f"{'-'*50}")
            for product in out_of_stock:
                print(f"{product[0]:<20} {product[1]:<8} {format_currency(product[2]):<12}")
        else:
            print("  Tidak ada produk yang habis")
        
        print(f"\n{'='*80}")

# =====================================================
# PRODUCT MANAGEMENT SYSTEM
# =====================================================

class ProductManagement:
    def __init__(self):
        pass
    
    def view_all_products(self):
        """Melihat semua produk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        id_produk, nama_produk, kategori, satuan, status,
                        created_at, updated_at
                    FROM produk 
                    ORDER BY id_produk
                """
                cur.execute(query)
                products = cur.fetchall()
                return products
                
        except Error as e:
            print(f"Error mengambil data produk: {e}")
            return None
        finally:
            conn.close()
    
    def display_products(self):
        """Menampilkan daftar produk"""
        products = self.view_all_products()
        
        if not products:
            print("Tidak ada data produk yang ditemukan!")
            return
        
        print_header("DAFTAR SEMUA PRODUK")
        
        print(f"{'ID':<5} {'Nama Produk':<20} {'Kategori':<10} {'Satuan':<8} {'Status':<10}")
        print(f"{'-'*60}")
        
        for product in products:
            id_produk, nama_produk, kategori, satuan, status, created_at, updated_at = product
            print(f"{id_produk:<5} {nama_produk:<20} {kategori:<10} {satuan:<8} {status:<10}")
        
        print(f"{'-'*60}")
        print(f"Total produk: {len(products)}")
        return products
    
    def add_product(self, nama_produk, kategori, satuan):
        """Menambah produk baru"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT id_produk FROM produk WHERE nama_produk = %s", (nama_produk,))
                existing = cur.fetchone()
                
                if existing:
                    print(f"Produk '{nama_produk}' sudah ada dalam database!")
                    return False
                
                cur.execute("""
                    INSERT INTO produk (nama_produk, kategori, satuan)
                    VALUES (%s, %s, %s)
                """, (nama_produk, kategori, satuan))
                conn.commit()
                print("Produk berhasil ditambahkan!")
                return True
                
        except Error as e:
            print(f"Error menambah produk: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def edit_product(self, id_produk, nama_produk=None, kategori=None, satuan=None):
        """Mengedit produk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                update_fields = []
                values = []
                
                if nama_produk:
                    update_fields.append("nama_produk = %s")
                    values.append(nama_produk)
                if kategori:
                    update_fields.append("kategori = %s")
                    values.append(kategori)
                if satuan:
                    update_fields.append("satuan = %s")
                    values.append(satuan)
                
                if update_fields:
                    values.append(id_produk)
                    query = f"UPDATE produk SET {', '.join(update_fields)} WHERE id_produk = %s"
                    cur.execute(query, values)
                    conn.commit()
                    print("Produk berhasil diupdate!")
                    return True
                else:
                    print("Tidak ada data yang diubah")
                    return False
                    
        except Error as e:
            print(f"Error mengedit produk: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def disable_product(self, id_produk):
        """Menonaktifkan produk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    return False
                
                cur.execute("UPDATE produk SET status = 'nonaktif' WHERE id_produk = %s", (id_produk,))
                conn.commit()
                print(f"Produk '{product[0]}' berhasil dinonaktifkan!")
                return True
                
        except Error as e:
            print(f"Error menonaktifkan produk: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def activate_product(self, id_produk):
        """Mengaktifkan produk yang dinonaktifkan"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    return False
                
                cur.execute("UPDATE produk SET status = 'aktif' WHERE id_produk = %s", (id_produk,))
                conn.commit()
                print(f"Produk '{product[0]}' berhasil diaktifkan!")
                return True
                
        except Error as e:
            print(f"Error mengaktifkan produk: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()

# =====================================================
# STOCK MANAGEMENT SYSTEM
# =====================================================

class StockManagement:
    def __init__(self):
        pass
    
    def add_stock(self, id_katalog, jumlah, harga_awal, harga_jual):
        """Menambah stok produk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT id_katalog, stok_tersedia FROM katalog WHERE id_katalog = %s", (id_katalog,))
                existing = cur.fetchone()
                
                if existing:
                    new_stock = existing[1] + jumlah
                    cur.execute("""
                        UPDATE katalog 
                        SET stok_tersedia = %s, harga_awal = %s, harga_jual = %s,
                            status = CASE WHEN %s > 0 THEN 'tersedia' ELSE 'habis' END
                        WHERE id_katalog = %s
                    """, (new_stock, harga_awal, harga_jual, new_stock, existing[0]))
                else:
                    cur.execute("""
                        INSERT INTO katalog (id_produk, stok_tersedia, harga_awal, harga_jual, tanggal_input)
                        SELECT id_produk, %s, %s, %s, CURRENT_DATE
                        FROM produk WHERE id_produk = %s
                    """, (jumlah, harga_awal, harga_jual, id_katalog))
                
                conn.commit()
                print("Stok berhasil ditambahkan!")
                return True
                
        except Error as e:
            print(f"Error menambah stok: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def update_price(self, id_katalog, harga_jual):
        """Mengedit harga produk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("""
                    UPDATE katalog SET harga_jual = %s 
                    WHERE id_katalog = %s
                """, (harga_jual, id_katalog))
                conn.commit()
                print("Harga berhasil diupdate!")
                return True
                
        except Error as e:
            print(f"Error mengedit harga: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def add_spoiled_product(self, id_katalog, qty_busuk, keterangan, id_user):
        """Mengurangi stok karena produk busuk"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT k.stok_tersedia, p.nama_produk, p.satuan, k.harga_jual
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.id_katalog = %s
                """, (id_katalog,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    return False
                
                stok_sebelum, nama_produk, satuan, harga_jual = product
                
                if qty_busuk > stok_sebelum:
                    print(f"Stok tidak mencukupi! Stok tersedia: {stok_sebelum} {satuan}")
                    return False
                
                stok_sesudah = stok_sebelum - qty_busuk
                nilai_kerugian = qty_busuk * harga_jual
                
                cur.execute("""
                    UPDATE katalog 
                    SET stok_tersedia = %s,
                        status = CASE WHEN %s > 0 THEN 'tersedia' ELSE 'habis' END
                    WHERE id_katalog = %s
                """, (stok_sesudah, stok_sesudah, id_katalog))
                
                cur.execute("""
                    INSERT INTO produk_busuk 
                    (id_katalog, nama_produk, qty_busuk, satuan, nilai_kerugian, keterangan, tanggal_pencatatan, id_user)
                    VALUES (%s, %s, %s, %s, %s, %s, CURRENT_DATE, %s)
                """, (id_katalog, nama_produk, qty_busuk, satuan, nilai_kerugian, keterangan, id_user))
                
                cur.execute("""
                    INSERT INTO log_stok 
                    (id_katalog, tipe_perubahan, jumlah_perubahan, stok_sebelum, stok_sesudah, keterangan, id_user)
                    VALUES (%s, 'produk_busuk', %s, %s, %s, %s, %s)
                """, (id_katalog, -qty_busuk, stok_sebelum, stok_sesudah, keterangan, id_user))
                
                conn.commit()
                print(f"Produk busuk berhasil dicatat! Kerugian: {format_currency(nilai_kerugian)}")
                return True
                
        except Error as e:
            print(f"Error mencatat produk busuk: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def view_catalog_with_prices(self):
        """Melihat katalog dengan harga"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        k.id_katalog,
                        p.nama_produk,
                        p.kategori,
                        p.satuan,
                        k.stok_tersedia,
                        k.harga_awal,
                        k.harga_jual,
                        (k.harga_jual - k.harga_awal) as margin
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE p.status = 'aktif'
                    ORDER BY p.nama_produk
                """
                cur.execute(query)
                catalog = cur.fetchall()
                return catalog
                
        except Error as e:
            print(f"Error mengambil data katalog: {e}")
            return None
        finally:
            conn.close()

# =====================================================
# USER MANAGEMENT SYSTEM
# =====================================================

class UserManagement:
    def __init__(self):
        pass
    
    def view_all_users(self):
        """Melihat semua user"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        id_user, username, nama_lengkap, role, status,
                        created_at, updated_at
                    FROM users 
                    ORDER BY id_user
                """
                cur.execute(query)
                users = cur.fetchall()
                return users
                
        except Error as e:
            print(f"Error mengambil data user: {e}")
            return None
        finally:
            conn.close()
    
    def display_users(self):
        """Menampilkan daftar user"""
        users = self.view_all_users()
        
        if not users:
            print("Tidak ada data user yang ditemukan!")
            return
        
        print_header("DAFTAR SEMUA USER")
        
        print(f"{'ID':<5} {'Username':<15} {'Nama Lengkap':<20} {'Role':<10} {'Status':<10}")
        print(f"{'-'*65}")
        
        for user in users:
            id_user, username, nama_lengkap, role, status, created_at, updated_at = user
            print(f"{id_user:<5} {username:<15} {nama_lengkap:<20} {role:<10} {status:<10}")
        
        print(f"{'-'*65}")
        print(f"Total user: {len(users)}")
        return users
    
    def add_user(self, username, password, nama_lengkap, role):
        """Menambah user baru"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT id_user FROM users WHERE username = %s", (username,))
                existing = cur.fetchone()
                
                if existing:
                    print(f"Username '{username}' sudah digunakan!")
                    return False
                
                cur.execute("""
                    INSERT INTO users (username, password, nama_lengkap, role)
                    VALUES (%s, %s, %s, %s)
                """, (username, password, nama_lengkap, role))
                conn.commit()
                print("User berhasil ditambahkan!")
                return True
                
        except Error as e:
            print(f"Error menambah user: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def edit_user(self, id_user, username=None, password=None, nama_lengkap=None, role=None):
        """Mengedit user"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                update_fields = []
                values = []
                
                if username:
                    update_fields.append("username = %s")
                    values.append(username)
                if password:
                    update_fields.append("password = %s")
                    values.append(password)
                if nama_lengkap:
                    update_fields.append("nama_lengkap = %s")
                    values.append(nama_lengkap)
                if role:
                    update_fields.append("role = %s")
                    values.append(role)
                
                if update_fields:
                    values.append(id_user)
                    query = f"UPDATE users SET {', '.join(update_fields)} WHERE id_user = %s"
                    cur.execute(query, values)
                    conn.commit()
                    print("User berhasil diupdate!")
                    return True
                else:
                    print("Tidak ada data yang diubah")
                    return False
                    
        except Error as e:
            print(f"Error mengedit user: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def disable_user(self, id_user):
        """Menonaktifkan user"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT username FROM users WHERE id_user = %s", (id_user,))
                user = cur.fetchone()
                
                if not user:
                    print("User tidak ditemukan!")
                    return False
                
                cur.execute("UPDATE users SET status = 'nonaktif' WHERE id_user = %s", (id_user,))
                conn.commit()
                print(f"User '{user[0]}' berhasil dinonaktifkan!")
                return True
                
        except Error as e:
            print(f"Error menonaktifkan user: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def activate_user(self, id_user):
        """Mengaktifkan user yang dinonaktifkan"""
        conn = get_db_connection()
        if not conn:
            print("Gagal terhubung ke database!")
            return False
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT username FROM users WHERE id_user = %s", (id_user,))
                user = cur.fetchone()
                
                if not user:
                    print("User tidak ditemukan!")
                    return False
                
                cur.execute("UPDATE users SET status = 'aktif' WHERE id_user = %s", (id_user,))
                conn.commit()
                print(f"User '{user[0]}' berhasil diaktifkan!")
                return True
                
        except Error as e:
            print(f"Error mengaktifkan user: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()

# =====================================================
# REPORT SYSTEM
# =====================================================

class ReportSystem:
    def __init__(self):
        pass
    
    def get_today_sales(self):
        """Mendapatkan data penjualan hari ini"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        t.kode_transaksi,
                        t.tanggal_transaksi,
                        u.nama_lengkap as kasir,
                        t.total_belanja,
                        t.uang_dibayar,
                        t.kembalian
                    FROM transaksi t
                    JOIN users u ON t.id_user = u.id_user
                    WHERE DATE(t.tanggal_transaksi) = CURRENT_DATE
                    ORDER BY t.tanggal_transaksi DESC
                """
                cur.execute(query)
                sales = cur.fetchall()
                return sales
                
        except Error as e:
            print(f"Error mengambil data penjualan: {e}")
            return None
        finally:
            conn.close()
    
    def display_today_sales(self):
        """Menampilkan data penjualan hari ini"""
        sales = self.get_today_sales()
        
        print_header("DATA PENJUALAN HARI INI")
        
        if not sales:
            print("Tidak ada transaksi hari ini!")
            return
        
        total_penjualan = 0
        total_transaksi = len(sales)
        
        print(f"{'Kode Transaksi':<20} {'Tanggal':<20} {'Kasir':<15} {'Total':<12} {'Bayar':<12} {'Kembali':<12}")
        print(f"{'-'*95}")
        
        for sale in sales:
            kode, tanggal, kasir, total, bayar, kembali = sale
            total_penjualan += total
            waktu = tanggal.strftime("%d/%m/%Y %H:%M")
            print(f"{kode:<20} {waktu:<20} {kasir:<15} {format_currency(total):<12} {format_currency(bayar):<12} {format_currency(kembali):<12}")
        
        print(f"{'-'*95}")
        print(f"Total Transaksi : {total_transaksi}")
        print(f"Total Penjualan : {format_currency(total_penjualan)}")
    
    def get_transaction_history(self, days=30):
        """Mendapatkan riwayat transaksi"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        t.kode_transaksi,
                        t.tanggal_transaksi,
                        u.nama_lengkap as kasir,
                        t.total_belanja,
                        t.uang_dibayar,
                        t.kembalian
                    FROM transaksi t
                    JOIN users u ON t.id_user = u.id_user
                    WHERE t.tanggal_transaksi >= CURRENT_DATE - INTERVAL '%s days'
                    ORDER BY t.tanggal_transaksi DESC
                """
                cur.execute(query, (days,))
                transactions = cur.fetchall()
                return transactions
                
        except Error as e:
            print(f"Error mengambil riwayat transaksi: {e}")
            return None
        finally:
            conn.close()
    
    def display_transaction_history(self):
        """Menampilkan riwayat transaksi"""
        print_header("RIWAYAT TRANSAKSI")
        
        print("Pilih periode:")
        print("1. 7 hari terakhir")
        print("2. 30 hari terakhir")
        print("3. Semua transaksi")
        
        choice = input("Pilih [1-3]: ").strip()
        
        if choice == '1':
            days = 7
        elif choice == '2':
            days = 30
        else:
            days = 3650  # 10 tahun
        
        transactions = self.get_transaction_history(days)
        
        if not transactions:
            print(f"Tidak ada transaksi dalam {days} hari terakhir!")
            return
        
        total_penjualan = 0
        total_transaksi = len(transactions)
        
        print(f"\nRIWAYAT TRANSAKSI ({days} hari terakhir):")
        print(f"{'Kode Transaksi':<20} {'Tanggal':<20} {'Kasir':<15} {'Total':<12} {'Bayar':<12} {'Kembali':<12}")
        print(f"{'-'*95}")
        
        for trans in transactions:
            kode, tanggal, kasir, total, bayar, kembali = trans
            total_penjualan += total
            waktu = tanggal.strftime("%d/%m/%Y %H:%M")
            print(f"{kode:<20} {waktu:<20} {kasir:<15} {format_currency(total):<12} {format_currency(bayar):<12} {format_currency(kembali):<12}")
        
        print(f"{'-'*95}")
        print(f"Total Transaksi : {total_transaksi}")
        print(f"Total Penjualan : {format_currency(total_penjualan)}")
    
    def get_weekly_sales(self):
        """Mendapatkan laporan penjualan mingguan"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        DATE_TRUNC('week', tanggal_transaksi) as minggu,
                        COUNT(*) as jumlah_transaksi,
                        SUM(total_belanja) as total_penjualan,
                        AVG(total_belanja) as rata_rata_transaksi
                    FROM transaksi
                    WHERE tanggal_transaksi >= CURRENT_DATE - INTERVAL '4 weeks'
                    GROUP BY DATE_TRUNC('week', tanggal_transaksi)
                    ORDER BY minggu DESC
                """
                cur.execute(query)
                weekly_sales = cur.fetchall()
                return weekly_sales
                
        except Error as e:
            print(f"Error mengambil data mingguan: {e}")
            return None
        finally:
            conn.close()
    
    def display_weekly_sales(self):
        """Menampilkan laporan penjualan mingguan"""
        weekly_sales = self.get_weekly_sales()
        
        print_header("LAPORAN PENJUALAN MINGGUAN")
        
        if not weekly_sales:
            print("Tidak ada data penjualan mingguan!")
            return
        
        print(f"{'Minggu':<20} {'Jml Transaksi':<15} {'Total Penjualan':<20} {'Rata2 Transaksi':<20}")
        print(f"{'-'*75}")
        
        total_all = 0
        for week in weekly_sales:
            minggu, jumlah, total, rata2 = week
            total_all += total if total else 0
            minggu_str = minggu.strftime("%d/%m/%Y")
            print(f"{minggu_str:<20} {jumlah:<15} {format_currency(total):<20} {format_currency(rata2):<20}")
        
        print(f"{'-'*75}")
        print(f"{'TOTAL':<35} {format_currency(total_all):<20}")
    
    def get_best_sellers(self, limit=10):
        """Mendapatkan produk terlaris"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        dt.nama_produk,
                        SUM(dt.qty) as total_terjual,
                        p.satuan,
                        SUM(dt.subtotal) as total_penjualan,
                        AVG(dt.harga_satuan) as harga_rata
                    FROM detail_transaksi dt
                    JOIN transaksi t ON dt.id_transaksi = t.id_transaksi
                    JOIN produk p ON dt.nama_produk = p.nama_produk
                    WHERE t.tanggal_transaksi >= CURRENT_DATE - INTERVAL '30 days'
                    GROUP BY dt.nama_produk, p.satuan
                    ORDER BY total_terjual DESC
                    LIMIT %s
                """
                cur.execute(query, (limit,))
                best_sellers = cur.fetchall()
                return best_sellers
                
        except Error as e:
            print(f"Error mengambil produk terlaris: {e}")
            return None
        finally:
            conn.close()
    
    def display_best_sellers(self):
        """Menampilkan produk terlaris"""
        best_sellers = self.get_best_sellers()
        
        print_header("PRODUK TERLARIS (30 HARI TERAKHIR)")
        
        if not best_sellers:
            print("Tidak ada data produk terlaris!")
            return
        
        print(f"{'Nama Produk':<20} {'Terjual':<10} {'Satuan':<8} {'Total Penjualan':<20} {'Harga Rata':<15}")
        print(f"{'-'*75}")
        
        for product in best_sellers:
            nama, terjual, satuan, total, harga_rata = product
            print(f"{nama:<20} {terjual:<10} {satuan:<8} {format_currency(total):<20} {format_currency(harga_rata):<15}")
    
    def get_slow_movers(self, limit=10):
        """Mendapatkan produk kurang laku"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        p.nama_produk,
                        p.satuan,
                        COALESCE(k.stok_tersedia, 0) as stok_tersedia,
                        COALESCE(k.harga_jual, 0) as harga_jual,
                        COALESCE(SUM(dt.qty), 0) as total_terjual
                    FROM produk p
                    LEFT JOIN katalog k ON p.id_produk = k.id_produk
                    LEFT JOIN detail_transaksi dt ON p.nama_produk = dt.nama_produk
                        AND dt.created_at >= CURRENT_DATE - INTERVAL '30 days'
                    WHERE p.status = 'aktif'
                    GROUP BY p.nama_produk, p.satuan, k.stok_tersedia, k.harga_jual
                    HAVING COALESCE(SUM(dt.qty), 0) <= 5
                    ORDER BY total_terjual ASC, p.nama_produk
                    LIMIT %s
                """
                cur.execute(query, (limit,))
                slow_movers = cur.fetchall()
                return slow_movers
                
        except Error as e:
            print(f"Error mengambil produk kurang laku: {e}")
            return None
        finally:
            conn.close()
    
    def display_slow_movers(self):
        """Menampilkan produk kurang laku"""
        slow_movers = self.get_slow_movers()
        
        print_header("PRODUK KURANG LAKU (TERJUAL <= 5 DALAM 30 HARI)")
        
        if not slow_movers:
            print("Tidak ada produk kurang laku!")
            return
        
        print(f"{'Nama Produk':<20} {'Satuan':<8} {'Stok':<10} {'Harga':<12} {'Terjual':<10}")
        print(f"{'-'*65}")
        
        for product in slow_movers:
            nama, satuan, stok, harga, terjual = product
            print(f"{nama:<20} {satuan:<8} {stok:<10} {format_currency(harga):<12} {terjual:<10}")

# =====================================================
# TRANSACTION RECEIPT SYSTEM
# =====================================================

class ReceiptSystem:
    def __init__(self):
        pass
    
    def get_receipt_data(self, kode_transaksi=None, id_transaksi=None):
        """Mendapatkan data struk transaksi"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                if kode_transaksi:
                    query = """
                        SELECT 
                            t.id_transaksi, t.kode_transaksi, t.tanggal_transaksi, 
                            u.nama_lengkap as kasir, t.total_belanja, t.uang_dibayar, t.kembalian
                        FROM transaksi t
                        JOIN users u ON t.id_user = u.id_user
                        WHERE t.kode_transaksi = %s
                    """
                    cur.execute(query, (kode_transaksi,))
                else:
                    query = """
                        SELECT 
                            t.id_transaksi, t.kode_transaksi, t.tanggal_transaksi, 
                            u.nama_lengkap as kasir, t.total_belanja, t.uang_dibayar, t.kembalian
                        FROM transaksi t
                        JOIN users u ON t.id_user = u.id_user
                        WHERE t.id_transaksi = %s
                    """
                    cur.execute(query, (id_transaksi,))
                
                transaksi = cur.fetchone()
                
                if not transaksi:
                    return None
                
                # Ambil detail transaksi
                query_detail = """
                    SELECT nama_produk, qty, satuan, harga_satuan, subtotal
                    FROM detail_transaksi
                    WHERE id_transaksi = %s
                    ORDER BY id_detail
                """
                cur.execute(query_detail, (transaksi[0],))
                detail = cur.fetchall()
                
                return transaksi, detail
                
        except Error as e:
            print(f"Error mengambil data struk: {e}")
            return None
        finally:
            conn.close()
    
    def print_receipt(self):
        """Mencetak struk transaksi"""
        print_header("CETAK STRUK TRANSAKSI")
        
        print("Pilih cara pencarian:")
        print("1. Cari dengan Kode Transaksi")
        print("2. Lihat transaksi hari ini")
        
        choice = input("Pilih [1-2]: ").strip()
        
        if choice == '1':
            kode_transaksi = input("Masukkan Kode Transaksi: ").strip()
            data = self.get_receipt_data(kode_transaksi=kode_transaksi)
        elif choice == '2':
            conn = get_db_connection()
            if not conn:
                pause()
                return
            
            try:
                with conn.cursor() as cur:
                    query = """
                        SELECT kode_transaksi, tanggal_transaksi, total_belanja
                        FROM transaksi
                        WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                        ORDER BY tanggal_transaksi DESC
                        LIMIT 10
                    """
                    cur.execute(query)
                    transactions = cur.fetchall()
                    
                    if not transactions:
                        print("Tidak ada transaksi hari ini!")
                        pause()
                        return
                    
                    print("\nTRANSAKSI HARI INI:")
                    print(f"{'No':<3} {'Kode Transaksi':<20} {'Waktu':<20} {'Total':<12}")
                    print(f"{'-'*60}")
                    
                    for i, trans in enumerate(transactions, 1):
                        kode, waktu, total = trans
                        waktu_str = waktu.strftime("%H:%M")
                        print(f"{i:<3} {kode:<20} {waktu_str:<20} {format_currency(total):<12}")
                    
                    print(f"{'-'*60}")
                    
                    try:
                        pilihan = int(input("\nPilih nomor transaksi: ").strip())
                        if 1 <= pilihan <= len(transactions):
                            kode_transaksi = transactions[pilihan-1][0]
                            data = self.get_receipt_data(kode_transaksi=kode_transaksi)
                        else:
                            print("Pilihan tidak valid!")
                            pause()
                            return
                    except ValueError:
                        print("Input harus angka!")
                        pause()
                        return
                    
            except Error as e:
                print(f"Error: {e}")
                pause()
                return
            finally:
                conn.close()
        else:
            print("Pilihan tidak valid!")
            pause()
            return
        
        if not data:
            print("Transaksi tidak ditemukan!")
            pause()
            return
        
        transaksi, detail = data
        id_transaksi, kode, tanggal, kasir, total, bayar, kembali = transaksi
        
        # Cetak struk
        print_header("STRUK TRANSAKSI")
        
        print(f"Kode Transaksi : {kode}")
        print(f"Tanggal        : {tanggal.strftime('%d/%m/%Y %H:%M:%S')}")
        print(f"Kasir          : {kasir}")
        print(f"{'-'*60}")
        print(f"{'Produk':<20} {'Qty':<6} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
        print(f"{'-'*60}")
        
        for item in detail:
            nama, qty, satuan, harga, subtotal = item
            print(f"{nama:<20} {qty:<6} {satuan:<8} {format_currency(harga):<12} {format_currency(subtotal):<12}")
        
        print(f"{'-'*60}")
        print(f"{'TOTAL':<34} {format_currency(total):<12}")
        print(f"{'UANG BAYAR':<34} {format_currency(bayar):<12}")
        print(f"{'KEMBALIAN':<34} {format_currency(kembali):<12}")
        print(f"{'-'*60}")
        print("Terima kasih atas kunjungan Anda!")
        
        # Tanya cetak ulang
        cetak_ulang = input("\nCetak struk lagi? (y/n): ").strip().lower()
        if cetak_ulang == 'y':
            self.print_receipt()

# =====================================================
# CASHIER TRANSACTION SYSTEM
# =====================================================

class CashierTransactionSystem:
    def __init__(self, user_data):
        self.user_data = user_data
        self.catalog = CatalogSystem()
    
    def generate_transaction_code(self):
        """Generate kode transaksi unik"""
        timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
        return f"TRX-{timestamp}"
    
    def get_available_products(self):
        """Mendapatkan produk yang tersedia untuk dijual"""
        return self.catalog.view_product_catalog(filter_tersedia=True)
    
    def display_available_products(self):
        """Menampilkan produk yang tersedia"""
        products = self.get_available_products()
        
        if not products:
            print("Tidak ada produk yang tersedia untuk dijual!")
            return None
        
        print_header("PRODUK TERSEDIA UNTUK DIJUAL")
        
        print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12}")
        print(f"{'-'*60}")
        
        for product in products:
            id_katalog, nama_produk, _, satuan, stok, harga, _, _ = product
            print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_currency(harga):<12}")
        
        print(f"{'-'*60}")
        return products
    
    def get_product_by_id(self, id_katalog):
        """Mendapatkan detail produk berdasarkan ID"""
        conn = get_db_connection()
        if not conn:
            return None
        
        try:
            with conn.cursor() as cur:
                query = """
                    SELECT 
                        k.id_katalog, p.nama_produk, p.satuan, k.stok_tersedia, k.harga_jual, k.harga_awal
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.id_katalog = %s AND p.status = 'aktif' AND k.stok_tersedia > 0
                """
                cur.execute(query, (id_katalog,))
                product = cur.fetchone()
                return product
                
        except Error as e:
            print(f"Error mengambil data produk: {e}")
            return None
        finally:
            conn.close()
    
    def process_sale(self):
        """Memproses transaksi penjualan"""
        print_header("TRANSAKSI PENJUALAN")
        
        products = self.display_available_products()
        if not products:
            pause()
            return
        
        cart = []
        total_belanja = 0
        
        print("\nMASUKKAN PRODUK YANG DIBELI:")
        print("Ketik 'selesai' untuk menyelesaikan transaksi")
        print("Ketik 'batal' untuk membatalkan transaksi")
        print()
        
        while True:
            try:
                id_input = input("Masukkan ID produk: ").strip()
                
                if id_input.lower() == 'selesai':
                    break
                elif id_input.lower() == 'batal':
                    print("Transaksi dibatalkan!")
                    return
                
                id_katalog = int(id_input)
                product = self.get_product_by_id(id_katalog)
                
                if not product:
                    print("ID produk tidak valid atau produk tidak tersedia!")
                    continue
                
                id_katalog, nama_produk, satuan, stok, harga_jual, harga_modal = product
                
                print(f"\nProduk: {nama_produk}")
                print(f"Stok tersedia: {stok} {satuan}")
                print(f"Harga: {format_currency(harga_jual)}")
                
                try:
                    qty = float(input(f"Jumlah {satuan} yang dibeli: ").strip())
                except ValueError:
                    print("Jumlah harus berupa angka!")
                    continue
                
                if qty <= 0:
                    print("Jumlah harus lebih dari 0!")
                    continue
                
                if qty > stok:
                    print(f"Stok tidak mencukupi! Stok tersedia: {stok} {satuan}")
                    continue
                
                subtotal = qty * harga_jual
                
                cart_item = {
                    'id_katalog': id_katalog,
                    'nama_produk': nama_produk,
                    'satuan': satuan,
                    'qty': qty,
                    'harga_satuan': harga_jual,
                    'harga_modal': harga_modal,
                    'subtotal': subtotal
                }
                
                cart.append(cart_item)
                total_belanja += subtotal
                
                print(f"Ditambahkan: {nama_produk} - {qty} {satuan} = {format_currency(subtotal)}")
                print(f"Total sementara: {format_currency(total_belanja)}")
                print()
                
            except ValueError:
                print("ID produk harus berupa angka!")
            except Exception as e:
                print(f"Error: {e}")
        
        if not cart:
            print("Tidak ada produk dalam keranjang!")
            pause()
            return
        
        self.display_cart_summary(cart, total_belanja)
        
        try:
            uang_dibayar = float(input("\nMasukkan uang yang dibayar: ").strip())
        except ValueError:
            print("Jumlah uang harus berupa angka!")
            pause()
            return
        
        if uang_dibayar < total_belanja:
            print("Uang yang dibayar kurang!")
            pause()
            return
        
        kembalian = uang_dibayar - total_belanja
        
        print(f"\nTOTAL BELANJA: {format_currency(total_belanja)}")
        print(f"UANG DIBAYAR : {format_currency(uang_dibayar)}")
        print(f"KEMBALIAN    : {format_currency(kembalian)}")
        
        confirm = input("\nKonfirmasi transaksi? (y/n): ").strip().lower()
        if confirm != 'y':
            print("Transaksi dibatalkan!")
            pause()
            return
        
        success = self.save_transaction(cart, total_belanja, uang_dibayar, kembalian)
        if success:
            print("Transaksi berhasil disimpan!")
            self.print_receipt_after_transaction(cart, total_belanja, uang_dibayar, kembalian)
        else:
            print("Gagal menyimpan transaksi!")
        
        pause()
    
    def display_cart_summary(self, cart, total_belanja):
        """Menampilkan ringkasan keranjang belanja"""
        print_header("RINGKASAN BELANJA")
        
        print(f"{'No':<3} {'Nama Produk':<20} {'Qty':<8} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
        print(f"{'-'*70}")
        
        for i, item in enumerate(cart, 1):
            print(f"{i:<3} {item['nama_produk']:<20} {item['qty']:<8} {item['satuan']:<8} {format_currency(item['harga_satuan']):<12} {format_currency(item['subtotal']):<12}")
        
        print(f"{'-'*70}")
        print(f"{'TOTAL BELANJA:':<50} {format_currency(total_belanja):<12}")
    
    def save_transaction(self, cart, total_belanja, uang_dibayar, kembalian):
        """Menyimpan transaksi ke database"""
        conn = get_db_connection()
        if not conn:
            return False
        
        try:
            with conn.cursor() as cur:
                kode_transaksi = self.generate_transaction_code()
                
                cur.execute("""
                    INSERT INTO transaksi (kode_transaksi, id_user, total_belanja, uang_dibayar, kembalian)
                    VALUES (%s, %s, %s, %s, %s)
                    RETURNING id_transaksi
                """, (kode_transaksi, self.user_data['id_user'], total_belanja, uang_dibayar, kembalian))
                
                id_transaksi = cur.fetchone()[0]
                
                for item in cart:
                    cur.execute("""
                        INSERT INTO detail_transaksi 
                        (id_transaksi, id_katalog, nama_produk, qty, satuan, harga_satuan, harga_modal, subtotal)
                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                    """, (id_transaksi, item['id_katalog'], item['nama_produk'], item['qty'], 
                          item['satuan'], item['harga_satuan'], item['harga_modal'], item['subtotal']))
                    
                    cur.execute("""
                        UPDATE katalog 
                        SET stok_tersedia = stok_tersedia - %s,
                            status = CASE 
                                WHEN (stok_tersedia - %s) <= 0 THEN 'habis'
                                ELSE 'tersedia'
                            END
                        WHERE id_katalog = %s
                    """, (item['qty'], item['qty'], item['id_katalog']))
                    
                    cur.execute("""
                        INSERT INTO log_stok 
                        (id_katalog, tipe_perubahan, jumlah_perubahan, stok_sebelum, stok_sesudah, keterangan, id_user)
                        SELECT 
                            %s, 'penjualan', %s, stok_tersedia, (stok_tersedia - %s), %s, %s
                        FROM katalog 
                        WHERE id_katalog = %s
                    """, (item['id_katalog'], item['qty'], item['qty'], 
                          f"Penjualan transaksi {kode_transaksi}", self.user_data['id_user'], item['id_katalog']))
                
                conn.commit()
                return True
                
        except Error as e:
            print(f"Error menyimpan transaksi: {e}")
            conn.rollback()
            return False
        finally:
            conn.close()
    
    def print_receipt_after_transaction(self, cart, total_belanja, uang_dibayar, kembalian):
        """Mencetak struk setelah transaksi"""
        print_header("STRUK TRANSAKSI")
        
        kode_transaksi = self.generate_transaction_code()
        current_time = datetime.now().strftime("%d/%m/%Y %H:%M:%S")
        
        print(f"Kode Transaksi : {kode_transaksi}")
        print(f"Tanggal        : {current_time}")
        print(f"Kasir          : {self.user_data['nama_lengkap']}")
        print(f"{'-'*60}")
        print(f"{'Produk':<20} {'Qty':<6} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
        print(f"{'-'*60}")
        
        for item in cart:
            print(f"{item['nama_produk']:<20} {item['qty']:<6} {item['satuan']:<8} {format_currency(item['harga_satuan']):<12} {format_currency(item['subtotal']):<12}")
        
        print(f"{'-'*60}")
        print(f"{'TOTAL':<34} {format_currency(total_belanja):<12}")
        print(f"{'UANG BAYAR':<34} {format_currency(uang_dibayar):<12}")
        print(f"{'KEMBALIAN':<34} {format_currency(kembalian):<12}")
        print(f"{'-'*60}")
        print("Terima kasih atas kunjungan Anda!")
        
        cetak_ulang = input("\nCetak struk lagi? (y/n): ").strip().lower()
        if cetak_ulang == 'y':
            self.print_receipt_after_transaction(cart, total_belanja, uang_dibayar, kembalian)

# =====================================================
# DASHBOARD SYSTEMS
# =====================================================

class AdminDashboard:
    def __init__(self, user_data):
        self.user_data = user_data
        self.catalog = CatalogSystem()
    
    def get_dashboard_data(self):
        """Mengambil data untuk dashboard admin"""
        conn = get_db_connection()
        if not conn:
            return None
            
        try:
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT 
                        COALESCE(SUM(total_belanja), 0) as penjualan_hari_ini,
                        COUNT(id_transaksi) as total_transaksi_hari_ini
                    FROM transaksi 
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                """)
                sales_today = cur.fetchone()
                
                cur.execute("""
                    SELECT 
                        COUNT(*) as total_produk,
                        COUNT(CASE WHEN stok_tersedia <= 5 AND stok_tersedia > 0 THEN 1 END) as stok_menipis,
                        COUNT(CASE WHEN stok_tersedia = 0 THEN 1 END) as stok_habis
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk 
                    WHERE p.status = 'aktif'
                """)
                stock_stats = cur.fetchone()
                
                cur.execute("""
                    SELECT dt.nama_produk, SUM(dt.qty) as total_terjual
                    FROM detail_transaksi dt
                    JOIN transaksi t ON dt.id_transaksi = t.id_transaksi
                    WHERE DATE(t.tanggal_transaksi) = CURRENT_DATE
                    GROUP BY dt.nama_produk
                    ORDER BY total_terjual DESC
                    LIMIT 1
                """)
                best_seller = cur.fetchone()
                
                cur.execute("""
                    SELECT p.nama_produk, k.stok_tersedia, p.satuan
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia <= 5 AND k.stok_tersedia > 0
                    AND p.status = 'aktif'
                    ORDER BY k.stok_tersedia ASC
                    LIMIT 5
                """)
                low_stock = cur.fetchall()
                
                cur.execute("""
                    SELECT p.nama_produk
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia = 0
                    AND p.status = 'aktif'
                    LIMIT 5
                """)
                out_of_stock = cur.fetchall()
                
                cur.execute("SELECT COUNT(*) FROM users WHERE status = 'aktif'")
                total_users = cur.fetchone()[0]
                
                return {
                    'penjualan_hari_ini': sales_today[0] if sales_today else 0,
                    'total_transaksi_hari_ini': sales_today[1] if sales_today else 0,
                    'total_produk': stock_stats[0] if stock_stats else 0,
                    'stok_menipis': stock_stats[1] if stock_stats else 0,
                    'stok_habis': stock_stats[2] if stock_stats else 0,
                    'produk_terlaris': best_seller[0] if best_seller else 'Tidak ada',
                    'total_terjual': best_seller[1] if best_seller else 0,
                    'low_stock_alerts': low_stock,
                    'out_of_stock_alerts': out_of_stock,
                    'total_users': total_users
                }
                
        except Error as e:
            print(f"Error mengambil data dashboard: {e}")
            return None
        finally:
            conn.close()
    
    def display_dashboard(self):
        """Menampilkan dashboard admin"""
        data = self.get_dashboard_data()
        if not data:
            print("Gagal memuat data dashboard")
            return
        
        print_header("DASHBOARD ADMIN - SISTEM MANAJEMEN TOKO")
        
        print(f"User: {self.user_data['nama_lengkap']} ({self.user_data['role'].upper()})")
        print(f"Tanggal: {datetime.now().strftime('%d/%m/%Y %H:%M:%S')}")
        print()
        
        print("STATISTIK HARI INI:")
        print(f"  Total Penjualan    : {format_currency(data['penjualan_hari_ini'])}")
        print(f"  Total Transaksi    : {data['total_transaksi_hari_ini']} transaksi")
        print(f"  Produk Terlaris    : {data['produk_terlaris']} ({data['total_terjual']} terjual)")
        print()
        
        print("STATISTIK STOK:")
        print(f"  Total Produk Aktif : {data['total_produk']} produk")
        print(f"  Stok Menipis       : {data['stok_menipis']} produk")
        print(f"  Stok Habis         : {data['stok_habis']} produk")
        print(f"  Total User Aktif   : {data['total_users']} user")
        print()
        
        print("NOTIFIKASI STOK MENIPIS:")
        if data['low_stock_alerts']:
            for i, product in enumerate(data['low_stock_alerts'], 1):
                print(f"  {i}. {product[0]} - {product[1]} {product[2]}")
        else:
            print("  Tidak ada stok menipis")
        print()
            
        print("NOTIFIKASI STOK HABIS:")
        if data['out_of_stock_alerts']:
            for i, product in enumerate(data['out_of_stock_alerts'], 1):
                print(f"  {i}. {product[0]}")
        else:
            print("  Tidak ada stok habis")
        
        print("\n" + "=" * 80)

class CashierDashboard:
    def __init__(self, user_data):
        self.user_data = user_data
        self.catalog = CatalogSystem()
    
    def get_dashboard_data(self):
        """Mengambil data untuk dashboard kasir"""
        conn = get_db_connection()
        if not conn:
            return None
            
        try:
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT 
                        COALESCE(SUM(total_belanja), 0) as penjualan_hari_ini,
                        COUNT(id_transaksi) as total_transaksi_hari_ini
                    FROM transaksi 
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE 
                    AND id_user = %s
                """, (self.user_data['id_user'],))
                sales_today = cur.fetchone()
                
                cur.execute("""
                    SELECT p.nama_produk, k.stok_tersedia, p.satuan
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia <= 5 AND k.stok_tersedia > 0
                    AND p.status = 'aktif'
                    ORDER BY k.stok_tersedia ASC
                    LIMIT 10
                """)
                low_stock = cur.fetchall()
                
                cur.execute("""
                    SELECT p.nama_produk
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia = 0
                    AND p.status = 'aktif'
                    LIMIT 10
                """)
                out_of_stock = cur.fetchall()
                
                return {
                    'penjualan_hari_ini': sales_today[0] if sales_today else 0,
                    'total_transaksi_hari_ini': sales_today[1] if sales_today else 0,
                    'low_stock_alerts': low_stock,
                    'out_of_stock_alerts': out_of_stock
                }
                
        except Error as e:
            print(f"Error mengambil data dashboard: {e}")
            return None
        finally:
            conn.close()
    
    def display_dashboard(self):
        """Menampilkan dashboard kasir"""
        data = self.get_dashboard_data()
        if not data:
            print("Gagal memuat data dashboard")
            return
        
        print_header("DASHBOARD KASIR - SISTEM PENJUALAN")
        
        print(f"Kasir: {self.user_data['nama_lengkap']}")
        print(f"Tanggal: {datetime.now().strftime('%d/%m/%Y %H:%M:%S')}")
        print()
        
        print("STATISTIK PENJUALAN HARI INI:")
        print(f"  Total Penjualan    : {format_currency(data['penjualan_hari_ini'])}")
        print(f"  Total Transaksi    : {data['total_transaksi_hari_ini']} transaksi")
        print()
        
        print("NOTIFIKASI STOK MENIPIS:")
        if data['low_stock_alerts']:
            for i, product in enumerate(data['low_stock_alerts'], 1):
                print(f"  {i}. {product[0]} - {product[1]} {product[2]}")
        else:
            print("  Tidak ada stok menipis")
        print()
            
        print("NOTIFIKASI STOK HABIS:")
        if data['out_of_stock_alerts']:
            for i, product in enumerate(data['out_of_stock_alerts'], 1):
                print(f"  {i}. {product[0]}")
        else:
            print("  Tidak ada stok habis")
        
        print("\n" + "=" * 80)

# =====================================================
# MENU SYSTEMS
# =====================================================

class AdminMenuSystem:
    def __init__(self, user_data):
        self.user_data = user_data
        self.dashboard = AdminDashboard(user_data)
        self.catalog = CatalogSystem()
        self.product_mgmt = ProductManagement()
        self.stock_mgmt = StockManagement()
        self.user_mgmt = UserManagement()
        self.report_system = ReportSystem()
        self.receipt_system = ReceiptSystem()
    
    def show_main_menu(self):
        """Menampilkan menu utama admin"""
        while True:
            self.dashboard.display_dashboard()
            
            print("\nMENU UTAMA ADMIN:")
            print("1. Manajemen Produk")
            print("2. Manajemen Katalog/Stok")
            print("3. Manajemen Users")
            print("4. Transaksi")
            print("5. Laporan & Analisis")
            print("6. Notifikasi")
            print("0. Logout")
            print()
            
            choice = input("Pilih menu [0-6]: ").strip()
            
            if choice == '1':
                self.manage_products()
            elif choice == '2':
                self.manage_stock()
            elif choice == '3':
                self.manage_users()
            elif choice == '4':
                self.manage_transactions()
            elif choice == '5':
                self.manage_reports()
            elif choice == '6':
                self.show_notifications()
            elif choice == '0':
                print("\nTerima kasih telah menggunakan sistem!")
                pause()
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_products(self):
        """Menu Manajemen Produk"""
        while True:
            print_header("MANAJEMEN PRODUK")
            print("1. Tambah Produk Baru")
            print("2. Edit Produk")
            print("3. Nonaktifkan Produk")
            print("4. Aktifkan Produk")
            print("5. Lihat Daftar Produk")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-5]: ").strip()
            
            if choice == '1':
                self.add_product()
            elif choice == '2':
                self.edit_product()
            elif choice == '3':
                self.disable_product()
            elif choice == '4':
                self.activate_product()
            elif choice == '5':
                self.view_products()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_stock(self):
        """Menu Manajemen Stok"""
        while True:
            print_header("MANAJEMEN STOK")
            print("1. Tambah Stok Produk")
            print("2. Edit Harga Produk")
            print("3. Kurangi Stok Produk Busuk")
            print("4. Lihat Ringkasan Stok")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-4]: ").strip()
            
            if choice == '1':
                self.add_stock()
            elif choice == '2':
                self.update_price()
            elif choice == '3':
                self.add_spoiled_product()
            elif choice == '4':
                self.view_stock_summary()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_users(self):
        """Menu Manajemen Users"""
        while True:
            print_header("MANAJEMEN USERS")
            print("1. Tambah User (Admin/Kasir)")
            print("2. Edit User")
            print("3. Nonaktifkan User")
            print("4. Aktifkan User")
            print("5. Lihat Daftar User")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-5]: ").strip()
            
            if choice == '1':
                self.add_user()
            elif choice == '2':
                self.edit_user()
            elif choice == '3':
                self.disable_user()
            elif choice == '4':
                self.activate_user()
            elif choice == '5':
                self.view_users()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_transactions(self):
        """Menu Transaksi"""
        while True:
            print_header("MANAJEMEN TRANSAKSI")
            print("1. Lihat Katalog Produk")
            print("2. Lakukan Transaksi Penjualan")
            print("3. Cetak Struk Transaksi")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-3]: ").strip()
            
            if choice == '1':
                self.view_product_catalog()
            elif choice == '2':
                self.make_sale()
            elif choice == '3':
                self.print_receipt()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_reports(self):
        """Menu Laporan & Analisis"""
        while True:
            print_header("LAPORAN & ANALISIS")
            print("1. Data Penjualan Hari Ini")
            print("2. Riwayat Transaksi")
            print("3. Laporan Penjualan Mingguan")
            print("4. Produk Terlaris")
            print("5. Produk Kurang Laku")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-5]: ").strip()
            
            if choice == '1':
                self.view_today_sales()
            elif choice == '2':
                self.view_transaction_history()
            elif choice == '3':
                self.view_weekly_sales()
            elif choice == '4':
                self.view_best_sellers()
            elif choice == '5':
                self.view_slow_movers()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def show_notifications(self):
        """Menu Notifikasi"""
        print_header("NOTIFIKASI SISTEM")
        
        data = self.dashboard.get_dashboard_data()
        if not data:
            print("Gagal memuat notifikasi")
            pause()
            return
        
        print("NOTIFIKASI STOK MENIPIS:")
        if data['low_stock_alerts']:
            for i, product in enumerate(data['low_stock_alerts'], 1):
                print(f"  {i}. {product[0]} - {product[1]} {product[2]}")
        else:
            print("  Tidak ada notifikasi stok menipis")
        print()
            
        print("NOTIFIKASI STOK HABIS:")
        if data['out_of_stock_alerts']:
            for i, product in enumerate(data['out_of_stock_alerts'], 1):
                print(f"  {i}. {product[0]}")
        else:
            print("  Tidak ada notifikasi stok habis")
        
        pause()
    
    # =====================================================
    # IMPLEMENTASI FITUR ADMIN
    # =====================================================
    
    def add_product(self):
        """Menambah produk baru"""
        print_header("TAMBAH PRODUK BARU")
        
        print("Masukkan data produk baru:")
        nama_produk = input("Nama Produk: ").strip()
        
        if not nama_produk:
            print("Nama produk tidak boleh kosong!")
            pause()
            return
        
        print("\nPilih Kategori:")
        print("1. Sayur")
        print("2. Buah")
        kategori_choice = input("Pilih [1-2]: ").strip()
        
        if kategori_choice == '1':
            kategori = 'sayur'
        elif kategori_choice == '2':
            kategori = 'buah'
        else:
            print("Pilihan kategori tidak valid!")
            pause()
            return
        
        print("\nPilih Satuan:")
        print("1. kg (kilogram)")
        print("2. pcs (buah)")
        satuan_choice = input("Pilih [1-2]: ").strip()
        
        if satuan_choice == '1':
            satuan = 'kg'
        elif satuan_choice == '2':
            satuan = 'pcs'
        else:
            print("Pilihan satuan tidak valid!")
            pause()
            return
        
        print(f"\nData produk yang akan ditambahkan:")
        print(f"Nama Produk: {nama_produk}")
        print(f"Kategori: {kategori}")
        print(f"Satuan: {satuan}")
        
        confirm = input("\nSimpan produk ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.product_mgmt.add_product(nama_produk, kategori, satuan)
            if success:
                print("Produk berhasil ditambahkan!")
            else:
                print("Gagal menambahkan produk!")
        else:
            print("Penambahan produk dibatalkan!")
        
        pause()
    
    def edit_product(self):
        """Mengedit produk"""
        print_header("EDIT PRODUK")
        
        products = self.product_mgmt.display_products()
        if not products:
            pause()
            return
        
        try:
            id_produk = int(input("\nMasukkan ID produk yang akan diedit: ").strip())
        except ValueError:
            print("ID produk harus berupa angka!")
            pause()
            return
        
        conn = get_db_connection()
        if not conn:
            pause()
            return
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT nama_produk, kategori, satuan FROM produk WHERE id_produk = %s", (id_produk,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    pause()
                    return
                
                print(f"\nData produk saat ini:")
                print(f"Nama: {product[0]}")
                print(f"Kategori: {product[1]}")
                print(f"Satuan: {product[2]}")
                print()
                
                nama_baru = input("Nama baru (kosongkan jika tidak diubah): ").strip()
                if not nama_baru:
                    nama_baru = None
                
                print("\nKategori baru:")
                print("1. Sayur")
                print("2. Buah")
                print("0. Tidak diubah")
                kategori_choice = input("Pilih [0-2]: ").strip()
                
                if kategori_choice == '1':
                    kategori_baru = 'sayur'
                elif kategori_choice == '2':
                    kategori_baru = 'buah'
                else:
                    kategori_baru = None
                
                print("\nSatuan baru:")
                print("1. kg")
                print("2. pcs")
                print("0. Tidak diubah")
                satuan_choice = input("Pilih [0-2]: ").strip()
                
                if satuan_choice == '1':
                    satuan_baru = 'kg'
                elif satuan_choice == '2':
                    satuan_baru = 'pcs'
                else:
                    satuan_baru = None
                
                if nama_baru or kategori_baru or satuan_baru:
                    confirm = input("\nUpdate produk? (y/n): ").strip().lower()
                    if confirm == 'y':
                        success = self.product_mgmt.edit_product(id_produk, nama_baru, kategori_baru, satuan_baru)
                        if success:
                            print("Produk berhasil diupdate!")
                        else:
                            print("Gagal mengupdate produk!")
                    else:
                        print("Update produk dibatalkan!")
                else:
                    print("Tidak ada perubahan yang dilakukan!")
                
        except Error as e:
            print(f"Error: {e}")
        finally:
            conn.close()
        
        pause()
    
    def disable_product(self):
        """Menonaktifkan produk"""
        print_header("NONAKTIFKAN PRODUK")
        
        products = self.product_mgmt.view_all_products()
        if not products:
            print("Tidak ada produk yang ditemukan!")
            pause()
            return
        
        active_products = [p for p in products if p[4] == 'aktif']
        if not active_products:
            print("Tidak ada produk aktif!")
            pause()
            return
        
        print("DAFTAR PRODUK AKTIF:")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Kategori':<10} {'Satuan':<8}")
        print(f"{'-'*50}")
        for product in active_products:
            if product[4] == 'aktif':
                print(f"{product[0]:<5} {product[1]:<20} {product[2]:<10} {product[3]:<8}")
        
        try:
            id_produk = int(input("\nMasukkan ID produk yang akan dinonaktifkan: ").strip())
        except ValueError:
            print("ID produk harus berupa angka!")
            pause()
            return
        
        confirm = input("Yakin ingin menonaktifkan produk ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.product_mgmt.disable_product(id_produk)
            if not success:
                print("Gagal menonaktifkan produk!")
        else:
            print("Produk tidak dinonaktifkan!")
        
        pause()
    
    def activate_product(self):
        """Mengaktifkan produk yang dinonaktifkan"""
        print_header("AKTIFKAN PRODUK")
        
        products = self.product_mgmt.view_all_products()
        if not products:
            print("Tidak ada produk yang ditemukan!")
            pause()
            return
        
        inactive_products = [p for p in products if p[4] == 'nonaktif']
        if not inactive_products:
            print("Tidak ada produk yang nonaktif!")
            pause()
            return
        
        print("DAFTAR PRODUK NONAKTIF:")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Kategori':<10} {'Satuan':<8}")
        print(f"{'-'*50}")
        for product in inactive_products:
            print(f"{product[0]:<5} {product[1]:<20} {product[2]:<10} {product[3]:<8}")
        
        try:
            id_produk = int(input("\nMasukkan ID produk yang akan diaktifkan: ").strip())
        except ValueError:
            print("ID produk harus berupa angka!")
            pause()
            return
        
        confirm = input("Yakin ingin mengaktifkan produk ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.product_mgmt.activate_product(id_produk)
            if not success:
                print("Gagal mengaktifkan produk!")
        else:
            print("Produk tidak diaktifkan!")
        
        pause()
    
    def view_products(self):
        """Melihat daftar produk"""
        self.product_mgmt.display_products()
        pause()
    
    def add_stock(self):
        """Menambah stok produk"""
        print_header("TAMBAH STOK PRODUK")
        
        catalog = self.catalog.view_product_catalog(filter_tersedia=False)
        if not catalog:
            print("Tidak ada produk dalam katalog!")
            pause()
            return
        
        print("DAFTAR PRODUK:")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga Awal':<12} {'Harga Jual':<12}")
        print(f"{'-'*80}")
        
        for item in catalog:
            id_katalog, nama_produk, _, satuan, stok, harga_jual, _, _ = item
            print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_currency(0):<12} {format_currency(harga_jual):<12}")
        
        print(f"{'-'*80}")
        
        try:
            id_katalog = int(input("\nMasukkan ID katalog: ").strip())
        except ValueError:
            print("ID harus berupa angka!")
            pause()
            return
        
        try:
            jumlah = float(input("Jumlah stok yang ditambahkan: ").strip())
            if jumlah <= 0:
                print("Jumlah harus lebih dari 0!")
                pause()
                return
        except ValueError:
            print("Jumlah harus berupa angka!")
            pause()
            return
        
        try:
            harga_awal = float(input("Harga awal (modal) per unit: ").strip())
            if harga_awal < 0:
                print("Harga tidak boleh negatif!")
                pause()
                return
        except ValueError:
            print("Harga harus berupa angka!")
            pause()
            return
        
        try:
            harga_jual = float(input("Harga jual per unit: ").strip())
            if harga_jual < 0:
                print("Harga tidak boleh negatif!")
                pause()
                return
        except ValueError:
            print("Harga harus berupa angka!")
            pause()
            return
        
        print(f"\nData yang akan ditambahkan:")
        print(f"ID Katalog: {id_katalog}")
        print(f"Jumlah: {jumlah}")
        print(f"Harga Awal: {format_currency(harga_awal)}")
        print(f"Harga Jual: {format_currency(harga_jual)}")
        
        confirm = input("\nTambahkan stok? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.stock_mgmt.add_stock(id_katalog, jumlah, harga_awal, harga_jual)
            if not success:
                print("Gagal menambah stok!")
        else:
            print("Penambahan stok dibatalkan!")
        
        pause()
    
    def update_price(self):
        """Mengedit harga produk"""
        print_header("EDIT HARGA PRODUK")
        
        catalog = self.catalog.view_product_catalog(filter_tersedia=False)
        if not catalog:
            print("Tidak ada produk dalam katalog!")
            pause()
            return
        
        print("DAFTAR PRODUK:")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga Jual':<12}")
        print(f"{'-'*60}")
        
        for item in catalog:
            id_katalog, nama_produk, _, satuan, stok, harga_jual, _, _ = item
            print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_currency(harga_jual):<12}")
        
        print(f"{'-'*60}")
        
        try:
            id_katalog = int(input("\nMasukkan ID katalog: ").strip())
        except ValueError:
            print("ID harus berupa angka!")
            pause()
            return
        
        try:
            harga_baru = float(input("Harga jual baru: ").strip())
            if harga_baru < 0:
                print("Harga tidak boleh negatif!")
                pause()
                return
        except ValueError:
            print("Harga harus berupa angka!")
            pause()
            return
        
        print(f"\nHarga baru: {format_currency(harga_baru)}")
        
        confirm = input("\nUpdate harga? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.stock_mgmt.update_price(id_katalog, harga_baru)
            if not success:
                print("Gagal mengupdate harga!")
        else:
            print("Update harga dibatalkan!")
        
        pause()
    
    def add_spoiled_product(self):
        """Mengurangi stok produk busuk"""
        print_header("PRODUK BUSUK")
        
        catalog = self.catalog.view_product_catalog(filter_tersedia=True)
        if not catalog:
            print("Tidak ada produk yang tersedia!")
            pause()
            return
        
        print("DAFTAR PRODUK TERSEDIA:")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12}")
        print(f"{'-'*60}")
        
        for item in catalog:
            id_katalog, nama_produk, _, satuan, stok, harga_jual, _, _ = item
            print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_currency(harga_jual):<12}")
        
        print(f"{'-'*60}")
        
        try:
            id_katalog = int(input("\nMasukkan ID katalog: ").strip())
        except ValueError:
            print("ID harus berupa angka!")
            pause()
            return
        
        try:
            qty_busuk = float(input("Jumlah produk yang busuk: ").strip())
            if qty_busuk <= 0:
                print("Jumlah harus lebih dari 0!")
                pause()
                return
        except ValueError:
            print("Jumlah harus berupa angka!")
            pause()
            return
        
        keterangan = input("Keterangan (misal: kadaluarsa, rusak): ").strip()
        if not keterangan:
            keterangan = "Produk busuk"
        
        print(f"\nData produk busuk:")
        print(f"ID Katalog: {id_katalog}")
        print(f"Jumlah busuk: {qty_busuk}")
        print(f"Keterangan: {keterangan}")
        
        confirm = input("\nSimpan data produk busuk? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.stock_mgmt.add_spoiled_product(id_katalog, qty_busuk, keterangan, self.user_data['id_user'])
            if not success:
                print("Gagal menyimpan data produk busuk!")
        else:
            print("Penyimpanan data dibatalkan!")
        
        pause()
    
    def view_stock_summary(self):
        """Melihat ringkasan stok"""
        self.catalog.display_stock_summary()
        pause()
    
    def add_user(self):
        """Menambah user"""
        print_header("TAMBAH USER")
        
        print("Masukkan data user baru:")
        username = input("Username: ").strip()
        
        if not username:
            print("Username tidak boleh kosong!")
            pause()
            return
        
        password = input("Password: ").strip()
        if not password:
            print("Password tidak boleh kosong!")
            pause()
            return
        
        nama_lengkap = input("Nama Lengkap: ").strip()
        if not nama_lengkap:
            print("Nama lengkap tidak boleh kosong!")
            pause()
            return
        
        print("\nPilih Role:")
        print("1. Admin")
        print("2. Kasir")
        role_choice = input("Pilih [1-2]: ").strip()
        
        if role_choice == '1':
            role = 'admin'
        elif role_choice == '2':
            role = 'kasir'
        else:
            print("Pilihan role tidak valid!")
            pause()
            return
        
        print(f"\nData user yang akan ditambahkan:")
        print(f"Username: {username}")
        print(f"Password: {'*' * len(password)}")
        print(f"Nama Lengkap: {nama_lengkap}")
        print(f"Role: {role}")
        
        confirm = input("\nSimpan user ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.user_mgmt.add_user(username, password, nama_lengkap, role)
            if success:
                print("User berhasil ditambahkan!")
            else:
                print("Gagal menambahkan user!")
        else:
            print("Penambahan user dibatalkan!")
        
        pause()
    
    def edit_user(self):
        """Mengedit user"""
        print_header("EDIT USER")
        
        users = self.user_mgmt.display_users()
        if not users:
            pause()
            return
        
        try:
            id_user = int(input("\nMasukkan ID user yang akan diedit: ").strip())
        except ValueError:
            print("ID user harus berupa angka!")
            pause()
            return
        
        conn = get_db_connection()
        if not conn:
            pause()
            return
        
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT username, nama_lengkap, role FROM users WHERE id_user = %s", (id_user,))
                user = cur.fetchone()
                
                if not user:
                    print("User tidak ditemukan!")
                    pause()
                    return
                
                print(f"\nData user saat ini:")
                print(f"Username: {user[0]}")
                print(f"Nama Lengkap: {user[1]}")
                print(f"Role: {user[2]}")
                print()
                
                username_baru = input("Username baru (kosongkan jika tidak diubah): ").strip()
                if not username_baru:
                    username_baru = None
                
                password_baru = input("Password baru (kosongkan jika tidak diubah): ").strip()
                if not password_baru:
                    password_baru = None
                
                nama_baru = input("Nama lengkap baru (kosongkan jika tidak diubah): ").strip()
                if not nama_baru:
                    nama_baru = None
                
                print("\nRole baru:")
                print("1. Admin")
                print("2. Kasir")
                print("0. Tidak diubah")
                role_choice = input("Pilih [0-2]: ").strip()
                
                if role_choice == '1':
                    role_baru = 'admin'
                elif role_choice == '2':
                    role_baru = 'kasir'
                else:
                    role_baru = None
                
                if username_baru or password_baru or nama_baru or role_baru:
                    confirm = input("\nUpdate user? (y/n): ").strip().lower()
                    if confirm == 'y':
                        success = self.user_mgmt.edit_user(id_user, username_baru, password_baru, nama_baru, role_baru)
                        if success:
                            print("User berhasil diupdate!")
                        else:
                            print("Gagal mengupdate user!")
                    else:
                        print("Update user dibatalkan!")
                else:
                    print("Tidak ada perubahan yang dilakukan!")
                
        except Error as e:
            print(f"Error: {e}")
        finally:
            conn.close()
        
        pause()
    
    def disable_user(self):
        """Menonaktifkan user"""
        print_header("NONAKTIFKAN USER")
        
        users = self.user_mgmt.view_all_users()
        if not users:
            print("Tidak ada user yang ditemukan!")
            pause()
            return
        
        active_users = [u for u in users if u[4] == 'aktif']
        if not active_users:
            print("Tidak ada user aktif!")
            pause()
            return
        
        print("DAFTAR USER AKTIF:")
        print(f"{'ID':<5} {'Username':<15} {'Nama Lengkap':<20} {'Role':<10}")
        print(f"{'-'*55}")
        for user in active_users:
            if user[4] == 'aktif':
                print(f"{user[0]:<5} {user[1]:<15} {user[2]:<20} {user[3]:<10}")
        
        try:
            id_user = int(input("\nMasukkan ID user yang akan dinonaktifkan: ").strip())
        except ValueError:
            print("ID user harus berupa angka!")
            pause()
            return
        
        confirm = input("Yakin ingin menonaktifkan user ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.user_mgmt.disable_user(id_user)
            if not success:
                print("Gagal menonaktifkan user!")
        else:
            print("User tidak dinonaktifkan!")
        
        pause()
    
    def activate_user(self):
        """Mengaktifkan user yang dinonaktifkan"""
        print_header("AKTIFKAN USER")
        
        users = self.user_mgmt.view_all_users()
        if not users:
            print("Tidak ada user yang ditemukan!")
            pause()
            return
        
        inactive_users = [u for u in users if u[4] == 'nonaktif']
        if not inactive_users:
            print("Tidak ada user yang nonaktif!")
            pause()
            return
        
        print("DAFTAR USER NONAKTIF:")
        print(f"{'ID':<5} {'Username':<15} {'Nama Lengkap':<20} {'Role':<10}")
        print(f"{'-'*55}")
        for user in inactive_users:
            print(f"{user[0]:<5} {user[1]:<15} {user[2]:<20} {user[3]:<10}")
        
        try:
            id_user = int(input("\nMasukkan ID user yang akan diaktifkan: ").strip())
        except ValueError:
            print("ID user harus berupa angka!")
            pause()
            return
        
        confirm = input("Yakin ingin mengaktifkan user ini? (y/n): ").strip().lower()
        if confirm == 'y':
            success = self.user_mgmt.activate_user(id_user)
            if not success:
                print("Gagal mengaktifkan user!")
        else:
            print("User tidak diaktifkan!")
        
        pause()
    
    def view_users(self):
        """Melihat daftar user"""
        self.user_mgmt.display_users()
        pause()
    
    def view_product_catalog(self):
        """Melihat katalog produk"""
        while True:
            print_header("KATALOG PRODUK")
            print("1. Lihat Semua Produk")
            print("2. Lihat Produk Tersedia Saja")
            print("3. Lihat Ringkasan Stok")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-3]: ").strip()
            
            if choice == '1':
                self.catalog.display_catalog(filter_tersedia=False)
                pause()
            elif choice == '2':
                self.catalog.display_catalog(filter_tersedia=True)
                pause()
            elif choice == '3':
                self.catalog.display_stock_summary()
                pause()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def make_sale(self):
        """Melakukan transaksi penjualan"""
        print_header("TRANSAKSI PENJUALAN")
        print("Fitur transaksi penjualan hanya tersedia untuk kasir.")
        print("Silakan login sebagai kasir untuk melakukan transaksi.")
        pause()
    
    def print_receipt(self):
        """Cetak struk transaksi"""
        self.receipt_system.print_receipt()
        pause()
    
    def view_today_sales(self):
        """Melihat data penjualan hari ini"""
        self.report_system.display_today_sales()
        pause()
    
    def view_transaction_history(self):
        """Melihat riwayat transaksi"""
        self.report_system.display_transaction_history()
        pause()
    
    def view_weekly_sales(self):
        """Melihat laporan penjualan mingguan"""
        self.report_system.display_weekly_sales()
        pause()
    
    def view_best_sellers(self):
        """Melihat produk terlaris"""
        self.report_system.display_best_sellers()
        pause()
    
    def view_slow_movers(self):
        """Melihat produk kurang laku"""
        self.report_system.display_slow_movers()
        pause()

class CashierMenuSystem:
    def __init__(self, user_data):
        self.user_data = user_data
        self.dashboard = CashierDashboard(user_data)
        self.transaction = CashierTransactionSystem(user_data)
        self.catalog = CatalogSystem()
        self.receipt_system = ReceiptSystem()
    
    def show_main_menu(self):
        """Menampilkan menu utama kasir"""
        while True:
            self.dashboard.display_dashboard()
            
            print("\nMENU UTAMA KASIR:")
            print("1. Katalog Produk")
            print("2. Transaksi Penjualan")
            print("3. Cetak Struk Transaksi")
            print("0. Logout")
            print()
            
            choice = input("Pilih menu [0-3]: ").strip()
            
            if choice == '1':
                self.manage_catalog()
            elif choice == '2':
                self.process_sale()
            elif choice == '3':
                self.print_receipt()
            elif choice == '0':
                print("\nTerima kasih telah menggunakan sistem!")
                pause()
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def manage_catalog(self):
        """Menu Katalog Produk untuk Kasir"""
        while True:
            print_header("KATALOG PRODUK - KASIR")
            print("1. Lihat Semua Produk")
            print("2. Lihat Produk Tersedia Saja")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-2]: ").strip()
            
            if choice == '1':
                self.catalog.display_catalog(filter_tersedia=False)
                pause()
            elif choice == '2':
                self.catalog.display_catalog(filter_tersedia=True)
                pause()
            elif choice == '0':
                break
            else:
                print("\nPilihan tidak valid!")
                pause()
    
    def process_sale(self):
        """Memproses transaksi penjualan"""
        self.transaction.process_sale()
    
    def print_receipt(self):
        """Cetak struk transaksi"""
        self.receipt_system.print_receipt()
        pause()

# =====================================================
# LOGIN FUNCTION
# =====================================================

def login():
    """Fungsi login untuk user"""
    print_header("LOGIN SISTEM GUDANG HORTIKULTURA")
    
    print("Silakan masukkan kredensial Anda:\n")
    username = input("Username: ").strip()
    password = input("Password: ").strip()
    
    if not username or not password:
        print("\nUsername dan Password tidak boleh kosong!")
        pause()
        return None
    
    conn = get_db_connection()
    if not conn:
        print("\nGagal terhubung ke database!")
        pause()
        return None
    
    try:
        cursor = conn.cursor()
        
        query = """
            SELECT id_user, nama_lengkap, role, status 
            FROM users 
            WHERE username = %s AND password = %s
        """
        cursor.execute(query, (username, password))
        user = cursor.fetchone()
        
        if user:
            if user[3] == 'aktif':
                print(f"\nLogin berhasil!")
                print(f"Selamat datang, {user[1]} ({user[2].upper()})")
                pause()
                return {
                    'id_user': user[0],
                    'nama_lengkap': user[1],
                    'role': user[2],
                    'status': user[3]
                }
            else:
                print("\nAkun Anda tidak aktif! Hubungi administrator.")
                pause()
                return None
        else:
            print("\nUsername atau Password salah!")
            pause()
            return None
            
    except Error as e:
        print(f"\nError saat login: {e}")
        pause()
        return None
    finally:
        close_connection(conn, cursor)

# =====================================================
# MAIN PROGRAM
# =====================================================

def main():
    """Program utama"""
    max_attempts = 3
    attempts = 0
    
    while attempts < max_attempts:
        user = login()
        
        if user:
            if user['role'] == 'admin':
                admin_system = AdminMenuSystem(user)
                admin_system.show_main_menu()
            elif user['role'] == 'kasir':
                cashier_system = CashierMenuSystem(user)
                cashier_system.show_main_menu()
            break
        else:
            attempts += 1
            remaining = max_attempts - attempts
            
            if remaining > 0:
                print(f"\nSisa percobaan login: {remaining}x")
                pause()
            else:
                print("\nAnda telah mencapai batas maksimal percobaan login!")
                print("Sistem akan ditutup untuk keamanan.")
                pause()
                break

if __name__ == "__main__":
    main()
