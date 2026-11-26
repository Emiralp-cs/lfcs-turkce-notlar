# Zamanlanmış Görevler

> **Resmi sınav maddesi:** Manage or schedule jobs for executing commands

Yedek almak, log temizlemek, rapor üretmek gibi işler belirli zamanlarda kendiliğinden çalışmalı. Linux'ta bunun için dört araç var:

| Araç | Ne için? | Örnek |
|---|---|---|
| **cron** | Düzenli tekrarlanan işler, dakika hassasiyetinde | Her gece 02:30'da yedek |
| **anacron** | Günlük/haftalık işler, sistem o saatte kapalıysa açılınca telafi eder | Dizüstünde günlük temizlik |
| **at** | Bir kez çalışacak iş | Bu akşam 23:00'te yeniden başlat |
| **systemd timer** | cron'un modern karşılığı, loglu ve servis gibi yönetilir | Her gün 02:30, kaçırırsa telafi et |

Senaryo: `/etc` dizinini her gece yedekleyen bir iş kuracağız, bir kullanıcıya her dakika çalışan bir görev tanımlayacağız ve bir bakım işini tek seferlik zamanlayacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denenmiştir. Minimal kurulumlarda paketler eksik olabilir: `sudo apt install cron at anacron`.

## İçindekiler

