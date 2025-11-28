import psycopg2
from psycopg2 import Error
import os
from datetime import datetime, timedelta

# =====================================================
# DATABASE CONNECTION
# =====================================================

def get_db_connection():
    """Membuat koneksi ke database PostgreSQL"""
    try:
        conn = psycopg2.connect(
            host="localhost",
            database="Projek",
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
                    # Hanya tampilkan produk yang tersedia
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
                else:
                    # Tampilkan semua produk termasuk yang habis
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
        
        # Hitung statistik
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
        
        # Group by kategori
        categories = {}
        for item in catalog:
            kategori = item[2]
            if kategori not in categories:
                categories[kategori] = []
            categories[kategori].append(item)
        
        # Tampilkan per kategori
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
                # Ringkasan stok per kategori
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
                
                # Produk dengan stok menipis
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
                
                # Produk habis
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
        """Menampilkan ringkasan stok dalam format yang rapi"""
        stock_summary, low_stock, out_of_stock = self.view_stock_summary()
        
        print_header("RINGKASAN STOK TOKO")
        
        if not stock_summary:
            print("Tidak ada data stok yang ditemukan!")
            return
        
        # Ringkasan per kategori
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
        
        # Stok Menipis
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
        
        # Stok Habis
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
                # Cek apakah produk sudah ada
                cur.execute("SELECT id_produk FROM produk WHERE nama_produk = %s", (nama_produk,))
                existing = cur.fetchone()
                
                if existing:
                    print(f"Produk '{nama_produk}' sudah ada dalam database!")
                    return False
                
                # Tambah produk baru
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
                # Cek apakah produk ada
                cur.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    return False
                
                # Nonaktifkan produk
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
                # Cek apakah produk ada
                cur.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
                product = cur.fetchone()
                
                if not product:
                    print("Produk tidak ditemukan!")
                    return False
                
                # Aktifkan produk
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
# DASHBOARD SYSTEM
# =====================================================

class DashboardSystem:
    def __init__(self, user_data):
        self.user_data = user_data
        self.catalog = CatalogSystem()
    
    def get_dashboard_data(self):
        """Mengambil data untuk dashboard"""
        conn = get_db_connection()
        if not conn:
            return None
            
        try:
            with conn.cursor() as cur:
                # Statistik Penjualan Hari Ini
                cur.execute("""
                    SELECT 
                        COALESCE(SUM(total_belanja), 0) as penjualan_hari_ini,
                        COUNT(id_transaksi) as total_transaksi_hari_ini
                    FROM transaksi 
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                """)
                sales_today = cur.fetchone()
                
                # Statistik Stok
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
                
                # Produk Terlaris Hari Ini
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
                
                # Notifikasi Stok Menipis
                cur.execute("""
                    SELECT p.nama_produk, k.stok_tersedia, k.satuan
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia <= 5 AND k.stok_tersedia > 0
                    AND p.status = 'aktif'
                    ORDER BY k.stok_tersedia ASC
                    LIMIT 5
                """)
                low_stock = cur.fetchall()
                
                # Notifikasi Stok Habis
                cur.execute("""
                    SELECT p.nama_produk
                    FROM katalog k
                    JOIN produk p ON k.id_produk = p.id_produk
                    WHERE k.stok_tersedia = 0
                    AND p.status = 'aktif'
                    LIMIT 5
                """)
                out_of_stock = cur.fetchall()
                
                # Total User Aktif
                cur.execute("""
                    SELECT COUNT(*) FROM users WHERE status = 'aktif'
                """)
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
        
        # Info User
        print(f"User: {self.user_data['nama_lengkap']} ({self.user_data['role'].upper()})")
        print(f"Tanggal: {datetime.now().strftime('%d/%m/%Y %H:%M:%S')}")
        print()
        
        # Statistik Utama
        print("STATISTIK HARI INI:")
        print(f"  Total Penjualan    : {format_currency(data['penjualan_hari_ini'])}")
        print(f"  Total Transaksi    : {data['total_transaksi_hari_ini']} transaksi")
        print(f"  Produk Terlaris    : {data['produk_terlaris']} ({data['total_terjual']} terjual)")
        print()
        
        # Statistik Stok
        print("STATISTIK STOK:")
        print(f"  Total Produk Aktif : {data['total_produk']} produk")
        print(f"  Stok Menipis       : {data['stok_menipis']} produk")
        print(f"  Stok Habis         : {data['stok_habis']} produk")
        print(f"  Total User Aktif   : {data['total_users']} user")
        print()
        
        # Notifikasi Stok Menipis
        print("NOTIFIKASI STOK MENIPIS:")
        if data['low_stock_alerts']:
            for i, product in enumerate(data['low_stock_alerts'], 1):
                print(f"  {i}. {product[0]} - {product[1]} {product[2]}")
        else:
            print("  Tidak ada stok menipis")
        print()
            
        # Notifikasi Stok Habis
        print("NOTIFIKASI STOK HABIS:")
        if data['out_of_stock_alerts']:
            for i, product in enumerate(data['out_of_stock_alerts'], 1):
                print(f"  {i}. {product[0]}")
        else:
            print("  Tidak ada stok habis")
        
        print("\n" + "=" * 80)

# =====================================================
# ADMIN MENU SYSTEM
# =====================================================

class AdminMenuSystem:
    def __init__(self, user_data):
        self.user_data = user_data
        self.dashboard = DashboardSystem(user_data)
        self.catalog = CatalogSystem()
        self.product_mgmt = ProductManagement()
    
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
            print("4. Lihat Daftar User")
            print("0. Kembali ke Menu Utama")
            print()
            
            choice = input("Pilih menu [0-4]: ").strip()
            
            if choice == '1':
                self.add_user()
            elif choice == '2':
                self.edit_user()
            elif choice == '3':
                self.disable_user()
            elif choice == '4':
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
    
    def view_stock_summary(self):
        """Melihat ringkasan stok"""
        self.catalog.display_stock_summary()
        pause()

    # =====================================================
    # IMPLEMENTASI FITUR MANAJEMEN PRODUK
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
        
        # Konfirmasi
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
        
        # Tampilkan daftar produk terlebih dahulu
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
        
        # Cek apakah produk exists
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
                
                # Input data baru
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
                
                # Konfirmasi
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
        
        # Tampilkan daftar produk aktif
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
        
        # Tampilkan daftar produk nonaktif
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

    # =====================================================
    # FUNGSI LAINNYA (Skeleton)
    # =====================================================
    
    def add_stock(self):
        """Menambah stok produk"""
        print_header("TAMBAH STOK PRODUK")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def update_price(self):
        """Mengedit harga produk"""
        print_header("EDIT HARGA PRODUK")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def add_spoiled_product(self):
        """Mengurangi stok produk busuk"""
        print_header("PRODUK BUSUK")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def add_user(self):
        """Menambah user"""
        print_header("TAMBAH USER")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def edit_user(self):
        """Mengedit user"""
        print_header("EDIT USER")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def disable_user(self):
        """Menonaktifkan user"""
        print_header("NONAKTIFKAN USER")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_users(self):
        """Melihat daftar user"""
        print_header("DAFTAR USER")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def make_sale(self):
        """Melakukan transaksi penjualan"""
        print_header("TRANSAKSI PENJUALAN")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def print_receipt(self):
        """Cetak struk transaksi"""
        print_header("CETAK STRUK")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_today_sales(self):
        """Melihat data penjualan hari ini"""
        print_header("PENJUALAN HARI INI")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_transaction_history(self):
        """Melihat riwayat transaksi"""
        print_header("RIWAYAT TRANSAKSI")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_weekly_sales(self):
        """Melihat laporan penjualan mingguan"""
        print_header("LAPORAN MINGGUAN")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_best_sellers(self):
        """Melihat produk terlaris"""
        print_header("PRODUK TERLARIS")
        print("Fitur ini akan diimplementasikan...")
        pause()
    
    def view_slow_movers(self):
        """Melihat produk kurang laku"""
        print_header("PRODUK KURANG LAKU")
        print("Fitur ini akan diimplementasikan...")
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
    
    # Validasi input kosong
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
        
        # Query untuk mencari user
        query = """
            SELECT id_user, nama_lengkap, role, status 
            FROM users 
            WHERE username = %s AND password = %s
        """
        cursor.execute(query, (username, password))
        user = cursor.fetchone()
        
        # Cek hasil login
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
            # Login berhasil - Tampilkan dashboard berdasarkan role
            if user['role'] == 'admin':
                # Admin masuk ke dashboard admin
                admin_system = AdminMenuSystem(user)
                admin_system.show_main_menu()
            else:
                # Kasir masuk ke menu kasir (bisa diimplementasikan nanti)
                print_header(f"WELCOME KASIR - {user['nama_lengkap']}")
                print("Menu untuk kasir akan diimplementasikan...")
                pause()
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
