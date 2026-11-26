# Ağ Yapılandırması: ip ve Netplan

> **İlgili resmi sınav maddeleri:** Configure IPv4 and IPv6 networking and hostname resolution · Configure static routing

Ubuntu'da ağ ayarı yapmanın iki katmanı var:

- **`ip` komutu:** Değişiklik anında uygulanır ama **geçicidir**. Yeniden başlatınca kaybolur. Bir şeyi hızlıca denemek ve sorun gidermek için kullanılır.
- **Netplan:** `/etc/netplan/` altındaki YAML dosyalarıdır. Her açılışta okunur, yani ayarlar **kalıcıdır**.

Senaryo: Ekibin sunucusuna ofis ağı için ikinci bir ağ kartı (`enp0s8`) takıldı. Kartın sabit bir adresi olacak ve yedekleme ağına (`192.168.50.0/24`) ayrı bir router üzerinden gidilecek:

| Ayar | Değer |
|---|---|
| Adres | `192.168.10.20/24` |
| Varsayılan ağ geçidi | `192.168.10.1` |
| DNS sunucuları | `192.168.10.1`, `1.1.1.1` |
| Yedekleme ağı | `192.168.50.0/24`, `192.168.10.254` üzerinden |

> [!NOTE]
> `ip` komutları Ubuntu 24.04'te bir ağ namespace'i içinde, `enp0s8` adlı sanal bir arayüz üzerinde denenmiştir. Netplan dosyaları `netplan generate` ve `netplan get` ile doğrulanmıştır. Deneme ortamı systemd ile çalışmadığı ve IPv6 desteği olmadığı için `netplan try/apply`, `networkctl`, `resolvectl` ve IPv6 komutları çalıştırılamadı. Bu kısımlar Netplan ve systemd belgelerine dayanıyor.

## İçindekiler

