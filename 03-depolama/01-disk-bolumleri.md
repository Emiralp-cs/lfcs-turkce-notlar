# Disk Bölümleri

> **İlgili resmi sınav maddesi:** Create, manage, and troubleshoot filesystems

Yeni bir disk sunucuya takıldığında doğrudan kullanılamaz: önce **bölümlere** (partition) ayrılır, sonra her bölüme bir dosya sistemi oluşturulur ([Dosya Sistemleri](02-dosya-sistemleri.md)), en son bir dizine bağlanır ([Mount ve fstab](03-mount-ve-fstab.md)). Bu dosya ilk adımı anlatıyor: diskleri görmek, bölüm tablosu oluşturmak, bölüm eklemek, silmek ve türünü değiştirmek.

Senaryo: Sunucuya 3 GB'lık yeni bir disk eklendi. Üzerinde 1 GB'lık bir yedek bölümü, 512 MB'lık bir swap bölümü ve geri kalanında bir veri bölümü oluşturacağız.

> [!NOTE]
> Örnekler Ubuntu 24.04 üzerinde, dosyadan oluşturulmuş disklerde (loop cihazı: `/dev/loop0`) denenmiştir. Gerçek bir sunucuda disk adları `/dev/sdb`, `/dev/vdb` veya `/dev/nvme1n1` gibi olur, komutlar aynıdır. `cfdisk`'in etkileşimli ekranı denenmedi, aynı işlemler `fdisk`, `sfdisk` ve `parted` ile denendi.

## İçindekiler

