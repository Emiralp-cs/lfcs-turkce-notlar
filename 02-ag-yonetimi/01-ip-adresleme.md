# IP Adresleme: IPv4, IPv6 ve CIDR

Ağ ayarı yapmadan önce bir IP adresinin nasıl okunduğunu bilmek gerekir: adresin hangi kısmı ağı, hangi kısmı cihazı gösteriyor, iki makine aynı ağda mı, bir ağa kaç cihaz sığar. Bu dosya teori ağırlıklı. Öğrendiklerimizi sonraki dosyada Netplan ile sunucuya uygulayacağız.

Örneklerde ekibimizin sunucusu ofis ağında `192.168.10.20` adresini kullanıyor.

> [!NOTE]
> Hesaplar Python'un `ipaddress` modülü ve `ipcalc` aracıyla doğrulanmıştır.

## İçindekiler

1. [IPv4 adresinin yapısı](#1-ipv4-adresinin-yapısı)
2. [CIDR: ağ kısmı ve cihaz kısmı](#2-cidr-ağ-kısmı-ve-cihaz-kısmı)
3. [Ağ adresi, broadcast ve kullanılabilir adresler](#3-ağ-adresi-broadcast-ve-kullanılabilir-adresler)
4. [Prefix ve netmask tablosu](#4-prefix-ve-netmask-tablosu)
5. [Oktet sınırına denk gelmeyen prefix'ler](#5-oktet-sınırına-denk-gelmeyen-prefixler)
6. [Özel ve özel amaçlı IPv4 adresleri](#6-özel-ve-özel-amaçlı-ipv4-adresleri)
7. [IPv6 adresinin yapısı](#7-ipv6-adresinin-yapısı)
8. [IPv6 kısaltma kuralları](#8-ipv6-kısaltma-kuralları)
9. [IPv6'da prefix ve özel adresler](#9-ipv6da-prefix-ve-özel-adresler)
10. [Sunucuda adresleri görmek](#10-sunucuda-adresleri-görmek)
11. [Özet](#11-özet)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. IPv4 adresinin yapısı

Bir IPv4 adresi noktalarla ayrılmış dört sayıdan oluşur. Her sayıya **oktet** denir:

```
192.168.10.20
```

Her oktet 8 bittir. 8 bitle 2⁸ = 256 farklı değer yazılabildiği için bir oktet 0 ile 255 arasında olur:

```
00000000 = 0
11111111 = 255
```

Adresin tamamı 4 × 8 = **32 bit** eder. Aynı adresin ikili (binary) hali:

```
192      .168      .10       .20
11000000.10101000.00001010.00010100
```

32 bit en fazla yaklaşık 4,3 milyar adres demek. İnternete bağlı cihaz sayısı için bu yetmediğinden IPv6 geliştirildi.

---

## 2. CIDR: ağ kısmı ve cihaz kısmı

Bir IP adresi tek başına eksik bilgidir. Adresin kaç bitinin **ağı**, kaç bitinin o ağdaki **cihazı** (host) gösterdiğini de bilmek gerekir. CIDR gösteriminde bu, adresin sonuna `/` ile yazılan sayıdır (prefix):

```
192.168.10.20/24
```

`/24`, ilk 24 bitin ağı gösterdiği anlamına gelir. Kalan 8 bit cihazı gösterir:

| Oktet | 192 | 168 | 10 | 20 |
|---|---|---|---|---|
| Rolü | ağ | ağ | ağ | cihaz |

### İki adres aynı ağda mı?

İki adresin **ağ kısmı aynıysa** aynı ağdadırlar ve aralarında router olmadan doğrudan konuşabilirler.

| Adres | `/24` ile ağ kısmı | Sunucuyla aynı ağda mı? |
|---|---|---|
| `192.168.10.20` (sunucu) | `192.168.10` | |
| `192.168.10.35` | `192.168.10` | Evet |
| `192.168.11.20` | `192.168.11` | Hayır, 3. oktet farklı |

### Prefix değişince kapsam değişir

Aynı sunucu adresi `/16` ile yazılsaydı ağ kısmı sadece ilk iki oktet olurdu:

```
192.168.10.20/16
11000000.10101000 . 00001010.00010100
|---- ağ (16) ----|   |-- cihaz (16) --|
```

Bu durumda `192.168.0.0` ile `192.168.255.255` arasındaki her adres aynı ağda sayılır ve `192.168.11.20` de sunucuyla aynı ağda olur.

> [!IMPORTANT]
> **Prefix küçüldükçe ağ büyür.** `/16`, `/24`'ten daha küçük bir sayı ama çok daha büyük bir ağ: ağ kısmı kısaldıkça cihazlara ayrılan bit sayısı artar.

---

## 3. Ağ adresi, broadcast ve kullanılabilir adresler

Bir ağdaki adreslerin hepsi cihazlara verilemez:

- **Ağ adresi:** Cihaz bitlerinin hepsi 0 olan ilk adres. Ağın kendisini temsil eder.
- **Broadcast adresi:** Cihaz bitlerinin hepsi 1 olan son adres. Ağdaki herkese aynı anda mesaj göndermek için kullanılır.
- **Kullanılabilir adresler:** Bu ikisinin arasında kalanlar.

`192.168.10.20/24` için:

| | Adres |
|---|---|
| Ağ adresi | `192.168.10.0` |
| İlk kullanılabilir | `192.168.10.1` |
| Son kullanılabilir | `192.168.10.254` |
| Broadcast | `192.168.10.255` |
| Toplam / kullanılabilir | 256 / 254 |

Kullanılabilir adres sayısı her zaman toplamdan 2 eksiktir.

---

## 4. Prefix ve netmask tablosu

Prefix bazen **netmask** olarak da yazılır. Netmask'te ağ bitleri 1, cihaz bitleri 0'dır. `/24` = `255.255.255.0`. Eski araçlarda ve bazı yapılandırma dosyalarında hala netmask görürsün.

| Prefix | Netmask | Toplam adres | Kullanılabilir |
|---|---|---|---|
| `/8` | `255.0.0.0` | 16.777.216 | 16.777.214 |
| `/16` | `255.255.0.0` | 65.536 | 65.534 |
| `/24` | `255.255.255.0` | 256 | 254 |
| `/25` | `255.255.255.128` | 128 | 126 |
| `/26` | `255.255.255.192` | 64 | 62 |
| `/27` | `255.255.255.224` | 32 | 30 |
| `/28` | `255.255.255.240` | 16 | 14 |
| `/29` | `255.255.255.248` | 8 | 6 |
| `/30` | `255.255.255.252` | 4 | 2 |
| `/32` | `255.255.255.255` | 1 | Tek bir adres |

Toplam adres sayısı = 2^(32 − prefix). Örneğin `/27` için 2⁵ = 32.

---

## 5. Oktet sınırına denk gelmeyen prefix'ler

`/8`, `/16` ve `/24` kolaydır, çünkü sınır tam oktetler arasından geçer. `/20` veya `/28` gibi prefix'lerde sınır bir oktetin ortasından geçer. Bunları hesaplamanın pratik bir yolu var: **blok büyüklüğü**.

### Yöntem

1. Netmask'te 255 veya 0 olmayan okteti bul.
2. Blok büyüklüğü = 256 − o oktetin değeri.
3. Ağ adresleri o oktette blok büyüklüğünün katlarıdır (0, 16, 32, 48...). Adresin düştüğü katı bul.

### Örnek 1: 192.168.10.70/28

1. `/28` = `255.255.255.240`. 255 olmayan oktet: 4. oktet, değeri 240.
2. Blok büyüklüğü = 256 − 240 = **16**.
3. 4. oktetteki ağlar: 0, 16, 32, 48, **64**, 80... 70, 64 ile 80 arasında.

| | Adres |
|---|---|
| Ağ adresi | `192.168.10.64` |
| Kullanılabilir | `192.168.10.65` – `192.168.10.78` |
| Broadcast | `192.168.10.79` (bir sonraki ağdan bir önce) |

### Örnek 2: 10.0.37.5/20

1. `/20` = `255.255.240.0`. 255 olmayan oktet: 3. oktet, değeri 240.
2. Blok büyüklüğü = **16**, ama bu sefer 3. oktette.
3. 3. oktetteki ağlar: 0, 16, **32**, 48... 37, 32 ile 48 arasında.

| | Adres |
|---|---|
| Ağ adresi | `10.0.32.0` |
| Kullanılabilir | `10.0.32.1` – `10.0.47.254` |
| Broadcast | `10.0.47.255` |
| Kullanılabilir adres sayısı | 4094 |

### ipcalc ile kontrol

`ipcalc` aracı bu hesapları yapar ve bitleri de gösterir. Kurulu değilse `sudo apt install ipcalc`:

```bash
ipcalc 10.0.37.5/20
```
```
Address:   10.0.37.5            00001010.00000000.0010 0101.00000101
Netmask:   255.255.240.0 = 20   11111111.11111111.1111 0000.00000000
Wildcard:  0.0.15.255           00000000.00000000.0000 1111.11111111
=>
Network:   10.0.32.0/20         00001010.00000000.0010 0000.00000000
HostMin:   10.0.32.1            00001010.00000000.0010 0000.00000001
HostMax:   10.0.47.254          00001010.00000000.0010 1111.11111110
Broadcast: 10.0.47.255          00001010.00000000.0010 1111.11111111
Hosts/Net: 4094                  Class A, Private Internet
```

Çıktıdaki boşluk, ağ bitlerinin bittiği yeri gösteriyor.

> [!TIP]
> Bir sistemde `ipcalc` olmayabilir. Blok büyüklüğü yöntemini bilirsen araca ihtiyacın kalmaz.

---

## 6. Özel ve özel amaçlı IPv4 adresleri

Bazı aralıklar özel amaçlar için ayrılmıştır:

| Aralık | Amacı |
|---|---|
| `10.0.0.0/8` | Özel (private) ağlar |
| `172.16.0.0/12` | Özel ağlar (`172.16.0.0` – `172.31.255.255`) |
| `192.168.0.0/16` | Özel ağlar |
| `127.0.0.0/8` | Loopback: makinenin kendisi (`127.0.0.1` = localhost) |
| `169.254.0.0/16` | Link-local: DHCP'den adres alınamayınca cihazın kendine verdiği adres |

Özel adresler internette yönlendirilmez. Evdeki ve ofisteki cihazlar bu aralıklardan adres alır, internete çıkarken router adresi NAT ile değiştirir (NAT'ı [ilgili dosyada](07-nat-ve-port-yonlendirme.md) göreceğiz).

> [!NOTE]
> `172.16.0.0/12` aralığı `172.31.255.255`'te biter. `172.32.0.1` özel bir adres **değildir**.

Bir makinenin ağ ayarlarında `169.254.x.x` görüyorsan, bu genelde DHCP sunucusuna ulaşılamadığının işaretidir.

---

## 7. IPv6 adresinin yapısı

IPv6 adresi 128 bittir ve iki nokta (`:`) ile ayrılmış 8 gruptan oluşur. Her grup 16 bittir ve onaltılık (hex) sayılarla yazılır:

```
2001:0db8:00ab:0000:0000:0000:0000:0042
```

| | IPv4 | IPv6 |
|---|---|---|
| Uzunluk | 32 bit | 128 bit |
| Bölüm sayısı | 4 oktet | 8 grup |
| Bölüm başına bit | 8 | 16 |
| Sayı sistemi | Ondalık (0–255) | Onaltılık (0–9, a–f) |
| Ayraç | `.` | `:` |

Gruplarda harf olmasının sebebi onaltılık sistem: 10–15 arası değerler `a`–`f` ile yazılır.

> [!NOTE]
> Bu dosyadaki `2001:db8::` ile başlayan adresler, belgelerde örnek olarak kullanılmak için ayrılmış aralıktandır. Gerçek bir ağda karşına çıkmaz.

---

## 8. IPv6 kısaltma kuralları

Uzun IPv6 adreslerini kısaltmanın iki kuralı var:

**Kural 1: Her gruptaki baştaki sıfırlar atılabilir.**

```
2001:0db8:00ab:0000:...  ->  2001:db8:ab:0:...
```

**Kural 2: Art arda gelen sıfır gruplarının yerine bir kez `::` yazılabilir.**

```
2001:db8:ab:0:0:0:0:42  ->  2001:db8:ab::42
```

Python'un `ipaddress` modülüyle kontrol ettiğimiz örnekler:

| Uzun hali | Kısa hali |
|---|---|
| `2001:0db8:00ab:0000:0000:0000:0000:0042` | `2001:db8:ab::42` |
| `2001:0db8:0000:0000:00ff:0000:0000:0001` | `2001:db8::ff:0:0:1` |
| `fe80:0000:0000:0000:02fc:00ff:fe00:0001` | `fe80::2fc:ff:fe00:1` |
| `0000:0000:0000:0000:0000:0000:0000:0001` | `::1` |

İkinci örnekte iki ayrı sıfır bloğu var. `::` sadece bir kez kullanılabildiği için ilk (soldaki) blok kısaltıldı, diğeri `0:0` olarak kaldı.

### Kısa adresi açmak

`2001:db8::ff:0:0:1` adresinde 5 grup görünüyor. Toplam 8 grup olması gerektiği için `::`, 8 − 5 = 3 sıfır grubunun yerini tutuyor:

```
2001:0db8:0000:0000:00ff:0000:0000:0001
```

> [!WARNING]
> `::` bir adreste **sadece bir kez** kullanılabilir. `2001:db8::1::2` geçersizdir, çünkü hangi `::`'nin kaç grup yerine geçtiği belirsiz kalır. Python da bu adresi geçersiz diye reddetti.

---

## 9. IPv6'da prefix ve özel adresler

CIDR IPv6'da da aynı şekilde çalışır:

```
2001:db8:abcd:12::20/64
```

`/64`, ilk 64 bitin ağı gösterdiği anlamına gelir. Yerel ağlar için standart prefix `/64`'tür. Kalan 64 bit cihazlara ayrılır, yani tek bir `/64` ağında 2⁶⁴ (yaklaşık 18 kentilyon) adres var.

Adresin kısaltılması prefix'i değiştirmez: `2001:0db8:abcd:0012:0000:0000:0000:0020/64` ile `2001:db8:abcd:12::20/64` aynı adres ve aynı ağ.

Sık karşılaşacağın IPv6 adresleri:

| Adres / aralık | Anlamı | IPv4'teki karşılığı |
|---|---|---|
| `::1` | Loopback | `127.0.0.1` |
| `fe80::/10` | Link-local. Her IPv6 arayüzü kendiliğinden bir tane alır, sadece aynı ağ parçasında geçerli | `169.254.0.0/16` |
| `fd00::/8` | Unique local, özel ağlar için | `10.0.0.0/8` vb. |
| `2000::/3` | Global, internette yönlendirilen adresler | Public adresler |

---

## 10. Sunucuda adresleri görmek

Arayüzlerin adreslerini görmek için:

```bash
ip -br addr       # kısa özet: arayüz, durum, adresler
ip addr           # ayrıntılı (kısaltması: ip a)
ip -4 addr        # sadece IPv4
ip -6 addr        # sadece IPv6
```

`ip -br addr` çıktısı:

```
lo               UNKNOWN        127.0.0.1/8
eth0             UP             192.0.2.2/24
```

`ip addr` çıktısında IPv4 adresleri `inet`, IPv6 adresleri `inet6` satırlarında görünür. `inet6` satırının sonundaki `scope link` adresin link-local olduğunu, `scope global` ise global bir adres olduğunu gösterir.

```bash
ip addr show eth0
```
```
4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:fc:00:00:00:01 brd ff:ff:ff:ff:ff:ff
    inet 192.0.2.2/24 brd 192.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
```

Bu satırdan okuyabileceklerin:

- `inet 192.0.2.2/24`: adres ve prefix
- `brd 192.0.2.255`: broadcast adresi (3. bölümdeki hesapla aynı)
- `link/ether 02:fc:...`: arayüzün MAC adresi

Adresleri ekleyip kaldırmayı ve kalıcı yapmayı [Ağ Yapılandırması](02-ag-yapilandirmasi.md) dosyasında yapacağız.

---

## 11. Özet

| Konu | Hatırlanacak |
|---|---|
| IPv4 | 32 bit, 4 oktet, her oktet 0–255 |
| IPv6 | 128 bit, 8 grup, onaltılık |
| CIDR | `/24` = ilk 24 bit ağ. Prefix küçüldükçe ağ büyür |
| Toplam adres | 2^(32 − prefix) |
| Kullanılabilir | Toplam − 2 (ağ adresi ve broadcast) |
| Oktet ortası prefix | Blok büyüklüğü = 256 − netmask okteti |
| Özel IPv4 | `10/8`, `172.16/12`, `192.168/16` |
| IPv6 kısaltma | Baştaki sıfırları at, bir kez `::` kullan |
| IPv6 özel | `::1` loopback, `fe80::/10` link-local |

---

## 12. Sık yapılan hatalar

- **Büyük prefix'in büyük ağ olduğunu sanmak.** `/16` ağı `/24`'ten çok daha büyüktür.
- **Ağ adresini veya broadcast'i bir cihaza vermek.** `/24`'te `.0` ve `.255` kullanılamaz.
- **Aynı ağ kararını sadece ilk oktetlere bakarak vermek.** Önce prefix'e bak, sonra ağ kısmını karşılaştır.
- **`172.16.0.0/12`'nin bütün `172.x` adreslerini kapsadığını sanmak.** Sadece `172.16`–`172.31` arası.
- **IPv6'da `::`'yi iki kez kullanmak.** Adres geçersiz olur.
- **Kısaltılmış IPv6 adresinin farklı bir ağda olduğunu sanmak.** Kısaltma adresi ve prefix'i değiştirmez.

---

## 13. Alıştırmalar

Önce kağıt kalemle çöz, sonra `ipcalc` ile kontrol et.

**1.** `192.168.10.20/24` ile `192.168.10.200/24` aynı ağda mı? Peki `192.168.10.20/25` ile `192.168.10.200/25`?

<details>
<summary>Çözüm</summary>

- `/24`: İkisinin de ağ kısmı `192.168.10`, aynı ağdalar.
- `/25`: Blok büyüklüğü 256 − 128 = 128. Ağlar `.0` ve `.128`. `.20` ilk ağda (`192.168.10.0/25`), `.200` ikinci ağda (`192.168.10.128/25`). **Farklı ağdalar.**

```bash
ipcalc 192.168.10.20/25 | grep Network
ipcalc 192.168.10.200/25 | grep Network
```
</details>

**2.** `172.20.5.9/12` adresinin ağ adresini ve broadcast adresini bul.

<details>
<summary>Çözüm</summary>

`/12` = `255.240.0.0`. 255 olmayan oktet 2. oktet, değeri 240. Blok büyüklüğü 16. 2. oktette ağlar 0, 16, 32... 20, 16 ile 32 arasında.

- Ağ adresi: `172.16.0.0`
- Broadcast: `172.31.255.255`

Bu da özel `172.16.0.0/12` aralığının ta kendisi.
</details>

**3.** Bir ofis ağında 50 cihaz var. Hepsine yetecek en küçük prefix hangisi?

<details>
<summary>Çözüm</summary>

`/27` 30 kullanılabilir adres verir, yetmez. `/26` 62 kullanılabilir adres verir, yeter. Cevap: **`/26`**.
</details>

**4.** Şu IPv6 adresini en kısa haliyle yaz: `2001:0db8:0000:0000:0000:0abc:0000:0001`

<details>
<summary>Çözüm</summary>

Baştaki sıfırlar atılınca: `2001:db8:0:0:0:abc:0:1`. En uzun sıfır bloğu (3 grup) `::` olur:

```
2001:db8::abc:0:1
```

Python ile kontrol:

```bash
python3 -c "import ipaddress; print(ipaddress.ip_address('2001:0db8:0000:0000:0000:0abc:0000:0001'))"
```
</details>

**5.** `fd12:3456::/64`, `fe80::1` ve `2001:db8::1` adreslerinin her birinin türünü yaz.

<details>
<summary>Çözüm</summary>

- `fd12:3456::/64`: unique local, özel bir ağ
- `fe80::1`: link-local, sadece aynı ağ parçasında geçerli
- `2001:db8::1`: belgelerde örnek olarak kullanılmak için ayrılmış aralık
</details>

---

Bölüm ana sayfası: [Ağ Yönetimi](README.md) · Sonraki konu: [Ağ Yapılandırması](02-ag-yapilandirmasi.md)
