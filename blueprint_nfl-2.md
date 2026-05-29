# 🦻 NFL — Nafal Faturizki Listener
### *"Mendengar itu gratis dan hak semua orang"*

> **by Nafal Faturizki**
> Open Source Hearing Aid — Zero Vendor Dependency
> Lisensi: CERN-OHL-S v2 (Hardware) + GPL-3.0 (Software)

---

## 🌟 Visi & Filosofi

Pendengaran bukan kemewahan. Di dunia yang penuh suara, jutaan manusia terpaksa bisu dari kehidupan sekitar mereka — bukan karena takdir, tapi karena harga. Proyek NFL lahir dari satu keyakinan sederhana:

> **Teknologi yang menyelamatkan tidak seharusnya menjadi bisnis eksklusif.**

NFL dirancang dari awal untuk:
- **Zero vendor lock-in** — tidak ada chip proprietary, tidak ada SDK berbayar, tidak ada cloud wajib
- **Dapat dirakit siapa saja** — teknisi, pelajar, komunitas desa, NGO
- **Fully offline** — semua proses di device, privasi pengguna terjaga
- **Bahasa Rust** — memory safe, bare-metal capable, tanpa garbage collector

---

## 📐 Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────────┐
│                     PERANGKAT NFL UNIT                      │
│                                                             │
│  [MEMS Mic] ──► [I2S] ──► [nRF5340 App Core] ──► [I2S] ──► [Speaker BA]
│                                   │                         │
│                            [DSP Pipeline]                   │
│                            (Rust/Embassy)                   │
│                                   │                         │
│                              [BLE 5.2]                      │
│                            [Net Core]                       │
└───────────────────────────────────┼─────────────────────────┘
                                    │ GATT Custom Profile
                                    │ (NFL UUID Space)
                        ┌───────────▼──────────────┐
                        │   APLIKASI NFL ANDROID    │
                        │   (Kotlin + Jetpack)      │
                        │                           │
                        │  • Scan & Pair Device     │
                        │  • Kalibrasi EQ (8 band)  │
                        │  • Hearing Test (PTA)     │
                        │  • Firmware OTA Update    │
                        │  • Manajemen Profil       │
                        └───────────────────────────┘
