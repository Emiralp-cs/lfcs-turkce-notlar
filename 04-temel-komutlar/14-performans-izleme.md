# Performans İzleme

> **Resmi sınav maddesi:** Monitor and troubleshoot system performance and services

"Sunucu yavaşladı" şikayeti geldiğinde ilk iş, yavaşlığın kaynağını bulmaktır: işlemci mi yetmiyor, bellek mi bitti, disk mi yetişemiyor, yoksa tek bir servis mi her şeyi tüketiyor? Bu dosya, bu soruları hangi araçla, hangi sırayla soracağını anlatıyor.

Senaryo: Kullanıcılar sunucunun yavaşladığını söylüyor. Durumu canlandırmak için sunucuya bilerek yük bindirdik: 2 çekirdekli makinede sürekli işlemci kullanan 3 süreç, 1,5 GB bellek tutan bir süreç ve diske yoğun yazan bir `dd`. Sonra araçlarla yükün kaynağını bulduk.

> [!NOTE]
> Bütün örnekler 2 çekirdekli, 8 GB bellekli bir Ubuntu 24.04 üzerinde denenmiştir. `iostat`, `mpstat`, `pidstat` ve `sar` için `sudo apt install sysstat`, `htop` için `sudo apt install htop` gerekir.

## İçindekiler