1. [Diskleri görmek: lsblk](#1-diskleri-görmek-lsblk)
2. [Disk ve bölüm adları](#2-disk-ve-bölüm-adları)
3. [Bölüm tablosunu görmek: fdisk -l](#3-bölüm-tablosunu-görmek-fdisk--l)
4. [GPT ve MBR](#4-gpt-ve-mbr)
5. [Bölüm oluşturmak: fdisk](#5-bölüm-oluşturmak-fdisk)
6. [Menüyle bölümlemek: cfdisk](#6-menüyle-bölümlemek-cfdisk)
7. [Tek komutla: sfdisk ve parted](#7-tek-komutla-sfdisk-ve-parted)
8. [Bölüm silmek ve türünü değiştirmek](#8-bölüm-silmek-ve-türünü-değiştirmek)
9. [Çekirdek yeni bölümü görmüyorsa](#9-çekirdek-yeni-bölümü-görmüyorsa)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Diskleri görmek: lsblk

```bash
lsblk
```

Bölümleme bittikten sonra test diskimiz:

```
NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0       7:0    0    3G  0 loop
|-loop0p1 259:0    0    1G  0 part
|-loop0p2 259:1    0  512M  0 part
`-loop0p3 259:2    0  1.5G  0 part
```

| Sütun | Anlamı |
|---|---|
| `NAME` | Cihaz adı. Girintili olanlar üstteki diskin bölümleri |
| `SIZE` | Boyut |
| `TYPE` | `disk`, `part` (bölüm), `lvm`, `rom`, `loop` |
| `MOUNTPOINTS` | Bağlı olduğu dizin, boşsa bağlı değil |

```bash
lsblk -f        # dosya sistemi türü, etiket, UUID
```
```
NAME      FSTYPE FSVER LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
|-loop0p1 xfs          Arsiv    89160837-d655-4493-ac33-218754e1c96c
|-loop0p2
`-loop0p3 ext4   1.0   SecondFS 7c3ca0c1-bcdb-4f7a-b57c-66e4d736a745
```

Yeni takılan bir diski bulmanın en kolay yolu: `lsblk`'te bölümü ve `MOUNTPOINTS`'i olmayan disk.

---

## 2. Disk ve bölüm adları

Her disk ve bölüm, `/dev/` altında bir dosyadır:

| Disk türü | Disk | Bölümleri |
|---|---|---|
| SATA, SAS, USB | `/dev/sda`, `/dev/sdb` | `/dev/sdb1`, `/dev/sdb2` |
| Sanal makinede virtio | `/dev/vda`, `/dev/vdb` | `/dev/vdb1`, `/dev/vdb2` |
| NVMe | `/dev/nvme0n1`, `/dev/nvme1n1` | `/dev/nvme1n1p1`, `/dev/nvme1n1p2` |
| Loop (dosyadan disk) | `/dev/loop0` | `/dev/loop0p1` |

Adı bir rakamla biten disklerde (`nvme0n1`, `loop0`) bölüm numarasının önüne `p` gelir.

> [!WARNING]
> `sda`, `sdb` gibi adlar diskin takılma sırasına göre verilir ve yeniden başlatınca değişebilir. Komutları çalıştırmadan önce doğru diskte olduğundan `lsblk` ile emin ol: yanlış diski bölümlemek üzerindeki bütün veriyi siler. Kalıcı yapılandırmalarda (fstab) ad yerine UUID kullanılır.

---

## 3. Bölüm tablosunu görmek: fdisk -l

```bash
sudo fdisk -l /dev/loop0      # tek disk
sudo fdisk -l                 # bütün diskler
```
```
Disklabel type: gpt
Disk identifier: B973FBAE-DE98-46E7-BF47-CC935884EBD5

Device         Start     End Sectors  Size Type
/dev/loop0p1    2048 2099199 2097152    1G Linux filesystem
/dev/loop0p2 2099200 3147775 1048576  512M Linux swap
/dev/loop0p3 3147776 6289407 3141632  1.5G Linux filesystem
```

| Sütun | Anlamı |
|---|---|
| `Start`, `End` | Bölümün başladığı ve bittiği sektör |
| `Sectors` | Sektör sayısı. Bir sektör 512 bayt |
| `Size` | Boyut |
| `Type` | Bölüm türü (Linux filesystem, Linux swap, EFI System, Linux LVM ...) |

İlk bölüm 2048. sektörde başlar: 2048 × 512 bayt = 1 MiB. Bu boşluk, bölümlerin diskin fiziksel yapısıyla hizalı olması içindir, araçlar bunu kendileri ayarlar.

---

## 4. GPT ve MBR

Bölüm tablosunun iki türü var:

| | GPT | MBR (DOS) |
|---|---|---|
| En büyük disk | Pratikte sınırsız | 2 TB |
| Bölüm sayısı | 128 | 4 birincil (fazlası için "extended" bölüm) |
| Yedek tablo | Diskin sonunda yedeği var | Yok |
| Kullanım | Modern sistemler, UEFI | Eski sistemler, BIOS |

Yeni disklerde **GPT** kullan. `fdisk`, tablosu olmayan bir diskte varsayılan olarak MBR oluşturur:

```
Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x1befd821.
```

GPT için önce `g` komutu verilmeli ([5. bölüm](#5-bölüm-oluşturmak-fdisk)).

---

## 5. Bölüm oluşturmak: fdisk

`fdisk` tek harfli komutlarla çalışan etkileşimli bir araçtır. **`w` yazana kadar diske hiçbir şey yazılmaz**, `q` ile her şeyden vazgeçip çıkabilirsin.

```bash
sudo fdisk /dev/loop1
```

| Komut | Ne yapar? |
|---|---|
| `m` | Yardım |
| `p` | Bölüm tablosunu göster |
| `g` | Yeni, boş bir **GPT** tablosu oluştur |
| `o` | Yeni, boş bir MBR tablosu oluştur |
| `n` | Yeni bölüm |
| `d` | Bölüm sil |
| `t` | Bölüm türünü değiştir |
| `l` | Bölüm türlerini listele (`t` içindeyken `L`) |
| `w` | Değişiklikleri yaz ve çık |
| `q` | Yazmadan çık |

Örnek oturum: GPT tablosu, 300 MB'lık bir bölüm ve diskin geri kalanında bir swap bölümü.

```
Command (m for help): g
Created a new GPT disklabel (GUID: F2DB8144-2B07-4449-80D3-E053B3422A5C).

Command (m for help): n
Partition number (1-128, default 1): <Enter>
First sector (2048-2097118, default 2048): <Enter>
Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): +300M
Created a new partition 1 of type 'Linux filesystem' and of size 300 MiB.

Command (m for help): n
Partition number (2-128, default 2): <Enter>
First sector (...): <Enter>
Last sector (...): <Enter>
Created a new partition 2 of type 'Linux filesystem' and of size 722 MiB.

Command (m for help): t
Partition number (1,2, default 2): 2
Partition type or alias (type L to list all): swap
Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): p
Device        Start     End Sectors  Size Type
/dev/loop1p1   2048  616447  614400  300M Linux filesystem
/dev/loop1p2 616448 2095103 1478656  722M Linux swap

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

- `Last sector` sorusuna `+300M` gibi boyut yazılır. Enter, kalan alanın tamamını verir.
- Bölüm türü numarayla veya takma adla verilebilir: `swap`, `linux`, `lvm`, `uefi`. Numaralar GPT ve MBR'de farklıdır, takma ad kullanmak daha güvenlidir.

Bölüm oluşturmak dosya sistemi oluşturmaz. Sıradaki adım: [Dosya Sistemleri](02-dosya-sistemleri.md).

---

## 6. Menüyle bölümlemek: cfdisk

`cfdisk` aynı işi ok tuşlarıyla gezilen bir menüyle yapar, `fdisk`'in komutlarını ezberlemek istemeyenler için kolaydır:

```bash
sudo cfdisk /dev/sdb
```

1. Tablosu olmayan bir diskte önce tablo türünü sorar: **gpt** seç.
2. Boş alanın üzerinde **[ New ]**, boyut yaz (`1G`), Enter.
3. Boş alana inip aynısını tekrarla.
4. Bir bölümün türünü değiştirmek için bölümü seç, **[ Type ]**, listeden seç (örneğin `Linux swap`).
5. **[ Write ]**, onay için `yes` yaz.
6. **[ Quit ]**.

| Menü | `fdisk` karşılığı |
|---|---|
| New | `n` |
| Delete | `d` |
| Type | `t` |
| Resize | (fdisk'te doğrudan yok) |
| Sort | Bölüm numaralarını diskteki sıraya göre yeniden düzenler |
| Write | `w` |
| Quit | `q` |

Bölüm silip yeniden oluşturduğunda numaralar diskteki fiziksel sırayla uyuşmayabilir (2. bölüm diskte 3.'nün arkasında olabilir). **Sort** bunu düzeltir. `fdisk`'te de `x` (uzman modu) içinde `f` aynı işi yapar.

---

## 7. Tek komutla: sfdisk ve parted

Script'lerde veya hızlı kurulumda etkileşimli araç yerine bölüm düzeni tek seferde verilebilir.

**sfdisk:** Düzen, satır satır stdin'den okunur. Senaryodaki diski böyle bölümledik:

```bash
sudo sfdisk /dev/loop0 <<'EOF'
label: gpt
size=1GiB, type=linux
size=512MiB, type=swap
type=linux
EOF
```
```
/dev/loop0p2 2099200 3147775 1048576  512M Linux swap
/dev/loop0p3 3147776 6289407 3141632  1.5G Linux filesystem

The partition table has been altered.
```

Boyutu verilmeyen son bölüm kalan alanın tamamını alır. Var olan bir diskin düzenini dosyaya kaydedip başka bir diske aynen uygulamak için: `sfdisk -d /dev/sdb > duzen.txt` ve `sfdisk /dev/sdc < duzen.txt`.

**parted:**

```bash
sudo parted -s /dev/loop1 print
sudo parted -s /dev/loop1 mkpart veri ext4 301MiB 100%
```
```
Number  Start   End     Size   File system  Name  Flags
 1      1049kB  316MB   315MB
 2      316MB   1073MB  757MB               veri
```

`-s` soru sormadan çalıştırır. `parted`, `fdisk`'in aksine her komutu **anında** diske yazar, "kaydetmeden çık" seçeneği yoktur.

---

## 8. Bölüm silmek ve türünü değiştirmek

`fdisk` ile:

```
Command (m for help): d
Partition number (1,2, default 2): 2
Partition 2 has been deleted.

Command (m for help): w
```

Bölümü silmeden önce bağlıysa ayır (`umount`), swap ise kapat (`swapoff`), `/etc/fstab`'daki satırını kaldır. Bölüm silmek içindeki veriyi hemen silmez ama tabloya artık yazılmadığı için erişilemez hale getirir.

Türü değiştirmek (`t`) sadece tablodaki etiketi değiştirir, içindeki veriye veya dosya sistemine dokunmaz. Linux, bölümü kullanırken türüne pek bakmaz (swap bölümü `Linux filesystem` türünde de çalışır), ama doğru tür, diski inceleyen insanlar ve bazı araçlar (`lsblk`, kurulum programları) için önemlidir.

---

## 9. Çekirdek yeni bölümü görmüyorsa

`fdisk` `w` ile çıkarken çekirdeğe bölüm tablosunu yeniden okumasını söyler. Disk kullanımdaysa (bölümlerinden biri bağlıysa) bu başarısız olabilir ve yeni bölüm `lsblk`'te görünmez:

```bash
sudo partprobe /dev/sdb        # çekirdeğe tabloyu yeniden okut
sudo partx -u /dev/sdb         # alternatif
```

Bu notun denendiği ortamda da tablo yazıldıktan sonra bölüm cihazları (`/dev/loop0p1`) oluşmadı, `partx -a /dev/loop0` ile eklendi. Gerçek sunucularda `partprobe` genelde yeterlidir, olmazsa yeniden başlatmak kesin çözümdür.

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `lsblk`, `lsblk -f` | Diskler ve bölümler, dosya sistemleriyle |
| `sudo fdisk -l [/dev/X]` | Bölüm tablosu |
| `sudo fdisk /dev/X` | Etkileşimli bölümleme (`g`, `n`, `d`, `t`, `p`, `w`, `q`) |
| `sudo cfdisk /dev/X` | Menüyle bölümleme |
| `sudo sfdisk /dev/X < duzen` | Düzeni tek seferde uygula |
| `sudo sfdisk -d /dev/X` | Düzeni dök |
| `sudo parted -s /dev/X print` | Tabloyu göster |
| `sudo parted -s /dev/X mkpart ad tür başlangıç bitiş` | Bölüm ekle (anında yazar) |
| `sudo partprobe /dev/X` | Çekirdeğe tabloyu yeniden okut |

---

## 11. Sık yapılan hatalar

- **Yanlış diski bölümlemek.** Her zaman önce `lsblk`.
- **`fdisk`'te `w` demeden çıkıp değişikliklerin kaydedildiğini sanmak.**
- **Yeni diskte `g` demeden bölüm oluşturmak.** MBR tablosu oluşur.
- **`parted`'ın da `fdisk` gibi yazmadan önce beklediğini sanmak.**
- **Bölüm oluşturduktan sonra dosya sistemi oluşturmayı unutmak.**
- **Bağlı bir bölümü silmek veya küçültmek.**
- **Yeni bölüm `lsblk`'te görünmüyor diye tekrar oluşturmaya çalışmak.** Önce `partprobe`.

---

## 12. Alıştırmalar

Gerçek disklere dokunmamak için bir test diski oluştur:

```bash
truncate -s 2G ~/test-disk.img
sudo losetup -fP --show ~/test-disk.img      # ör. /dev/loop0
```

**1.** Test diskini ve varsa bölümlerini listele, bölüm tablosu olup olmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
lsblk /dev/loop0
sudo fdisk -l /dev/loop0     # tablo yoksa "Disklabel type" satırı olmaz
```
</details>

**2.** `fdisk` ile test diskinde GPT tablosu oluştur ve 500 MB'lık bir Linux bölümü ekle.

<details>
<summary>Çözüm</summary>

```bash
sudo fdisk /dev/loop0
# g, n, Enter, Enter, +500M, w
lsblk /dev/loop0
```
</details>

**3.** Aynı diske 256 MB'lık bir bölüm daha ekle ve türünü Linux swap yap.

<details>
<summary>Çözüm</summary>

```bash
sudo fdisk /dev/loop0
# n, Enter, Enter, +256M
# t, 2, swap
# p (kontrol), w
sudo fdisk -l /dev/loop0
```
</details>

**4.** 2. bölümü sil, kalan alanın tamamını kullanan yeni bir bölüm oluştur. Bunu `parted` ile yap.

<details>
<summary>Çözüm</summary>

```bash
sudo parted -s /dev/loop0 rm 2
sudo parted -s /dev/loop0 mkpart veri ext4 501MiB 100%
sudo parted -s /dev/loop0 print
```
</details>

**5.** Test diskinin bölüm düzenini bir dosyaya kaydet, başka bir test diskine aynen uygula.

<details>
<summary>Çözüm</summary>

```bash
sudo sfdisk -d /dev/loop0 > duzen.txt
truncate -s 2G ~/test-disk2.img
sudo losetup -fP --show ~/test-disk2.img      # ör. /dev/loop1
sudo sfdisk /dev/loop1 < duzen.txt
lsblk /dev/loop1
```
</details>

Bitirince: `sudo losetup -D` ve `rm ~/test-disk*.img`.

---

Sonraki konu: [Dosya Sistemleri](02-dosya-sistemleri.md)