1. [cron zaman ifadesi](#1-cron-zaman-ifadesi)
2. [Kullanıcı crontab'ı](#2-kullanıcı-crontabı)
3. [Sistem geneli: /etc/crontab ve /etc/cron.d](#3-sistem-geneli-etccrontab-ve-etccrond)
4. [Hazır dizinler: /etc/cron.daily ve diğerleri](#4-hazır-dizinler-etccrondaily-ve-diğerleri)
5. [cron işinin ortamı ve logları](#5-cron-işinin-ortamı-ve-logları)
6. [anacron](#6-anacron)
7. [Tek seferlik işler: at](#7-tek-seferlik-işler-at)
8. [systemd timer](#8-systemd-timer)
9. [Hangisi ne zaman?](#9-hangisi-ne-zaman)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. cron zaman ifadesi

Bir cron satırı 5 zaman alanı ve bir komuttan oluşur:

```
┌───────────── dakika (0-59)
│ ┌─────────── saat (0-23)
│ │ ┌───────── ayın günü (1-31)
│ │ │ ┌─────── ay (1-12)
│ │ │ │ ┌───── haftanın günü (0-7, 0 ve 7 pazar)
│ │ │ │ │
* * * * *  komut
```

| Karakter | Anlamı | Örnek |
|---|---|---|
| `*` | Her değer | `*` saat alanında: her saat |
| `,` | Liste | `15,45`: 15. ve 45. dakika |
| `-` | Aralık | `1-5`: pazartesi–cuma |
| `/` | Adım | `*/10`: her 10 dakikada |

| İfade | Ne zaman? |
|---|---|
| `30 2 * * *` | Her gün 02:30 |
| `0 * * * *` | Her saat başı |
| `*/15 * * * *` | 15 dakikada bir |
| `0 9 * * 1-5` | Hafta içi her gün 09:00 |
| `0 0 1 * *` | Her ayın 1'i, gece yarısı |
| `0 3 * * 0` | Her pazar 03:00 |
| `0 8-18/2 * * *` | 08:00'den 18:00'e, 2 saatte bir |

Kısaltmalar: `@reboot` (açılışta bir kez), `@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly`.

Ay ve gün alanlarının ikisi birden verilirse (`0 0 13 * 5`), ikisinden **biri** tutunca çalışır: her ayın 13'ünde **ve** her cuma. Emin olmadığın bir ifadeyi crontab.guru gibi bir sitede veya `man 5 crontab` ile kontrol et.

---

## 2. Kullanıcı crontab'ı

Her kullanıcının kendi crontab'ı vardır:

```bash
crontab -e      # düzenle (ilk seferde hangi editörün kullanılacağını sorar)
crontab -l      # listele
crontab -r      # TAMAMEN sil (onay sormaz)
```

Root, başka bir kullanıcının crontab'ını yönetebilir:

```bash
sudo crontab -u ayse -e
sudo crontab -u ayse -l
sudo crontab -u ayse -r
```

`crontab -e` kaydederken satırları kontrol eder. Hatalı bir satır kurulmaz ve eski crontab olduğu gibi kalır. Denemede `61 * * * *` yazdık:

```
"-":0: bad minute
errors in crontab file, can't install.
```

Kullanıcı crontab'ları `/var/spool/cron/crontabs/` altında, kullanıcının adıyla saklanır:

```
-rw------- 1 ayse crontab 302 Nov 26 09:02 ayse
```

Bu dosyaları elle düzenleme, her zaman `crontab` komutunu kullan. Bir dosyadaki satırları doğrudan crontab olarak kurmak için: `crontab -u ayse dosya.cron`.

> [!WARNING]
> `crontab -r`, `-e`'nin hemen yanındaki tuştur ve hiçbir şey sormadan bütün crontab'ı siler. Önemli crontab'ların yedeğini al: `crontab -l > crontab.yedek`.

### Kimler kullanabilir?

`/etc/cron.allow` varsa sadece içindeki kullanıcılar, yoksa `/etc/cron.deny`'da olmayan herkes crontab kullanabilir. Ubuntu'da ikisi de varsayılan olarak yok, yani herkes kullanabilir. `at` için aynı mantıkla `/etc/at.allow` ve `/etc/at.deny` çalışır.

---

## 3. Sistem geneli: /etc/crontab ve /etc/cron.d

`/etc/crontab` ve `/etc/cron.d/` altındaki dosyalar, kullanıcı crontab'ından farklı olarak **6. alanda kullanıcı adı** içerir:

```
17 *	* * *	root	cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
```

Kendi sistem işini eklemek için `/etc/crontab`'ı değiştirmek yerine `/etc/cron.d/` altına ayrı bir dosya koy:

```bash
echo '30 2 * * * root /usr/bin/tar -czf /yedek/etc.tar.gz /etc' | sudo tee /etc/cron.d/etc-yedek
```

| | Kullanıcı crontab'ı | `/etc/crontab`, `/etc/cron.d/` |
|---|---|---|
| Düzenleme | `crontab -e` | Doğrudan dosya |
| Kullanıcı alanı | Yok | Var (6. alan) |
| Söz dizimi kontrolü | Kaydederken | Yok, hata sadece logda görünür |

`/etc/cron.d/` altındaki dosya adlarında nokta olmamalı (bkz. sonraki bölüm).

---

## 4. Hazır dizinler: /etc/cron.daily ve diğerleri

Saati önemli olmayan işler için hazır dizinler var. İçlerine konan script'ler `run-parts` ile çalıştırılır:

| Dizin | Ne zaman? |
|---|---|
| `/etc/cron.hourly/` | Saatte bir |
| `/etc/cron.daily/` | Günde bir |
| `/etc/cron.weekly/` | Haftada bir |
| `/etc/cron.monthly/` | Ayda bir |

```bash
sudo cp yedekle /etc/cron.daily/
sudo chmod +x /etc/cron.daily/yedekle
```

> [!IMPORTANT]
> `run-parts`, adında **nokta** olan dosyaları **çalıştırmaz** ve hiçbir hata vermez. Üç script koyup hangilerinin çalışacağını sorduk:

```bash
run-parts --test /etc/cron.daily
```

| Dosya | Çalıştırma izni | Çalışır mı? |
|---|---|---|
| `yedek` | Var | Evet |
| `yedek.sh` | Var | **Hayır** (adında nokta var) |
| `izinsiz` | Yok | **Hayır** |

Script'i `/etc/cron.daily/` altına koyarken `.sh` uzantısını kaldır ve `chmod +x` ver. `run-parts --test` komutu, neyin çalışacağını hiçbir şeyi çalıştırmadan gösterir.

Ubuntu'da anacron kuruluysa `daily`, `weekly` ve `monthly` dizinlerini cron değil anacron çalıştırır (`/etc/crontab`'daki `test -x /usr/sbin/anacron ||` kısmı bunu sağlar).

---

## 5. cron işinin ortamı ve logları

Bir komut terminalde çalışıp cron'da çalışmıyorsa, sebep genelde ortam farkıdır. Bir cron işine ortam değişkenlerini yazdırdık:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin HOME=/home/ayse SHELL=/bin/sh
```

- **SHELL=/bin/sh**: Komutlar bash ile değil `sh` (dash) ile çalışır. `[[ ]]`, `{1..10}` gibi bash'e özgü yazımlar hata verir. Bash gerekiyorsa script'in ilk satırına `#!/bin/bash` yaz veya crontab'ın başına `SHELL=/bin/bash` ekle.
- **PATH**: Ubuntu'da cron, PATH'i PAM üzerinden `/etc/environment`'tan okur, bu yüzden geniş bir PATH görürsün. Başka dağıtımlarda cron'un PATH'i sadece `/usr/bin:/bin`'dir. Ayrıca `~/.bashrc`'de PATH'e eklediğin dizinler cron'da **yoktur**. Komutları her zaman **tam yolla** yaz: `/usr/bin/tar`, `/usr/local/bin/yedekle`.
- **Çalışma dizini**: Kullanıcının ev dizini. Göreli yollar beklediğin yere gitmeyebilir.
- **`%` karakteri**: crontab satırında satır sonu anlamına gelir. `date +%F` yazacaksan `date +\%F`.

### Çalıştı mı?

```bash
journalctl -u cron -n 20
grep CRON /var/log/syslog
```
```
CRON[2030]: (ayse) CMD (date >> /tmp/cron-calisti.txt)
```

Bu satır cron'un işi **başlattığını** gösterir, başarılı olduğunu değil. İşin çıktısını ve hatalarını görmek için satırın sonuna yönlendirme ekle:

```
30 2 * * * /usr/local/bin/yedekle >> /var/log/yedekle.log 2>&1
```

Yönlendirme olmazsa cron çıktıyı e-postayla göndermeye çalışır, mail sistemi kurulu değilse çıktı kaybolur.

---

## 6. anacron

cron, işin zamanında sistem kapalıysa o işi **atlar**. anacron ise "en son ne zaman çalıştı?" diye bakar, süresi geçmişse sistem açılınca çalıştırır. Sürekli açık olmayan makineler için uygundur.

`/etc/anacrontab`:

```
1	5	cron.daily	run-parts --report /etc/cron.daily
7	10	cron.weekly	run-parts --report /etc/cron.weekly
@monthly	15	cron.monthly	run-parts --report /etc/cron.monthly
```

| Alan | Anlamı |
|---|---|
| 1. Periyot | Kaç günde bir (`1` her gün, `7` her hafta, `@monthly`) |
| 2. Gecikme | Anacron başladıktan kaç dakika sonra |
| 3. İş adı | Benzersiz bir ad. Son çalışma zamanı `/var/spool/anacron/İŞ-ADI` dosyasında tutulur |
| 4. Komut | Çalışacak komut |

Kendi işini eklemek:

```
3	10	etc-yedek	/usr/bin/tar -czf /yedek/etc.tar.gz /etc
```

3 günde bir, anacron başladıktan 10 dakika sonra çalışır.

```bash
sudo anacron -T          # anacrontab'ı kontrol et (hata yoksa sessiz)
sudo anacron -f -n       # bütün işleri hemen çalıştır, gecikmeyi bekleme
```

anacron'un sınırları: en küçük birim **gündür** (saat veremezsin) ve sadece root için çalışır.

---

## 7. Tek seferlik işler: at

```bash
at 23:30
at> /usr/local/bin/bakim
at> <Ctrl+D>
```

Komutu borudan da verebilirsin:

```bash
echo 'date > /tmp/at-calisti.txt' | at now + 1 minute
```
```
warning: commands will be executed using /bin/sh
job 1 at Thu Nov 26 09:03:00 2026
```

Kabul edilen zaman biçimleri (hepsini denedik):

| Yazım | Sonuç |
|---|---|
| `at 23:30` | Bugün 23:30 (geçtiyse yarın) |
| `at now + 3 hours` | 3 saat sonra (`minutes`, `days`, `weeks` de olur) |
| `at 15:00 tomorrow` | Yarın 15:00 |
| `at 2:30 PM` | 14:30 |
| `at teatime` | 16:00 |
| `at 'August 20 2027'` | O gün, **şu anki saatte** |

```bash
atq            # bekleyen işler
at -c 5        # 5 numaralı işin içeriği
atrm 5         # sil
```
```
4	Fri Nov 27 15:00:00 2026 a root
1	Thu Nov 26 09:03:00 2026 a root
```

`at -c` çıktısı uzundur, çünkü `at` işi oluşturduğun andaki ortam değişkenlerini de kaydeder. Asıl komut en alttadır. `at` için `atd` servisinin çalışıyor olması gerekir: `systemctl status atd`.

---

## 8. systemd timer

Timer, bir servisin ne zaman çalışacağını belirleyen bir unit'tir. İki dosyadan oluşur: işi yapan `.service` ve zamanı belirleyen aynı adlı `.timer`.

**İş:**

```ini
# /etc/systemd/system/yedekle.service
[Unit]
Description=Gece yedegi

[Service]
Type=oneshot
ExecStart=/usr/bin/tar -czf /tmp/etc-yedek.tar.gz /etc
```

**Zaman:**

```ini
# /etc/systemd/system/yedekle.timer
[Unit]
Description=Gece yedegini her gun 02:30'da calistir

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Etkinleştir** (servisi değil, timer'ı):

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now yedekle.timer
systemctl list-timers yedekle.timer
```
```
NEXT                        LEFT LAST PASSED UNIT          ACTIVATES
Fri 2026-11-27 02:30:00 UTC  17h -         - yedekle.timer yedekle.service
```

İşi zamanını beklemeden denemek için servisi doğrudan başlat: `sudo systemctl start yedekle.service`. Çıktısı journal'a gider: `journalctl -u yedekle`.

### OnCalendar yazımı

Biçim `HaftaGünü Yıl-Ay-Gün Saat:Dakika:Saniye`. Bir ifadenin ne zaman tetikleneceğini `systemd-analyze calendar` ile kontrol et:

```bash
systemd-analyze calendar 'Mon..Fri 08:00' '*:0/15' weekly
```

| İfade | Açılımı | Anlamı |
|---|---|---|
| `*-*-* 02:30:00` | | Her gün 02:30 |
| `Mon..Fri 08:00` | `Mon..Fri *-*-* 08:00:00` | Hafta içi 08:00 |
| `weekly` | `Mon *-*-* 00:00:00` | Pazartesi gece yarısı |
| `*:0/15` | `*-*-* *:00/15:00` | 15 dakikada bir |
| `daily`, `hourly`, `monthly` | | Kısaltmalar |

Diğer yararlı ayarlar:

| Ayar | Anlamı |
|---|---|
| `Persistent=true` | Sistem kapalıyken kaçırılan çalıştırmayı açılınca yap (anacron gibi) |
| `OnBootSec=5min` | Açılıştan 5 dakika sonra |
| `OnUnitActiveSec=1h` | Son çalışmadan 1 saat sonra tekrar |
| `RandomizedDelaySec=10min` | 0–10 dakika arası rastgele gecikme |

### Tek seferlik timer

`at`'in systemd karşılığı:

```bash
sudo systemd-run --on-active=2min /usr/bin/touch /tmp/tek-sefer
```
```
Running timer as unit: run-...timer
Will run service as unit: run-...service
```

| | cron | systemd timer |
|---|---|---|
| Kurulum | Tek satır | İki dosya |
| Log | Sadece başladığı | Bütün çıktısı journal'da |
| Kaçırılanı telafi | Hayır (anacron gerekir) | `Persistent=true` |
| Kaynak sınırı, kullanıcı, bağımlılık | Yok | Servis gibi ([Uygulama ve Servis Kısıtlamaları](../01-temel-komutlar/16-uygulama-ve-servis-kisitlamalari.md)) |
| Listeleme | `crontab -l` | `systemctl list-timers` |

---

## 9. Hangisi ne zaman?

| İhtiyaç | Araç |
|---|---|
| Her gün belirli bir saatte, sunucu hep açık | cron veya systemd timer |
| Kullanıcının kendi işi | `crontab -e` |
| Saat önemli değil, günde bir | `/etc/cron.daily/` |
| Makine kapalıysa açılınca telafi etsin | anacron veya `Persistent=true` timer |
| Bir kez, belirli bir zamanda | `at` veya `systemd-run --on-calendar` |
| Ayrıntılı log, kaynak sınırı gerekiyor | systemd timer |

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `crontab -e` / `-l` / `-r` | Kendi crontab'ını düzenle / listele / sil |
| `sudo crontab -u ayse -l` | Başka kullanıcının crontab'ı |
| `crontab dosya` | Dosyayı crontab olarak kur |
| `/etc/cron.d/dosya` | Sistem işi (kullanıcı alanıyla) |
| `/etc/cron.daily/` | Günlük script'ler (adında nokta olmasın) |
| `run-parts --test /etc/cron.daily` | Hangi script'ler çalışacak |
| `journalctl -u cron` | cron logları |
| `/etc/anacrontab` | anacron işleri |
| `anacron -T`, `anacron -f -n` | Kontrol et, hemen çalıştır |
| `at 23:30`, `at now + 1 hour` | Tek seferlik iş |
| `atq`, `at -c N`, `atrm N` | Listele, içeriğini gör, sil |
| `systemctl list-timers` | Timer'lar |
| `systemctl enable --now ad.timer` | Timer'ı etkinleştir |
| `systemd-analyze calendar 'ifade'` | Takvim ifadesini kontrol et |
| `systemd-run --on-active=5min komut` | Tek seferlik timer |

---

## 11. Sık yapılan hatalar

- **`/etc/cron.daily/`'ye `.sh` uzantılı veya çalıştırma izni olmayan script koymak.** Sessizce atlanır.
- **Kullanıcı crontab'ına kullanıcı alanı yazmak veya `/etc/cron.d/`'de unutmak.**
- **Komutlarda tam yol kullanmamak.**
- **Cron'da bash'e özgü yazım kullanmak.** `sh` ile çalışır.
- **`%` karakterini kaçırmamak.**
- **Cron işinin çıktısını yönlendirmemek.** Hatayı göremezsin.
- **`crontab -e` yerine `crontab -r` yazmak.**
- **systemd timer'da `.service`'i enable etmek.** Etkinleştirilen `.timer` olmalı.
- **`at`'in `atd` servisi durmuşken iş beklemek.**

---

## 12. Alıştırmalar

**1.** `ayse` kullanıcısı için, her hafta içi saat 18:15'te `/home/ayse/rapor` dizinini `/tmp/rapor.tar.gz` olarak arşivleyen bir cron işi kur. Kurulduğunu kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo crontab -u ayse -e
# 15 18 * * 1-5 /usr/bin/tar -czf /tmp/rapor.tar.gz /home/ayse/rapor

sudo crontab -u ayse -l
```
</details>

**2.** Root olarak, her 10 dakikada bir `/usr/bin/df -h /` çıktısını `/var/log/disk.log` dosyasına **ekleyen** bir sistem işi tanımla. `/etc/crontab`'ı değiştirme.

<details>
<summary>Çözüm</summary>

```bash
echo '*/10 * * * * root /usr/bin/df -h / >> /var/log/disk.log 2>&1' | sudo tee /etc/cron.d/disk-izleme
```

`/etc/cron.d/` altında 6. alan kullanıcı adıdır.
</details>

**3.** `/etc/cron.daily/` altına `temizle.sh` adında bir script koymak istiyorsun. Çalışacağından nasıl emin olursun?

<details>
<summary>Çözüm</summary>

```bash
sudo cp temizle.sh /etc/cron.daily/temizle      # nokta olmadan
sudo chmod +x /etc/cron.daily/temizle
run-parts --test /etc/cron.daily                # listede görünmeli
```
</details>

**4.** Bugün saat 22:00'de `/usr/sbin/reboot` çalıştıracak tek seferlik bir iş oluştur, bekleyen işlerde gör, içeriğini kontrol et, sonra iptal et.

<details>
<summary>Çözüm</summary>

```bash
echo '/usr/sbin/reboot' | sudo at 22:00
sudo atq
sudo at -c NUMARA | tail -2
sudo atrm NUMARA
```
</details>

**5.** `/usr/local/bin/temizle` script'ini her gün 03:00'te çalıştıran ve sistem o saatte kapalıysa açılınca çalıştıran bir systemd timer kur. Bir sonraki çalışma zamanını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo tee /etc/systemd/system/temizle.service > /dev/null <<'EOF'
[Unit]
Description=Gunluk temizlik

[Service]
Type=oneshot
ExecStart=/usr/local/bin/temizle
EOF

sudo tee /etc/systemd/system/temizle.timer > /dev/null <<'EOF'
[Unit]
Description=Gunluk temizlik zamanlayicisi

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now temizle.timer
systemctl list-timers temizle.timer
```
</details>

---

Önceki konu: [Log Dosyaları](04-log-dosyalari.md) · Sonraki konu: [Paket Yönetimi](06-paket-yonetimi.md)