```

---

## 🔩 Spesifikasi Hardware

> Bagian ini hanya referensi komponen. Desain PCB, skema, dan casing **tidak masuk ke dalam struktur repositori** — dikelola sebagai dokumen terpisah (KiCad project, Gerbers, STL).

### Bill of Materials (BOM) Ringkas

| # | Komponen | Part Number | Spesifikasi Kunci |
|---|---|---|---|
| 1 | Mikrofon MEMS | Knowles SPH0645LM4H-B | I2S digital, SNR 65dB |
| 2 | Microcontroller | Nordic nRF5340 (SiP) | Dual-core ARM Cortex-M33, BLE 5.2, 1MB Flash, 512KB RAM |
| 3 | Amplifier Headphone | TI TPA6132A2 | 25mW/ch, Class-G, THD < 0.01% |
| 4 | Speaker Balanced Armature | Knowles ED-29689 | 200Hz–10kHz, 32Ω |
| 5 | Baterai LiPo | Cellevia LP401730 | 180mAh, siklus >500x |
| 6 | IC Pengisi Daya | TI BQ25185 | USB-C PD, 1A charge |
| 7 | Regulator Tegangan | TI TPS62840 | 1.8V/3.3V, quiescent 60nA |
| 8 | Flash Eksternal | Winbond W25Q64JV | 8MB SPI NOR, -40°C~85°C |
| 9 | PCB | 4-layer FR-4, ENIG | 22mm × 18mm |

**Estimasi biaya rakitan mandiri:** ~Rp 420.000/unit | **Produksi batch 1000:** ~Rp 180.000–220.000/unit

### Estimasi Konsumsi Daya

| Mode | Konsumsi | Daya Tahan (180mAh) |
|---|---|---|
| Normal (BLE aktif) | ~8.5 mA | ~21 jam |
| Normal (BLE standby) | ~5.2 mA | ~34 jam |
| Deep sleep | ~0.8 mA | ~9 hari |

---

## 💻 BLUEPRINT FIRMWARE

### Prinsip Firmware NFL
- **No binary blobs** — semua dikompilasi dari source
- **Bare-metal** — Embassy async runtime, tanpa OS
- **No cloud** — semua kalkulasi di device
- **Fixed-point DSP** — tidak ada floating point di hot path
- **Reproducible build** — siapapun bisa rebuild biner identik

### Bahasa & Toolchain

| Komponen | Pilihan |
|---|---|
| Bahasa | Rust (edition 2021) |
| Async Runtime | Embassy (bare-metal) |
| Target | `thumbv8m.main-none-eabihf` |
| HAL | `embassy-nrf` (nRF5340 App Core) |
| Logging | `defmt` via RTT |
| Flash & Debug | `probe-rs` |

---

### 🗂️ Struktur Repositori Firmware

```
firmware/
├── Cargo.toml                   # Manifest + workspace deps
├── .cargo/
│   └── config.toml              # Target: thumbv8m.main-none-eabihf
├── memory.x                     # Linker script nRF5340
│
└── src/
    ├── main.rs                  # Entry point, Embassy executor
    ├── lib.rs                   # Re-export public modules
    │
    ├── audio/
    │   ├── capture.rs           # I2S driver — SPH0645 MEMS mic
    │   ├── pipeline.rs          # Orkestrasi DSP pipeline
    │   ├── playback.rs          # I2S output — TPA6132A2 amp
    │   └── dsp/
    │       ├── noise_gate.rs    # Adaptive noise gate
    │       ├── equalizer.rs     # 8-band parametric EQ (fixed-point)
    │       ├── compressor.rs    # Dynamic range compressor
    │       └── filters.rs       # High-pass & limiter IIR filter
    │
    ├── ble/
    │   ├── gatt_server.rs       # Setup GATT server & advertising
    │   ├── advertising.rs       # BLE advertising payload
    │   └── profiles/
    │       ├── audio_config.rs  # Service: EQ, preset, noise gate
    │       ├── device_info.rs   # Service: versi, baterai, stats
    │       └── ota.rs           # Service: firmware update OTA
    │
    ├── storage/
    │   ├── flash.rs             # Driver SPI NOR W25Q64JV
    │   ├── profile.rs           # Simpan/load UserProfile
    │   └── config.rs            # Konfigurasi default & NVS
    │
    ├── power/
    │   ├── manager.rs           # State machine sleep/wake
    │   └── battery.rs           # Baca level baterai ADC
    │
    └── hal/
        ├── i2s.rs               # Abstraksi peripheral I2S
        ├── spi.rs               # Abstraksi peripheral SPI
        └── gpio.rs              # GPIO (tombol, LED status)
```

---

### 🔊 DSP Pipeline Detail

Pipeline audio berjalan di **App Core** (120MHz). **Net Core** eksklusif untuk BLE stack.

**Spesifikasi pipeline:**
- Sample rate: 16 kHz
- Frame size: 256 sampel = 16ms per frame
- Target latency: < 15ms end-to-end
- Aritmetik: fixed-point (`fixed` crate) — tidak ada `f32` di hot path

**Alur pemrosesan per frame:**

```
Input I2S  ──►  NOISE GATE  ──►  HIGH-PASS  ──►  8-BAND EQ  ──►  COMPRESSOR  ──►  LIMITER  ──►  Output I2S
(SPH0645)       noise_gate.rs    filters.rs      equalizer.rs    compressor.rs   filters.rs     (TPA6132A2)
```

| Blok | Implementasi | Parameter |
|---|---|---|
| **Noise Gate** | Adaptive threshold | -50dBFS default, adjustable via BLE |
| **High-Pass** | IIR Butterworth order 2 | Cutoff 100Hz, buang angin & gemuruh |
| **8-Band EQ** | Parametric IIR biquad | Bands: 250, 500, 1k, 2k, 3k, 4k, 6k, 8k Hz — gain ±20dB s/d +40dB |
| **Compressor** | Soft-knee 4:1 | Attack 10ms, Release 100ms |
| **Limiter** | Hard clip | Ceiling -6dBFS, proteksi absolut |

**Format profil pengguna (64 bytes, muat 1000+ profil di flash 8MB):**

```rust
#[repr(C, packed)]
struct UserProfile {
    magic: u16,           // 0xNF4C — validasi
    version: u8,
    profile_id: u8,
    name: [u8; 16],       // Nama profil UTF-8
    eq_gains: [i8; 8],    // Gain per band (dB)
    noise_gate_db: i8,    // Threshold noise gate
    compression: u8,      // Rasio x10 (10 = 1.0x, 40 = 4.0x)
    environment: u8,      // 0=Calm, 1=Indoor, 2=Outdoor, 3=Noisy, 4=Custom
    reserved: [u8; 31],
    checksum: u16,        // CRC-16
}
```

---

### 📡 BLE GATT Profile Custom

UUID Base: `4e464c00-xxxx-xxxx-xxxx-xxxxxxxxxxxx` (NFL = 0x4e, 0x46, 0x4c)

```
Service: NFL Audio Config  (4e464c01-...)
├── EQ Profile Write        (4e464c02-...)  Write/WriteNoRsp
│     Format: [i8 x8] — gain per band dalam dB
├── Environment Preset      (4e464c03-...)  Read/Write
│     0=Calm  1=Indoor  2=Outdoor  3=Noisy  4=Custom
├── Noise Gate Threshold    (4e464c04-...)  Read/Write
│     Format: i8 (-80 to 0 dBFS)
└── Compression Ratio       (4e464c05-...)  Read/Write
      Format: u8 (10=1.0x, 40=4.0x)

