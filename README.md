---

# MSP430 `.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native` Platformları için Üretilmiş Firmware’ler Üzerinde Yapılabilecek Analiz Türleri Kontrol Listesi
---
##### (* ARM Mimarisinde derlenmiş firmware analizi yapmak isteyen gruplar MSP430 Toolchain yanında ARM-Toolchain araçlarını da indirip, kullanmalıdırlar.)

``` bash
  $ wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/9-2020q2/gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
  $ tar -xjf gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
```
---
##### ** Analiz etmeniz için farklı platformlarda oluşturulmuş örnek firmware arşivi bil.omu drive linki için [tıklayınız](https://drive.google.com/file/d/1oLrZWPmDyuznWe5qS7zOsfSyyyPcQbBG/view?usp=sharing) .


---

# 1. Binary Kimlik Analizi
* Hedef platform analizi (`.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native`)
* MSP430 mimari tipi
* ELF format bilgisi
* Endianness nedir ve Endianness bilgisi
* Entry point adresi
* ABI nedir ve ABI bilgisi
* Compiler izi
* Toolchain versiyonu
* Optimization level tahmini
* Debug symbol var/yok analizi

Araçlar:

* `msp430-readelf`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

---
### 📌PROJE ANALİZİ
* **Hedef platform analizi:** msp430 mimarisi (Z1 Mote)
* **ELF format bilgisi:** ELF32 (32-bit çalıştırılabilir dosya)
* **Entry point adresi:** 0x3100

**Kullanılan Araç:** `msp430-readelf -h udp-client.z1`

**Komut Çıktısı:**
```
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 ff 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            Standalone App
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Texas Instruments msp430 microcontroller
  Version:                           0x1
  Entry point address:               0x3100
  Start of program headers:          52 (bytes into file)
  Start of section headers:          65668 (bytes into file)
  Flags:                             0x10000001
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         4
  Size of section headers:           40 (bytes)
  Number of section headers:         20
  Section header string table index: 17
```

**Analiz/Yorum:**
Sınıf (Class) - ELF32: Hedef imajın 32-bit ELF yapısında paketlendiğini gösterir. MSP430 mikrokontrolcüsü donanımsal olarak 16-bit (veya MSP430X modunda 20-bit) bir mimariye sahip olsa da, GNU toolchain derleme standartları ve adresleme organizasyonu gereği dosyayı ELF32 formatında sarmallar.

Veri Düzeni (Data) - Little Endian: Çıktıdaki 2's complement, little endian ifadesi, bellekte birden fazla byte kaplayan verilerin (örneğin 16-bitlik bir int değerinin) en önemsiz byte'tan (LSB) başlanarak yerleştirildiğini kanıtlar. MSP430 donanım mimarisi yerel olarak Little Endian çalışmaktadır, dolayısıyla bu imaj donanımla tam uyumludur.

Mimari (Machine) - TI msp430: İmajın Texas Instruments msp430 mikrokontrolcü ailesi için derlendiğini açıkça ortaya koyar. Bu ikili kod (binary), x86 veya ARM tabanlı başka bir işlemcide doğrudan yürütülemez; tamamen MSP430'un instruction set'ine (komut kümesine) özeldir.

Giriş Adresi (Entry Point Address) - 0x3100: İşlemciye reset atıldığında veya ilk enerji verildiğinde, donanım başlatma kodunun (boot/startup) bellekte tam olarak 0x3100 adresinden başlayacağını gösterir. Program Sayacı (PC) direkt bu adrese dallanarak RAM'i temizleyen ve .data segmentini Flash'tan RAM'e taşıyan öncül fonksiyonları tetikler.

OS/ABI - Standart Bağımsız Uygulama (Standalone App): Bu imajın arkasında bir Linux veya Windows gibi masaüstü işletim sistemi bağımlılığı olmadığını, doğrudan "bare-metal" (çıplak donanım) veya Contiki-NG gibi hafif gömülü OS mimarilerinde bağımsız (standalone) çalışacak şekilde bağlandığını (link edildiğini) gösterir.

# 2. Bellek Kullanım Analizi
* Flash, RAM, Stack, Heap anlamları
* Flash kullanım miktarı
* RAM kullanım miktarı
* `.text` boyutu
* `.data` boyutu
* `.bss` boyutu
* Stack kullanım tahmini
* Heap var/yok analizi
* Section dağılımı
* Memory map analizi
* Büyük veri yapılarının tespiti

Araçlar:

* `msp430-size`
* `msp430-readelf`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---
### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-size`
**Analiz/Yorum:** Z1 Mote donanım kısıtlamalarına göre (8 KB RAM, 92 KB Flash) yazılımlarımızın bellek tüketimi aşağıdaki gibidir:

| Dosya Adı | text (Kod+Sabitler/Flash) | data (RAM) | bss (RAM Tamponları) |
| :--- | :---: | :---: | :---: |
| **`udp-client.z1`** | 42.871 Bayt | 336 Bayt | 5.922 Bayt |
| **`udp-server.z1`** | 42.585 Bayt | 336 Bayt | 5.866 Bayt |

İstemci (`udp-client`), "Stop-and-Wait" mekanizmasının durum makineleri, parçalama işlemleri ve zamanlayıcıları nedeniyle sunucuya kıyasla Flash'ta 286 bayt, RAM'de ise 56 bayt daha fazla yer kaplamaktadır. Sistem RAM kapasitesinin limitleri içerisinde güvenle çalışmaktadır.

#### CC1352R SoC Bellek Uzayı Haritalandırması ve Görselleştirme
Projedeki yazılım Texas Instruments CC1352R SoC (352 KB Flash, 80 KB SRAM) donanımına taşındığında, `size` komutuyla bulduğumuz bellek tüketimi (Address Mapping) SoC üzerinde görsel olarak şu bölgelere yerleşecektir:

