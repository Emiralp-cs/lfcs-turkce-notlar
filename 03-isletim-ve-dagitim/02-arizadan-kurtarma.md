# Arızadan Kurtarma

> **Resmi sınav maddesi:** Recover from hardware, operating system, or filesystem failures

Sunucu açılmıyor, bir disk bağlanmıyor ya da root parolasını kimse hatırlamıyor. Bu dosya üç tür arızayı ele alıyor: **açılış** sorunları (GRUB, fstab, unutulan parola), **dosya sistemi** bozulmaları (`fsck`, `xfs_repair`) ve **donanım** arızalarının belirtilerini tanımak.

Senaryo: Bir veri diskinin dosya sistemi elektrik kesintisinden sonra bozuldu, `/etc/fstab`'a yanlış bir satır eklendi ve sunucuda root parolası bilinmiyor. Her birini çözeceğiz.

> [!NOTE]
> Dosya sistemi onarımları ve fstab denetimi, test için oluşturulmuş disklerde (loop cihazı) bilerek bozulma yaratılarak Ubuntu 24.04 üzerinde denenmiştir. GRUB menüsü, açılışta kurtarma moduna girmek ve root parolasını sıfırlamak, deneme ortamında açılış süreci olmadığı için **denenmedi**. Bu adımlar Ubuntu ve GRUB belgelerine göre yazılmıştır.

## İçindekiler

