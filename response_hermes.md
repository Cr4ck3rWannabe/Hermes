Semua lokasi sudah terbaca lengkap dengan line number presisi. Berikut implementation plan berdasarkan source code aktual.

---

# IMPLEMENTATION PLAN — FIX CONNECTION POOL LEAK

**Status:** RENCANA SAJA. Tidak ada file yang diedit, tidak ada service di-restart, tidak ada perubahan pada database. Seluruh pemeriksaan read-only.

---

## Konfirmasi Akar Masalah (basis rencana ini)

| Fakta | Bukti |
|---|---|
| `pool.acquire()` mengembalikan `Connection` mentah | `aiomysql/pool.py` — `_PoolAcquireContextManager.__aenter__` → `return await self._coro` |
| `release()` hanya dipanggil via `__aexit__` | `_PoolAcquireContextManager.__aexit__` → `await self._pool.release(self._conn)` |
| `Connection.close()` tidak menyentuh pool | `connection.py` → hanya `self._writer.transport.close()`, set `_writer/_reader = None` |
| Hanya 7 pemakaian bermasalah di kode aktif | `grep "= await get_mysql_connection()"` → tepat 7 hasil, tidak ada yang lain |
| Tidak ada bentuk leak lain | `grep "pool.acquire()"` tanpa `async with` → hanya `db.py:45` (definisi helper itu sendiri) |
| Repo **bukan** git | `/opt/suoy_bot` → `fatal: not a git repository` |

Konvensi benar yang sudah ada di codebase: `config.py:69-71`, `admin.py:147-148`, `admin.py:186-187`, `admin.py:232-233` — semua pakai `get_pool()` + `async with pool.acquire() as conn:`. **Patch akan menyeragamkan ke konvensi ini.**

> ⚠️ **Gotcha kritis:** `db.py:16` mendefinisikan `pool = None` di module level, dan `init_pool()` baru mengisinya. Maka `from db import pool` akan menangkap `None` permanen. **WAJIB pakai `get_pool()`** (fungsi, resolve saat runtime). Ini alasan semua 7 patch memakai `get_pool()`, bukan import variabel.

---

# Analisis Per Lokasi

## LOCATION 1 — `handlers/admin.py:56`

### 1. Kode saat ini
```python
        conn = await get_mysql_connection()          # baris 56
        if not conn:
            await message.reply_text("❌ Gagal terhubung ke Database.")
            return

        response_text = ""
        try:
            async with conn.cursor() as cursor:      # baris 63
                await cursor.execute("""
                    INSERT IGNORE INTO bot_users ...
                """, (target_id, target_name))
                # ... 8 blok UPDATE (addwl/removewl/ban/unban/lead/rmlead/addadmin/rmadmin)
        except Exception as e:
            response_text = f"❌ Error DB: {e}"
            logger.error(f"Error admin command: {e}")
        finally:
            conn.close()                             # baris 111
```
Handler: `admin_command_handler` — melayani `/addwl /removewl /ban /unban /lead /rmlead /addadmin /rmadmin`.

### 2. Kenapa leak
`conn` berasal dari `pool.acquire()` → masuk `pool._used`. `finally: conn.close()` hanya menutup transport socket; **slot di `_used` tidak pernah dilepas**. Setiap pemanggilan satu perintah di atas = **1 slot pool hilang permanen**. Setelah 20× → `Pool._acquire()` menggantung di `await self._cond.wait()` tanpa timeout.

### 3. Perubahan minimal
```python
        response_text = ""
        db_pool = get_pool()                          # ← ganti acquire manual
        if not db_pool:
            await message.reply_text("❌ Gagal terhubung ke Database.")
            return
        try:
            async with db_pool.acquire() as conn:     # ← slot auto-release
                async with conn.cursor() as cursor:
                    # (body existing, indent +4)
        except Exception as e:
            response_text = f"❌ Error DB: {e}"
            logger.error(f"Error admin command: {e}")
                                                      # ← finally: conn.close() DIHAPUS
```
Import baris 13: `from db import check_user_access, get_mysql_connection` → `from db import check_user_access, get_pool`

### 4. Risiko perubahan behavior
- **Rendah.** Query, `response_text`, `parse_mode`, dan urutan reply tidak berubah.
- **NiK: tidak menambah `commit()`.** 8 blok UPDATE di sini tetap **tidak ter-commit** → tetap rollback saat koneksi di-release (perilaku identik dengan hari ini). *(Lihat bagian "Temuan Terpisah" di bawah.)*
- Efek observasi: bila `pool` belum siap, jalur `else` sekarang **tercapai**. Sebelumnya `get_mysql_connection()` melempar `RuntimeError("DB Pool belum siap...")` → uncaught → PTB log exception. Perubahan ini justru **menghidupkan fallback yang sudah ditulis developer** ("❌ Gagal terhubung ke Database."), bukan mengubah intent.

### 5. Cara test
1. `/ban <reply pesan user>` → harus balas "🚫 Pengguna X telah diblokir permanen."
2. `/addwl <reply>` → balas "✅ Pengguna X telah diizinkan (Active)."
3. `/lead <reply>` → balas "⭐ Pengguna X berhasil dijadikan LEAD."
4. `/addadmin <reply>` → balas HTML dengan `<b>nama</b>` (uji `parse_mode=HTML` tetap jalan)
5. `/rmadmin <reply>` → balas "🔽 Hak admin ... telah dicabut."
6. `/addwl 999999999999` (tanpa reply, arg valid) → balas sukses
7. `/addwl abc` (arg invalid) → balas "⚠️ User ID tidak valid."
8. Jalankan **21×** berurutan → heartbeat 30s harus tetap jalan (bukti pool tidak habis)

---

## LOCATION 2 — `handlers/search.py:228`

