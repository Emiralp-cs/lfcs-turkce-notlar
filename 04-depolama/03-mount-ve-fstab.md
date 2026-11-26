# Mount ve /etc/fstab

> **İlgili resmi sınav maddesi:** Create, manage, and troubleshoot filesystems

Linux'ta bütün dosyalar tek bir ağaçta (`/` altında) durur. Bir diskteki dosya sistemini kullanmak için onu bu ağaçta bir dizine **bağlarsın** (mount). O dizin, artık o diskin kökü gibi davranır. `mount` ile yapılan bağlama yeniden başlatınca kaybolur, kalıcı olması için `/etc/fstab` dosyasına yazılır.

Senaryo: [Dosya Sistemleri](02-dosya-sistemleri.md) dosyasında oluşturduğumuz XFS'i `/yedekler`'e, ext4'ü `/veri`'ye bağlayacağız. Önce elle, sonra açılışta kendiliğinden bağlanacak şekilde.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve systemd 255 ile, test diskinin bölümlerinde denenmiştir. Gerçekten yeniden başlatma yapılmadı. Açılıştaki davranış `findmnt --verify`, `systemctl daemon-reload` ve `mount -a` ile kontrol edildi.

## İçindekiler

1. [Elle bağlamak: mount](#1-elle-bağlamak-mount)
2. [Ayırmak: umount](#2-ayırmak-umount)
3. [Kalıcı bağlama: /etc/fstab](#3-kalıcı-bağlama-etcfstab)
4. [Cihazı tanımlamak: UUID ve LABEL](#4-cihazı-tanımlamak-uuid-ve-label)
5. [fstab değişikliğini uygulamak ve kontrol etmek](#5-fstab-değişikliğini-uygulamak-ve-kontrol-etmek)
6. [fstab'da swap](#6-fstabda-swap)
7. [Komut özeti](#7-komut-özeti)
8. [Sık yapılan hatalar](#8-sık-yapılan-hatalar)
9. [Alıştırmalar](#9-alıştırmalar)

---

## 1. Elle bağlamak: mount

```bash
sudo mkdir -p /yedekler                   # bağlama noktası (boş bir dizin)
sudo mount /dev/loop0p1 /yedekler
findmnt /yedekler
df -h /yedekler
```
```
TARGET    SOURCE       FSTYPE OPTIONS
/yedekler /dev/loop0p1 xfs    rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
/dev/loop0p1    960M   51M  910M   6% /yedekler
```

`mount` dosya sisteminin türünü kendisi bulur, `-t xfs` yazmak gerekmez.

> [!WARNING]
> Bağlama noktası **boş** olmalı. İçinde dosyalar olan bir dizine bağlarsan o dosyalar silinmez ama disk ayrılana kadar **görünmez** olur. Bu, "dosyalarım kayboldu" ve "disk dolu görünüyor ama dosya yok" sorunlarına yol açar ([Disk Alanı Sorunları](../01-temel-komutlar/15-disk-alani-sorunlari.md#8-durum-5-df-ile-du-birbirini-tutmuyor-gizli-veri)). Bağlamadan önce `ls -A /yedekler` ile bak.

Neyin nereye bağlı olduğunu görmek:

```bash
findmnt                        # ağaç halinde, hepsi
findmnt -t xfs,ext4            # sadece bu türler
lsblk -f                       # disk bazında
mount | grep loop0             # eski yöntem
```

---

## 2. Ayırmak: umount

```bash
sudo umount /yedekler          # bağlama noktasıyla
sudo umount /dev/loop0p1       # veya cihazla
```

Komutun adı **umount**'tur, `unmount` değil (`unmount: command not found`).

İçinde bulunduğun veya bir sürecin kullandığı bir dosya sistemi ayrılamaz:

```bash
cd /yedekler
sudo umount /yedekler
```
```
umount: /yedekler: target is busy.
```

Önce oradan çık (`cd /`). Başka bir süreç kullanıyorsa onu bul: `sudo lsof +D /yedekler` veya `sudo fuser -vm /yedekler` ([Depolama Performansı](11-depolama-performansi.md)).

---

## 3. Kalıcı bağlama: /etc/fstab

`/etc/fstab` dosyasındaki her satır, açılışta bağlanacak bir dosya sistemidir. Her satır boşlukla ayrılmış 6 alandan oluşur:

```
UUID=89160837-d655-4493-ac33-218754e1c96c  /yedekler  xfs   defaults  0  2
LABEL=SecondFS                             /veri      ext4  defaults  0  2
```

| # | Alan | Anlamı | Örnek |
|---|---|---|---|
| 1 | Cihaz | Hangi dosya sistemi | `UUID=...`, `LABEL=...`, `/dev/sdb1` |
| 2 | Bağlama noktası | Hangi dizine | `/yedekler` (swap için `none`) |
| 3 | Tür | Dosya sistemi türü | `xfs`, `ext4`, `swap`, `nfs` |
| 4 | Seçenekler | Nasıl bağlanacak | `defaults`, `ro,noexec` ([Mount Seçenekleri](04-mount-secenekleri.md)) |
| 5 | Dump | Eski `dump` yedekleme aracı için | Neredeyse her zaman `0` |
| 6 | Pass | Açılışta `fsck` sırası | `1` kök dizin, `2` diğerleri, `0` kontrol etme |

Kök (`/`) için 6. alan `1`, diğer ext4 diskleri için `2`, swap ve ağ dosya sistemleri için `0`. XFS'te `fsck` hiçbir şey yapmadığı için `0` veya `2` fark etmez.

> [!CAUTION]
> `/etc/fstab`'daki hatalı bir satır, sistemin açılmamasına ve emergency moduna düşmesine yol açabilir ([Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md#4-hatalı-fstab-yüzünden-açılmayan-sistem)). Düzenlemeden önce bir yedek al: `sudo cp /etc/fstab /etc/fstab.yedek`. Açılış için şart olmayan diskler için seçeneklere `nofail` ekle.

---

## 4. Cihazı tanımlamak: UUID ve LABEL

1. alana `/dev/sdb1` yazmak çalışır, ama disk adları takılma sırasına göre değişebilir: başka bir disk eklenince dün `/dev/sdb` olan disk bugün `/dev/sdc` olabilir ve fstab yanlış diski bağlar veya hiç bağlayamaz. Bunun yerine dosya sisteminin değişmeyen kimliği kullanılır.

```bash
sudo blkid /dev/loop0p1
```
```
/dev/loop0p1: LABEL="Arsiv" UUID="89160837-d655-4493-ac33-218754e1c96c" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="..."
```

| fstab'da | Ne zaman değişir? |
|---|---|
| `UUID=89160837-...` | Dosya sistemi yeniden oluşturulursa (`mkfs`) |
| `LABEL=Arsiv` | Etiket değiştirilirse. Aynı etiketli iki disk varsa karışır |
| `/dev/sdb1` | Diskler farklı sırayla algılanırsa |

UUID'ler `/dev/disk/by-uuid/` altında, cihazlara bağlantı olarak da görünür:

```bash
ls -l /dev/disk/by-uuid/
```
```
89160837-d655-4493-ac33-218754e1c96c -> ../../loop0p1
7c3ca0c1-bcdb-4f7a-b57c-66e4d736a745 -> ../../loop0p3
```

Etiketler için `/dev/disk/by-label/`. UUID'yi elle kopyalarken yanlış yazmamak için: `echo "UUID=$(sudo blkid -s UUID -o value /dev/loop0p1) /yedekler xfs defaults 0 2" | sudo tee -a /etc/fstab`.

---

## 5. fstab değişikliğini uygulamak ve kontrol etmek

Ubuntu'da açılışta fstab'ı systemd okur ve her satır için bir `.mount` unit'i oluşturur. Satırları ekledikten sonra:

**1. Kontrol et:**

```bash
sudo findmnt --verify
```

Satırları ekleyip başka bir şey yapmadan çalıştırdığımızda:

```
0 parse errors, 0 errors, 1 warning
   [W] your fstab has been modified, but systemd still uses the old version;
       use 'systemctl daemon-reload' to reload
```

**2. systemd'ye bildir:**

```bash
sudo systemctl daemon-reload
```

**3. Bağla:**

```bash
sudo mount -a
```

`mount -a`, fstab'daki henüz bağlı olmayan her şeyi bağlamayı dener. Hata vermezse açılışta da sorun çıkmaz. Sadece belirli bir satırı denemek için bağlama noktasını vermek yeterli, mount gerisini fstab'dan okur:

```bash
sudo mount /veri
```

**4. Doğrula:**

```bash
findmnt -t xfs,ext4
systemctl list-units --type=mount | grep -E 'yedekler|veri'
```
```
/yedekler        /dev/loop0p1 xfs    rw,relatime,...
/veri            /dev/loop0p3 ext4   rw,relatime
  veri.mount        loaded active mounted /veri
  yedekler.mount    loaded active mounted /yedekler
```

Her fstab satırı bir systemd unit'i olarak da görünür. Adı bağlama noktasından gelir (`/veri` → `veri.mount`, `/srv/web` → `srv-web.mount`).

---

## 6. fstab'da swap

Swap alanları da fstab'da tanımlanır. Bağlama noktası `none`, tür `swap`:

```
UUID=51ef8613-46e4-465f-ae9e-61dd0b293963  none  swap  defaults  0  0
/swapfile                                  none  swap  pri=10    0  0
```

| Alan | Swap'ta |
|---|---|
| 2. Bağlama noktası | `none`, çünkü bir dizine bağlanmaz |
| 3. Tür | `swap` |
| 5. ve 6. | Her zaman `0 0` |

Swap dosyaları için UUID yerine dosyanın yolu yazılır. Uygulamak için `mount -a` değil, `sudo swapon -a`. Ayrıntılar: [Swap](05-swap.md).

---

## 7. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sudo mount /dev/X /dizin` | Bağla |
| `sudo umount /dizin` (veya `/dev/X`) | Ayır |
| `findmnt`, `findmnt -t xfs,ext4`, `findmnt /dizin` | Bağlı dosya sistemleri |
| `lsblk -f` | Diskler ve bağlama noktaları |
| `sudo blkid [-s UUID -o value] /dev/X` | UUID, etiket, tür |
| `ls -l /dev/disk/by-uuid/` | UUID → cihaz |
| `sudo cp /etc/fstab /etc/fstab.yedek` | Düzenlemeden önce yedek |
| `sudo findmnt --verify` | fstab'ı kontrol et |
| `sudo systemctl daemon-reload` | fstab değişikliğini systemd'ye bildir |
| `sudo mount -a` | fstab'daki her şeyi bağla |
| `sudo mount /dizin` | fstab'daki tek satırı bağla |
| `sudo swapon -a` | fstab'daki swap'ları aç |
| `sudo lsof +D /dizin`, `fuser -vm /dizin` | `target is busy` sebebi |

---

## 8. Sık yapılan hatalar

- **`unmount` yazmak.** Doğrusu `umount`.
- **İçi dolu bir dizine bağlamak.** Eski dosyalar görünmez olur.
- **Bağlama noktası dizinini oluşturmadan fstab'a yazmak.** `mount -a` ve açılış hata verir.
- **fstab'da `/dev/sdb1` gibi değişebilen adlar kullanmak.** UUID kullan.
- **fstab'ı değiştirip `mount -a` ile denemeden yeniden başlatmak.**
- **Disk yeniden biçimlendirildikten sonra fstab'daki eski UUID'yi güncellememek.**
- **Swap satırında bağlama noktasına bir dizin yazmak.** `none` olmalı.
- **Açılış için şart olmayan disklere `nofail` eklememek.**
- **Bulunduğun dizindeki diski ayırmaya çalışmak.** `target is busy`.

---

## 9. Alıştırmalar

[Dosya Sistemleri](02-dosya-sistemleri.md) alıştırmalarındaki gibi bir XFS (`/dev/loop0p1`) ve bir ext4 (`/dev/loop0p2`) bölümün olsun.

**1.** XFS bölümünü `/mnt/loglar`'a elle bağla, içine bir dosya yaz, bağlandığını iki farklı komutla göster, sonra ayır.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /mnt/loglar
sudo mount /dev/loop0p1 /mnt/loglar
sudo touch /mnt/loglar/deneme
findmnt /mnt/loglar
df -h /mnt/loglar
sudo umount /mnt/loglar
```
</details>

**2.** ext4 bölümünün `/onbellek`'e açılışta kendiliğinden bağlanmasını sağla. Cihazı UUID ile tanımla. Yeniden başlatmadan bağla ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo cp /etc/fstab /etc/fstab.yedek
sudo mkdir -p /onbellek
echo "UUID=$(sudo blkid -s UUID -o value /dev/loop0p2) /onbellek ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo findmnt --verify
sudo systemctl daemon-reload
sudo mount -a
findmnt /onbellek
```
</details>

**3.** `/onbellek`'e girip (`cd`) ayırmayı dene. Hatayı gör ve çöz.

<details>
<summary>Çözüm</summary>

```bash
cd /onbellek
sudo umount /onbellek      # target is busy
cd /
sudo umount /onbellek
```
</details>

**4.** XFS bölümü için, UUID yerine etiketini kullanan bir fstab satırı ekle (`/mnt/loglar`'a bağlansın). Önce etiketi olup olmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo blkid /dev/loop0p1                  # LABEL var mı?
sudo xfs_admin -L "Loglar" /dev/loop0p1  # yoksa ver
echo "LABEL=Loglar /mnt/loglar xfs defaults 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount /mnt/loglar
findmnt /mnt/loglar
```
</details>

**5.** `/etc/fstab`'a bilerek hatalı bir satır ekle (`UUID=11111111-1111-1111-1111-111111111111 /yok ext4 defaults 0 2`). Yeniden başlatmadan hatayı bul, sonra satırı kaldır.

<details>
<summary>Çözüm</summary>

```bash
echo "UUID=11111111-1111-1111-1111-111111111111 /yok ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo findmnt --verify          # hem kaynak hem hedef bulunamıyor
sudo sed -i '/11111111-1111/d' /etc/fstab
sudo findmnt --verify
```
</details>

Bitirince: eklediğin fstab satırlarını sil (`sudo cp /etc/fstab.yedek /etc/fstab`), diskleri ayır, `sudo systemctl daemon-reload`.

---

Önceki konu: [Dosya Sistemleri](02-dosya-sistemleri.md) · Sonraki konu: [Mount Seçenekleri](04-mount-secenekleri.md)