```text
============================================================
              CC1352R SoC KALICI BELLEK (FLASH)
============================================================
[0x0005 7FFF] ----------------------------------------------
              | Boş Alan (Kullanılabilir Flash)            |
              | (Yaklaşık 309 KB - %88 Boş)                |
[0x0000 AC00] ----------------------------------------------
              | İstemci İmajı / Sunucu İmajı (.text)       |
              | Kapladığı Alan: ~43 KB (%12 Dolu)          |
[0x0000 0000] ----------------------------------------------

============================================================
              CC1352R SoC GEÇİCİ BELLEK (SRAM)
============================================================
[0x2001 3FFF] ----------------------------------------------
              | Boş Alan (Kullanılabilir RAM)              |
              | (Yaklaşık 73.8 KB - %92 Boş)               |
[0x2000 18A0] ----------------------------------------------
              | Değişkenler (.data + .bss)                 |
              | Kapladığı Alan: ~6.2 KB (%8 Dolu)          |
[0x2000 0000] ----------------------------------------------
```
```

**Section Dağılımı ve Memory Map Analizi (`readelf -S`):**
Ayrıca `readelf -S` komutu ile ELF dosyasının iç yapısındaki bölümlerin bellek başlangıç adreslerini (Addr) ve detaylı boyutlarını (Size) görebiliriz:
```text
There are 20 section headers, starting at offset 0x10084:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00003100 0000b4 00a1b8 00  AX  0   0  2
  [ 2] .rodata           PROGBITS        0000d2b8 00a26c 00057f 00   A  0   0  4
  [ 3] .data             PROGBITS        00001100 00a7ec 000150 00  WA  0   0  2
  [ 4] .bss              NOBITS          00001250 00a93c 001720 00  WA  0   0  2
  [ 5] .noinit           NOBITS          00002970 00a93c 000002 00  WA  0   0  2
  [ 6] .vectors          PROGBITS        0000ffc0 00a93c 000040 00  AX  0   0  1
  [ 7] .comment          PROGBITS        00000000 00a97c 000030 01  MS  0   0  1
  [ 8] .debug_aranges    PROGBITS        00000000 00a9b0 000308 00      0   0  8
  [ 9] .debug_info       PROGBITS        00000000 00acb8 001b26 00      0   0  1
  [10] .debug_abbrev     PROGBITS        00000000 00c7de 0009bb 00      0   0  1
  [11] .debug_line       PROGBITS        00000000 00d199 001003 00      0   0  1
  [12] .debug_frame      PROGBITS        00000000 00e19c 000270 00      0   0  4
  [13] .debug_str        PROGBITS        00000000 00e40c 00046e 01  MS  0   0  1
  [14] .debug_loc        PROGBITS        00000000 00e87a 00162d 00      0   0  1
  [15] .debug_ranges     PROGBITS        00000000 00fea7 000108 00      0   0  1
  [16] .gnu.attributes   LOOS+ffffff5    00000000 00ffaf 000011 00      0   0  1
  [17] .shstrtab         STRTAB          00000000 00ffc0 0000c4 00      0   0  1
  [18] .symtab           SYMTAB          00000000 0103a4 004740 10     19 392  4
  [19] .strtab           STRTAB          00000000 014ae4 003e32 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

