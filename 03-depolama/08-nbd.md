# NBD (Network Block Device)

> **Resmi sınav maddesi:** Use remote filesystems and network block devices

[NFS](07-nfs.md) bir **dizini** paylaşır: dosya sistemi sunucuda durur, istemci dosyaları ister. **NBD** ise bir **diski** paylaşır: istemci, sunucudaki bir diski veya bölümü ağ üzerinden kendi diski gibi görür (`/dev/nbd0`). Üzerindeki dosya sistemini istemci bağlar ve yönetir. İstemci isterse diski bölümlendirebilir veya biçimlendirebilir.

| | NFS | NBD |
|---|---|---|
| Paylaşılan | Dizin | Disk veya bölüm |
| İstemcide görünen | Bağlanmış bir dizin | `/dev/nbd0` diye bir disk |
| Dosya sistemini yöneten | Sunucu | İstemci |
| Aynı anda birden fazla istemci | Evet | Okuma-yazma için **hayır**, dosya sistemi bozulur |

Senaryo: Depolama sunucusu (`10.0.5.15`), ikinci diskini `arsiv` adıyla paylaşacak. Uygulama sunucusu bu diske bağlanıp üzerindeki ext4 dosya sistemini `/mnt/arsiv`'e bağlayacak.

> [!WARNING]
> Deneme ortamının çekirdeğinde **`nbd` modülü yoktu.** Sunucu tarafı (kurulum, yapılandırma, servisi başlatmak, paylaşımları listelemek, izin sorunları) gerçekten denendi ve çıktılar oradan alındı. Sunucudaki disk yerine 200 MB'lık bir dosyaya bağlı `/dev/loop5` kullanıldı. İstemci, sunucuyla anlaşmayı başarıyla tamamladı (`size = 200MB`), ama `/dev/nbd0` oluşturulamadığı için **istemcide diski bağlamak, `lsblk` çıktısı, mount ve bağlantıyı kesmek denenmedi.**

## İçindekiler

