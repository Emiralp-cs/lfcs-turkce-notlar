# Ağ İzleme ve Sorun Giderme

> **Resmi sınav maddesi:** Monitor and troubleshoot networking

"Siteye bağlanamıyorum" şikayeti geldiğinde sorun birçok yerde olabilir: servis çalışmıyordur, yanlış adresi dinliyordur, güvenlik duvarı engelliyordur, route yoktur ya da isim çözümlenmiyordur. Bu dosyada sunucuda neyin hangi portu dinlediğini görmeyi, servisleri yönetmeyi ve bağlantı sorununu adım adım daraltmayı ele alıyoruz.

Senaryo: Ekibin sunucusunda (`192.168.10.20`) iki servis çalışıyor:

| Servis | Dinlediği adres | Kim erişebilmeli? |
|---|---|---|
| Web uygulaması (`webapp` kullanıcısıyla) | `0.0.0.0:8080` | Ofisteki herkes |
| Veritabanı | `127.0.0.1:5432` | Sadece sunucunun kendisi |

Ofisteki bir bilgisayar (`192.168.10.50`) bu servislere bağlanmaya çalışacak.

> [!NOTE]
> `ss`, `ps`, `lsof`, `netstat`, `ping`, `nc`, `curl`, `tracepath` ve `traceroute` Ubuntu 24.04'te, ağ namespace'leriyle kurulan bir sunucu, istemci ve router düzeninde denenmiştir. Deneme ortamı systemd ile çalışmadığı için `systemctl` komutları çalıştırılamadı. O kısım systemd belgelerine dayanıyor ve örnek çıktı içermiyor.

## İçindekiler

