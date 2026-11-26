# UFW ile Paket Filtreleme

> **Resmi sınav maddesi:** Configure packet filtering, port redirection, and NAT

Güvenlik duvarı (firewall), istenmeyen ağ trafiğinin sunucudaki servislere ulaşmasını engeller. Bir servis kendisine gelen paketi hiç görmezse, o servisteki bir güvenlik açığı da o paketle istismar edilemez.

İki tür güvenlik duvarı vardır:

| | Uygulama tabanlı | Paket filtreleme |
|---|---|---|
| Neye bakar? | Trafiği hangi programın ürettiğine | Paketin kaynak/hedef adresine, portuna, protokolüne |
| Örnek | Windows Güvenlik Duvarı'nda "bu programa izin ver" | Linux'ta iptables/nftables, UFW |

Linux'ta çekirdeğin içindeki paket filtresi **netfilter**'dır. iptables ve nftables bu filtreye kural yazan araçlardır. **UFW** (Uncomplicated Firewall) ise Ubuntu'da bu kuralları daha kolay yazmak için kullanılan bir arayüzdür: arka planda iptables kuralları üretir.

Senaryo: Ekibin sunucusunun iki adresi var, ofis ağında `192.168.10.20` ve internete açık `203.0.113.10`.

| Servis | Port | Kim erişebilmeli? |
|---|---|---|
| SSH | 22/tcp | Sadece ofis ağı (`192.168.10.0/24`) |
| Web uygulaması | 80/tcp | Sadece ofis ağı |
| Test servisi | 8080/tcp | Sadece Ayşe'nin bilgisayarı (`192.168.10.50`) |

Ofis ağındaki `192.168.10.37` adresli bir bilgisayardan şüpheli trafik geliyor, o tamamen engellenecek.

> [!NOTE]
> Bütün kurallar Ubuntu 24.04 ve UFW 0.36.2 ile, ağ namespace'leriyle kurulan bir sunucu, ofis ve dış ağ düzeninde denenmiş ve her kuraldan sonra bağlantılar gerçekten test edilmiştir. Deneme ortamında IPv6 desteği olmadığı için UFW'nin IPv6 kuralları kapalıydı. Gerçek bir sistemde her kuralın altında bir de `(v6)` satırı görürsün.

## İçindekiler