1. [Arayüzleri görmek](#1-arayüzleri-görmek)
2. [Arayüzü açmak ve kapatmak](#2-arayüzü-açmak-ve-kapatmak)
3. [Geçici IP adresi eklemek ve silmek](#3-geçici-ip-adresi-eklemek-ve-silmek)
4. [Geçici route eklemek](#4-geçici-route-eklemek)
5. [Netplan nasıl çalışır?](#5-netplan-nasıl-çalışır)
6. [Kalıcı statik IP yapılandırması](#6-kalıcı-statik-ip-yapılandırması)
7. [Değişikliği uygulamak: netplan try ve apply](#7-değişikliği-uygulamak-netplan-try-ve-apply)
8. [Netplan'da route ve DNS](#8-netplanda-route-ve-dns)
9. [Bütün arayüzler için DNS: resolved.conf](#9-bütün-arayüzler-için-dns-resolvedconf)
10. [Yerel isim çözümleme: /etc/hosts](#10-yerel-isim-çözümleme-etchosts)
11. [Yardım kaynakları](#11-yardım-kaynakları)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. Arayüzleri görmek

```bash
ip link          # arayüzler ve durumları
ip -br addr      # kısa özet: arayüz, durum, adresler
ip addr          # ayrıntılı (kısaltması: ip a)
```

Yeni takılan kartın ilk hali:

```bash
ip link
```
```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s8@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 02:9d:ba:81:2d:6b brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Denemede sanal bir arayüz kullandığımız için adın yanında `@if8` ve satır sonunda `link-netnsid 0` görünüyor. Gerçek bir ağ kartında sadece `enp0s8:` yazar.

- `lo` loopback arayüzü, makinenin kendisiyle konuşması için.
- `enp0s8` gibi isimler fiziksel (veya sanal makinedeki) ağ kartlarıdır. İsim, kartın bağlı olduğu yuvaya göre verilir.

Renkli çıktı için `-c` bayrağı kullanılır, ama **komutun ortasına**, `ip`'den hemen sonra yazılmalıdır:

```bash
ip -c addr     # doğru
ip addr -c     # Command "-c" is unknown, try "ip address help".
```

---

## 2. Arayüzü açmak ve kapatmak

```bash
sudo ip link set dev enp0s8 up
sudo ip link set dev enp0s8 down
```

| Parça | Anlamı |
|---|---|
| `link` | Arayüz seviyesinde işlem |
| `set` | Bir özelliği değiştir |
| `dev enp0s8` | Hangi arayüz |
| `up` / `down` | Aç / kapat |

### UP bayrağı ve durum farkı

Kartı `up` yaptıktan sonra çıktıda iki farklı bilgi görünür:

```
enp0s8@if8       DOWN           02:9d:ba:81:2d:6b <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

- `<...UP>` bayrağı: arayüzü **biz** açtık.
- `DOWN` durumu: ama arayüz çalışmıyor, çünkü `NO-CARRIER`, yani kabloda karşı taraf yok.

Karşı taraf bağlanınca durum `UP` oldu.

> [!IMPORTANT]
> Bir arayüzün `DOWN` görünmesinin sebebi IP adresi olmaması **değildir**. Denemede arayüzün bütün adreslerini sildik (`ip addr flush`), durum `UP` olarak kaldı. `DOWN` görüyorsan iki şeye bak: arayüz `up` yapılmış mı (bayraklarda `UP` var mı) ve `NO-CARRIER` var mı (kablo veya sanal bağlantı sorunu).

---

## 3. Geçici IP adresi eklemek ve silmek

```bash
sudo ip addr add 192.168.10.20/24 dev enp0s8
```

Bir arayüze birden fazla adres verilebilir:

```bash
sudo ip addr add 192.168.10.40/24 dev enp0s8
ip addr show enp0s8
```
```
    inet 192.168.10.20/24 scope global enp0s8
    inet 192.168.10.40/24 scope global secondary enp0s8
```

Aynı ağdaki ikinci adres `secondary` olarak işaretlenir.

IPv6 için de aynı komut kullanılır:

```bash
sudo ip addr add fd10:20::20/64 dev enp0s8
```

Silmek için:

```bash
sudo ip addr del 192.168.10.40/24 dev enp0s8   # tek bir adres
sudo ip addr flush dev enp0s8                   # arayüzdeki bütün adresler
```

`del` yazarken prefix'i (`/24`) de eklemelisin.

> [!WARNING]
> `ip` ile yapılan **her değişiklik geçicidir**. Sunucu yeniden başlayınca kaybolur. Kalıcı ayar için [Netplan](#6-kalıcı-statik-ip-yapılandırması) kullanılır.

> [!NOTE]
> IPv6 açık bir arayüz, `up` yapıldığında kendiliğinden bir `fe80::` link-local adresi alır. Bu adresleri elle vermene gerek yoktur.

---

## 4. Geçici route eklemek

Route tablosu, bir pakete hangi yoldan gideceğini söyler:

```bash
ip route
```
```
192.168.10.0/24 dev enp0s8 proto kernel scope link src 192.168.10.20
```

Bir adres eklediğinde, o adresin ağı için route'u çekirdek kendiliğinden ekler (`proto kernel`).

### Belirli bir ağa route

```bash
sudo ip route add 192.168.50.0/24 via 192.168.10.254
```

"`192.168.50.0/24` ağına giden paketleri `192.168.10.254` router'ına gönder" demek.

> [!IMPORTANT]
> `via` ile yazılan ağ geçidine doğrudan ulaşılabilmeli, yani bağlı olduğun ağlardan birinde olmalı. Denemede bağlı olmadığımız bir ağdaki geçidi yazınca hata aldık:
> ```
> Error: Nexthop has invalid gateway.
> ```

### Varsayılan route

Başka hiçbir route'a uymayan bütün trafik (internet dahil) varsayılan route'tan gider:

```bash
sudo ip route add default via 192.168.10.1
```

### Paket hangi yoldan gider?

`ip route get`, belirli bir adrese giden paketin hangi route'u kullanacağını gösterir. Denemedeki sonuçlar:

```
ip route get 192.168.50.7   ->  192.168.50.7 via 192.168.10.254 dev enp0s8 src 192.168.10.20
ip route get 8.8.8.8        ->  8.8.8.8 via 192.168.10.1 dev enp0s8 src 192.168.10.20
```

`192.168.50.7` kendi route'unu kullandı, `8.8.8.8` ise varsayılan route'a düştü.

### Birden fazla varsayılan route ve metric

Birden fazla varsayılan route olabilir. Hangisinin kullanılacağına `metric` karar verir, **küçük olan kazanır**:

```bash
sudo ip route add default via 192.168.10.2 metric 200
ip route
```
```
default via 192.168.10.1 dev enp0s8
default via 192.168.10.2 dev enp0s8 metric 200
192.168.10.0/24 dev enp0s8 proto kernel scope link src 192.168.10.20
192.168.50.0/24 via 192.168.10.254 dev enp0s8
```

`ip route get 1.1.1.1` metric'i yazılmayan (yani 0 olan) `192.168.10.1` route'unu seçti.

Route silmek için `add` yerine `del` yazılır:

```bash
sudo ip route del 192.168.50.0/24
```

Arayüz `down` yapıldığında, o arayüz üzerinden giden bütün route'lar tablodan kalkar.

---

## 5. Netplan nasıl çalışır?

Netplan, `/etc/netplan/` altındaki YAML dosyalarını okur ve asıl ağ servisinin anlayacağı yapılandırmaya çevirir. Sunucularda bu servis genelde `systemd-networkd`, masaüstünde NetworkManager'dır.

```bash
ls /etc/netplan/
```
```
50-cloud-init.yaml
```

Kurulumda oluşturulan dosya genelde böyledir:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

Bütün dosyaların birleşmiş halini görmek için:

```bash
sudo netplan get                              # hepsi
sudo netplan get ethernets.enp0s8.addresses   # tek bir ayar
```

---

## 6. Kalıcı statik IP yapılandırması

Mevcut dosyayı değiştirmek yerine kendi ayarlarımız için yeni bir dosya oluşturalım:

```bash
sudo vim /etc/netplan/99-ofis.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.10.20/24
```

İzinleri düzelt:

```bash
sudo chmod 600 /etc/netplan/99-ofis.yaml
```

İzinler açık kalırsa Netplan uyarır:

```
** (generate:910): WARNING **: 14:01:04.957: Permissions for /etc/netplan/99-ofis.yaml are too open. Netplan configuration should NOT be accessible by others.
```

### Dosya adı neden 99 ile başlıyor?

Netplan dosyaları **alfabetik sırayla** okur. Aynı ayar birden fazla dosyada varsa **sonra okunan kazanır**. Denedik: `50-cloud-init.yaml`'da `dhcp4: true`, kendi dosyamızda `dhcp4: false` vardı.

| Bizim dosyanın adı | Sonuç (`netplan get ethernets.enp0s8.dhcp4`) |
|---|---|
| `99-ofis.yaml` (50'den sonra okunur) | `false`, bizim ayarımız geçerli |
| `10-ofis.yaml` (50'den önce okunur) | `true`, cloud-init'in ayarı geçerli |

### YAML'da girinti

YAML'da girinti yapının kendisidir. Denediğimiz iki hata:

```
/etc/netplan/99-ofis.yaml:7:6: Invalid YAML: inconsistent indentation:
     addresses:
     ^
```

```
/etc/netplan/99-ofis.yaml:5:1: Invalid YAML: tabs are not allowed for indent:
	dhcp4: true
^
```

- Aynı seviyedeki satırlar aynı sayıda boşlukla başlamalı.
- **Tab kullanılamaz**, sadece boşluk.
- `-` ile başlayan satırlar liste elemanıdır (adresler, route'lar).

### Netplan arka planda ne üretiyor?

`netplan generate`, YAML'ı `systemd-networkd` dosyalarına çevirir ama ağa dokunmaz. Sözdizimini kontrol etmenin güvenli bir yolu:

```bash
sudo netplan generate
```

Bu dosyanın tam halinden ([8. bölümdeki](#8-netplanda-route-ve-dns) örnek) üretilen dosya:

```bash
cat /run/systemd/network/10-netplan-enp0s8.network
```
```
[Match]
Name=enp0s8

[Network]
LinkLocalAddressing=ipv6
Address=192.168.10.20/24
Address=fd10:20::20/64
DNS=192.168.10.1
DNS=1.1.1.1
Domains=ekip.local

[Route]
Destination=0.0.0.0/0
Gateway=192.168.10.1

[Route]
Destination=192.168.50.0/24
Gateway=192.168.10.254
```

YAML'daki her ayarın bir karşılığı var. Bunu görmek Netplan'ın ne yaptığını anlamayı kolaylaştırır.

---

## 7. Değişikliği uygulamak: netplan try ve apply

```bash
sudo netplan try
```

`netplan try` ayarları uygular ve bir geri sayım başlatır. Süre dolmadan Enter'a basıp onaylamazsan **eski ayarlara geri döner**. SSH ile bağlıyken yanlış bir ayar yüzünden sunucuya erişimi kaybetme riskine karşı en güvenli yöntem budur. Süreyi `--timeout` ile değiştirebilirsin.

```bash
sudo netplan apply
```

`netplan apply` ayarları onay beklemeden doğrudan uygular.

### apply çalışmazsa

Bazı ortamlarda `netplan try` veya `apply` izin hatası verebilir. Bu durumda adımları elle yapabilirsin:

```bash
sudo netplan generate                  # YAML'ı networkd dosyalarına çevir
sudo networkctl reload                 # networkd yeni dosyaları okusun
sudo networkctl reconfigure enp0s8     # sadece bu arayüzü yeniden yapılandır
ip -c addr show enp0s8                 # kontrol
```

Bu yolda otomatik geri alma yoktur.

---

## 8. Netplan'da route ve DNS

Senaryonun tamamı tek bir dosyada:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.10.20/24
        - fd10:20::20/64
      nameservers:
        search: [ekip.local]
        addresses: [192.168.10.1, 1.1.1.1]
      routes:
        - to: default
          via: 192.168.10.1
        - to: 192.168.50.0/24
          via: 192.168.10.254
```

| Blok | Ne işe yarar? |
|---|---|
| `addresses` | Arayüzün adresleri. IPv4 ve IPv6 birlikte yazılabilir |
| `nameservers.addresses` | Bu arayüz için DNS sunucuları |
| `nameservers.search` | Kısa isimlere eklenecek alan adı: `db01` yazınca `db01.ekip.local` da denenir |
| `routes: to: default` | Varsayılan ağ geçidi |
| `routes: to: 192.168.50.0/24` | Belirli bir ağ için route |

Bir route'a öncelik vermek için `metric` eklenebilir:

```yaml
      routes:
        - to: default
          via: 192.168.10.1
          metric: 100
```

Uyguladıktan sonra kontrol:

```bash
sudo netplan try
ip route
resolvectl status enp0s8
```

> [!NOTE]
> Sunucunun başka bir arayüzü DHCP'den varsayılan route alıyorsa, `ip route` çıktısında iki `default` satırı görmek normaldir. [4. bölümdeki](#birden-fazla-varsayılan-route-ve-metric) gibi metric'i küçük olan kullanılır.

---

## 9. Bütün arayüzler için DNS: resolved.conf

DNS ayarı iki yerde yapılabilir. Hangisini kullanacağın, ayarın kimi etkilemesi gerektiğine bağlı:

| İhtiyaç | Nereye yazılır? | Kontrol |
|---|---|---|
| Sadece bir arayüz için | Netplan'da o arayüzün `nameservers` bloğu | `resolvectl status enp0s8` |
| Bütün arayüzler için (global) | `/etc/systemd/resolved.conf` | `resolvectl status`, "Global" kısmı |

Global DNS için:

```bash
sudo vim /etc/systemd/resolved.conf
```

`[Resolve]` bölümündeki `#DNS=` satırının başındaki `#`'yi kaldır ve sunucuları **boşlukla** ayırarak yaz:

```ini
[Resolve]
DNS=1.1.1.1 9.9.9.9
```

Servisi yeniden başlat ve kontrol et:

```bash
sudo systemctl restart systemd-resolved
resolvectl status    # Global kısmında "DNS Servers" satırı
resolvectl dns       # sadece DNS adresleri, arayüz arayüz
```

> [!TIP]
> Sorudaki ifadeye dikkat et. "Bütün arayüzler için" veya "global" geçiyorsa `resolved.conf`, belirli bir arayüz adı geçiyorsa Netplan.

---

## 10. Yerel isim çözümleme: /etc/hosts

`/etc/hosts`, bir isme hangi IP'nin karşılık geldiğini **sadece bu makine için** tanımlar. Formatı:

```
IP_ADRESI   tam_ad   kisa_ad
```

Veritabanı sunucusunu adıyla çağırabilmek için:

```bash
echo "192.168.10.30  db01.ekip.local  db01" | sudo tee -a /etc/hosts
getent hosts db01
```
```
192.168.10.30   db01.ekip.local db01
```

Aynı satıra birden fazla isim yazılabilir, hepsi aynı adresi gösterir.

### /etc/hosts DNS'ten önce gelir

İsim çözümleme sırası `/etc/nsswitch.conf`'ta yazar:

```bash
grep '^hosts' /etc/nsswitch.conf
```
```
hosts:          files dns
```

Önce `files` (yani `/etc/hosts`), sonra DNS. Bu yüzden gerçek bir alan adını bile buraya yazarsan DNS'teki kaydın önüne geçer. Denemede `example.com`'u `/etc/hosts`'a `10.0.0.1` olarak yazdık ve `getent hosts example.com` bu adresi verdi. Bu, test ortamlarında bir siteyi geçici olarak başka bir sunucuya yönlendirmek için sık kullanılan bir yöntem.

> [!TIP]
> İsim çözümlemeyi test ederken `getent hosts isim` veya `ping isim` kullan. `dig` ve `nslookup` doğrudan DNS sunucusuna sorar ve `/etc/hosts`'a bakmaz. `/etc/hosts`'a eklediğin bir ismi `dig` ile bulamaman normaldir.

---

## 11. Yardım kaynakları

Sınavda YAML sözdizimini hatırlayamazsan iki kaynak var.

**Man sayfası:**

```bash
man netplan
```

İçinde `/kelime` yazıp Enter'a basınca arama yapar, `n` bir sonraki eşleşmeye, `q` çıkışa gider. Örneğin `/routes` ile route örneklerine atlayabilirsin.

**Hazır örnek dosyalar:**

```bash
ls /usr/share/doc/netplan/examples/
```

Bu dizinde `static.yaml`, `dhcp.yaml`, `static-routes.yaml`, `route_metric.yaml`, `bridge.yaml`, `bonding.yaml` gibi onlarca örnek var. Örneğin `static.yaml`:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      addresses:
        - 10.10.10.2/24
      nameservers:
        search: [mydomain, otherdomain]
        addresses: [10.10.10.1, 1.1.1.1]
      routes:
        - to: default
          via: 10.10.10.1
```

Uygun örneği `/etc/netplan/` altına kopyalayıp düzenlemek, sıfırdan yazmaktan daha az hataya açıktır:

```bash
sudo cp /usr/share/doc/netplan/examples/static.yaml /etc/netplan/99-ofis.yaml
sudo chmod 600 /etc/netplan/99-ofis.yaml
```

---

## 12. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `ip link` | Arayüzleri ve durumlarını listeler |
| `ip -br addr` / `ip addr` | Adresleri kısa / ayrıntılı gösterir |
| `ip -c addr` | Renkli çıktı (`-c` ortada) |
| `ip link set dev X up/down` | Arayüzü açar / kapatır |
| `ip addr add IP/PREFIX dev X` | Geçici adres ekler |
| `ip addr del IP/PREFIX dev X` | Adres siler |
| `ip addr flush dev X` | Arayüzün bütün adreslerini siler |
| `ip route` | Route tablosunu gösterir |
| `ip route add AG via GECIT` | Geçici route ekler |
| `ip route add default via GECIT` | Varsayılan route ekler |
| `ip route get ADRES` | Paketin hangi yoldan gideceğini gösterir |
| `/etc/netplan/*.yaml` | Kalıcı ayarlar, alfabetik sırayla okunur |
| `netplan get` | Birleşmiş Netplan ayarlarını gösterir |
| `netplan generate` | YAML'ı kontrol eder ve networkd dosyalarını üretir |
| `netplan try` | Uygular, onay gelmezse geri alır |
| `netplan apply` | Doğrudan uygular |
| `chmod 600 dosya.yaml` | Netplan izin uyarısını çözer |
| `/etc/systemd/resolved.conf` | Global DNS (`DNS=` satırı, boşlukla ayrılmış) |
| `resolvectl status [arayüz]` | DNS ayarlarını gösterir |
| `/etc/hosts` | Bu makineye özel isim-IP eşleşmeleri |
| `getent hosts isim` | İsmi nsswitch sırasına göre çözer |
| `/usr/share/doc/netplan/examples/` | Hazır Netplan örnekleri |

---

## 13. Sık yapılan hatalar

- **`ip` ile yapılan ayarın kalıcı olduğunu sanmak.** Yeniden başlatınca kaybolur.
- **`DOWN` durumunu IP eksikliğine bağlamak.** Arayüzün açık olup olmadığına ve `NO-CARRIER`'a bak.
- **`ip addr del`'de prefix'i unutmak.** `192.168.10.40/24` şeklinde yaz.
- **Ulaşılamayan bir ağ geçidi yazmak.** `via` adresi bağlı olduğun bir ağda olmalı.
- **YAML'da tab veya tutarsız girinti kullanmak.** Sadece boşluk, her seviye aynı.
- **Netplan dosyasının adını küçük numarayla başlatmak.** Önce okunan dosyanın ayarı, sonra okunanınkiyle ezilir.
- **Netplan dosyasının izinlerini açık bırakmak.** `chmod 600`.
- **SSH bağlantısı üzerinden `netplan apply` kullanmak.** Hata varsa erişimi kaybedersin. `netplan try` kullan.
- **`resolved.conf`'ta DNS sunucularını virgülle ayırmak.** Boşlukla ayrılır.
- **`/etc/hosts`'a eklenen ismi `dig` ile test etmek.** `dig` bu dosyaya bakmaz, `getent hosts` kullan.

---

## 14. Alıştırmalar

**1.** `enp0s8` arayüzüne geçici olarak `192.168.10.20/24` ve `192.168.10.21/24` adreslerini ver. Sonra sadece ikincisini sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ip link set dev enp0s8 up
sudo ip addr add 192.168.10.20/24 dev enp0s8
sudo ip addr add 192.168.10.21/24 dev enp0s8
ip -br addr show enp0s8
sudo ip addr del 192.168.10.21/24 dev enp0s8
ip -br addr show enp0s8
```
</details>

**2.** `192.168.50.0/24` ağına `192.168.10.254` üzerinden giden geçici bir route ekle. `192.168.50.7` adresine giden paketin bu route'u kullandığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo ip route add 192.168.50.0/24 via 192.168.10.254
ip route get 192.168.50.7
```

Çıktıda `via 192.168.10.254` görünmeli.
</details>

**3.** `enp0s8` için kalıcı bir Netplan dosyası yaz: adres `192.168.10.20/24`, varsayılan ağ geçidi `192.168.10.1`, DNS sunucuları `192.168.10.1` ve `9.9.9.9`. Dosyayı uygulamadan önce kontrol et.

<details>
<summary>Çözüm</summary>

`/etc/netplan/99-ofis.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.10.20/24
      nameservers:
        addresses: [192.168.10.1, 9.9.9.9]
      routes:
        - to: default
          via: 192.168.10.1
```

```bash
sudo chmod 600 /etc/netplan/99-ofis.yaml
sudo netplan generate         # hata vermemeli
sudo netplan get ethernets.enp0s8
sudo netplan try
```
</details>

**4.** Bu makinede `nas01` ismi `192.168.10.40` adresine gitsin. DNS'e dokunmadan yap ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo "192.168.10.40  nas01" | sudo tee -a /etc/hosts
getent hosts nas01
```
</details>

**5.** Aşağıdaki Netplan dosyasında üç hata var. Bul.

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
	  dhcp4: false
      addresses:
        192.168.10.20/24
      routes:
        - to: default
          via: 10.0.0.1
```

<details>
<summary>Çözüm</summary>

1. `dhcp4` satırı tab ile girintilenmiş. Sadece boşluk kullanılmalı.
2. `addresses` altındaki adres liste elemanı olmalı: `- 192.168.10.20/24`. Tire olmadan `netplan generate` şu hatayı verir: `Error in network definition: expected sequence`.
3. Ağ geçidi `10.0.0.1`, arayüzün ağında (`192.168.10.0/24`) değil. Ulaşılamayan bir geçit route olarak çalışmaz. Doğrusu `192.168.10.1` gibi aynı ağdan bir adres.
</details>

---

Önceki konu: [IP Adresleme](01-ip-adresleme.md) · Sonraki konu: [Saat Senkronizasyonu](03-saat-senkronizasyonu.md)
