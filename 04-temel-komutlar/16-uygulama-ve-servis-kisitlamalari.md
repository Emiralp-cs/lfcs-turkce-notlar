# Uygulama ve Servis Kısıtlamaları

> **Resmi sınav maddesi:** Determine application and service specific constraints

Bir servis kendi başına çalışmaz: belirli bir kullanıcıyla çalışır, belirli portları dinler, belirli dosyalara erişir, belirli bir miktar bellek, işlemci ve dosya tanıtıcısıyla sınırlıdır, başka servislere bağımlıdır. Bu sınırların her biri bir **kısıtlamadır** (constraint). Bir servis "sebepsiz" çalışmıyorsa, genelde bu kısıtlamalardan birine takılmıştır. Ya da tersine, bir servisin sistemin geri kalanını etkilememesi için bu kısıtlamaları sen koyarsın.

Bu dosya, bir servisin hangi kısıtlamalar altında çalıştığını nasıl **tespit edeceğini** ve gerektiğinde nasıl **değiştireceğini** anlatıyor.

Senaryo: Ekip, `/usr/local/bin/web.py` adında küçük bir durum paneli yazdı. Bunu `panel.service` adıyla, root olmayan `nabizci` kullanıcısıyla, 80 portunda çalıştıracağız. Önce açılmayacak, sonra hangi kısıtlamaya takıldığını bulup çözeceğiz, en son da panelin sunucunun geri kalanını etkilemesini önleyecek sınırlar koyacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve systemd 255 ile denenmiştir. Servis dosyası yazma, drop-in ve sorun giderme adımları için önce [Servis Oluşturma ve Sorun Giderme](13-servis-olusturma-ve-sorun-giderme.md) dosyasını oku.

## İçindekiler

