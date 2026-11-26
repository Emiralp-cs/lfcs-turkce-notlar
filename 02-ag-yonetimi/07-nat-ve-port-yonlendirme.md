# NAT ve Port Yönlendirme

> **Resmi sınav maddesi:** Configure packet filtering, port redirection, and NAT

Bir Linux sunucusu iki ağ arasında router gibi çalışabilir: bir arayüzü internete, diğeri iç ağa bakar. Bu durumda iki ihtiyaç ortaya çıkar:

- **Port yönlendirme (DNAT):** İnternetten sunucunun bir portuna gelen bağlantıyı iç ağdaki bir makineye aktarmak.
- **Masquerade (SNAT):** İç ağdaki makinelerin, sunucunun tek public adresini paylaşarak internete çıkması.

Ev modemleri tam olarak bu ikisini yapar.

Senaryo: Ekibin web uygulaması iç ağdaki `10.0.0.5` makinesinde çalışıyor. Bu makinenin internete açık bir adresi yok. Önünde iki arayüzlü bir ağ geçidi sunucusu var:

| Arayüz (denemedeki adı) | Adres | Nereye bakıyor? |
|---|---|---|
| `r-dis` (gerçek sistemde örn. `enp0s3`) | `203.0.113.10` | İnternet |
| `r-lan` (gerçek sistemde örn. `enp0s8`) | `10.0.0.1/24` | İç ağ |

Hedefler:

1. İnternetten `203.0.113.10:80`'e gelen istekler `10.0.0.5:80`'e gitsin.
2. İç ağdaki makineler internete çıkabilsin.

Dışarıdaki istemci olarak `203.0.113.99` adresli bir makine kullanıyoruz.

> [!NOTE]
> Bütün adımlar Ubuntu 24.04'te, ağ namespace'leriyle kurulan bir dış istemci, ağ geçidi ve iç web sunucusu düzeninde denenmiş, her kuraldan sonra `curl` ile gerçek bağlantı test edilmiştir. Deneme ortamı systemd ile çalışmadığı için `iptables-persistent` ve `nftables` servislerinin açılışta kuralları yüklemesi test edilemedi.

## İçindekiler

