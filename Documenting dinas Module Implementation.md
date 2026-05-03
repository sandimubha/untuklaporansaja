# Chat Conversation

Note: _This is purely the output of the chat conversation and does not contain any raw data, codebase snippets, etc. used to generate the output._

### User Input

analisa project
Carikan saya kode 
1. Menu kelola Cuti di side bar
2. Logic Perhitungan Durasi dan logika aturannya
3. Fungsi Validasi (Error Handling)
4. Halaman Canvas Cuti

buatkan saya laporan ini beserta kodenya , untuk saya jadikan laporan 

*Grep searched codebase*

*Viewed [CutiController.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/app/Http/Controllers/CutiController.php) *

*Viewed [CutiService.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/app/Services/CutiService.php) *

*Viewed [kelola.blade.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views/cuti/kelola.blade.php) *

### Planner Response

Tentu, berikut adalah laporan analisis proyek untuk modul **Manajemen Cuti** pada aplikasi SIGAP. Laporan ini mencakup lokasi menu, logika perhitungan, fungsi validasi, dan halaman visualisasi (canvas) yang Anda minta.

---

# Laporan Analisis Modul Manajemen Cuti

## 1. Menu Kelola Cuti di Sidebar
Menu ini terletak pada file layout sidebar utama. Menu ini diproteksi menggunakan sistem *Permission* Laravel (Spatie) sehingga hanya pengguna dengan hak akses tertentu yang dapat melihatnya.

**File:** `resources/views/layouts/partials/sidebar.blade.php`
**Kode Sidebar:**
```html
<!-- Fitur: KELOLA CUTI -->
@can('nav kelola cuti')
<li class="nav-item {{ Request::is('admin/cuti*') ? 'active' : '' }}">
    <a class="nav-link {{ Request::is('admin/cuti*') ? '' : 'collapsed' }}" href="#" data-toggle="collapse" 
       data-target="#collapseCuti" aria-expanded="true" aria-controls="collapseCuti">
        <i class="fas fa-fw fa-calendar-check"></i>
        <span>KELOLA CUTI</span>
    </a>
    <div id="collapseCuti" class="collapse {{ Request::is('admin/cuti*') ? 'show' : '' }}" 
         aria-labelledby="headingCuti" data-parent="#accordionSidebar">
        <div class="bg-white py-2 collapse-inner rounded">
            <a class="collapse-item {{ Request::is('admin/cuti/kelola') ? 'active' : '' }}" 
               href="{{ route('cuti.kelola') }}">Monitoring & Seting</a>
        </div>
    </div>
</li>
@endcan
```

---

## 2. Logika Perhitungan Durasi dan Aturan Bisnis
Sistem menggunakan *Service Layer* untuk menghitung durasi cuti secara efektif. Perhitungan ini dapat membedakan antara "Hari Kerja" (mengabaikan Sabtu/Minggu & Libur Nasional) dan "Hari Kalender".

**File Service:** `app/Services/CutiService.php`
**Logika Perhitungan:**
```php
public function hitungDurasiEfektif($start, $end, $isHariKerja = true)
{
    $startDate = Carbon::parse($start);
    $endDate = Carbon::parse($end);
    $period = CarbonPeriod::create($startDate, $endDate);

    $holidays = CutiLibur::pluck('tanggal')->toArray(); // Ambil hari libur dari database
    $weekendConfig = $this->getWeekendConfig(); // Default: [Saturday, Sunday]

    $duration = 0;
    foreach ($period as $date) {
        $dateString = $date->format('Y-m-d');

        // Jika Jenis Cuti adalah 'Hari Kalender' (misal: Melahirkan), hitung semua hari
        if (!$isHariKerja) {
            $duration++;
            continue;
        }

        // Jika 'Hari Kerja', abaikan Akhir Pekan & Hari Libur Nasional
        if (in_array($date->format('l'), $weekendConfig)) continue;
        if (in_array($dateString, $holidays)) continue;

        $duration++;
    }
    return $duration;
}
```

---

## 3. Fungsi Validasi (Error Handling)
Validasi dilakukan di Controller sebelum data disimpan. Terdapat beberapa lapis pengecekan: validasi input dasar, pengecekan *overlap* (jadwal bentrok), validasi gender, dan kecukupan saldo/kuota.

