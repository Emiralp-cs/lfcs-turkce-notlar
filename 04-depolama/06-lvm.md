# LVM

> **Resmi sınav maddesi:** Configure and manage LVM storage

Normal bölümlerin boyutu disk üzerinde sabittir: bir bölüm dolduğunda, arkasında boş yer yoksa büyütemezsin. **LVM** (Logical Volume Manager) diskleri bir havuzda toplar ve bu havuzdan istediğin boyutta, sonradan büyütülüp küçültülebilen **mantıksal birimler** ayırır. Birden fazla diski tek bir büyük birim gibi kullanmak, sistemi kapatmadan disk eklemek ve birimleri büyütmek LVM ile kolaylaşır.

| Kavram | Kısaltma | Ne? |
|---|---|---|
| Physical Volume | **PV** | LVM'e verilen disk veya bölüm |
| Volume Group | **VG** | PV'lerden oluşan havuz |
| Logical Volume | **LV** | VG'den ayrılan, dosya sistemi oluşturulan birim |
| Physical Extent | **PE** | VG'nin bölündüğü küçük parçalar (varsayılan 4 MiB). LV'ler bu parçalardan oluşur |

```
 /dev/loop2   /dev/loop3        ← disk veya bölümler
     │            │
    PV           PV             ← pvcreate
      \          /
       veri_vg                  ← vgcreate (havuz: 2 GB)
      /          \
 yedek_lv      web_lv           ← lvcreate (istediğin boyutta)
     │            │
  mkfs.xfs     mkfs.ext4        ← dosya sistemi
     │            │
 /yedekler     /srv/web         ← mount
```

Senaryo: Sunucuya iki adet 1 GB'lık disk eklendi. Bunları tek bir havuzda toplayıp yedekler ve web dosyaları için iki birim ayıracağız. Sonra havuza üçüncü bir disk ekleyip birimleri büyüteceğiz.

> [!WARNING]
> Deneme ortamının çekirdeğinde **device-mapper yoktu.** PV, VG ve LV oluşturma, büyütme, küçültme ve silme komutları LVM'in sadece meta veriyi değiştiren modunda (`--driverloaded n`) gerçekten çalıştırıldı ve çıktılar bu denemelerden alındı. Ama LV'ler etkinleştirilemediği için **LV üzerinde dosya sistemi oluşturmak, bağlamak, `pvmove` ve dosya sistemiyle birlikte büyütmek (`--resizefs`) denenmedi.** Bu kısımlar `man lvresize` ve `man fsadm`'e göre yazıldı. Aynı sebeple `lvs`'deki `Attr` sütunu ve `lvdisplay`'deki `LV Status` satırı, denemede normal bir sistemden farklı görünüyordu. Bu iki yerde normal sistemde göreceğin değer yazıldı.

## İçindekiler