### 1. Kode saat ini
```python
    conn = await get_mysql_connection()              # baris 228
    if conn:
        try:
            async with conn.cursor(aiomysql.DictCursor) as cursor:
                query = "SELECT sales_name, wo_count FROM fat_sales WHERE fat_code = %s AND ..."
                await cursor.execute(query, (fat_id,))
                sales_data = await cursor.fetchall()
            if sales_data:
                ... await update.message.reply_text(msg, parse_mode=ParseMode.MARKDOWN)
            else:
                ... await update.message.reply_text("❌ Maaf Boss, saya belum punya data Sales ...")
        except Exception as e:
            logger.error(f"Error Query DB Sales: {e}")
            await update.message.reply_text("❌ Terjadi kesalahan sistem saat membaca database.")
        finally:
            conn.close()                             # baris 255
    else:
        await update.message.reply_text("❌ Gagal terhubung ke Database MariaDB.")
```
Handler: `sales_command_handler` — `/sales [KODE_FAT]` atau reply WO + `/sales`.

### 2. Kenapa leak
Sama: `conn` dari `pool.acquire()` masuk `_used`; `conn.close()` tidak me-release. **1 slot hilang per pemakaian `/sales`.** Read-only, jadi dampaknya murni kehabisan pool — tidak ada efek data.

### 3. Perubahan minimal
```python
    db_pool = get_pool()                             # ← ganti
    if db_pool:
        try:
            async with db_pool.acquire() as conn:    # ← tambah
                async with conn.cursor(aiomysql.DictCursor) as cursor:
                    # (body existing, indent +4)
        except Exception as e:
            logger.error(f"Error Query DB Sales: {e}")
            await update.message.reply_text("❌ Terjadi kesalahan sistem saat membaca database.")
                                                     # ← finally: conn.close() DIHAPUS
    else:
        await update.message.reply_text("❌ Gagal terhubung ke Database MariaDB.")
```
Import baris 11: `from db import check_user_access, find_fat_in_db, get_mysql_connection, load_bot_data_from_db` → ganti `get_mysql_connection` → `get_pool`
Catatan: `aiomysql` tetap diimpor (dipakai `aiomysql.DictCursor`) — **jangan hapus** baris 5.

### 4. Risiko perubahan behavior
- **Rendah.** Ketiga cabang reply (sukses / kosong / error) dipertahankan persis.
- `logger.error` + reply error tetap di dalam `except` — tidak diubah.
- `if db_pool:` sekarang benar-benar bisa `False` (sebelumnya `if conn:` selalu `True` atau raise). Perilaku fallback jadi sesuai intent kode.

### 5. Cara test
1. `/sales PGP-MBT-D04-S01-A16` → cari FAT yang ada di tabel (mis. dari sample: `PGP-MBT-D04-S01-A16`) → harus balas "✅ *Data Sales Ditemukan*" + daftar
2. `/sales XXXXX-NOT-EXIST` → harus balas "❌ Maaf Boss, saya belum punya data Sales ..."
3. Reply pesan WO lalu `/sales` → parsing `PORT FAT` + regex FAT harus tetap menghasilkan data
4. `/sales` tanpa arg tanpa reply → balas "🔍 *Format Salah / FAT Tidak Terbaca*"
5. Jalankan **21×** → heartbeat tetap jalan

---

## LOCATION 3 — `handlers/search.py:268`

### 1. Kode saat ini
```python
    total_data = 0
    conn = await get_mysql_connection()              # baris 268
    if conn:
        try:
            async with conn.cursor() as cursor:
                await cursor.execute("SELECT COUNT(DISTINCT fat_code) FROM fat_sales")
                result = await cursor.fetchone()
                total_data = result[0] if result else 0
        except Exception as e:
            logger.error(f"Error Hitung Sales DB: {e}")
        finally:
            conn.close()                             # baris 278
    else:
        await update.message.reply_text("❌ Database sedang offline.")
        return

    await update.message.reply_text(
        f"📊 *Statistik Data Sales (Database)*\n\n"
        f"✅ Total FAT yang sudah terpetakan: *{total_data} data*\n\n"
        f"_Data ini tersimpan aman di dalam MariaDB VPS._",
        parse_mode=ParseMode.MARKDOWN
    )
```
Handler: `ceksales_command` — `/ceksales`.

### 2. Kenapa leak
`conn` dari `pool.acquire()`; `conn.close()` tidak me-release → 1 slot hilang per `/ceksales`.

### 3. Perubahan minimal
```python
    total_data = 0
    db_pool = get_pool()
    if db_pool:
        try:
            async with db_pool.acquire() as conn:
                async with conn.cursor() as cursor:
                    await cursor.execute("SELECT COUNT(DISTINCT fat_code) FROM fat_sales")
                    result = await cursor.fetchone()
                    total_data = result[0] if result else 0
        except Exception as e:
            logger.error(f"Error Hitung Sales DB: {e}")
    else:
        await update.message.reply_text("❌ Database sedang offline.")
        return
    # reply statistik di bawah — TIDAK DIUBAH
```
**Variabel `total_data` dipakai setelah blok** → pastikan inisialisasi `total_data = 0` tetap di **luar** `async with` (seperti sekarang). Ini titik paling rawan saat re-indent.

### 4. Risiko perubahan behavior
- **Rendah–sedang.** Risiko satu-satunya: salah indent membuat `total_data` ter-scope lokal di dalam blok sehingga reply statistik menampilkan `0`. Mitigasi: pertahankan `total_data = 0` persis di baris sebelum `db_pool = get_pool()`.

### 5. Cara test
1. `/ceksales` → harus balas "📊 *Statistik Data Sales (Database)*" dengan angka **2329-level FAT count yang sama seperti sebelum patch** (bandingkan angka sebelum/sesudah — harus identik karena sumber data sama)
2. Verifikasi angka **bukan 0** (bukti `total_data` tidak ter-shadow)
3. Jalankan **21×** → heartbeat tetap jalan

---

## LOCATION 4 — `handlers/admin.py:389`

### 1. Kode saat ini
```python
    # 3. Ambil user aktif dari MariaDB
    conn = await get_mysql_connection()              # baris 389
    if not conn: return
    active_users = []
    try:
        async with conn.cursor() as cursor:
            await cursor.execute("SELECT user_id FROM bot_users WHERE status = 'active'")
            rows = await cursor.fetchall()
            active_users = [row[0] for row in rows]
    except Exception as e:
        logger.error(f"Error fetch users: {e}")
    finally:
        conn.close()                                 # baris 400

    if not active_users:
        await update.message.reply_text("⚠️ Tidak ada pengguna aktif di database.")
        return

    # 4. Kirim Pesan (Broadcast) — loop context.bot.send_message + asyncio.sleep(0.05)
```
Handler: `notif_command` — `/notif <pesan>`. Hanya owner.