1. [Nereden başlamalı?](#1-nereden-başlamalı)
2. [Genel yük: uptime ve load average](#2-genel-yük-uptime-ve-load-average)
3. [Süreçleri canlı izlemek: top ve htop](#3-süreçleri-canlı-izlemek-top-ve-htop)
4. [En çok kaynak kullananları listelemek: ps](#4-en-çok-kaynak-kullananları-listelemek-ps)
5. [Bellek: free](#5-bellek-free)
6. [Hepsine birden bakmak: vmstat](#6-hepsine-birden-bakmak-vmstat)
7. [Çekirdek ve süreç bazında işlemci: mpstat ve pidstat](#7-çekirdek-ve-süreç-bazında-işlemci-mpstat-ve-pidstat)
8. [Disk: iostat ve pidstat -d](#8-disk-iostat-ve-pidstat--d)
9. [Servis bazında kaynak kullanımı](#9-servis-bazında-kaynak-kullanımı)
10. [Bellek bittiğinde: OOM killer](#10-bellek-bittiğinde-oom-killer)
11. [Sorunu bulduktan sonra](#11-sorunu-bulduktan-sonra)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. Nereden başlamalı?

Genelden özele git:

```
uptime              → sistem genel olarak yüklü mü?
   │
top / vmstat        → yük işlemcide mi, bellekte mi, diskte mi?
   │
ps / pidstat        → hangi süreç?
   │
systemctl status    → hangi servis? loglarında ne var?
   │
düzelt              → durdur, önceliğini düşür, yeniden başlat, sınırla
```

| Belirti | Bakılacak yer |
|---|---|
| Load average çekirdek sayısının üstünde, CPU %idle düşük | İşlemci: `top`, `mpstat`, `pidstat -u` |
| `available` bellek az, swap kullanılıyor | Bellek: `free`, `ps --sort=-%mem` |
| Load yüksek ama CPU boşta, `wa` yüksek | Disk: `iostat`, `pidstat -d` |
| Tek bir servis yavaş | `systemctl status`, `journalctl -u` |

---

## 2. Genel yük: uptime ve load average

```bash
uptime
```
```
 08:45:03 up 41 min,  0 user,  load average: 2.19, 0.74, 0.25
```

Sondaki üç sayı **load average**: son 1, 5 ve 15 dakikada, işlemci için sırada bekleyen veya çalışan ortalama süreç sayısı.

Bu sayı tek başına anlamsızdır, **çekirdek sayısıyla** karşılaştırılır:

```bash
nproc      # 2
```

| Load (2 çekirdekte) | Anlamı |
|---|---|
| 1.0 | Çekirdeklerin yarısı dolu |
| 2.0 | İki çekirdek de tam dolu, bekleyen yok |
| 4.0 | Her an 2 süreç çalışıyor, 2 süreç sırada bekliyor |

### Üç sayının farkı

Load average anlık bir değer değil, yavaş değişen bir ortalamadır. 3 işlemci yükü başlattıktan sonra:

| Zaman | Load average |
|---|---|
| 20 saniye sonra | `0.97, 0.28, 0.08` |
| 40 saniye sonra | `1.42, 0.41, 0.13` |
| 80 saniye sonra | `2.19, 0.74, 0.25` |

Yük ilk saniyeden beri tam gücüyle sürmesine rağmen 1 dakikalık değer kademeli yükseldi. Bu yüzden:

- **1 dakikalık değer 15 dakikalıktan büyükse** yük artıyor (yukarıdaki gibi).
- **1 dakikalık değer 15 dakikalıktan küçükse** yük azalıyor, sorun geçmiş olabilir.

> [!IMPORTANT]
> Linux'ta load average sadece işlemci bekleyen süreçleri değil, **diskten cevap bekleyen** süreçleri de sayar (`D` durumundakiler). Load yüksek ama `top`'ta işlemci boşta görünüyorsa sorun büyük ihtimalle disktedir ([8. bölüm](#8-disk-iostat-ve-pidstat--d)).

---

## 3. Süreçleri canlı izlemek: top ve htop

```bash
top
```

Yük altındaki ekranın üst kısmı:

```
top - 08:44:06 up 40 min,  0 user,  load average: 0.97, 0.28, 0.08
Tasks:  91 total,   4 running,  87 sleeping,   0 stopped,   0 zombie
%Cpu(s): 18.2 us, 81.8 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   8023.6 total,   5667.1 free,   2149.6 used,    449.8 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   5874.1 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
10087 root      20   0    2728   1636   1512 R 100.0   0.0   0:13.25 yes
10085 root      20   0    2728   1640   1512 R  60.0   0.0   0:12.97 yes
10089 root      20   0    2728   1640   1512 R  50.0   0.0   0:13.24 yes
```

### %Cpu satırı

| Alan | Anlamı | Yüksekse |
|---|---|---|
| `us` | Kullanıcı programları | Uygulama işlemciyi yoğun kullanıyor |
| `sy` | Çekirdek (sistem çağrıları) | Çok sayıda sistem çağrısı, sürücü yükü |
| `ni` | Önceliği düşürülmüş (nice) programlar | |
| `id` | Boşta | Düşükse işlemci dolu |
| `wa` | Disk/ağ G/Ç beklemesi | Disk yetişemiyor |
| `st` | Sanal makinede, fiziksel sunucunun başka makinelere ayırdığı süre | Sanallaştırma sunucusu aşırı yüklü |

### Süreç sütunları

| Sütun | Anlamı |
|---|---|
| `PR` / `NI` | Öncelik / nice değeri |
| `VIRT` | Sürecin ayırdığı toplam sanal bellek (gerçek kullanım değil) |
| `RES` | Gerçekten RAM'de duran kısmı. Bellek kullanımına bakarken bu sütuna bak |
| `S` | Durum: `R` çalışıyor, `S` uyuyor, `D` disk bekliyor, `Z` zombi |
| `%CPU` | Tek çekirdeğe göre yüzde. 2 çekirdekte en fazla %200 olabilir |

### top içindeki tuşlar

| Tuş | Ne yapar? |
|---|---|
| `P` | İşlemci kullanımına göre sırala (varsayılan) |
| `M` | Bellek kullanımına göre sırala |
| `1` | Her çekirdeği ayrı göster |
| `k` | Süreç öldür (PID sorar) |
| `r` | Nice değerini değiştir |
| `u` | Belirli bir kullanıcının süreçleri |
| `q` | Çık |

Script'lerde veya çıktıyı kaydetmek için tek seferlik çalıştırma: `top -bn1 -o %MEM | head -15`.

`htop` aynı bilgileri renkli, fareyle kullanılabilen bir arayüzde gösterir. Sınav makinesinde kurulu olmayabilir, `top` her yerde vardır.

---

## 4. En çok kaynak kullananları listelemek: ps

`top` canlı izler, `ps` bir anlık görüntü verir ve sonucu filtrelemek, kaydetmek kolaydır:

```bash
ps -eo pid,user,%cpu,%mem,rss,comm --sort=-%cpu | head -5
```
```
  PID USER     %CPU %MEM   RSS COMMAND
10087 root     65.3  0.0  1636 yes
10089 root     65.2  0.0  1640 yes
10085 root     63.9  0.0  1640 yes
10090 root      3.5 18.8 1546688 python3
```

```bash
ps -eo pid,user,%cpu,%mem,rss,comm --sort=-%mem | head -3
```
```
  PID USER     %CPU %MEM   RSS COMMAND
10090 root      3.5 18.8 1546688 python3
  105 root      1.1  4.4 365028 claude
```

`--sort=-%cpu` büyükten küçüğe sıralar. `rss`, `top`'taki `RES` ile aynı şeydir (KB).

> [!NOTE]
> `ps`'teki `%CPU`, sürecin **başladığından beri** ortalama kullanımıdır. `top`'taki ise son birkaç saniyenin değeridir. Uzun süredir çalışan bir süreç yeni yoğunlaştıysa `ps` bunu olduğundan düşük gösterir. Anlık durum için `top` veya `pidstat`.

`ps`, `pgrep`, `kill`, `nice` gibi süreç yönetimi komutlarının ayrıntısı: [Süreçler](../01-isletim-ve-dagitim/03-surecler.md).

---

## 5. Bellek: free

```bash
free -h
```

1,5 GB tutan süreç çalışırken:

```
               total        used        free      shared  buff/cache   available
Mem:           7.8Gi       2.1Gi       5.5Gi        26Mi       449Mi       5.7Gi
Swap:             0B          0B          0B
```

| Sütun | Anlamı |
|---|---|
| `used` | Programların kullandığı |
| `free` | Hiç kullanılmayan |
| `buff/cache` | Linux'un disk okumalarını hızlandırmak için kullandığı, gerektiğinde geri verebileceği bellek |
| `available` | Yeni programlara verilebilecek bellek, yani **asıl bakılacak sütun** |

Linux boş belleği boşa bırakmaz, önbellek olarak kullanır. Uzun süredir açık bir sunucuda `free` çok düşük görünebilir, bu normaldir. Sorun `available` düşük olduğunda ve swap kullanımı arttığında başlar.

Swap kullanımı ve `si`/`so` değerleri için `vmstat`'a bak: sürekli sıfırdan büyükse bellek yetmiyor ve sistem diske taşıyor demektir. Swap yönetimi: [Swap](../03-depolama/05-swap.md).

---

## 6. Hepsine birden bakmak: vmstat

`vmstat` tek satırda işlemci, bellek, swap ve disk durumunu gösterir. `vmstat 1 5` her saniye bir satır, 5 satır yazar. **İlk satır açılıştan beri ortalamadır**, sonrakiler o saniyenin değerleridir.

İşlemci yükü altında:

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 3  0      0 5804960  13928 446916    0    0     0     0  534  377 33 66  0  0  2  0
```

Disk yükü altında:

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  1      0 7341420  14072 447872    0    0    24 718848 2433 3129  0  7 48 44  0  0
```

| Sütun | Anlamı | İlk örnekte | İkinci örnekte |
|---|---|---|---|
| `r` | İşlemci için çalışan/bekleyen süreç | **3** (2 çekirdeğe 3 iş) | 0 |
| `b` | Disk bekleyen süreç | 0 | **1** |
| `si` / `so` | Swap'tan okunan / swap'a yazılan | 0 | 0 |
| `bi` / `bo` | Diskten okunan / diske yazılan blok | 0 | **718848** |
| `id` | İşlemci boşta | **0** | 48 |
| `wa` | Disk bekleme | 0 | **44** |

İki farklı sorunun imzası açıkça farklı: birincisinde `r` çekirdek sayısını aşmış ve `id` sıfır, ikincisinde `b`, `bo` ve `wa` yükselmiş.

---

## 7. Çekirdek ve süreç bazında işlemci: mpstat ve pidstat

`mpstat -P ALL 1 1`, her çekirdeği ayrı gösterir. Tek çekirdekli bir program bir çekirdeği %100 kullanırken diğerleri boşsa, `top`'taki genel ortalama bunu gizleyebilir:

```
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   29.85    0.00   70.15    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       0   26.00    0.00   74.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       1   33.66    0.00   66.34    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```

`pidstat -u 1 5`, süreç bazında işlemci kullanımını belirli aralıklarla ölçer. `ps`'in aksine o aralıktaki gerçek kullanımı verir:

```
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0     10087   21.78   45.54    0.00   30.69   67.33     -  yes
```

`%wait` burada sürecin işlemci sırası beklediği süredir: 3 süreç 2 çekirdeği paylaştığı için her biri zamanının yaklaşık üçte birini bekleyerek geçirdi.

---

## 8. Disk: iostat ve pidstat -d

```bash
iostat -dxz 1 3
```

`-d` sadece diskler, `-x` ayrıntılı, `-z` hiç kullanılmayan diskleri gizle. Disk yükü sırasında:

```
Device   r/s   rkB/s ...    w/s     wkB/s ... w_await ... aqu-sz  %util
vda     6.00   24.00 ... 1590.00 814080.00 ...   0.72 ...   1.20  91.60
```

| Sütun | Anlamı |
|---|---|
| `r/s`, `w/s` | Saniyedeki okuma/yazma işlemi |
| `rkB/s`, `wkB/s` | Saniyede okunan/yazılan KB |
| `r_await`, `w_await` | Bir işlemin ortalama süresi (ms). Yüksekse disk yavaş cevap veriyor |
| `aqu-sz` | Diskin kuyruğunda bekleyen ortalama işlem |
| `%util` | Diskin meşgul olduğu zaman yüzdesi |

`%util` %100'e yakınsa disk sürekli meşguldür. Hangi sürecin yazdığını bulmak için:

```bash
pidstat -d 1 3
```
```
08:45:09      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
08:45:09        0     10133     28.00 792624.00      0.00       0  dd
```

Saniyede yaklaşık 790 MB yazan süreç `dd`, PID'i 10133. Bunu bir mount noktasının neden ayrılamadığını bulmakla birleştiren tam senaryo: [Depolama Performansı](../03-depolama/11-depolama-performansi.md).

---

## 9. Servis bazında kaynak kullanımı

systemd her servisi ayrı bir kontrol grubunda (cgroup) çalıştırdığı için kaynak kullanımı servis bazında da görülebilir. Bir servisin birden fazla süreci varsa, `top`'ta dağınık görünen kullanımı tek satırda toplar.

```bash
systemctl status redis-server
```
```
      Tasks: 5 (limit: 9605)
     Memory: 3.7M ()
```

Bütün servisler, `top` gibi canlı:

```bash
systemd-cgtop                  # canlı, q ile çık
systemd-cgtop -b -n 1 --order=memory
```
```
system.slice/docker.service                     9      -    83.8M        -        -
system.slice/containerd.service                 9      -    41.3M        -        -
system.slice/system-postgresql.slice            6      -    27.6M        -        -
```

Sütunlar: süreç sayısı, işlemci, bellek, disk okuma ve yazma.

### Açılış neden yavaş?

```bash
systemd-analyze                # açılışın toplam süresi
systemd-analyze blame          # servisleri başlama sürelerine göre sıralar
systemd-analyze critical-chain # açılışı geciktiren zincir
```
```
4.938s postgresql@16-main.service
3.230s docker.service
1.726s systemd-tmpfiles-setup.service
```

### Servis bozulduysa

Performans sorunu bazen bir servisin sürekli çöküp yeniden başlamasıdır. `systemctl status`'ta `NRestarts` veya journal'da `Scheduled restart job` satırları görürsen bunun ayrıntısı [Servis Oluşturma ve Sorun Giderme](13-servis-olusturma-ve-sorun-giderme.md) dosyasında.

```bash
systemctl --failed                      # hata veren servisler
journalctl -p err -b                    # bu açılıştan beri bütün hatalar
systemctl show servis -p NRestarts      # kaç kez yeniden başlatıldı
```

---

## 10. Bellek bittiğinde: OOM killer

Bellek ve swap tamamen dolarsa çekirdek, sistemi ayakta tutmak için en çok bellek kullanan süreçlerden birini öldürür. Buna **OOM killer** (Out Of Memory) denir. Bir servis sebepsiz yere ölmüşse ilk bakılacak yerlerden biri:

```bash
journalctl -k | grep -i -E 'out of memory|oom'
sudo dmesg | grep -i oom
```

Kayıtlarda `Out of memory: Killed process 1234 (python3)` gibi bir satır, hangi sürecin neden öldürüldüğünü gösterir. Servis için `systemctl status` çıktısında `Result: oom-kill` görülür.

> [!NOTE]
> OOM durumu bu notun hazırlandığı ortamda bilerek tetiklenmedi, çünkü ortamın kendisini de etkileyebilirdi. Komutlar ve mesaj biçimi `man journalctl` ve çekirdek belgelerine göre yazılmıştır.

Bir servisin belleğini önceden sınırlamak ([Uygulama ve Servis Kısıtlamaları](16-uygulama-ve-servis-kisitlamalari.md)) OOM'un bütün sistemi etkilemesini önler.

---

## 11. Sorunu bulduktan sonra

| Durum | Seçenekler |
|---|---|
| Gereksiz veya takılmış bir süreç | `kill PID`, olmazsa `kill -9 PID` |
| Önemli ama başkalarını yavaşlatan bir iş (yedekleme, sıkıştırma) | Önceliğini düşür: `renice -n 19 -p PID` |
| Disk yükü yapan iş | `ionice -c3 -p PID` (sadece disk boştayken çalışsın) |
| Hatalı davranan servis | `systemctl restart servis`, loglarını incele |
| Bir servis sürekli fazla kaynak tüketiyor | systemd ile sınırla: `MemoryMax=`, `CPUQuota=` |
| Bellek gerçekten yetmiyor | Swap ekle, kalıcı çözüm için RAM |

`nice`, `renice`, sinyaller ve `kill` kullanımı: [Süreçler](../01-isletim-ve-dagitim/03-surecler.md).

---

## 12. Komut özeti

| Komut | Ne gösterir? |
|---|---|
| `uptime` | Load average (1, 5, 15 dk) |
| `nproc`, `lscpu` | Çekirdek sayısı |
| `top`, `htop` | Canlı süreç listesi |
| `top -bn1 -o %MEM` | Tek seferlik, belleğe göre |
| `ps -eo pid,user,%cpu,%mem,rss,comm --sort=-%cpu` | En çok işlemci kullananlar |
| `free -h` | Bellek ve swap |
| `vmstat 1 5` | İşlemci, bellek, swap, disk özeti |
| `mpstat -P ALL 1 1` | Çekirdek bazında işlemci |
| `pidstat -u 1 5` | Süreç bazında işlemci |
| `pidstat -d 1 5` | Süreç bazında disk |
| `pidstat -r 1 5` | Süreç bazında bellek |
| `iostat -dxz 1 3` | Disk bazında kullanım |
| `systemd-cgtop` | Servis bazında kaynak kullanımı |
| `systemd-analyze blame` | Açılış süreleri |
| `journalctl -p err -b` | Bu açılıştaki hatalar |
| `journalctl -k \| grep -i oom` | OOM kayıtları |

---

## 13. Sık yapılan hatalar

- **Load average'ı çekirdek sayısına bölmeden yorumlamak.** 2 çekirdekte 2.0 tam dolu, 16 çekirdekte rahat.
- **Load average'ın anlık değer olduğunu sanmak.** 1 dakikalık değer bile yük başladıktan sonra yavaş yükselir.
- **Load yüksekse sorunu hep işlemcide aramak.** `wa` ve `b` sütunlarına da bak.
- **`free` sütunu düşük diye bellek bitti sanmak.** `available`'a bak.
- **`VIRT` sütununu bellek kullanımı sanmak.** Gerçek kullanım `RES`/`RSS`.
- **`vmstat`'ın ilk satırını o anın değeri sanmak.** İlk satır açılıştan beri ortalamadır.
- **`ps`'in `%CPU`'sunu anlık kullanım sanmak.**
- **Sorunlu süreci bulmadan servisleri rastgele yeniden başlatmak.**

---

## 14. Alıştırmalar

**1.** Sistemin çekirdek sayısını ve son 1, 5 ve 15 dakikalık load average değerlerini bul. Sistem şu an aşırı yüklü mü?

<details>
<summary>Çözüm</summary>

```bash
nproc
uptime
```

Load average değerleri `nproc` sonucundan küçükse sistem aşırı yüklü değil.
</details>

**2.** Arka planda iki tane `yes > /dev/null` başlat. En çok işlemci kullanan 3 süreci `ps` ile listele, sonra `vmstat` ile `r` ve `id` sütunlarına bak. İşin bitince süreçleri durdur.

<details>
<summary>Çözüm</summary>

```bash
yes > /dev/null &
yes > /dev/null &
ps -eo pid,%cpu,comm --sort=-%cpu | head -4
vmstat 1 3
pkill yes
```

`r` sütunu 2 civarında, `id` sıfıra yakın olmalı.
</details>

**3.** Sistemde yeni programlara ne kadar bellek verilebileceğini ve en çok bellek kullanan 5 süreci bul.

<details>
<summary>Çözüm</summary>

```bash
free -h                                              # available sütunu
ps -eo pid,user,rss,%mem,comm --sort=-%mem | head -6
```
</details>

**4.** `sysstat` paketini kur. Arka planda diske yazan bir `dd` başlat, hangi sürecin saniyede ne kadar yazdığını ve diskin ne kadar meşgul olduğunu bul. Sonra dosyayı sil.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install sysstat
dd if=/dev/zero of=/tmp/test bs=1M count=3000 oflag=direct &
pidstat -d 1 3          # kB_wr/s sütununda dd
iostat -dxz 1 3         # %util
wait
rm /tmp/test
```
</details>

**5.** Sistemde en çok bellek kullanan servisleri listele ve açılışta en uzun süren 3 servisi bul.

<details>
<summary>Çözüm</summary>

```bash
systemd-cgtop -b -n 1 --order=memory | head
systemd-analyze blame | head -3
```
</details>

---

Önceki konu: [Servis Oluşturma ve Sorun Giderme](13-servis-olusturma-ve-sorun-giderme.md) · Sonraki konu: [Disk Alanı Sorunları](15-disk-alani-sorunlari.md)