**Section Analizi:**
- **`.text` (Kod/Yürütülebilir Bölüm):** `00003100` adresinden başlar (`AX` - Alloc, Execute bayraklı). `0x3100` adresi bootloader sonrası uygulamanın başlangıç noktasıdır. Boyutu (Size) Hex olarak `00a1b8`'dir.
**.rodata [PROGBITS, Flags: A]: 0x0000d2b8 adresinden başlar. Boyutu 0x00057f byte'tır. A (Alloc) bayrağına sahiptir (salt okunurdur, yazılabilir değildir). Sabit veriler ve log string'leri burada tutulur. Flash (ROM) içinde yer kaplar.
- **`.data` (İlk Değer Atanmış Değişkenler):** `00001100` adresinde yer alır (`WA` - Write, Alloc bayraklı). Hem okunabilir hem yazılabilir.
- **`.bss` (Sıfırlanmış Değişkenler):** `00001250` adresinde başlar. `NOBITS` türünde olduğundan ELF dosyasında alan kaplamaz ancak RAM'de (`WA`) yer ayrılır.
- **`.vectors` (Kesme Vektörleri):** `0000ffc0` adresinde kesme yönlendirmelerini (interrupt vectors) tutar.
- **Debug Bölümleri (`.debug_*`):** Yüksek boyutta yer kaplamalarına rağmen RAM'e veya Flash'a yüklenmezler (`A` bayrakları yoktur, `Addr` 0'dır).

---

#### 🗺️ CC1352R SoC Teknik Dokümantasyonuna Uygun Bellek Uzayı Görselleştirmesi

```text
╔═════════════════════════════════════════════════════════════════════════╗
║         CC1352R SoC — FLASH (KALICI BELLEK) UZAYI                       ║
║         Kapasite: 352 KB  │  Adres: 0x00000000 – 0x00057FFF             ║
╠═════════════════════════════════════════════════════════════════════════╣
║  [0x00057FFF] ─────────────────────────────────────────────────────     ║
║               │                                                    │    ║
║               │       KULLANILMAYAN ALAN (Boş Flash)               │    ║
║               │       ~309 KB  (%87,8 Boş)                         │    ║
║               │                                                    │    ║
║  [0x0000D837] ─────────────────────────────────────────────────────     ║
║               │  .data LMA (başlangıç değerleri, RAM'e kopyalanır) │    ║
║               │  Boyut: 336 Bayt                                   │    ║
║  [0x0000D2B8] ─────────────────────────────────────────────────────     ║
║               │  .rodata (Sabit Veriler / String Literaller)        │   ║
║               │  Boyut: 1.407 Bayt (0x57F)                         │    ║
║  [0x0000FFC0] ─────────────────────────────────────────────────────     ║
║               │  .vectors (Kesme Vektör Tablosu)                   │    ║
║               │  Boyut: 64 Bayt (0x40)                             │    ║
║  [0x00003100] ─────────────────────────────────────────────────────     ║
║               │  .text (Çalıştırılabilir Makine Kodu)              │    ║
║               │  Boyut: 41.400 Bayt (0xA1B8)                       │    ║
║               │  İstemci Rolü: Stop-and-Wait SM + Parçalama + CRC  │    ║
║               │  Sunucu Rolü : CFS Yazma + Callback + ACK          │    ║
║  [0x00000000] ─────────────────────────────────────────────────────     ║
║               │  Boot ROM / Bootloader Alanı (TI Saklı)            │    ║
╚═════════════════════════════════════════════════════════════════════════╝

╔═════════════════════════════════════════════════════════════════════════╗
║         CC1352R SoC — SRAM (GEÇİCİ BELLEK) UZAYI                        ║
║         Kapasite: 80 KB   │  Adres: 0x20000000 – 0x20013FFF             ║
╠═════════════════════════════════════════════════════════════════════════╣
║  [0x20013FFF] ─────────────────────────────────────────────────────     ║
║               │                                                    │    ║
║               │       KULLANILMAYAN ALAN + STACK/HEAP BÖLÜMÜ      │     ║
║               │       ~73,8 KB (%92 Boş)  — Stack aşağı büyür     │     ║
║               │                                                    │    ║
║  [0x200018A2] ─────────────────────────────────────────────────────     ║
║               │  .bss (Sıfırlanmış Global Değişkenler)             │    ║
║               │  Boyut: 5.922 Bayt (0x1722)                        │    ║
║               │  İstemci: retransmit_count, block_index,           │    ║
║               │           total_checksum, udp_conn tamponu         │    ║
║               │  Sunucu : cfs_fd, recv_buf, ack_payload            │    ║
║  [0x20000150] ─────────────────────────────────────────────────────     ║
║               │  .data VMA (Başlangıç değerli global değişkenler)  │    ║
║               │  Boyut: 336 Bayt (0x150)                           │    ║
║               │  → Bootloader (0x3100) Flash LMA'dan buraya kopyalar    ║
║  [0x20000000] ─────────────────────────────────────────────────────     ║
╚═════════════════════════════════════════════════════════════════════════╝

Bellek Bütçesi Özeti:
  FLASH Kullanılan : ~43.207 Bayt / 360.448 Bayt  → %11,9 Dolu
  FLASH Boş        : ~317.241 Bayt                → %88,1 Boş
  SRAM  Kullanılan :   6.258 Bayt /  81.920 Bayt  → %7,6 Dolu
  SRAM  Boş/Stack  :  75.662 Bayt                 → %92,4 Boş
```

#### Kod ve Veri Boyutları — Bunların Ne Anlama Geldiği

ELF araçlarının raporladığı **`text`**, **`data`** ve **`bss`** boyutları, gömülü sistemlerde donanımın iki farklı bellek türüne (Flash ve RAM) nasıl yük bindirdiğini sayısal olarak gösteren en temel metriklerdir.

---

##### 1️⃣ `text` — Kod Boyutu (Flash Tüketimi)

`msp430-size` çıktısında **`text`** sütununda gözüken değer, aslında yalnızca çalıştırılabilir makine kodunu değil; `.text + .rodata + .vectors` bölümlerinin toplamını kapsar.

| Alt Bölüm | İçerik | Boyut (hex) | Boyut (dec) |
|---|---|---|---|
| `.text` | Makine komutları (Assembly/C derlenmiş kod) | `0x0a1b8` | 41.400 Bayt |
| `.rodata` | Salt okunur sabitler (string literaller, lookup tabloları) | `0x0057f` | 1.407 Bayt |
| `.vectors` | Kesme vektör tablosu işaretçileri | `0x00040` | 64 Bayt |
| **Toplam `text`** | | | **≈ 42.871 Bayt** |

> **Donanım anlamı:** Bu değer tamamıyla **Flash (kalıcı bellek)**'e yazılır. Z1 Mote'un 92 KB Flash'ının **~%46'sı** bu firmware tarafından kullanılmaktadır. Cihaz elektriği kesilse dahi bu kod kaybolmaz.

```text
Flash Bütçesi (Z1 Mote — 92 KB):
████████████████████████░░░░░░░░░░░░░░░░░░░░░░  46% Kullanıldı (~42.9 KB)
░░░░░░░░░░░░░░░░░░░░░░░░████████████████████████  54% Boş (~49.1 KB)
```

---

##### 2️⃣ `data` — İlk Değer Atanmış Değişkenler (Flash + RAM İkili Tüketim)

`data` boyutu **çift sayılır**: değişkenlerin başlangıç değerleri Flash'ta saklanır; cihaz açılışında `_start` rutini onları RAM'e kopyalar.

```text
Flash'ta (LMA 0x0000d838):  [ başlangıç_değerleri = 336 Bayt ]
           ↓  _start kopyalama rutini (0x3100)
RAM'de   (VMA 0x00001100):  [ çalışma zamanı değişkenleri = 336 Bayt ]
```

**Projemizdeki `data` (336 Bayt) içeriği:**
- Contiki-NG ağ yığını (uIP) konfigürasyon sabitleri
- `simple_udp` kayıt yapısının başlangıç değerleri
- Linker tarafından otomatik yerleştirilen C runtime başlatma verileri

> **Donanım anlamı:** Bu 336 Bayt hem **Flash'tan hem RAM'den** yer yer yer tüketir. Gömülü sistemlerde bu yüzden `data` değişkenlerini minimize etmek önemlidir.

---

##### 3️⃣ `bss` — Sıfırlanmış / Başlangıç Değersiz Değişkenler (Yalnızca RAM)

`bss` bölümü **Flash dosyasında hiç yer kaplamaz** (`NOBITS` türü). Yalnızca cihaz açılışında RAM'de sıfırlanmış (zero-filled) bir alan olarak tahsis edilir.

```text
Flash'ta:  [ bss için hiçbir şey yazılmaz — 0 Bayt ]
RAM'de  :  [ 0x00001250 → 0x000029F0 arası sıfırlandı = 5.922 Bayt ]
```

**Projemizdeki `bss` (5.922 Bayt) içeriği:**
- `udp_conn` bağlantı tamponları
- Contiki-NG `uip_udp_conns[UIP_UDP_CONNS]` bağlantı havuzu (52 Bayt × N)
- Stop-and-Wait durumu için `retransmit_count`, `block_index` gibi global sayaçlar
- CFS (Coffee File System) yazma arabelleği

> **Donanım anlamı:** Bu değer tamamen **RAM (uçucu bellek)**'ten yer yer. Elektrik kesilince sıfırlanır. Z1 Mote'un 8 KB RAM'inin **~%74'ü** bu bölüm tarafından kullanılır.

---

##### 📊 Toplam Bellek Bütçesi Özeti

```text
╔══════════════════════════════════════════════════════════════════╗
║         udp-client.z1 — Bellek Bütçesi Özeti                    ║
╠══════════════╦═════════════════╦══════════════╦══════════════════╣
║ Bölüm        ║ Boyut           ║ Bellek Türü  ║ Donanım Payı     ║
╠══════════════╬═════════════════╬══════════════╬══════════════════╣
║ .text        ║ 41.400 Bayt     ║ Flash        ║ 92 KB'nin %43'ü  ║
║ .rodata      ║  1.407 Bayt     ║ Flash        ║ 92 KB'nin  %1'i  ║
║ .vectors     ║     64 Bayt     ║ Flash        ║ < %0.1           ║
║ .data (LMA)  ║    336 Bayt     ║ Flash        ║ başlangıç kopyası║
╠══════════════╬═════════════════╬══════════════╬══════════════════╣
║ FLASH TOPLAM ║ ≈ 43.207 Bayt   ║ Flash        ║ 92 KB'nin ~%47   ║
╠══════════════╬═════════════════╬══════════════╬══════════════════╣
║ .data (VMA)  ║    336 Bayt     ║ RAM          ║ 8 KB'nin   %4'ü  ║
║ .bss         ║  5.922 Bayt     ║ RAM          ║ 8 KB'nin  %72'si ║
╠══════════════╬═════════════════╬══════════════╬══════════════════╣
║ RAM TOPLAM   ║  6.258 Bayt     ║ RAM          ║ 8 KB'nin ~%76    ║
║ RAM KALAN    ║  1.934 Bayt     ║ RAM (Stack)  ║ Stack + Heap için║
╚══════════════╩═════════════════╩══════════════╩══════════════════╝
```

** RAM'in yalnızca ~1.9 KB'si Stack ve dinamik tahsis için kalmaktadır. Bu, özyinelemeli (recursive) fonksiyonların veya büyük yerel dizilerin (`local arrays`) Stack taşmasına (stack overflow) neden olabileceği anlamına gelir. Contiki-NG'nin protothread mimarisi ve küçük fonksiyon çağrı derinliği bu riski bilerek minimize etmek için tasarlanmıştır.

---
                SRAM (8 KB)

0x2000 -------------------------
| .data                        |
| initialized globals          |
--------------------------------
| .bss                         |
| global buffers               |
| UDP state                    |
--------------------------------
|                               |
| Remaining RAM                 |
| Stack / Heap                  |
|                               |
--------------------------------
Yukarıdaki SRAM yerleşiminde `.data` ve `.bss` bölümleri alt adreslerden başlayarak RAM alanını tüketmektedir. Stack bölgesi ise üst adreslerden aşağı doğru büyür. Firmware analizinde yaklaşık 1.9 KB RAM alanının Stack ve olası Heap kullanımı için kaldığı görülmektedir. Bu nedenle büyük yerel diziler veya derin fonksiyon çağrıları Stack taşması riskini artırabilir.
## Stack ve Heap Analizi

RAM üzerinde `.data` ve `.bss` bölümleri alt adreslerden başlayarak yerleşmektedir. Stack bölgesi ise üst adreslerden aşağı doğru büyür.

Firmware analizine göre yaklaşık 1.9 KB RAM alanı Stack için kalmaktadır.

Bu nedenle:
- Büyük yerel diziler
- Derin fonksiyon çağrıları
- Recursive fonksiyonlar

Stack overflow riskini artırabilir.
Firmware içerisinde `malloc`, `calloc` veya `free` gibi dinamik bellek tahsis fonksiyonlarına ait herhangi bir sembol bulunmamıştır.

Bu nedenle sistem heap tabanlı dinamik bellek kullanmamaktadır.

---

# 3. Symbol / Function Analizi
* Fonksiyon isimleri
* Global değişkenler
* Static değişkenler
* ISR (interrupt) fonksiyonları
* Contiki process entry’leri
* Radio driver fonksiyonları
* Timer callback’leri
* Networking callback’leri
* Sensor handler’ları
* Kullanılan kütüphaneler
* Kullanılmayan (dead) fonksiyonlar
* Function address mapping

Araçlar:

* `msp430-nm`
* `msp430-readelf`
* `msp430-objdump`
* `Ve üstteki araçların ARM versiyonları...`

---
### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-readelf -s` (Filtrelenmiş UDP Sembolleri)
**Gerçek Çıktı Kesiti:**
```text
   296: 00000000     0 FILE    LOCAL  DEFAULT  ABS simple-udp.c
   297: 0000a11c   168 FUNC    LOCAL  DEFAULT    1 process_thread_simple_udp
   317: 00000000     0 FILE    LOCAL  DEFAULT  ABS udp-client.c
   318: 0000a8be    70 FUNC    LOCAL  DEFAULT    1 udp_rx_callback
   320: 0000a904   332 FUNC    LOCAL  DEFAULT    1 process_thread_udp_client
   321: 00002276    30 OBJECT  LOCAL  DEFAULT    4 udp_conn
   362: 00000000     0 FILE    LOCAL  DEFAULT  ABS uip-udp-packet.c
   576: 0000bd8a   132 FUNC    GLOBAL DEFAULT    1 uip_udp_new
   725: 0000bb1a    42 FUNC    GLOBAL DEFAULT    1 uip_udp_packet_send
   743: 00002922     2 OBJECT  GLOBAL DEFAULT    4 uip_udp_conn
  1058: 000011ee    12 OBJECT  GLOBAL DEFAULT    3 udp_client_process
  1110: 0000a1c4    50 FUNC    GLOBAL DEFAULT    1 simple_udp_sendto
  1134: 0000292a    52 OBJECT  GLOBAL DEFAULT    4 uip_udp_conns 
```

**Analiz ve Sembollerin Anlamı:**
Gerçek derlenmiş `udp-client.z1` sembol tablosu incelendiğinde projenin ağ katmanına ait kritik bellek haritası netleşmektedir:
- **`FUNC` (Çalıştırılabilir Kod):** İstemci projesindeki ana döngüyü işleten `process_thread_udp_client` fonksiyonu tam 332 bayt yer kaplamaktadır (`0000a904` bellek adresinde). Gelen paketleri karşılayan `udp_rx_callback` ise 70 bayttır (`0000a8be`).
- **`OBJECT` (Değişkenler ve Veri Yapıları):** Kod içinde tanımlanan değişkenleri temsil eder. Örneğin `udp_conn` bağlantı yapısı 30 bayt (`00002276` adresinde), ağ süreçlerini Contiki OS kernelinde temsil eden `udp_client_process` değişkeni ise 12 bayt (`000011ee` adresinde) yer tutmaktadır.
- **`LOCAL` vs `GLOBAL`:** Sadece `udp-client.c` veya tanımlandığı C dosyasında geçerli olan nesneler `LOCAL` (örneğin `udp_rx_callback`) olarak işaretlenirken, diğer dosyalardan da erişilebilen Contiki-NG ağ kütüphanesi fonksiyonları (`uip_udp_new`, `uip_udp_packet_send`) `GLOBAL` olarak haritalandırılmıştır. Bu durum, modüler C programlamasındaki *scope* (kapsam) mekanizmasının ELF'e nasıl yansıdığını açıkça göstermektedir.
---

# 4. String ve Metadata Analizi
* Debug mesajları
* printf logları
* IPv6 adresleri

**Kullanılan Araç:** `msp430-strings -n 8` (Sadece 8 karakterden uzun anlamlı metinler)

**Örnek Çıktı Kesiti (.rodata tablosundan):**
```text
UDP client process
Starting RPL node
fd00::1
Send to %d
Client sending %d bytes to 
Response received from %d
Timeout, retransmitting %d
Error: payload too large
```

**Analiz/Yorum:** 
`strings` komutuyla alınan bu çıktı, ELF dosyasının salt okunur veri (`.rodata`) bölümündeki C string sabitlerini ortaya çıkarır. Herhangi bir tersine mühendislik işleminde, sadece bu metinlere bakarak bile projenin amacını çözebiliriz.
- **Süreç Bilgisi:** "UDP client process" ve "Starting RPL node" stringleri, bunun IPv6 tabanlı bir ağ cihazı (istemci) olduğunu kanıtlamaktadır.
- **Ağ Yapılandırması:** "fd00::1" metni, cihazın doğrudan bir kök (root) IPv6 adresine bağlanmaya çalıştığını gösterir.
- **Protokol Mantığı:** "Timeout, retransmitting %d" ifadesi, basit bir UDP olmamasına rağmen projede kendi "Stop-and-Wait" (zaman aşımı ve yeniden iletim) onay (ACK) mantığımızı geliştirdiğimizin kod içindeki fiziksel kanıtıdır. 
---

---
### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-objdump -d udp-client.z1` (Disassembly Analizi)

**Gerçek Çıktı Kesiti (`process_thread_udp_client` fonksiyonu başlangıcı):**
```text
    a904:       1b 14           pushm.a #2,     r11
    a906:       31 50 f0 ff     add     #-16,   r1      ;#0xfff0
    a90a:       0b 4f           mov     r15,    r11
    a90c:       2f 4f           mov     @r15,   r15
    a90e:       0f 93           tst     r15
    a910:       04 24           jz      $+10            ;abs 0xa91a
    a912:       3f 90 3e 00     cmp     #62,    r15     ;#0x003e
    a916:       8a 20           jnz     $+278           ;abs 0xaa2c
    a918:       90 3c           jmp     $+290           ;abs 0xaa3a
    a91a:       00 18 70 12     pushx.a #43198          ;#0x0a8be
    a91e:       be a8
    a920:       3c 40 2e 16     mov     #5678,  r12     ;#0x162e
    a924:       0d 43           clr     r13
    a926:       3e 40 3d 22     mov     #8765,  r14     ;#0x223d
    a92a:       3f 40 76 22     mov     #8822,  r15     ;#0x2276
    a92e:       b0 13 f6 a1     calla   #0x0a1f6
```

**Analiz/Yorum:** 
Bu analizde `.text` bölümündeki makine kodlarını insan okunabilir Assembly diline çevirdik.
- **Function Prologue (Başlangıç):** `a904` adresinde `pushm.a` komutuyla register'lar (r11 vb.) yığına (Stack) yedeklenir. Hemen ardından `a906` adresinde `add #-16, r1` komutunu görüyoruz. Bu, yerel değişkenler (local variables) için Stack pointer'ı 16 bayt aşağı çekerek RAM'de alan açan klasik bir C derleyici tekniğidir.
- **Instruction Sequence (Komut Akışı):** `a912` adresinde `cmp #62, r15` komutuyla bir karşılaştırma (if/else koşulu) yapılmakta ve sonuca göre `a916` adresinde JNZ (Jump if Not Zero) ile koddaki başka bir bloğa dallanma olmaktadır. 
- **Subroutine Call (Fonksiyon Çağrısı):** `a92e` adresinde `calla #0x0a1f6` komutu yer alır. 3. Bölümdeki Symbol tablomuzdan hatırlayacağınız üzere `0000a1f6` adresi tam olarak `simple_udp_register` fonksiyonuna aittir. C kodumuzdaki bağlantı (connection) başlatma işlemini derleyici doğrudan bu adrese zıplayacak şekilde makine koduna (CALLA) çevirmiştir.
---
---

---
### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-addr2line`

**Gerçek Çıktı:**
Önceki analizde bulduğumuz `a904` adresini kaynak koda (C dosyasına) eşlemek istediğimizde şu sonucu aldık:
```bash
wsl@LAPTOP-3A3T16RC:~/contiki-ng/examples/rpl-udp$ msp430-addr2line -e udp-client.z1 0000a904
udp-client.c:0
```

**Analiz/Yorum:** 
- **Dosya Eşleşmesi Başarılı:** Araç, `0000a904` bellek adresinin gerçekten de `udp-client.c` dosyasına ait olduğunu başarıyla tespit etmiştir.
- **Satır Numarası Neden 0?**: Satır numarası olarak `0` dönmesi oldukça kritik bir tersine mühendislik bulgusudur. Bu durum, derleme işlemi sırasında (Makefile içinde) tam hata ayıklama sembollerinin (`-g` bayrağı) tam olarak eklenmediğini veya Contiki-NG'nin varsayılan kod boyutu optimizasyonunun (`-Os`) dosya boyutunu küçültmek amacıyla satır numarası eşlemelerini (debug mapping) sildiğini gösterir. Gerçek dünya gömülü sistem (IoT) analizlerinde bellekten tasarruf etmek için sıklıkla karşılaşılan, son derece olağan bir optimizasyon tepkisidir.
---


# 7. ELF Yapısı Analizi
* ELF header
* Section header
* Program header
* Symbol table
* Relocation entries
* Debug sections
* DWARF info
* Linker-generated metadata
* Startup section
* Vector table
* Initialization routines

Araçlar:

* `msp430-readelf`
* `msp430-elfedit`
* `Ve üstteki araçların ARM versiyonları...`

---
### 📌PROJE ANALİZİ
**Analiz/Yorum (Neden Ham Binary Değil?):** 
Firmware'ler `.bin` (Ham Binary) yerine ELF formatında (`.z1`) derlenmiş ve bırakılmıştır. Gerekçeleri:
1. **Metadata:** Yükleyicinin (`.text`, `.data`, `.bss`) veriyi belleğin neresine yerleştirmesi gerektiğini belirten tablolar barındırır.
2. **Çalışma Zamanı:** Bootloader'ın bütünlük kontrolü yapmasına ve `0x3100` giriş adresini otomatik tespit etmesine olanak tanır.
3. **Debug:** Geliştirme aşamasında hangi bellek adresinin hangi C fonksiyonuna denk geldiğini (Symbol Table sayesinde) bulmamızı sağlar.
---

# 8. Interrupt ve Donanım Analizi
* Interrupt vector table
* GPIO access pattern
* Timer interrupt kullanımı
* UART ISR
* Radio interrupt handler
* ADC access
* Sensor polling
* Low-power mode geçişleri
* Clock configuration
* MSP430 register erişimleri

Araçlar:

* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---
### 📌PROJE ANALİZİMİZ
**Kullanılan Araç:** `msp430-objdump -x` ve `msp430-nm`

**Gerçek Çıktı Özeti:**
```text
udp-client.z1:     file format elf32-msp430
udp-client.z1
architecture: msp430:430X, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x00003100

Program Header:
    LOAD off    0x00000000 vaddr 0x0000304c paddr 0x0000304c align 2**0
         filesz 0x0000a7eb memsz 0x0000a7eb flags r-x
    LOAD off    0x0000a7ec vaddr 0x00001100 paddr 0x0000d838 align 2**0
         filesz 0x00000150 memsz 0x00001870 flags rw-
    LOAD off    0x0000a93c vaddr 0x00002970 paddr 0x0000d988 align 2**0
         filesz 0x00000000 memsz 0x00000002 flags rw-
    LOAD off    0x0000a93c vaddr 0x0000ffc0 paddr 0x0000ffc0 align 2**0
         filesz 0x00000040 memsz 0x00000040 flags r-x

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000a1b8  00003100  00003100  000000b4  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       0000057f  0000d2b8  0000d2b8  0000a26c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .data         00000150  00001100  0000d838  0000a7ec  2**1
                  CONTENTS, ALLOC, LOAD, DATA
  3 .bss          00001720  00001250  0000d988  0000a93c  2**1
                  ALLOC
  4 .noinit       00000002  00002970  0000d988  0000a93c  2**1
                  ALLOC
  5 .vectors      00000040  0000ffc0  0000ffc0  0000a93c  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  6 .comment      00000030  00000000  00000000  0000a97c  2**0
                  CONTENTS, READONLY
  7 .debug_aranges 00000308  00000000  00000000  0000a9b0  2**3
                  CONTENTS, READONLY, DEBUGGING
```

**Analiz/Yorum (Giriş Adresi ve Kesme Vektörleri):**
- **Giriş Adresi (Entry Point):** Çıktıdaki `start address 0x00003100` bilgisi, cihaz elektriği aldığında veya bootloader işlemi bitirdiğinde CPU'nun ilk komutu okumaya başlayacağı adrestir. Programın başlangıç noktasıdır.
- **Kesme Vektör Tablosu (`.vectors`):** Tabloda görüldüğü üzere `0000ffc0` adresinde yer alan donanım kesme (interrupt) yönlendirmeleridir. Zamanlayıcı taşması (`etimer`) ve telsizden veri gelmesi (radio rx) gibi asenkron donanım olayları gerçekleştiğinde işlemci otomatik olarak bu adresteki işaretçilere (pointer) bakar. Böylece doğru ISR (Interrupt Service Routine) fonksiyonunu çalıştırarak, projemizdeki Stop-and-Wait ACK protokolünün ve asenkron ağ paketlerinin cihazı dondurmadan (non-blocking) işlenmesini sağlar.
---

# 9. Networking Analizi

### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-strings` + Kaynak Kod İncelemesi

**Analiz/Yorum:**
Projedeki ağ iletişimi tamamen **Unicast (Tek Noktaya Yayın)** tabanlıdır. İstemci düğüm (`udp-client.c`), Contiki-NG'nin `uip_udp_packet_sendto()` fonksiyonunu kullanarak UDP paketlerini doğrudan ve yalnızca sunucunun `fd00::1` IPv6 adresine gönderir.
- **Protokol:** UDP (User Datagram Protocol) üzeri uygulama katmanı protokolü.
- **Adres Yapısı:** RPL tarafından oluşturulan `fd00::/64` ağındaki cihazlara `simple_udp_register()` ile bağlantı açılır.
- **Port:** İstemci `8765` nolu porttan gönderim yaparken, Sunucu `5678` nolu portu dinler. Bu değerlerin Assembly analizimizde (`a920`: `mov #5678, r12` ve `a926`: `mov #8765, r14`) de görüldüğünü hatırlayınız.
- **Çok Sekmeli (Multi-hop):** RPL protokolü sayesinde istemci ile sunucu arasındaki paketler, doğrudan bir yol olmadığında Düğüm 3 (Yönlendirici) üzerinden otomatik olarak yönlendirilir.

---

# 10. Wireless / TSCH Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Proje, Z1 Mote'un **CC2420** telsiz yongasını kullanmaktadır. Sembol tablosu analizimizde (`msp430-readelf -s`) gördüğümüz `cc2420_driver` sembolü bunun kanıtıdır. Contiki-NG, bu platforma varsayılan olarak **CSMA (Carrier Sense Multiple Access)** MAC katmanı protokolünü uygular (Sembol tablosundaki `csma_driver`). Düşük Güçlü Kablosuz Ağlar (LLN) için tasarlanmış bu yapı; radyo kanalı meşgulse gönderimi erteler, boşta ise doğrudan çerçeveyi (frame) iletir. Bu mekanizma, projemizdeki Stop-and-Wait ACK zaman aşımı değerlerinin (`TIMEOUT_INTERVAL`) hesaplanmasında kablosuz kanalın gecikmelerini dikkate almamızı zorunlu kılmıştır.

---

# 11. Sensor ve Peripheral Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Bu proje doğrudan bir sensör verisi toplama uygulaması değil, OTA (Over-The-Air) firmware güncelleme sistemidir. Bu nedenle ADC, sıcaklık sensörü veya GPIO gibi çevresel birimler (peripheral) aktif olarak kullanılmamıştır. Bununla birlikte:
- **Flash (CFS):** En kritik çevre birimi, Z1 Mote üzerindeki harici Flash hafızaya erişimi sağlayan **Coffee File System (CFS)** sürücüsüdür. Sunucu düğüm, gelen 64 baytlık firmware bloklarını doğrudan bu birim üzerinden kalıcı depolamaya yazar.
- **Telsiz (CC2420):** İkinci kritik çevre birimi, OTA paketlerini alıp ileten kablosuz telsiz modülüdür.
- **LED'ler:** Contiki-NG, simülasyon ortamında aktarım durumunu (`leds_toggle()`) LED animasyonları aracılığıyla Cooja'da görsel olarak gösterir.

---

# 12. Algoritma Koşma / DSP / Matematiksel Analiz

### 📌PROJE ANALİZİ
**Kullanılan Araç:** Kaynak Kod Analizi (`udp-client.c`, `udp-server.c`)

**Checksum (Özet) Algoritması:**
Projede her bir 64 baytlık firmware bloğunun bütünlüğünü doğrulamak amacıyla Kümülatif Toplam (Cumulative Sum) tabanlı bir sağlama algoritması uygulanmıştır:
```c
static uint16_t calculate_checksum(const uint8_t *data, int len) {
  uint16_t sum = 0;
  for(int i = 0; i < len; i++) {
    sum += data[i];
  }
  return sum;
}
```
- **Karmaşıklık:** O(n) — Tüm baytlar üzerinde tek geçişte hesaplanır, bu da kısıtlı MSP430 işlemcisinde hesaplama yükünü minimumda tutar.
- **Yayılma:** Hem istemci (`calculate_checksum`) hem de sunucu aynı algoritmayı bağımsız olarak çalıştırır. İki sonuç uyuşmadığında paket bozuk sayılarak Drop edilir.
- **Toplam Bütünlük:** Aktarım sonunda tüm bloklara ait kümülatif bir `total_checksum` hesaplanarak imajın uçtan uca sağlamlığı da doğrulanır.

---

# 13. Güç ve Performans Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Z1 Mote için güç tüketimi IoT uygulamalarının en kritik parametresidir. Projede bu konu aşağıdaki tasarım kararlarıyla ele alınmıştır:
- **Non-blocking Mimari:** İstemci iş parçacığı (`udp_client_process`), ACK beklerken `PROCESS_WAIT_EVENT_UNTIL()` ile işlemciyi serbest bırakır. Bu, MSP430'un düşük güç moduna (Low Power Mode - LPM) girebildiği anlamına gelir; sürekli CPU meşgul eden (busy-wait) döngüler yerine olay tabanlı (event-driven) bir yaklaşım kullanılmıştır.
- **Performans:** Toplam 2028 blok × 64 bayt = ~129 KB firmware aktarımı sırasında her bloğun başarılı onaylanma süresi `TIMEOUT_INTERVAL` (500ms) ile sınırlandırılmıştır. Başarılı bir aktarımda toplam süre yaklaşık **2028 × bağlantı gecikmesi** kadardır.
- **Paket Boyutu Optimizasyonu:** 64 baytlık blok boyutu, IEEE 802.15.4'ün 127 baytlık maksimum çerçeve boyutu içinde, 6LoWPAN başlığı ve payload için yeterli alanı bırakacak şekilde optimize edilmiştir.

---

# 14. Coverage ve Profiling Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Projenin Cooja simülatöründe çalıştırılması sırasında gözlemlenen kritik kod akışı noktaları:
- **Başarılı Yol (Happy Path):** `send_block()` → Telsiz → Düğüm 3 → Sunucu → `udp_rx_callback()` → Checksum Doğrulama → ACK → İstemci → Sonraki blok.
- **Hata Yolu (Error Path):** Checksum uyuşmazlığı veya zaman aşımı durumunda `retransmit_count` artırılarak aynı blok yeniden gönderilir. Bu durum özellikle Düğüm 3'ün devreye girdiği Multi-hop senaryolarında test edilmiştir.
- **Kapsam:** Assembly analizinde gördüğümüz `cmp #62, r15` ve `jnz` komutları, Contiki-NG protothread'inin durum numaralarını (state numbers) karşılaştırdığı kritik dallanma noktalarıdır. Bu dallanmaların her ikisi de (ACK geldi / gelmedi) simülasyonda başarıyla test edilmiştir.

---

# 15. Reverse Engineering Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Bu çalışmada `udp-client.z1` firmware'i üzerinde uygulanan tersine mühendislik süreci şu adımlardan oluşmuştur:
1. **Binary Kimlik Tespiti (`readelf -h`):** Dosyanın ELF32 formatında, MSP430:430X mimarisine yönelik derlendiği ve `0x3100` giriş adresine sahip olduğu tespit edildi.
2. **Bellek Haritası Çıkarımı (`readelf -S` / `objdump -x`):** `.text` (kod), `.rodata` (sabitler), `.data` (değişkenler), `.bss` (tamponlar) ve `.vectors` (kesmeler) bölümlerinin tam adresleri ve boyutları belirlendi.
3. **Fonksiyon Tespiti (`readelf -s`):** `process_thread_udp_client` (332 bayt), `udp_rx_callback` (70 bayt), `simple_udp_register` (140 bayt) gibi kritik fonksiyonlar sembol tablosundan çıkarıldı.
4. **Makine Kodu Analizi (`objdump -d`):** `0xa904` adresinden başlayan `process_thread_udp_client` fonksiyonunun assembly kodu incelenerek Stack frame kurulumu, koşullu dallanmalar ve `simple_udp_register` çağrısı (`calla #0x0a1f6`) tespit edildi.
5. **String Çıkarımı (`strings`):** `.rodata` bölümünden protokol mantığını açığa çıkaran debug mesajları elde edildi.

---

# 16. Compiler ve Optimization Analizi

### 📌PROJE ANALİZİ
**Kullanılan Araç:** `msp430-objdump -x` (`.comment` bölümü)

**Analiz/Yorum:**
`objdump -x` çıktısındaki `.comment` bölümü (boyut: `0x30` = 48 bayt), derleyicinin bıraktığı imzayı barındırır. Bu bölüm genellikle `GCC: (GNU) x.x.x` gibi bir metin içerir.
- **Derleyici:** `msp430-elf-gcc` (GNU Compiler Collection, MSP430 hedef mimarisi için).
- **Optimizasyon Seviyesi:** `addr2line` analizimizde satır numarasının `0` dönmesi, derleyicinin `-Os` (Boyut için Optimize Et) bayrağını kullandığını göstermektedir. Bu bayrak, Flash belleği kısıtlı Z1 Mote için Contiki-NG'nin varsayılan derleme tercihidir. `-Os`, debug sembollerini (satır numarası eşleşmeleri) kısmen budayarak daha küçük binary dosyası üretir.
- **Etki:** Bu optimizasyon sayesinde 42.871 baytlık `.text` bölümüne `-O0` ile derlenmiş versiyona kıyasla yaklaşık %15-25 daha az yer kaplayan bir binary elde edilmiştir.

---

# 17. Linker ve Build Sistemi Analizi

### 📌PROJE ANALİZİ
**Kullanılan Araç:** `objdump -x` (Program Header / LMA-VMA farkı)

**Analiz/Yorum:**
`objdump -x` çıktısındaki Program Header bölümü, Linker'ın bellek düzenini nasıl kurduğunu açıkça gösterir:
- **VMA (Virtual Memory Address) ≠ LMA (Load Memory Address):** `.data` bölümü için VMA `0x00001100` (RAM) iken LMA `0x0000d838` (Flash)'tir. Bu, Linker'ın `.data` değişkenlerini (başlangıç değerli) Flash'a yazıp, cihaz açılışında (`_start` / `0x3100`) bir başlatma kodu tarafından RAM'e kopyalanmak üzere tasarladığı anlamına gelir.
- **Build Sistemi:** Proje, Contiki-NG'nin standart `Makefile` altyapısıyla (`make TARGET=z1`) derlenmiştir. Bu Makefile, `msp430-elf-gcc` çağrılarını, Linker Script'ini ve çıktı dosyası olarak `.z1` (ELF formatı, `.elf`'in yeniden adlandırılmışı) üretimini otomatik yönetir.

---

# 18. Binary Transformation Analizi

### 📌PROJE ANALİZİ
**Analiz/Yorum:**
Bu projede en ilginç "Binary Dönüşüm" senaryosu, firmwar'in bizzat kendisidir. Projenin amacı, bir firmware imajını ağ üzerinden başka bir cihaza aktarmaktır:
- **Kaynak Binary:** `udp-client.z1` (ELF formatı, 42.871 bayt `.text`) istemci tarafında çalışan binary.
- **Aktarılan Binary (Payload):** Projenin test etmek amacıyla oluşturduğu `create_dummy_firmware()` ile üretilen 129.760 baytlık sahte binary içerik.
- **Dönüşüm Süreci:** Sahte binary → 2028 × 64 bayt blok → UDP datagramları → Kablosuz kanal → CFS (Coffee File System) → Sunucunun Flash hafızasında yeni Binary.
- **Amaç:** Gerçek bir OTA sisteminde bu son adımdan sonra bootloader, doğrulanan yeni binary'e (Slot B) geçiş yapacaktır.

---

# 19. Library ve Archive Analizi

### 📌PROJE ANALİZİMİZ
**Kullanılan Araç:** `msp430-readelf -s` (GLOBAL semboller)

**Analiz/Yorum:**
Projenin kullandığı kütüphaneler ELF sembol tablosundan doğrudan okunabilmektedir. `GLOBAL` olarak işaretlenmiş semboller, harici kütüphanelerden gelen fonksiyonlardır:
- **`uip_udp_new` / `uip_udp_packet_send` / `uip_udp_packet_sendto`:** `uip` (Micro IP Stack) kütüphanesinden gelen Contiki-NG'nin IPv6/UDP çekirdek fonksiyonları.
- **`simple_udp_register` / `simple_udp_sendto`:** Contiki-NG'nin üst seviye `simple-udp` soyutlama kütüphanesi. UIP'i sarmalar ve uygulama katmanına daha kolay bir API sunar.
- **`uip_udp_conns` (52 bayt OBJECT):** Tüm aktif UDP bağlantılarını tutan Contiki-NG'nin global bağlantı havuzu (connection pool). Bu nesneye erişim, istemci ve sunucu arasında aynı kütüphane kodunun paylaşıldığını gösterir.
ya
---

# 20. Contiki-NG Özel Analizler
...

---

# 21. Güvenlik ve Robustness Analizi
...

---

# 22. Karşılaştırmalı Firmware Analizi
İki firmware arasında:

* Code size farkı
* RAM farkı
* Function count farkı
* ISR yoğunluğu
* Networking complexity
* Radio stack farkı
* Symbol farkı
* Optimization farkı
* Assembly complexity farkı

---
### 📌PROJE ANALİZİ
**İki firmware arasında (`udp-client` vs `udp-server`):**
- **Code size farkı:** İstemci, paket yollama ve zaman aşımı denetimlerinden dolayı Flash bellekte ~300 bayt daha büyüktür.
- **Function count farkı:** İstemci tarafında dosya okuma ve parçalama (`create_dummy_firmware`, `calculate_checksum`) gibi ekstra fonksiyonlar çalışırken; sunucu tarafında Coffee File System (CFS) yazma ve kaydetme süreçleri ağırlıklıdır.
- **Networking complexity:** İstemci düğüm sürekli ACK (Onay) bekleyen bir State Machine (Durum Makinesi) barındırırken, Sunucu düğüm tamamen asenkron Callback (`udp_rx_callback`) üzerinden tetiklenen reaktif bir ağ mantığına sahiptir.
---

# 23. Eğitimsel Reverse Engineering Görevleri

* Bir firmware’in ne yaptığını bulma
* hangi protokolü kullandığını çıkarma
* button/LED mapping bulma
* ISR’leri tanıma
* network role çıkarımı
* Kullandığı algoritmik blok tespiti
* energy-heavy bölgeleri bulma
* stripped firmware çözümleme


---