### 2. Kenapa leak
`conn` dari `pool.acquire()`; `conn.close()` tidak me-release → 1 slot hilang **per broadcast**. Karena `/notif` mengirim ke banyak user dalam satu handler, handler ini berjalan lama (loop + `asyncio.sleep(0.05)`), sehingga memblokir antrean update PTB lebih lama — amplifikasi dampak leak.

### 3. Perubahan minimal
```python
    # 3. Ambil user aktif dari MariaDB
    active_users = []
    db_pool = get_pool()
    if not db_pool:
        return
    try:
        async with db_pool.acquire() as conn:
            async with conn.cursor() as cursor:
                await cursor.execute("SELECT user_id FROM bot_users WHERE status = 'active'")
                rows = await cursor.fetchall()
                active_users = [row[0] for row in rows]
    except Exception as e:
        logger.error(f"Error fetch users: {e}")
    # ← finally: conn.close() DIHAPUS
```
Catatan: `active_users = []` **dipindah ke atas** `if not db_pool: return` supaya tidak `UnboundLocalError` bila pool kosong. Broadcast loop (baris 405+) **tidak disentuh**.

### 4. Risiko perubahan behavior
- **Rendah.** `active_users` tetap list biasa; guard "Tidak ada pengguna aktif" tetap berfungsi.
- ⚠️ **Risiko laten:** `active_users` diisi dari `bot_users WHERE status='active'`. Karena write dari `track_user_db`/`admin.py` tidak ter-commit (lihat Temuan Terpisah), isi whitelist DB mencerminkan data 2026-09-11. Ini **perilaku existing** — patch pool **tidak mengubahnya**, tapi test harus memakai ekspektasi "daftar user yang ada di DB saat ini", bukan hasil broadcast sebelumnya.

### 5. Cara test
1. `/notif TEST POOL FIX` sebagai owner → status "⏳ Mengirim broadcast ke N user aktif..."
2. Verifikasi `N` konsisten dengan `SELECT COUNT(*) FROM bot_users WHERE status='active'` (baca manual)
3. Verifikasi ringkasan akhir broadcast muncul
4. `/notif` tanpa teks → balas "⚠️ **Format Salah!**"
5. `/notif test` sebagai non-owner → balas "⛔ Anda tidak memiliki akses untuk broadcast."
6. Jalankan **21×** → heartbeat tetap jalan

---

## LOCATION 5 — `handlers/smart.py:979`

### 1. Kode saat ini
```python
        if data == 'admin_stats':
            # Ambil statistik langsung dari database (bukan cache memori)
            db_conn_stat = await get_mysql_connection()          # baris 979
            users_db = []
            if db_conn_stat:
                try:
                    async with db_conn_stat.cursor(aiomysql.DictCursor) as cur:
                        await cur.execute("""
                            SELECT user_id, first_name, username, hit_count, last_seen, role, status, expiry_date
                            FROM bot_users ORDER BY hit_count DESC LIMIT 30
                        """)
                        users_db = await cur.fetchall()
                except Exception as e:
                    logger.error(f'Error admin_stats DB: {e}')
                finally:
                    db_conn_stat.close()                         # baris 992
            text = f'📊 *Statistik Pengguna Bot*\nTotal pengguna: {len(users_db)}\n\n'
            # ... render text + kirim
```
Handler: `button_callback` → tombol **`admin_stats`** di menu Admin.

### 2. Kenapa leak
`db_conn_stat` dari `pool.acquire()`; `db_conn_stat.close()` tidak me-release → 1 slot hilang per klik tombol "Statistik".

### 3. Perubahan minimal
```python
        if data == 'admin_stats':
            users_db = []
            db_pool_stat = get_pool()                            # ← nama lokal baru
            if db_pool_stat:
                try:
                    async with db_pool_stat.acquire() as db_conn_stat:
                        async with db_conn_stat.cursor(aiomysql.DictCursor) as cur:
                            await cur.execute("""...""")
                            users_db = await cur.fetchall()
                except Exception as e:
                    logger.error(f'Error admin_stats DB: {e}')
                                                             # ← finally DIHAPUS
            text = f'📊 *Statistik Pengguna Bot*\nTotal pengguna: {len(users_db)}\n\n'
            # render + kirim — TIDAK DIUBAH
```
Import baris 19: ganti `get_mysql_connection` → `get_pool` (daftar import panjang: `add_fat_to_db, check_user_access, find_fat_in_db, save_customer_to_db, save_sales_to_db, track_user_db, insert_material_transaction, get_material_transactions, delete_material_transactions` — **jangan ubah yang lain**).

### 4. Risiko perubahan behavior
- **Rendah.** `users_db` tetap list of dict (karena `aiomysql.DictCursor`), sehingga seluruh rendering `u['username']`, `u.get('role','')`, `u.get('last_seen')` tetap bekerja identik.
- `len(users_db)` untuk header "Total pengguna" tetap sama.
- ⚠️ Ada cabang `else: text += '_Belum ada data pengguna._'` — pastikan tetap ter-cover saat `users_db` kosong.

### 5. Cara test
1. Buka menu Admin → klik **📊 Statistik** → harus tampil "Total pengguna: N" + daftar 30 user teratas dengan ikon role (`👑`/`⭐`/`👤`)
2. Verifikasi format tanggal `last_seen` tetap "dd Mmm yyyy, HH:MM"
3. Verifikasi tombol "⬅️ Kembali ke Menu Admin" tetap berfungsi
4. Klik tombol **21×** → heartbeat tetap jalan, tidak ada `Error admin_stats DB`

---

## LOCATION 6 — `handlers/smart.py:1095`