1. [Kurulum ve diskleri görmek](#1-kurulum-ve-diskleri-görmek)
2. [Physical Volume: pvcreate](#2-physical-volume-pvcreate)
3. [Volume Group: vgcreate](#3-volume-group-vgcreate)
4. [Logical Volume: lvcreate](#4-logical-volume-lvcreate)
5. [LV üzerinde dosya sistemi ve mount](#5-lv-üzerinde-dosya-sistemi-ve-mount)
6. [VG'ye disk eklemek ve çıkarmak](#6-vgye-disk-eklemek-ve-çıkarmak)
7. [LV'yi büyütmek](#7-lvyi-büyütmek)
8. [LV'yi küçültmek](#8-lvyi-küçültmek)
9. [Ayrıntılı bilgi: display komutları](#9-ayrıntılı-bilgi-display-komutları)
10. [Silmek](#10-silmek)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Kurulum ve diskleri görmek

```bash
sudo apt install lvm2
sudo lvmdiskscan
lsblk
```

`lvmdiskscan`, LVM'in kullanabileceği diskleri ve bölümleri listeler. LVM komutları üç gruba ayrılır ve hepsi aynı mantıkla adlandırılmıştır:

| | PV | VG | LV |
|---|---|---|---|
| Oluştur | `pvcreate` | `vgcreate` | `lvcreate` |
| Kısa liste | `pvs` | `vgs` | `lvs` |
| Ayrıntılı | `pvdisplay` | `vgdisplay` | `lvdisplay` |
| Sil | `pvremove` | `vgremove` | `lvremove` |
| Değiştir | | `vgextend`, `vgreduce` | `lvresize`, `lvextend`, `lvreduce` |

`pv` yazıp Tab'a iki kez basmak bütün PV komutlarını gösterir.

---

## 2. Physical Volume: pvcreate

```bash
sudo pvcreate /dev/loop2 /dev/loop3
sudo pvs
```
```
  Physical volume "/dev/loop2" successfully created.
  Physical volume "/dev/loop3" successfully created.
  PV         VG Fmt  Attr PSize PFree
  /dev/loop2    lvm2 ---  1.00g 1.00g
  /dev/loop3    lvm2 ---  1.00g 1.00g
```

`VG` sütunu boş: PV'ler henüz bir havuza ait değil. PV olarak bütün bir disk (`/dev/sdb`) veya bir bölüm (`/dev/sdb1`, türü `Linux LVM` olmalı) kullanılabilir. `pvcreate` diskteki her şeyi siler.

---

## 3. Volume Group: vgcreate

```bash
sudo vgcreate veri_vg /dev/loop2 /dev/loop3
sudo vgs
sudo pvs
```
```
  Volume group "veri_vg" successfully created
  VG      #PV #LV #SN Attr   VSize VFree
  veri_vg   2   0   0 wz--n- 1.99g 1.99g
  PV         VG      Fmt  Attr PSize    PFree
  /dev/loop2 veri_vg lvm2 a--  1020.00m 1020.00m
  /dev/loop3 veri_vg lvm2 a--  1020.00m 1020.00m
```

İki 1 GB'lık disk, 2 GB'lık tek bir havuz oldu. Her PV'nin boyutu 1020 MB görünüyor: LVM her diskin başında kendi bilgileri (meta veri) için küçük bir alan ayırır.

`vgcreate`'e verilen diskler PV değilse, LVM onları kendisi PV yapar:

```bash
sudo vgcreate deneme_vg /dev/loop4
```
```
  Physical volume "/dev/loop4" successfully created.
  Volume group "deneme_vg" successfully created
```

Yani `pvcreate` adımı atlanabilir, ama ayrı yapmak neyin olduğunu görmeyi kolaylaştırır.

---

## 4. Logical Volume: lvcreate

```bash
sudo lvcreate --size 500M --name yedek_lv veri_vg
sudo lvcreate -L 600M -n web_lv veri_vg      # kısa yazım
sudo lvs
sudo vgs
```
```
  Logical volume "yedek_lv" created.
  Logical volume "web_lv" created.
  LV       VG      Attr       LSize
  web_lv   veri_vg -wi-a----- 600.00m
  yedek_lv veri_vg -wi-a----- 500.00m
  VG      #PV #LV #SN Attr   VSize VFree
  veri_vg   2   2   0 wz--n- 1.99g 940.00m
```

Havuzda 940 MB boş yer kaldı. `Attr` sütunundaki `a`, LV'nin etkin (active) olduğunu gösterir.

| Seçenek | Anlamı |
|---|---|
| `-L 500M` / `--size 500M` | Boyutu birimle ver (`M`, `G`, `T`) |
| `-l 100` / `--extents 100` | Boyutu PE sayısıyla ver (100 × 4 MiB = 400 MiB) |
| `-l 100%FREE` | Havuzdaki bütün boş alan |
| `-l 50%VG` | Havuzun yarısı |
| `-n ad` / `--name ad` | LV'nin adı |

LV'ye iki yoldan erişilir, ikisi de aynı birimi gösterir:

```
/dev/veri_vg/yedek_lv
/dev/mapper/veri_vg-yedek_lv
```

---

## 5. LV üzerinde dosya sistemi ve mount

Bir LV, normal bir bölüm gibi kullanılır:

```bash
sudo mkfs.xfs /dev/veri_vg/yedek_lv
sudo mkfs.ext4 /dev/veri_vg/web_lv

sudo mkdir -p /yedekler /srv/web
sudo mount /dev/veri_vg/yedek_lv /yedekler
sudo mount /dev/veri_vg/web_lv /srv/web
```

fstab'da LV için `/dev/veri_vg/yedek_lv` yolu kullanılabilir: LV adları disk adları gibi değişmez. UUID de olur.

```
/dev/veri_vg/yedek_lv  /yedekler  xfs   defaults  0  0
/dev/veri_vg/web_lv    /srv/web   ext4  defaults  0  2
```

Dosya sistemleri ve fstab: [Dosya Sistemleri](02-dosya-sistemleri.md), [Mount ve fstab](03-mount-ve-fstab.md).

---

## 6. VG'ye disk eklemek ve çıkarmak

Havuz doldu, sunucuya üçüncü bir disk eklendi:

```bash
sudo pvcreate /dev/loop4
sudo vgextend veri_vg /dev/loop4
sudo vgs
```
```
  Volume group "veri_vg" successfully extended
  VG      #PV #LV #SN Attr   VSize  VFree
  veri_vg   3   2   0 wz--n- <2.99g 1.91g
```

Havuz 3 GB'a çıktı, boş alan 940 MB'tan 1.91 GB'a yükseldi. LV'ler ve dosya sistemleri kesintisiz çalışmaya devam eder.

Bir diski havuzdan çıkarmak:

```bash
sudo vgreduce veri_vg /dev/loop4
sudo pvremove /dev/loop4
```
```
  Removed "/dev/loop4" from volume group "veri_vg"
  Labels on physical volume "/dev/loop4" successfully wiped.
```

`pvremove` diskteki LVM etiketini siler, artık normal bir disk olur.

`vgreduce` sadece **kullanılmayan** bir PV'yi çıkarabilir. Bir LV'nin hangi disklerde durduğunu `lvs -o +devices` gösterir. Aşağıdaki çıktı, bölüm 7'deki büyütmelerden sonra alındı:

```bash
sudo lvs -o lv_name,devices
sudo vgreduce veri_vg /dev/loop3
```
```
  LV       Devices
  web_lv   /dev/loop3(0)
  web_lv   /dev/loop2(225)
  yedek_lv /dev/loop2(0)
  Physical volume "/dev/loop3" still in use
```

`web_lv` iki diske yayılmış: bir kısmı `loop3`'ün başında, bir kısmı `loop2`'nin 225. parçasından itibaren. Diskte LV verisi varsa önce `sudo pvmove /dev/loop3` ile veriyi havuzdaki diğer disklere taşımak gerekir (diğer disklerde o kadar boş yer olmalı).

Havuzdaki bir disk için `pvremove` da reddedilir:

```
  PV /dev/loop2 is used by VG veri_vg so please use vgreduce first.
```

---

## 7. LV'yi büyütmek

Büyütme iki adımdır: **LV**'yi büyüt, sonra üzerindeki **dosya sistemini** büyüt. Sadece LV'yi büyütürsen `df` eski boyutu göstermeye devam eder.

### LV'yi büyütmek

```bash
sudo lvresize -L 800M veri_vg/yedek_lv       # boyutu 800M yap
sudo lvextend -L +100M veri_vg/yedek_lv      # 100M ekle
sudo lvresize -l +100%FREE veri_vg/web_lv    # havuzdaki bütün boş alanı ekle
sudo lvs
```
```
  Size of logical volume veri_vg/yedek_lv changed from 500.00 MiB (125 extents) to 800.00 MiB (200 extents).
  Size of logical volume veri_vg/yedek_lv changed from 800.00 MiB (200 extents) to 900.00 MiB (225 extents).
  Size of logical volume veri_vg/web_lv changed from 600.00 MiB (150 extents) to 1.11 GiB (285 extents).
  LV       VG      Attr       LSize
  web_lv   veri_vg -wi-a-----   1.11g
  yedek_lv veri_vg -wi-a----- 900.00m
```

Boyutta `+` varsa **eklenir**, yoksa boyut o değere **ayarlanır**. `-L 800M` ile `-L +800M` çok farklı sonuçlar verir. `-l +100%FREE` ile `web_lv` havuzdaki bütün boş alanı aldı, `vgs` artık `VFree 0` gösterir.

LV boyutu her zaman PE'nin (4 MiB) katıdır. Kat olmayan bir boyut verirsen LVM yukarı yuvarlar:

```bash
sudo lvresize -L +50M veri_vg/yedek_lv
```
```
  Rounding size to boundary between physical extents: 52.00 MiB.
```

### Dosya sistemiyle birlikte büyütmek: --resizefs

En kolay yol, `-r` (`--resizefs`) ile ikisini birden yapmak:

```bash
sudo lvresize -r -L +500M veri_vg/yedek_lv
```

`-r`, LV'yi büyüttükten sonra dosya sistemini de büyütür (arka planda `fsadm` aracını kullanır). XFS ve ext4 bağlıyken büyütülebilir, sistemi kapatmak gerekmez.

### Dosya sistemini ayrıca büyütmek

LV'yi `-r` olmadan büyüttüysen:

| Dosya sistemi | Komut | Argüman |
|---|---|---|
| XFS | `sudo xfs_growfs /yedekler` | **Bağlama noktası** |
| ext4 | `sudo resize2fs /dev/veri_vg/web_lv` | **Cihaz** |

Sonuç: `df -h /yedekler`.

---

## 8. LV'yi küçültmek

> [!CAUTION]
> Küçültme veri kaybına yol açabilir. LV, içindeki dosya sisteminden küçük yapılırsa dosya sisteminin son kısmı kesilir ve bozulur. Önce yedek al.

| Dosya sistemi | Küçültülebilir mi? |
|---|---|
| XFS | **Hayır.** Yedekle, LV'yi küçült, yeniden oluştur, geri yükle |
| ext4 | Evet, ama **bağlı değilken** |

ext4 için doğru sıra, dosya sistemini önce küçültmek, sonra LV'yi. `-r` bunu doğru sırayla yapar:

```bash
sudo umount /srv/web
sudo lvresize -r -L 400M veri_vg/web_lv
sudo mount /srv/web
```

`-r` olmadan aktif bir LV'yi küçültmeye çalışırsan LVM uyarır ve onay ister:

```
WARNING: Reducing active logical volume to 400.00 MiB.
THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce veri_vg/web_lv? [y/n]:
```

Dosya sistemini küçültmeden burada `y` demek dosya sistemini bozar.

> [!NOTE]
> Denemede etkin olmayan bir LV'yi `lvreduce` ile küçülttüğümüzde bu uyarı çıkmadı, LVM soru sormadan küçülttü. Uyarı, LV etkin olduğunda ve içinde dosya sistemi olduğunda gösterilir. Yani her zaman uyarıya güvenme, küçültmeden önce neyi küçülttüğünü bil.

---

## 9. Ayrıntılı bilgi: display komutları

`pvs`, `vgs`, `lvs` kısa özet verir. Ayrıntı için `display` komutları:

```bash
sudo lvdisplay veri_vg/yedek_lv
```
```
  --- Logical volume ---
  LV Path                /dev/veri_vg/yedek_lv
  LV Name                yedek_lv
  VG Name                veri_vg
  LV Write Access        read/write
  LV Status              available
  LV Size                400.00 MiB
  Current LE             100
  Segments               1
```

`Current LE 100`: LV 100 parçadan oluşuyor (100 × 4 MiB = 400 MiB). `Segments 1`: tek bir diskte, tek parça halinde duruyor.

```bash
sudo vgdisplay veri_vg
```
```
  VG Size               1.99 GiB
  PE Size               4.00 MiB
  Total PE              510
  Alloc PE / Size       385 / 1.50 GiB
  Free  PE / Size       125 / 500.00 MiB
```

`Free PE`, havuzda kalan boş parça sayısıdır. Bir LV'yi tam olarak kalan alan kadar büyütmek için `-l +125` veya `-l +100%FREE` kullanılabilir.

`lvs`'deki `Attr` sütunu: `-wi-a-----`'daki `a` LV'nin etkin (active) olduğunu, `o` (`-wi-ao----`) bağlı (open) olduğunu gösterir.

---

## 10. Silmek

Sıra, oluşturmanın tersidir:

```bash
sudo umount /yedekler                 # 1. dosya sistemini ayır, fstab'dan sil
sudo lvremove veri_vg/yedek_lv        # 2. LV'yi sil (onay ister, -y ile sormaz)
sudo vgremove veri_vg                 # 3. VG'yi sil (içinde LV kalmamalı)
sudo pvremove /dev/loop2 /dev/loop3   # 4. PV etiketlerini sil
```

`lvremove` ve içinde LV olan bir VG için `vgremove` onay ister:

```
Do you really want to remove and DISCARD logical volume veri_vg/yedek_lv? [y/n]:
Do you really want to remove volume group "veri_vg" containing 2 logical volumes? [y/n]:
```

`vgremove`'a `y` dersen içindeki bütün LV'ler de silinir. `lvremove`, LV'deki bütün veriyi siler.

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sudo lvmdiskscan` | Kullanılabilir diskler |
| `sudo pvcreate /dev/X /dev/Y` | PV oluştur |
| `sudo vgcreate vg_adı /dev/X /dev/Y` | VG oluştur |
| `sudo lvcreate -L 500M -n lv_adı vg_adı` | LV oluştur (boyutla) |
| `sudo lvcreate -l 100%FREE -n lv_adı vg_adı` | Kalan bütün alanla |
| `pvs`, `vgs`, `lvs` | Özet |
| `pvdisplay`, `vgdisplay`, `lvdisplay` | Ayrıntı |
| `sudo vgextend vg /dev/Z` | Havuza disk ekle |
| `sudo pvmove /dev/Z` | Diskteki veriyi diğer PV'lere taşı |
| `sudo vgreduce vg /dev/Z` | Diski havuzdan çıkar |
| `sudo lvresize -r -L +1G vg/lv` | LV ve dosya sistemini büyüt |
| `sudo lvextend -l +100%FREE vg/lv` | Bütün boş alanı ekle |
| `sudo xfs_growfs /bağlama` | XFS'i büyüt |
| `sudo resize2fs /dev/vg/lv` | ext4'ü büyüt veya küçült |
| `sudo lvremove`, `vgremove`, `pvremove` | Sil |

---

## 12. Sık yapılan hatalar

- **LV'yi büyütüp dosya sistemini büyütmemek.** `df` eski boyutu gösterir. `-r` kullan.
- **`-L 1G` ile `-L +1G`'yi karıştırmak.** Biri ayarlar, biri ekler.
- **XFS'i küçültmeye çalışmak.**
- **ext4'ü bağlıyken küçültmek veya LV'yi dosya sisteminden önce küçültmek.**
- **`xfs_growfs`'e cihaz, `resize2fs`'e bağlama noktası vermek.** Tersi doğru.
- **Kullanılan bir PV'yi `pvmove` yapmadan `vgreduce` ile çıkarmaya çalışmak.**
- **Silme sırasını karıştırmak.** LV → VG → PV.
- **`pvcreate`'in diskteki veriyi koruduğunu sanmak.**

---

## 13. Alıştırmalar

Üç test diski hazırla:

```bash
for i in 1 2 3; do truncate -s 1G ~/pv$i.img; sudo losetup -f --show ~/pv$i.img; done
# çıktıdaki cihazları kullan, ör. /dev/loop2 /dev/loop3 /dev/loop4
```

**1.** İlk iki diski PV yap ve `depo_vg` adında bir VG'de topla. VG'nin toplam ve boş alanını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo pvcreate /dev/loop2 /dev/loop3
sudo vgcreate depo_vg /dev/loop2 /dev/loop3
sudo vgs depo_vg
```
</details>

**2.** `depo_vg`'den 700 MB'lık `arsiv_lv` oluştur, XFS ile biçimlendir ve `/arsiv`'e bağla.

<details>
<summary>Çözüm</summary>

```bash
sudo lvcreate -L 700M -n arsiv_lv depo_vg
sudo mkfs.xfs /dev/depo_vg/arsiv_lv
sudo mkdir -p /arsiv
sudo mount /dev/depo_vg/arsiv_lv /arsiv
df -h /arsiv
```
</details>

**3.** Üçüncü diski `depo_vg`'ye ekle. Sonra `arsiv_lv`'yi, dosya sistemiyle birlikte, 500 MB büyüt.

<details>
<summary>Çözüm</summary>

```bash
sudo pvcreate /dev/loop4
sudo vgextend depo_vg /dev/loop4
sudo lvresize -r -L +500M depo_vg/arsiv_lv
df -h /arsiv
```
</details>

**4.** `depo_vg`'de kalan bütün boş alanla `kayit_lv` oluştur ve ext4 ile biçimlendir. VG'de boş yer kalmadığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo lvcreate -l 100%FREE -n kayit_lv depo_vg
sudo mkfs.ext4 /dev/depo_vg/kayit_lv
sudo vgs depo_vg            # VFree 0
```
</details>

**5.** Her şeyi doğru sırayla sil.

<details>
<summary>Çözüm</summary>

```bash
sudo umount /arsiv
sudo lvremove -y depo_vg/arsiv_lv depo_vg/kayit_lv
sudo vgremove depo_vg
sudo pvremove /dev/loop2 /dev/loop3 /dev/loop4
sudo losetup -D && rm ~/pv*.img
```
</details>

---

Önceki konu: [Swap](05-swap.md) · Sonraki konu: [NFS](07-nfs.md)