Service: NFL Device Info   (4e464c10-...)
├── Firmware Version        (4e464c11-...)  Read
├── Battery Level           (4e464c12-...)  Read/Notify (1 menit)
└── Device Stats            (4e464c13-...)  Read
      Uptime, error count, MCU temperature

Service: NFL OTA Update    (4e464c20-...)
├── OTA Control             (4e464c21-...)  Write
│     0x01=Start  0x02=Confirm  0x03=Cancel
├── OTA Data                (4e464c22-...)  Write
│     Chunks 512 bytes/chunk + CRC-32 per chunk
└── OTA Status              (4e464c23-...)  Notify
      Progress (%), error code
```

---

### 🔄 Proses OTA Firmware Update

```
App                          Firmware
 │                               │
 │── START_OTA ─────────────────►│  Aktifkan slot update di flash
 │◄─ ACK ────────────────────────│
 │                               │
 │── DATA chunk[0] + CRC32 ─────►│  Tulis ke flash, verifikasi CRC
 │◄─ CHUNK_OK ───────────────────│
 │── DATA chunk[1] + CRC32 ─────►│
 │◄─ CHUNK_OK ───────────────────│
 │   ... (N chunks) ...          │
 │── SHA-256 hash final ────────►│  Verifikasi integritas total
 │◄─ HASH_OK ────────────────────│
 │── APPLY ────────────────────►│  Reboot ke slot baru
 │                               │  [Jika boot gagal → auto rollback]
