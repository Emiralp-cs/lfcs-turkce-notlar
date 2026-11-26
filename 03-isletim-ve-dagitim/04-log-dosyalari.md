# Log Dosyaları

> **İlgili resmi sınav maddeleri:** Diagnose, identify, manage, and troubleshoot processes and services · Monitor and troubleshoot system performance and services

Bir şeyin neden çalışmadığını bulmanın ilk yolu loglara bakmaktır. Ubuntu'da loglar iki yerde tutulur: **systemd journal** (`journalctl` ile okunur) ve **rsyslog**'un `/var/log` altına yazdığı metin dosyaları. Bu dosya ikisini de okumayı, filtrelemeyi ve canlı izlemeyi anlatıyor. Sınavda sık çıkan "şu servisin şu öncelikteki loglarını bul" türü sorular buraya girer.

Senaryo: Sunucuya dışarıdan SSH ile giriş denemeleri yapılıyor ve gece çalışan yedekleme script'i bazen hata veriyor. İkisinin izini loglarda süreceğiz.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve systemd 255 ile denenmiştir. Örnek log satırları, `logger` komutuyla bilerek oluşturuldu.

## İçindekiler

1. [Loglar nereye yazılır?](#1-loglar-nereye-yazılır)
2. [/var/log altındaki dosyalar](#2-varlog-altındaki-dosyalar)
3. [Logları okumak için yetki](#3-logları-okumak-için-yetki)
4. [Metin loglarında arama ve canlı izleme](#4-metin-loglarında-arama-ve-canlı-izleme)
5. [journalctl: temel kullanım](#5-journalctl-temel-kullanım)
6. [Önceliğe göre filtrelemek: -p](#6-önceliğe-göre-filtrelemek--p)
7. [Mesaj içinde aramak: -g](#7-mesaj-içinde-aramak--g)
8. [Zamana ve açılışa göre filtrelemek](#8-zamana-ve-açılışa-göre-filtrelemek)
9. [Alanlara göre filtrelemek](#9-alanlara-göre-filtrelemek)
10. [Kendi log kaydını yazmak: logger ve systemd-cat](#10-kendi-log-kaydını-yazmak-logger-ve-systemd-cat)
11. [Giriş geçmişi: last, lastb, lastlog](#11-giriş-geçmişi-last-lastb-lastlog)
12. [Journal'ın kalıcılığı ve log döndürme](#12-journalın-kalıcılığı-ve-log-döndürme)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Loglar nereye yazılır?

```
programlar, çekirdek, servisler
            │
            ▼
   systemd-journald ──────────► journal (/var/log/journal/)  →  journalctl
            │
            ▼ (rsyslog kuruluysa kopyasını alır)
         rsyslog ─────────────► /var/log/syslog, auth.log, kern.log ...  →  less, grep, tail
```

Servislerin ekrana yazdığı her şey, `logger` ile gönderilen mesajlar ve çekirdek mesajları önce **journald**'a gider. **rsyslog** kuruluysa bunların kopyasını alıp türüne göre `/var/log` altındaki metin dosyalarına dağıtır.

> [!IMPORTANT]
> Ubuntu Server'da rsyslog normalde kuruludur, ama minimal kurulumlarda ve container'larda olmayabilir. Bu notun denendiği ortamda başta `/var/log/syslog` ve `/var/log/auth.log` yoktu, `sudo apt install rsyslog` sonrasında oluştu. **journalctl her systemd sisteminde çalışır**, `/var/log/syslog` yoksa onu kullan.

---

## 2. /var/log altındaki dosyalar

| Dosya | Ne kaydeder? |
|---|---|
| `/var/log/syslog` | Genel sistem mesajları, servisler, cron |
| `/var/log/auth.log` | Kimlik doğrulama: SSH girişleri, `sudo`, `su`, parola değişiklikleri |
| `/var/log/kern.log` | Çekirdek mesajları |
| `/var/log/dpkg.log` | Paket kurma/kaldırma |
| `/var/log/apt/history.log` | `apt` ile yapılan işlemler |
| `/var/log/nginx/`, `/var/log/postgresql/` | Uygulamaların kendi logları |
| `/var/log/wtmp`, `btmp`, `lastlog` | Giriş kayıtları, ikili dosya ([11. bölüm](#11-giriş-geçmişi-last-lastb-lastlog)) |
| `/var/log/journal/` | systemd journal, ikili dosya |

Döndürülmüş eski loglar aynı dizinde durur: `syslog.1` bir önceki, `syslog.2.gz` daha eskisi ve sıkıştırılmış. Sıkıştırılmış logları açmadan okumak için `zcat`, `zgrep`, `zless`.

Bir uygulamanın logunun nerede olduğunu bilmiyorsan önce `journalctl -u servis`'e, sonra `/var/log/` altında uygulamanın adını taşıyan bir dosya veya dizine, en son da uygulamanın yapılandırma dosyasında `log` geçen satırlara bak.

---

## 3. Logları okumak için yetki

```bash
ls -l /var/log/syslog /var/log/auth.log
```
```
-rw-r----- 1 syslog adm 48858 Nov 26 08:59 /var/log/syslog
-rw-r----- 1 syslog adm   112 Nov 26 08:59 /var/log/auth.log
```

Dosyalar `adm` grubuna aittir. Normal bir kullanıcıyla denedik:

```
tail: cannot open '/var/log/syslog' for reading: Permission denied
```

Kullanıcıyı `adm` grubuna ekleyince (`sudo usermod -aG adm mert`, yeniden giriş gerekir) hem `/var/log/syslog`'u hem de bütün journal'ı `sudo` olmadan okuyabildi. `journalctl` de grupta olmayan kullanıcıya bunu söylüyor:

```
Hint: You are currently not seeing messages from other users and the system.
      Users in groups 'adm', 'systemd-journal' can see all messages.
```

Yani logları okumak için `sudo` veya `adm` grubu gerekir. Sınavda "Permission denied" alırsan komutun başına `sudo` ekle.

---

## 4. Metin loglarında arama ve canlı izleme

```bash
sudo less /var/log/auth.log                    # oku (/ ile ara, G ile sona git, q ile çık)
sudo grep 'Failed password' /var/log/auth.log  # başarısız SSH girişleri
sudo grep -r 'ssh' /var/log/                   # hangi dosyada olduğunu bilmiyorsan
sudo zgrep 'Failed password' /var/log/auth.log.2.gz
```
```
2026-11-26T08:59:43.113158+00:00 vm sshd: Failed password for invalid user test from 203.0.113.9 port 4242 ssh2
```

`grep` seçenekleri ve düzenli ifadeler: [grep ve Düzenli İfadeler](../01-temel-komutlar/07-grep-ve-regex.md). En çok deneme yapan IP'leri saymak için o dosyadaki `grep -oE ... | sort | uniq -c` kalıbı kullanılabilir.

### Canlı izlemek: tail -f ve tail -F

```bash
sudo tail -f /var/log/syslog                      # yeni satırları anında göster
sudo tail -F /var/log/auth.log | grep sudo        # filtreleyerek
```

Ctrl+C ile çıkılır. `-f` ile `-F` farkı: log döndürüldüğünde (`auth.log` → `auth.log.1`) `-f` eski dosyayı izlemeye devam eder ve yeni satırları göremez. `-F` dosya adını takip eder, yeni oluşturulan `auth.log`'a geçer. Uzun süre izleyeceksen `-F`.

---

## 5. journalctl: temel kullanım

```bash
journalctl                 # bütün journal, en eskiden başlar (less içinde açılır)
journalctl -e              # sona atla
journalctl -n 20           # son 20 satır
journalctl -f              # canlı izle
journalctl -r              # en yeniden en eskiye
journalctl --no-pager      # less açmadan doğrudan yaz (script ve grep için)
journalctl -x              # bazı mesajlara açıklama ekle
```

### Servise göre: -u

```bash
journalctl -u ssh                    # ssh servisinin logları (.service yazılmasa da olur)
journalctl -u ssh -f                 # canlı izle
journalctl -u ssh --since today      # bugünkü
journalctl -u ssh -u nginx           # iki servis birlikte
```

Servis sorun giderme akışında `journalctl -u`'nun yeri: [Servis Oluşturma ve Sorun Giderme](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md#10-sorun-giderme-adımları).

### Programa göre

Bir programın tam yolunu vererek onun loglarını alabilirsin:

```bash
journalctl /usr/bin/sudo
```
```
Nov 26 08:59:44 vm sudo[2001]: pam_unix(sudo:session): session opened for user nabizci(uid=996) by (uid=0)
Nov 26 08:59:44 vm sudo[2001]: pam_unix(sudo:session): session closed for user nabizci
```

Programın yolunu bilmiyorsan: `which sudo`.

### Çıktı biçimi: -o

| Seçenek | Çıktı |
|---|---|
| `-o short` (varsayılan) | Tarih, makine, program, mesaj |
| `-o short-iso` | ISO tarih biçimiyle |
| `-o cat` | Sadece mesaj |
| `-o verbose` | Bütün alanlar |
| `-o json-pretty` | JSON |

---

## 6. Önceliğe göre filtrelemek: -p

Her log kaydının bir **önceliği** (priority) vardır:

| No | Ad | Anlamı |
|---|---|---|
| 0 | `emerg` | Sistem kullanılamaz |
| 1 | `alert` | Hemen müdahale gerekli |
| 2 | `crit` | Kritik durum |
| 3 | `err` | Hata |
| 4 | `warning` | Uyarı |
| 5 | `notice` | Normal ama dikkat çekici |
| 6 | `info` | Bilgi |
| 7 | `debug` | Hata ayıklama |

```bash
journalctl -p err           # err ve DAHA ÖNEMLİ olanlar (0-3)
journalctl -p warning       # 0-4
journalctl -p 3             # numarayla da yazılabilir
```

> [!IMPORTANT]
> `-p info` sadece info seviyesini değil, **info ve daha önemli** bütün seviyeleri (emerg'den info'ya kadar) gösterir. Sadece tek bir seviye istiyorsan aralık ver: `-p info..info`.

Üç farklı öncelikle log yazdık (`err`, `info` ve `logger`'ın varsayılanı `notice`):

| Komut | Gösterdiği |
|---|---|
| `journalctl -t yedekleme -p err` | Sadece err olanlar |
| `journalctl -t yedekleme -p info` | **Hepsi** (err, notice ve info) |
| `journalctl -t yedekleme -p info..info` | Sadece info olan tek kayıt |

Bir sınav sorusu "info öncelikli loglar" diyorsa, sorunun ne istediğine dikkat et: "info ve üstü" için `-p info`, "sadece info" için `-p info..info`.

---

## 7. Mesaj içinde aramak: -g

`-g` (grep), mesaj metninde düzenli ifadeyle arar. `journalctl | grep` yazmaya gerek kalmaz ve diğer filtrelerle birleşir:

```bash
journalctl -g 'failed'
journalctl -u ssh -g 'Failed password'
journalctl -g 'fail|deny'          # fail VEYA deny
journalctl -g 'error$'             # error ile BİTEN mesajlar
journalctl -p info -g '^c'         # info ve üstü, VE c ile başlayan
```

### Büyük-küçük harf: dikkat

`-g`, aranan ifade **tamamen küçük harfliyse büyük-küçük harfe bakmaz** (smart case). İçinde büyük harf varsa duyarlı olur. Üç mesajla denedik: `Bootstrap hatasi`, `baslangic tamamlandi`, `Basarili bitti`.

| Komut | Bulduğu |
|---|---|
| `-g '^b'` | Üçü de (küçük harf → harfe duyarsız) |
| `-g '^B'` | `Bootstrap...`, `Basarili...` (büyük harf → duyarlı) |
| `--case-sensitive=true -g '^b'` | Sadece `baslangic...` |

Yani "küçük c harfiyle başlayan" mesajlar isteniyorsa `-g '^c'` tek başına **büyük C ile başlayanları da getirir**. Kesin sonuç için `--case-sensitive=true` ekle.

Düzenli ifade sözdizimi: [grep ve Düzenli İfadeler](../01-temel-komutlar/07-grep-ve-regex.md). `-g` genişletilmiş (ERE benzeri) sözdizimi kullanır, `|` için ters bölü gerekmez.

---

## 8. Zamana ve açılışa göre filtrelemek

```bash
journalctl --since today
journalctl --since yesterday --until today
journalctl -S "10 min ago"
journalctl -S 01:00 -U 02:00                      # bugün 01:00–02:00
journalctl -S "2026-11-25 23:00" -U "2026-11-26 01:00"
```

`-S` `--since`'in, `-U` `--until`'in kısaltmasıdır. Sadece saat verilirse bugünün o saati anlaşılır.

### Açılışa göre: -b

```bash
journalctl --list-boots      # kayıtlı açılışlar
journalctl -b                # bu açılıştan beri (-b 0 ile aynı)
journalctl -b -1             # bir önceki açılış
journalctl -b -1 -p err      # önceki açılışın hataları (neden çöktü?)
```

Önceki açılışları görebilmek için journal'ın diske kalıcı yazılması gerekir ([12. bölüm](#12-journalın-kalıcılığı-ve-log-döndürme)).

Çekirdek mesajları için `journalctl -k` (`dmesg`'in journal karşılığı).

---

## 9. Alanlara göre filtrelemek

Journal'daki her kayıt, mesajın yanında birçok **alan** taşır. `-o verbose` veya `-o json-pretty` ile görülebilir:

```
	"SYSLOG_IDENTIFIER" : "yedekleme",
	"MESSAGE" : "Basarili bitti",
	"PRIORITY" : "5"
```

Bu alanlara göre `ALAN=değer` biçiminde filtrelenir:

```bash
journalctl _SYSTEMD_UNIT=ssh.service              # -u ssh ile aynı
journalctl _SYSTEMD_UNIT=ssh.service _PID=1457    # ikisi birden (VE)
journalctl _COMM=sudo                             # programın adına göre
journalctl _UID=1000                              # kullanıcıya göre
journalctl -t yedekleme                           # SYSLOG_IDENTIFIER'a göre
journalctl -F _SYSTEMD_UNIT                       # bu alanın aldığı bütün değerler
```

Farklı alanlar **VE** ile birleşir. Aynı alan iki kez verilirse **VEYA** olur: `journalctl _SYSTEMD_UNIT=ssh.service _SYSTEMD_UNIT=cron.service`. Bütün alanların açıklaması: `man systemd.journal-fields`.

---

## 10. Kendi log kaydını yazmak: logger ve systemd-cat

Script'lerinin journal'a ve syslog'a yazması için:

```bash
logger -t yedekleme "baslangic tamamlandi"                   # öncelik: notice
logger -t yedekleme -p user.err "Bootstrap hatasi: disk dolu"
echo "yedek basladi" | systemd-cat -t yedekleme -p info
```

`-t` etiket (`SYSLOG_IDENTIFIER`), `-p` öncelik. Yazılan kayıtlar hem journal'da hem (rsyslog varsa) `/var/log/syslog`'da görünür:

```bash
journalctl -t yedekleme
grep yedekleme /var/log/syslog
```
```
2026-11-26T08:59:28.789096+00:00 vm yedekleme: Bootstrap hatasi: disk dolu
2026-11-26T08:59:28.802727+00:00 vm yedekleme: baslangic tamamlandi
```

`logger -p auth.info` ile yazılan bir kayıt `auth.log`'a gitti: rsyslog, kaydı türüne (facility) göre dosyalara dağıtır. Kuralları `/etc/rsyslog.d/50-default.conf` dosyasındadır.

---

## 11. Giriş geçmişi: last, lastb, lastlog

| Komut | Ne gösterir? | Kaynak dosya |
|---|---|---|
| `last` | Başarılı girişler ve yeniden başlatmalar | `/var/log/wtmp` |
| `sudo lastb` | Başarısız giriş denemeleri | `/var/log/btmp` |
| `lastlog` | Her kullanıcının en son girişi | `/var/log/lastlog` |
| `who`, `w` | Şu an giriş yapmış olanlar | |

```bash
last -n 5
last mert              # bir kullanıcının girişleri
last reboot            # yeniden başlatma geçmişi
lastlog -u root
```
```
reboot   system boot  6.18.44-fc-v880  Thu Nov 26 08:21   still running
```

Bu dosyalar ikilidir, `cat` veya `grep` ile okunamaz.

---

## 12. Journal'ın kalıcılığı ve log döndürme

### Kalıcı journal

`/etc/systemd/journald.conf` içindeki `Storage=` ayarı:

| Değer | Davranış |
|---|---|
| `auto` (varsayılan) | `/var/log/journal/` dizini **varsa** diske yazar, yoksa sadece bellekte tutar (yeniden başlatınca silinir) |
| `persistent` | Her zaman diske yazar, dizini gerekirse oluşturur |
| `volatile` | Sadece bellekte |

Ubuntu'da `/var/log/journal/` normalde vardır, yani journal kalıcıdır. `journalctl -b -1` "No entries" veriyorsa veya `--list-boots` tek satır gösteriyorsa kalıcı değildir:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

Boyut sınırı (`SystemMaxUse=`) ve eski kayıtları temizleme (`journalctl --vacuum-size`): [Disk Alanı Sorunları](../01-temel-komutlar/15-disk-alani-sorunlari.md#9-güvenle-yer-açmak).

### logrotate

`/var/log` altındaki metin dosyalarını **logrotate** döndürür: belirli aralıklarla dosyayı yeniden adlandırır, eskileri sıkıştırır, en eskileri siler. Genel ayarlar `/etc/logrotate.conf`, uygulamaya özel kurallar `/etc/logrotate.d/` altında:

```
/var/log/syslog
/var/log/auth.log
{
	rotate 4          # 4 eski kopya tut
	weekly            # haftada bir döndür
	missingok         # dosya yoksa hata verme
	notifempty        # boşsa döndürme
	compress          # eskileri gzip'le
	delaycompress     # en yeni eski kopyayı (.1) sıkıştırma
	postrotate
		/usr/lib/rsyslog/rsyslog-rotate
	endscript
}
```

`sudo logrotate -d /etc/logrotate.d/rsyslog` hiçbir şeyi değiştirmeden ne yapacağını gösterir. `-f` ile hemen döndürmeye zorlanır.

---

## 13. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `less`, `grep`, `zgrep` dosya | Metin loglarını oku, ara |
| `tail -F dosya` | Canlı izle, döndürmeye dayanıklı |
| `journalctl -e`, `-n 20`, `-f`, `-r` | Sona git, son 20, canlı, ters sıra |
| `journalctl -u servis` | Servisin logları |
| `journalctl /usr/bin/program` | Programın logları |
| `journalctl -p err` | err ve daha önemliler |
| `journalctl -p info..info` | Sadece info |
| `journalctl -g 'desen'` | Mesajda ara (küçük harfse harfe duyarsız) |
| `journalctl --case-sensitive=true -g` | Harfe duyarlı ara |
| `journalctl -S 01:00 -U 02:00` | Zaman aralığı |
| `journalctl -b`, `-b -1` | Bu açılış, önceki açılış |
| `journalctl -k` | Çekirdek mesajları |
| `journalctl -t etiket` | Etikete göre |
| `journalctl _COMM=sudo`, `_PID=`, `_UID=` | Alana göre |
| `journalctl -F ALAN` | Alanın değerleri |
| `journalctl -o cat`, `-o verbose` | Çıktı biçimi |
| `logger -t etiket -p user.err "mesaj"` | Log yaz |
| `last`, `lastb`, `lastlog` | Giriş geçmişi |
| `logrotate -d dosya` | Döndürme kuralını dene |

---

## 14. Sık yapılan hatalar

- **`-p info`'nun sadece info gösterdiğini sanmak.** Info ve üstünü gösterir.
- **`-g`'nin her zaman harfe duyarlı olduğunu sanmak.** Küçük harfli desende değildir.
- **`/var/log/syslog`'un her sistemde olduğunu varsaymak.** rsyslog kurulu değilse yoktur, `journalctl` her zaman var.
- **`sudo` veya `adm` grubu olmadan logların eksik görünmesini fark etmemek.** `journalctl` bir ipucu satırı basar.
- **Uzun izlemede `tail -f` kullanmak.** Log döndürülünce yeni satırları kaçırır.
- **`journalctl | grep` ile filtrelemek.** Çalışır ama `-g`, `-u`, `-p` hem daha hızlı hem birleştirilebilir.
- **Kalıcı olmayan journal'da önceki açılışın loglarını aramak.**
- **`wtmp`/`btmp`'yi `cat` ile okumaya çalışmak.**

---

## 15. Alıştırmalar

**1.** `logger` ile `deneme` etiketli üç kayıt yaz: biri `err`, biri `warning`, biri `info` öncelikli. Sonra sadece warning ve daha önemli olanları, ardından sadece info olanı göster.

<details>
<summary>Çözüm</summary>

```bash
logger -t deneme -p user.err "veritabani baglantisi koptu"
logger -t deneme -p user.warning "disk %85 dolu"
logger -t deneme -p user.info "yedek tamamlandi"

journalctl -t deneme -p warning
journalctl -t deneme -p info..info
```
</details>

**2.** `deneme` etiketli kayıtlardan, mesajı **küçük** "d" harfiyle başlayanları bul. Büyük "D" ile başlayan bir kayıt da yaz ve sonuca karışmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
logger -t deneme "Disk kontrolu basladi"
journalctl -t deneme -g '^d'                          # iki kayıt: disk... ve Disk...
journalctl -t deneme --case-sensitive=true -g '^d'    # sadece disk...
```
</details>

**3.** Bu açılıştan beri `err` ve daha önemli seviyedeki bütün logları, sayfalayıcı açmadan, sadece mesaj metniyle `/tmp/hatalar.txt` dosyasına kaydet.

<details>
<summary>Çözüm</summary>

```bash
sudo journalctl -b -p err --no-pager -o cat > /tmp/hatalar.txt
wc -l /tmp/hatalar.txt
```
</details>

**4.** `sudo` programının son 1 saatteki loglarını göster. Programın yolunu bilmediğini varsay.

<details>
<summary>Çözüm</summary>

```bash
which sudo
sudo journalctl /usr/bin/sudo --since "1 hour ago"
```

`journalctl _COMM=sudo --since "1 hour ago"` de olur.
</details>

**5.** `/var/log` altında "Failed password" geçen bütün kayıtları dosya adlarıyla bul. Journal'da da aynısını `ssh` servisi için ara.

<details>
<summary>Çözüm</summary>

```bash
sudo grep -r "Failed password" /var/log/ 2>/dev/null
sudo journalctl -u ssh -g "Failed password"
```

İkinci komut harfe duyarlıdır, çünkü desende büyük harf var.
</details>

---

Önceki konu: [Süreçler](03-surecler.md) · Sonraki konu: [Zamanlanmış Görevler](05-zamanlanmis-gorevler.md)
