import psycopg2
from psycopg2 import Error
import os
from datetime import datetime, timedelta
import sys
import traceback


# KONEKSI DATABASE


def koneksi_database():
    try:
        koneksi = psycopg2.connect(
            host="localhost",
            database="PROJEK AKHIR BASDA ALGO",
            user="postgres",
            password="password_baru",
            port= "5433",
            connect_timeout=10
        )
        return koneksi
    except Error as e:
        print(f"Error koneksi ke database: {e}")
        traceback.print_exc()
        return None


def tutup_koneksi(koneksi, kursor=None):
    if kursor:
        kursor.close()
    if koneksi:
        koneksi.close()
    


# FUNGSI UTILITAS


def bersihkan_layar():
    os.system('cls' if os.name == 'nt' else 'clear')


def jeda():
    input("\nTekan Enter untuk melanjutkan...")


def tampilkan_header(judul):
    bersihkan_layar()
    print("=" * 80)
    print(f"{judul.center(80)}")
    print("=" * 80)
    print()


def format_mata_uang(jumlah):
    return f"Rp {jumlah:,.2f}"


def validasi_input_float(prompt):
    while True:
        try:
            nilai = float(input(prompt).strip())
            if nilai < 0:
                print("Nilai tidak boleh negatif!")
                continue
            return nilai
        except ValueError:
            print("Input harus berupa angka!")


def validasi_input_integer(prompt):
    while True:
        try:
            nilai = int(input(prompt).strip())
            if nilai < 0:
                print("Nilai tidak boleh negatif!")
                continue
            return nilai
        except ValueError:
            print("Input harus berupa bilangan bulat!")


# FUNGSI KATALOG


def lihat_katalog_produk(filter_tersedia=True):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return None
    
    try:
        with koneksi.cursor() as kursor:
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
            
            kursor.execute(query)
            katalog = kursor.fetchall()
            return katalog
            
    except Error as e:
        print(f"Error mengambil data katalog: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_katalog(filter_tersedia=True, untuk_admin=False):
    katalog = lihat_katalog_produk(filter_tersedia)
    
    if not katalog:
        print("Tidak ada data katalog yang ditemukan!")
        return
    
    if untuk_admin:
        tampilkan_header("KATALOG PRODUK TOKO")
    else:
        tampilkan_header("KATALOG PRODUK TOKO")
    
    total_produk = len(katalog)
    produk_tersedia = len([p for p in katalog if p[4] > 0])
    produk_menipis = len([p for p in katalog if p[4] <= 5 and p[4] > 0])
    produk_habis = len([p for p in katalog if p[4] == 0])
    
    print("STATISTIK KATALOG:")
    print(f"  Total Produk    : {total_produk}")
    print(f"  Tersedia        : {produk_tersedia}")
    print(f"  Stok Menipis    : {produk_menipis}")
    print(f"  Stok Habis      : {produk_habis}")
    print()
    
    kategori_dict = {}
    for item in katalog:
        kategori = item[2]
        if kategori not in kategori_dict:
            kategori_dict[kategori] = []
        kategori_dict[kategori].append(item)
    
    for kategori, items in kategori_dict.items():
        print(f"\n{'='*80}")
        print(f"KATEGORI: {kategori.upper()}")
        print(f"{'='*80}")
        print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12} {'Status':<15}")
        print(f"{'-'*80}")
        
        for item in items:
            id_katalog, nama_produk, _, satuan, stok, harga, _, status_stok = item
            print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_mata_uang(harga):<12} {status_stok:<15}")
    
    print(f"\n{'='*80}")
    return katalog


def lihat_ringkasan_stok():
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return None, None, None
    
    try:
        with koneksi.cursor() as kursor:
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
            kursor.execute(query)
            ringkasan_stok = kursor.fetchall()
            
            query_stok_menipis = """
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
            kursor.execute(query_stok_menipis)
            stok_menipis = kursor.fetchall()
            
            query_stok_habis = """
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
            kursor.execute(query_stok_habis)
            stok_habis = kursor.fetchall()
            
            return ringkasan_stok, stok_menipis, stok_habis
            
    except Error as e:
        print(f"Error mengambil ringkasan stok: {e}")
        traceback.print_exc()
        return None, None, None
    finally:
        koneksi.close()


def tampilkan_ringkasan_stok():
    ringkasan_stok, stok_menipis, stok_habis = lihat_ringkasan_stok()
    
    tampilkan_header("RINGKASAN STOK TOKO")
    
    if not ringkasan_stok:
        print("Tidak ada data stok yang ditemukan!")
        return
    
    print("RINGKASAN STOK PER KATEGORI:")
    print(f"{'='*80}")
    print(f"{'Kategori':<15} {'Total Produk':<12} {'Total Stok':<12} {'Habis':<8} {'Menipis':<10} {'Nilai Stok':<15}")
    print(f"{'-'*80}")
    
    total_nilai_stok = 0
    for kategori in ringkasan_stok:
        nama_kategori, total_produk, total_stok, habis, menipis, nilai_stok = kategori
        total_nilai_stok += nilai_stok if nilai_stok else 0
        print(f"{nama_kategori:<15} {total_produk:<12} {total_stok:<12.2f} {habis:<8} {menipis:<10} {format_mata_uang(nilai_stok) if nilai_stok else 'Rp 0':<15}")
    
    print(f"{'='*80}")
    print(f"TOTAL NILAI STOK: {format_mata_uang(total_nilai_stok)}")
    print()
    
    print("PRODUK DENGAN STOK MENIPIS (<= 5):")
    if stok_menipis:
        print(f"{'-'*60}")
        print(f"{'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12}")
        print(f"{'-'*60}")
        for produk in stok_menipis:
            print(f"{produk[0]:<20} {produk[1]:<10} {produk[2]:<8} {format_mata_uang(produk[3]):<12}")
    else:
        print("  Tidak ada produk dengan stok menipis")
    print()
    
    print("PRODUK HABIS:")
    if stok_habis:
        print(f"{'-'*50}")
        print(f"{'Nama Produk':<20} {'Satuan':<8} {'Harga':<12}")
        print(f"{'-'*50}")
        for produk in stok_habis:
            print(f"{produk[0]:<20} {produk[1]:<8} {format_mata_uang(produk[2]):<12}")
    else:
        print("  Tidak ada produk yang habis")
    
    print(f"\n{'='*80}")


# FUNGSI MANAJEMEN PRODUK