```

File distribusi `.bin` dapat disebarkan via GitHub Releases atau dikirim langsung dari app — **tidak ada server update wajib**.

---

### ⚙️ Dependencies Firmware (Cargo.toml)

```toml
[dependencies]
embassy-executor  = { version = "0.5", features = ["arch-cortex-m"] }
embassy-nrf       = { version = "0.1",  features = ["nrf5340-app-s"] }
embassy-time      = { version = "0.3" }
embassy-sync      = { version = "0.5" }
nrf-softdevice   = { version = "0.1",  features = ["ble-peripheral"] }
heapless         = "0.8"    # Fixed-size collections, no heap
fixed            = "1.23"   # Fixed-point arithmetic DSP
defmt            = "0.3"    # Logging efisien embedded
defmt-rtt        = "0.4"
panic-probe      = "0.3"
cortex-m         = "0.7"
cortex-m-rt      = "0.7"
micromath        = "0.2"    # Math no_std
```

> **Tidak ada dependency proprietary.** Semua crate dari crates.io, open source.

---

## 📱 BLUEPRINT APLIKASI ANDROID

### Prinsip Aplikasi NFL Android
- **Native Android** — Kotlin + Jetpack Compose, tidak ada framework lintas platform
- **No cloud, no account** — semua data lokal di Room database
- **BLE-first** — komunikasi penuh via custom GATT NFL
- **Offline capable** — fitur utama berjalan tanpa internet
- **Material You** — mengikuti design system Android terbaru

### Stack Teknologi

| Layer | Pilihan |
|---|---|
| Bahasa | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Arsitektur | MVVM + Clean Architecture |
| DI | Hilt |
| BLE | Android BLE API + custom wrapper |
| Database | Room (SQLite lokal) |
| Async | Kotlin Coroutines + Flow |
| Navigasi | Jetpack Navigation Compose |

---

### 🗂️ Struktur Repositori Android

```
android/
├── build.gradle.kts             # Project-level build config
├── settings.gradle.kts
│
└── app/
    ├── build.gradle.kts         # App-level: deps, minSdk 26, targetSdk 35
    ├── src/main/
    │   ├── AndroidManifest.xml  # Permissions: BLUETOOTH_SCAN, CONNECT, FINE_LOCATION
    │   │
    │   └── java/id/nfl/listener/
    │       │
    │       ├── data/
    │       │   ├── ble/
    │       │   │   ├── NflBleManager.kt         # Koneksi, scan, GATT ops
    │       │   │   ├── NflGattClient.kt          # Read/Write/Notify per characteristic
    │       │   │   ├── NflGattUuids.kt           # Konstanta UUID (match firmware)
    │       │   │   └── OtaManager.kt             # Orkestrasi proses OTA
    │       │   │
    │       │   ├── db/
    │       │   │   ├── NflDatabase.kt            # Room database instance
    │       │   │   ├── ProfileDao.kt             # CRUD profil EQ
    │       │   │   └── DeviceDao.kt              # Simpan device yang pernah dipair
    │       │   │
    │       │   └── repository/
    │       │       ├── DeviceRepository.kt       # Sumber data tunggal untuk device & BLE
    │       │       └── ProfileRepository.kt      # Sumber data tunggal untuk profil EQ
    │       │
    │       ├── domain/
    │       │   ├── model/
    │       │   │   ├── NflDevice.kt              # Model device BLE
    │       │   │   ├── EqProfile.kt              # 8 band gain + metadata
    │       │   │   ├── Audiogram.kt              # Threshold per frekuensi
    │       │   │   └── FirmwareInfo.kt           # Versi, status OTA
    │       │   │
    │       │   └── usecase/
    │       │       ├── ConnectDeviceUseCase.kt
    │       │       ├── SendEqProfileUseCase.kt   # Kirim EQ via BLE
    │       │       ├── RunHearingTestUseCase.kt  # Orkestrasi tes PTA
    │       │       ├── GenerateEqFromAudiogramUseCase.kt  # NAL-NL2
    │       │       └── UpdateFirmwareUseCase.kt  # Orkestrasi OTA
    │       │
    │       └── ui/
    │           ├── MainActivity.kt
    │           ├── navigation/
    │           │   └── NflNavGraph.kt            # Routing antar screen
    │           │
    │           ├── screen/
    │           │   ├── scan/
    │           │   │   ├── ScanScreen.kt         # Temukan & pair device
    │           │   │   └── ScanViewModel.kt
    │           │   │
    │           │   ├── home/
    │           │   │   ├── HomeScreen.kt         # Dashboard utama
    │           │   │   └── HomeViewModel.kt      # Status koneksi, baterai
    │           │   │
    │           │   ├── equalizer/
    │           │   │   ├── EqualizerScreen.kt    # Slider 8 band + preset
    │           │   │   └── EqualizerViewModel.kt # Debounce kirim ke BLE
    │           │   │
    │           │   ├── hearingtest/
    │           │   │   ├── HearingTestScreen.kt  # PTA step-by-step
    │           │   │   ├── HearingTestViewModel.kt
    │           │   │   └── AudiogramScreen.kt    # Tampilkan hasil audiogram
    │           │   │
    │           │   ├── profiles/
    │           │   │   ├── ProfileListScreen.kt  # Daftar profil tersimpan
    │           │   │   ├── ProfileDetailScreen.kt
    │           │   │   └── ProfileViewModel.kt
    │           │   │
    │           │   └── firmware/
    │           │       ├── FirmwareScreen.kt     # Info versi + tombol update
    │           │       └── FirmwareViewModel.kt  # Progress OTA, error handling
    │           │
    │           └── component/
    │               ├── EqBandSlider.kt           # Slider vertikal per band
    │               ├── AudiogramChart.kt         # Grafik audiogram kanan/kiri
    │               ├── BatteryIndicator.kt
    │               ├── ConnectionStatusBar.kt
    │               └── OtaProgressDialog.kt
