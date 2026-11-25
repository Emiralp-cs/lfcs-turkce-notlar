# Saat Dilimi ve NTP ile Saat Senkronizasyonu

Bilgisayarların donanım saati zamanla kayar. Bir sunucu birkaç hafta içinde saniyeler hatta dakikalar sapabilir. Bu küçük gibi görünen fark ciddi sorunlara yol açar:

- **Loglar:** Birden fazla sunucunun loglarını yan yana koyduğunda olayların sırası karışır.
- **Sertifikalar:** Saati yanlış olan bir makine, geçerli bir TLS sertifikasını "henüz geçerli değil" veya "süresi dolmuş" diye reddedebilir.
- **Zamanlanmış işler:** Cron görevleri yanlış saatte çalışır.

**NTP** (Network Time Protocol), saati ağdaki zaman sunucularıyla sürekli eşitleyerek bu kaymayı düzeltir. Saat dilimi ise ayrı bir konudur: saatin doğru olması ile doğru bölgeye göre gösterilmesi farklı şeylerdir.

Senaryo: Ekibin sunucusu İstanbul'da. Loglar yerel saatle okunacak ve saat Türkiye'deki NTP sunucularıyla eşitlenecek.

> [!NOTE]
> Saat dilimi dosyaları, `TZ` değişkeni ve timesyncd yapılandırma dosyaları Ubuntu 24.04'te denenmiştir. Deneme ortamı systemd ile çalışmadığı, saat değiştirmeye izin vermediği ve NTP trafiğini dışarı çıkarmadığı için `timedatectl` komutları ve senkronizasyonun kendisi çalıştırılamadı. Bu kısımlar systemd ve chrony belgelerine dayanıyor.

## İçindekiler

