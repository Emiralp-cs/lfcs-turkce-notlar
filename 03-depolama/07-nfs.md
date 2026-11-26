# NFS

> **Resmi sınav maddesi:** Use remote filesystems and network block devices

**NFS** (Network File System), bir sunucudaki dizini ağ üzerinden başka makinelere açar. İstemci bu dizini kendi dosya sistemine bağlar ve yerel bir dizin gibi kullanır. Dosyalar sunucuda durur, istemciler aynı dosyaları görür.

Senaryo: Ofis sunucusu `10.0.5.10`. İki dizin paylaşılacak:

- `/srv/paylasim/belgeler`: Ofis ağındaki (`10.0.5.0/24`) herkes okuyup yazabilecek.
- `/srv/paylasim/yedek`: Sadece yedekleme sunucusu (`10.0.5.20`) yazabilecek ve orada root olarak çalışabilecek. Ofisteki diğer makineler sadece okuyabilecek.

> [!WARNING]
> Deneme ortamının çekirdeğinde **NFS desteği yoktu** (ne sunucu ne istemci). Paket kurulumu, `/etc/exports` dosyası ve `exportfs` komutları gerçekten denendi ve çıktılar oradan alındı. `exportfs` dosyayı çekirdek olmadan da okuyup doğruladığı için yazım hataları ve tuzaklar gerçek çıktılarla gösterildi. fstab satırlarının systemd tarafından nasıl işlendiği de denendi. **NFS sunucusunu başlatmak, `showmount` ve istemcide `mount` denenmedi.** Ortamda sunucu servisi şu hatayla başlamadı: `mount: /proc/fs/nfsd: unknown filesystem type 'nfsd'`.

## İçindekiler