1. [UFW'nin varsayılanları](#1-ufwnin-varsayılanları)
2. [Önce SSH, sonra enable](#2-önce-ssh-sonra-enable)
3. [Kural yazmak: kaynak, hedef, port, protokol](#3-kural-yazmak-kaynak-hedef-port-protokol)
4. [Kural sırası: ilk eşleşen kazanır](#4-kural-sırası-ilk-eşleşen-kazanır)
5. [Kural silmek](#5-kural-silmek)
6. [deny ve reject farkı](#6-deny-ve-reject-farkı)
7. [Arayüze ve hedef adrese göre kural](#7-arayüze-ve-hedef-adrese-göre-kural)
8. [Giden trafik](#8-giden-trafik)
9. [Diğer kullanışlı özellikler](#9-diğer-kullanışlı-özellikler)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. UFW'nin varsayılanları

UFW kurulu gelir ama kapalıdır:

```bash
sudo ufw status
```
```
Status: inactive
```

Açıldığında varsayılan davranış şudur:

| Yön | Varsayılan |
|---|---|
| Gelen (incoming) | **deny**: izin verilmeyen her şey engellenir |
| Giden (outgoing) | **allow**: sunucu dışarıya istediği yere bağlanabilir |
| Yönlendirilen (routed) | **disabled**: sunucu router gibi davranmaz ([NAT](07-nat-ve-port-yonlendirme.md) dosyasında) |

Yani UFW **beyaz liste** mantığıyla çalışır: gelen trafikte sadece açıkça izin verdiğin şeyler geçer.

---

## 2. Önce SSH, sonra enable

UFW'yi açtığın anda gelen her şey engellenir, SSH dahil. Sunucuya SSH ile bağlıysan önce SSH'a izin vermelisin:

```bash
sudo ufw allow 22
sudo ufw enable
```
```
Rules updated
Firewall is active and enabled on system startup
```

`enable`, güvenlik duvarını hemen açar ve açılışta da çalışacak şekilde ayarlar. Etkileşimli bir terminalde mevcut SSH bağlantılarının etkilenebileceğine dair bir onay sorusu gelir.

```bash
sudo ufw status verbose
```
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22                         ALLOW IN    Anywhere
```

Denemede `enable`'dan önce dış ağdan web portuna (80) bağlanılabiliyordu. Sonra:

| Kaynak | 22 | 80 |
|---|---|---|
| Ofis | Açık | Kapalı |
| Dış ağ | Açık | Kapalı |

80. port için kural yazmadığımız için kapandı.

> [!NOTE]
> Port numarasının yanına protokol yazılmazsa kural **hem TCP hem UDP** için geçerli olur. Arka planda iki ayrı iptables kuralı oluşur:
> ```
> -A ufw-user-input -p tcp -m tcp --dport 22 -j ACCEPT
> -A ufw-user-input -p udp -m udp --dport 22 -j ACCEPT
> ```
> SSH sadece TCP kullandığı için `sudo ufw allow 22/tcp` yazmak daha doğru.

> [!TIP]
> Bazı paketler UFW'ye hazır uygulama profilleri ekler: `sudo ufw app list`. Ubuntu'da `openssh-server` kuruluysa `OpenSSH` profili vardır ve `sudo ufw allow OpenSSH` 22/tcp'ye izin verir. Profilin hangi portları açtığını görmek için `sudo ufw app info OpenSSH`.

---

## 3. Kural yazmak: kaynak, hedef, port, protokol

Genel biçim:

```
ufw allow|deny|reject [in|out] [on ARAYÜZ] [from KAYNAK] [to HEDEF] [port PORT] [proto tcp|udp]
```

SSH'ı sadece ofis ağına açalım:

```bash
sudo ufw allow from 192.168.10.0/24 to any port 22 proto tcp
```

| Parça | Anlamı |
|---|---|
| `from 192.168.10.0/24` | Kaynak: bu ağdan gelen paketler |
| `to any` | Hedef: sunucunun herhangi bir adresi |
| `port 22` | Hedef port |
| `proto tcp` | Sadece TCP |

`from` tek bir adres de olabilir: `from 192.168.10.50`.

Web uygulamasını da ofise açalım:

```bash
sudo ufw allow from 192.168.10.0/24 to any port 80 proto tcp
```

---

## 4. Kural sırası: ilk eşleşen kazanır

UFW gelen bir paketi kurallarla **yukarıdan aşağıya** karşılaştırır ve **ilk eşleşen** kuralı uygular. Kalan kurallara bakılmaz. Kuralları numaralarıyla görmek için:

```bash
sudo ufw status numbered
```

### Sorun 1: Genel kural kısıtlamayı etkisiz bırakıyor

2. bölümdeki `allow 22` hala duruyorken ofis kuralını ekledik:

```
     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    Anywhere
[ 2] 22/tcp                     ALLOW IN    192.168.10.0/24
```

Dış ağdan SSH denedik: **açık**. Bütün paketler 1. kurala uyuyor, 2. kurala hiç sıra gelmiyor. Kısıtlamanın çalışması için genel kuralı silmek gerekiyor ([5. bölüm](#5-kural-silmek)). Sildikten sonra dış ağdan SSH kapandı, ofisten açık kaldı.

### Sorun 2: deny kuralı sona eklendi

Şüpheli bilgisayarı engellemek için:

```bash
sudo ufw deny from 192.168.10.37
sudo ufw status numbered
```
```
[ 1] 22/tcp                     ALLOW IN    192.168.10.0/24
[ 2] 80/tcp                     ALLOW IN    192.168.10.0/24
[ 3] Anywhere                   DENY IN     192.168.10.37
```

`192.168.10.37`'den SSH ve web denedik: **ikisi de açık**. `.37`, `192.168.10.0/24` ağının içinde olduğu için 1. ve 2. kurallara takılıp izin aldı. 3. kurala hiç gelmedi.

`ufw allow` ve `ufw deny` kuralları **listenin sonuna** ekler. Bir kuralın daha önce değerlendirilmesini istiyorsan `insert` ile istediğin sıraya koyarsın:

```bash
sudo ufw delete deny from 192.168.10.37
sudo ufw insert 1 deny from 192.168.10.37
sudo ufw status numbered
```
```
[ 1] Anywhere                   DENY IN     192.168.10.37
[ 2] 22/tcp                     ALLOW IN    192.168.10.0/24
[ 3] 80/tcp                     ALLOW IN    192.168.10.0/24
```

Artık `.37` engellendi, `.50` gibi diğer ofis bilgisayarları hala erişebiliyor.

> [!IMPORTANT]
> Kural yazarken şunu kendine sor: "Bu kuraldan daha genel bir kural, bu kuraldan **önce** mi geliyor?" Geliyorsa yazdığın kural hiç çalışmaz. Özel kurallar (tek bir IP'yi engellemek gibi) genel kuralların **üstünde** olmalı.

---

## 5. Kural silmek

İki yöntem var:

**Numarayla:**

```bash
sudo ufw status numbered
sudo ufw delete 1
```
```
Deleting:
 allow 22
Proceed with operation (y|n)? y
Rule deleted
```

Onay sorulur. Script'lerde soruyu atlamak için `sudo ufw --force delete 1`.

**Kuralın kendisini yazarak:**

```bash
sudo ufw delete allow 22
```

Kuralı eklerken yazdığın şeyin başına `delete` koyarsın. Var olmayan bir kuralı silmeye çalışırsan:

```
Could not delete non-existent rule
```

> [!WARNING]
> Bir kuralı sildiğinde altındaki kuralların numaraları değişir. Arka arkaya birden fazla kural silerken her silmeden önce `ufw status numbered`'a tekrar bak, yoksa yanlış kuralı silebilirsin.

---

## 6. deny ve reject farkı

İkisi de trafiği engeller ama karşı tarafa farklı davranır:

| | `deny` | `reject` |
|---|---|---|
| Ne yapar? | Paketi sessizce düşürür | Paketi reddeder ve karşı tarafa bildirir |
| İstemci ne görür? | Cevap gelmez, zaman aşımına kadar bekler | Hemen "Connection refused" |

`.37`'yi önce `deny`, sonra `reject` ile engelleyip 3 saniye zaman aşımıyla bağlanmayı denedik:

```
deny:    nc: connect to 192.168.10.20 port 80 (tcp) timed out: Operation now in progress    (3.008 sn)
reject:  nc: connect to 192.168.10.20 port 80 (tcp) failed: Connection refused              (0.004 sn)
```

Varsayılan gelen politikası da `deny` olduğu için, izin verilmeyen bir porta dışarıdan bağlanmaya çalışan da zaman aşımı yaşar.

`deny` saldırganlara daha az bilgi verir, bu yüzden internete açık arayüzlerde genelde tercih edilir. `reject` iç ağlarda, kullanıcıların uzun süre beklemeden hatayı görmesi için kullanışlıdır.

---

## 7. Arayüze ve hedef adrese göre kural

Test servisine (8080) sadece Ayşe'nin bilgisayarı, sadece ofis arayüzü üzerinden ulaşabilsin:

```bash
sudo ufw allow in on s-ofis from 192.168.10.50 to 192.168.10.20 port 8080 proto tcp
```

| Parça | Anlamı |
|---|---|
| `in on s-ofis` | Sadece `s-ofis` arayüzünden gelen paketler (gerçek sistemde `enp0s8` gibi) |
| `from 192.168.10.50` | Sadece Ayşe'nin bilgisayarı |
| `to 192.168.10.20` | Sadece sunucunun ofis adresine gelen paketler |

Denemede `.50` bağlanabildi, `.37` bağlanamadı. Liste şöyle görünüyor:

```
[ 4] 192.168.10.20 8080/tcp on s-ofis ALLOW IN    192.168.10.50
```

`to any` yerine belirli bir adres yazmak, servisin sunucunun internete açık adresinden ulaşılmasını da engeller.

---

## 8. Giden trafik

Kurallar varsayılan olarak gelen trafik içindir. Giden trafik için `out` yazılır. Sunucunun dış ağdaki `203.0.113.99` adresine bağlanmasını engelleyelim:

```bash
ping -c 2 203.0.113.99              # önce: 2 received
sudo ufw deny out on s-dis to 203.0.113.99
ping -c 2 203.0.113.99              # sonra: 0 received, 100% packet loss
```

Diğer adreslere giden trafik etkilenmedi.

```
[ 3] 203.0.113.99               DENY OUT    Anywhere on s-dis          (out)
```

### from ve to yöne göre anlam değiştirir

| Yön | `from` | `to` |
|---|---|---|
| Gelen (`in`) | Dışarıdaki kaynak | Bu sunucu |
| Giden (`out`) | Bu sunucu | Dışarıdaki hedef |

Kural yazmadan önce "paket sunucuya mı geliyor, sunucudan mı çıkıyor?" diye sor.

### Cevap paketleri için ayrı kural gerekmez

UFW bağlantıları takip eder. Gelen bir bağlantıya izin verdiysen, o bağlantının **cevap paketleri** ayrı bir kurala gerek kalmadan dışarı çıkar. Denemede giden trafiğin varsayılanını da `deny` yaptık:

```bash
sudo ufw default deny outgoing
```

Ofisten web uygulamasına bağlantı **çalışmaya devam etti** ve sayfa geldi. Ama sunucunun kendisi dışarıya yeni bir bağlantı (örneğin `ping`) başlatamadı.

Yani gelen bir servise izin verirken "cevaplar dönebilsin diye" bir de `out` kuralı yazmana gerek yok. `out` kuralları sadece sunucunun **kendi başlattığı** bağlantılar içindir.

> [!CAUTION]
> `default deny outgoing` sunucunun DNS sorgusu, güncelleme ve NTP gibi bütün dış bağlantılarını keser. Kullanmak istiyorsan önce gereken her giden trafik için `allow out` kuralı yaz. Denemeden sonra varsayılanı geri aldık: `sudo ufw default allow outgoing`.

---

## 9. Diğer kullanışlı özellikler

### Kaba kuvvet denemelerine karşı: limit

```bash
sudo ufw insert 1 limit 22/tcp
```

`limit`, aynı IP'den 30 saniye içinde 6 veya daha fazla bağlantı denemesini engeller. Denemede arka arkaya 8 bağlantı açtık:

```
1:ok 2:ok 3:ok 4:ok 5:ok 6:ENGEL 7:ENGEL 8:ENGEL
```

Bu kuralın da sırası önemli: önce sona eklemiştik ve `22/tcp ALLOW IN 192.168.10.0/24` kuralının altında kaldığı için hiç çalışmadı. `insert 1` ile en üste koyunca çalıştı.

### Yazdığın kuralları komut olarak görmek

```bash
sudo ufw show added
```
```
Added user rules (see 'ufw status' for running firewall):
ufw allow from 192.168.10.0/24 to any port 22 proto tcp
ufw allow from 192.168.10.0/24 to any port 80 proto tcp
ufw deny out on s-dis to 203.0.113.99
ufw allow in on s-ofis from 192.168.10.50 to 192.168.10.20 port 8080 proto tcp
```

Bir kuralı silmek için tam olarak ne yazman gerektiğini buradan kopyalayabilirsin.

### Kuralların saklandığı yer

| Dosya | İçerik |
|---|---|
| `/etc/default/ufw` | Varsayılan politikalar, IPv6 açık mı (`IPV6=yes`) |
| `/etc/ufw/user.rules` | Senin yazdığın IPv4 kuralları (`user6.rules` IPv6 için) |
| `/etc/ufw/before.rules` | Senin kurallarından önce işlenen kurallar (NAT burada) |
| `/etc/ufw/sysctl.conf` | UFW'nin uyguladığı çekirdek ayarları |

`user.rules` içinde her kuralın UFW karşılığı ve ürettiği iptables satırı görünür:

```
### tuple ### allow tcp 22 0.0.0.0/0 any 192.168.10.0/24 in
-A ufw-user-input -p tcp --dport 22 -s 192.168.10.0/24 -j ACCEPT
```

Bu dosyaları elle düzenlemek yerine `ufw` komutunu kullan.

### Kapatmak ve sıfırlamak

```bash
sudo ufw disable    # kapatır, kurallar durur
sudo ufw reset      # kapatır ve bütün kuralları siler
```

`reset` silmeden önce dosyaların yedeğini alır:

```
Backing up 'user.rules' to '/etc/ufw/user.rules.20261125_142722'
```

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `ufw status` / `status verbose` | Durum / varsayılanlarla birlikte ayrıntılı durum |
| `ufw status numbered` | Kuralları numaralı listeler |
| `ufw enable` / `disable` | Açar / kapatır |
| `ufw allow 22/tcp` | Bir porta herkesten izin verir |
| `ufw allow from AĞ to any port P proto tcp` | Belirli kaynaktan belirli porta izin verir |
| `ufw deny from IP` | Bir adresi engeller (sona eklenir) |
| `ufw insert N KURAL` | Kuralı N. sıraya ekler |
| `ufw reject ...` | Engeller ve karşı tarafa bildirir |
| `ufw allow in on ARAYÜZ from A to B port P proto tcp` | Arayüz ve hedef adresle sınırlı kural |
| `ufw deny out on ARAYÜZ to IP` | Giden trafiği engeller |
| `ufw delete N` | N numaralı kuralı siler (onay ister) |
| `ufw delete KURAL` | Kuralı içeriğiyle siler |
| `ufw limit 22/tcp` | Çok sık bağlantı denemelerini engeller |
| `ufw default deny incoming` | Varsayılan gelen politikası |
| `ufw app list` / `app info AD` | Uygulama profilleri |
| `ufw show added` | Eklenen kuralları komut olarak gösterir |
| `ufw reset` | Bütün kuralları siler (yedek alarak) |
| `ufw --help` | Bütün seçenekler |

---

## 11. Sık yapılan hatalar

- **SSH kuralı olmadan `ufw enable`.** Uzaktan bağlıysan dışarıda kalırsın.
- **Genel `allow` kuralını silmeyi unutmak.** Sonradan eklenen kısıtlayıcı kural hiç çalışmaz.
- **Özel bir `deny` kuralını `insert` olmadan eklemek.** Sona eklenir ve üstteki daha geniş `allow` kuralları onu etkisiz bırakır.
- **Protokol yazmamak.** `allow 22` hem TCP hem UDP açar.
- **Arka arkaya numarayla silerken listeyi yenilememek.** Numaralar her silmede kayar.
- **Gelen ve giden kurallarda `from`/`to`'yu karıştırmak.** Önce paketin yönünü belirle.
- **Cevap paketleri için `out` kuralı yazmak.** Gerekmez, UFW bağlantıları takip eder.
- **`default deny outgoing`'i hazırlıksız açmak.** DNS ve güncellemeler dahil her şey kesilir.
- **SSH portunu değiştirip güvenlik duvarını unutmak.** Önce yeni portu aç, sonra SSH'ı değiştir.

---

## 12. Alıştırmalar

**1.** UFW kapalı bir sunucuda SSH'ı (sadece TCP) açık bırakarak güvenlik duvarını etkinleştir ve varsayılan politikaları doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

`Default: deny (incoming), allow (outgoing), disabled (routed)` görmelisin.
</details>

**2.** SSH sadece `10.20.30.0/24` ağından erişilebilir olsun. 1. alıştırmadaki genel kuralı kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw allow from 10.20.30.0/24 to any port 22 proto tcp
sudo ufw delete allow 22/tcp
sudo ufw status numbered
```

Kısıtlayıcı kuralı **önce** ekle, genel kuralı **sonra** sil. Tersini yaparsan arada SSH'ın kapalı kaldığı bir an olur.
</details>

**3.** `10.20.30.0/24` ağında izin var ama `10.20.30.66` adresi tamamen engellensin. Kuralın gerçekten çalışacağından emin ol.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw insert 1 deny from 10.20.30.66
sudo ufw status numbered
```

`deny` kuralının, ağın tamamına izin veren kuralın **üstünde** olduğunu kontrol et.
</details>

**4.** Aşağıdaki listede `10.20.30.66` engellenmiyor. Neden? Nasıl düzeltirsin?

```
[ 1] 22/tcp                     ALLOW IN    10.20.30.0/24
[ 2] Anywhere                   DENY IN     10.20.30.66
```

<details>
<summary>Çözüm</summary>

`10.20.30.66`, `10.20.30.0/24` ağının içinde. Paketler 1. kurala uyuyor ve izin alıyor, 2. kurala hiç bakılmıyor.

```bash
sudo ufw delete 2
sudo ufw insert 1 deny from 10.20.30.66
```
</details>

**5.** Sunucu `enp0s3` arayüzünden `1.1.1.1` adresine bağlanamasın. Diğer giden trafik etkilenmesin. Sonra kuralı sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw deny out on enp0s3 to 1.1.1.1
ping -c 2 1.1.1.1          # cevap gelmemeli
ping -c 2 8.8.8.8          # çalışmalı
sudo ufw delete deny out on enp0s3 to 1.1.1.1
```
</details>

---

Önceki konu: [SSH](05-ssh.md) · Sonraki konu: [NAT ve Port Yönlendirme](07-nat-ve-port-yonlendirme.md)
