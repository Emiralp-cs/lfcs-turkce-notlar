# Bridge ve Bonding

> **Resmi sınav maddesi:** Configure bridge and bonding devices

İkisi de birden fazla ağ arayüzünü tek bir sanal arayüzün altında toplar, ama amaçları farklıdır:

- **Bridge (köprü):** Birden fazla ağ bağlantısını **tek bir ağ** haline getirir. Sunucu, arayüzleri arasında bir switch gibi davranır.
- **Bond (bağ):** Aynı ağa giden birden fazla bağlantıyı **tek bir bağlantı** gibi kullanır. Amaç yedeklilik ve/veya hız.

Senaryo:

- **Bridge:** Sunucuda sanal makineler çalışıyor. İki sanal makinenin ağ kartları sunucudaki `enp0s8` ve `enp0s9`'a bağlı. Hepsi aynı ağda (`192.168.20.0/24`) birbirleriyle konuşabilmeli.
- **Bond:** Sunucunun ofis ağına iki kabloyla bağlanması isteniyor. Bir kablo veya kart arızalanırsa bağlantı kopmamalı.

> [!NOTE]
> Bridge, Ubuntu 24.04'te ağ namespace'leriyle iki sanal makine bağlanarak gerçekten denendi. Netplan dosyaları `netplan generate` ile doğrulandı. Deneme ortamının çekirdeğinde **bonding desteği olmadığı** için bond arayüzü oluşturulamadı. Bonding kısmındaki çalışma zamanı çıktıları ve davranışlar Linux çekirdeğinin bonding belgelerine dayanıyor, Netplan bond dosyaları ise `netplan generate` ile doğrulandı.

## İçindekiler