1. [Kısıtlama türleri](#1-kısıtlama-türleri)
2. [Bir servisin kısıtlamalarını okumak](#2-bir-servisin-kısıtlamalarını-okumak)
3. [Kullanıcı ve dosya izinleri](#3-kullanıcı-ve-dosya-izinleri)
4. [Port kısıtlamaları](#4-port-kısıtlamaları)
5. [Süreç limitleri: ulimit ve LimitNOFILE](#5-süreç-limitleri-ulimit-ve-limitnofile)
6. [Kaynak limitleri: bellek, işlemci, süreç sayısı](#6-kaynak-limitleri-bellek-işlemci-süreç-sayısı)
7. [Dosya sistemi erişimini kısıtlamak](#7-dosya-sistemi-erişimini-kısıtlamak)
8. [Güvenlik modülleri: SELinux ve AppArmor](#8-güvenlik-modülleri-selinux-ve-apparmor)
9. [Bağımlılıklar](#9-bağımlılıklar)
10. [Kontrol listesi](#10-kontrol-listesi)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Kısıtlama türleri

| Tür | Örnek | Nasıl görülür? |
|---|---|---|
| Kullanıcı ve izinler | Servis `nabizci` olarak çalışıyor, `/etc/panel.conf` sadece root'a açık | `systemctl show -p User`, `ls -l` |
| Ağ | 1024'ün altındaki portlar, başka servisin kullandığı port | `ss -tlnp` |
| Süreç limitleri | Açık dosya sayısı, süreç sayısı | `/proc/PID/limits` |
| Kaynak limitleri | En fazla 200 MB bellek, %20 işlemci | `systemctl show`, `systemctl status` |
| Dosya sistemi | Sadece `/var/lib/panel`'e yazabilir | `systemctl show -p ProtectSystem -p ReadWritePaths` |
| Güvenlik modülü | SELinux bağlamı, AppArmor profili | `ls -Z`, `aa-status` |
| Bağımlılık | Önce veritabanı başlamalı, bir kütüphane yüklü olmalı | `systemctl list-dependencies`, `ldd` |

---

## 2. Bir servisin kısıtlamalarını okumak

Unit dosyası (`systemctl cat`) sadece **yazılanları** gösterir. Varsayılanlar dahil, şu an **geçerli** olan değerler için:

```bash
systemctl show panel -p User -p MemoryMax -p TasksMax -p LimitNOFILE \
  -p ProtectSystem -p ReadWritePaths -p AmbientCapabilities -p CPUQuotaPerSecUSec
```
```
CPUQuotaPerSecUSec=infinity
MemoryMax=209715200
TasksMax=9605
LimitNOFILE=524288
AmbientCapabilities=cap_net_bind_service
User=nabizci
ReadWritePaths=/var/lib/panel
ProtectSystem=strict
```

`infinity` sınır yok demektir. `systemctl show panel` tek başına yüzlerce ayarın tamamını yazar, `| grep -i limit` gibi filtrelenebilir.

Çalışan sürecin gerçekte hangi limitlerle çalıştığı ise `/proc` altında:

```bash
cat /proc/$(systemctl show -p MainPID --value panel)/limits
```

---

## 3. Kullanıcı ve dosya izinleri

`User=` verilmezse servis root olarak çalışır ve hiçbir izin sorunu yaşamaz. Güvenlik için ayrı bir kullanıcıyla çalıştırınca ([13. dosya, Güvenlik](13-servis-olusturma-ve-sorun-giderme.md#6-servis-dosyasının-bölümleri)) o kullanıcının erişemediği her dosya bir kısıtlama olur.

Bir servis `Permission denied` ile çöküyorsa:

1. Servis hangi kullanıcıyla çalışıyor? `systemctl show panel -p User -p Group`
2. Erişmeye çalıştığı dosya veya dizin kimin? `ls -l /etc/panel.conf`, `ls -ld /var/lib/panel`
3. O kullanıcı olarak denediğinde ne oluyor? `sudo -u nabizci cat /etc/panel.conf`

Çözüm, servise root yetkisi vermek değil, ihtiyaç duyduğu dosyaların sahipliğini veya grubunu ayarlamaktır:

```bash
sudo mkdir -p /var/lib/panel
sudo chown nabizci: /var/lib/panel
sudo chgrp nabizci /etc/panel.conf && sudo chmod 640 /etc/panel.conf
```

İzin ve ACL ayrıntıları: [İzinler ve Özel Bitler](04-izinler-ve-ozel-bitler.md), [ACL](../05-kullanicilar-ve-gruplar/06-acl.md).

---

## 4. Port kısıtlamaları

### 1024'ün altındaki portlar

Panel servisini `User=nabizci` ile 80 portunda başlattık. Servis açılmadı, `journalctl -u panel`:

```
web.py[1636]: PermissionError: [Errno 13] Permission denied
panel.service: Main process exited, code=exited, status=1/FAILURE
panel.service: Start request repeated too quickly.
```

Dosya izni değil, **port** izni. Linux'ta 1024'ün altındaki portları sadece root (veya özel yetkisi olan süreçler) açabilir:

```bash
sysctl net.ipv4.ip_unprivileged_port_start
```
```
net.ipv4.ip_unprivileged_port_start = 1024
```

Çözüm seçenekleri:

| Yöntem | Nasıl? |
|---|---|
| 1024 üstü bir port kullan | Uygulamayı 8080'de çalıştır, önüne nginx gibi bir ters proxy koy ([Ters Proxy](../02-ag-yonetimi/09-ters-proxy-ve-yuk-dengeleme.md)) |
| Servise sadece bu yetkiyi ver | Drop-in'e `AmbientCapabilities=CAP_NET_BIND_SERVICE` |
| Root olarak çalıştır | Önerilmez |

İkinci yöntemi denedik:

```bash
sudo mkdir -p /etc/systemd/system/panel.service.d
printf '[Service]\nAmbientCapabilities=CAP_NET_BIND_SERVICE\n' | sudo tee /etc/systemd/system/panel.service.d/port.conf
sudo systemctl daemon-reload
sudo systemctl reset-failed panel
sudo systemctl start panel
```

Servis root olmadan 80 portunu açtı:

```bash
sudo ss -tlnp 'sport = :80'
```
```
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      5            0.0.0.0:80        0.0.0.0:*    users:(("python3",pid=1668,fd=3))
```

`CAP_NET_BIND_SERVICE`, root'un bütün yetkileri yerine sadece "düşük port açabilme" yetkisidir.

### Port zaten kullanımda

Panel çalışırken aynı porta ikinci bir program başlatmaya çalıştık:

```
OSError: [Errno 98] Address already in use
```

Bir portu kimin kullandığını bulmak için:

```bash
sudo ss -tlnp                    # dinleyen bütün TCP portları ve süreçleri
sudo ss -tlnp 'sport = :80'      # sadece 80
sudo lsof -i :80 -sTCP:LISTEN
```
```
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3 1668 nabizci    3u  IPv4  28628      0t0  TCP *:http (LISTEN)
```

`-t` TCP, `-u` UDP, `-l` dinleyenler, `-n` port numaralarını isme çevirme, `-p` süreci göster (`-p` için `sudo` gerekir, yoksa süreç sütunu boş kalır). Ağ izleme komutlarının ayrıntısı: [Ağ İzleme](../02-ag-yonetimi/04-ag-izleme.md).

Port açık ama bağlanılamıyorsa bir sonraki kısıtlama güvenlik duvarıdır ([UFW](../02-ag-yonetimi/06-ufw.md)). Bir de `Local Address` sütununa bak: `127.0.0.1:6379` sadece sunucunun kendisinden bağlantı kabul eder, `0.0.0.0:80` her yerden.

---

## 5. Süreç limitleri: ulimit ve LimitNOFILE

Her süreç, çekirdeğin koyduğu limitlerle çalışır: en fazla kaç dosya açabileceği, kaç alt süreç oluşturabileceği gibi. Çok sayıda bağlantı açan bir veritabanı veya web sunucusu "açık dosya" limitine takılınca `Too many open files` hatası verir.

Kullanıcı oturumları için bu limitler `/etc/security/limits.conf` ile ayarlanır ([Kaynak Limitleri](../05-kullanicilar-ve-gruplar/05-kaynak-limitleri.md)). Ama:

> [!IMPORTANT]
> `/etc/security/limits.conf`, **systemd servislerine uygulanmaz.** Bu dosya PAM tarafından, kullanıcı giriş yaptığında okunur. Servisler giriş yapmaz.

Denedik: `limits.d/` altına `nabizci - nofile 2048` yazdık.

| Nasıl başlatıldı? | `Max open files` (soft) |
|---|---|
| `su - nabizci` ile oturum açıp `ulimit -n` | 2048 |
| `panel.service` (`User=nabizci`) | 1024 (değişmedi) |

Servisin limiti unit dosyasında, `Limit...=` ile verilir:

```ini
[Service]
LimitNOFILE=4096
```
```bash
sudo systemctl daemon-reload && sudo systemctl restart panel
cat /proc/$(systemctl show -p MainPID --value panel)/limits | grep 'open files'
```
```
Max open files            4096                 4096                 files
```

| ulimit | systemd |
|---|---|
| `ulimit -n` (açık dosya) | `LimitNOFILE=` |
| `ulimit -u` (süreç sayısı) | `LimitNPROC=` |
| `ulimit -c` (core dosyası) | `LimitCORE=` |
| `ulimit -s` (yığın boyutu) | `LimitSTACK=` |

---

## 6. Kaynak limitleri: bellek, işlemci, süreç sayısı

systemd her servisi ayrı bir cgroup'ta çalıştırdığı için, servisin kullanabileceği toplam kaynağı sınırlayabilir. Bir servisin hatası veya aşırı yükü böylece bütün sunucuyu etkilemez.

| Ayar | Ne yapar? |
|---|---|
| `MemoryMax=200M` | Servisin bütün süreçleri en fazla 200 MB. Aşarsa çekirdek süreci öldürür |
| `MemoryHigh=150M` | Bu sınırı aşınca servis yavaşlatılır ve belleği geri alınır, öldürülmez |
| `CPUQuota=20%` | En fazla bir çekirdeğin %20'si. `200%` iki çekirdek demek |
| `TasksMax=50` | En fazla 50 süreç/thread |
| `IOWeight=` | Disk erişiminde öncelik (1–10000, varsayılan 100) |

### Denemeler

Ayarları denemek için `systemd-run` kullandık. Bu komut, bir unit dosyası yazmadan geçici bir servis başlatır ve `-p` ile ayar verilebilir.

**Bellek:** 50 MB sınırla 100 MB ayırmaya çalışan bir program:

```bash
sudo systemd-run --unit=bellek-test -p MemoryMax=50M -p MemorySwapMax=0 \
  python3 -c "a=bytearray(100*1024*1024); import time; time.sleep(30)"
```
```
bellek-test.service: Main process exited, code=killed, status=9/KILL
bellek-test.service: Failed with result 'signal'.
```

Sınırı aştığı an öldürüldü. Gerçek bir sunucuda `systemctl status` çıktısında genelde `Result: oom-kill` görünür ([OOM killer](14-performans-izleme.md#10-bellek-bittiğinde-oom-killer)).

**İşlemci:** Aynı anda iki sonsuz döngü başlattık, birine `CPUQuota=20%` verdik:

```bash
sudo systemd-run --unit=cpu-test -p CPUQuota=20% sh -c 'while :; do :; done'
sudo systemd-run --unit=cpu-serbest sh -c 'while :; do :; done'
pidstat -u -p PID1,PID2 3 1
```
```
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0      1814   20.00    0.00    0.00   79.67   20.00     -  sh
Average:        0      1816   88.00    0.00    0.00   11.33   88.00     -  sh
```

Sınırlı olan tam %20'de kaldı.

**Süreç sayısı:** `TasksMax=5` olan bir servis 10 alt süreç başlatmaya çalıştı:

```
bash[1825]: /usr/bin/bash: fork: retry: Resource temporarily unavailable
```
```
      Tasks: 5 (limit: 5)
```

`TasksMax`, bir hatanın sonsuz döngüde süreç oluşturup sunucuyu kilitlemesini (fork bomb) önler.

### Kalıcı ve geçici ayar

**Kalıcı:** drop-in dosyasına yaz.

```ini
[Service]
MemoryMax=200M
CPUQuota=50%
```

**Çalışan servise, yeniden başlatmadan:**

```bash
sudo systemctl set-property panel MemoryMax=200M              # kalıcı
sudo systemctl set-property --runtime panel MemoryMax=200M    # sadece yeniden açılışa kadar
```

`set-property` ayarı kendisi bir drop-in'e yazar (`50-MemoryMax.conf`) ve anında uygular:

```bash
systemctl status panel | grep Memory
```
```
     Memory: 8.0M (max: 200.0M available: 191.9M)
```

---

## 7. Dosya sistemi erişimini kısıtlamak

Servisin sadece gerçekten ihtiyaç duyduğu yerlere yazabilmesi, ele geçirilse bile sisteme zarar verememesi demektir. systemd bunun için hazır seçenekler sunar:

| Ayar | Ne yapar? |
|---|---|
| `ProtectSystem=strict` | Bütün dosya sistemi servis için salt okunur olur |
| `ReadWritePaths=/var/lib/panel` | Bu dizinler yine yazılabilir kalır |
| `ProtectHome=yes` | `/home`, `/root` görünmez |
| `PrivateTmp=yes` | Servis kendine ait boş bir `/tmp` görür |
| `NoNewPrivileges=yes` | Servis sudo, setuid gibi yollarla yetki yükseltemez |

Panele ekledik:

```ini
[Service]
ProtectSystem=strict
ReadWritePaths=/var/lib/panel
PrivateTmp=yes
```

Servisin içinden bakıldığında:

| Deneme | Sonuç |
|---|---|
| `/etc/deneme` dosyası oluşturmak | `Read-only file system` |
| `/var/lib/panel/ok` dosyası oluşturmak | Başarılı |
| `/tmp`'yi listelemek | Boş (sistemin `/tmp`'si görünmüyor) |

Bir servis `Read-only file system` hatası veriyorsa ama disk gerçekten salt okunur değilse, servisin `ProtectSystem` ayarına bak.

Bir servisin ne kadar korumalı olduğunu systemd puanlayabilir:

```bash
systemd-analyze security panel
```
```
→ Overall exposure level for panel.service: 8.7 EXPOSED :-(
```

0 en korumalı, 10 hiç korumasız. Çıktının üst kısmı hangi seçeneklerin eksik olduğunu tek tek listeler. Argümansız `systemd-analyze security` bütün servisleri puanlar.

---

## 8. Güvenlik modülleri: SELinux ve AppArmor

Dosya izinleri tamamken bile bir servis bir dosyaya erişemiyorsa, araya bir güvenlik modülü giriyor olabilir:

| | Nerede varsayılan? | Durumu görmek |
|---|---|---|
| AppArmor | Ubuntu, Debian | `sudo aa-status` |
| SELinux | RHEL, Fedora, Rocky | `getenforce`, `ls -Z`, `ps -eZ` |

İkisi de bir programın hangi dosyalara ve portlara erişebileceğini, normal izinlerin **üstünde** sınırlar. Reddedilen erişimler loglanır:

```bash
sudo journalctl | grep -i -E 'apparmor="DENIED"|avc:  denied'
```

AppArmor'da bir profili geçici olarak sadece loglayan moda almak: `sudo aa-complain /usr/sbin/nginx`. SELinux'un ayrıntılı kullanımı: [SELinux](../01-isletim-ve-dagitim/10-selinux.md).

> [!NOTE]
> Bu notun denendiği ortamda AppArmor araçları kurulu değildi, bu bölümdeki komutlar denenmedi.

---

## 9. Bağımlılıklar

### Servis bağımlılıkları

Bir servis başka bir servise ihtiyaç duyuyorsa (panelin veritabanına bağlanması gibi), unit dosyasında belirtilir ([13. dosya, [Unit] bölümü](13-servis-olusturma-ve-sorun-giderme.md#6-servis-dosyasının-bölümleri)):

```ini
[Unit]
Requires=postgresql.service
After=postgresql.service
```

Hangi servislerin neye bağlı olduğunu görmek:

```bash
systemctl list-dependencies panel             # panel neye ihtiyaç duyuyor
systemctl list-dependencies --reverse redis-server   # redis-server'a kim ihtiyaç duyuyor
```

Bir servisi durdurmadan önce `--reverse` ile ona bağlı olanlara bak.

### Paket ve kütüphane bağımlılıkları

```bash
apt-cache depends redis-server        # paketin bağımlılıkları
ldd /usr/bin/redis-server             # programın kullandığı kütüphaneler
```
```
	libatomic.so.1 => /lib/x86_64-linux-gnu/libatomic.so.1 (0x00007fe4e2ea1000)
	liblzf.so.1 => /lib/x86_64-linux-gnu/liblzf.so.1 (0x00007fe4e2bec000)
```

`ldd` çıktısında `not found` görünen bir kütüphane, programın neden açılmadığını açıklar. Paket yönetimi: [Paket Yönetimi](../01-isletim-ve-dagitim/06-paket-yonetimi.md).

### Uygulamanın kendi kısıtlamaları

Bazı kısıtlamaları uygulamanın kendisi koyar: bir veritabanının en fazla bağlantı sayısı, bir web sunucusunun en fazla çalışan sayısı gibi. Bunlar uygulamanın yapılandırma dosyasında (`/etc/uygulama/`) ve belgelerinde (`man uygulama`, `/usr/share/doc/uygulama/`) yazar. Sınavda bir uygulamanın limiti sorulursa ilk bakılacak yerler bunlardır.

---

## 10. Kontrol listesi

Bir servis çalışmıyor veya beklenmedik davranıyorsa:

| Soru | Komut |
|---|---|
| Hangi kullanıcıyla çalışıyor? | `systemctl show servis -p User` |
| Hata mesajı ne? | `journalctl -u servis -n 30` |
| `Permission denied`: dosya mı, port mu? | `ls -l dosya`, `ss -tlnp` |
| Port 1024'ün altında mı, başkası kullanıyor mu? | `ss -tlnp 'sport = :PORT'` |
| `Too many open files`? | `cat /proc/PID/limits`, `LimitNOFILE=` |
| Bellek sınırına takılıp öldürüldü mü? | `systemctl status`, `systemctl show -p MemoryMax` |
| `Read-only file system`? | `systemctl show -p ProtectSystem -p ReadWritePaths` |
| İzinler doğru ama yine reddediliyor mu? | `aa-status`, `getenforce`, journal'da `DENIED` |
| Bağlı olduğu servis çalışıyor mu? | `systemctl list-dependencies servis` |
| Kütüphanesi eksik mi? | `ldd /yol/program` |

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `systemctl show servis -p Ayar1 -p Ayar2` | Geçerli kısıtlamalar |
| `cat /proc/PID/limits` | Sürecin gerçek limitleri |
| `ss -tlnp` | Dinleyen portlar ve süreçler |
| `lsof -i :PORT` | Portu kim kullanıyor |
| `sysctl net.ipv4.ip_unprivileged_port_start` | Ayrıcalıksız portların başladığı numara |
| `AmbientCapabilities=CAP_NET_BIND_SERVICE` | Root olmadan düşük port açma yetkisi |
| `LimitNOFILE=` | Servisin açık dosya limiti |
| `MemoryMax=`, `MemoryHigh=` | Bellek sınırı |
| `CPUQuota=` | İşlemci sınırı |
| `TasksMax=` | Süreç sayısı sınırı |
| `systemctl set-property servis Ayar=Değer` | Çalışan servise sınır koy |
| `systemd-run -p Ayar=Değer komut` | Geçici, sınırlı servis |
| `ProtectSystem=strict`, `ReadWritePaths=` | Dosya sistemi erişimini kısıtla |
| `systemd-analyze security servis` | Güvenlik puanı |
| `aa-status`, `getenforce` | Güvenlik modülü durumu |
| `systemctl list-dependencies [--reverse] servis` | Servis bağımlılıkları |
| `ldd /yol/program` | Kütüphane bağımlılıkları |

---

## 12. Sık yapılan hatalar

- **Servisin limitini `/etc/security/limits.conf` ile değiştirmeye çalışmak.** Servislere uygulanmaz, `LimitNOFILE=` kullan.
- **Düşük port için servisi root yapmak.** `CAP_NET_BIND_SERVICE` veya ters proxy yeterli.
- **Her `Permission denied` hatasını dosya izni sanmak.** Port, SELinux/AppArmor ve `ProtectSystem` de aynı hatayı verebilir.
- **`systemctl cat`'e bakıp ayarın geçerli olduğunu sanmak.** Geçerli değer `systemctl show`'da.
- **Bellek sınırını servisin normal kullanımının altına koymak.** Servis sürekli öldürülür. Önce `systemctl status` ile normal kullanımına bak.
- **`ss -tlnp`'yi `sudo` olmadan çalıştırıp süreç sütununu boş görmek.**
- **Servisi durdurmadan önce ona bağlı servislere bakmamak.**

---

## 13. Alıştırmalar

**1.** `ssh` servisinin hangi kullanıcıyla çalıştığını, açık dosya limitini ve bellek sınırını bul. Hangi portu dinlediğini de göster.

<details>
<summary>Çözüm</summary>

```bash
systemctl show ssh -p User -p LimitNOFILE -p MemoryMax
sudo ss -tlnp | grep ssh
```

`User=` boşsa servis root olarak çalışıyordur. `MemoryMax=infinity` sınır yok demektir.
</details>

**2.** Normal bir kullanıcıyla `python3 -m http.server 80` çalıştır, hatayı gör. Sonra aynı komutu 8080 portuyla çalıştırıp başka bir terminalden hangi sürecin 8080'i dinlediğini bul.

<details>
<summary>Çözüm</summary>

```bash
python3 -m http.server 80        # PermissionError: [Errno 13] Permission denied
python3 -m http.server 8080 &
sudo ss -tlnp 'sport = :8080'
kill %1
```
</details>

**3.** `systemd-run` ile, en fazla 100 MB bellek ve %30 işlemci kullanabilen, sonsuz döngüde çalışan geçici bir servis başlat. Limitlerin uygulandığını kontrol et ve servisi durdur.

<details>
<summary>Çözüm</summary>

```bash
sudo systemd-run --unit=sinirli -p MemoryMax=100M -p CPUQuota=30% sh -c 'while :; do :; done'
systemctl show sinirli -p MemoryMax -p CPUQuotaPerSecUSec
top -bn1 | grep ' sh$'          # yaklaşık %30
sudo systemctl stop sinirli
```
</details>

**4.** [13. dosyadaki](13-servis-olusturma-ve-sorun-giderme.md#14-alıştırmalar) `saat.service`'e (veya kendi servislerinden birine) kalıcı olarak 4096 açık dosya limiti ver ve bunu çalışan sürecin `/proc` bilgisinden doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /etc/systemd/system/saat.service.d
printf '[Service]\nLimitNOFILE=4096\n' | sudo tee /etc/systemd/system/saat.service.d/limit.conf
sudo systemctl daemon-reload
sudo systemctl restart saat
grep 'open files' /proc/$(systemctl show -p MainPID --value saat)/limits
```
</details>

**5.** `redis-server` (veya sistemindeki başka bir servis) için hangi servislerin ona bağlı olduğunu ve programın hangi kütüphaneleri kullandığını bul. Servisin güvenlik puanına da bak.

<details>
<summary>Çözüm</summary>

```bash
systemctl list-dependencies --reverse redis-server
ldd /usr/bin/redis-server
systemd-analyze security redis-server
```
</details>

---

Önceki konu: [Disk Alanı Sorunları](15-disk-alani-sorunlari.md) · Bölüm ana sayfası: [Temel Komutlar](README.md)