```

---

### 📲 Screen & Flow Aplikasi

#### 1. Scan & Pair Screen
Pengguna pertama kali membuka app atau menambah device baru.

```
[Mulai Scan]
     │
     ▼
Tampilkan daftar device BLE terdekat dengan nama "NFL-xxxx"
     │
User tap device
     │
     ▼
Koneksi GATT ──► Baca Device Info Service
     │
     ▼
Home Screen (dashboard)
```

#### 2. Home Screen (Dashboard)
Titik masuk utama setelah terkoneksi.

```
┌─────────────────────────────────┐
│  🔋 87%   NFL-A3B2   ✅ Connected│
├─────────────────────────────────┤
│  Profil Aktif: "Kantor"         │
│  Environment: Indoor            │
├─────────────────────────────────┤
│  [🎚 Equalizer]  [👂 Hearing Test] │
│  [📁 Profil]     [⚙ Firmware]   │
└─────────────────────────────────┘
```

#### 3. Equalizer Screen
Inti fitur — setting EQ dan environment preset.

```
┌─────────────────────────────────────────┐
│  Equalizer                              │
│                                         │
│  Preset: [Calm][Indoor][Outdoor][Noisy] │
│                                         │
│  250  500  1k   2k   3k   4k   6k   8k │
│   ▲    ▲    ▲    ▲    ▲    ▲    ▲    ▲  │
│   │    │    │    │    │    │    │    │  │  Slider vertikal
│  +6  +10  +12  +14  +14  +12  +8   +4  │  (−20 s/d +40 dB)
│                                         │
│  Noise Gate: ────●─────── (-45 dBFS)    │
│  Compression: ──────●──── (3.0x)        │
│                                         │
│  [💾 Simpan Profil]  [↗ Kirim ke Device]│
└─────────────────────────────────────────┘
```

**Behavior penting:**
- Slider menggunakan debounce 150ms sebelum kirim ke BLE — tidak spam GATT write
- Perubahan langsung terdengar di device (latency BLE + firmware < 30ms total)
- "Simpan Profil" menyimpan ke Room DB lokal dan juga push ke flash firmware

#### 4. Hearing Test Screen
Pure Tone Audiometry (PTA) standar ISO 8253-1.

```
Frekuensi diuji: 250Hz → 500Hz → 1kHz → 2kHz → 3kHz → 4kHz → 6kHz → 8kHz
                 (Kanan dulu, lalu Kiri)

Per frekuensi — Modified Hughson-Westlake:
1. Mulai dari 40 dB HL
2. Terdengar → turun 10 dB
3. Tidak terdengar → naik 5 dB
4. Threshold = level terendah terdengar 2/3 percobaan

Output: Audiogram (dB HL per frekuensi, kanan & kiri)
         → GenerateEqFromAudiogramUseCase (NAL-NL2)
         → EqProfile otomatis
         → Push ke device via BLE
