# Depolama Performansı

> **Resmi sınav maddesi:** Monitor storage performance

Bir sunucu yavaşladığında sebep çoğu zaman işlemci değil disktir: bir süreç diske aralıksız yazıyordur ve diğer herkes sırasını bekler. Bu notta tek bir olayı baştan sona izliyoruz: **hangi disk** meşgul, **hangi süreç** yazıyor, **hangi dosyaya**, nasıl durdurulur ve sonra o dosya sistemi neden ayrılamıyor.

Genel performans araçları (`top`, `vmstat`, `iostat` sütunlarının anlamı) [Performans İzleme](../01-temel-komutlar/14-performans-izleme.md) notunda. Burada disk tarafına odaklanıyoruz.

Senaryo: Sunucuda `/yedekler` (XFS, `/dev/loop0p1`) dizinine bağlı bir disk var. Kullanıcılar sunucunun ağırlaştığını söylüyor. `dolduran` kullanıcısının çalıştırdığı bir betik, sürekli büyük dosyalar yazıyor, ama bunu henüz bilmiyoruz.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denendi. Disk, kök diskteki (`vda`) bir dosyaya bağlı loop cihazıydı. Bu yüzden yazılan veri iki kez görünür: önce `loop0`'da, sonra arkasındaki `vda`'da. Gerçek bir diskte sadece disk görünür. `iostat` ve `pidstat` için `sudo apt install sysstat`.

## İçindekiler

