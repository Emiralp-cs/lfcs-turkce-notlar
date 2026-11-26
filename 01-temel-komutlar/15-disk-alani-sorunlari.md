# Disk Alanı Sorunları

> **Resmi sınav maddesi:** Troubleshoot diskspace issues

Disk dolunca servisler log yazamaz, veritabanları durur, kullanıcılar dosya kaydedemez. Hata mesajı hep aynıdır: `No space left on device`. Ama bu mesajın arkasında birbirinden çok farklı sebepler olabilir: gerçekten dolmuş bir disk, bitmiş inode'lar, silindiği halde yer açmayan bir dosya ya da bir mount noktasının altında saklanan veri. Bu dosya her birini nasıl tanıyacağını ve nasıl çözeceğini anlatıyor.

Senaryo: Uygulama sunucusunda veriler `/srv/veri` diskinde duruyor. Bu diski bilerek doldurduk ve gerçek hayatta en sık karşılaşılan beş durumu oluşturduk.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denenmiştir. Diskin dolduğu durumları güvenle oluşturmak için `/srv/veri`, bir dosyadan oluşturulmuş 100 MB'lık bir disk (loop cihazı) olarak bağlandı. Gerçek bir diskte de komutlar ve çıktılar aynıdır.

## İçindekiler

1. [Genel bakış: df](#1-genel-bakış-df)
2. [Yeri kim kaplıyor: du](#2-yeri-kim-kaplıyor-du)
3. [Büyük dosyaları bulmak: find](#3-büyük-dosyaları-bulmak-find)
4. [Durum 1: Disk gerçekten dolu](#4-durum-1-disk-gerçekten-dolu)
5. [Durum 2: Dosyayı sildim ama yer açılmadı](#5-durum-2-dosyayı-sildim-ama-yer-açılmadı)
6. [Durum 3: Yer var ama dosya oluşturulamıyor (inode)](#6-durum-3-yer-var-ama-dosya-oluşturulamıyor-inode)
7. [Durum 4: Root yazabiliyor, kullanıcılar yazamıyor](#7-durum-4-root-yazabiliyor-kullanıcılar-yazamıyor)
8. [Durum 5: df ile du birbirini tutmuyor (gizli veri)](#8-durum-5-df-ile-du-birbirini-tutmuyor-gizli-veri)
9. [Güvenle yer açmak](#9-güvenle-yer-açmak)
10. [Kalıcı çözümler](#10-kalıcı-çözümler)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Genel bakış: df

`df` (disk free), bağlı dosya sistemlerinin ne kadar dolu olduğunu gösterir. `-h` boyutları okunabilir biçimde yazar:

```bash
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda        252G   13G   30G  30% /
/dev/loop0       96M   94M     0 100% /srv/veri
```

| Sütun | Anlamı |
|---|---|
| `Filesystem` | Hangi disk veya bölüm |
| `Size`, `Used`, `Avail` | Toplam, kullanılan, boş |
| `Use%` | Doluluk |
| `Mounted on` | Hangi dizine bağlı |

Belirli bir dizinin hangi diskte olduğunu ve o diskin durumunu görmek için dizini argüman olarak ver:

```bash
df -h /srv/veri/loglar
```

Çıktıda `tmpfs` ile başlayan satırlar bellekte duran sanal dosya sistemleridir, diskle ilgisi yoktur. Bunları gizlemek için `df -h -x tmpfs`, sadece belirli türleri görmek için `df -h -t ext4`.

> [!TIP]
> Dikkat: `Size` ile `Used + Avail` toplamı tutmayabilir. Yukarıda 96M'lık diskte 94M kullanılmış, ama boş yer 2M değil 0 görünüyor. Aradaki fark root'a ayrılmış yerdir ([7. bölüm](#7-durum-4-root-yazabiliyor-kullanıcılar-yazamıyor)).

---

## 2. Yeri kim kaplıyor: du

`df` hangi diskin dolu olduğunu söyler, **hangi dizinin** doldurduğunu söylemez. Bunun için `du` (disk usage):

```bash
du -sh /srv/veri/* | sort -h
```
```
16K	/srv/veri/lost+found
19M	/srv/veri/uygulama
31M	/srv/veri/loglar
46M	/srv/veri/yedek
```

| Seçenek | Anlamı |
|---|---|
| `-s` | Her argüman için tek toplam (alt dizinleri tek tek yazma) |
| `-h` | Okunabilir boyut |
| `-x` | Başka bir diske geçme (sadece bu dosya sistemi) |
| `--max-depth=1` veya `-d 1` | Sadece bir seviye derine in |

`sort -h`, `46M` ve `16K` gibi okunabilir boyutları doğru sıralar ([Metin İşleme](06-metin-isleme.md)). En büyükler en altta kalır.

Dolu bir diskte dizin dizin aşağı inerek suçluyu bulursun:

```bash
du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -5    # kökte en büyük dizinler
du -xh --max-depth=1 /var | sort -h | tail -5            # /var'da en büyükler
```

`-x` burada önemli: olmazsa `du /` başka disklerdeki dizinleri (ör. `/srv/veri`) de sayar ve kök diskin neden dolduğunu yanlış gösterir. `2>/dev/null` erişim izni olmayan dizinlerin hatalarını gizler.

---

## 3. Büyük dosyaları bulmak: find

```bash
find /srv/veri -xdev -type f -size +20M -exec ls -lh {} \;
```
```
-rw-r--r-- 1 root root 45M Nov 26 08:47 /srv/veri/yedek/eski-yedek.tar
-rw-r--r-- 1 root root 30M Nov 26 08:47 /srv/veri/loglar/uygulama.log
```

Boyuta göre sıralı liste:

```bash
find / -xdev -type f -size +100M -printf '%s %p\n' 2>/dev/null | sort -n | tail
```

`-xdev`, `du -x` gibi başka disklere geçmeyi engeller. Uzun süredir değişmemiş büyük dosyalar genelde silinmeye en uygun adaylardır: `-mtime +30` ekle. `find`'ın bütün seçenekleri: [find](05-find.md).

---

## 4. Durum 1: Disk gerçekten dolu

Diske 50 MB yazmaya çalıştık, sadece 8 MB yazılabildi:

```
dd: error writing '/srv/veri/uygulama/yeni.db': No space left on device
```
```bash
df -h /srv/veri
```
```
/dev/loop0       96M   94M     0 100% /srv/veri
```

`du` ve `find` ile yukarıdaki gibi en büyük dizin ve dosyaları buldun. Sonra:

- Gerçekten gereksiz olanları sil (eski yedekler, eski loglar, geçici dosyalar).
- Gerekli ama büyük olanları başka bir diske taşı veya sıkıştır ([Arşivleme ve Sıkıştırma](08-arsivleme-ve-sikistirma.md)).
- Yer açtıktan sonra `df -h` ile kontrol et.

Silmeden önce [9. bölümü](#9-güvenle-yer-açmak) oku: çalışan bir programın log dosyasını `rm` ile silmek yer açmayabilir.

---

## 5. Durum 2: Dosyayı sildim ama yer açılmadı

En kafa karıştıran durum. Bir uygulama 30 MB'lık log dosyasını açık tutarken dosyayı sildik:

```bash
rm /srv/veri/loglar/uygulama.log
df -h /srv/veri
du -sh /srv/veri
```
```
/dev/loop0       96M   86M  3.5M  97% /srv/veri
56M	/srv/veri
```

`du` 56M diyor, `df` 86M. Dosya dizinden silindi ama diskten silinmedi. Linux'ta bir dosyanın verisi, onu açık tutan son program kapatana kadar diskte kalır ([Hard ve Soft Link](03-hard-ve-soft-link.md)). `rm` sadece adını kaldırdı.

### Bulmak: lsof

```bash
sudo lsof +L1 /srv/veri
```
```
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NLINK NODE NAME
tail    10249 root    3w   REG    7,0 31457280     0   16 /srv/veri/loglar/uygulama.log (deleted)
```

`+L1`, bağlantı sayısı 1'den az, yani silinmiş ama açık dosyaları listeler. `NLINK 0` ve sondaki `(deleted)` bunu gösterir. Sadece mount noktası verilirse o diskteki açık dosyaları, verilmezse bütün sistemdekileri listeler. `lsof` yoksa `sudo apt install lsof`, veya:

```bash
sudo ls -l /proc/*/fd 2>/dev/null | grep deleted
```

### Çözmek

**1. Programı yeniden başlat.** Program dosyayı kapatınca yer açılır. Servisse `sudo systemctl restart servis`. En temiz yol budur.

**2. Yeniden başlatamıyorsan dosyayı boşalt.** Program çalışmaya devam ederken, açık dosyanın içeriği `/proc` üzerinden sıfırlanabilir. `lsof` çıktısındaki `PID` ve `FD` (sondaki harf hariç) kullanılır:

```bash
sudo truncate -s 0 /proc/10249/fd/3
df -h /srv/veri
```
```
/dev/loop0       96M   56M   34M  63% /srv/veri
```

30 MB geri geldi, program durmadı.

---

## 6. Durum 3: Yer var ama dosya oluşturulamıyor (inode)

Her dosya, boyutu ne olursa olsun, bir **inode** kullanır. Dosya sistemi oluşturulurken inode sayısı sabitlenir. Çok sayıda küçük dosya (oturum dosyaları, önbellek, mail kuyruğu) disk dolmadan inode'ları bitirebilir.

Diske küçük dosyalar yazan bir döngü 1983. dosyada durdu:

```
sess_1983: No space left on device
touch: cannot touch '/srv/veri/yeni.txt': No space left on device
```

Ama:

```bash
df -h /srv/veri
```
```
/dev/loop0       96M   56M   34M  63% /srv/veri
```

Disk %63 dolu. Asıl sorun:

```bash
df -i /srv/veri
```
```
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop0       2000  2000     0  100% /srv/veri
```

`No space left on device` görüp `df -h`'ta yer varsa **her zaman `df -i`'ye bak.**

### Hangi dizinde bu kadar dosya var?

`du` boyutu sayar, dosya sayısını saymaz. Her alt dizindeki dosya sayısı için:

```bash
for d in /srv/veri/*/; do echo "$(find "$d" -xdev | wc -l) $d"; done | sort -n
```
```
3 /srv/veri/uygulama/
1983 /srv/veri/oturumlar/
```

Gereksiz dosyalar silinince inode'lar geri gelir. Dosyalar tek tek silinemeyecek kadar çoksa `find /srv/veri/oturumlar -type f -mtime +7 -delete`.

> [!NOTE]
> ext4'te inode sayısı sonradan artırılamaz, dosya sistemini daha fazla inode'la (`mkfs.ext4 -N` veya `-i`) yeniden oluşturmak gerekir. XFS inode'ları ihtiyaca göre ayırır, bu sorunu çok daha seyrek yaşar ([Dosya Sistemleri](../04-depolama/02-dosya-sistemleri.md)).

---

## 7. Durum 4: Root yazabiliyor, kullanıcılar yazamıyor

ext4, varsayılan olarak diskin **%5'ini** root'a ayırır. Disk normal kullanıcılar için dolduğunda bile sistem servisleri ve yönetici çalışmaya devam edebilsin diye.

Normal bir kullanıcı diski doldurdu:

```
dd: error writing '/srv/veri/kullanici.dat': No space left on device
```
```
/dev/loop0       96M   89M  408K 100% /srv/veri
```

Aynı anda root aynı diske 3 MB daha yazabildi. Ayrılan alanı görmek için:

```bash
sudo tune2fs -l /dev/loop0 | grep -E 'Block count|Reserved block count'
```
```
Block count:              25600
Reserved block count:     1280
```

1280 / 25600 = %5. Büyük veri disklerinde (root'un yazması gerekmeyen) bu oran düşürülebilir:

```bash
sudo tune2fs -m 1 /dev/loop0
```
```
Setting reserved blocks percentage to 1% (256 blocks)
```

`df`'teki boş yer anında arttı. Sistemin kök diskinde (`/`) bu oranı düşürme: o alan, disk dolduğunda yöneticinin giriş yapıp sorunu çözebilmesi içindir.

---

## 8. Durum 5: df ile du birbirini tutmuyor (gizli veri)

Bir dizine dosya yazıldıktan **sonra** o dizine bir disk bağlanırsa, eski dosyalar görünmez olur ama yer kaplamaya devam eder. Bu genelde bir disk bir süre bağlı olmadığında, uygulama aynı yola yazmaya devam ettiğinde olur.

Denemede `/srv/veri` bağlı değilken oraya 20 MB'lık bir dosya yazdık, sonra diski tekrar bağladık. `ls /srv/veri` dosyayı göstermedi, `du` da saymadı. Ama 20 MB kök diskte yer kaplıyordu.

Görmek için kök dosya sistemini başka bir yere **bind mount** et. Bu, diğer diskleri olmadan kök diskin kendisini gösterir:

```bash
sudo mkdir -p /mnt/kok
sudo mount --bind / /mnt/kok
ls -lh /mnt/kok/srv/veri/
```
```
total 20M
-rw-r--r-- 1 root root 20M Nov 26 08:48 gizli.dat
```
```bash
sudo umount /mnt/kok
```

Dosyayı silmek için ya buradan sil (`/mnt/kok/srv/veri/gizli.dat`) ya da diski ayırıp sil, sonra tekrar bağla.

Kök diskte `df` çok, `du -xh /` az gösteriyorsa ve silinmiş açık dosya da yoksa ([5. bölüm](#5-durum-2-dosyayı-sildim-ama-yer-açılmadı)) bu ihtimale bak.

---

## 9. Güvenle yer açmak

| Ne? | Nasıl? | Not |
|---|---|---|
| Çalışan bir programın log dosyası | `sudo truncate -s 0 dosya` | `rm` yer açmaz, [5. bölüm](#5-durum-2-dosyayı-sildim-ama-yer-açılmadı) |
| Systemd journal | `sudo journalctl --vacuum-size=200M` veya `--vacuum-time=7d` | `/var/log/journal` altındaki dosyaları elle silme |
| apt'nin indirdiği paketler | `sudo apt clean` | `/var/cache/apt/archives` |
| Artık gerekmeyen bağımlılıklar | `sudo apt autoremove` | Eski çekirdekler de gider |
| Eski, döndürülmüş loglar | `/var/log` altında `.gz` ve `.1` uzantılı dosyalar | Aktif log dosyasına dokunma |
| Eski yedekler, geçici dosyalar | `find ... -mtime +30 -delete` | Önce `-delete` olmadan çalıştırıp listeye bak |

Journal'ın ne kadar yer kapladığını görmek için:

```bash
journalctl --disk-usage
```
```
Archived and active journals take up 8.0M in the file system.
```

> [!CAUTION]
> Yer açmak için dosya silerken acele etme. `/var/lib` altındaki veritabanı dosyaları, `/boot` altındaki çekirdekler veya bir uygulamanın veri dizini silinirse sistem açılmayabilir ya da veri kaybolur. Neyi sildiğinden emin değilsen önce başka bir yere taşı veya sıkıştır.

---

## 10. Kalıcı çözümler

Yer açmak sorunu bugün çözer. Tekrar olmaması için:

- **Log döndürme:** `logrotate`, logları belirli aralıklarla sıkıştırıp eskilerini siler. Kuralları `/etc/logrotate.d/` altındadır. Kendi uygulamanın logları da buraya eklenmeli.
- **Journal sınırı:** `/etc/systemd/journald.conf` içinde `SystemMaxUse=500M`, sonra `sudo systemctl restart systemd-journald`.
- **Ayrı diskler:** `/var/log` veya uygulama verisi ayrı bir diskte olursa, orası dolduğunda kök disk etkilenmez.
- **Büyütmek:** Disk LVM üzerindeyse, sistemi kapatmadan büyütülebilir ([LVM](../04-depolama/06-lvm.md)).
- **İzleme:** `df -h` çıktısını düzenli kontrol eden bir script veya izleme aracı, disk dolmadan uyarır.

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `df -h` | Disklerin doluluğu |
| `df -h /dizin` | Bu dizinin bulunduğu disk |
| `df -i` | Inode doluluğu |
| `df -h -x tmpfs` | Sanal dosya sistemlerini gizle |
| `du -sh dizin/* \| sort -h` | Alt dizinlerin boyutu, sıralı |
| `du -xh --max-depth=1 / \| sort -h` | Kök diskte en büyük dizinler |
| `find / -xdev -type f -size +100M` | 100 MB'tan büyük dosyalar |
| `lsof +L1` | Silinmiş ama açık dosyalar |
| `truncate -s 0 dosya` | Dosyayı silmeden boşalt |
| `truncate -s 0 /proc/PID/fd/N` | Silinmiş açık dosyayı boşalt |
| `tune2fs -l /dev/X` | ext4 bilgileri, ayrılmış blok sayısı |
| `tune2fs -m 1 /dev/X` | Root'a ayrılan alanı %1 yap |
| `mount --bind / /mnt/kok` | Mount altında gizli dosyaları gör |
| `journalctl --disk-usage` | Journal'ın boyutu |
| `journalctl --vacuum-size=200M` | Journal'ı küçült |
| `apt clean`, `apt autoremove` | Paket önbelleğini ve gereksiz paketleri temizle |

---

## 12. Sık yapılan hatalar

- **`No space left on device` görüp sadece `df -h`'a bakmak.** `df -i` de kontrol edilmeli.
- **Açık bir log dosyasını `rm` ile silmek.** Yer açılmaz, dosya artık görünmez de olur. `truncate` kullan.
- **`du /` çalıştırırken `-x` kullanmamak.** Diğer diskler de sayılır.
- **`df` ile `du` farkını görmezden gelmek.** Fark, silinmiş açık dosyaları veya mount altındaki gizli veriyi gösterir.
- **`/var/log/journal` altındaki dosyaları elle silmek.** `journalctl --vacuum-*` kullan.
- **Kök diskte root'a ayrılan alanı sıfırlamak.**
- **Ne olduğunu bilmediğin büyük dosyaları silmek.**

---

## 13. Alıştırmalar

**1.** Sistemdeki bütün diskleri, sanal dosya sistemleri hariç, doluluklarıyla listele. Kök diskin inode doluluğuna da bak.

<details>
<summary>Çözüm</summary>

```bash
df -h -x tmpfs
df -i /
```
</details>

**2.** `/var` altında en çok yer kaplayan 5 dizini bul, sadece kök diski say.

<details>
<summary>Çözüm</summary>

```bash
sudo du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail -5
```
</details>

**3.** Kök diskte 50 MB'tan büyük bütün dosyaları boyutlarına göre sıralı listele.

<details>
<summary>Çözüm</summary>

```bash
sudo find / -xdev -type f -size +50M -printf '%s %p\n' 2>/dev/null | sort -n
```
</details>

**4.** Silinmiş ama açık dosya durumunu kendin oluştur ve çöz:

```bash
dd if=/dev/zero of=/tmp/buyuk.log bs=1M count=100
tail -f /tmp/buyuk.log > /dev/null &
rm /tmp/buyuk.log
```

Dosyayı tutan süreci bul, yer açılmadığını gör, süreci durdurmadan yeri geri kazan, sonra süreci durdur.

<details>
<summary>Çözüm</summary>

```bash
df -h /tmp
sudo lsof +L1 | grep buyuk.log      # PID ve FD (ör. 3r) sütunlarına bak
sudo truncate -s 0 /proc/PID/fd/3   # PID ve 3 yerine kendi değerlerini yaz
df -h /tmp
kill %1
```
</details>

**5.** Journal'ın ne kadar yer kapladığına bak ve 3 günden eski kayıtları temizle.

<details>
<summary>Çözüm</summary>

```bash
journalctl --disk-usage
sudo journalctl --vacuum-time=3d
journalctl --disk-usage
```
</details>

---

Önceki konu: [Performans İzleme](14-performans-izleme.md) · Sonraki konu: [Uygulama ve Servis Kısıtlamaları](16-uygulama-ve-servis-kisitlamalari.md)