### 1. Kode saat ini
```python
        elif data in ['admin_blacklist_list', 'admin_whitelist_list']:
            is_whitelist = "whitelist" in data
            list_type = "Whitelist" if is_whitelist else "Blacklist"
            # Ambil dari DB langsung: whitelist = status active, blacklist = status blocked
            db_conn2 = await get_mysql_connection()              # baris 1095
            db_users = []
            if db_conn2:
                try:
                    async with db_conn2.cursor(aiomysql.DictCursor) as cur:
                        if is_whitelist:
                            await cur.execute("""SELECT ... FROM bot_users WHERE status='active' ORDER BY role DESC, first_name ASC LIMIT 50""")
                        else:
                            await cur.execute("""SELECT ... FROM bot_users WHERE status='blocked' ORDER BY first_name ASC LIMIT 50""")
                        db_users = await cur.fetchall()
                except Exception as e:
                    logger.error(f"Error WL/BL list DB: {e}")
                finally:
                    db_conn2.close()                             # baris 1116
            text = f"""📋 *Daftar Pengguna — {list_type}*\nTotal: {len(db_users)}\n\n"""
            # ... render + kirim dengan keyboard
```
Handler: `button_callback` → tombol **`admin_whitelist_list`** / **`admin_blacklist_list`**.

### 2. Kenapa leak
`db_conn2` dari `pool.acquire()`; `db_conn2.close()` tidak me-release → 1 slot hilang per klik "Lihat Whitelist"/"Lihat Blacklist".

### 3. Perubahan minimal
```python
            db_users = []
            db_pool_wl = get_pool()                              # ← nama lokal baru
            if db_pool_wl:
                try:
                    async with db_pool_wl.acquire() as db_conn2:
                        async with db_conn2.cursor(aiomysql.DictCursor) as cur:
                            if is_whitelist:
                                await cur.execute("""...""")
                            else:
                                await cur.execute("""...""")
                            db_users = await cur.fetchall()
                except Exception as e:
                    logger.error(f"Error WL/BL list DB: {e}")
                                                             # ← finally DIHAPUS
            text = f"""..."""
            # render + keyboard — TIDAK DIUBAH
```

### 4. Risiko perubahan behavior
- **Rendah.** Kedua cabang query (whitelist/blacklist) tidak berubah; `db_users` tetap list of dict.
- `expiry_date` handling (`hasattr(exp, 'strftime')`) tetap bekerja karena `DictCursor` mengembalikan objek `datetime`/`date` asli.
- Nama variabel lokal baru (`db_pool_wl`, `db_pool_stat`) menghindari tabrakan; `smart.py` tidak punya variabel `pool` (0 occurrence) → **tidak ada shadowing**.

### 5. Cara test
1. Buka menu Admin → tombol **Whitelist** → tampil "📋 *Daftar Pengguna — Whitelist*" + `Total: N`
2. Tombol **Blacklist** → tampil "📋 *Daftar Pengguna — Blacklist*" + `Total: N`
3. Verifikasi baris "Exp: dd-Mmm-yyyy" atau "Lifetime ♾️"
4. Verifikasi tombol "⬅️ Kembali" berfungsi
5. Klik kedua tombol **21×** → heartbeat tetap jalan, tidak ada `Error WL/BL list DB`

---

## LOCATION 7 — `handlers/moratel.py:603`

### 1. Kode saat ini
```python
        if sn_api:
            sales_name  = sn_api
            sales_email = se_api if se_api else "-"
            # Simpan ke fat_sales DB (API menjadi sumber truth, tanpa email)
            if fat_code_raw and fat_code_raw not in ("-", "None"):
                banned_sales = [s.upper() for s in BOT_DATA.get("banned_sales", [])]
                if (sn_api.upper() not in ["TECHNICIAN","UNKNOWN","UNKNOWN (AUTO-MAPPING)","-"]
                        and sn_api.upper() not in banned_sales):
                    _conn_s = await get_mysql_connection()          # baris 603
                    if _conn_s:
                        try:
                            async with _conn_s.cursor() as _cur:
                                await _cur.execute(
                                    "INSERT INTO fat_sales (fat_code, sales_name, wo_count) "
                                    "VALUES (%s, %s, 1) ON DUPLICATE KEY UPDATE wo_count = wo_count + 1",
                                    (fat_code_raw.upper(), sn_api)
                                )
                        except Exception:
                            pass
                        finally:
                            _conn_s.close()                         # baris 615
```
Handler: dipanggil dari jalur pemrosesan detail WO (`_fetch_and_send_detail_wo`) — **HOT PATH**. Ini satu-satunya lokasi **WRITE** di antara 7.

### 2. Kenapa leak
`_conn_s` dari `pool.acquire()`; `_conn_s.close()` tidak me-release → 1 slot hilang **setiap kali diproses WO dengan `sales_name` dari API**. Ini kandidat pemicu terakhir pada insiden 06:42 hari ini: log `06:42:42 [WORKORDER2] ✅ Cookie aktif dipakai` muncul **1 detik** sebelum respons terakhir, lalu pool habis.

### 3. Perubahan minimal
```python
                    _pool_s = get_pool()                            # ← ganti acquire manual
                    if _pool_s:
                        try:
                            async with _pool_s.acquire() as _conn_s:  # ← auto-release
                                async with _conn_s.cursor() as _cur:
                                    await _cur.execute(
                                        "INSERT INTO fat_sales (fat_code, sales_name, wo_count) "
                                        "VALUES (%s, %s, 1) ON DUPLICATE KEY UPDATE wo_count = wo_count + 1",
                                        (fat_code_raw.upper(), sn_api)
                                    )
                        except Exception:
                            pass                                  # ← silent, DIPERTAHANKAN
                                                             # ← finally DIHAPUS
```
Import baris 14: tambah `get_pool`, hapus `get_mysql_connection`:
`from db import add_fat_to_db, check_user_access, find_fat_in_db, get_pool, get_sales_prediction, get_top_sales_for_fat, hitung_jarak, get_material_transactions`
Catatan: `moratel.py` **tidak** mengimpor `aiomysql` — tidak perlu ditambah, karena blok ini hanya pakai `conn.cursor()` biasa.