```

**Kalibrasi earphone:** App menyertakan koreksi untuk earphone umum (Apple EarPods, Samsung AKG, 3.5mm generic) agar hasil akurat tanpa audiometer eksternal.

#### 5. Firmware Screen
Manajemen versi dan OTA update.

```
┌─────────────────────────────────┐
│  Firmware                       │
│                                 │
│  Versi terpasang:  v0.2.1       │
│  Build date:       2026-05-01   │
│                                 │
│  [📂 Pilih File .bin]           │
│  Hash SHA-256: ████████████     │
│                                 │
│  [▶ Mulai Update]               │
│                                 │
│  Progress: ████████░░░ 73%      │
│  Status: Mengirim chunk 148/203 │
└─────────────────────────────────┘
```

---

### 🔵 BLE Layer Android

`NflBleManager.kt` adalah wrapper di atas Android BLE API yang mengekspos coroutine-friendly API:

```kotlin
// Contoh penggunaan di ViewModel
class EqualizerViewModel(
    private val bleManager: NflBleManager,
    private val sendEqProfile: SendEqProfileUseCase
) : ViewModel() {

    // Debounce: jangan spam BLE setiap slider gerak
    private val _pendingProfile = MutableStateFlow<EqProfile?>(null)

    init {
        viewModelScope.launch {
            _pendingProfile
                .filterNotNull()
                .debounce(150)
                .collect { profile ->
                    sendEqProfile(profile)
                }
        }
    }

    fun onSliderChanged(band: Int, gainDb: Int) {
        _pendingProfile.value = currentProfile.copy(
            gains = currentProfile.gains.toMutableList()
                .also { it[band] = gainDb }
        )
    }
}
```

**Status koneksi** di-expose sebagai `Flow<ConnectionState>` sehingga semua screen reaktif terhadap disconnect/reconnect otomatis.

---

### 💾 Model Data Lokal (Room)

```kotlin
@Entity(tableName = "eq_profiles")
data class EqProfileEntity(
    @PrimaryKey val id: String,           // UUID lokal
    val name: String,
    val gains: String,                    // JSON "[0,6,10,12,14,12,8,4]"
    val noiseGateDb: Int,
    val compressionRatio: Float,
    val environment: Int,                 // 0–4
    val createdAt: Long,
    val lastUsedAt: Long
)

@Entity(tableName = "paired_devices")
data class DeviceEntity(
    @PrimaryKey val address: String,      // MAC address BLE
    val name: String,
    val firmwareVersion: String,
    val lastConnectedAt: Long,
    val activeProfileId: String?
)
```

---

### ⚙️ Permissions Android (AndroidManifest.xml)

```xml
<!-- BLE — wajib API 31+ -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation"/>
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"/>

<!-- Lokasi — diperlukan BLE scan API < 31 -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>