1. [Açılış süreci](#1-açılış-süreci)
2. [GRUB menüsüne girmek ve açılış parametresi vermek](#2-grub-menüsüne-girmek-ve-açılış-parametresi-vermek)
3. [Root parolasını sıfırlamak](#3-root-parolasını-sıfırlamak)
4. [Hatalı fstab yüzünden açılmayan sistem](#4-hatalı-fstab-yüzünden-açılmayan-sistem)
5. [Dosya sistemini kontrol etmek ve onarmak: ext4](#5-dosya-sistemini-kontrol-etmek-ve-onarmak-ext4)
6. [Bozuk superblock](#6-bozuk-superblock)
7. [XFS onarımı](#7-xfs-onarımı)
8. [Donanım arızası belirtileri](#8-donanım-arızası-belirtileri)
9. [GRUB ve initramfs'i yeniden oluşturmak](#9-grub-ve-initramfsi-yeniden-oluşturmak)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Açılış süreci

Sorunun nerede olduğunu anlamak için açılışın hangi aşamada durduğunu bilmek gerekir:

```
Firmware (BIOS/UEFI)  →  GRUB  →  Çekirdek + initramfs  →  systemd  →  default.target
   donanım kontrolü     menü,       kök diski bulur,       servisler,    giriş ekranı
                        çekirdeği   bağlar                 fstab
                        yükler
```

| Durduğu yer | Belirti | Olası sebep |
|---|---|---|
| GRUB | `grub rescue>` istemi | GRUB dosyaları veya disk bozulmuş |
| Çekirdek/initramfs | `Kernel panic`, `(initramfs)` istemi | Kök disk bulunamıyor, initramfs bozuk |
| systemd | `You are in emergency mode` | `/etc/fstab` hatası, bozuk dosya sistemi |
| Servisler | Giriş ekranı geliyor ama servis çalışmıyor | Servis sorunu ([Servis Oluşturma ve Sorun Giderme](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md)) |

---

## 2. GRUB menüsüne girmek ve açılış parametresi vermek

Sunucular genelde GRUB menüsünü göstermeden açılır. Menüyü görmek için açılışta **Shift** (BIOS) veya **Esc** (UEFI) tuşuna basılı tut.

Menüde:

- **Advanced options for Ubuntu → recovery mode**: Ubuntu'nun hazır kurtarma menüsü. `fsck`, `root` (kabuk), `network` gibi seçenekler sunar.
- Normal satırın üzerindeyken **`e`**: o açılışın parametrelerini geçici olarak düzenle.

`e` ile açılan ekranda `linux` ile başlayan satırın **sonuna** bir parametre eklenir, **Ctrl+X** (veya F10) ile açılır:

| Eklenen parametre | Sonuç |
|---|---|
| `systemd.unit=rescue.target` | Rescue moduna açılır |
| `systemd.unit=emergency.target` | Emergency moduna açılır |
| `systemd.unit=multi-user.target` | Grafik arayüz olmadan açılır |
| `init=/bin/bash` | systemd hiç başlamaz, doğrudan root kabuğu açılır |
| `fsck.mode=force` | Açılışta dosya sistemleri kontrol edilir |

Bu değişiklikler **sadece o açılış için** geçerlidir. Rescue ve emergency modlarının farkı: [Açma, Kapatma ve Target'lar](01-acma-kapatma-ve-targetlar.md#7-rescue-ve-emergency).

Kalıcı GRUB ayarları `/etc/default/grub` dosyasındadır ([9. bölüm](#9-grub-ve-initramfsi-yeniden-oluşturmak)).

---

## 3. Root parolasını sıfırlamak

Rescue ve emergency modları root parolası ister. Parola bilinmiyorsa veya Ubuntu'daki gibi root hesabı kilitliyse (`passwd -S root` çıktısında `L`), bu modlara girilemez. O zaman `init=/bin/bash`:

**1.** GRUB menüsünde normal açılış satırının üzerinde `e`'ye bas.

**2.** `linux` ile başlayan satırın sonuna, genelde orada bulunan `ro quiet splash` gibi parametrelerden sonra ekle:

```
init=/bin/bash
```

**3.** Ctrl+X ile aç. Parola sormadan bir root kabuğu gelir.

**4.** Kök dosya sistemi salt okunur bağlıdır. Yazılabilir yap:

```bash
mount -o remount,rw /
```

**5.** Parolayı değiştir:

```bash
passwd root            # veya parolasını unuttuğun kullanıcı: passwd ayse
```

**6.** Diske yazıldığından emin ol ve sistemi yeniden başlat. Bu kabukta systemd çalışmadığı için normal `reboot` çalışmaz:

```bash
sync
exec /sbin/init        # normal açılışa devam et
# veya
reboot -f
```

> [!IMPORTANT]
> RHEL ailesinde SELinux etkinse, parolayı değiştirdikten sonra `touch /.autorelabel` gerekir. Yoksa `/etc/shadow`'un SELinux etiketi bozulur ve kimse giriş yapamaz. Ubuntu'da bu adım gerekmez.

Bu yöntem, sunucuya fiziksel veya konsol erişimi olan herkesin root olabileceğini gösterir. Bunu önlemek için GRUB'a parola konulabilir (`grub-mkpasswd-pbkdf2`), ama bu sınav kapsamının dışında.

---

## 4. Hatalı fstab yüzünden açılmayan sistem

`/etc/fstab`'daki bir disk açılışta bulunamazsa veya bağlanamazsa systemd, sistemi **emergency** moduna düşürür:

```
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
```

Çözüm:

```bash
mount -o remount,rw /          # emergency'de kök salt okunur
nano /etc/fstab                # hatalı satırı düzelt veya başına # koy
systemctl daemon-reload
mount -a                       # bütün fstab'ı dene, hata vermemeli
systemctl default              # normal açılışa devam et
```

### Önlemek: findmnt --verify ve nofail

`/etc/fstab`'ı her değiştirdiğinde, **yeniden başlatmadan önce** kontrol et. Hatalı satırlar içeren bir test dosyasıyla denedik:

```bash
findmnt --verify --tab-file test.fstab
```
```
/yedek
   [E] unreachable on boot required target: No such file or directory
   [E] unreachable on boot required source: UUID=yok-uuid
/veri
   [E] unreachable on boot required target: No such file or directory
   [W] ext5 seems unsupported by the current kernel
   [W] ext5 does not match with on-disk xfs

0 parse errors, 3 errors, 2 warnings
```

Var olmayan bağlama noktaları, bulunamayan UUID, yanlış dosya sistemi türü, hepsini açılıştan önce yakaladı. `--tab-file` verilmezse `/etc/fstab`'ı kontrol eder: `sudo findmnt --verify`.

Açılış için şart olmayan diskler (yedek diski, harici disk) için seçeneklere `nofail` eklenirse, disk bulunamadığında sistem emergency'ye düşmeden açılmaya devam eder:

```
UUID=...  /yedek  ext4  defaults,nofail  0  2
```

fstab biçimi: [Mount ve fstab](../04-depolama/03-mount-ve-fstab.md).

---

## 5. Dosya sistemini kontrol etmek ve onarmak: ext4

> [!CAUTION]
> Dosya sistemi kontrolü ve onarımı **bağlı olmayan** bir diskte yapılır. Bağlı bir dosya sistemini onarmaya çalışmak onu daha da bozabilir. Kök dosya sistemi çalışırken ayrılamadığı için onu ancak kurtarma modunda, bir canlı USB'den veya açılışta `fsck.mode=force` ile kontrol edebilirsin.

Önce diski ayır:

```bash
sudo umount /srv/veri
```

Bağlıyken çalıştırmaya çalışırsan `e2fsck` uyarır: `Warning!  /dev/loop0 is mounted.`

### -f olmadan kontrol yapılmayabilir

Diskteki bir inode'u bilerek bozduk ve kontrol ettik:

```bash
sudo e2fsck -n /dev/loop0
```
```
/dev/loop0: clean, 17/2000 files, 15255/25600 blocks
```

`clean` dedi, ama disk bozuk. `e2fsck`, dosya sistemi kendini "temiz kapatıldım" diye işaretlemişse kontrol yapmadan çıkar. Bozulma bu işareti değiştirmediği için hiçbir şey bulmadı. **`-f` (force) ile** gerçekten kontrol eder:

```bash
sudo e2fsck -f -p /dev/loop0
```
```
/dev/loop0: Entry 'loglar' in / (2) has deleted/unused inode 12.  CLEARED.
/dev/loop0: Inode 2 ref count is 6, should be 5.  FIXED.
/dev/loop0: 16/2000 files (6.3% non-contiguous), 15254/25600 blocks
```

### Seçenekler

| Seçenek | Anlamı |
|---|---|
| `-f` | Temiz görünse bile kontrol et |
| `-n` | Hiçbir şeyi değiştirme, sadece raporla. Önce bununla bakmak güvenlidir |
| `-p` | Güvenli sorunları sormadan onar (preen). Ciddi bir sorunda durur ve elle çalıştırmanı ister |
| `-y` | Her soruya "evet" de, her şeyi onar |
| `-v` | Ayrıntılı çıktı |
| `-b BLOK` | Yedek superblock'u kullan ([6. bölüm](#6-bozuk-superblock)) |

Önerilen sıra: önce `-f -n` ile bak, sonra `-f -p`, `-p` "elle çalıştır" derse `-f -y` veya soruları tek tek cevaplayarak `-f`.

`fsck /dev/loop0` da aynı işi yapar: `fsck` dosya sisteminin türüne bakıp `fsck.ext4`'ü (yani `e2fsck`'i) çağırır.

Onarım sırasında sahipsiz kalan dosyalar diskin kökündeki `lost+found` dizinine, inode numaralarıyla adlandırılarak taşınır.

---

## 6. Bozuk superblock

Superblock, dosya sisteminin boyutu, blok sayısı gibi temel bilgilerini tutan yapıdır. Bozulursa disk hiç bağlanmaz. Bir diskin superblock'unu bilerek sildik:

```
mount: /mnt: wrong fs type, bad option, bad superblock on /dev/loop1, ...
```
```
e2fsck: Bad magic number in super-block while trying to open /dev/loop1
```

ext4, superblock'un yedeklerini diskin farklı yerlerinde tutar. Yerlerini bulmak için:

```bash
sudo dumpe2fs /dev/loop1 | grep -i 'superblock at'
```

Asıl superblock bozuk olduğu için bu komut da çalışmayabilir. O zaman `mke2fs -n` ile, aynı ayarlarla bir dosya sistemi oluşturulsaydı yedeklerin nerede olacağını sor. **`-n` sayesinde hiçbir şey yazılmaz:**

```bash
sudo mke2fs -n /dev/loop1
```
```
Superblock backups stored on blocks:
	32768
```

Yedekle onar:

```bash
sudo e2fsck -f -y -b 32768 /dev/loop1
```
```
/dev/loop1: ***** FILE SYSTEM WAS MODIFIED *****
```

Disk tekrar bağlandı ve içindeki dosya yerindeydi.

> [!WARNING]
> `mke2fs`'i **`-n` olmadan** çalıştırırsan diski biçimlendirir ve bütün veri gider. Komutu iki kez kontrol et.

Çok küçük dosya sistemlerinde (tek blok grubu olanlar, örneğin 100 MB'lık test diskimiz) yedek superblock hiç yoktur. Bu durumda `e2fsck -b` çalışmaz.

---

## 7. XFS onarımı

XFS'in kontrol ve onarım aracı `xfs_repair`'dir. `fsck` XFS'te işe yaramaz:

```bash
sudo fsck.xfs /dev/loop1
```
```
If you wish to check the consistency of an XFS filesystem or
repair a damaged filesystem, see xfs_repair(8).
```

Hiçbir şey kontrol etmedi ve başarılı (çıkış kodu 0) döndü. XFS'te her zaman `xfs_repair` kullan:

```bash
sudo umount /dev/loop1
sudo xfs_repair -n /dev/loop1     # sadece kontrol et, değiştirme
sudo xfs_repair /dev/loop1        # onar
```

Bağlı bir XFS'te çalıştırınca reddetti:

```
xfs_repair: /dev/loop1 contains a mounted and writable filesystem
```

`xfs_repair` "journal'da işlenmemiş kayıtlar var" diye durursa, önce diski bir kez bağlayıp ayır (bu, journal'ı işler) ve tekrar dene. Bu da işe yaramazsa son çare `xfs_repair -L`: journal'ı siler, son değişiklikler kaybolabilir.

`xfs_repair` için `xfsprogs` paketi gerekir. XFS ve ext4 dosya sistemlerinin oluşturulması: [Dosya Sistemleri](../04-depolama/02-dosya-sistemleri.md).

| | ext4 | XFS |
|---|---|---|
| Kontrol | `e2fsck -f -n` | `xfs_repair -n` |
| Onarım | `e2fsck -f -p` veya `-y` | `xfs_repair` |
| Son çare | `e2fsck -b YEDEK` | `xfs_repair -L` |
| Bağlıyken | Uyarır | Reddeder |

---

## 8. Donanım arızası belirtileri

Dosya sistemi tekrar tekrar bozuluyorsa sorun yazılımda değil, diskte olabilir. Belirtiler çekirdek loglarında görünür:

```bash
journalctl -k -p err             # çekirdekten gelen hatalar
sudo dmesg -T | grep -i -E 'error|fail|i/o'
```

Diskin kendisinden bir sorun geliyorsa `I/O error`, `Buffer I/O error on dev sdb`, `reset`, `timeout` gibi satırlar görürsün.

**SMART** (diskin kendi sağlık kaydı) için `smartmontools` paketi:

```bash
sudo smartctl -H /dev/sda        # genel sağlık: PASSED / FAILED
sudo smartctl -a /dev/sda        # ayrıntılar: yeniden ayrılmış sektörler, hata sayaçları
```

Sanal makinelerde ve bazı RAID kartlarının arkasındaki disklerde SMART bilgisi okunamaz.

Bir önceki açılışta ne olduğuna bakmak (çekirdek çökmesi, beklenmedik kapanma):

```bash
journalctl --list-boots
journalctl -b -1 -p err          # bir önceki açılışın hataları
```

Bu, journal'ın kalıcı tutulmasını gerektirir ([Log Dosyaları](04-log-dosyalari.md)).

| Belirti | Bakılacak yer |
|---|---|
| Disk hataları, dosya sistemi tekrar tekrar bozuluyor | `dmesg`, `smartctl -a` |
| Rastgele çökmeler, `Kernel panic` | Bellek arızası olabilir: GRUB menüsündeki `memtest` |
| Disk listede yok | `lsblk`, `dmesg \| grep -i sd` |

---

## 9. GRUB ve initramfs'i yeniden oluşturmak

Ubuntu'da GRUB ayarları `/etc/default/grub` dosyasındadır:

```
GRUB_TIMEOUT=5                       # menünün bekleme süresi
GRUB_TIMEOUT_STYLE=menu              # menüyü her açılışta göster (hidden: gizle)
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"   # her açılışta eklenecek parametreler
```

Dosyayı değiştirdikten sonra GRUB yapılandırmasını yeniden oluştur:

```bash
sudo update-grub
```

`/boot/grub/grub.cfg`'yi elle düzenleme, `update-grub` her seferinde onu baştan yazar.

**initramfs**, çekirdeğin kök diski bulup bağlayabilmesi için gereken küçük sistemdir. Disk sürücüsü değiştiyse veya initramfs bozulduysa:

```bash
sudo update-initramfs -u           # çalışan çekirdek için yeniden oluştur
```

Sistem hiç açılmıyorsa bu işlemler bir canlı USB'den açılıp, sistemin diskini bağlayıp `chroot` ile o sistemin içine girerek yapılır.

> [!NOTE]
> Bu bölümdeki komutlar, deneme ortamında GRUB kurulu olmadığı için denenmedi.

---

## 10. Komut özeti

| Komut / Parametre | Ne yapar? |
|---|---|
| GRUB'da `e`, sonra `systemd.unit=rescue.target` | Rescue moduna aç |
| GRUB'da `init=/bin/bash` | Parolasız root kabuğu |
| `mount -o remount,rw /` | Kök dosya sistemini yazılabilir yap |
| `exec /sbin/init` | `init=/bin/bash`'ten normal açılışa geç |
| `findmnt --verify` | `/etc/fstab`'ı kontrol et |
| `mount -a` | fstab'daki her şeyi bağlamayı dene |
| `nofail` | Disk yoksa açılışı durdurma |
| `e2fsck -f -n /dev/X` | ext4'ü kontrol et, değiştirme |
| `e2fsck -f -p /dev/X` | Güvenli onarım |
| `e2fsck -f -y /dev/X` | Her şeyi onar |
| `mke2fs -n /dev/X` | Yedek superblock yerlerini göster (yazmaz) |
| `e2fsck -b 32768 /dev/X` | Yedek superblock ile onar |
| `xfs_repair -n /dev/X` | XFS'i kontrol et |
| `xfs_repair /dev/X` | XFS'i onar |
| `journalctl -k -p err` | Çekirdek hataları |
| `smartctl -H /dev/X` | Disk sağlığı |
| `journalctl -b -1` | Önceki açılışın logları |
| `update-grub` | GRUB yapılandırmasını oluştur |
| `update-initramfs -u` | initramfs'i yeniden oluştur |

---

## 11. Sık yapılan hatalar

- **Bağlı bir dosya sistemini onarmaya çalışmak.**
- **`e2fsck`'i `-f` olmadan çalıştırıp "clean" çıktısına güvenmek.**
- **XFS'te `fsck` kullanmak.** Hiçbir şey yapmaz ve başarılı döner.
- **Yedek superblock'u bulmak için `mke2fs`'i `-n` olmadan çalıştırmak.** Disk biçimlendirilir.
- **`/etc/fstab`'ı değiştirip kontrol etmeden yeniden başlatmak.** `findmnt --verify` ve `mount -a`.
- **Açılış için şart olmayan disklere `nofail` eklememek.**
- **`init=/bin/bash` kabuğunda `mount -o remount,rw /` yapmadan parola değiştirmeye çalışmak.** `Authentication token manipulation error` alırsın.
- **`/boot/grub/grub.cfg`'yi elle düzenlemek.**

---

## 12. Alıştırmalar

Alıştırmalar için gerçek disklerine dokunmamak adına bir test diski oluştur:

```bash
dd if=/dev/zero of=~/test.img bs=1M count=300
sudo losetup -f --show ~/test.img      # çıktıdaki cihazı (ör. /dev/loop0) kullan
sudo mkfs.ext4 /dev/loop0
```

**1.** Test diskini `/mnt`'e bağla, içine bir dosya yaz, sonra dosya sistemini **değiştirmeden** kontrol et. Bağlıyken ne oluyor?

<details>
<summary>Çözüm</summary>

```bash
sudo mount /dev/loop0 /mnt
echo deneme | sudo tee /mnt/dosya.txt
sudo e2fsck -n /dev/loop0      # Warning: /dev/loop0 is mounted.
sudo umount /mnt
sudo e2fsck -f -n /dev/loop0
```
</details>

**2.** Test diskinin superblock yedeklerinin nerede olduğunu, diske hiçbir şey yazmadan bul.

<details>
<summary>Çözüm</summary>

```bash
sudo dumpe2fs /dev/loop0 | grep -i 'superblock at'
sudo mke2fs -n /dev/loop0 | grep -A1 -i backups
```
</details>

**3.** Test diskinin asıl superblock'unu boz, bağlanmadığını gör, yedekten onar ve dosyanın yerinde olduğunu kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo dd if=/dev/zero of=/dev/loop0 bs=1024 seek=1 count=1 conv=notrunc
sudo mount /dev/loop0 /mnt          # bad superblock
sudo e2fsck -f -y -b 32768 /dev/loop0
sudo mount /dev/loop0 /mnt
cat /mnt/dosya.txt
sudo umount /mnt
```
</details>

**4.** Aşağıdaki satırları içeren bir test dosyası oluştur ve hataları açılıştan önce bul:

```
UUID=11111111-2222-3333-4444-555555555555 /arsiv ext4 defaults 0 2
/dev/loop0 /mnt ext4 defaults 0 2
```

<details>
<summary>Çözüm</summary>

```bash
findmnt --verify --tab-file test.fstab
```

İlk satırda UUID bulunamadığı ve `/arsiv` dizini olmadığı için hata verir. Bu satır gerçek fstab'da olsaydı ve `nofail` olmasaydı sistem emergency moduna düşerdi.
</details>

**5.** Root parolasını unuttuğun bir Ubuntu sunucusunda yapacağın adımları sırayla yaz. (Kendi sanal makinende dene.)

<details>
<summary>Çözüm</summary>

1. Açılışta Shift/Esc ile GRUB menüsünü aç.
2. Normal satırda `e`'ye bas.
3. `linux` satırının sonuna `init=/bin/bash` ekle, Ctrl+X.
4. `mount -o remount,rw /`
5. `passwd root`
6. `sync` ve `exec /sbin/init` (veya `reboot -f`).
</details>

Bitirince: `sudo losetup -d /dev/loop0 && rm ~/test.img`

---

Önceki konu: [Açma, Kapatma ve Target'lar](01-acma-kapatma-ve-targetlar.md) · Sonraki konu: [Süreçler](03-surecler.md)