### 4. Risiko perubahan behavior
- **Rendah**, tapi ada satu efek teknis yang harus disadari (bukan bug):
  - Karena `create_pool()` memakai default `autocommit=False` (dikonfirmasi: `SELECT @@autocommit` pada koneksi aiomysql = **0**), blok `INSERT` ini **memulai transaksi** (`get_transaction_status()` → `True`).
  - Saat `release()` dipanggil: `if in_trans: conn.close()` → **koneksi fisik ditutup**, bukan dikembalikan ke `_free`.
  - **Tidak ada leak** (`self._used.remove(conn)` terjadi lebih dulu), tapi ada **connection churn**: 1 handshake MySQL baru per WO write.
  - **Hasil data identik dengan hari ini**: tanpa `commit()`, transaksi tetap **rollback**. Patch ini **bukan** memperbaiki persistensi data — hanya memperbaiki pool.
- **Jangan** menambahkan `commit()` di patch ini (di luar scope, dan akan mengubah perilaku data). Lihat rekomendasi terpisah di bawah.

### 5. Cara test
1. Kirim/tarik detail WO yang menghasilkan `sales_name` dari API (mis. via tombol team-task) → respons detail WO harus tetap muncul normal, **tidak ada perubahan output**
2. Verifikasi log tetap muncul `[PATCH] fetching WO team=... ` dan `API status=success`
3. Verifikasi **tidak ada** traceback baru di `journalctl` untuk `handlers.moratel`
4. Proses **21 WO** berurutan → heartbeat 30s tetap jalan
5. Cross-check: `SELECT MAX(updated_at) FROM suoy_db.fat_sales` **tetap tidak berubah** setelah patch (= perilaku rollback dipertahankan; jika nilainya berubah, berarti ada `commit()` yang tidak sengaja ditambahkan)

---

# Files To Change

**Hanya 4 file. Tidak ada file lain.**

| # | File | Perubahan | Lokasi |
|---|---|---|---|
| 1 | `/opt/suoy_bot/handlers/admin.py` | Import L13 + 2 blok | `:56` (admin_command_handler), `:389` (notif_command) |
| 2 | `/opt/suoy_bot/handlers/search.py` | Import L11 + 2 blok | `:228` (sales_command_handler), `:268` (ceksales_command) |
| 3 | `/opt/suoy_bot/handlers/smart.py` | Import L19 + 2 blok | `:979` (admin_stats), `:1095` (wl/bl list) |
| 4 | `/opt/suoy_bot/handlers/moratel.py` | Import L14 + 1 blok | `:603` (WO detail → fat_sales write) |

**TIDAK boleh disentuh:**
- `db.py` — kecuali **opsional** menambah guard/deprecate `get_mysql_connection()` (lihat Fase 3). `init_pool()`, `maxsize`, `minsize`, `pool_recycle` **tidak diubah**.
- `main.py` — `run_polling()`, handler registration, `threading.Thread` **tidak diubah**.
- `config.py`, `utils.py`, `keysboards.py`, `reports.py`, `templates.py`, `wo_parser.py`, `material_parser.py` — tidak berhubungan.
- `handlers/basic.py`, `tikor.py`, `photos.py`, `fat.py` — sudah bersih (tidak memakai `get_mysql_connection`).
- `workorder.py`, `workorder2.py`, `mora.py` — di luar scope.
- File `backup_*` dan `secrets_legacy/` — dead code, tidak diimpor. **Jangan diedit** (mengaburkan audit trail).

**Pre-flight (belum dijalankan, hanya rencana):** karena `/opt/suoy_bot` bukan git repo, langkah pertama sebaiknya membuat **backup copy** ke direktori terpisah sebelum edit, mis. `cp -a /opt/suoy_bot /opt/suoy_bot.bak_poolfix_$(date +%Y%m%d_%H%M%S)`. Ini *satu-satunya* penulisan yang disarankan dan **belum dilakukan**.

---

# Minimal Patch Plan

**Prinsip: 1 baris acquire → 1 blok `async with`. Tidak ada perubahan query, reply, template, atau urutan logika.**

### Urutan eksekusi

| Fase | Aksi | Dampak risiko |
|---|---|---|
| **0** | Backup `cp -a` seluruh `/opt/suoy_bot` | Nol (hanya salinan) |
| **1** | Patch **6 lokasi READ** (search.py ×2, admin.py ×2, smart.py ×2) | Rendah, read-only, tidak menyentuh data |
| **2** | Patch **1 lokasi WRITE** (moratel.py:603) | Rendah, tapi ini hot path → uji WO lebih hati-hati |
| **3** | Verifikasi pool (lihat Test Plan) | Nol |
| **4** | *Opsional, terpisah:* tambah guard pada `db.py:42 get_mysql_connection()` (mis. `warnings.warn` + docstring "gunakan `async with pool.acquire()`") — **tanpa menghapus** fungsi, supaya import lama tidak patah | Nol fungsional |

### Patch signature (ringkas)

| Lokasi | Sebelum | Sesudah |
|---|---|---|
| admin.py:56 | `conn = await get_mysql_connection()` + `if not conn:` + `finally: conn.close()` | `db_pool = get_pool()` + `if not db_pool:` + `async with db_pool.acquire() as conn:` − `finally` |
| admin.py:389 | idem | idem (`db_pool`) + pindahkan `active_users = []` ke atas guard |
| search.py:228 | `conn = await get_mysql_connection()` + `if conn:` + `finally: conn.close()` | `db_pool = get_pool()` + `if db_pool:` + `async with db_pool.acquire() as conn:` − `finally` |
| search.py:268 | idem | idem + **pertahankan** `total_data = 0` di luar blok |
| smart.py:979 | `db_conn_stat = await get_mysql_connection()` + `finally: .close()` | `db_pool_stat = get_pool()` + `async with db_pool_stat.acquire() as db_conn_stat:` − `finally` |
| smart.py:1095 | `db_conn2 = await get_mysql_connection()` + `finally: .close()` | `db_pool_wl = get_pool()` + `async with db_pool_wl.acquire() as db_conn2:` − `finally` |
| moratel.py:603 | `_conn_s = await get_mysql_connection()` + `if _conn_s:` + `finally: _conn_s.close()` | `_pool_s = get_pool()` + `if _pool_s:` + `async with _pool_s.acquire() as _conn_s:` − `finally` |