<!-- Baca file .bin dari storage untuk OTA -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32"/>
```

**minSdk: 26** (Android 8.0) — mencakup >95% perangkat aktif Indonesia.

---

## 🗺️ ROADMAP PENGEMBANGAN

### Phase 0 — Fondasi (Minggu 1–3)
**Target: Lingkungan development siap, kedua repo bisa di-build**

- [ ] Setup Rust toolchain nRF5340 (`rustup target add thumbv8m.main-none-eabihf`)
- [ ] Setup Embassy + probe-rs, hello world LED berkedip di nRF5340-DK
- [ ] Buat project Android baru: Kotlin, Compose, Hilt, Room, BLE permission
- [ ] Struktur folder sesuai blueprint (firmware/ dan android/)
- [ ] CI/CD dasar: GitHub Actions build firmware + lint Android

---

### Phase 1 — Audio Core Firmware (Minggu 4–8)
**Target: Suara masuk, diproses DSP, keluar. Latency < 20ms.**

- [ ] Driver I2S capture SPH0645 (`audio/capture.rs`)
- [ ] Driver I2S playback TPA6132A2 (`audio/playback.rs`)
- [ ] Pipeline passthrough (mic → speaker, tanpa DSP)
- [ ] Ukur latency dengan oscilloscope
- [ ] Implementasi noise gate (`dsp/noise_gate.rs`)
- [ ] Implementasi 8-band IIR EQ fixed-point (`dsp/equalizer.rs`)
- [ ] Implementasi compressor & limiter (`dsp/compressor.rs`, `filters.rs`)
- [ ] Host simulator (`tools/host_sim/`) untuk test pipeline tanpa hardware

---

### Phase 2 — BLE Firmware + Android Basic (Minggu 9–14)
**Target: App Android bisa konek dan kirim EQ ke device**

**Firmware:**
- [ ] GATT server setup (`ble/gatt_server.rs`)
- [ ] Service Audio Config: write EQ profile via BLE (`ble/profiles/audio_config.rs`)
- [ ] Service Device Info: baca versi & baterai (`ble/profiles/device_info.rs`)
- [ ] Storage profil ke flash W25Q64JV (`storage/flash.rs`, `storage/profile.rs`)
- [ ] Load profil aktif saat boot

**Android:**
- [ ] `NflBleManager.kt` — scan, connect, disconnect
- [ ] `NflGattClient.kt` — write EQ, read device info, subscribe notify baterai
- [ ] Scan Screen & Home Screen (dashboard)
- [ ] Equalizer Screen: 8 slider + preset lingkungan, kirim ke BLE real-time

---

### Phase 3 — Hearing Test & OTA (Minggu 15–22)
**Target: App bisa generate profil EQ dari tes pendengaran + update firmware OTA**

**Android:**
- [ ] Hearing Test Screen — alur PTA per frekuensi dengan audio dari earphone
- [ ] `GenerateEqFromAudiogramUseCase` — algoritma NAL-NL2 (Kotlin pure)
- [ ] Audiogram Screen — grafik visual hasil tes
- [ ] Profile List & Detail Screen
- [ ] `OtaManager.kt` — kirim `.bin` dalam chunks, tracking progress
- [ ] Firmware Screen — pilih file, progress bar, error handling

**Firmware:**
- [ ] Service OTA: terima chunks, verifikasi CRC per chunk (`ble/profiles/ota.rs`)
- [ ] Verifikasi SHA-256 total sebelum apply
- [ ] Auto rollback jika boot dari slot baru gagal
- [ ] Power manager: sleep mode & wake on BLE (`power/manager.rs`)

---

### Phase 4 — Polish & Validasi (Minggu 23–28)
**Target: Siap dipakai oleh orang non-teknis**

- [ ] UX polish: onboarding, error message yang jelas, haptic feedback
- [ ] Test dengan 20+ pengguna berbagai tipe gangguan pendengaran
- [ ] Validasi algoritma EQ dengan audiolog volunteer
- [ ] Pengujian ketahanan firmware: 72 jam continuous
- [ ] Instrumentasi: log error ke file lokal (bukan cloud)
- [ ] Dokumentasi in-app: panduan hearing test, penjelasan slider EQ

---

### Phase 5 — Open Source Launch (Bulan 8+)
**Target: Dunia bisa fork, build, dan rakit sendiri**

- [ ] Publish ke GitHub: CERN-OHL-S v2 (hardware) + GPL-3.0 (software)
- [ ] Release APK di GitHub Releases (tidak wajib Google Play)
- [ ] Video tutorial assembly + penggunaan app (bahasa Indonesia + English)
- [ ] Dokumentasi lengkap: `docs/getting-started.md`, `docs/architecture.md`
- [ ] Komunitas Discord / Telegram untuk builder

---

## 🧪 Standar Kualitas Target

| Metrik | Target NFL v1 |
|---|---|
| Latency firmware (mic → speaker) | < 15ms |
| Latency BLE write → firmware apply | < 30ms tambahan |
| EQ range | −20 dB s/d +40 dB per band |
| THD (Total Harmonic Distortion) | < 2% |
| Noise floor | < −55 dBFS |
| Battery life | 20+ jam (BLE aktif) |
| OTA success rate | > 99% (auto rollback jika gagal) |
| Android minSdk | 26 (Android 8.0) |
| Ukuran APK | < 15 MB |

---

## ⚖️ Lisensi

| Layer | Lisensi | Artinya |
|---|---|---|
| **Firmware** | GNU GPL v3.0 | Modifikasi harus tetap open source |
| **Aplikasi Android** | GNU GPL v3.0 | Modifikasi harus tetap open source |
| **Dokumentasi** | CC BY-SA 4.0 | Bebas digunakan dengan atribusi |

> **Tidak ada bagian dari proyek ini yang boleh dipatenkan atau ditutup.**

---

## 💬 Penutup

Alat bantu dengar bukan kemewahan. Jutaan manusia hari ini tidak bisa mendengar tawa anaknya, suara ibunya — bukan karena tidak ada teknologinya, tapi karena ada yang memutuskan teknologi itu hanya untuk mereka yang bisa membayar.

NFL adalah penolakan terhadap premis itu. Satu firmware, satu aplikasi, satu orang yang kembali bisa mendengar dunianya — itu sudah cukup untuk memulai.

> *"Mendengar itu gratis dan hak semua orang."*

---

**Nafal Faturizki** · Inisiator Proyek NFL
Versi Blueprint: 2.0.0 · 2026
*Lisensi dokumen: CC BY-SA 4.0*
