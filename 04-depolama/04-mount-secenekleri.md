# Mount Seçenekleri

> **İlgili resmi sınav maddesi:** Create, manage, and troubleshoot filesystems

Bir dosya sistemi bağlanırken **nasıl** davranacağı seçeneklerle belirlenir: salt okunur mu olsun, içindeki programlar çalıştırılabilsin mi, SUID bitleri geçerli olsun mu, dosya sistemine özgü performans ayarları ne olsun. Seçenekler hem `mount -o` ile hem de `/etc/fstab`'ın 4. alanında verilir.

Senaryo: `/veri` dizinine bağlı disk, kullanıcıların dosya yüklediği bir alan. Oraya yüklenen bir programın çalıştırılmasını ve SUID bitiyle yetki yükseltilmesini engelleyeceğiz. Bakım sırasında diski geçici olarak salt okunur yapacağız. `/yedekler`'deki XFS'e de dosya sistemine özgü bir ayar vereceğiz.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde, test diskinin bölümlerinde denenmiştir. `/veri` ext4, `/yedekler` XFS.

## İçindekiler

1. [Seçenekleri görmek: findmnt](#1-seçenekleri-görmek-findmnt)
2. [ro ve rw](#2-ro-ve-rw)
3. [Bağlıyken değiştirmek: remount](#3-bağlıyken-değiştirmek-remount)
4. [Güvenlik seçenekleri: noexec, nosuid, nodev](#4-güvenlik-seçenekleri-noexec-nosuid-nodev)
5. [remount'un tuzağı](#5-remountun-tuzağı)
6. [Genel seçenekler](#6-genel-seçenekler)
7. [Dosya sistemine özgü seçenekler](#7-dosya-sistemine-özgü-seçenekler)
8. [Seçenekleri fstab'a yazmak](#8-seçenekleri-fstaba-yazmak)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Seçenekleri görmek: findmnt

```bash
findmnt -t xfs,ext4
findmnt -no OPTIONS /veri       # sadece seçenekler
```
```
TARGET    SOURCE       FSTYPE OPTIONS
/yedekler /dev/loop0p1 xfs    rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
/veri     /dev/loop0p3 ext4   rw,relatime
```

`OPTIONS` sütunu, dosya sisteminin **şu an** hangi seçeneklerle bağlı olduğunu gösterir. Varsayılan olarak eklenen seçenekler de görünür (`relatime`, XFS'te `inode64`, `logbufs=8` gibi).

`findmnt`, `/proc`, `/sys` gibi sanal dosya sistemleriyle birlikte uzun bir liste verir. Gerçek dosya sistemlerini görmek için `-t` ile türü belirt. `findmnt` sadece **bağlı** olanları gösterir: fstab'da olup bağlanmamış bir disk listede çıkmaz.

---

## 2. ro ve rw

| Seçenek | Anlamı |
|---|---|
| `rw` | Okuma-yazma (varsayılan) |
| `ro` | Salt okunur |

```bash
sudo mount -o ro /dev/loop0p3 /veri
```

Salt okunur bir dosya sisteminde yazmaya çalışınca:

```
touch: cannot touch '/veri/x': Read-only file system
```

Bu, root için de geçerlidir. Bir disk bozulmaya başladığında çekirdek onu kendiliğinden `ro` yapabilir; beklemediğin bir `Read-only file system` hatası görürsen `dmesg`'e bak.

---

## 3. Bağlıyken değiştirmek: remount

Bağlı bir dosya sistemini aynı yere tekrar bağlamaya çalışınca:

```bash
sudo mount -o ro,noexec,nosuid /dev/loop0p3 /veri
```
```
mount: /veri: /dev/loop0p3 already mounted on /veri.
```

Ayırıp tekrar bağlamak yerine `remount`:

```bash
sudo mount -o remount,ro /veri
findmnt -no OPTIONS /veri
```
```
ro,relatime
```

`remount`, dosya sistemini ayırmadan seçeneklerini değiştirir. Kullanımdaki (`target is busy`) bir dosya sisteminde de çalışır. Bakım sırasında bir diski geçici olarak salt okunur yapmanın yolu budur. Geri almak için `sudo mount -o remount,rw /veri`.

---

## 4. Güvenlik seçenekleri: noexec, nosuid, nodev

Kullanıcıların dosya yazabildiği alanlar (yükleme dizinleri, `/tmp`, ayrı veri diskleri) için:

| Seçenek | Ne engeller? |
|---|---|
| `noexec` | Bu dosya sistemindeki dosyaların program olarak çalıştırılmasını |
| `nosuid` | SUID ve SGID bitlerinin etkisini ([İzinler ve Özel Bitler](../01-temel-komutlar/04-izinler-ve-ozel-bitler.md)) |
| `nodev` | Cihaz dosyalarının (`/dev/...` gibi) çalışmasını |

```bash
sudo mount -o remount,rw,noexec,nosuid /veri
findmnt -no OPTIONS /veri
```
```
rw,nosuid,noexec,relatime
```

### noexec

Çalıştırma izni olan bir script'i doğrudan çalıştırmayı denedik:

```
/veri/s.sh: Permission denied
```

Dosyanın `x` izni vardı, ama dosya sistemi izin vermedi. Ancak:

```bash
sh /veri/s.sh
```
```
calisti
```

`noexec` sadece dosyanın **doğrudan** çalıştırılmasını engeller. Script'i bir yorumlayıcıya (`sh`, `python3`) argüman olarak vermek çalışır, çünkü çalışan program `/bin/sh`'tır. Yani `noexec` tek başına bir güvenlik garantisi değil, bir katmandır.

### nosuid

`id` komutunun SUID bitli bir kopyasını diske koyup normal bir kullanıcıyla çalıştırdık:

| Seçenekler | Çıktı |
|---|---|
| `nosuid` yok | `uid=30035(deneme) ... euid=0(root)`: program root yetkisiyle çalıştı |
| `nosuid` | `uid=30035(deneme) gid=30035(deneme)`: SUID bitinin etkisi yok |

SUID bitli bir programla yetki yükseltmeyi, dışarıdan yüklenen dosyalar için engellemenin yolu `nosuid`'dir.

---

## 5. remount'un tuzağı

`noexec,nosuid` ile bağlı diskte sadece çalıştırmaya izin vermek istedik:

```bash
sudo mount -o remount,rw,noexec,nosuid /veri
findmnt -no OPTIONS /veri      # rw,nosuid,noexec,relatime
sudo mount -o remount,exec /veri
findmnt -no OPTIONS /veri      # rw,relatime
```

`nosuid` da **kayboldu**. `mount -o remount`, disk fstab'da tanımlıysa, yeni seçenekleri o anki seçeneklere değil **fstab'daki seçeneklere** ekler. fstab'da `defaults` yazdığı için `nosuid` düştü.

Remount yaparken korunmasını istediğin bütün seçenekleri tekrar yaz:

```bash
sudo mount -o remount,exec,nosuid /veri
```

ve sonucu her zaman `findmnt` ile kontrol et. Kalıcı bir değişiklik istiyorsan seçenekleri fstab'a yaz ([8. bölüm](#8-seçenekleri-fstaba-yazmak)).

---

## 6. Genel seçenekler

| Seçenek | Anlamı |
|---|---|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` |
| `auto` / `noauto` | `mount -a` ve açılışta bağlansın / bağlanmasın |
| `user` / `nouser` | Normal kullanıcı bağlayabilsin / bağlayamasın |
| `nofail` | Cihaz yoksa açılışı durdurma |
| `_netdev` | Ağ hazır olunca bağla (NFS, iSCSI) |
| `relatime` (varsayılan) | Erişim zamanını (atime) sadece gerektiğinde güncelle |
| `noatime` | Erişim zamanını hiç güncelleme, okumalar diske yazma yapmaz |
| `sync` / `async` | Yazmalar hemen diske / önbellekte bekleyerek |

Bütün genel seçenekler: `man mount`, "FILESYSTEM-INDEPENDENT MOUNT OPTIONS" bölümü.

---

## 7. Dosya sistemine özgü seçenekler

Bazı seçenekler sadece belirli bir dosya sisteminde geçerlidir. Örnek: XFS'in `allocsize`'ı, dosyalar büyürken diskte önceden ne kadar yer ayrılacağını belirler. XFS bunu normalde kendisi ayarlar, sabit bir değer vermek çok sayıda dosyaya aynı anda yazılan sistemlerde parçalanmayı azaltabilir.

```bash
sudo umount /yedekler
sudo mount -o allocsize=32K /dev/loop0p1 /yedekler
findmnt -no OPTIONS /yedekler
```
```
rw,relatime,inode64,allocsize=32k,logbufs=8,logbsize=32k,noquota
```

Aynı seçeneği ext4'e vermeye çalıştık:

```
mount: /mnt: wrong fs type, bad option, bad superblock on /dev/loop0p3, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
```

Hata mesajı birçok olası sebep sayıyor. Asıl sebep `dmesg`'te:

```
ext4: Unknown parameter 'allocsize'
```

`wrong fs type, bad option` gördüğünde ilk iş `sudo dmesg | tail`.

Dosya sistemine özgü seçenekler kendi man sayfalarında: `man xfs`, `man ext4`, `man nfs`.

---

## 8. Seçenekleri fstab'a yazmak

`mount -o` ile verilen seçenekler yeniden başlatınca kaybolur. Kalıcı olması için fstab'ın 4. alanına, aralarında boşluk olmadan virgülle yazılır:

```
UUID=89160837-...  /yedekler  xfs   defaults,allocsize=32k  0  0
LABEL=SecondFS     /veri      ext4  rw,noexec,nosuid,nodev   0  2
```

Sonra:

```bash
sudo systemctl daemon-reload
sudo mount -o remount /veri      # fstab'daki seçeneklerle yeniden bağla
findmnt -no OPTIONS /veri
```

`remount`'a sadece bağlama noktası verilince seçenekleri fstab'dan okur. Denemede `/veri` fstab'daki `rw,noexec,nosuid,nodev` ile yeniden bağlandı.

Bazı seçenekler remount ile değişmez. `mount -o remount,allocsize=64k /yedekler` hata vermedi, ama `findmnt` değerin hala `allocsize=32k` olduğunu gösterdi: değişiklik sessizce yok sayıldı. Böyle seçenekler için `umount` + `mount` gerekir. Remount sonrası her zaman `findmnt` ile kontrol et.

---

## 9. Komut özeti

| Komut / Seçenek | Ne yapar? |
|---|---|
| `findmnt -t xfs,ext4` | Bağlı dosya sistemleri ve seçenekleri |
| `findmnt -no OPTIONS /dizin` | Tek bağlamanın seçenekleri |
| `sudo mount -o ro /dev/X /dizin` | Seçenekle bağla |
| `sudo mount -o remount,ro /dizin` | Bağlıyken değiştir |
| `sudo mount -o remount /dizin` | fstab'daki seçeneklerle yeniden bağla |
| `ro`, `rw` | Salt okunur, okuma-yazma |
| `noexec`, `nosuid`, `nodev` | Çalıştırma, SUID, cihaz dosyası yok |
| `noatime` | Erişim zamanı güncelleme |
| `nofail`, `_netdev`, `noauto` | Açılış davranışı |
| `allocsize=` (XFS) | Önceden ayrılacak alan |
| `sudo dmesg \| tail` | `bad option` hatasının asıl sebebi |

---

## 10. Sık yapılan hatalar

- **Bağlı diski `remount` olmadan tekrar bağlamaya çalışmak.** `already mounted`.
- **`remount`'un diğer seçenekleri koruduğunu sanmak.** fstab'daki seçenekleri temel alır, sonucu `findmnt` ile kontrol et.
- **`noexec`'in her türlü çalıştırmayı engellediğini sanmak.** `sh dosya` çalışır.
- **fstab'da seçeneklerin arasına boşluk koymak.** `rw, noexec` iki ayrı alan sayılır ve satır bozulur.
- **Bir dosya sisteminin seçeneğini başka bir dosya sistemine vermek.**
- **`wrong fs type, bad option` hatasında `dmesg`'e bakmamak.**
- **`mount -o` ile verilen seçeneğin kalıcı olduğunu sanmak.**

---

## 11. Alıştırmalar

Bir ext4 test bölümünün `/veri`'ye bağlı olsun.

**1.** `/veri`'nin şu anki seçeneklerini göster. Sonra yeniden bağlamadan salt okunur yap ve bir dosya oluşturmayı dene.

<details>
<summary>Çözüm</summary>

```bash
findmnt -no OPTIONS /veri
sudo mount -o remount,ro /veri
sudo touch /veri/deneme        # Read-only file system
```
</details>

**2.** `/veri`'yi tekrar yazılabilir yap, ama içindeki programlar çalıştırılamasın ve SUID bitleri etkisiz olsun. Bir script ile `noexec`'i dene.

<details>
<summary>Çözüm</summary>

```bash
sudo mount -o remount,rw,noexec,nosuid /veri
findmnt -no OPTIONS /veri
printf '#!/bin/sh\necho merhaba\n' | sudo tee /veri/s.sh
sudo chmod +x /veri/s.sh
/veri/s.sh                     # Permission denied
sh /veri/s.sh                  # merhaba
```
</details>

**3.** 2. alıştırmadan sonra sadece `exec` vererek remount yap. Hangi seçenekler kaldı? `nosuid`'i koruyarak tekrar dene.

<details>
<summary>Çözüm</summary>

```bash
sudo mount -o remount,exec /veri
findmnt -no OPTIONS /veri      # nosuid gitti (fstab'da defaults varsa)
sudo mount -o remount,exec,nosuid /veri
findmnt -no OPTIONS /veri
```
</details>

**4.** `/veri`'nin açılışta `noexec,nosuid,nodev` seçenekleriyle bağlanmasını sağla, yeniden başlatmadan uygula ve doğrula.

<details>
<summary>Çözüm</summary>

`/etc/fstab`'daki `/veri` satırının 4. alanını `rw,noexec,nosuid,nodev` yap, sonra:

```bash
sudo systemctl daemon-reload
sudo mount -o remount /veri
findmnt -no OPTIONS /veri
```
</details>

**5.** Bir ext4 bölümünü `allocsize=64K` ile bağlamayı dene. Hatanın asıl sebebini bul.

<details>
<summary>Çözüm</summary>

```bash
sudo mount -o allocsize=64K /dev/loop0p2 /mnt      # wrong fs type, bad option
sudo dmesg | tail -1                                # ext4: Unknown parameter 'allocsize'
```

`allocsize` XFS'e özgü bir seçenektir.
</details>

---

Önceki konu: [Mount ve fstab](03-mount-ve-fstab.md) · Sonraki konu: [Swap](05-swap.md)