### Aturan wajib saat menerapkan

1. **Pakai `get_pool()`, bukan `from db import pool`** — kalau salah, `pool` = `None` selamanya dan semua handler langsung masuk cabang "gagal terhubung".
2. **Hapus seluruh `finally: conn.close()`** — kalau tertinggal, koneksi di-close setelah di-release oleh pool → koneksi sehat di `_free` menjadi mati, lalu `assert not conn.closed` di `_acquire` bisa memicu `AssertionError`.
3. **Jangan tambah `commit()`** di fase ini.
4. **Jangan ubah string query, teks reply, `parse_mode`, atau urutan `logger`** — patch harus murni struktural.
5. **Perhatikan `total_data` (search.py:268)** — pertahankan inisialisasi di luar blok.
6. **Jangan ubah `maxsize`, `minsize`, `pool_recycle`, atau `run_polling()`.**

---

# Regression Risk

| # | Risiko | Penyebab | Mitigasi |
|---|---|---|---|
| R1 | **`pool = None` permanen** | `from db import pool` (bukan `get_pool()`) | Wajib `get_pool()`. Verifikasi: semua handler tidak langsung membalas "❌ Gagal terhubung" |
| R2 | **AssertionError `not conn.closed`** | `finally: conn.close()` tertinggal setelah di-patch | Hapus `finally` sepenuhnya. Cek `grep -n "conn.close()"` di 4 file → **0 hasil** setelah patch |
| R3 | **`UnboundLocalError: total_data`** | Re-indent search.py:268 | `total_data = 0` tetap di **luar** `async with` |
| R4 | **`UnboundLocalError: active_users`** | Guard `return` di admin.py:389 dipindah setelah inisialisasi | Pindahkan `active_users = []` ke **atas** `if not db_pool: return` |
| R5 | **Shadowing nama `pool`** | Menambah variabel global `pool` di handler | Pakai nama lokal unik (`db_pool`, `db_pool_stat`, `db_pool_wl`, `_pool_s`). `smart.py`/`search.py`/`moratel.py` sudah 0 occurrence `pool` — aman |
| R6 | **`aiomysql` import terhapus** | search.py & smart.py butuh `aiomysql.DictCursor` | Pertahankan `import aiomysql` di L5 (search) dan L10 (smart) |
| R7 | **`DictCursor` tertukar jadi cursor biasa** | Re-indent blok smart.py | `db_users`/`users_db` harus tetap list-of-dict; kalau jadi tuple, `u['username']` → `TypeError` |
| R8 | **Connection churn di moratel.py** | `in_trans=True` saat release → `conn.close()` | Bukan bug. Pantau: jumlah handshake MySQL naik. Jangan "diperbaiki" dengan `commit()` di patch ini |
| R9 | **Perilaku fallback berubah** | `if conn:` (selalu truthy/raise) → `if db_pool:` (bisa `False`) | Ini **intended behavior** yang kini tercapai. Uji: pastikan pesan fallback yang benar muncul, bukan `RuntimeError` di log |
| R10 | **Broadcast `/notif` mengirim ke daftar berbeda** | Bukan efek patch, tapi `bot_users` berisi data beku | Catat `SELECT COUNT(*) FROM bot_users WHERE status='active'` **sebelum** test sebagai baseline pembanding |
| R11 | **Workorder2 terpengaruh** | `moratel.py:603` berada di jalur WO | Patch hanya membungkus blok INSERT; tidak ada perubahan pada `workorder2`/`mora`. Uji 1 WO penuh end-to-end |
| R12 | **Tidak ada rollback plan** | Repo bukan git | Backup `cp -a` di Fase 0 = satu-satunya jalur rollback |
| R13 | **Patch sebagian** | 7 lokasi, mudah terlewat 1 | Checklist: `grep -c "= await get_mysql_connection()"` di 4 file → **harus 0** setelah patch |

---

# Test Plan

### A. Test per handler (fungsional)

| # | Handler | Trigger | Ekspektasi (harus SAMA sebelum & sesudah) |
|---|---|---|---|
| T1 | `admin_command_handler` | `/ban` (reply) | "🚫 Pengguna X telah diblokir permanen." |
| T2 | `admin_command_handler` | `/addwl` (reply) | "✅ Pengguna X telah diizinkan (Active)." |
| T3 | `admin_command_handler` | `/removewl` (reply) | "🗑️ Izin pengguna X telah dicabut." |
| T4 | `admin_command_handler` | `/addadmin` (reply) | HTML: "👑 Pengguna **X** (ID: `...`) berhasil dijadikan **Admin**." |
| T5 | `admin_command_handler` | `/lead` (reply) | "⭐ Pengguna X berhasil dijadikan LEAD." |
| T6 | `admin_command_handler` | `/addwl abc` | "⚠️ User ID tidak valid." |
| T7 | `sales_command_handler` | `/sales <FAT_ADA>` | "✅ *Data Sales Ditemukan*" + list |
| T8 | `sales_command_handler` | `/sales <FAT_TIDAK_ADA>` | "❌ Maaf Boss, saya belum punya data Sales ..." |
| T9 | `sales_command_handler` | reply WO + `/sales` | Data ditemukan (uji parsing `PORT FAT` + regex) |
| T10 | `sales_command_handler` | `/sales` (tanpa arg) | "🔍 *Format Salah / FAT Tidak Terbaca*" |
| T11 | `ceksales_command` | `/ceksales` | "📊 *Statistik Data Sales (Database)*" + angka **≠ 0** & identik dengan pre-patch |
| T12 | `notif_command` | `/notif TEST` (owner) | Progres broadcast + ringkasan akhir |
| T13 | `notif_command` | `/notif` (tanpa teks) | "⚠️ **Format Salah!**" |
| T14 | `button_callback` | klik `admin_stats` | "📊 *Statistik Pengguna Bot*" + 30 user + ikon role |
| T15 | `button_callback` | klik `admin_whitelist_list` | "📋 *Daftar Pengguna — Whitelist*" + Exp/Lifetime |
| T16 | `button_callback` | klik `admin_blacklist_list` | "📋 *Daftar Pengguna — Blacklist*" |
| T17 | `moratel` WO path | proses 1 WO | Detail WO normal, tanpa traceback |

