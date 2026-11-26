# Dosya Sistemleri: XFS ve ext4

> **Resmi sınav maddesi:** Create, manage, and troubleshoot filesystems

Bir bölüm, üzerinde bir **dosya sistemi** oluşturulana kadar dosya saklayamaz. Dosya sistemi, verinin diskte nasıl düzenleneceğini belirler: dosya adları, dizinler, izinler, boş alanın takibi. Linux'ta en yaygın iki dosya sistemi **ext4** (Ubuntu'nun varsayılanı) ve **XFS** (Red Hat ailesinin varsayılanı) bu dosyanın konusu.

Senaryo: [Disk Bölümleri](01-disk-bolumleri.md) dosyasında oluşturduğumuz bölümlerden ilkine XFS (yedekler için), üçüncüsüne ext4 (veri için) oluşturacağız, etiket vereceğiz ve bilgilerini inceleyeceğiz.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde, test diskinin bölümlerinde (`/dev/loop0p1`, `/dev/loop0p3`) denenmiştir. XFS araçları için `sudo apt install xfsprogs` gerekebilir.

## İçindekiler

1. [ext4 mü, XFS mi?](#1-ext4-mü-xfs-mi)
2. [Önce kontrol et](#2-önce-kontrol-et)
3. [XFS oluşturmak: mkfs.xfs](#3-xfs-oluşturmak-mkfsxfs)
4. [XFS yönetimi: xfs_admin ve xfs_info](#4-xfs-yönetimi-xfs_admin-ve-xfs_info)
5. [ext4 oluşturmak: mkfs.ext4](#5-ext4-oluşturmak-mkfsext4)
6. [ext4 yönetimi: tune2fs ve e2label](#6-ext4-yönetimi-tune2fs-ve-e2label)
7. [Var olan dosya sisteminin üzerine yazmak](#7-var-olan-dosya-sisteminin-üzerine-yazmak)
8. [Dosya sistemini tanımak: blkid ve lsblk -f](#8-dosya-sistemini-tanımak-blkid-ve-lsblk--f)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. ext4 mü, XFS mi?

| | ext4 | XFS |
|---|---|---|
| Varsayılan olduğu yer | Ubuntu, Debian | RHEL, Rocky, Fedora Server |
| Büyütmek | Bağlıyken de olur (`resize2fs`) | Bağlıyken olur (`xfs_growfs`) |
| Küçültmek | Bağlı değilken olur | **Olmaz** |
| Inode sayısı | Oluştururken sabitlenir | İhtiyaca göre ayrılır |
| Etiket en fazla | 16 karakter (fazlası kesilir) | **12 karakter** (fazlası reddedilir) |
| Kontrol/onarım | `e2fsck` | `xfs_repair` |
| Yönetim aracı | `tune2fs` | `xfs_admin` |

İkisi de güvenilirdir. Genel kural: sistemin varsayılanını kullan, ileride küçültmen gerekebilecek bir bölümse ext4.

`mkfs.xfs` ve `mkfs.ext4`, `mkfs -t xfs` ve `mkfs -t ext4` ile aynıdır. `mkfs` "make file system" demektir.

---

## 2. Önce kontrol et

```bash
sudo fdisk -l /dev/loop0
lsblk -f /dev/loop0
```

Dosya sistemi oluşturmak, bölümdeki bütün veriyi siler. Doğru bölümde olduğundan, üzerinde işine yarar bir şey olmadığından ve bağlı olmadığından (`MOUNTPOINTS` boş) emin ol.

---

## 3. XFS oluşturmak: mkfs.xfs

```bash
sudo mkfs.xfs /dev/loop0p1
```
```
meta-data=/dev/loop0p1           isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
...
```

### Etiket: -L

```bash
sudo mkfs.xfs -L "Yedekler" /dev/loop0p1
```

XFS etiketi **en fazla 12 karakter** olabilir. 13 karakterlik bir etiket verdik:

```bash
sudo mkfs.xfs -f -L "YedeklerDiski" /dev/loop0p1
```
```
Invalid value YedeklerDiski for -L option
Usage: mkfs.xfs
```

Uzun bir kullanım listesi basıp hiçbir şey yapmadı. Uzun yardım çıktısında asıl hata mesajı ilk satırdadır.

### Diğer seçenekler

| Seçenek | Anlamı |
|---|---|
| `-L etiket` | Etiket |
| `-f` | Bölümde dosya sistemi olsa bile üzerine yaz ([7. bölüm](#7-var-olan-dosya-sisteminin-üzerine-yazmak)) |
| `-i size=512` | Inode boyutu (bayt). Güncel sürümlerde varsayılan zaten 512 |
| `-b size=4096` | Blok boyutu |

Hepsi birden:

```bash
sudo mkfs.xfs -f -i size=512 -L "Yedekler" /dev/loop0p1
```

Bütün seçenekler: `man mkfs.xfs`.

---

## 4. XFS yönetimi: xfs_admin ve xfs_info

`xfs_` ile başlayan araçlar XFS'e özgüdür. Terminalde `xfs_` yazıp Tab'a iki kez basarak hepsini görebilirsin.

```bash
sudo xfs_admin -l /dev/loop0p1          # etiketi göster (küçük l)
sudo xfs_admin -L "Arsiv" /dev/loop0p1  # etiketi değiştir (büyük L)
sudo xfs_admin -l /dev/loop0p1
```
```
label = "Yedekler"
writing all SBs
new label = "Arsiv"
label = "Arsiv"
```

Eski kaynaklarda etiket değiştirmek için dosya sisteminin bağlı olmaması gerektiği yazar. Ubuntu 24.04'teki `xfsprogs` sürümünde bağlı bir XFS'in etiketini de değiştirebildik. Diğer XFS araçları:

| Araç | Ne yapar? |
|---|---|
| `xfs_info /bağlama/noktası` | Bağlı bir XFS'in ayarları |
| `xfs_admin -u /dev/X` | UUID'yi göster |
| `xfs_growfs /bağlama/noktası` | Büyüt ([LVM](06-lvm.md)) |
| `xfs_repair /dev/X` | Kontrol ve onarım ([Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md#7-xfs-onarımı)) |

---

## 5. ext4 oluşturmak: mkfs.ext4

```bash
sudo mkfs.ext4 /dev/loop0p3
```
```
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 392704 4k blocks and 98304 inodes
Filesystem UUID: bed1a7c1-b08a-444c-9e35-c6e4cf84a2db
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376
Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
```

Çıktıdaki `Superblock backups` satırı, superblock bozulduğunda kullanılacak yedeklerin yerini gösterir ([Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md#6-bozuk-superblock)). `mkfs.ext4`, arka planda `mke2fs`'i çağırır, ayrıntılı seçenekler `man mke2fs`'te.

### Inode sayısı: -N ve -i

ext4'te her dosya bir inode kullanır ve inode sayısı oluştururken **sabitlenir**. Varsayılanla bu 1,5 GB'lık bölümde 98304 inode oluştu. Çok sayıda küçük dosya saklanacaksa (önbellek, mail kuyruğu) daha fazlası gerekir:

```bash
sudo mkfs.ext4 -N 500000 /dev/loop0p3
sudo tune2fs -l /dev/loop0p3 | grep 'Inode count'
```
```
Inode count:              500224
```

| Seçenek | Anlamı |
|---|---|
| `-N sayı` | En az bu kadar inode oluştur |
| `-i bayt` | Her bu kadar baytlık alan için bir inode (küçük değer = daha çok inode) |
| `-L etiket` | Etiket (en fazla 16 karakter) |
| `-m yüzde` | root'a ayrılan alan (varsayılan %5) |

Inode'lar bitince disk dolmadan `No space left on device` hatası alınır: [Disk Alanı Sorunları](../01-temel-komutlar/15-disk-alani-sorunlari.md#6-durum-3-yer-var-ama-dosya-oluşturulamıyor-inode).

---

## 6. ext4 yönetimi: tune2fs ve e2label

```bash
sudo tune2fs -l /dev/loop0p3           # bütün özellikler
sudo tune2fs -L "SecondFS" /dev/loop0p3   # etiket ver
sudo e2label /dev/loop0p3              # etiketi göster
```
```
Filesystem volume name:   <none>
Inode count:              500224
...
SecondFS
```

`tune2fs -l` çıktısında sık bakılanlar:

| Satır | Anlamı |
|---|---|
| `Filesystem volume name` | Etiket |
| `Filesystem UUID` | UUID |
| `Inode count`, `Free inodes` | Inode durumu |
| `Block count`, `Reserved block count` | Toplam ve root'a ayrılan bloklar |
| `Mount count`, `Last checked` | Kaç kez bağlandı, en son ne zaman kontrol edildi |

ext4 etiketi 16 karakteri geçerse hata vermez, sadece `Warning: label too long, truncating.` uyarısı yazıp **keser**: `tune2fs -L "OnAltiKarakterdenUzun"` sonrasında etiket `OnAltiKarakterde` oldu. Uyarı kolay gözden kaçar, etiketi `e2label` ile kontrol et.

Diğer `tune2fs` işlemleri: `-m 1` (ayrılan alanı %1 yap), `-c 30` (her 30 bağlamada bir kontrol), `-U random` (yeni UUID). Etiket ve bu ayarlar bağlıyken de değiştirilebilir.

---

## 7. Var olan dosya sisteminin üzerine yazmak

**XFS** her zaman durur:

```bash
sudo mkfs.xfs /dev/loop0p1
```
```
mkfs.xfs: /dev/loop0p1 appears to contain an existing filesystem (xfs).
mkfs.xfs: Use the -f option to force overwrite.
```

Bilerek üzerine yazmak için `-f`.

**ext4** ise kimin çalıştırdığına göre davranır. Terminalde (etkileşimli) çalıştırdığımızda sordu:

```
/dev/loop0p3 contains a ext4 file system labelled 'SecondFS'
	created on Thu Nov 26 09:29:55 2026
Proceed anyway? (y,N)
```

Ama aynı komutu bir script'ten, girdisi terminal olmadan çalıştırdığımızda **hiçbir şey sormadan** biçimlendirdi. Script'lerde `mkfs.ext4` kullanırken doğru cihazı verdiğinden kesinlikle emin ol. Terminalde soruyu atlamak için `-F`.

> [!CAUTION]
> Hem `-f` hem `-F`, bölümdeki dosya sistemini ve içindeki bütün veriyi geri dönüşsüz siler. Sadece neyin üzerine yazdığını bildiğinde kullan.

---

## 8. Dosya sistemini tanımak: blkid ve lsblk -f

```bash
sudo blkid /dev/loop0p1 /dev/loop0p3
```
```
/dev/loop0p1: LABEL="Arsiv" UUID="89160837-d655-4493-ac33-218754e1c96c" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="49f0b219-..."
/dev/loop0p3: LABEL="SecondFS" UUID="7c3ca0c1-bcdb-4f7a-b57c-66e4d736a745" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="4fba79e8-..."
```

| Alan | Anlamı |
|---|---|
| `LABEL` | Etiket |
| `UUID` | Dosya sisteminin benzersiz kimliği. Dosya sistemi yeniden oluşturulunca değişir |
| `TYPE` | Dosya sistemi türü |
| `PARTUUID` | Bölümün kimliği (GPT). Dosya sistemi değişse de aynı kalır |

Sadece UUID'yi almak, script'lerde fstab satırı yazarken işe yarar:

```bash
sudo blkid -s UUID -o value /dev/loop0p1
```

`lsblk -f` aynı bilgiyi ağaç halinde, bağlama noktalarıyla gösterir. Etiket ve UUID, diskleri fstab'da `/dev/sdb1` gibi değişebilen adlar yerine tanımlamak için kullanılır: [Mount ve fstab](03-mount-ve-fstab.md).

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sudo mkfs.xfs [-f] [-L etiket] [-i size=512] /dev/X` | XFS oluştur |
| `sudo xfs_admin -l` / `-L etiket` / `-u /dev/X` | XFS etiketini göster / değiştir, UUID |
| `xfs_info /bağlama` | Bağlı XFS'in bilgisi |
| `sudo mkfs.ext4 [-F] [-L etiket] [-N inode] /dev/X` | ext4 oluştur |
| `sudo tune2fs -l /dev/X` | ext4 bilgisi |
| `sudo tune2fs -L etiket /dev/X`, `e2label /dev/X` | ext4 etiketi |
| `sudo tune2fs -m 1 /dev/X` | root'a ayrılan alanı değiştir |
| `sudo blkid [-s UUID -o value] /dev/X` | Tür, etiket, UUID |
| `lsblk -f` | Bütün dosya sistemleri |

---

## 10. Sık yapılan hatalar

- **Yanlış bölümde `mkfs` çalıştırmak.** Önce `lsblk -f`.
- **XFS etiketine 12 karakterden uzun ad vermek.**
- **Script'lerde `mkfs.ext4`'ün onay soracağını sanmak.** Terminal yoksa sormaz.
- **`-f`/`-F`'yi alışkanlıkla her komuta eklemek.**
- **XFS'i küçültmeye çalışmak.** Mümkün değil; yedekle, yeniden oluştur, geri yükle.
- **Çok sayıda küçük dosya için varsayılan inode sayısıyla ext4 oluşturmak.**
- **Dosya sistemini yeniden oluşturduktan sonra fstab'daki eski UUID'yi güncellemeyi unutmak.**

---

## 11. Alıştırmalar

[Disk Bölümleri](01-disk-bolumleri.md) alıştırmalarındaki gibi iki bölümlü bir test diski hazırla (ör. `/dev/loop0p1` ve `/dev/loop0p2`).

**1.** 1. bölümde `Loglar` etiketli bir XFS oluştur, etiketini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo mkfs.xfs -L "Loglar" /dev/loop0p1
sudo xfs_admin -l /dev/loop0p1
```
</details>

**2.** Aynı bölümün etiketini `UygulamaLoglari` yapmayı dene. Neden olmuyor? Uygun bir etiket ver.

<details>
<summary>Çözüm</summary>

`UygulamaLoglari` 15 karakter, XFS en fazla 12 karakter kabul eder.

```bash
sudo xfs_admin -L "UygLoglari" /dev/loop0p1
sudo xfs_admin -l /dev/loop0p1
```
</details>

**3.** 2. bölümde, en az 200000 inode'u olan ve `Onbellek` etiketli bir ext4 oluştur. Inode sayısını ve etiketi doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkfs.ext4 -N 200000 -L "Onbellek" /dev/loop0p2
sudo tune2fs -l /dev/loop0p2 | grep -E 'Inode count|volume name'
```
</details>

**4.** İki bölümün türünü, etiketini ve UUID'sini tek komutla listele. Sonra sadece 1. bölümün UUID'sini yazdır.

<details>
<summary>Çözüm</summary>

```bash
sudo blkid /dev/loop0p1 /dev/loop0p2
sudo blkid -s UUID -o value /dev/loop0p1
```
</details>

**5.** 1. bölümde XFS varken `mkfs.xfs` ile yeniden oluşturmayı dene. Mesajı oku, sonra bilerek üzerine yaz. UUID değişti mi?

<details>
<summary>Çözüm</summary>

```bash
sudo blkid -s UUID -o value /dev/loop0p1
sudo mkfs.xfs /dev/loop0p1          # appears to contain an existing filesystem
sudo mkfs.xfs -f /dev/loop0p1
sudo blkid -s UUID -o value /dev/loop0p1    # farklı bir UUID
```
</details>

---

Önceki konu: [Disk Bölümleri](01-disk-bolumleri.md) · Sonraki konu: [Mount ve fstab](03-mount-ve-fstab.md)