def lihat_semua_produk():
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return None
    
    try:
        with koneksi.cursor() as kursor:
            query = """
                SELECT 
                    id_produk, nama_produk, kategori, satuan, status,
                    created_at, updated_at
                FROM produk 
                ORDER BY id_produk
            """
            kursor.execute(query)
            produk_list = kursor.fetchall()
            return produk_list
            
    except Error as e:
        print(f"Error mengambil data produk: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_produk():
    produk_list = lihat_semua_produk()
    
    if not produk_list:
        print("Tidak ada data produk yang ditemukan!")
        return
    
    tampilkan_header("DAFTAR SEMUA PRODUK")
    
    print(f"{'ID':<5} {'Nama Produk':<20} {'Kategori':<10} {'Satuan':<8} {'Status':<10}")
    print(f"{'-'*60}")
    
    for produk in produk_list:
        id_produk, nama_produk, kategori, satuan, status, created_at, updated_at = produk
        print(f"{id_produk:<5} {nama_produk:<20} {kategori:<10} {satuan:<8} {status:<10}")
    
    print(f"{'-'*60}")
    print(f"Total produk: {len(produk_list)}")
    return produk_list


def tambah_produk(nama_produk, kategori, satuan):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:  # Cek apakah produk sudah ada
            kursor.execute("SELECT id_produk FROM produk WHERE nama_produk = %s", (nama_produk,))
            existing = kursor.fetchone()
            
            if existing:
                print(f"Produk '{nama_produk}' sudah ada dalam database!")
                return False
        
            if not nama_produk or not kategori or not satuan: # Validasi input tidak boleh kosong
                print("Semua field harus diisi!")
                return False
            
            # Tambah produk
            kursor.execute("""
                INSERT INTO produk (nama_produk, kategori, satuan)
                VALUES (%s, %s, %s)
                RETURNING id_produk
            """, (nama_produk, kategori, satuan))
            
            id_produk = kursor.fetchone()[0]
            
            # Otomatis buat entry di katalog dengan stok 0
            kursor.execute("""
                INSERT INTO katalog (id_produk, stok_tersedia, harga_awal, harga_jual, status, tanggal_input)
                VALUES (%s, 0, 0, 0, 'habis', CURRENT_DATE)
            """, (id_produk,))
            
            koneksi.commit()
            print("Produk berhasil ditambahkan dan entry katalog dibuat!")
            return True
            
    except Error as e:
        print(f"Error menambah produk: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def edit_produk(id_produk, nama_produk=None, kategori=None, satuan=None):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
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
                kursor.execute(query, values)
                koneksi.commit()
                print("Produk berhasil diupdate!")
                return True
            else:
                print("Tidak ada data yang diubah")
                return False
                
    except Error as e:
        print(f"Error mengedit produk: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def nonaktifkan_produk(id_produk):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kursor.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
            produk = kursor.fetchone()
            
            if not produk:
                print("Produk tidak ditemukan!")
                return False
            
            kursor.execute("UPDATE produk SET status = 'nonaktif' WHERE id_produk = %s", (id_produk,))
            koneksi.commit()
            print(f"Produk '{produk[0]}' berhasil dinonaktifkan!")
            return True
            
    except Error as e:
        print(f"Error menonaktifkan produk: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def aktifkan_produk(id_produk):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kursor.execute("SELECT nama_produk FROM produk WHERE id_produk = %s", (id_produk,))
            produk = kursor.fetchone()
            
            if not produk:
                print("Produk tidak ditemukan!")
                return False
            
            kursor.execute("UPDATE produk SET status = 'aktif' WHERE id_produk = %s", (id_produk,))
            koneksi.commit()
            print(f"Produk '{produk[0]}' berhasil diaktifkan!")
            return True
            
    except Error as e:
        print(f"Error mengaktifkan produk: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


# FUNGSI MANAJEMEN STOK


def tambah_stok(id_katalog, jumlah, harga_awal, harga_jual):     #Menambah stok produk - DENGAN TRIGGER"""
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    try:
        with koneksi.cursor() as kursor:
            kursor.execute(
                "SELECT id_katalog, stok_tersedia FROM katalog WHERE id_katalog = %s",
                (id_katalog,)
            )
            existing = kursor.fetchone()


            if existing:
                stok_baru = existing[1] + jumlah


                kursor.execute("""  
                    UPDATE katalog
                    SET stok_tersedia = %s,
                        harga_awal = %s,
                        harga_jual = %s
                    WHERE id_katalog = %s
                """, (stok_baru, harga_awal, harga_jual, existing[0]))
            else:
                kursor.execute("""
                    INSERT INTO katalog (id_produk, stok_tersedia, harga_awal, harga_jual, tanggal_input)
                    SELECT id_produk, %s, %s, %s, CURRENT_DATE
                    FROM produk WHERE id_produk = %s
                """, (jumlah, harga_awal, harga_jual, id_katalog))


        koneksi.commit()
        print("Stok berhasil ditambahkan!")
        return True


    except Error as e:
        print(f"Error menambah stok: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False


    finally:
        koneksi.close()


def update_harga(id_katalog, harga_jual):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kursor.execute("""
                UPDATE katalog SET harga_jual = %s 
                WHERE id_katalog = %s
            """, (harga_jual, id_katalog))
            koneksi.commit()
            print("Harga berhasil diupdate!")
            return True
            
    except Error as e:
        print(f"Error mengedit harga: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


from decimal import Decimal, InvalidOperation
from datetime import date


from decimal import Decimal, InvalidOperation
from datetime import date


def tambah_produk_busuk(id_katalog, qty_busuk_raw, id_pengguna):
    try:
        qty_busuk = Decimal(str(qty_busuk_raw))
    except (InvalidOperation, ValueError):
        print("Jumlah produk busuk tidak valid.")
        return False


    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke DB!")
        return False


    try:
        with koneksi.cursor() as kursor:
            kursor.execute("""
                SELECT k.stok_tersedia, p.nama_produk, p.satuan, k.harga_awal
                FROM katalog k
                JOIN produk p ON k.id_produk = p.id_produk
                WHERE k.id_katalog = %s
                FOR UPDATE
            """, (id_katalog,))
            
            baris = kursor.fetchone()
            if not baris:
                print("Katalog tidak ditemukan.")
                return False


            stok_sebelum = Decimal(baris[0])
            nama_produk = baris[1]
            satuan = baris[2]
            harga_awal = Decimal(baris[3])


            if qty_busuk <= 0:
                print("Jumlah busuk harus > 0.")
                return False


            if qty_busuk > stok_sebelum:
                print("Jumlah busuk melebihi stok!")
                return False


            stok_sesudah = stok_sebelum - qty_busuk


            # Status sesuai stok:
            if stok_sesudah == 0:
                status_str = 'habis'
            elif stok_sesudah <= Decimal('5'):
                status_str = 'menipis'
            else:
                status_str = 'tersedia'


            nilai_kerugian = qty_busuk * harga_awal


            # INSERT tanpa kolom keterangan
            kursor.execute("""
                INSERT INTO produk_busuk 
                (id_katalog, nama_produk, qty_busuk, satuan, nilai_kerugian,
                 tanggal_pencatatan, id_user)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
            """, (id_katalog, nama_produk, qty_busuk, satuan,
                  nilai_kerugian, date.today(), id_pengguna))


            # Log stok
            kursor.execute("""
                INSERT INTO log_stok
                (id_katalog, tipe_perubahan, jumlah_perubahan,
                 stok_sebelum, stok_sesudah, keterangan, id_user)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
            """, (id_katalog, 'produk_busuk', -qty_busuk,
                  stok_sebelum, stok_sesudah, f"Produk busuk: {nama_produk}", id_pengguna))


            # Update katalog
            kursor.execute("""
                UPDATE katalog
                SET stok_tersedia = %s,
                    status = %s
                WHERE id_katalog = %s
            """, (stok_sesudah, status_str, id_katalog))


        koneksi.commit()
        print("Produk busuk berhasil dicatat.")
        return True


    except Exception as e:
        koneksi.rollback()
        print(f"Error mencatat produk busuk: {e}")
        traceback.print_exc()  # Tambahkan ini untuk debug detail
        return False


    finally:
        koneksi.close()


def lihat_katalog_dengan_harga():
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return None
    
    try:
        with koneksi.cursor() as kursor:
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
            kursor.execute(query)
            katalog = kursor.fetchall()
            return katalog
            
    except Error as e:
        print(f"Error mengambil data katalog: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


# FUNGSI MANAJEMEN PENGGUNA


def lihat_semua_pengguna():
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return None
    
    try:
        with koneksi.cursor() as kursor:
            query = """
                SELECT 
                    id_user, username, nama_lengkap, role, status,
                    created_at, updated_at
                FROM users 
                ORDER BY id_user
            """
            kursor.execute(query)
            daftar_pengguna = kursor.fetchall()
            return daftar_pengguna
            
    except Error as e:
        print(f"Error mengambil data pengguna: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_pengguna():
    daftar_pengguna = lihat_semua_pengguna()
    
    if not daftar_pengguna:
        print("Tidak ada data pengguna yang ditemukan!")
        return
    
    tampilkan_header("DAFTAR SEMUA PENGGUNA")
    
    print(f"{'ID':<5} {'Username':<15} {'Nama Lengkap':<20} {'Role':<10} {'Status':<10}")
    print(f"{'-'*65}")
    
    for pengguna in daftar_pengguna:
        id_pengguna, username, nama_lengkap, role, status, created_at, updated_at = pengguna
        print(f"{id_pengguna:<5} {username:<15} {nama_lengkap:<20} {role:<10} {status:<10}")
    
    print(f"{'-'*65}")
    print(f"Total pengguna: {len(daftar_pengguna)}")
    return daftar_pengguna


def tambah_pengguna(username, password, nama_lengkap, role):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            if not username or not password or not nama_lengkap or not role:
                print("Semua field harus diisi!")
                return False
            
            if role.lower() not in ['admin', 'kasir']:
                print("Role harus 'admin' atau 'kasir'!")
                return False
            
            kursor.execute("SELECT id_user FROM users WHERE username = %s", (username,))
            existing = kursor.fetchone()
            
            if existing:
                print(f"Username '{username}' sudah digunakan!")
                return False
            
            kursor.execute("""
                INSERT INTO users (username, password, nama_lengkap, role)
                VALUES (%s, %s, %s, %s)
            """, (username, password, nama_lengkap, role))
            koneksi.commit()
            print("Pengguna berhasil ditambahkan!")
            return True
            
    except Error as e:
        print(f"Error menambah pengguna: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def edit_pengguna(id_pengguna, username=None, password=None, nama_lengkap=None, role=None):
    """Mengedit pengguna"""
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
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
                if role.lower() not in ['admin', 'kasir']:
                    print("Role harus 'admin' atau 'kasir'!")
                    return False
                update_fields.append("role = %s")
                values.append(role)
            
            if update_fields:
                values.append(id_pengguna)
                query = f"UPDATE users SET {', '.join(update_fields)} WHERE id_user = %s"
                kursor.execute(query, values)
                koneksi.commit()
                print("Pengguna berhasil diupdate!")
                return True
            else:
                print("Tidak ada data yang diubah")
                return False
                
    except Error as e:
        print(f"Error mengedit pengguna: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def nonaktifkan_pengguna(id_pengguna):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kursor.execute("SELECT username FROM users WHERE id_user = %s", (id_pengguna,))
            pengguna = kursor.fetchone()
            
            if not pengguna:
                print("Pengguna tidak ditemukan!")
                return False
            
            kursor.execute("UPDATE users SET status = 'nonaktif' WHERE id_user = %s", (id_pengguna,))
            koneksi.commit()
            print(f"Pengguna '{pengguna[0]}' berhasil dinonaktifkan!")
            return True
            
    except Error as e:
        print(f"Error menonaktifkan pengguna: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def aktifkan_pengguna(id_pengguna):
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kursor.execute("SELECT username FROM users WHERE id_user = %s", (id_pengguna,))
            pengguna = kursor.fetchone()
            
            if not pengguna:
                print("Pengguna tidak ditemukan!")
                return False
            
            kursor.execute("UPDATE users SET status = 'aktif' WHERE id_user = %s", (id_pengguna,))
            koneksi.commit()
            print(f"Pengguna '{pengguna[0]}' berhasil diaktifkan!")
            return True
            
    except Error as e:
        print(f"Error mengaktifkan pengguna: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


# FUNGSI LAPORAN


def dapatkan_penjualan_hari_ini():
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
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
            kursor.execute(query)
            penjualan = kursor.fetchall()
            return penjualan
            
    except Error as e:
        print(f"Error mengambil data penjualan: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_penjualan_hari_ini():
    penjualan = dapatkan_penjualan_hari_ini()
    
    tampilkan_header("DATA PENJUALAN HARI INI")
    
    if not penjualan:
        print("Tidak ada transaksi hari ini!")
        return
    
    total_penjualan = 0
    total_transaksi = len(penjualan)
    
    print(f"{'Kode Transaksi':<20} {'Tanggal':<20} {'Kasir':<15} {'Total':<12} {'Bayar':<12} {'Kembali':<12}")
    print(f"{'-'*95}")
    
    for penjualan_item in penjualan:
        kode, tanggal, kasir, total, bayar, kembali = penjualan_item
        total_penjualan += total
        waktu = tanggal.strftime("%d/%m/%Y %H:%M")
        print(f"{kode:<20} {waktu:<20} {kasir:<15} {format_mata_uang(total):<12} {format_mata_uang(bayar):<12} {format_mata_uang(kembali):<12}")
    
    print(f"{'-'*95}")
    print(f"Total Transaksi : {total_transaksi}")
    print(f"Total Penjualan : {format_mata_uang(total_penjualan)}")


def dapatkan_semua_transaksi():
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
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
                ORDER BY t.tanggal_transaksi DESC
            """
            kursor.execute(query)
            transaksi_list = kursor.fetchall()
            return transaksi_list
                
    except Error as e:
        print(f"Error mengambil riwayat transaksi: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_riwayat_transaksi():
    transaksi_list = dapatkan_semua_transaksi()
    
    tampilkan_header("RIWAYAT TRANSAKSI")
    
    if not transaksi_list:
        print("Tidak ada transaksi!")
        return
    
    total_penjualan = 0
    total_transaksi = len(transaksi_list)
    
    print(f"\nRIWAYAT TRANSAKSI:")
    print(f"{'Kode Transaksi':<20} {'Tanggal':<20} {'Kasir':<15} {'Total':<12} {'Bayar':<12} {'Kembali':<12}")
    print(f"{'-'*95}")
    
    for transaksi in transaksi_list:
        kode, tanggal, kasir, total, bayar, kembali = transaksi
        total_penjualan += total
        waktu = tanggal.strftime("%d/%m/%Y %H:%M")
        print(f"{kode:<20} {waktu:<20} {kasir:<15} {format_mata_uang(total):<12} {format_mata_uang(bayar):<12} {format_mata_uang(kembali):<12}")
    
    print(f"{'-'*95}")
    print(f"Total Transaksi : {total_transaksi}")
    print(f"Total Penjualan : {format_mata_uang(total_penjualan)}")


def dapatkan_penjualan_mingguan():
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
            query = """
                SELECT 
                    DATE_TRUNC('week', t.tanggal_transaksi) as minggu,
                    COUNT(*) as jumlah_transaksi,
                    SUM(t.total_belanja) as total_penjualan,
                    AVG(t.total_belanja) as rata_rata_transaksi,
                    SUM(dt.qty * dt.harga_modal) as total_modal,
                    (SUM(t.total_belanja) - SUM(dt.qty * dt.harga_modal)) as laba_bersih
                FROM transaksi t
                JOIN detail_transaksi dt ON t.id_transaksi = dt.id_transaksi
                WHERE t.tanggal_transaksi >= CURRENT_DATE - INTERVAL '4 weeks'
                GROUP BY DATE_TRUNC('week', t.tanggal_transaksi)
                ORDER BY minggu DESC
            """
            kursor.execute(query)
            penjualan_mingguan = kursor.fetchall()
            return penjualan_mingguan
                
    except Error as e:
        print(f"Error mengambil data mingguan: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_penjualan_mingguan():
    penjualan_mingguan = dapatkan_penjualan_mingguan()
    
    tampilkan_header("LAPORAN PENJUALAN MINGGUAN")
    
    if not penjualan_mingguan:
        print("Tidak ada data penjualan mingguan!")
        return
    
    print(f"{'Minggu':<20} {'Jml Transaksi':<15} {'Total Penjualan':<20} {'Rata2 Transaksi':<20} {'Laba Bersih':<20}")
    print(f"{'-'*95}")
    
    total_penjualan = 0
    total_laba = 0
    
    for minggu_item in penjualan_mingguan:
        minggu, jumlah, total, rata2, modal, laba = minggu_item
        total_penjualan += total if total else 0
        total_laba += laba if laba else 0
        minggu_str = minggu.strftime("%d/%m/%Y")
        print(f"{minggu_str:<20} {jumlah:<15} {format_mata_uang(total):<20} {format_mata_uang(rata2):<20} {format_mata_uang(laba):<20}")
    
    print(f"{'-'*95}")
    print(f"{'TOTAL':<55} {format_mata_uang(total_penjualan):<20} {format_mata_uang(total_laba):<20}")


def dapatkan_produk_terlaris(periode='all', limit=10):
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
            if periode == '7hari':
                interval = '7 days'
                judul = "7 HARI TERAKHIR"
            else:  # 'all'
                interval = '1000 days'  # Semua waktu
                judul = "SEMUA WAKTU"
            
            query = f"""
                SELECT 
                    dt.nama_produk,
                    SUM(dt.qty) as total_terjual,
                    p.satuan,
                    SUM(dt.subtotal) as total_penjualan,
                    AVG(dt.harga_satuan) as harga_rata
                FROM detail_transaksi dt
                JOIN transaksi t ON dt.id_transaksi = t.id_transaksi
                JOIN produk p ON dt.nama_produk = p.nama_produk
                WHERE t.tanggal_transaksi >= CURRENT_DATE - INTERVAL '{interval}'
                GROUP BY dt.nama_produk, p.satuan
                ORDER BY total_terjual DESC
                LIMIT %s
            """
            kursor.execute(query, (limit,))
            produk_terlaris = kursor.fetchall()
            return produk_terlaris, judul
                
    except Error as e:
        print(f"Error mengambil produk terlaris: {e}")
        traceback.print_exc()
        return None, None
    finally:
        koneksi.close()


def tampilkan_produk_terlaris():
    tampilkan_header("PRODUK TERLARIS")
    
    print("Pilih periode:")
    print("1. 7 hari terakhir")
    print("2. Semua waktu")
    
    pilihan = input("Pilih [1-2]: ").strip()
    
    if pilihan == '1':
        periode = '7hari'
    else:
        periode = 'all'
    
    produk_terlaris, judul = dapatkan_produk_terlaris(periode)
    
    if not produk_terlaris:
        print(f"Tidak ada data produk terlaris {judul.lower()}!")
        return
    
    tampilkan_header(f"PRODUK TERLARIS ({judul})")
    
    print(f"{'Nama Produk':<20} {'Terjual':<10} {'Satuan':<8} {'Total Penjualan':<20} {'Harga Rata':<15}")
    print(f"{'-'*75}")
    
    for produk in produk_terlaris:
        nama, terjual, satuan, total, harga_rata = produk
        print(f"{nama:<20} {terjual:<10} {satuan:<8} {format_mata_uang(total):<20} {format_mata_uang(harga_rata):<15}")


def dapatkan_produk_kurang_laku(periode='all', limit=10):
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
            if periode == '7hari':
                interval = '7 days'
                judul = "7 HARI TERAKHIR"
            else:  # 'all'
                interval = '1000 days'  # Semua waktu
                judul = "SEMUA WAKTU"
            
            query = f"""
                SELECT 
                    p.nama_produk,
                    p.satuan,
                    COALESCE(k.stok_tersedia, 0) as stok_tersedia,
                    COALESCE(k.harga_jual, 0) as harga_jual,
                    COALESCE(SUM(dt.qty), 0) as total_terjual
                FROM produk p
                LEFT JOIN katalog k ON p.id_produk = k.id_produk
                LEFT JOIN detail_transaksi dt ON p.nama_produk = dt.nama_produk
                    AND dt.created_at >= CURRENT_DATE - INTERVAL '{interval}'
                WHERE p.status = 'aktif'
                GROUP BY p.nama_produk, p.satuan, k.stok_tersedia, k.harga_jual
                HAVING COALESCE(SUM(dt.qty), 0) <= 5
                ORDER BY total_terjual ASC, p.nama_produk
                LIMIT %s
            """
            kursor.execute(query, (limit,))
            produk_kurang_laku = kursor.fetchall()
            return produk_kurang_laku, judul
                
    except Error as e:
        print(f"Error mengambil produk kurang laku: {e}")
        traceback.print_exc()
        return None, None
    finally:
        koneksi.close()


def tampilkan_produk_kurang_laku():
    tampilkan_header("PRODUK KURANG LAKU")
    
    print("Pilih periode:")
    print("1. 7 hari terakhir")
    print("2. Semua waktu")
    
    pilihan = input("Pilih [1-2]: ").strip()
    
    if pilihan == '1':
        periode = '7hari'
    else:
        periode = 'all'
    
    produk_kurang_laku, judul = dapatkan_produk_kurang_laku(periode)
    
    if not produk_kurang_laku:
        print(f"Tidak ada produk kurang laku {judul.lower()}!")
        return
    
    tampilkan_header(f"PRODUK KURANG LAKU ({judul})")
    
    print(f"{'Nama Produk':<20} {'Satuan':<8} {'Stok':<10} {'Harga':<12} {'Terjual':<10}")
    print(f"{'-'*65}")
    
    for produk in produk_kurang_laku:
        nama, satuan, stok, harga, terjual = produk
        print(f"{nama:<20} {satuan:<8} {stok:<10} {format_mata_uang(harga):<12} {terjual:<10}")


# FUNGSI STRUK TRANSAKSI


def dapatkan_data_struk(kode_transaksi=None, id_transaksi=None):
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
            if kode_transaksi:
                query = """
                    SELECT 
                        t.id_transaksi, t.kode_transaksi, t.tanggal_transaksi, 
                        u.nama_lengkap as kasir, t.total_belanja, t.uang_dibayar, t.kembalian
                    FROM transaksi t
                    JOIN users u ON t.id_user = u.id_user
                    WHERE t.kode_transaksi = %s
                """
                kursor.execute(query, (kode_transaksi,))
            else:
                query = """
                    SELECT 
                        t.id_transaksi, t.kode_transaksi, t.tanggal_transaksi, 
                        u.nama_lengkap as kasir, t.total_belanja, t.uang_dibayar, t.kembalian
                    FROM transaksi t
                    JOIN users u ON t.id_user = u.id_user
                    WHERE t.id_transaksi = %s
                """
                kursor.execute(query, (id_transaksi,))
            
            transaksi = kursor.fetchone()
            
            if not transaksi:
                return None
            
            # Ambil detail transaksi
            query_detail = """
                SELECT nama_produk, qty, satuan, harga_satuan, subtotal
                FROM detail_transaksi
                WHERE id_transaksi = %s
                ORDER BY id_detail
            """
            kursor.execute(query_detail, (transaksi[0],))
            detail = kursor.fetchall()
            
            return transaksi, detail
            
    except Error as e:
        print(f"Error mengambil data struk: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def cetak_struk():
    tampilkan_header("CETAK STRUK TRANSAKSI")
    
    print("Pilih cara pencarian:")
    print("1. Cari dengan Kode Transaksi")
    print("2. Lihat transaksi hari ini")
    
    pilihan = input("Pilih [1-2]: ").strip()
    
    if pilihan == '1':
        kode_transaksi = input("Masukkan Kode Transaksi: ").strip()
        data = dapatkan_data_struk(kode_transaksi=kode_transaksi)
    elif pilihan == '2':
        koneksi = koneksi_database()
        if not koneksi:
            jeda()
            return
        
        try:
            with koneksi.cursor() as kursor:
                query = """
                    SELECT kode_transaksi, tanggal_transaksi, total_belanja
                    FROM transaksi
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                    ORDER BY tanggal_transaksi DESC
                    LIMIT 10
                """
                kursor.execute(query)
                transaksi_list = kursor.fetchall()
                
                if not transaksi_list:
                    print("Tidak ada transaksi hari ini!")
                    jeda()
                    return
                
                print("\nTRANSAKSI HARI INI:")
                print(f"{'No':<3} {'Kode Transaksi':<20} {'Waktu':<20} {'Total':<12}")
                print(f"{'-'*60}")
                
                for i, trans in enumerate(transaksi_list, 1):
                    kode, waktu, total = trans
                    waktu_str = waktu.strftime("%H:%M")
                    print(f"{i:<3} {kode:<20} {waktu_str:<20} {format_mata_uang(total):<12}")
                
                print(f"{'-'*60}")
                
                try:
                    pilihan_nomor = int(input("\nPilih nomor transaksi: ").strip())
                    if 1 <= pilihan_nomor <= len(transaksi_list):
                        kode_transaksi = transaksi_list[pilihan_nomor-1][0]
                        data = dapatkan_data_struk(kode_transaksi=kode_transaksi)
                    else:
                        print("Pilihan tidak valid!")
                        jeda()
                        return
                except ValueError:
                    print("Input harus angka!")
                    jeda()
                    return
                
        except Error as e:
            print(f"Error: {e}")
            traceback.print_exc()
            jeda()
            return
        finally:
            koneksi.close()
    else:
        print("Pilihan tidak valid!")
        jeda()
        return
    
    if not data:
        print("Transaksi tidak ditemukan!")
        jeda()
        return
    
    transaksi, detail = data
    id_transaksi, kode, tanggal, kasir, total, bayar, kembali = transaksi
    
    # Cetak struk
    tampilkan_header("STRUK TRANSAKSI")
    
    print(f"Kode Transaksi : {kode}")
    print(f"Tanggal        : {tanggal.strftime('%d/%m/%Y %H:%M:%S')}")
    print(f"Kasir          : {kasir}")
    print(f"{'-'*60}")
    print(f"{'Produk':<20} {'Qty':<6} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
    print(f"{'-'*60}")
    
    for item in detail:
        nama, qty, satuan, harga, subtotal = item
        print(f"{nama:<20} {qty:<6} {satuan:<8} {format_mata_uang(harga):<12} {format_mata_uang(subtotal):<12}")
    
    print(f"{'-'*60}")
    print(f"{'TOTAL':<34} {format_mata_uang(total):<12}")
    print(f"{'UANG BAYAR':<34} {format_mata_uang(bayar):<12}")
    print(f"{'KEMBALIAN':<34} {format_mata_uang(kembali):<12}")
    print(f"{'-'*60}")
    print("Terima kasih atas kunjungan Anda!")
    
    # Tanya cetak ulang
    cetak_ulang = input("\nCetak struk lagi? (y/n): ").strip().lower()
    if cetak_ulang == 'y':
        cetak_struk()


# FUNGSI TRANSAKSI KASIR


def generate_kode_transaksi():
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
    return f"TRX-{timestamp}"


def dapatkan_produk_by_id(id_katalog):
    koneksi = koneksi_database()
    if not koneksi:
        return None
    
    try:
        with koneksi.cursor() as kursor:
            query = """
                SELECT 
                    k.id_katalog, p.nama_produk, p.satuan, k.stok_tersedia, k.harga_jual, k.harga_awal
                FROM katalog k
                JOIN produk p ON k.id_produk = p.id_produk
                WHERE k.id_katalog = %s AND p.status = 'aktif' AND k.stok_tersedia > 0
            """
            kursor.execute(query, (id_katalog,))
            produk = kursor.fetchone()
            return produk
            
    except Error as e:
        print(f"Error mengambil data produk: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def proses_penjualan(data_pengguna):
    tampilkan_header("TRANSAKSI PENJUALAN")
    
    produk_list = lihat_katalog_produk(filter_tersedia=True)
    if not produk_list:
        print("Tidak ada produk yang tersedia untuk dijual!")
        jeda()
        return
    
    print("PRODUK TERSEDIA UNTUK DIJUAL:")
    print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Satuan':<8} {'Harga':<12}")
    print(f"{'-'*60}")
    
    for produk in produk_list:
        id_katalog, nama_produk, _, satuan, stok, harga, _, _ = produk
        print(f"{id_katalog:<5} {nama_produk:<20} {stok:<10} {satuan:<8} {format_mata_uang(harga):<12}")
    
    print(f"{'-'*60}")
    
    keranjang = []
    total_belanja = 0
    
    print("\nMASUKKAN PRODUK YANG DIBELI:")
    print("Ketik 'selesai' untuk menyelesaikan transaksi")
    print("Ketik 'batal' untuk membatalkan transaksi")
    print()
    
    while True:
        try:
            input_id = input("Masukkan ID produk: ").strip()
            
            if input_id.lower() == 'selesai':
                break
            elif input_id.lower() == 'batal':
                print("Transaksi dibatalkan!")
                return
            
            id_katalog = int(input_id)
            produk = dapatkan_produk_by_id(id_katalog)
            
            if not produk:
                print("ID produk tidak valid atau produk tidak tersedia!")
                continue
            
            id_katalog, nama_produk, satuan, stok, harga_jual, harga_modal = produk
            
            print(f"\nProduk: {nama_produk}")
            print(f"Stok tersedia: {stok} {satuan}")
            print(f"Harga: {format_mata_uang(harga_jual)}")
            
            try:
                from decimal import Decimal
                qty = Decimal(input(f"Jumlah {satuan} yang dibeli: ").strip())
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
            
            item_keranjang = {
                'id_katalog': id_katalog,
                'nama_produk': nama_produk,
                'satuan': satuan,
                'qty': qty,
                'harga_satuan': harga_jual,
                'harga_modal': harga_modal,
                'subtotal': subtotal
            }
            
            keranjang.append(item_keranjang)
            total_belanja += subtotal
            
            print(f"Ditambahkan: {nama_produk} - {qty} {satuan} = {format_mata_uang(subtotal)}")
            print(f"Total sementara: {format_mata_uang(total_belanja)}")
            print()
            
        except ValueError:
            print("ID produk harus berupa angka!")
        except Exception as e:
            print(f"Error: {e}")
            traceback.print_exc()
    
    if not keranjang:
        print("Tidak ada produk dalam keranjang!")
        jeda()
        return
    
    # Tampilkan ringkasan belanja
    tampilkan_header("RINGKASAN BELANJA")
    
    print(f"{'No':<3} {'Nama Produk':<20} {'Qty':<8} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
    print(f"{'-'*70}")
    
    for i, item in enumerate(keranjang, 1):
        print(f"{i:<3} {item['nama_produk']:<20} {item['qty']:<8} {item['satuan']:<8} {format_mata_uang(item['harga_satuan']):<12} {format_mata_uang(item['subtotal']):<12}")
    
    print(f"{'-'*70}")
    print(f"{'TOTAL BELANJA:':<50} {format_mata_uang(total_belanja):<12}")
    
    try:
        from decimal import Decimal
        uang_dibayar = Decimal(input("\nMasukkan uang yang dibayar: ").strip())
    except ValueError:
        print("Jumlah uang harus berupa angka!")
        jeda()
        return
    
    if uang_dibayar < total_belanja:
        print("Uang yang dibayar kurang!")
        jeda()
        return
    
    kembalian = uang_dibayar - total_belanja
    
    print(f"\nTOTAL BELANJA: {format_mata_uang(total_belanja)}")
    print(f"UANG DIBAYAR : {format_mata_uang(uang_dibayar)}")
    print(f"KEMBALIAN    : {format_mata_uang(kembalian)}")
    
    konfirmasi = input("\nKonfirmasi transaksi? (y/n): ").strip().lower()
    if konfirmasi != 'y':
        print("Transaksi dibatalkan!")
        jeda()
        return
    
    sukses = simpan_transaksi(keranjang, total_belanja, uang_dibayar, kembalian, data_pengguna)
    if sukses:
        print("Transaksi berhasil disimpan!")
        cetak_struk_setelah_transaksi(keranjang, total_belanja, uang_dibayar, kembalian, data_pengguna)
    else:
        print("Gagal menyimpan transaksi!")
    
    jeda()


def simpan_transaksi(keranjang, total_belanja, uang_dibayar, kembalian, data_pengguna):
    koneksi = koneksi_database()
    if not koneksi:
        return False
    
    try:
        with koneksi.cursor() as kursor:
            kode_transaksi = generate_kode_transaksi()


            # Insert transaksi
            kursor.execute("""
                INSERT INTO transaksi (kode_transaksi, id_user, total_belanja, uang_dibayar, kembalian)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id_transaksi
            """, (kode_transaksi, data_pengguna['id_user'], total_belanja, uang_dibayar, kembalian))


            id_transaksi = kursor.fetchone()[0]


            for item in keranjang:
                # Ambil stok sebelum update
                kursor.execute("SELECT stok_tersedia FROM katalog WHERE id_katalog = %s", (item['id_katalog'],))
                stok_sebelum = kursor.fetchone()[0]


                # Hitung stok baru
                stok_baru = stok_sebelum - item['qty']


                # Insert detail transaksi
                kursor.execute("""
                    INSERT INTO detail_transaksi 
                    (id_transaksi, id_katalog, nama_produk, qty, satuan, harga_satuan, harga_modal, subtotal)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                """, (id_transaksi, item['id_katalog'], item['nama_produk'], item['qty'],
                      item['satuan'], item['harga_satuan'], item['harga_modal'], item['subtotal']))


                # Update HANYA stok_tersedia
                # TRIGGER akan otomatis set status yang benar!
                kursor.execute("""
                    UPDATE katalog
                    SET stok_tersedia = %s
                    WHERE id_katalog = %s
                """, (stok_baru, item['id_katalog']))


                # Insert log stok
                kursor.execute("""
                    INSERT INTO log_stok
                    (id_katalog, tipe_perubahan, jumlah_perubahan, stok_sebelum, stok_sesudah, id_user)
                    VALUES (%s, 'penjualan', %s, %s, %s, %s, %s)
                """, (item['id_katalog'], -item['qty'], stok_sebelum, stok_baru,
                      f"Penjualan transaksi {kode_transaksi}", data_pengguna['id_user']))


            koneksi.commit()
            return True


    except Error as e:
        print(f"Error menyimpan transaksi: {e}")
        traceback.print_exc()
        koneksi.rollback()
        return False
    finally:
        koneksi.close()


def cetak_struk_setelah_transaksi(keranjang, total_belanja, uang_dibayar, kembalian, data_pengguna):
    tampilkan_header("STRUK TRANSAKSI")
    
    kode_transaksi = generate_kode_transaksi()
    waktu_sekarang = datetime.now().strftime("%d/%m/%Y %H:%M:%S")
    
    print(f"Kode Transaksi : {kode_transaksi}")
    print(f"Tanggal        : {waktu_sekarang}")
    print(f"Kasir          : {data_pengguna['nama_lengkap']}")
    print(f"{'-'*60}")
    print(f"{'Produk':<20} {'Qty':<6} {'Satuan':<8} {'Harga':<12} {'Subtotal':<12}")
    print(f"{'-'*60}")
    
    for item in keranjang:
        print(f"{item['nama_produk']:<20} {item['qty']:<6} {item['satuan']:<8} {format_mata_uang(item['harga_satuan']):<12} {format_mata_uang(item['subtotal']):<12}")
    
    print(f"{'-'*60}")
    print(f"{'TOTAL':<34} {format_mata_uang(total_belanja):<12}")
    print(f"{'UANG BAYAR':<34} {format_mata_uang(uang_dibayar):<12}")
    print(f"{'KEMBALIAN':<34} {format_mata_uang(kembalian):<12}")
    print(f"{'-'*60}")
    print("Terima kasih atas kunjungan Anda!")
    
    cetak_ulang = input("\nCetak struk lagi? (y/n): ").strip().lower()
    if cetak_ulang == 'y':
        cetak_struk_setelah_transaksi(keranjang, total_belanja, uang_dibayar, kembalian, data_pengguna)


# FUNGSI DASHBOARD


def dapatkan_data_dashboard(data_pengguna):
    koneksi = koneksi_database()
    if not koneksi:
        return None
        
    try:
        with koneksi.cursor() as kursor:
            # Penjualan hari ini
            if data_pengguna['role'] == 'admin':
                kursor.execute("""
                    SELECT 
                        COALESCE(SUM(total_belanja), 0) as penjualan_hari_ini,
                        COUNT(id_transaksi) as total_transaksi_hari_ini
                    FROM transaksi 
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                """)
            else:
                kursor.execute("""
                    SELECT 
                        COALESCE(SUM(total_belanja), 0) as penjualan_hari_ini,
                        COUNT(id_transaksi) as total_transaksi_hari_ini
                    FROM transaksi 
                    WHERE DATE(tanggal_transaksi) = CURRENT_DATE
                    AND id_user = %s
                """, (data_pengguna['id_user'],))
            
            penjualan_hari_ini = kursor.fetchone()
            
            # Statistik stok
            kursor.execute("""
                SELECT 
                    COUNT(*) as total_produk,
                    COUNT(CASE WHEN k.stok_tersedia <= 5 AND k.stok_tersedia > 0 THEN 1 END) as stok_menipis,
                    COUNT(CASE WHEN k.stok_tersedia = 0 THEN 1 END) as stok_habis
                FROM katalog k
                JOIN produk p ON k.id_produk = p.id_produk 
                WHERE p.status = 'aktif'
            """)
            statistik_stok = kursor.fetchone()
            
            # Produk dengan stok menipis
            kursor.execute("""
                SELECT p.nama_produk, k.stok_tersedia, p.satuan
                FROM katalog k
                JOIN produk p ON k.id_produk = p.id_produk
                WHERE k.stok_tersedia <= 5 AND k.stok_tersedia > 0
                AND p.status = 'aktif'
                ORDER BY k.stok_tersedia ASC
                LIMIT 5
            """)
            stok_menipis = kursor.fetchall()
            
            # Produk habis
            kursor.execute("""
                SELECT p.nama_produk
                FROM katalog k
                JOIN produk p ON k.id_produk = p.id_produk
                WHERE k.stok_tersedia = 0
                AND p.status = 'aktif'
                LIMIT 5
            """)
            stok_habis = kursor.fetchall()
            
            return {
                'penjualan_hari_ini': penjualan_hari_ini[0] if penjualan_hari_ini else 0,
                'total_transaksi_hari_ini': penjualan_hari_ini[1] if penjualan_hari_ini else 0,
                'total_produk': statistik_stok[0] if statistik_stok else 0,
                'stok_menipis': statistik_stok[1] if statistik_stok else 0,
                'stok_habis': statistik_stok[2] if statistik_stok else 0,
                'alert_stok_menipis': stok_menipis,
                'alert_stok_habis': stok_habis
            }
            
    except Error as e:
        print(f"Error mengambil data dashboard: {e}")
        traceback.print_exc()
        return None
    finally:
        koneksi.close()


def tampilkan_dashboard(data_pengguna):
    if data_pengguna['role'] == 'admin':
        tampilkan_header("DASHBOARD ADMIN - SISTEM MANAJEMEN TOKO")
    else:
        tampilkan_header("DASHBOARD KASIR - SISTEM MANAJEMEN TOKO")
    
    print(f"Pengguna: {data_pengguna['nama_lengkap']} ({data_pengguna['role'].upper()})")
    print(f"Tanggal: {datetime.now().strftime('%d/%m/%Y %H:%M:%S')}")
    print()
    
    data = dapatkan_data_dashboard(data_pengguna)
    if not data:
        print("Gagal memuat data dashboard")
        return
    
    print("STATISTIK HARI INI:")
    print(f"  Total Penjualan    : {format_mata_uang(data['penjualan_hari_ini'])}")
    print(f"  Total Transaksi    : {data['total_transaksi_hari_ini']} transaksi")
    print()
    
    print("STATISTIK STOK:")
    print(f"  Total Produk Aktif : {data['total_produk']} produk")
    print(f"  Stok Menipis       : {data['stok_menipis']} produk")
    print(f"  Stok Habis         : {data['stok_habis']} produk")
    print()
    
    # Notifikasi stok menipis
    if data['alert_stok_menipis']:
        print("NOTIFIKASI STOK MENIPIS:")
        for i, produk in enumerate(data['alert_stok_menipis'], 1):
            print(f"  {i}. {produk[0]} - {produk[1]} {produk[2]}")
        print()
    
    # Notifikasi stok habis
    if data['alert_stok_habis']:
        print("NOTIFIKASI STOK HABIS:")
        for i, produk in enumerate(data['alert_stok_habis'], 1):
            print(f"  {i}. {produk[0]}")
        print()
    
    print("=" * 80)


# FUNGSI MENU


def menu_manajemen_produk():
    while True:
        tampilkan_header("MANAJEMEN PRODUK")
        
        print("1.  Lihat Semua Produk")
        print("2.  Tambah Produk Baru")
        print("3.  Edit Produk")
        print("4.  Nonaktifkan Produk")
        print("5.  Aktifkan Produk")
        print("6.  Kembali ke Menu Utama")
        
        pilihan = input("\nPilih menu [1-6]: ").strip()
        
        if pilihan == '1':
            tampilkan_produk()
            jeda()
        elif pilihan == '2':
            tampilkan_header("TAMBAH PRODUK BARU")
            nama = input("Nama Produk: ").strip()
            kategori = input("Kategori: ").strip()
            satuan = input("Satuan: ").strip()
            
            if tambah_produk(nama, kategori, satuan):
                print("Produk berhasil ditambahkan!")
            jeda()
        elif pilihan == '3':
            tampilkan_produk()
            try:
                id_produk = int(input("\nID Produk yang akan diedit: ").strip())
                
                print("\nBiarkan kosong jika tidak ingin mengubah")
                nama = input("Nama Produk baru: ").strip() or None
                kategori = input("Kategori baru: ").strip() or None
                satuan = input("Satuan baru: ").strip() or None
                
                edit_produk(id_produk, nama, kategori, satuan)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '4':
            tampilkan_produk()
            try:
                id_produk = int(input("\nID Produk yang akan dinonaktifkan: ").strip())
                nonaktifkan_produk(id_produk)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '5':
            tampilkan_produk()
            try:
                id_produk = int(input("\nID Produk yang akan diaktifkan: ").strip())
                aktifkan_produk(id_produk)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '6':
            break
        else:
            print("Pilihan tidak valid!")
            jeda()


def menu_manajemen_stok(pengguna_sekarang):
    while True:
        tampilkan_header("MANAJEMEN STOK")
        
        print("1.  Lihat Katalog dengan Harga")
        print("2.  Tambah Stok Produk")
        print("3.  Edit Harga Produk")
        print("4.  Catat Produk Busuk")
        print("5.  Kembali ke Menu Utama")
        
        pilihan = input("\nPilih menu [1-5]: ").strip()
        
        if pilihan == '1':
            katalog = lihat_katalog_dengan_harga()
            if katalog:
                tampilkan_header("KATALOG PRODUK DENGAN HARGA")
                print(f"{'ID':<5} {'Nama Produk':<20} {'Stok':<10} {'Harga Awal':<12} {'Harga Jual':<12} {'Margin':<12}")
                print(f"{'-'*75}")
                for item in katalog:
                    print(f"{item[0]:<5} {item[1]:<20} {item[4]:<10} {format_mata_uang(item[5]):<12} {format_mata_uang(item[6]):<12} {format_mata_uang(item[7]):<12}")
                print(f"{'-'*75}")
            jeda()
        elif pilihan == '2':
            tampilkan_katalog(filter_tersedia=False)
            try:
                id_katalog = int(input("\nID Katalog: ").strip())
                from decimal import Decimal
                jumlah = Decimal(input("Jumlah stok yang ditambah: ").strip())
                harga_awal = Decimal(input("Harga awal (modal): ").strip())
                harga_jual = Decimal(input("Harga jual: ").strip())
                
                if jumlah > 0 and harga_awal > 0 and harga_jual > 0:
                    tambah_stok(id_katalog, jumlah, harga_awal, harga_jual)
                else:
                    print("Nilai harus positif!")
            except ValueError:
                print("Input harus berupa angka!")
            jeda()
        elif pilihan == '3':
            tampilkan_katalog(filter_tersedia=False)
            try:
                id_katalog = int(input("\nID Katalog: ").strip())
                harga_jual = float(input("Harga jual baru: ").strip())
                
                if harga_jual > 0:
                    update_harga(id_katalog, harga_jual)
                else:
                    print("Harga harus positif!")
            except ValueError:
                print("Input harus berupa angka!")
            jeda()
        elif pilihan == '4':
            tampilkan_katalog(filter_tersedia=False)
            try:
                id_katalog = int(input("\nID Katalog: ").strip())


                from decimal import Decimal
                qty_busuk = Decimal(input("Jumlah produk busuk: ").strip())
                
                if qty_busuk > 0:
                    # Gunakan pengguna_sekarang yang sudah diterima sebagai parameter
                    tambah_produk_busuk(id_katalog, qty_busuk, pengguna_sekarang['id_user'])
                else:
                    print("Jumlah harus positif!")
            except ValueError:
                print("Input harus berupa angka!")
            jeda()
        elif pilihan == '5':
            break
        else:
            print("Pilihan tidak valid!")
            jeda()


def menu_manajemen_pengguna():
    while True:
        tampilkan_header("MANAJEMEN PENGGUNA")
        
        print("1.  Lihat Semua Pengguna")
        print("2.  Tambah Pengguna Baru")
        print("3.  Edit Pengguna")
        print("4.  Nonaktifkan Pengguna")
        print("5.  Aktifkan Pengguna")
        print("6.  Kembali ke Menu Utama")
        
        pilihan = input("\nPilih menu [1-6]: ").strip()
        
        if pilihan == '1':
            tampilkan_pengguna()
            jeda()
        elif pilihan == '2':
            tampilkan_header("TAMBAH PENGGUNA BARU")
            username = input("Username: ").strip()
            password = input("Password: ").strip()
            nama_lengkap = input("Nama Lengkap: ").strip()
            role = input("Role (admin/kasir): ").strip().lower()
            
            if tambah_pengguna(username, password, nama_lengkap, role):
                print("Pengguna berhasil ditambahkan!")
            jeda()
        elif pilihan == '3':
            tampilkan_pengguna()
            try:
                id_pengguna = int(input("\nID Pengguna yang akan diedit: ").strip())
                
                print("\nBiarkan kosong jika tidak ingin mengubah")
                username = input("Username baru: ").strip() or None
                password = input("Password baru: ").strip() or None
                nama_lengkap = input("Nama lengkap baru: ").strip() or None
                role = input("Role baru (admin/kasir): ").strip().lower() or None
                
                if role and role not in ['admin', 'kasir']:
                    print("Role harus 'admin' atau 'kasir'!")
                else:
                    edit_pengguna(id_pengguna, username, password, nama_lengkap, role)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '4':
            tampilkan_pengguna()
            try:
                id_pengguna = int(input("\nID Pengguna yang akan dinonaktifkan: ").strip())
                nonaktifkan_pengguna(id_pengguna)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '5':
            tampilkan_pengguna()
            try:
                id_pengguna = int(input("\nID Pengguna yang akan diaktifkan: ").strip())
                aktifkan_pengguna(id_pengguna)
            except ValueError:
                print("ID harus berupa angka!")
            jeda()
        elif pilihan == '6':
            break
        else:
            print("Pilihan tidak valid!")
            jeda()


def menu_laporan():
    while True:
        tampilkan_header("LAPORAN & ANALISIS")
        
        print("1.  Penjualan Hari Ini")
        print("2.  Riwayat Transaksi")
        print("3.  Laporan Mingguan")
        print("4.  Produk Terlaris")
        print("5.  Produk Kurang Laku")
        print("6.  Kembali ke Menu Utama")
        
        pilihan = input("\nPilih menu [1-6]: ").strip()
        
        if pilihan == '1':
            tampilkan_penjualan_hari_ini()
            jeda()
        elif pilihan == '2':
            tampilkan_riwayat_transaksi()
            jeda()
        elif pilihan == '3':
            tampilkan_penjualan_mingguan()
            jeda()
        elif pilihan == '4':
            tampilkan_produk_terlaris()
            jeda()
        elif pilihan == '5':
            tampilkan_produk_kurang_laku()
            jeda()
        elif pilihan == '6':
            break
        else:
            print("Pilihan tidak valid!")
            jeda()


# LOGIN & APLIKASI UTAMA


def login():
    tampilkan_header("LOGIN SISTEM MANAJEMEN TOKO")
    
    koneksi = koneksi_database()
    if not koneksi:
        print("Gagal terhubung ke database!")
        jeda()
        return None
    
    try:
        username = input("Username: ").strip()
        password = input("Password: ").strip()
        
        with koneksi.cursor() as kursor:
            kursor.execute("""
                SELECT id_user, username, nama_lengkap, role, status 
                FROM users 
                WHERE username = %s AND password = %s AND status = 'aktif'
            """, (username, password))
            
            pengguna = kursor.fetchone()
            
            if pengguna:
                data_pengguna = {
                    'id_user': pengguna[0],
                    'username': pengguna[1],
                    'nama_lengkap': pengguna[2],
                    'role': pengguna[3],
                    'status': pengguna[4]
                }
                print(f"\nLogin berhasil!")
                print(f"Selamat datang, {data_pengguna['nama_lengkap']} ({data_pengguna['role'].upper()})")
                jeda()
                return data_pengguna
            else:
                print("\nLogin gagal! Username atau password salah.")
                jeda()
                return None
                
    except Error as e:
        print(f"Error saat login: {e}")
        traceback.print_exc()
        jeda()
        return None
    finally:
        koneksi.close()


def menu_admin(pengguna_sekarang):
    while True:
        tampilkan_dashboard(pengguna_sekarang)
        
        print("\nMENU UTAMA:")
        print("1.  Transaksi Penjualan")
        print("2.  Manajemen Produk")
        print("3.  Manajemen Stok")
        print("4.  Manajemen Pengguna")
        print("5.  Laporan & Analisis")
        print("6.  Katalog Produk")
        print("7.  Ringkasan Stok")
        print("8.  Logout")
        print("9.  Keluar Program")
        
        pilihan = input("\nPilih menu [1-9]: ").strip()
        
        if pilihan == '1':
            proses_penjualan(pengguna_sekarang)
        elif pilihan == '2':
            menu_manajemen_produk()
        elif pilihan == '3':
            menu_manajemen_stok(pengguna_sekarang)
        elif pilihan == '4':
            menu_manajemen_pengguna()
        elif pilihan == '5':
            menu_laporan()
        elif pilihan == '6':
            tampilkan_katalog(filter_tersedia=False, untuk_admin=True)
            jeda()
        elif pilihan == '7':
            tampilkan_ringkasan_stok()
            jeda()
        elif pilihan == '8':
            print("Logout berhasil!")
            return False
        elif pilihan == '9':
            print("Terima kasih telah menggunakan sistem!")
            sys.exit(0)
        else:
            print("Pilihan tidak valid!")
            jeda()
    
    return True


def menu_kasir(pengguna_sekarang):
    while True:
        tampilkan_dashboard(pengguna_sekarang)
        
        print("\nMENU UTAMA:")
        print("1.  Transaksi Penjualan")
        print("2.  Katalog Produk")
        print("3.  Logout")
        print("4.  Keluar Program")
        
        pilihan = input("\nPilih menu [1-4]: ").strip()
        
        if pilihan == '1':
            proses_penjualan(pengguna_sekarang)
        elif pilihan == '2':
            tampilkan_katalog(filter_tersedia=True)
            jeda()
        elif pilihan == '3':
            print("Logout berhasil!")
            return False
        elif pilihan == '4':
            print("Terima kasih telah menggunakan sistem!")
            sys.exit(0)
        else:
            print("Pilihan tidak valid!")
            jeda()
    
    return True


# PROGRAM UTAMA


def utama():
    tampilkan_header("SISTEM MANAJEMEN TOKO - KELOMPOK 3")
    
    # Loop login
    while True:
        pengguna_sekarang = login()
        if pengguna_sekarang:
            if pengguna_sekarang['role'] == 'admin':
                while menu_admin(pengguna_sekarang):
                    pass
            elif pengguna_sekarang['role'] == 'kasir':
                while menu_kasir(pengguna_sekarang):
                    pass
        else:
            coba_lagi = input("\nLogin gagal. Coba lagi? (y/n): ").strip().lower()
            if coba_lagi != 'y':
                print("Terima kasih!")
                break


if __name__ == "__main__":
    utama()