### B. Test khusus pool release (inti perbaikan)

**B1 — Stress 21× per site (melebihi `maxsize=20`)**

Urutan: setiap handler dijalankan **21×**. Ambang kritis = 20. Jika leak masih ada, pemanggilan ke-21 akan **menggantung** (bot diam, `getUpdates` tetap jalan, heartbeat berhenti). Jika patch benar, semuanya selesai normal.

```
21× /ban      → 21× /addwl    → 21× /sales    → 21× /ceksales
→ 21× /notif  → 21× admin_stats → 21× whitelist → 21× blacklist
→ 21× proses WO
```
Total 189 require-leak-path. **Kriteria lulus: semua selesai, tidak ada hang.**

**B2 — Canary heartbeat (paling sensitif & paling murah)**

Handler/log yang paling cepat mendeteksi exhaustion adalah task `_reload_bot_data_loop` (`db.py:256`, interval 30s). Selama stress B1:

```
journalctl -u suoy -f | grep "BOT_DATA dimuat dari DB"
```
**Kriteria lulus:** detak tiap **~30 detik tanpa jeda** sepanjang seluruh stress test. Jeda ≥60s = leak masih ada (persis signature insiden 06:42 hari ini).

**B3 — Test handler yang paling mirip pemicu insiden**

`/notif` dan `moratel.py:603` adalah dua yang paling berisiko (loop panjang & hot path). Jalankan `/notif` **21×** berturut-turut saat traffic lain aktif — ini skenario terberat.

### C. Test negative (memastikan tidak over-fix)

| # | Test | Kriteria |
|---|---|---|
| C1 | `grep -rn "conn.close()" handlers/{admin,search,smart,moratel}.py` | **0 hasil** |
| C2 | `grep -c "= await get_mysql_connection()"` di 4 file | **0 hasil** |
| C3 | `grep -rn "commit()" handlers/` | **tidak ada penambahan** dibanding baseline (tidak ada commit baru) |
| C4 | `SELECT MAX(updated_at) FROM suoy_db.fat_sales` | **tetap `2026-09-11 16:40:42`** (membuktikan tidak sengaja menambah commit) |
| C5 | `wc -l` keempat file | Selisih wajar (penghapusan `finally`, penambahan `async with`) — bukan indikasi blok terhapus |
| C6 | `python3 -m py_compile handlers/{admin,search,smart,moratel}.py` | Syntax OK (tanpa menjalankan bot) |

---

# Verification

Bagaimana membuktikan — setelah fix — bahwa **`pool._used` kembali turun** dan **pool tidak terus bertambah**.

### V1 — Probe langsung state pool (paling definitif)

Jalankan dari dalam konteks proses bot (mis. via REPL/debug handler sementara, atau import di interpreter terpisah **tanpa** mengubah file):

```python
from db import get_pool
p = get_pool()
print("freesize :", p.freesize)   # = len(p._free)
print("used     :", len(p._used))
print("acquiring:", p._acquiring)
print("size     :", p.size)        # freesize + used + acquiring
print("maxsize  :", p.maxsize)     # 20
```

**Kriteria lulus per handler:**

| Metrik | Sebelum handler | Sesudah handler | Kesimpulan |
|---|---|---|---|
| `len(p._used)` | 0 (idle) | **0 (idle)** | ✅ slot dilepas |
| `len(p._used)` | 0 | **+1 permanen** | ❌ masih leak |

**Uji berulang (paling penting):** catat `len(p._used)` sebelum & sesudah, loop **21×**:
```
mulai   : used=0
setelah 1×  : used=0   ← wajib kembali 0
setelah 5×  : used=0
setelah 21× : used=0   ← wajib 0; kalau 20 → leak belum beres
```
Hari ini (pre-patch) pola ini akan menghasilkan `used=1,2,3...20` lalu **hang**. Setelah patch harus tetap **0**.

### V2 — Counter koneksi MariaDB (independen dari internal Python)

```sql
SELECT COUNT(*) FROM information_schema.PROCESSLIST WHERE USER='suoy_user';
```
**Baseline terukur hari ini: 5** (= `minsize`).

- Idle → **5–6** (5 pool + kemungkinan probe) → ✅ normal
- Selama traffic → boleh naik (pool growth), **tapi harus kembali ke ~5 saat idle**
- **Tidak boleh menetap di/near 20** → ❌ indikasi exhaustion

Pemantauan berulang:
```bash
for i in $(seq 1 30); do
  mysql -N -e "SELECT COUNT(*) FROM information_schema.PROCESSLIST WHERE USER='suoy_user';"
  sleep 10
done
```
Kriteria: **tidak ada tren naik** menuju 20.

> Catatan penting: metode ini **saja tidak cukup**. Pada insiden hari ini, MariaDB hanya menampilkan 5 koneksi saat pool bot sudah bocor — karena `conn.close()` mematikan socket sehingga koneksi bocor **tak terlihat dari sisi server**. Karena itu **V1 wajib**, V2 hanya konfirmasi.

### V3 — Canary heartbeat (deteksi dini, tanpa akses proses)

```bash
journalctl -u suoy --since "10 min ago" -o short-iso | grep "BOT_DATA dimuat dari DB"
```
Ukur interval antar detak. Kriteria lulus: **median ~30s, maksimum <45s, tanpa gap**.

Pembeda diagnosis (dari audit sebelumnya):

| Gejala | Interpretasi |
|---|---|
| Heartbeat berhenti + `getUpdates` tetap jalan | **Pool exhausted** (leak masih ada) |
| Heartbeat & `getUpdates` sama-sama berhenti | Event loop / proses mati (masalah lain) |

### V4 — Metrik server sebagai bukti pendukung