**File Controller:** `app/Http/Controllers/CutiController.php`
**Kode Validasi Utama:**
```php
public function store(Request $request)
{
    // 1. Validasi Input Dasar
    $request->validate([
        'cuti_jenis_id' => 'required|exists:cuti_jenis,id',
        'tgl_mulai' => 'required|date',
        'tgl_selesai' => 'required|date|after_or_equal:tgl_mulai',
    ]);

    // 2. Validasi Overlap (Cek apakah pegawai sudah punya agenda di tanggal tersebut)
    $existing = CutiPengajuan::where('nip', $pegawai->nip)
        ->where('status', '!=', 'Ditolak')
        ->where(function ($q) use ($request) {
            $q->where('tgl_mulai', '<=', $request->tgl_selesai)
              ->where('tgl_selesai', '>=', $request->tgl_mulai);
        })->first();

    if ($existing) {
        return redirect()->back()->with('error', "Anda sudah memiliki agenda cuti pada rentang tersebut.");
    }

    // 3. Validasi Saldo (Khusus Cuti Tahunan/CT)
    if ($jenis->kode == 'CT') {
        $saldo = $this->cutiService->syncSaldoTahunan($pegawai->nip, $tahun);
        if (($saldo->sisa - $pending) < $durasi) {
            return redirect()->back()->with('error', "Saldo cuti tidak cukup.");
        }
    }
}
```

---

## 4. Halaman Canvas Cuti (Matrix Monitoring)
Halaman "Canvas" atau Matrix Monitoring ini digunakan untuk melihat sebaran cuti pegawai dalam bentuk kalender horizontal. Ini memudahkan admin melihat siapa saja yang sedang cuti secara visual.

**File View:** `resources/views/cuti/kelola.blade.php` (Tab Kalender)
**Kode Visualisasi Matrix:**
```html
<!-- Matrix Table -->
<div class="table-responsive bg-white border">
    <table class="table table-bordered table-sm">
        <thead class="bg-primary text-white text-center">
            <tr>
                <th class="sticky-col">PEGAWAI</th>
                @foreach($matrixDays as $day)
                    <th class="{{ $day->isWeekend() ? 'bg-secondary' : '' }}">
                        {{ $day->format('d') }}
                    </th>
                @endforeach
            </tr>
        </thead>
        <tbody>
            @foreach($pegawaiKanwil as $p)
                <tr>
                    <td class="sticky-col bg-white">{{ $p->nama_lengkap }}</td>
                    @foreach($matrixDays as $day)
                        @php 
                            $info = $matrixData[$p->nip][$day->format('Y-m-d')] ?? null;
                        @endphp
                        @if($info)
                            <td class="{{ $info['color'] }} text-center text-white">
                                <i class="fas fa-check"></i>
                            </td>
                        @else
                            <td class="{{ $day->isWeekend() ? 'bg-light' : '' }}"></td>
                        @endif
                    @endforeach
                </tr>
            @endforeach
        </tbody>
    </table>
</div>
```

---

### Ringkasan Teknis:
*   **Database**: Menggunakan tabel `cuti_pengajuans`, `cuti_saldos`, `cuti_jenis`, dan `cuti_liburs`.
*   **Keamanan**: Menggunakan middleware `permission` (Spatie) untuk memisahkan akses pegawai biasa (hanya lihat milik sendiri) dan Admin (bisa mengelola semua).
*   **Frontend**: Berbasis **Bootstrap 4** dengan kustomisasi CSS untuk tampilan *Sticky Column* pada Matrix dan *Stepper* status pelaksanaan.

Laporan ini disusun berdasarkan kode sumber aktif di dalam proyek SIGAP. Jika Anda membutuhkan penjelasan lebih detail pada bagian tertentu, silakan beri tahu saya!

### User Input

Sekarang analisa dan buatkan juga untuk fitur dinas luar
1. Menu kelola Cuti di side bar dan dashboard rekap dinas luar
2. kolom pilihan lebih dari satu pegawai
3. kode kolom pencarian
4. logika arsip surat 


*Viewed [README.md](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/README.md) *

*Grep searched codebase*

*Grep searched codebase*

*Grep searched codebase*

*Listed directory [views](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views) *

*Listed directory [dinas-luar](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views/dinas-luar) *

*Viewed [web.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/routes/web.php) *

*Viewed [DinasLuarController.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/app/Http/Controllers/DinasLuarController.php) *

*Viewed [index.blade.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views/dinas-luar/index.blade.php) *

*Grep searched codebase*

*Listed directory [partials](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views/layouts/partials) *