1. [Önce: IP yönlendirmeyi açmak](#1-önce-ip-yönlendirmeyi-açmak)
2. [netfilter, iptables ve nftables](#2-netfilter-iptables-ve-nftables)
3. [Port yönlendirme: DNAT](#3-port-yönlendirme-dnat)
4. [Masquerade ne zaman gerekir?](#4-masquerade-ne-zaman-gerekir)
5. [Kuralları görmek ve silmek](#5-kuralları-görmek-ve-silmek)
6. [Aynı kurallar nftables ile](#6-aynı-kurallar-nftables-ile)
7. [Kuralları kalıcı yapmak](#7-kuralları-kalıcı-yapmak)
8. [UFW ile NAT: before.rules ve route kuralları](#8-ufw-ile-nat-beforerules-ve-route-kuralları)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Önce: IP yönlendirmeyi açmak

Linux varsayılan olarak bir arayüzden gelip başka bir arayüze giden paketleri **iletmez**:

```bash
sysctl net.ipv4.ip_forward
```
```
net.ipv4.ip_forward = 0
```

Denemede DNAT kuralını yazdık ama yönlendirme kapalıyken dışarıdan bağlantı zaman aşımına uğradı. Açınca çalıştı:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

`sysctl -w` ile yapılan ayar geçicidir, yeniden başlatınca kaybolur. Kalıcı yapmak için `/etc/sysctl.d/` altına bir dosya yaz:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/90-yonlendirme.conf
sudo sysctl --system
```
```
* Applying /etc/sysctl.d/90-yonlendirme.conf ...
net.ipv4.ip_forward = 1
```

`sysctl --system` bütün ayar dosyalarını yeniden okur. Tek bir dosyayı okutmak için `sudo sysctl -p /etc/sysctl.d/90-yonlendirme.conf` da kullanılabilir.

`/etc/sysctl.conf`'taki `#net.ipv4.ip_forward=1` satırının başındaki `#`'yi kaldırmak da çalışır, ama kendi ayarlarını ayrı bir dosyada tutmak hangi ayarı neden değiştirdiğini takip etmeyi kolaylaştırır.

> [!NOTE]
> UFW kullanıyorsan aynı ayar `/etc/ufw/sysctl.conf` dosyasında da var ve UFW her açıldığında o dosyayı uygular. Orada biçim biraz farklı: `net/ipv4/ip_forward=1`. [8. bölümde](#8-ufw-ile-nat-beforerules-ve-route-kuralları) bu yolu kullanıyoruz.

---

## 2. netfilter, iptables ve nftables

Paketleri çekirdeğin içindeki **netfilter** işler. Kuralları yazmak için iki araç var:

- **iptables:** Eski ve hala çok yaygın araç.
- **nftables (`nft`):** Daha yeni araç ve sözdizimi.

Ubuntu 24.04'te `iptables` komutu aslında nftables'ın üzerinde çalışır:

```bash
iptables -V
```
```
iptables v1.8.10 (nf_tables)
```

Bu yüzden `iptables` ile yazdığın kurallar `sudo nft list ruleset` çıktısında da görünür.

NAT kuralları `nat` tablosunda, iki ayrı noktada çalışır:

| Zincir | Ne zaman? | Ne yapılır? |
|---|---|---|
| `PREROUTING` | Paket geldiği anda, nereye gideceğine karar verilmeden önce | Hedef adresi değiştirmek: **DNAT** |
| `POSTROUTING` | Paket çıkmak üzereyken, en son anda | Kaynak adresi değiştirmek: **SNAT / MASQUERADE** |

Kısaca: PREROUTING "nereye gidiyor"u, POSTROUTING "nereden geliyor"u değiştirir.

---

## 3. Port yönlendirme: DNAT

```bash
sudo iptables -t nat -A PREROUTING -i r-dis -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:80
```

| Parça | Anlamı |
|---|---|
| `-t nat` | NAT tablosu |
| `-A PREROUTING` | PREROUTING zincirinin sonuna ekle |
| `-i r-dis` | Sadece internet arayüzünden gelen paketler |
| `-p tcp --dport 80` | TCP, hedef port 80 |
| `-j DNAT --to-destination 10.0.0.5:80` | Hedefi `10.0.0.5:80` yap |

Belirli kaynaklarla sınırlamak istersen `-s 198.51.100.0/24` gibi bir kaynak filtresi de eklenebilir. Hedef port farklı olabilir: `--dport 8080 ... --to-destination 10.0.0.5:80` dışarıdaki 8080'i içerideki 80'e yönlendirir.

Dışarıdan test:

```bash
curl http://203.0.113.10/
```
```
web sunucusu, istemci: 203.0.113.99
```

Denemedeki web uygulaması, bağlanan istemcinin adresini yazıyordu. İstek iç sunucuya ulaştı ve iç sunucu gerçek istemci adresini (`203.0.113.99`) gördü.

**Bu çalışma için MASQUERADE kuralı gerekmedi.** Sebebi bir sonraki bölümde.

---

## 4. Masquerade ne zaman gerekir?

DNAT'tan sonra iç sunucunun cevabı, isteği gönderen `203.0.113.99`'a gitmeli. Bu cevabın **ağ geçidinden geri geçmesi** gerekir, çünkü ağ geçidi bağlantıyı takip eder ve cevabın kaynak adresini `10.0.0.5`'ten tekrar `203.0.113.10`'a çevirir.

İç sunucunun **varsayılan ağ geçidi** zaten bu sunucuysa (`default via 10.0.0.1`), cevap kendiliğinden buradan geçer. Masquerade gerekmez.

Bunu üç durumda denedik:

| Durum | Sonuç |
|---|---|
| İç sunucunun gateway'i ağ geçidi, masquerade yok | Çalıştı, iç sunucu gerçek istemci adresini gördü |
| İç sunucunun gateway'i yok, masquerade yok | Zaman aşımı: iç sunucu cevabı nereye göndereceğini bilmiyor |
| İç sunucunun gateway'i yok, iç arayüzde masquerade var | Çalıştı, ama iç sunucu istemciyi `10.0.0.1` olarak gördü |

Üçüncü durumdaki kural:

```bash
sudo iptables -t nat -A POSTROUTING -o r-lan -j MASQUERADE
```

Bu kural iç sunucuya giden paketlerin kaynağını ağ geçidinin iç adresi yapar. İç sunucu her zaman ağ geçidine cevap verir. Ama bedeli var: iç sunucu artık gerçek istemci adresini göremez, loglarında hep `10.0.0.1` görünür.

### Asıl masquerade: iç ağın internete çıkması

MASQUERADE'in en yaygın kullanımı port yönlendirme değil, **iç ağdaki makinelerin internete çıkmasıdır**. İç sunucudan dışarıdaki bir makineye bağlanmayı denedik:

```
Masquerade yok:   000   (zaman aşımı)
Masquerade var:   200
```

Masquerade olmadan paket dışarı çıkıyor ama kaynak adresi `10.0.0.5` kalıyor. Dışarıdaki makine bu özel adrese nasıl cevap vereceğini bilmiyor. Masquerade, çıkan paketlerin kaynağını ağ geçidinin public adresi yapar:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o r-dis -j MASQUERADE
```

| Parça | Anlamı |
|---|---|
| `-A POSTROUTING` | Paket çıkmak üzereyken |
| `-s 10.0.0.0/24` | Sadece iç ağdan gelen paketler |
| `-o r-dis` | Sadece internet arayüzünden çıkanlar |
| `-j MASQUERADE` | Kaynağı çıkış arayüzünün adresi yap |

`MASQUERADE`, çıkış arayüzünün adresini her seferinde kendisi bulur. Adres DHCP ile değişse bile çalışmaya devam eder. Adres sabitse aynı işi `-j SNAT --to-source 203.0.113.10` de yapar.

> [!IMPORTANT]
> Özet: Port yönlendirme için **DNAT** gerekir. **MASQUERADE** iç ağın internete çıkması için gerekir. Port yönlendirmede ise sadece iç sunucunun cevabı ağ geçidine dönmüyorsa (gateway'i başka bir yerse) gerekir. `man ufw-framework`'teki örnek de aynı şekilde ayrılmış: port yönlendirme örneğinde sadece DNAT var, masquerade ayrı bir örnek.

---

## 5. Kuralları görmek ve silmek

```bash
sudo iptables -t nat -L -n -v --line-numbers
```
```
Chain PREROUTING (policy ACCEPT 2 packets, 120 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        6   360 DNAT       6    --  r-dis  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:10.0.0.5:80

Chain POSTROUTING (policy ACCEPT 1 packets, 60 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        1    60 MASQUERADE  0    --  *      r-dis   10.0.0.0/24          0.0.0.0/0
```

`pkts` sütunu kurala uyan paket sayısıdır. Test sırasında sayı artmıyorsa paket bu kurala hiç ulaşmıyordur, bu sorun gidermede çok işe yarar.

`-t nat` yazmayı unutursan varsayılan `filter` tablosunu görürsün ve NAT kuralları listede çıkmaz.

Kuralları komut biçiminde görmek için:

```bash
sudo iptables-save -t nat
```
```
*nat
:PREROUTING ACCEPT [2:120]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [1:60]
-A PREROUTING -i r-dis -p tcp -m tcp --dport 80 -j DNAT --to-destination 10.0.0.5:80
-A POSTROUTING -s 10.0.0.0/24 -o r-dis -j MASQUERADE
COMMIT
```

Silmek için:

```bash
sudo iptables -t nat -D POSTROUTING 1     # numarayla
sudo iptables -t nat -D POSTROUTING -o r-lan -j MASQUERADE   # kuralın aynısını -D ile yazarak
sudo iptables -t nat -F                   # nat tablosundaki bütün kurallar
```

---

## 6. Aynı kurallar nftables ile

nftables'ta kurallar bir dosyaya yazılıp yüklenebilir:

```
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "r-dis" tcp dport 80 dnat to 10.0.0.5:80
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "r-dis" ip saddr 10.0.0.0/24 masquerade
    }
}
```

```bash
sudo nft -c -f nat.nft     # sadece sözdizimini kontrol et
sudo nft -f nat.nft        # yükle
sudo nft list table ip nat
```

Denemede iptables kurallarını silip bu dosyayı yükledik. Dışarıdan web uygulamasına ve iç ağdan dışarıya bağlantı aynı şekilde çalıştı.

Karşılaştırma:

| iptables | nftables |
|---|---|
| `-t nat -A PREROUTING` | `chain prerouting { type nat hook prerouting ... }` |
| `-i r-dis` / `-o r-dis` | `iifname "r-dis"` / `oifname "r-dis"` |
| `-s 10.0.0.0/24` | `ip saddr 10.0.0.0/24` |
| `-p tcp --dport 80` | `tcp dport 80` |
| `-j DNAT --to-destination X` | `dnat to X` |
| `-j MASQUERADE` | `masquerade` |

> [!WARNING]
> Ubuntu'da iptables kuralları da nftables altyapısında durduğu için `sudo nft list ruleset` çıktısında görünür, başında `# Warning: table ip nat is managed by iptables-nft, do not touch!` uyarısıyla. Aynı tabloyu iki araçla birlikte yönetme. Birini seç.

---

## 7. Kuralları kalıcı yapmak

`iptables` ve `nft` ile terminalde yazılan kurallar **yeniden başlatınca kaybolur**.

### iptables-persistent

```bash
sudo apt install iptables-persistent
```

Kurulum sırasında mevcut kuralları kaydetmek isteyip istemediğini sorar. Kurallar `/etc/iptables/rules.v4` (ve IPv6 için `rules.v6`) dosyasına `iptables-save` biçiminde kaydedilir ve açılışta yüklenir. Sonradan değişiklik yaparsan tekrar kaydet:

```bash
sudo netfilter-persistent save
```

### nftables

nftables kullanıyorsan kurallarını `/etc/nftables.conf` dosyasına yazıp servisi açarsın:

```bash
sudo systemctl enable --now nftables
```

`/etc/nftables.conf`'un başında `flush ruleset` satırı vardır: dosya yüklenince önce bütün kurallar silinir. Bu dosyaya sadece nftables kuralları yazılmalı.

İki yöntemden birini seç. iptables-persistent ve nftables'ı birlikte kullanma.

---

## 8. UFW ile NAT: before.rules ve route kuralları

Sunucuda UFW açıksa NAT kurallarını UFW'nin kendi dosyasına yazmak en temiz yol. UFW her açıldığında ve yeniden yüklendiğinde bu dosyayı okur, ayrıca kalıcılık paketi gerekmez.

Hazır örnek `man ufw-framework` içinde var. Man sayfasında `/Full example` yazıp Enter'a basınca doğrudan oraya atlarsın.

### 1. Yönlendirmeyi aç

`/etc/ufw/sysctl.conf` dosyasında satırın başındaki `#`'yi kaldır:

```
net/ipv4/ip_forward=1
```

### 2. NAT kurallarını before.rules'a ekle

Man sayfasındaki talimat: dosyanın **sonuna**, `*filter` bölümünün **ardından** ekle.

```bash
sudo vim /etc/ufw/before.rules
```

Dosyanın sonundaki `COMMIT` satırından sonra:

```
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -p tcp -i r-dis --dport 80 -j DNAT --to-destination 10.0.0.5:80
-A POSTROUTING -s 10.0.0.0/24 -o r-dis -j MASQUERADE
COMMIT
```

- Bu dosya `iptables-save` biçimindedir. Satırlar doğrudan `-A` ile başlar, başlarına `sudo iptables -t nat` **yazılmaz**.
- `*nat` hangi tabloya yazıldığını, `COMMIT` bölümün bittiğini gösterir. `COMMIT` unutulursa kurallar yüklenmez.

### 3. UFW'yi aç veya yeniden yükle

```bash
sudo ufw allow 22/tcp
sudo ufw enable        # zaten açıksa: sudo ufw reload
```

UFW açılınca `/etc/ufw/sysctl.conf`'u uyguladı ve yönlendirme kendiliğinden açıldı (`net.ipv4.ip_forward = 1`). Varsayılanlarda da bir değişiklik görünür:

```
Default: deny (incoming), allow (outgoing), deny (routed)
```

Yönlendirme kapalıyken `disabled (routed)` yazıyordu, şimdi `deny (routed)`: UFW yönlendirilen trafiği de filtreliyor ve varsayılan olarak engelliyor.

### 4. Yönlendirilen trafiğe izin ver

Denemede NAT kuralları yüklü olduğu halde hem dışarıdan web uygulamasına hem iç ağdan dışarıya bağlantı **zaman aşımına uğradı**. UFW'nin yönlendirme politikası engelliyordu. Bu trafik için `ufw route` kuralları gerekir:

```bash
sudo ufw route allow in on r-dis out on r-lan to 10.0.0.5 port 80 proto tcp
```

Dışarıdan web uygulaması çalıştı. İç ağın dışarı çıkışı ise hala kapalıydı, onun için ayrı bir kural gerekti:

```bash
sudo ufw route allow in on r-lan out on r-dis from 10.0.0.0/24
```

```bash
sudo ufw status numbered
```
```
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 10.0.0.5 80/tcp on r-lan   ALLOW FWD   Anywhere on r-dis
[ 3] Anywhere on r-dis          ALLOW FWD   10.0.0.0/24 on r-lan
```

`route` kurallarının `Action` sütununda `ALLOW FWD` yazar. Normal `allow` kuralları sunucunun **kendisine** gelen trafik içindir, sunucunun **üzerinden geçen** trafiği etkilemez.

> [!IMPORTANT]
> UFW açıkken NAT için üç şey birlikte gerekir: yönlendirmenin açık olması, `before.rules`'taki NAT kuralları ve her yön için `ufw route allow` kuralları. Biri eksikse bağlantı çalışmaz.

---

## 9. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `sysctl net.ipv4.ip_forward` | Yönlendirme açık mı? |
| `sysctl -w net.ipv4.ip_forward=1` | Geçici olarak açar |
| `/etc/sysctl.d/90-yonlendirme.conf` | Kalıcı ayar (`net.ipv4.ip_forward=1`) |
| `sysctl --system` | Bütün sysctl dosyalarını uygular |
| `iptables -t nat -A PREROUTING -i ARAYÜZ -p tcp --dport P -j DNAT --to-destination IP:P` | Port yönlendirme |
| `iptables -t nat -A POSTROUTING -s AĞ -o ARAYÜZ -j MASQUERADE` | İç ağın internete çıkması |
| `iptables -t nat -L -n -v --line-numbers` | NAT kuralları ve sayaçları |
| `iptables-save -t nat` | Kuralları komut biçiminde gösterir |
| `iptables -t nat -D ZİNCİR N` | Kural siler |
| `nft -f dosya` / `nft list ruleset` | nftables kurallarını yükler / gösterir |
| `apt install iptables-persistent` | iptables kurallarını kalıcı yapar |
| `netfilter-persistent save` | Mevcut kuralları kaydeder |
| `/etc/nftables.conf` | Kalıcı nftables kuralları |
| `/etc/ufw/sysctl.conf` | UFW'nin uyguladığı sysctl ayarları |
| `/etc/ufw/before.rules` | UFW ile NAT kuralları (`*filter`'dan sonra, `*nat` ... `COMMIT`) |
| `ufw reload` | before.rules değişikliğini uygular |
| `ufw route allow in on A out on B ...` | Yönlendirilen trafiğe izin verir |
| `man ufw-framework` | Hazır örnekler (`/Full example`) |

---

## 10. Sık yapılan hatalar

- **IP yönlendirmeyi açmamak.** NAT kuralları doğru olsa bile paketler iletilmez.
- **`sysctl -w` ile açıp kalıcı yapmamak.** Yeniden başlatınca kapanır.
- **DNAT'ın tek başına çalışmayacağını sanıp her zaman masquerade eklemek.** İç sunucunun gateway'i ağ geçidiyse gerekmez ve gereksiz masquerade iç sunucunun gerçek istemci adreslerini görmesini engeller.
- **İç ağın internete çıkması için masquerade'i unutmak.** Dışarıdaki makineler özel adreslere cevap veremez.
- **`-t nat` yazmadan listelemek.** NAT kuralları görünmez.
- **`before.rules`'a `sudo iptables -t nat` önekiyle yazmak.** Dosyada satırlar doğrudan `-A` ile başlar.
- **`before.rules`'ta `COMMIT`'i unutmak.** Bölüm yüklenmez.
- **UFW açıkken `ufw route allow`'u unutmak.** Yönlendirilen trafik varsayılan olarak engellenir.
- **Normal `ufw allow` kuralının yönlendirilen trafiği de kapsadığını sanmak.** Sunucunun üzerinden geçen trafik için `ufw route` gerekir.
- **Aynı kuralları iptables ve nftables ile birlikte yönetmek.** Birini seç.

---

## 11. Alıştırmalar

Aşağıdaki alıştırmalarda ağ geçidinin arayüzleri: internet `enp0s3`, iç ağ `enp0s8` (`192.168.100.0/24`).

**1.** IP yönlendirmeyi kalıcı olarak aç ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/90-yonlendirme.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward
```
</details>

**2.** Ağ geçidinin `2222` portuna internetten gelen bağlantılar iç ağdaki `192.168.100.20` makinesinin `22` portuna gitsin. iptables kullan.

<details>
<summary>Çözüm</summary>

```bash
sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 2222 -j DNAT --to-destination 192.168.100.20:22
sudo iptables -t nat -L PREROUTING -n -v
```

`192.168.100.20`'nin varsayılan ağ geçidi bu sunucu değilse, iç arayüzde masquerade de gerekir.
</details>

**3.** İç ağdaki bütün makineler internete çıkabilsin. Kuralları kalıcı yap.

<details>
<summary>Çözüm</summary>

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
</details>

**4.** 3. alıştırmayı iptables yerine UFW ile yap. UFW zaten açık ve SSH'a izin veriyor.

<details>
<summary>Çözüm</summary>

`/etc/ufw/sysctl.conf`:

```
net/ipv4/ip_forward=1
```

`/etc/ufw/before.rules` dosyasının sonuna:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.100.0/24 -o enp0s3 -j MASQUERADE
COMMIT
```

```bash
sudo ufw route allow in on enp0s8 out on enp0s3 from 192.168.100.0/24
sudo ufw reload
```
</details>

**5.** NAT kuralları yüklü, yönlendirme açık, ama dışarıdan iç web sunucusuna bağlanılamıyor. UFW açık. Neleri kontrol edersin?

<details>
<summary>Çözüm</summary>

```bash
sysctl net.ipv4.ip_forward                   # 1 mi?
sudo iptables -t nat -L PREROUTING -n -v     # DNAT kuralı var mı, sayaç artıyor mu?
sudo ufw status verbose                      # routed: deny mı?
sudo ufw status numbered                     # ALLOW FWD kuralı var mı?
```

En sık sebep `ufw route allow` kuralının eksik olması. Sayaç artıyor ama bağlantı yine olmuyorsa iç sunucunun varsayılan ağ geçidini kontrol et: cevaplar ağ geçidine dönüyor mu?
</details>

---

Önceki konu: [UFW ile Paket Filtreleme](06-ufw.md) · Sonraki konu: [Bridge ve Bonding](08-bridge-ve-bonding.md)