```sql
SHOW GLOBAL STATUS LIKE 'Aborted_clients';   -- baseline hari ini: 3505
SHOW GLOBAL STATUS LIKE 'Connections';       -- baseline hari ini: 11573
```
Ambil delta sebelum/sesudah stress test B1:
- **Sebelum patch:** delta `Aborted_clients` naik (socket ditutup paksa) + `Connections` naik karena pool terus membuat koneksi baru.
- **Setelah patch:** delta keduanya kecil/nol saat idle → slot dipakai ulang, bukan dibuang.

### V5 — Uji shutdown bersih (regression test terbaik untuk insiden hari ini)

Setelah fix, saat maintenance window berikutnya, perhatikan:
```bash
journalctl -u suoy --since "5 min ago" | grep -E "Stopping|stop-sigterm|Killing|Started|Failed with result"
```
**Kriteria lulus:** `Stopping` → `Started` dalam **< 5 detik**, dan **TIDAK ada** `State 'stop-sigterm' timed out` / `SIGKILL` / `Failed with result 'timeout'`.

Hari ini: `Stopping 06:56:58` → `stop-sigterm timed out 06:58:28` (90s) → `SIGKILL`. Setelah fix, pola itu harus hilang. Karena `Failed with result 'timeout'` terbukti muncul berulang (09-07, 09-08, 09-19), hilangnya pola ini = **bukti kualitas jangka panjang**.

### V6 — Bukti jangka panjang (uji sesungguhnya)

Leak akumulatif butuh waktu (insiden kemarin butuh ~70 jam masa hidup proses). Maka:

1. **Bandingkan `polls after last heartbeat`** dengan metodologi dari audit: jalankan analisis yang sama setelah beberapa hari. Sebelumnya `pid 309109` = **98 poll / 882s** (outlier ekstrem), semua PID lain ≤6 poll (≤29s). Setelah fix, nilai ini harus tetap **≤6 poll**.
2. **Umur proses:** sebelumnya `pid 309109` hidup 69.8h hingga mati karena leak. Setelah fix, proses harus bertahan jauh lebih lama tanpa heartbeat mati.
3. **Alert dini:** pertimbangkan monitoring sederhana pada metrik V1 (`len(pool._used)`) sehingga jika ada leak baru (mis. dari kode yang belum ditulis), terdeteksi sebelum bot mati.

---

# Temuan Terpisah (DI LUAR SCOPE — butuh keputusan, jangan dibundel)

Saat membaca source, ditemukan **bug kedua yang berbeda** dan **tidak boleh dicampur** ke patch ini:

**`commit()` absen pada write path, sementara koneksi berjalan `autocommit=0`.**

Bukti:
- `SELECT @@autocommit` pada koneksi aiomysql = **0** (meskipun global MariaDB = 1) — diverifikasi lewat probe read-only.
- 7 lokasi leak **tidak ada satu pun** yang memanggil `conn.commit()`.
- Write path lain juga tanpa commit — mis. `db.py:98 track_user_db()` (dipanggil setiap pesan user non-admin) juga **tidak** punya `commit()`.
- **Konsekuensi terukur:** `SELECT MAX(last_seen) FROM bot_users` = **`2026-09-11 16:57:21`** dan `SELECT MAX(updated_at) FROM fat_sales` = **`2026-09-11 16:40:42`** — data write membeku **8 hari** padahal bot aktif. Bandingkan `material_transactions` = `2026-09-17` (jalur itu **punya** `commit()` di `db.py:554`).

**Implikasi untuk patch ini:**
- Patch pool **tidak memperbaiki** dan **tidak memperburuk** hal ini. Behavior tetap rollback (sekarang via socket close, nanti via `release()`→`conn.close()` karena `in_trans=True`).
- **JANGAN** menambahkan `commit()` di patch ini: itu perubahan **perilaku data** (write mulai persisten), di luar scope "hanya perbaiki pool leak", dan berpotensi mengubah isi whitelist/blacklist yang dipakai handler lain.

**Saran:** tangani sebagai **task terpisah** setelah patch pool stabil dan terverifikasi. Saat itu, keputusan eksplisit diperlukan: apakah `/ban`, `/addwl`, `/lead`, `/notif`, dan akumulasi `fat_sales.wo_count` **memang** dimaksudkan untuk persisten. Bukti historis (`Com_commit=616`, `material_transactions` punya commit, komentar "Simpan ke fat_sales DB (API menjadi sumber truth)") mengindikasikan **niatnya ya**, tapi ini keputusan pemilik sistem, bukan keputusan patch.

---

# Ringkasan

| Item | Nilai |
|---|---|
| **Akar masalah** | `await pool.acquire()` dipakai sebagai coroutine + `conn.close()` → `pool.release()` tidak pernah jalan → `pool._used` penuh → `acquire()` hang permanen |
| **Jumlah lokasi** | **7** (terkonfirmasi dengan grep, tidak ada bentuk leak lain) |
| **Jumlah file** | **4** |
| **Read vs write** | 6 read (murni pool) + 1 write (`moratel.py:603`, hot path) |
| **Bentuk patch** | Struktural saja: `get_pool()` + `async with pool.acquire() as conn:` − `finally: conn.close()` |
| **Perubahan query/reply/template** | **Nol** |
| **Konvensi acuan** | `config.py:69-71`, `admin.py:147-149/186-189/232-235` (sudah benar) |
| **Bahaya utama** | `from db import pool` (harus `get_pool()`); `finally: conn.close()` tertinggal; re-indent `total_data`/`active_users` |
| **Uji kunci** | 21× per handler + heartbeat 30s tidak boleh putus + `len(pool._used)` kembali 0 |
| **Rollback** | Bukan git repo → wajib backup `cp -a` di Fase 0 |
| **Isu terkait (terpisah)** | `commit()` absen → write rollback sejak 2026-09-11. **Jangan dibundel** |

> **Rencana ini belum dieksekusi.** Tidak ada file di `/opt/suoy_bot` yang diubah, tidak ada service yang di-restart, tidak ada DML dijalankan. Seluruh probe database bersifat `SELECT`; probe koneksi ke MariaDB dibuat lalu ditutup tanpa memodifikasi data.