1. ["Dinlemek" ne demek?](#1-dinlemek-ne-demek)
2. [ss ile dinleyen portları görmek](#2-ss-ile-dinleyen-portları-görmek)
3. [ss çıktısını okumak](#3-ss-çıktısını-okumak)
4. [Portu kullanan süreci incelemek: ps ve lsof](#4-portu-kullanan-süreci-incelemek-ps-ve-lsof)
5. [netstat](#5-netstat)
6. [Servisleri yönetmek: systemctl](#6-servisleri-yönetmek-systemctl)
7. [Bağlantıyı test etmek: ping, nc ve curl](#7-bağlantıyı-test-etmek-ping-nc-ve-curl)
8. [Yolu izlemek: tracepath ve traceroute](#8-yolu-izlemek-tracepath-ve-traceroute)
9. [Adım adım sorun giderme](#9-adım-adım-sorun-giderme)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. "Dinlemek" ne demek?

Bir servis çalışınca bir port açar ve o porta gelecek bağlantıları beklemeye başlar. Buna **dinlemek** (listen) denir. Web uygulaması 8080'i dinliyorsa, bu sunucunun 8080 portuna gelen her bağlantı işletim sistemi tarafından ona verilir.

Bir servisin dinlediği şey sadece port değil, **adres + port** ikilisidir. Bu ayrım sorun gidermede çok önemli, 3. bölümde göreceğiz.

---

## 2. ss ile dinleyen portları görmek

```bash
sudo ss -ltunp
```

Bayraklar:

| Bayrak | Anlamı |
|---|---|
| `-l` | Sadece dinleyen (listening) soketler |
| `-t` | TCP |
| `-u` | UDP |
| `-n` | Port ve adresleri sayı olarak göster, isme çevirme |
| `-p` | Soketi kullanan süreci göster |

Senaryodaki sunucuda çıktı (ilgili satırlar):

```
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
udp   UNCONN 0      0            0.0.0.0:5353       0.0.0.0:*    users:(("python3",pid=232,fd=3))
tcp   LISTEN 0      5            0.0.0.0:8080       0.0.0.0:*    users:(("python3",pid=230,fd=3))
tcp   LISTEN 0      1          127.0.0.1:5432       0.0.0.0:*    users:(("nc",pid=231,fd=3))
```

UDP'de bağlantı kavramı olmadığı için durum `LISTEN` değil `UNCONN` görünür.

### -n olmadan

`-n` yazmazsan port numaraları `/etc/services`'teki isimlere çevrilir:

```bash
ss -lt
```
```
LISTEN 0      5            0.0.0.0:http-alt        0.0.0.0:*
LISTEN 0      1          127.0.0.1:postgresql      0.0.0.0:*
```

Port numarası ararken bu kafa karıştırır. Her zaman `-n` kullanmak alışkanlık olsun.

### -p için sudo gerekir

Normal kullanıcı olarak çalıştırınca başka kullanıcıların süreçleri görünmez:

```
LISTEN 0      1          127.0.0.1:5432      0.0.0.0:*
```

Satır var ama `Process` sütunu boş. Hangi sürecin portu tuttuğunu görmek için `sudo` şart.

### Tek bir porta bakmak

```bash
sudo ss -ltnp 'sport = :8080'
sudo ss -ltnp | grep ':8080'
```

İkisi de çalışır. İlki `ss`'in kendi filtresi, ikincisi `grep` ile süzme.

Kurulan bağlantıları görmek için `-l`'yi kaldır. İstemci bağlıyken:

```bash
sudo ss -tnp 'sport = :8080'
```
```
State Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
ESTAB 0      0      192.168.10.20:8080 192.168.10.50:56488 users:(("python3",pid=230,fd=4))
```

`ESTAB` kurulmuş bir bağlantı, `Peer Address` karşı taraf.

---

## 3. ss çıktısını okumak

En önemli sütun `Local Address:Port`, yani servisin **hangi adreste** dinlediği:

| Local Address | Anlamı | Senaryoda |
|---|---|---|
| `127.0.0.1:5432` | Sadece sunucunun kendisinden gelen bağlantılar | Veritabanı, dışarıya kapalı |
| `0.0.0.0:8080` | Bütün IPv4 arayüzlerinden gelen bağlantılar | Web uygulaması, ofise açık |
| `[::]:22` | Bütün IPv6 arayüzleri | |
| `192.168.10.20:8080` | Sadece bu adrese gelen bağlantılar | |

Denemede istemciden iki servise de bağlanmayı denedik:

```
nc -zv 192.168.10.20 8080  ->  Connection to 192.168.10.20 8080 port [tcp/http-alt] succeeded!
nc -zv 192.168.10.20 5432  ->  nc: connect to 192.168.10.20 port 5432 (tcp) failed: Connection refused
```

Sunucunun kendisinden ise veritabanına bağlanılabildi:

```
nc -zv 127.0.0.1 5432      ->  Connection to 127.0.0.1 5432 port [tcp/postgresql] succeeded!
```

Servis çalışıyor, port açık ama dışarıdan bağlanılamıyor. Sebep güvenlik duvarı değil, servisin sadece `127.0.0.1`'i dinlemesi. Dışarıya açılması gerekiyorsa servisin kendi ayarında dinleme adresi değiştirilir.

Bir servisin hem `0.0.0.0:22` hem `[::]:22` satırı olması normaldir: IPv4 ve IPv6'yı birlikte dinliyordur.

---

## 4. Portu kullanan süreci incelemek: ps ve lsof

`ss` çıktısında süreç numarası (`pid=230`) var. Bu numarayla sürecin ne olduğuna bakabiliriz:

```bash
ps -fp 230
```
```
UID        PID  PPID  C STIME TTY          TIME CMD
webapp     230   229  0 14:15 ?        00:00:00 python3 -m http.server 8080 --bind 0.0.0.0
```

Süreci hangi kullanıcının çalıştırdığını ve tam komut satırını gösteriyor. Sadece istediğin sütunlar için:

```bash
ps -p 230 -o pid,user,cmd
```

### lsof

`lsof` (list open files) bir sürecin açtığı her şeyi gösterir: dosyalar, kütüphaneler, soketler.

```bash
sudo lsof -p 230
```
```
COMMAND PID   USER   FD   TYPE DEVICE SIZE/OFF     NODE NAME
python3 230 webapp  cwd    DIR  254,0     4096   804080 /tmp/site
python3 230 webapp  rtd    DIR  254,0     4096        2 /
python3 230 webapp  txt    REG  254,0 31150928      200 /usr/local/bin/python3.13
...
```

`cwd` sürecin çalıştığı dizin, `txt` çalışan programın kendisi.

`lsof`'u ters yönde de kullanabilirsin: "bu portu kim tutuyor?"

```bash
sudo lsof -i :8080
sudo lsof -i :5432 -P -n
```
```
COMMAND PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3 230 webapp    3u  IPv4   2141      0t0  TCP *:http-alt (LISTEN)

COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      231 root    3u  IPv4    439      0t0  TCP 127.0.0.1:5432 (LISTEN)
```

`-P` port numaralarını, `-n` adresleri isme çevirmeden gösterir. Normal kullanıcı olarak çalıştırınca başkasının sürecini hiç göstermez, `sudo` gerekir.

Aynı soruyu `fuser` da cevaplar:

```bash
sudo fuser -v -n tcp 8080
```
```
                     USER        PID ACCESS COMMAND
8080/tcp:            webapp      230 F.... python3
```

Sorun giderirken tipik akış: `ss` ile portu ve PID'i bul, `ps` ile sürecin ne olduğunu gör, gerekirse `lsof` ile neleri açık tuttuğuna bak.

---

## 5. netstat

`netstat`, `ss`'ten önce kullanılan eski araçtır. Bayrakları büyük ölçüde aynıdır:

```bash
sudo netstat -ltunp
```
```
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      230/python3
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      231/nc
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           232/python3
```

Modern dağıtımlarda varsayılan olarak kurulu gelmeyebilir. Ubuntu'da `net-tools` paketiyle gelir. Sınavda önce `ss`'i kullan, `netstat` her sistemde olmayabilir.

---

## 6. Servisleri yönetmek: systemctl

Bir porta bakmadan önce servisin çalışıp çalışmadığını kontrol etmek gerekir:

```bash
systemctl status ssh
```

Çıktıda bakılacak satırlar:

| Satır | Ne söyler? |
|---|---|
| `Loaded:` | Servis dosyasının yeri ve `enabled`/`disabled`, yani açılışta başlayıp başlamayacağı |
| `Active:` | `active (running)`, `inactive (dead)` veya `failed` |
| `Main PID:` | Servisin ana süreci. `ss` çıktısındaki `pid=` ile karşılaştırarak hangi servisin hangi portu tuttuğunu doğrulayabilirsin |
| Son satırlar | Servisin son log satırları. Hata mesajları genelde burada |

Kısa cevap isteyen sorular için:

```bash
systemctl is-active ssh     # active / inactive
systemctl is-enabled ssh    # enabled / disabled
```

### Başlatmak, durdurmak, açılışa eklemek

| Komut | Etkisi |
|---|---|
| `sudo systemctl start servis` | Şimdi başlat |
| `sudo systemctl stop servis` | Şimdi durdur |
| `sudo systemctl restart servis` | Durdurup yeniden başlat |
| `sudo systemctl reload servis` | Bağlantıları koparmadan ayarları yeniden oku (servis destekliyorsa) |
| `sudo systemctl enable servis` | Açılışta başlasın |
| `sudo systemctl disable servis` | Açılışta başlamasın |
| `sudo systemctl enable --now servis` | Açılışa ekle ve şimdi başlat |
| `sudo systemctl disable --now servis` | Açılıştan çıkar ve şimdi durdur |

> [!IMPORTANT]
> `stop` ile `disable` farklı şeylerdir. `stop` servisi şimdi durdurur ama servis `enabled` ise yeniden başlatınca geri gelir. `disable` ise servisi o an durdurmaz, sadece bir sonraki açılışta başlamasını engeller. "Servis çalışmasın ve açılışta da başlamasın" deniyorsa ikisi birden gerekir: `disable --now`.

Durdurduktan sonra portun gerçekten kapandığını `ss` ile kontrol et. Denemede web uygulamasının sürecini durdurduğumuzda `ss -ltn 'sport = :8080'` boş döndü ve istemcinin bağlanma girişimi `Connection refused` aldı.

### Servis adları dağıtıma göre değişir

| Servis | Ubuntu/Debian | RHEL ailesi |
|---|---|---|
| SSH | `ssh` | `sshd` |
| Web sunucusu (Apache) | `apache2` | `httpd` |
| Saat senkronizasyonu (chrony) | `chrony` | `chronyd` |

Yanlış ad yazınca `Unit ... not found` hatası alırsın. Adı bilmiyorsan `systemctl list-units --type=service | grep ssh` ile ara.

> [!NOTE]
> Ubuntu 22.10 ve sonrasında SSH varsayılan olarak `ssh.socket` ile başlatılır: 22. portu önce systemd dinler, bağlantı gelince `ssh.service`'i çalıştırır. Bu yüzden `ss -ltnp` çıktısında 22. portun sahibi `systemd` görünebilir. SSH'ı tamamen kapatmak için `ssh.socket`'i de durdurman gerekir.

---

## 7. Bağlantıyı test etmek: ping, nc ve curl

### ping: makineye ulaşılıyor mu?

```bash
ping -c 3 192.168.10.20
```
```
64 bytes from 192.168.10.20: icmp_seq=1 ttl=64 time=0.062 ms
...
3 packets transmitted, 3 received, 0% packet loss, time 2025ms
```

`-c` kaç paket gönderileceğini belirler. Yazmazsan `Ctrl+C`'ye basana kadar devam eder.

Var olmayan bir adrese:

```
2 packets transmitted, 0 received, 100% packet loss, time 1012ms
```

Hiç route olmayan bir ağa ise paket hiç gönderilmez:

```
ping: connect: Network is unreachable
```

> [!TIP]
> `ping` cevap vermiyorsa makine kapalı demek değildir. Birçok sunucu ve güvenlik duvarı ICMP'yi engeller. `ping` çalışıyorsa ağ yolu var demektir, ama bu servisin çalıştığını göstermez.

### nc: port açık mı?

`nc` (netcat) bir porta TCP bağlantısı kurmayı dener:

```bash
nc -zv -w 3 192.168.10.20 8080
```

`-z` veri göndermeden sadece bağlanmayı dener, `-v` sonucu yazar, `-w 3` en fazla 3 saniye bekler.

Üç farklı sonuç var ve her biri farklı bir şey söylüyor:

| Sonuç | Anlamı |
|---|---|
| `succeeded!` | Port açık, bir servis dinliyor |
| `Connection refused` | Makineye ulaşıldı ama o portu dinleyen yok, ya da güvenlik duvarı `REJECT` ile reddediyor |
| `timed out` | Cevap gelmedi: güvenlik duvarı paketi sessizce düşürüyor (`DROP`) veya makineye ulaşılamıyor |

Denemede aynı porta önce `DROP`, sonra `REJECT` kuralı koyduk:

```
DROP:    nc: connect to 192.168.10.20 port 8080 (tcp) timed out: Operation now in progress
REJECT:  nc: connect to 192.168.10.20 port 8080 (tcp) failed: Connection refused
```

`timed out` görüyorsan önce güvenlik duvarından şüphelen.

### curl: uygulama cevap veriyor mu?

Web servisleri için `curl` bir adım daha ileri gider ve gerçek bir istek gönderir:

```bash
curl -s http://192.168.10.20:8080/
```
```
merhaba
```

Servis kapalıyken `-v` ile ayrıntı:

```
* connect to 192.168.10.20 port 8080 from 192.168.10.50 port 41434 failed: Connection refused
* Failed to connect to 192.168.10.20 port 8080 after 0 ms: Couldn't connect to server
```

### İsim çözümlemeyi test etmek

Bağlantıda isim kullanıyorsan önce ismin doğru adrese çözüldüğünü kontrol et:

```bash
getent hosts db01        # /etc/hosts ve DNS, sistemin kullandığı sırayla
dig +short db01.ekip.local   # sadece DNS
```

Denemede `db01`'i `/etc/hosts`'a ekledik: `getent` adresi buldu, `dig` hiçbir şey döndürmedi. Çünkü `dig` doğrudan DNS sunucusuna sorar ve `/etc/hosts`'a bakmaz ([Ağ Yapılandırması](02-ag-yapilandirmasi.md#10-yerel-isim-çözümleme-etchosts) dosyasında anlattık).

---

## 8. Yolu izlemek: tracepath ve traceroute

Başka bir ağdaki makineye ulaşılamıyorsa, paketin nerede kaldığını görmek gerekir. Denemede sunucu, yedekleme ağındaki `192.168.50.10` makinesine `192.168.10.1` router'ı üzerinden gidiyordu:

```bash
tracepath -n 192.168.50.10
```
```
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.10.1                                          0.039ms
 1:  192.168.10.1                                          0.008ms
 2:  192.168.50.10                                         0.013ms reached
     Resume: pmtu 1500 hops 2 back 2
```

`traceroute` aynı işi yapar, Ubuntu'da ayrıca kurulması gerekebilir (`sudo apt install traceroute`). `tracepath` için root yetkisi gerekmez.

```bash
traceroute -n 192.168.50.10
```
```
traceroute to 192.168.50.10 (192.168.50.10), 30 hops max, 60 byte packets
 1  192.168.10.1  0.017 ms  0.003 ms  0.008 ms
 2  192.168.50.10  0.008 ms  0.003 ms  0.002 ms
```

Router'da yönlendirmeyi engelleyip tekrar denedik:

```
 1  192.168.10.1  0.017 ms
 2  *
 3  *
 4  *
```

Paket router'a kadar gitti ama sonrasında kayboldu. Sorun router'da ya da ondan sonraki bir noktada.

Paketin hangi route'u kullanacağını görmek için de `ip route get` var:

```bash
ip route get 192.168.50.10
```
```
192.168.50.10 via 192.168.10.1 dev vlab src 192.168.10.20 uid 0
```

### Arayüz istatistikleri

Bir arayüzde hata veya düşen paket olup olmadığını görmek için:

```bash
ip -s link show eth0
```
```
    RX:  bytes packets errors dropped  missed   mcast
          1935      27      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
          1423      17      0       0       0       0
```

`errors` ve `dropped` sütunlarında artan sayılar kablo, kart veya sürücü sorununa işaret edebilir.

---

## 9. Adım adım sorun giderme

"İstemci sunucudaki servise bağlanamıyor" durumunda, sunucudan istemciye doğru şu sırayla ilerle:

| # | Soru | Komut |
|---|---|---|
| 1 | Servis çalışıyor mu? | `systemctl status servis` |
| 2 | Port dinleniyor mu? | `sudo ss -ltnp 'sport = :PORT'` |
| 3 | Doğru adreste mi dinliyor? | `ss` çıktısında `127.0.0.1` mi `0.0.0.0` mı? |
| 4 | Sunucunun kendisinden bağlanılıyor mu? | `nc -zv 127.0.0.1 PORT` veya `curl` |
| 5 | Güvenlik duvarı izin veriyor mu? | `sudo ufw status` (sonraki dosyalarda) |
| 6 | İstemciden makineye ulaşılıyor mu? | `ping`, `ip route get` |
| 7 | İstemciden port açık mı? | `nc -zv SUNUCU PORT`: refused mı, timed out mı? |
| 8 | İsim doğru çözülüyor mu? | `getent hosts isim` |
| 9 | Başka ağdaysa yol nerede kesiliyor? | `tracepath -n hedef` |

Senaryodaki veritabanı sorunu 3. adımda çözülüyordu: servis çalışıyor, port açık, ama sadece `127.0.0.1`'i dinliyor.

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sudo ss -ltunp` | Dinleyen TCP/UDP portları ve süreçleri |
| `sudo ss -ltnp 'sport = :PORT'` | Tek bir portu gösterir |
| `sudo ss -tnp` | Kurulmuş TCP bağlantıları |
| `ss -s` | Soket sayılarının özeti |
| `sudo netstat -ltunp` | `ss`'in eski karşılığı (`net-tools` paketi) |
| `ps -fp PID` | Sürecin kullanıcısını ve komut satırını gösterir |
| `sudo lsof -p PID` | Sürecin açık dosya ve soketleri |
| `sudo lsof -i :PORT` | Portu kullanan süreç |
| `sudo fuser -v -n tcp PORT` | Portu kullanan süreç |
| `systemctl status servis` | Servisin durumu ve Main PID'i |
| `systemctl is-active` / `is-enabled servis` | Kısa durum bilgisi |
| `systemctl start` / `stop` / `restart servis` | Şimdi başlat / durdur / yeniden başlat |
| `systemctl enable` / `disable servis` | Açılışta başlasın / başlamasın |
| `systemctl enable --now` / `disable --now` | İkisini birlikte yapar |
| `ping -c 3 adres` | Makineye ulaşılıyor mu? |
| `nc -zv -w 3 adres port` | Port açık mı? |
| `curl -v http://adres:port/` | Web servisi cevap veriyor mu? |
| `getent hosts isim` | İsim çözümleme (hosts + DNS) |
| `dig +short isim` | Sadece DNS sorgusu |
| `tracepath -n` / `traceroute -n adres` | Paketin geçtiği router'lar |
| `ip route get adres` | Paketin kullanacağı route |
| `ip -s link show arayüz` | Arayüz istatistikleri ve hatalar |

---

## 11. Sık yapılan hatalar

- **`ss`'i `sudo`'suz çalıştırıp süreç bilgisini aramak.** Başka kullanıcıların süreçleri görünmez.
- **`-n` kullanmadan port numarası aramak.** Portlar isme çevrilir (`8080` yerine `http-alt`).
- **Port açık diye servisin dışarıdan erişilebilir olduğunu sanmak.** `127.0.0.1`'i dinleyen servise dışarıdan bağlanılamaz.
- **`stop` ile `disable`'ı karıştırmak.** `stop` geçici, `disable` sadece açılışı etkiler.
- **Servis adını dağıtıma göre değiştirmemek.** Ubuntu'da `ssh`, RHEL'de `sshd`.
- **`ping` cevap vermiyor diye makinenin kapalı olduğuna karar vermek.** ICMP engellenmiş olabilir. Portu `nc` ile dene.
- **`refused` ile `timed out`'u aynı sanmak.** İlki "dinleyen yok veya reddedildi", ikincisi genelde "güvenlik duvarı düşürüyor".
- **`/etc/hosts`'taki ismi `dig` ile test etmek.** `getent hosts` kullan.

---

## 12. Alıştırmalar

**1.** Sunucuda TCP 22. portu hangi sürecin dinlediğini bul. Sürecin hangi kullanıcıyla ve hangi komut satırıyla çalıştığını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo ss -ltnp 'sport = :22'
ps -fp PID        # ss çıktısındaki pid= değeri
```

Ubuntu'da süreç `systemd` olarak görünüyorsa SSH socket activation ile çalışıyordur.
</details>

**2.** Sunucuda sadece yerel bağlantıları kabul eden (dışarıdan erişilemeyen) bütün TCP portlarını listele.

<details>
<summary>Çözüm</summary>

```bash
sudo ss -ltnp | grep '127.0.0.1'
```

IPv6 yerel adresleri de dahil etmek için `grep -E '127\.0\.0\.1|\[::1\]'`.
</details>

**3.** `apache2` servisini durdur ve açılışta da başlamamasını sağla. Sonra 80. portun artık dinlenmediğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl disable --now apache2
systemctl is-active apache2      # inactive
systemctl is-enabled apache2     # disabled
sudo ss -ltn 'sport = :80'       # boş
```
</details>

**4.** Bir istemciden `nc -zv 192.168.10.20 8080` çalıştırdın ve `timed out` aldın. Sunucuda `ss` 8080'in `0.0.0.0` üzerinde dinlendiğini gösteriyor. Sorun büyük ihtimalle nerede?

<details>
<summary>Çözüm</summary>

Servis çalışıyor ve her adresi dinliyor. `timed out`, paketlerin cevapsız kaldığını gösteriyor. En olası sebep sunucuda veya aradaki bir cihazda paketi `DROP` eden bir güvenlik duvarı kuralı. `sudo ufw status` veya `sudo iptables -L -n` ile kontrol et.
</details>

**5.** `192.168.50.10` adresine ulaşılamıyor. Paketin hangi route'u kullandığını ve yolda nerede kaybolduğunu bul.

<details>
<summary>Çözüm</summary>

```bash
ip route get 192.168.50.10
tracepath -n 192.168.50.10
```

`tracepath` çıktısında `*` görünmeye başladığı yerden önceki son adres, paketin ulaşabildiği son noktadır.
</details>

---

Önceki konu: [Saat Senkronizasyonu](03-saat-senkronizasyonu.md) · Sonraki konu: [SSH](05-ssh.md)