1. [Sunucu: kurulum ve yapılandırma](#1-sunucu-kurulum-ve-yapılandırma)
2. [Sunucu: izin tuzağı](#2-sunucu-izin-tuzağı)
3. [Modül ve bağlantı (istemci)](#3-modül-ve-bağlantı-istemci)
4. [Diski kullanmak ve bağlantıyı kesmek (istemci)](#4-diski-kullanmak-ve-bağlantıyı-kesmek-istemci)
5. [Açılışta otomatik bağlanmak](#5-açılışta-otomatik-bağlanmak)
6. [Komut özeti](#6-komut-özeti)
7. [Sık yapılan hatalar](#7-sık-yapılan-hatalar)
8. [Alıştırmalar](#8-alıştırmalar)

---

## 1. Sunucu: kurulum ve yapılandırma

```bash
sudo apt install nbd-server
```

Yapılandırma dosyası `/etc/nbd-server/config`. Kurulumdan sonra içinde sadece `[generic]` bölümü vardır:

```ini
[generic]
	user = nbd
	group = nbd
	includedir = /etc/nbd-server/conf.d
```

`[generic]` sunucunun genel ayarlarıdır ve silinmemelidir. Her paylaşım, köşeli parantez içinde kendi adıyla ayrı bir bölümdür:

```ini
[generic]
	user = nbd
	group = disk
	includedir = /etc/nbd-server/conf.d
	allowlist = true

[arsiv]
	exportname = /dev/sdb
```

| Satır | Anlamı |
|---|---|
| `allowlist = true` | İstemcilerin `nbd-client -l` ile paylaşımları listelemesine izin ver |
| `[arsiv]` | Paylaşımın adı. İstemci bu adla bağlanır |
| `exportname = /dev/sdb` | Paylaşılan disk, bölüm (`/dev/sdb1`) veya disk imajı dosyası |
| `group = disk` | Bir sonraki bölümde |

```bash
sudo systemctl restart nbd-server
sudo ss -tlnp | grep nbd
```
```
LISTEN 0      10           0.0.0.0:10809      0.0.0.0:*    users:(("nbd-server",pid=11860,fd=3))
```

NBD sunucusu **TCP 10809** portunu dinler. ufw açıksa: `sudo ufw allow from 10.0.5.0/24 to any port 10809 proto tcp`.

---

## 2. Sunucu: izin tuzağı

Sunucu `root` olarak değil, `user` ve `group` satırlarındaki kullanıcıyla çalışır. Varsayılan `nbd:nbd`'dir. Ama diskler `root:disk` sahipliğindedir ve sadece bu ikisi okuyabilir:

```
brw-rw---- 1 root disk 7, 5 Nov 26 09:47 /dev/loop5
```

Varsayılan ayarlarla bir diski paylaşınca servis sorunsuz başlar, paylaşım listede de görünür. Ama istemci bağlanmaya çalışınca reddedilir:

```bash
sudo nbd-client 10.0.5.15 -N arsiv /dev/nbd0
```
```
Negotiation: ..Error: Connection not allowed by server policy. Server said: Access denied by server configuratio
Exiting.
```

Hata mesajı izinden bahsetmiyor, yapılandırmada bir şey yanlış gibi görünüyor. Asıl sebep, `nbd` kullanıcısının diski açamaması.

**`usermod -aG disk nbd` yetmez.** Denemede `nbd` kullanıcısını `disk` grubuna ekleyip servisi yeniden başlattık, hata devam etti. Sunucu sadece `group` satırındaki grubu kullanıyor, kullanıcının ek gruplarını değil:

```
USER     GROUP    SUPGRP   CMD
nbd      disk     -        /bin/nbd-server
```

Çözüm, `[generic]` içinde `group = disk` yazmak. Bundan sonra istemci anlaşmayı tamamladı:

```
Negotiation: ..size = 200MB
```

Diğer yol, `user` ve `group` satırlarını silip sunucuyu root olarak çalıştırmak. Bu da çalıştı, ama gereğinden fazla yetki verir.

> [!NOTE]
> Paylaşılan şey bir dosyaysa (ör. `exportname = /srv/disk.img`), dosyanın `nbd` kullanıcısı tarafından okunup yazılabilmesi yeterlidir: `sudo chown nbd:nbd /srv/disk.img`.

---

## 3. Modül ve bağlantı (istemci)

```bash
sudo apt install nbd-client
sudo modprobe nbd
```

`nbd` çekirdek modülü `/dev/nbd0`, `/dev/nbd1`, ... cihazlarını oluşturur. Modül yüklü değilse bu cihazlar yoktur. Açılışta yüklenmesi için:

```bash
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
```

Paket ayrıca `/etc/modprobe.d/nbd-client.conf` dosyasına `options nbd max_part=15` yazar: her NBD diskinde 15'e kadar bölüm görülebilir.

Sunucudaki paylaşımları listele:

```bash
nbd-client -l 10.0.5.15
```
```
Negotiation: ..
arsiv
```

Sunucuda `allowlist = true` yoksa:

```
E: listing not allowed by server.
Server said: Listing of exports denied by server configuration
```

Bağlan:

```bash
sudo nbd-client 10.0.5.15 -N arsiv /dev/nbd0
```

`-N` paylaşımın adıdır. Yanlış bir ad verirsen:

```
Negotiation: ..Error: Unknown error returned by server. Server said: Export unknow
```

(Mesajın sonu çıktıda da böyle kesik.)

---

## 4. Diski kullanmak ve bağlantıyı kesmek (istemci)

Bağlandıktan sonra `/dev/nbd0` normal bir disktir:

```bash
lsblk /dev/nbd0
sudo mkdir -p /mnt/arsiv
sudo mount /dev/nbd0 /mnt/arsiv        # disk bölümlendirilmişse /dev/nbd0p1
df -h /mnt/arsiv
```

Disk boşsa önce biçimlendirilir (`sudo mkfs.ext4 /dev/nbd0`), bölümlendirilebilir (`sudo fdisk /dev/nbd0`), LVM'e bile PV olarak verilebilir.

Bağlantıyı kesmek için **önce dosya sistemini ayır**, sonra bağlantıyı kes:

```bash
sudo umount /mnt/arsiv
sudo nbd-client -d /dev/nbd0
```

Sırayı ters yaparsan, bağlı dosya sisteminin altındaki disk kaybolur: henüz yazılmamış veriler kaybolabilir.

> [!CAUTION]
> Aynı NBD paylaşımına iki istemci okuma-yazma ile bağlanıp aynı ext4 veya XFS'i bağlarsa dosya sistemi bozulur. Bu dosya sistemleri, diski aynı anda başka bir makinenin değiştirdiğinden habersizdir. Ortak dosyalar gerekiyorsa NFS kullan.

---

## 5. Açılışta otomatik bağlanmak

İstemci paketi `/etc/nbdtab` dosyasını okur:

```
# cihaz  sunucu      paylaşım  seçenekler
nbd0     10.0.5.15   arsiv
```

Bu satırdan sonra `sudo nbd-client nbd0` yazmak, sunucuyu ve paylaşımı dosyadan okuyarak bağlanır. Paket ayrıca `nbd@.service` birimini getirir. Açılışta bağlanması için:

```bash
sudo systemctl enable --now nbd@nbd0
```

fstab'da bu diskin üzerindeki dosya sistemi için `_netdev` şarttır. Yoksa systemd bunun ağ üzerinden geldiğini bilemez ve ağ gelmeden bağlamaya çalışır:

```
/dev/nbd0  /mnt/arsiv  ext4  defaults,_netdev  0  0
```

---

## 6. Komut özeti

| Komut | Nerede | Ne yapar? |
|---|---|---|
| `sudo apt install nbd-server` | Sunucu | Sunucu paketi |
| `/etc/nbd-server/config` | Sunucu | `[generic]` ve paylaşım bölümleri |
| `sudo systemctl restart nbd-server` | Sunucu | Yapılandırmayı uygula |
| `sudo apt install nbd-client` | İstemci | İstemci paketi |
| `sudo modprobe nbd` | İstemci | `/dev/nbdX` cihazlarını oluştur |
| `/etc/modules-load.d/nbd.conf` | İstemci | Modülü açılışta yükle |
| `nbd-client -l sunucu` | İstemci | Paylaşımları listele |
| `sudo nbd-client sunucu -N ad /dev/nbd0` | İstemci | Bağlan |
| `sudo nbd-client -d /dev/nbd0` | İstemci | Bağlantıyı kes |
| `/etc/nbdtab`, `nbd@nbd0` | İstemci | Kalıcı bağlantı |

---

## 7. Sık yapılan hatalar

- **Sunucuyu `nbd` kullanıcısıyla çalıştırıp bir disk paylaşmak.** "Access denied by server configuration" hatası. `group = disk` yaz.
- **Sunucuda `[generic]` bölümünü silmek.**
- **Yapılandırmadan sonra `nbd-server`'ı yeniden başlatmamak.**
- **İstemcide `modprobe nbd` yapmamak.** `/dev/nbd0` yoktur.
- **`allowlist = true` olmadan `-l` ile listelemeye çalışmak.**
- **Bağlı dosya sistemini ayırmadan `nbd-client -d` yapmak.**
- **Aynı paylaşımı iki istemciye okuma-yazma ile bağlamak.**
- **fstab'da `_netdev`'i unutmak.**

---

## 8. Alıştırmalar

**1.** Sunucuda `/dev/vdc` diskini `veritabani` adıyla paylaş. İstemcilerin paylaşımları listeleyebilmesini sağla.

<details>
<summary>Çözüm</summary>

`/etc/nbd-server/config`:

```ini
[generic]
	user = nbd
	group = disk
	includedir = /etc/nbd-server/conf.d
	allowlist = true

[veritabani]
	exportname = /dev/vdc
```

```bash
sudo systemctl restart nbd-server
```
</details>

**2.** Bir arkadaşın sunucuyu kurmuş. `nbd-client -l` paylaşımı gösteriyor, ama bağlanmaya çalışınca "Access denied by server configuration" alıyor. `nbd` kullanıcısını `disk` grubuna eklemiş ama düzelmemiş. Ne yapmalı?

<details>
<summary>Çözüm</summary>

Sunucu ek grupları kullanmaz. `[generic]` içinde `group = nbd` satırını `group = disk` yapıp `sudo systemctl restart nbd-server` çalıştırmalı.
</details>

**3.** İstemcide sunucunun (`192.168.50.15`) paylaşımlarını listele, `veritabani`'na `/dev/nbd1` üzerinden bağlan.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install nbd-client
sudo modprobe nbd
nbd-client -l 192.168.50.15
sudo nbd-client 192.168.50.15 -N veritabani /dev/nbd1
lsblk /dev/nbd1
```
</details>

**4.** `/dev/nbd1`'i ext4 ile biçimlendir, `/mnt/vt`'ye bağla, sonra güvenli şekilde ayırıp bağlantıyı kes.

<details>
<summary>Çözüm</summary>

```bash
sudo mkfs.ext4 /dev/nbd1
sudo mkdir -p /mnt/vt
sudo mount /dev/nbd1 /mnt/vt
sudo umount /mnt/vt
sudo nbd-client -d /dev/nbd1
```
</details>

**5.** `nbd` modülünün açılışta yüklenmesini sağla.

<details>
<summary>Çözüm</summary>

```bash
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
```
</details>

---

Önceki konu: [NFS](07-nfs.md) · Sonraki konu: [Otomatik Bağlama](09-otomatik-baglama.md)