1. [Bridge nedir?](#1-bridge-nedir)
2. [Bridge'i ip ile denemek](#2-bridgei-ip-ile-denemek)
3. [Bridge'i Netplan ile kalıcı yapmak](#3-bridgei-netplan-ile-kalıcı-yapmak)
4. [Bond nedir?](#4-bond-nedir)
5. [Bonding modları](#5-bonding-modları)
6. [Bond'u Netplan ile yapılandırmak](#6-bondu-netplan-ile-yapılandırmak)
7. [Bond'u kontrol etmek](#7-bondu-kontrol-etmek)
8. [Bridge ve bond farkı](#8-bridge-ve-bond-farkı)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Bridge nedir?

Bir switch'i düşün: kablolarla bağlanan bütün cihazlar aynı ağda olur ve birbirleriyle doğrudan konuşur. Linux'ta bridge, bunu yazılımla yapar. Bridge'e eklenen her arayüze **port** denir.

En yaygın kullanımı sanal makineler ve konteynerler. Sanallaştırma yazılımı fiziksel kartı ve her sanal makinenin sanal kartını aynı bridge'e bağlar. Böylece sanal makineler fiziksel ağdaki diğer cihazlarla aynı ağda olur.

> [!IMPORTANT]
> Bridge, farklı **IP ağlarını** birbirine bağlamaz. Bağladığı bağlantıları tek bir ağ yapar ve cihazların aynı IP ağında olması gerekir. Farklı IP ağları arasında geçiş **yönlendirme** (routing) işidir ([Ağ Yapılandırması](02-ag-yapilandirmasi.md#4-geçici-route-eklemek) ve [NAT](07-nat-ve-port-yonlendirme.md) dosyalarında). [2. bölümde](#2-bridgei-ip-ile-denemek) bunu deniyoruz.

---

## 2. Bridge'i ip ile denemek

İki sanal makine (`vm1`: `192.168.20.11`, `vm2`: `192.168.20.12`) sunucunun `enp0s8` ve `enp0s9` arayüzlerine bağlı. Henüz bridge yok:

```bash
ping -c 1 192.168.20.12     # vm1'den
```
```
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

İki bağlantı birbirinden ayrı olduğu için paket karşıya ulaşmıyor.

### Bridge oluşturmak

```bash
sudo ip link add br0 type bridge
sudo ip link set enp0s8 master br0
sudo ip link set enp0s9 master br0
sudo ip link set br0 up
```

`master br0`, arayüzü bridge'in portu yapar.

```bash
ping -c 2 192.168.20.12     # vm1'den
```
```
2 packets transmitted, 2 received, 0% packet loss, time 1027ms
```

### Farklı IP ağıyla

`vm2`'nin adresini `192.168.30.12/24` yapıp tekrar denedik:

```
ping: connect: Network is unreachable
```

Bridge aynı, kablolar aynı, ama iki makine farklı IP ağlarında olduğu için konuşamıyor. Bridge sadece aynı ağdaki cihazları birleştirir.

### Bridge'i görmek

```bash
ip link show master br0      # br0'ın portları
ip -br link show type bridge # bridge arayüzleri
bridge link                  # portların durumu
```
```
5: enp0s8@ifb0: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 master br0 state forwarding priority 32 cost 2
6: enp0s9@ifb0: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 master br0 state forwarding priority 32 cost 2
```

`state forwarding`, portun trafik ilettiğini gösterir. `ip link` çıktısında da port arayüzlerinin satırında `master br0` yazar.

Bridge, hangi MAC adresinin hangi portun arkasında olduğunu öğrenir. Bu tabloyu görmek için:

```bash
bridge fdb show br br0
```

### Sunucunun kendi adresi

Sunucunun da bu ağda bir adresi olacaksa adres **bridge'e** verilir, port arayüzlerine değil:

```bash
sudo ip addr add 192.168.20.1/24 dev br0
```

Denemede `vm1`, `192.168.20.1`'e ping atabildi. Port arayüzlerinde (`enp0s8`, `enp0s9`) IP adresi yoktu ve olmamalı.

### Bridge'i kaldırmak

```bash
sudo ip link delete br0
```

Bridge silinince portlar serbest kalır, arayüzler yerinde durur.

> [!WARNING]
> Bütün bunlar geçicidir. Yeniden başlatınca bridge kaybolur. Kalıcı yapmak için Netplan kullanılır. Ayrıca sunucuya bağlı olduğun arayüzü (örneğin SSH bağlantısının geldiği `enp0s3`) yanlışlıkla bridge'e eklersen, o arayüzün adresi işlevsiz kalır ve bağlantın kopar.

---

## 3. Bridge'i Netplan ile kalıcı yapmak

Netplan'ın hazır örneğiyle başlamak en kolayı:

```bash
cat /usr/share/doc/netplan/examples/bridge.yaml
```
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: no
  bridges:
    br0:
      dhcp4: yes
      interfaces:
        - enp3s0
```

Örneği kopyalayıp kendi arayüz isimlerinle düzenle. Önce arayüz isimlerini öğren:

```bash
ip -br link
```

```bash
sudo cp /usr/share/doc/netplan/examples/bridge.yaml /etc/netplan/99-kopru.yaml
sudo chmod 600 /etc/netplan/99-kopru.yaml
sudo vim /etc/netplan/99-kopru.yaml
```

Senaryomuz için:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
    enp0s9:
      dhcp4: false
  bridges:
    br0:
      addresses: [192.168.20.1/24]
      interfaces: [enp0s8, enp0s9]
```

- Port arayüzleri `ethernets` altında tanımlanır ama kendi adresleri olmaz (`dhcp4: false`).
- Adres (veya `dhcp4: true`) bridge'e verilir.
- `interfaces`, bridge'e hangi portların ekleneceğini belirler.

Netplan'ın bu dosyadan ürettiği networkd yapılandırması:

```
## 10-netplan-br0.netdev
[NetDev]
Name=br0
Kind=bridge

## 10-netplan-enp0s8.network
[Match]
Name=enp0s8

[Network]
LinkLocalAddressing=no
Bridge=br0
```

`enp0s8` için `Bridge=br0` satırı, `ip link set enp0s8 master br0` komutunun kalıcı karşılığı.

Uygula ve kontrol et:

```bash
sudo netplan try
ip link show master br0
```

Kalıcı bir bridge'i kaldırmak için önce YAML dosyasını sil, sonra `sudo ip link delete br0`. Dosyayı silmezsen bridge açılışta tekrar oluşur.

---

## 4. Bond nedir?

Sunucunun iki ağ kartı aynı switch'e bağlı olsun. Bond olmadan bunlar iki ayrı bağlantıdır. Sunucunun adresi hangisindeyse, o kart veya kablo arızalanınca bağlantı kesilir.

Bond, iki kartı **tek bir sanal arayüz** (`bond0`) altında toplar. Adres bond arayüzüne verilir. Uygulamalar sadece `bond0`'ı görür, arkasında kaç kart olduğunu bilmez.

Bond'un sağladıkları:

| Fayda | Açıklama |
|---|---|
| Yedeklilik | Bir kart veya kablo arızalanınca trafik diğerinden devam eder |
| Daha yüksek toplam bant genişliği | Trafik birden fazla karta dağıtılabilir (moda bağlı) |

Bond'a eklenen arayüzlere eski belgelerde **slave**, bond'un kendisine **master** denir. Yeni belgelerde "port" veya "bağlı arayüz" terimleri de kullanılıyor.

---

## 5. Bonding modları

Linux 7 bonding modu sunar. Mod, trafiğin kartlar arasında nasıl paylaşılacağını belirler. Netplan'da numara değil **isim** kullanılır:

| Mod | Netplan adı | Ne yapar? | Switch ayarı gerekir mi? |
|---|---|---|---|
| 0 | `balance-rr` | Paketleri kartlara sırayla gönderir | Evet (portlar gruplanmalı) |
| 1 | `active-backup` | Tek kart aktif, diğeri yedekte bekler | Hayır |
| 2 | `balance-xor` | Kaynak/hedef bilgisine göre her bağlantı hep aynı karttan gider | Evet (portlar gruplanmalı) |
| 3 | `broadcast` | Her paketi bütün kartlardan gönderir | Evet (portlar gruplanmalı) |
| 4 | `802.3ad` | LACP protokolüyle switch ile birlikte dinamik birleştirme | Evet (switch'te LACP) |
| 5 | `balance-tlb` | Giden trafiği kartların yüküne göre dağıtır | Hayır |
| 6 | `balance-alb` | Hem giden hem gelen trafiği dağıtır | Hayır |

Netplan'da numara veya başka bir ad yazmayı denedik:

```
mode: 1      ->  Error in network definition: unknown bond mode '1'
mode: mode4  ->  Error in network definition: unknown bond mode 'mode4'
```

Yedi ismin yedisi de kabul edildi.

Hangisini seçmeli?

- **Sadece yedeklilik, en basit kurulum:** `active-backup`. Switch tarafında hiçbir ayar gerektirmez. Performans artışı sağlamaz.
- **Switch LACP destekliyorsa:** `802.3ad`. Hem yedeklilik hem toplam bant genişliği sağlar ve standarttır.
- **Switch ayarına dokunamıyorsan ve yük dağıtmak istiyorsan:** `balance-alb`.

`broadcast` aynı veriyi bütün kartlardan gönderdiği için bant genişliğini boşa harcar, sadece çok özel durumlarda kullanılır.

> [!NOTE]
> Switch ayarı gerektirmeyen modlar 1, 5 ve 6'dır. 0, 2 ve 3, switch'te portların bir grup olarak (statik EtherChannel/trunk) tanımlanmasını bekler. 4 ise LACP ile bunu dinamik olarak yapar.

---

## 6. Bond'u Netplan ile yapılandırmak

Netplan'ın hazır örneği:

```bash
cat /usr/share/doc/netplan/examples/bonding.yaml
```
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0: {}
    enp4s0: {}
  bonds:
    bond0:
      dhcp4: yes
      interfaces:
        - enp3s0
        - enp4s0
      parameters:
        mode: active-backup
        primary: enp3s0
        mii-monitor-interval: 100
```

Senaryomuz için `/etc/netplan/99-bag.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8: {}
    enp0s9: {}
  bonds:
    bond0:
      addresses: [192.168.10.20/24]
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: active-backup
        primary: enp0s8
        mii-monitor-interval: 100
```

```bash
sudo chmod 600 /etc/netplan/99-bag.yaml
sudo netplan generate
sudo netplan try
```

| Parametre | Anlamı |
|---|---|
| `mode: active-backup` | Mod 1: bir kart aktif, diğeri yedek |
| `primary: enp0s8` | İki kart da sağlamken aktif olacak kart |
| `mii-monitor-interval: 100` | Kartların bağlantı durumunu 100 ms'de bir kontrol et. Arıza bu sayede fark edilir |

`enp0s8: {}`, arayüzü hiçbir ayar vermeden tanımlamak demek. Adres bond'a verildiği için kartların kendi ayarı yok.

Netplan'ın ürettiği yapılandırma:

```
## 10-netplan-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup
MIIMonitorSec=100ms

## 10-netplan-enp0s8.network
[Match]
Name=enp0s8

[Network]
LinkLocalAddressing=no
Bond=bond0
PrimarySlave=true
```

Bütün bond parametreleri için `man netplan` içinde `/bonds` ile arama yap.

---

## 7. Bond'u kontrol etmek

> [!NOTE]
> Bu bölümdeki komutlar deneme ortamında çalıştırılamadı (çekirdekte bonding desteği yoktu). Çıktı biçimleri Linux çekirdeğinin bonding belgelerine göre anlatılıyor.

```bash
ip link
```

Bond uygulandıktan sonra:

- Kartların satırında `SLAVE` bayrağı ve `master bond0` görünür.
- `bond0` satırında `MASTER` bayrağı görünür.
- `active-backup` modunda kartlar ve bond arayüzü varsayılan olarak **aynı MAC adresini** kullanır. Ağdaki diğer cihazlar tek bir kimlik görür, aktif kart değişse bile.

Bond'un ayrıntılı durumu çekirdeğin sunduğu bir dosyada:

```bash
cat /proc/net/bonding/bond0
```

Bu dosyada bakılacak satırlar:

| Satır | Anlamı |
|---|---|
| `Bonding Mode:` | Kullanılan mod |
| `Currently Active Slave:` | `active-backup`'ta şu an trafiği taşıyan kart |
| `MII Status:` | Bond'un ve her kartın bağlantı durumu (`up` / `down`) |
| `MII Polling Interval (ms):` | `mii-monitor-interval` değeri |
| `Link Failure Count:` | Kartın kaç kez arızaya geçtiği |

Arızayı denemek için aktif kartı kapatıp `Currently Active Slave` satırının diğer karta geçtiğini görebilirsin:

```bash
sudo ip link set enp0s8 down
grep 'Currently Active' /proc/net/bonding/bond0
sudo ip link set enp0s8 up
```

Bond arayüzü sonrasında normal bir arayüz gibi yönetilir: `ip addr add ... dev bond0`, `ip link set bond0 down` gibi.

---

## 8. Bridge ve bond farkı

| | Bridge | Bond |
|---|---|---|
| Ne yapar? | Birden fazla bağlantıyı tek bir ağ yapar (switch gibi) | Aynı ağa giden birden fazla bağlantıyı tek bağlantı yapar |
| Kartlar nereye bağlı? | Farklı cihazlara (her biri ayrı bir bağlantı) | Aynı ağa, genelde aynı switch'e |
| Amaç | Cihazları aynı ağda buluşturmak | Yedeklilik ve/veya bant genişliği |
| Tipik kullanım | Sanal makineler, konteynerler | Sunucunun ağ bağlantısını sağlamlaştırmak |
| Netplan bloğu | `bridges:` | `bonds:` |
| Portları gösteren bayrak | `master br0` | `SLAVE`, `master bond0` |

Soruyu okurken: "makineleri aynı ağa bağla" veya "sanal makineler fiziksel ağa erişsin" deniyorsa bridge, "yedeklilik", "bir kart arızalanırsa" veya "iki kartı birleştir" deniyorsa bond.

İkisi birlikte de kullanılabilir: önce iki kart bond'a, sonra `bond0` bir bridge'e eklenir. Böylece sanal makineler de yedekli bir bağlantı kullanır.

---

## 9. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `ip link add br0 type bridge` | Bridge oluşturur (geçici) |
| `ip link set ARAYÜZ master br0` | Arayüzü bridge'e port olarak ekler |
| `ip link set ARAYÜZ nomaster` | Arayüzü bridge'den çıkarır |
| `ip link show master br0` | Bridge'in portları |
| `bridge link` | Portların durumu |
| `bridge fdb show br br0` | Öğrenilmiş MAC adresleri |
| `ip link delete br0` | Bridge'i siler |
| `/usr/share/doc/netplan/examples/bridge.yaml` | Netplan bridge örneği |
| `/usr/share/doc/netplan/examples/bonding.yaml` | Netplan bond örneği |
| `bridges: br0: interfaces: [...]` | Netplan'da bridge |
| `bonds: bond0: interfaces: [...] parameters: mode: ...` | Netplan'da bond |
| `cat /proc/net/bonding/bond0` | Bond'un ayrıntılı durumu |
| `man netplan` | `/bridges` ve `/bonds` ile parametreler |

---

## 10. Sık yapılan hatalar

- **Bridge'in farklı IP ağlarını bağladığını sanmak.** Cihazların aynı IP ağında olması gerekir.
- **IP adresini port arayüzüne vermek.** Adres bridge'e veya bond'a verilir.
- **SSH bağlantısının geldiği arayüzü bridge/bond'a eklemek.** Bağlantı kopar.
- **Netplan'da mod numarası yazmak.** `mode: 1` değil `mode: active-backup`.
- **`802.3ad`'yi switch'te LACP ayarlamadan kullanmak.** Bond düzgün çalışmaz.
- **`mii-monitor-interval`'ı unutmak.** Arıza fark edilmeyebilir.
- **Kalıcı bridge'i silerken YAML dosyasını bırakmak.** Açılışta tekrar oluşur.
- **Hangisinin isteneceğini karıştırmak.** Aynı ağda buluşmak bridge, yedeklilik bond.

---

## 11. Alıştırmalar

**1.** `enp0s8` ve `enp0s9` arayüzlerini `br1` adlı geçici bir bridge'e ekle, bridge'e `10.50.0.1/24` adresini ver ve portları doğrula. Sonra bridge'i sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ip link add br1 type bridge
sudo ip link set enp0s8 master br1
sudo ip link set enp0s9 master br1
sudo ip link set br1 up
sudo ip addr add 10.50.0.1/24 dev br1
ip link show master br1
bridge link
sudo ip link delete br1
```
</details>

**2.** Aynı bridge'i Netplan ile kalıcı yap. Bridge adresini DHCP'den alsın.

<details>
<summary>Çözüm</summary>

`/etc/netplan/99-kopru.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
    enp0s9:
      dhcp4: false
  bridges:
    br1:
      dhcp4: true
      interfaces: [enp0s8, enp0s9]
```

```bash
sudo chmod 600 /etc/netplan/99-kopru.yaml
sudo netplan generate
sudo netplan try
```
</details>

**3.** `enp0s8` ve `enp0s9`'u `bond0` altında birleştir. Switch'e dokunmadan hem yedeklilik hem gelen ve giden trafik için yük dağılımı istiyorsun. Bond'un adresi `192.168.10.20/24` olsun.

<details>
<summary>Çözüm</summary>

Switch ayarı gerektirmeden iki yönde yük dağıtan mod `balance-alb`.

```yaml
network:
  version: 2
  ethernets:
    enp0s8: {}
    enp0s9: {}
  bonds:
    bond0:
      addresses: [192.168.10.20/24]
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: balance-alb
        mii-monitor-interval: 100
```

```bash
sudo chmod 600 /etc/netplan/99-bag.yaml
sudo netplan try
cat /proc/net/bonding/bond0
```
</details>

**4.** Aşağıdaki durumların her biri için bridge mi bond mu kullanırsın?

a) Sunucudaki sanal makineler ofis ağında kendi adreslerini alsın.
b) Sunucunun ağ kartı bozulursa bağlantı kesilmesin.
c) Switch LACP destekliyor, iki 1 Gbit kart birlikte kullanılsın.

<details>
<summary>Çözüm</summary>

a) Bridge. b) Bond (`active-backup` yeterli). c) Bond, `802.3ad` modu.
</details>

**5.** Aşağıdaki Netplan dosyasında iki hata var. Bul.

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses: [192.168.10.20/24]
    enp0s9: {}
  bonds:
    bond0:
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: 1
```

<details>
<summary>Çözüm</summary>

1. Adres bond'a ait karta (`enp0s8`) verilmiş. Adres `bond0`'a verilmeli.
2. `mode: 1` geçersiz. Netplan isim ister: `mode: active-backup`.

Ayrıca `mii-monitor-interval` eklemek iyi olur, yoksa arıza fark edilmeyebilir.
</details>

---

Önceki konu: [NAT ve Port Yönlendirme](07-nat-ve-port-yonlendirme.md) · Sonraki konu: [Ters Proxy ve Yük Dengeleme](09-ters-proxy-ve-yuk-dengeleme.md)