*Viewed [sidebar.blade.php](file:///c:/Users/USER/.gemini/antigravity/scratch/sigap-ver/resources/views/layouts/partials/sidebar.blade.php) *

### Planner Response

Berikut adalah laporan analisis proyek untuk fitur **Dinas Luar (Business Trip)** pada aplikasi SIGAP. Laporan ini merinci implementasi menu, dashboard statistik, fitur multi-select pegawai, serta logika pengarsipan surat.

---

# Laporan Analisis Fitur Dinas Luar

## 1. Menu Sidebar dan Dashboard Rekap
Fitur Dinas Luar terbagi menjadi dua bagian: manajemen terpusat (Admin) dan riwayat pribadi (Pegawai). Dashboard rekap menyajikan visualisasi data *real-time* mengenai status penugasan.

**File Sidebar:** `resources/views/layouts/partials/sidebar.blade.php`
*   **Admin (Rekap):** `REKAP DINAS LUAR` (mengarah ke `dinas-luar.index`)
*   **Personal (Riwayat):** `RIWAYAT DINAS LUAR` (mengarah ke `dinas-luar.personal-history`)

**Dashboard Statistik (Rekap):**
Terletak pada bagian atas halaman rekap, mencakup 3 metrik utama:
1.  **Sedang DL Hari Ini**: Menghitung total pegawai (PIC + Anggota) yang sedang bertugas hari ini.
2.  **Antrian Arsip**: Menghitung data yang sudah selesai tanggal tugasnya namun belum disetujui/diarsipkan.
3.  **Progress StarASN**: Persentase sinkronisasi data ke sistem StarASN nasional.

---

## 2. Kolom Pilihan Lebih dari Satu Pegawai
Sistem memungkinkan input satu PIC (Ketua Tim) dan banyak Anggota dalam satu Surat Tugas. Ini diimplementasikan menggunakan library **Select2** dengan fitur *AJAX loading* dan *Multiple Selection*.

**File View:** `resources/views/dinas-luar/index.blade.php`
**Kode Implementasi:**
```html
<!-- Input PIC (Single) -->
<select name="pic_nip" id="pic_nip" class="form-control select2-ajax" required>
    <option value=""></option>
</select>

<!-- Input Anggota (Multiple) -->
<select name="anggota_nip[]" id="anggota_nip" class="form-control select2-ajax" multiple="multiple">
    <!-- User bisa memilih banyak nama di sini -->
</select>
```

**Logika Penyimpanan (Controller):**
```php
// app/Http/Controllers/DinasLuarController.php
if (!empty($request->anggota_nip)) {
    // Menggunakan relasi Many-to-Many untuk menyimpan banyak anggota sekaligus
    $dl->anggota()->sync($request->anggota_nip);
}
```

---

## 3. Kode Kolom Pencarian
Pencarian dirancang untuk sangat fleksibel (*Deep Search*). Admin dapat mencari berdasarkan Nomor Surat Tugas, Nama PIC, maupun Nama Anggota yang terlibat.

**Logika Pencarian (Eloquent):**
```php
if ($request->filled('q')) {
    $search = $request->q;
    $query->where(function ($q) use ($search) {
        $q->where('no_st', 'like', "%$search%") // Cari No Surat Tugas
          ->orWhereHas('pic', function ($pq) use ($search) {
              $pq->where('nama_lengkap', 'like', "%$search%"); // Cari Nama PIC
          })
          ->orWhereHas('anggota', function ($aq) use ($search) {
              $aq->where('nama_lengkap', 'like', "%$search%"); // Cari Nama Anggota
          });
    });
}
```

---

## 4. Logika Arsip Surat
Sistem pengarsipan dalam modul ini tidak hanya memindahkan data, tetapi juga melibatkan validasi status pelaksanaan tugas.

**A. Status Pelaksanaan (Stepper):**
Sistem secara otomatis menentukan status berdasarkan tanggal hari ini:
*   **Belum DL**: Tanggal mulai > Hari ini.
*   **Sedang DL**: Hari ini berada di antara tanggal mulai dan selesai.
*   **Selesai**: Tanggal selesai < Hari ini.

**B. Logika Pengarsipan (Approval):**
Data yang sudah "Selesai" secara fisik namun belum dikonfirmasi oleh Admin akan masuk ke kategori **"Antrian Arsip"**. Proses arsip dilakukan dengan tombol `Approve`.

**Kriteria Antrian Arsip:**
```php
$menungguArsipCount = DinasLuar::where('tgl_selesai', '<', now())
    ->where('status_approval', '!=', 'approved')
    ->count();
```

**C. Sinkronisasi StarASN:**
Sebagai bagian dari *compliance*, setiap data yang diarsipkan memiliki *toggle* status StarASN (Sudah/Belum) untuk memastikan setiap penugasan telah diinput ke sistem pusat.

---

### Kesimpulan Teknis:
Modul Dinas Luar menggunakan pendekatan **Relational Integrity** yang kuat (Relasi PIC dan Anggota) serta visualisasi **Status Stepper** untuk memantau perjalanan dinas secara *real-time*. Laporan ini siap digunakan sebagai dokumentasi teknis proyek Anda.