1. [Sunucu: kurulum](#1-sunucu-kurulum)
2. [Sunucu: /etc/exports](#2-sunucu-etcexports)
3. [Seçenekler](#3-seçenekler)
4. [exportfs ile uygulamak](#4-exportfs-ile-uygulamak)
5. [Parantez önündeki boşluk tuzağı](#5-parantez-önündeki-boşluk-tuzağı)
6. [Paylaşımı bağlamak (istemci)](#6-paylaşımı-bağlamak-istemci)
7. [Kalıcı bağlama: fstab (istemci)](#7-kalıcı-bağlama-fstab-istemci)
8. [Güvenlik duvarı](#8-güvenlik-duvarı)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Sunucu: kurulum

```bash
sudo apt install nfs-kernel-server
sudo systemctl status nfs-server
```

Paket kurulunca `nfs-server` servisi etkinleşir. Paylaşılacak dizinleri oluştur:

```bash
sudo mkdir -p /srv/paylasim/belgeler /srv/paylasim/yedek
```

---

## 2. Sunucu: /etc/exports

Neyin, kime ve hangi seçeneklerle paylaşılacağı `/etc/exports` dosyasına yazılır. Her satır bir dizindir:

```
dizin   istemci1(seçenekler)   istemci2(seçenekler)
```

Senaryomuz:

```
# Ofis paylaşımları
/srv/paylasim/belgeler  10.0.5.0/24(rw,sync,no_subtree_check)
/srv/paylasim/yedek     10.0.5.20(rw,sync,no_root_squash,no_subtree_check) 10.0.5.0/24(ro,sync,no_subtree_check)
```

İkinci satırda aynı dizin iki istemciye farklı seçeneklerle açılıyor. 10.0.5.20 iki kurala da uyuyor: hem kendisi yazılmış, hem de 10.0.5.0/24 ağının içinde. `man exports`'a göre bu durumda satırdaki sıraya değil, istemci türüne bakılır. Önce tek makine, sonra ağ, sonra joker (`*`) kuralları gelir. Yani 10.0.5.20 için `rw,no_root_squash` geçerlidir.

İstemci şu şekillerde yazılabilir:

| Yazım | Anlamı |
|---|---|
| `10.0.5.20` | Tek bir IP |
| `10.0.5.0/24` | Bir ağ |
| `yedek.ofis.local` | Bir makine adı |
| `*.ofis.local` | Bir alan adındaki bütün makineler |
| `*` | Herkes |

---

## 3. Seçenekler

| Seçenek | Anlamı |
|---|---|
| `rw` | Okuma ve yazma |
| `ro` | Sadece okuma (**varsayılan**) |
| `sync` | Sunucu, veriyi diske yazdıktan sonra istemciye "tamam" der. Güvenli (**varsayılan**) |
| `async` | Veriyi diske yazmadan "tamam" der. Hızlı ama sunucu çökerse veri kaybolabilir |
| `root_squash` | İstemcideki root, sunucuda `nobody` (UID 65534) olur (**varsayılan**) |
| `no_root_squash` | İstemcideki root, sunucuda da root olur. Sadece güvendiğin makinelere ver |
| `all_squash` | Bütün kullanıcılar `nobody` olur |
| `subtree_check` | Paylaşılan dizinin, dosya sisteminin sadece bir alt dizini olduğu durumda ek kontrol yapar |
| `no_subtree_check` | Bu kontrolü kapatır. Önerilen ve varsayılan |

`root_squash`'ın sebebi: İstemcide root olan biri, sunucudaki dosyalarda da root yetkisine sahip olmamalı. Yedekleme sunucusunun dosya sahipliklerini koruyarak yedek alabilmesi için ona `no_root_squash` verdik.

`exportfs -v` hiç yazmadığın varsayılanları da gösterir. İlk satıra `ro` veya `root_squash` yazmadığımız halde çıktıda var:

```
/srv/paylasim/belgeler
		10.0.5.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,root_squash,no_all_squash)
```

---

## 4. exportfs ile uygulamak

`/etc/exports`'u değiştirdikten sonra servisi yeniden başlatmak gerekmez:

```bash
sudo exportfs -ra      # dosyayı yeniden oku, paylaşımları güncelle
sudo exportfs -v       # şu anki paylaşımlar ve seçenekleri
```
```
/srv/paylasim/yedek
		10.0.5.20(sync,wdelay,hide,no_subtree_check,sec=sys,rw,no_root_squash,no_all_squash)
/srv/paylasim/belgeler
		10.0.5.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,root_squash,no_all_squash)
/srv/paylasim/yedek
		10.0.5.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,ro,root_squash,no_all_squash)
```

`-r` (re-export) dosyayla eşitler: dosyadan silinen bir satır paylaşımdan da kalkar. `-a` bütün satırları uygular.

`exportfs` dosyadaki hataları gösterir. Var olmayan bir dizin ve yanlış yazılmış bir seçenek:

```
exportfs: Failed to stat /srv/olmayan: No such file or directory
exportfs: /etc/exports:1: unknown keyword "synk"
```

İkisinde de komut 1 ile çıkar. Seçenek yazmazsan uyarı verir ama paylaşır (varsayılanlarla, yani `ro`):

```
exportfs: No options for /srv/paylasim/belgeler 10.0.5.0/24: suggest 10.0.5.0/24(sync) to avoid warning
```

### Geçici paylaşım

Dosyaya yazmadan, yeniden başlatmaya kadar geçerli bir paylaşım:

```bash
sudo exportfs -o rw,sync,no_subtree_check 10.0.5.30:/srv/paylasim/yedek   # ekle
sudo exportfs -u 10.0.5.30:/srv/paylasim/yedek                             # kaldır
```

`exportfs -s` paylaşımları her biri tek satırda gösterir. Geçici paylaşım bir sonraki `exportfs -ra`'da da kalkar, çünkü dosyada yok (denendi).

### Sunucudan paylaşımları görmek

```bash
showmount -e 10.0.5.10
```

Bu komut bir sunucunun hangi dizinleri paylaştığını sorar. İstemciden de çalıştırılabilir.

---

## 5. Parantez önündeki boşluk tuzağı

`/etc/exports`'taki en tehlikeli hata, istemci ile parantez arasına boşluk koymaktır:

```
/srv/paylasim/belgeler  10.0.5.0/24 (rw,sync,no_subtree_check)
                                   ^ boşluk
```

Bu satır hata vermez. `exportfs -ra` sadece uyarı verir ve 0 ile çıkar:

```
exportfs: No options for /srv/paylasim/belgeler 10.0.5.0/24: suggest 10.0.5.0/24(sync) to avoid warning
exportfs: No host name given with /srv/paylasim/belgeler (rw,sync,no_subtree_check), suggest *(rw,sync,no_subtree_check) to avoid warning
```

Sonuç:

```
/srv/paylasim/belgeler
		10.0.5.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,ro,root_squash,no_all_squash)
/srv/paylasim/belgeler
		<world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,root_squash,no_all_squash)
```

`man exports` bunu açıkça yazar: istemci ile seçenek listesi arasında boşluk olamaz. Boşluk yüzünden satır iki ayrı kural olarak okundu: `10.0.5.0/24` seçeneksiz kaldı ve **sadece okuyabiliyor**, `(rw,...)` ise istemcisiz kaldı ve **herkese** (`<world>`) okuma-yazma verildi. İstediğinin tam tersi. Her değişiklikten sonra `exportfs -v` çıktısına bak.

---

## 6. Paylaşımı bağlamak (istemci)

```bash
sudo apt install nfs-common
sudo mkdir -p /mnt/belgeler
sudo mount 10.0.5.10:/srv/paylasim/belgeler /mnt/belgeler
df -h /mnt/belgeler
```

Kaynak `sunucu:dizin` biçimindedir. `mount` bunu görünce türü kendisi anlar, `-t nfs` yazmak şart değil. Ayırmak için:

```bash
sudo umount /mnt/belgeler
```

Bağlıyken `mount | grep nfs` veya `findmnt -t nfs4` bağlantının seçeneklerini gösterir. Ubuntu istemcisi varsayılan olarak NFS sürüm 4 kullanır, bu yüzden tür `nfs4` görünür.

---

## 7. Kalıcı bağlama: fstab (istemci)

```
10.0.5.10:/srv/paylasim/belgeler  /mnt/belgeler  nfs  defaults,_netdev  0  0
```

Son iki sütun `0 0` olmalı: ağ dosya sistemleri yedeklenmez (`dump`) ve `fsck` ile denetlenmez.

`_netdev`, bu bağlamanın ağ gerektirdiğini söyler. Ağ gelmeden bağlanmaya çalışılırsa açılış takılır. Denemede bu satırın iki halini ekleyip `systemctl daemon-reload` yaptık ve systemd'nin oluşturduğu mount birimine baktık:

```bash
sudo systemctl show mnt-belgeler.mount -p After -p RequiredBy
```
```
After=network-online.target network.target remote-fs-pre.target ...
RequiredBy=remote-fs.target
```

**`_netdev` olan ve olmayan satırda sonuç aynıydı.** systemd, türü `nfs` olan bir satırı zaten ağ dosya sistemi olarak tanır ve ağdan sonra bağlar. `_netdev` asıl olarak, systemd'nin ağ üzerinden geldiğini anlayamayacağı bağlamalar için gereklidir. Örneğin iSCSI veya NBD diski üzerindeki bir ext4 ([NBD](08-nbd.md)). NFS'te yazmak zarar vermez, okuyana niyetini de gösterir.

Satırı ekledikten sonra:

```bash
sudo mkdir -p /mnt/belgeler
sudo systemctl daemon-reload
sudo findmnt --verify
sudo mount -a
```

`findmnt --verify` NFS kaynaklarını denetleyemez ve bunu söyler: `do not check 10.0.5.10:/srv/paylasim/belgeler source (pseudo/net)`. Ama bağlama noktasının olup olmadığını denetler.

> [!TIP]
> Sunucu her zaman açık değilse, istemcinin her açılışta bağlamaya çalışması yerine dizine erişildiğinde bağlaması daha iyidir. Bunun için [Otomatik Bağlama](09-otomatik-baglama.md).

---

## 8. Güvenlik duvarı

NFSv4 tek bir port kullanır: **TCP 2049**. Sunucuda ufw açıksa:

```bash
sudo ufw allow from 10.0.5.0/24 to any port nfs
```

`nfs` adı `/etc/services`'ten 2049'a çevrilir. `ufw --dry-run` ile bakınca bu kuralın hem TCP hem UDP 2049 için izin eklediği görülür.

---

## 9. Komut özeti

| Komut | Nerede | Ne yapar? |
|---|---|---|
| `sudo apt install nfs-kernel-server` | Sunucu | Sunucu paketi |
| `sudo vi /etc/exports` | Sunucu | Paylaşımları tanımla |
| `sudo exportfs -ra` | Sunucu | Dosyayı uygula |
| `sudo exportfs -v` | Sunucu | Paylaşımlar ve bütün seçenekleri |
| `sudo exportfs -o seç istemci:dizin` | Sunucu | Geçici paylaşım |
| `sudo exportfs -u istemci:dizin` | Sunucu | Paylaşımı kaldır |
| `showmount -e sunucu` | İkisi de | Sunucunun paylaşımları |
| `sudo apt install nfs-common` | İstemci | İstemci paketi |
| `sudo mount sunucu:/dizin /bağlama` | İstemci | Bağla |
| `sunucu:/dizin /bağlama nfs defaults,_netdev 0 0` | İstemci | fstab satırı |

---

## 10. Sık yapılan hatalar

- **İstemci ile parantez arasına boşluk koymak.** Hata vermez, herkese yazma izni verir.
- **`rw` yazmayı unutmak.** Varsayılan `ro`'dur, istemciler yazamaz.
- **`/etc/exports`'u değiştirip `exportfs -ra` çalıştırmamak.**
- **İstemcide root olarak yazılan dosyaların sahibinin `nobody` olmasına şaşırmak.** `root_squash` varsayılandır.
- **`no_root_squash`'ı bütün ağa vermek.**
- **İstemcide `nfs-common` kurmamak.** `mount` NFS kaynağını bağlayamaz.
- **fstab'da son iki sütunu `0 0` dışında bırakmak.**
- **Sunucuda paylaşılan dizinin Linux izinlerini unutmak.** NFS `rw` verse bile, dizine yazma izni olmayan kullanıcı yazamaz.

---

## 11. Alıştırmalar

**1.** Sunucuda `/srv/ortak` dizinini `192.168.50.0/24` ağına okuma-yazma, senkron ve `no_subtree_check` ile paylaş. Paylaşımı uygula ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /srv/ortak
echo '/srv/ortak  192.168.50.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo exportfs -v
```
</details>

**2.** Aynı dizini `192.168.50.40` makinesine `no_root_squash` ile ek olarak aç. Diğerleri için ayar değişmesin.

<details>
<summary>Çözüm</summary>

Satırın sonuna ikinci istemciyi ekle:

```
/srv/ortak  192.168.50.0/24(rw,sync,no_subtree_check) 192.168.50.40(rw,sync,no_root_squash,no_subtree_check)
```

```bash
sudo exportfs -ra
sudo exportfs -v
```
</details>

**3.** Bir arkadaşın exports dosyasına şunu yazmış ve "istemciler yazamıyor, ama ağ dışındaki makineler yazabiliyor" diyor. Sorun ne?

```
/srv/ortak  192.168.50.0/24 (rw,sync,no_subtree_check)
```

<details>
<summary>Çözüm</summary>

`24` ile `(` arasındaki boşluk. Ağ seçeneksiz (yani `ro`) kaldı, seçenekler de istemcisiz kaldığı için herkese uygulandı. Boşluğu sil, `sudo exportfs -ra` ve `sudo exportfs -v` ile kontrol et.
</details>

**4.** İstemcide `192.168.50.10:/srv/ortak` paylaşımını `/mnt/ortak`'a bağla, sonra ayır.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install nfs-common
sudo mkdir -p /mnt/ortak
sudo mount 192.168.50.10:/srv/ortak /mnt/ortak
findmnt /mnt/ortak
sudo umount /mnt/ortak
```
</details>

**5.** Aynı paylaşımın açılışta bağlanmasını sağla.

<details>
<summary>Çözüm</summary>

```bash
echo '192.168.50.10:/srv/ortak  /mnt/ortak  nfs  defaults,_netdev  0  0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/ortak
```
</details>

---

Önceki konu: [LVM](06-lvm.md) · Sonraki konu: [NBD](08-nbd.md)