1. [Sistem saati, UTC ve saat dilimi](#1-sistem-saati-utc-ve-saat-dilimi)
2. [Saat dilimini görmek ve ayarlamak](#2-saat-dilimini-görmek-ve-ayarlamak)
3. [timedatectl çıktısını okumak](#3-timedatectl-çıktısını-okumak)
4. [NTP senkronizasyonunu açmak: systemd-timesyncd](#4-ntp-senkronizasyonunu-açmak-systemd-timesyncd)
5. [Kendi NTP sunucularını kullanmak](#5-kendi-ntp-sunucularını-kullanmak)
6. [Senkronizasyonu doğrulamak](#6-senkronizasyonu-doğrulamak)
7. [Alternatif: chrony](#7-alternatif-chrony)
8. [Sorun giderme](#8-sorun-giderme)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Sistem saati, UTC ve saat dilimi

Linux saati içeride **UTC** olarak tutar. Saat dilimi sadece saatin **nasıl gösterileceğini** belirler.

Bunu görmenin en kolay yolu `TZ` değişkeni. Bu değişken tek bir komut için saat dilimini değiştirir, sistemin ayarına dokunmaz. Aynı anda farklı bölgelere baktık:

```bash
date -u
TZ=Europe/Istanbul date
TZ=Europe/Berlin date
TZ=Asia/Singapore date
```
```
Wed Nov 25 14:07:34 UTC 2026
Wed Nov 25 17:07:34 +03 2026
Wed Nov 25 15:07:34 CET 2026
Wed Nov 25 22:07:34 +08 2026
```

Dört satır da aynı anı gösteriyor, sadece gösterim farklı. Saat dilimini değiştirmek saati ileri veya geri almaz.

> [!TIP]
> Farklı ülkelerde sunucuları olan ekipler çoğu zaman bütün sunucuları UTC'de tutar. Böylece loglar karşılaştırılırken kimse saat farkı hesaplamak zorunda kalmaz. Tek bir ofise hizmet eden bir sunucu içinse yerel saat dilimi daha pratik olabilir.

---

## 2. Saat dilimini görmek ve ayarlamak

### timedatectl ile

```bash
timedatectl                                  # mevcut durum
timedatectl list-timezones                   # bütün saat dilimleri
timedatectl list-timezones | grep -i istanbul
sudo timedatectl set-timezone Europe/Istanbul
```

Saat dilimi adları `Bölge/Şehir` biçimindedir:

- Birden fazla kelimeli şehir adlarında boşluk yerine alt çizgi kullanılır: `America/Los_Angeles`.
- Bazı adlar üç parçalıdır: `America/Argentina/Buenos_Aires`. Ubuntu 24.04'te eski kısa ad `America/Buenos_Aires` varsayılan olarak yüklü değil, denemede sadece üç parçalı ad bulundu.

Bir şehrin tam adını tahmin etmek yerine `list-timezones` çıktısında ara.

### Elle: /etc/localtime

`timedatectl` aslında `/etc/localtime` bağlantısını değiştirir. Bu dosya, `/usr/share/zoneinfo/` altındaki saat dilimi dosyalarından birini gösterir:

```bash
ls -l /etc/localtime
```
```
lrwxrwxrwx 1 root root 27 Oct 27 06:10 /etc/localtime -> /usr/share/zoneinfo/Etc/UTC
```

`timedatectl` kullanılamayan bir sistemde bağlantıyı elle değiştirebilirsin:

```bash
sudo ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime
date
```
```
Wed Nov 25 17:07:48 +03 2026
```

Yapabiliyorsan `timedatectl set-timezone` kullan, ama arka planda olanı bilmek sorun giderirken işe yarar.

---

## 3. timedatectl çıktısını okumak

`timedatectl` komutunu parametresiz çalıştırınca şu satırları görürsün:

| Satır | Anlamı |
|---|---|
| `Local time` | Ayarlı saat dilimine göre yerel saat |
| `Universal time` | UTC saati |
| `RTC time` | Anakarttaki donanım saati |
| `Time zone` | Saat dilimi ve UTC'ye göre farkı, örn. `Europe/Istanbul (+03, +0300)` |
| `System clock synchronized` | Saat bir NTP sunucusuyla eşitlenmiş mi (`yes` / `no`) |
| `NTP service` | Senkronizasyon servisi açık mı (`active` / `inactive`) |
| `RTC in local TZ` | Donanım saati yerel saatte mi tutuluyor. Linux'ta `no`, yani UTC olmalı |

En önemli iki satır `System clock synchronized` ve `NTP service`. İkisi de olumlu değilse saat eşitlenmiyor demektir.

---

## 4. NTP senkronizasyonunu açmak: systemd-timesyncd

Ubuntu'da varsayılan senkronizasyon servisi **systemd-timesyncd**. Ubuntu 24.04'te ayrı bir paket olarak gelir. Kurulu değilse:

```bash
sudo apt install systemd-timesyncd
```

Senkronizasyonu aç:

```bash
sudo timedatectl set-ntp true
```

Bu komut senkronizasyon servisini başlatır ve açılışta çalışacak şekilde ayarlar. Kapatmak için `set-ntp false` yazılır.

Servisin durumunu ayrıntılı görmek için:

```bash
systemctl status systemd-timesyncd
```

Çıktının `Status:` satırında servisin en son hangi zaman sunucusuyla konuştuğu yazar.

> [!TIP]
> `timedatectl ` yazıp iki kez Tab'a basarsan `show-timesync`, `timesync-status` gibi diğer alt komutları görebilirsin.

---

## 5. Kendi NTP sunucularını kullanmak

Varsayılan yapılandırma dosyası `/etc/systemd/timesyncd.conf`. Ubuntu 24.04'teki hali (açıklama satırları hariç):

```ini
[Time]
#NTP=
#FallbackNTP=ntp.ubuntu.com
#RootDistanceMaxSec=5
#PollIntervalMinSec=32
#PollIntervalMaxSec=2048
#ConnectionRetrySec=30
#SaveIntervalSec=60
```

Başında `#` olan satırlar varsayılan değerleri gösterir ama etkin değildir.

| Ayar | Anlamı |
|---|---|
| `NTP=` | Kullanılacak NTP sunucuları, **boşlukla** ayrılır |
| `FallbackNTP=` | `NTP=` boşsa veya sunuculara ulaşılamazsa kullanılacak yedekler |

### Ayrı bir dosyayla (önerilen)

Ana dosyayı değiştirmek yerine `timesyncd.conf.d` dizinine kendi dosyanı koyabilirsin. Paket güncellemeleri bu dosyaya dokunmaz:

```bash
sudo mkdir -p /etc/systemd/timesyncd.conf.d
sudo vim /etc/systemd/timesyncd.conf.d/ekip.conf
```

```ini
[Time]
NTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org
FallbackNTP=ntp.ubuntu.com
```

Ayarların gerçekten okunduğunu, bütün dosyaları birleştirip gösteren şu komutla kontrol edebilirsin:

```bash
systemd-analyze cat-config systemd/timesyncd.conf
```
```
# /etc/systemd/timesyncd.conf
[Time]
...
# /etc/systemd/timesyncd.conf.d/ekip.conf
[Time]
NTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org
FallbackNTP=ntp.ubuntu.com
```

> [!WARNING]
> `timesyncd.conf.d` altındaki dosyanın adı **`.conf` ile bitmeli**. Denemede aynı dosyayı uzantısız `ekip` adıyla koyduğumuzda `cat-config` çıktısında hiç görünmedi.

Değişikliği uygulamak için servisi yeniden başlat:

```bash
sudo systemctl restart systemd-timesyncd
```

`pool.ntp.org`, dünya çapında gönüllülerin işlettiği bir NTP sunucu havuzu. `tr.pool.ntp.org` Türkiye'deki, `de.pool.ntp.org` Almanya'daki sunucuları verir. Başlarındaki `0.`, `1.` farklı sunucular almak içindir.

---

## 6. Senkronizasyonu doğrulamak

```bash
timedatectl show-timesync      # kullanılan sunucu ve ayarlar
timedatectl timesync-status    # senkronizasyonun ayrıntıları
```

`timesync-status` çıktısındaki alanlar:

| Alan | Anlamı |
|---|---|
| `Server` | Şu an kullanılan NTP sunucusu |
| `Poll interval` | Sunucuya ne sıklıkla sorulduğu |
| `Stratum` | Sunucunun referans saate uzaklığı. 1, atom saati veya GPS'e doğrudan bağlı sunucu demektir. Küçük sayı daha iyidir |
| `Offset` | Yerel saat ile sunucu saati arasındaki fark |
| `Delay` | Sunucuya gidip gelme süresi |
| `Jitter` | Gecikmedeki dalgalanma |

Saatin eşitlendiğinin asıl kanıtı `timedatectl` çıktısındaki `System clock synchronized: yes` satırıdır.

---

## 7. Alternatif: chrony

RHEL ailesinde ve bazı Ubuntu kurulumlarında senkronizasyon için **chrony** kullanılır. LFCS sınavı belirli bir dağıtıma bağlı olmadığı için chrony'yi tanımak faydalı.

Ubuntu'da chrony kurmak timesyncd'yi kaldırır, çünkü iki senkronizasyon servisi aynı anda çalışmamalı. `apt` simülasyonunda bu açıkça görünüyor:

```bash
apt-get install -s chrony | grep -E '^(Inst|Remv)'
```
```
Remv systemd-timesyncd [255.4-1ubuntu8.19]
Inst tzdata [2026e-0ubuntu0.24.04.1] (2026f-0ubuntu0.24.04.1 Ubuntu:24.04/noble-updates, Ubuntu:24.04/noble-security [all])
Inst tzdata-legacy (2026f-0ubuntu0.24.04.1 Ubuntu:24.04/noble-updates, Ubuntu:24.04/noble-security [all])
Inst chrony (4.5-1ubuntu4.2 Ubuntu:24.04/noble-updates [amd64])
```

İlk satır (`Remv`) timesyncd'nin kaldırılacağını gösteriyor.

Ubuntu'daki varsayılan yapılandırma dosyası `/etc/chrony/chrony.conf`'tan (RHEL'de `/etc/chrony.conf`) bir kısım:

```
pool ntp.ubuntu.com        iburst maxsources 4
pool 0.ubuntu.pool.ntp.org iburst maxsources 1
pool 1.ubuntu.pool.ntp.org iburst maxsources 1
pool 2.ubuntu.pool.ntp.org iburst maxsources 2
...
makestep 1 3
```

- `pool` birden fazla adres döndüren bir havuzu, `server` tek bir sunucuyu ekler.
- `iburst` açılışta hızlı eşitleme için ilk istekleri arka arkaya gönderir.

Kendi sunucularını eklemek için bu satırları değiştir veya `server ntp.ornek.local iburst` gibi satırlar ekle, sonra servisi yeniden başlat. Servisin adı Ubuntu'da `chrony`, RHEL'de `chronyd`:

```bash
sudo systemctl restart chrony     # RHEL'de: chronyd
chronyc sources -v                # sunucular ve durumları
chronyc tracking                  # senkronizasyonun ayrıntıları
```

`chronyc sources` çıktısında başında `*` olan satır, şu an saatin eşitlendiği sunucudur.

---

## 8. Sorun giderme

**`System clock synchronized: no` görüyorsan:**

1. `NTP service: active` mi? Değilse `sudo timedatectl set-ntp true`.
2. `systemctl status systemd-timesyncd` çıktısında hata var mı?
3. NTP **UDP 123** portunu kullanır. Güvenlik duvarı veya ağdaki bir firewall bu trafiği engelliyorsa senkronizasyon olmaz.
4. Yazdığın sunucu adları çözümlenebiliyor mu? `getent hosts 0.tr.pool.ntp.org`.
5. İki senkronizasyon servisi (timesyncd ve chrony) birlikte kurulu değil mi?

---

## 9. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `date` / `date -u` | Yerel saati / UTC'yi gösterir |
| `TZ=Bölge/Şehir date` | Tek bir komut için başka saat dilimini kullanır |
| `timedatectl` | Saat, saat dilimi ve senkronizasyon durumunu gösterir |
| `timedatectl list-timezones` | Saat dilimlerini listeler |
| `timedatectl set-timezone Europe/Istanbul` | Saat dilimini ayarlar |
| `/etc/localtime` | Geçerli saat dilimi dosyasına bağlantı |
| `timedatectl set-ntp true` | NTP senkronizasyonunu açar |
| `systemctl status systemd-timesyncd` | Senkronizasyon servisinin durumu |
| `/etc/systemd/timesyncd.conf` | timesyncd ayarları (`NTP=`, `FallbackNTP=`) |
| `/etc/systemd/timesyncd.conf.d/*.conf` | Ayrı ayar dosyaları |
| `systemd-analyze cat-config systemd/timesyncd.conf` | Birleşmiş ayarları gösterir |
| `systemctl restart systemd-timesyncd` | Ayarları uygular |
| `timedatectl show-timesync` | Kullanılan sunucuyu gösterir |
| `timedatectl timesync-status` | Stratum, offset gibi ayrıntıları gösterir |
| `/etc/chrony/chrony.conf` | chrony ayarları (RHEL: `/etc/chrony.conf`) |
| `chronyc sources -v` / `chronyc tracking` | chrony'nin sunucuları ve durumu |

---

## 10. Sık yapılan hatalar

- **Saat dilimi ile saat senkronizasyonunu karıştırmak.** Saat dilimi gösterimi, NTP saatin doğruluğunu belirler. İkisi de ayrı ayrı ayarlanmalı.
- **Saat dilimi adını tahmin etmek.** `timedatectl list-timezones | grep` ile tam adı bul.
- **NTP sunucu adlarında alt çizgi kullanmak.** Alt çizgi sadece saat dilimi adlarında boşluk yerine kullanılır (`Los_Angeles`). Sunucu adları normal alan adlarıdır: `0.tr.pool.ntp.org`.
- **`NTP=` satırında sunucuları virgülle ayırmak.** Boşlukla ayrılır.
- **Ayar dosyasını değiştirip servisi yeniden başlatmamak.** Değişiklik uygulanmaz.
- **Ayrı ayar dosyasına `.conf` uzantısı vermemek.** Dosya okunmaz.
- **timesyncd ve chrony'yi birlikte çalıştırmak.** İki servis saati birbirine karşı düzeltmeye çalışır.
- **UDP 123'ü unutmak.** Senkronizasyon olmuyorsa güvenlik duvarına bak.

---

## 11. Alıştırmalar

**1.** Sunucunun saat dilimini `Europe/Istanbul` yap ve doğrula. Sistem ayarını değiştirmeden Tokyo'da saatin kaç olduğunu göster.

<details>
<summary>Çözüm</summary>

```bash
sudo timedatectl set-timezone Europe/Istanbul
timedatectl | grep 'Time zone'
TZ=Asia/Tokyo date
```
</details>

**2.** Buenos Aires'in saat dilimi adını bul.

<details>
<summary>Çözüm</summary>

```bash
timedatectl list-timezones | grep -i buenos
```

Sonuç: `America/Argentina/Buenos_Aires`.
</details>

**3.** timesyncd, `0.tr.pool.ntp.org` ve `1.tr.pool.ntp.org` sunucularını kullansın. Ana yapılandırma dosyasına dokunma. Ayarın okunduğunu doğrula ve uygula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /etc/systemd/timesyncd.conf.d
printf '[Time]\nNTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org\n' | sudo tee /etc/systemd/timesyncd.conf.d/ekip.conf
systemd-analyze cat-config systemd/timesyncd.conf
sudo systemctl restart systemd-timesyncd
timedatectl show-timesync
```
</details>

**4.** `timedatectl` çıktısında `NTP service: inactive` ve `System clock synchronized: no` görüyorsun. Ne yaparsın?

<details>
<summary>Çözüm</summary>

```bash
sudo timedatectl set-ntp true
systemctl status systemd-timesyncd
timedatectl
```

Birkaç saniye sonra hala `synchronized: no` ise güvenlik duvarında UDP 123'ün açık olup olmadığını ve sunucu adlarının çözümlenip çözümlenmediğini kontrol et.
</details>

**5.** Aşağıdaki dosyada iki hata var. Bul.

Dosya adı: `/etc/systemd/timesyncd.conf.d/ntp`

```ini
[Time]
NTP=0.tr.pool.ntp.org,1.tr.pool.ntp.org
```

<details>
<summary>Çözüm</summary>

1. Dosya adı `.conf` ile bitmiyor, okunmaz. `ntp.conf` olmalı.
2. Sunucular virgülle değil boşlukla ayrılmalı: `NTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org`.
</details>

---

Önceki konu: [Ağ Yapılandırması](02-ag-yapilandirmasi.md) · Sonraki konu: [Ağ İzleme ve Sorun Giderme](04-ag-izleme.md)