1. [Belirti: G/Ç beklemesi](#1-belirti-gç-beklemesi)
2. [Hangi disk? iostat](#2-hangi-disk-iostat)
3. [Hangi süreç? pidstat -d](#3-hangi-süreç-pidstat--d)
4. [Hangi dosya ve dizin?](#4-hangi-dosya-ve-dizin)
5. [Durdurmak: doğru süreci öldürmek](#5-durdurmak-doğru-süreci-öldürmek)
6. ["target is busy": dosya sistemini ayırmak](#6-target-is-busy-dosya-sistemini-ayırmak)
7. [umount -l ve tehlikesi](#7-umount--l-ve-tehlikesi)
8. [Diskin hızını ölçmek: dd](#8-diskin-hızını-ölçmek-dd)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Belirti: G/Ç beklemesi

```bash
vmstat 1 4
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  1      0 6967484   2012 785200    0    0   132 1843349 4154 10686  0 21 37 42  1  0
 0  1      0 6966272   2012 785200    0    0     0 1843253 5091 12701  1 17 38 44  1  0
 0  1      0 6965932   2012 785200    0    0     0 1843253 5037 12556  0 19 38 43  1  0
```

(İlk satır açılıştan beri ortalamadır, atlandı.)

| Sütun | Değer | Anlamı |
|---|---|---|
| `b` | 1 | Diskten cevap bekleyen (`D` durumunda) bir süreç var |
| `bo` | ~1.8 milyon | Saniyede diske yazılan blok (KB). Çok yüksek |
| `wa` | 42-44 | İşlemci zamanının %43'ü diski beklemekle geçiyor |
| `us` | 0-1 | Kullanıcı programları neredeyse hiç işlemci kullanmıyor |

İşlemci boşta ama `wa` yüksek: sorun disk.

---

## 2. Hangi disk? iostat

```bash
iostat -dxyz 1 1
```

| Seçenek | Anlamı |
|---|---|
| `-d` | Sadece diskler (işlemci özeti yok) |
| `-x` | Ayrıntılı sütunlar |
| `-y` | **Açılıştan beri olan ilk raporu atla** |
| `-z` | Hiç etkinlik olmayan diskleri gösterme |
| `1 1` | 1 saniye aralıkla 1 rapor |

`-y` önemli. `iostat -dxz 1 2` ile `-y` olmadan baktığımızda ilk rapordaki `loop0` satırı şöyleydi:

```
Device   w/s     wkB/s  ...  %util
loop0    2.61  1354.72  ...   0.13
```

Disk boşta gibi görünüyor, çünkü bu açılıştan beri ortalama. Bir saniye sonraki ikinci rapor:

```
Device      w/s      wkB/s  ...  w_await  aqu-sz  %util
loop0   1573.00  804354.00  ...     0.58    1.39  74.80
```

Gerçek durum bu: saniyede ~800 MB yazılıyor ve disk zamanın %75'inde meşgul. Sadece ilk rapora bakan biri diski temiz sanar.

Hangi **bölüm** meşgul? `-p` ile:

```bash
iostat -dxyz -p loop0 1 1
```
```
Device   w/s      wkB/s      w_await  aqu-sz  %util
loop0    1893.00  967689.00  0.53     1.68    88.40
loop0p1  1893.00  967689.00  0.53     1.00    90.40
```

(Sütunlar kısaltıldı.) Bütün yazma `loop0p1`'e. Hangi dizin olduğu:

```bash
lsblk -o NAME,FSTYPE,MOUNTPOINTS /dev/loop0
```
```
NAME      FSTYPE MOUNTPOINTS
loop0
|-loop0p1 xfs    /yedekler
|-loop0p2 swap   [SWAP]
`-loop0p3 ext4   /veri
```

Sorun `/yedekler`'de. Daha okunaklı bir özet için `iostat -dyzh 1 1` (`-h` birimleri `900.0M` gibi yazar).

---

## 3. Hangi süreç? pidstat -d

```bash
pidstat -d -U 1 1
```
```
Average:        USER       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
Average:        root     33069      0.00 921792.00      0.00       0  kworker/u8:4+loop0
Average:    dolduran     36531      0.00 921600.00      0.00       0  bash
Average:    dolduran     36696      0.00   4096.00      0.00       0  dd
```

`-U` UID yerine kullanıcı adını gösterir. (`kworker/...+loop0`, loop cihazının çekirdek işçisi. Gerçek bir diskte görünmez.)

**Burada bir tuzak var:** En çok yazan süreç `dd` değil, `bash` görünüyor. Sebebi, betiğin her biri bir saniyeden kısa süren `dd`'ler çalıştırması. Bir süreç bitince yaptığı G/Ç, onu başlatan sürecin (ebeveynin) hesabına eklenir. Saniyede ~920 MB'ı yazan, döngüyle `dd` başlatan `bash`.

Sürecin ne olduğuna bak:

```bash
pgrep -ax dd
ps -o pid,ppid,user,stat,etime,cmd -p 36635
ps -o pid,user,cmd -p 36531
```
```
36635 dd if=/dev/zero of=arsiv.tmp bs=1M count=300 oflag=direct status=none
    PID    PPID USER     STAT     ELAPSED CMD
  36635   36531 dolduran D          00:00 dd if=/dev/zero of=arsiv.tmp bs=1M count=300 oflag=direct status=none
    PID USER     CMD
  36531 dolduran bash -c cd /yedekler/is && while :; do dd if=/dev/zero of=arsiv.tmp bs=1M count=300 oflag=direct status=none; done
```

`STAT D`: süreç diski bekliyor. `ELAPSED 00:00`: yeni başlamış. `PPID 36531`: onu başlatan `bash`. `bash`'in komut satırında sonsuz bir döngü (`while :`) var.

> [!TIP]
> `pgrep dd` yazma, `pgrep -x dd` yaz. `-x` olmadan adında "dd" geçen her süreç eşleşir. Denemede `pgrep -a dd`, `kthreadd` ve `fsidd`'yi de buldu.

---

## 4. Hangi dosya ve dizin?

Sürecin çalıştığı dizin:

```bash
pwdx 36531
```
```
36531: /yedekler/is
```

Sürecin açık dosyaları `/proc/PID/fd` altında. `dd`'ler bir saniyeden kısa yaşadığı için her komutta farklı bir PID görürsün, o anki `dd`'yi `pgrep -x dd` ile al:

```bash
ls -l /proc/$(pgrep -x dd)/fd
```
```
lr-x------ 1 dolduran dolduran 64 Nov 26 09:58 0 -> /dev/zero
l-wx------ 1 dolduran dolduran 64 Nov 26 09:58 1 -> /yedekler/is/arsiv.tmp
```

`dd`'nin girdisi (`0`) `/dev/zero`, çıktısı (`1`) `/yedekler/is/arsiv.tmp`. Dosyanın hangi dosya sisteminde olduğu:

```bash
df -h /yedekler/is/arsiv.tmp
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0p1    960M   56M  905M   6% /yedekler
```

Zincir tamamlandı: `iostat`'taki `loop0p1` → `/yedekler` → `/yedekler/is/arsiv.tmp` → `dd` → `bash` döngüsü → kullanıcı `dolduran`.

---

## 5. Durdurmak: doğru süreci öldürmek

Önce sadece `dd`'yi öldürdük:

```bash
kill $(pgrep -x dd)
pidstat -d -U 1 1
```

Bir saniye sonra `pidstat` yeni bir `dd` (PID 36696) gösterdi. Döngü hemen bir sonrakini başlattı. Kaynağı, yani ebeveyni durdurmak gerekir:

```bash
kill 36531              # bash döngüsü
pkill -x -u dolduran dd # o anda çalışan son dd
pgrep -a -u dolduran
iostat -dxyz -p loop0 1 1
```

`iostat`'ta artık `loop0` satırı yok: `-z` etkinlik olmayan diskleri göstermez. Disk sakinleşti.

`kill` varsayılan olarak `TERM` gönderir, sürecin düzgün kapanmasına izin verir. Süreç `TERM`'e cevap vermiyorsa `kill -9` ([Süreçler](../03-isletim-ve-dagitim/03-surecler.md)). `D` durumundaki bir süreç disk işlemi bitene kadar sinyalleri işleyemez, `kill -9` bile biraz gecikebilir.

---

## 6. "target is busy": dosya sistemini ayırmak

Şimdi diski kontrol etmek için `/yedekler`'i ayırmak istiyoruz:

```bash
sudo umount /yedekler
```
```
umount: /yedekler: target is busy.
```

Dosya sistemini hala biri kullanıyor. Kimin kullandığını bulmanın iki yolu:

```bash
sudo lsof +D /yedekler
```
```
COMMAND   PID     USER   FD   TYPE DEVICE  SIZE/OFF NODE NAME
sleep   12586     root  cwd    DIR  259,0        23  132 /yedekler/is
tail    12601 dolduran    3r   REG  259,0 314572800  133 /yedekler/is/arsiv.tmp
```

```bash
sudo fuser -vm /yedekler
```
```
                     USER        PID ACCESS COMMAND
/yedekler:           root     kernel mount /yedekler
                     root      12586 ..c.. sleep
                     dolduran  12601 f.... tail
```

| İşaret | `lsof` | `fuser` | Anlamı |
|---|---|---|---|
| Çalışma dizini | `cwd` | `c` | Süreç o dizinde duruyor (ör. `cd /yedekler/is` yapılmış bir kabuk) |
| Açık dosya | `3r`, `1w` | `f` | Süreç bir dosyayı açmış (`r` okuma, `w` yazma) |

İki ayrı sebep var:

- **Bir kabuk o dizinde duruyor** (`cwd`). Genelde unutulmuş bir terminal, bazen de kendi terminalin. Kendi terminalinse `cd /` yeterli. Başkasınınsa süreci durdur.
- **Bir süreç dosya açmış** (`tail -f` bir günlüğü izliyor). Süreci durdur.

```bash
cd /                     # kendi kabuğun oradaysa
sudo kill 12586 12601
sudo umount /yedekler
```

`fuser -km /yedekler` dosya sistemini kullanan bütün süreçlere `KILL` gönderir. Hızlıdır ama kimin ne yaptığına bakmadan öldürür, dikkatli kullan.

> [!NOTE]
> `lsof +D` dizinin altını tek tek tarar, büyük dizinlerde yavaştır. `fuser -vm` veya `lsof /yedekler` (bağlama noktası verilince bütün dosya sistemine bakar) daha hızlıdır.

---

## 7. umount -l ve tehlikesi

`umount -l` (lazy) "target is busy"de bile hata vermez:

```bash
sudo umount -l /yedekler
findmnt /yedekler          # hiçbir şey göstermez
```

Ama dosya sistemi **ayrılmadı, sadece gizlendi.** Denemede `umount -l`'den sonra iki süreç dosya sistemini kullanmaya devam etti. Yolları kökten itibaren değişti:

```
3 -> /is/arsiv.tmp
/proc/12586/cwd -> /is
```

Dosya sistemi, onu kullanan son süreç kapanınca gerçekten ayrılır. O zamana kadar:

- `findmnt` ve `/proc/mounts` diski bağlı göstermez.
- Diski bağlı sanan araçlar artık öyle düşünmez. Denemede `xfs_repair -n /dev/loop0p1`, dosya sistemi hala kullanımdayken itiraz etmeden çalıştı. Normal bağlı bir XFS'te `xfs_repair` çalışmayı reddeder ([Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md)). `-n` olmadan çalıştırılsaydı, kullanımdaki bir dosya sistemini değiştirip bozabilirdi.

> [!CAUTION]
> `umount -l` kullanıcıları bulup durdurmanın yerini tutmaz. Disk çıkarılacak, onarılacak veya yeniden biçimlendirilecekse önce `fuser -vm` ile kullanıcıları bul ve normal `umount` kullan. `umount -l`, sadece NFS sunucusu cevap vermediği için takılmış bir bağlama gibi başka çaresi olmayan durumlar içindir.

---

## 8. Diskin hızını ölçmek: dd

Bir diskin gerçekte ne kadar hızlı yazdığını ölçmek için `dd` kullanılabilir. Ama bayrak seçimi sonucu tamamen değiştirir. Aynı diske 200 MB yazdık:

```bash
dd if=/dev/zero of=/yedekler/hiz.test bs=1M count=200
dd if=/dev/zero of=/yedekler/hiz.test bs=1M count=200 conv=fdatasync
dd if=/dev/zero of=/yedekler/hiz.test bs=1M count=200 oflag=direct
dd if=/dev/zero of=/yedekler/hiz.test bs=1M count=50  oflag=dsync
```

| Bayrak | Sonuç | Ne ölçüldü? |
|---|---|---|
| (yok) | 3.1 GB/s | **RAM.** Veri önbelleğe yazıldı, diske henüz gitmedi |
| `conv=fdatasync` | 636 MB/s | Hepsini yaz, sonunda bir kez diske gönder. Sıralı yazma hızına en yakın |
| `oflag=direct` | 1.8 GB/s | Önbelleği atla, doğrudan diske yaz |
| `oflag=dsync` | 182 MB/s | **Her blokta** diskin "yazdım" demesini bekle. Veritabanı gibi sık senkron yazan uygulamaların gördüğü hız |

Bayraksız ilk satır en sık yapılan hatadır: "diskim 3 GB/s yazıyor" sonucu yanlıştır.

Okuma hızı için önce önbelleği boşalt, yoksa dosya RAM'den okunur:

```bash
sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
dd if=/yedekler/hiz.test of=/dev/null bs=1M iflag=direct
rm /yedekler/hiz.test
```

> [!WARNING]
> `dd`'de `of=` olarak bir **cihaz** (`of=/dev/sdb`) verirsen, o diskteki her şeyi silersin. Ölçüm için her zaman bağlı dosya sisteminde bir **dosyaya** yaz.

Daha ayrıntılı ölçüm için `fio` aracı vardır (`sudo apt install fio`), ama sınav için `dd` ile hangi bayrağın neyi ölçtüğünü bilmek yeterlidir.

### iotop

`iotop` (Ubuntu'da `iotop-c` paketi), `top` gibi canlı olarak en çok G/Ç yapan süreçleri listeler: `sudo iotop-c -o` (`-o` sadece G/Ç yapanlar). Çalışması için `kernel.task_delayacct` açık olmalıdır, kapalıysa araç uyarır. Deneme ortamında toplam yazma hızını gösterdi ama süreç satırlarını göstermedi. `pidstat -d` her durumda çalıştı.

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `vmstat 1 5` | `b` ve `wa` sütunları: disk bekleyen süreçler |
| `iostat -dxyz 1 3` | Disk bazında kullanım, ilk rapor atlanmış |
| `iostat -dxyz -p sda 1 3` | Bölüm bazında |
| `iostat -dyzh 1 3` | Okunaklı birimlerle özet |
| `pidstat -d -U 1 3` | Süreç bazında disk okuma/yazma |
| `pgrep -ax ad` | Tam adla süreç bul |
| `ps -o pid,ppid,user,stat,etime,cmd -p PID` | Süreç ayrıntısı ve ebeveyni |
| `pwdx PID` | Sürecin çalışma dizini |
| `ls -l /proc/PID/fd` | Sürecin açık dosyaları |
| `df -h dosya` | Dosyanın dosya sistemi |
| `lsof +D /dizin` / `lsof /bağlama` | Dizini veya dosya sistemini kullananlar |
| `fuser -vm /bağlama` | Dosya sistemini kullananlar |
| `umount -l` | Gizle, kullanıcılar bitince ayır (son çare) |
| `dd ... conv=fdatasync` / `oflag=dsync` | Yazma hızı ölçümü |

---

## 10. Sık yapılan hatalar

- **`iostat`'ın ilk raporuna bakmak.** Açılıştan beri ortalamadır. `-y` kullan veya ikinci rapora bak.
- **Kısa ömürlü süreçleri öldürüp kaynağı (döngüyü, cron'u, servisi) bırakmak.** `ps -o ppid` ile ebeveyne bak.
- **`pgrep`'i `-x` olmadan kullanmak.**
- **"target is busy"de kendi kabuğunun o dizinde olduğunu fark etmemek.** `cd /`.
- **`umount -l`'yi normal çözüm sanmak.**
- **`dd` ile bayraksız hız ölçmek.** RAM'i ölçersin.
- **`dd`'de `of=`'u bir cihaz yapmak.**

---

## 11. Alıştırmalar

Hazırlık: Bir dosya sistemi bağla ve arka planda üzerine sürekli yazan bir döngü başlat.

```bash
sudo truncate -s 1G /srv/test.img && sudo mkfs.xfs -q /srv/test.img
sudo mkdir -p /mnt/test && sudo mount /srv/test.img /mnt/test
sudo mkdir /mnt/test/is
sudo bash -c 'cd /mnt/test/is && while :; do dd if=/dev/zero of=veri bs=1M count=300 oflag=direct status=none; done' &
```

**1.** Hangi diskin meşgul olduğunu bul. İlk raporun yanıltmamasına dikkat et.

<details>
<summary>Çözüm</summary>

```bash
iostat -dxyz 1 3
losetup -l          # loop cihazının hangi dosyaya bağlı olduğu
```

`%util` ve `wkB/s` yüksek olan loop cihazı.
</details>

**2.** Diske en çok yazan süreci ve kullanıcısını bul.

<details>
<summary>Çözüm</summary>

```bash
pidstat -d -U 1 3
```

`bash` (döngü) ve kısa ömürlü `dd`'ler. `ps -o pid,ppid,cmd -p <dd PID>` ile `dd`'nin ebeveyninin `bash` olduğunu doğrula.
</details>

**3.** Yazılan dosyanın tam yolunu ve dosya sistemini bul.

<details>
<summary>Çözüm</summary>

```bash
P=$(pgrep -x dd)
pwdx $P
sudo ls -l /proc/$P/fd
df -h /mnt/test/is/veri
```
</details>

**4.** Yazmayı tamamen durdur ve diskin sakinleştiğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo kill <bash PID>
sudo pkill -x dd
iostat -dxyz 1 2        # loop satırı görünmemeli
```
</details>

**5.** Bir terminalde `cd /mnt/test/is` yap. Başka bir terminalden `/mnt/test`'i ayırmayı dene, engelleyeni bul ve doğru şekilde ayır.

<details>
<summary>Çözüm</summary>

```bash
sudo umount /mnt/test          # target is busy
sudo fuser -vm /mnt/test       # cwd (c) olan bash
# ilk terminalde: cd /
sudo umount /mnt/test
```
</details>

---

Önceki konu: [Sanal Dosya Sistemleri](10-sanal-dosya-sistemleri.md) · Bölüm ana sayfası: [Depolama](README.md)
