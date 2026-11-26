# Sanal Dosya Sistemleri

> **Resmi sınav maddesi:** Manage and configure the virtual file system

Bazı dosya sistemlerinin arkasında bir disk yoktur. `/proc` ve `/sys`'teki dosyalar, onları okuduğun anda çekirdek tarafından üretilir. `/run` ve `/dev/shm` RAM'de durur ve yeniden başlatınca boşalır. Bunlara **sanal** (veya sözde, pseudo) dosya sistemleri denir. Linux'un her şeyi dosya olarak gösterme yaklaşımının bir parçasıdırlar: çekirdeğin durumunu `cat` ile okur, bazı ayarlarını `echo` ile değiştirirsin.

> [!NOTE]
> **Bu maddenin yorumu.** Resmi listede "the virtual file system" ifadesinin tam olarak neyi kapsadığı açıklanmıyor. Linux çekirdeğinde VFS (Virtual File System), bütün dosya sistemlerini aynı komutlarla kullanmayı sağlayan iç katmanın adıdır ve yönetilecek bir şey değildir. Bu notta, bir sistem yöneticisinin gerçekten yönettiği ve yapılandırdığı sanal dosya sistemleri ele alındı: `/proc`, `/sys`, `tmpfs` ve dosya üzerinde sanal diskler (loop cihazları).

Senaryo: Bir uygulama geçici dosyalarını hızlı bir yerde tutmak istiyor. Bunun için 50 MB'lık, RAM'de duran bir dizin hazırlayacağız. Ayrıca bir diskin okuma ayarını `/sys` üzerinden değiştirip kalıcı yapacağız ve bir disk imajı dosyasını disk gibi kullanacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denendi. `/sys` örneklerindeki disk, bir dosyaya bağlı `/dev/loop0`'dı. Gerçek bir sistemde `/dev/sda` veya `/dev/vda` için aynı dosyalar vardır.

## İçindekiler

1. [Hangi dosya sistemleri sanal?](#1-hangi-dosya-sistemleri-sanal)
2. [/proc: süreçler ve çekirdek](#2-proc-süreçler-ve-çekirdek)
3. [/sys: cihazlar ve sürücüler](#3-sys-cihazlar-ve-sürücüler)
4. [/sys ayarlarını kalıcı yapmak](#4-sys-ayarlarını-kalıcı-yapmak)
5. [tmpfs: RAM'de dosya sistemi](#5-tmpfs-ramde-dosya-sistemi)
6. [Dosya üzerinde sanal disk: loop cihazları](#6-dosya-üzerinde-sanal-disk-loop-cihazları)
7. [Komut özeti](#7-komut-özeti)
8. [Sık yapılan hatalar](#8-sık-yapılan-hatalar)
9. [Alıştırmalar](#9-alıştırmalar)

---

## 1. Hangi dosya sistemleri sanal?

Çekirdeğin tanıdığı dosya sistemleri `/proc/filesystems`'tadır. Başında `nodev` olanlar bir disk gerektirmez:

```bash
grep nodev /proc/filesystems | head -5
grep -v nodev /proc/filesystems
```
```
nodev	sysfs
nodev	tmpfs
nodev	proc
nodev	cgroup
nodev	cgroup2
	ext3
	ext2
	ext4
	squashfs
	fuseblk
	xfs
	erofs
	btrfs
```

Bu liste çekirdeğe ve yüklü modüllere göre değişir. Modül olarak gelen bir dosya sistemi (ör. `vfat`) ilk kullanıldığında yüklenir ve listeye o zaman eklenir.

Bağlı olan sanal dosya sistemleri:

```bash
findmnt -t proc,sysfs,devtmpfs,tmpfs -o TARGET,FSTYPE,SIZE
```
```
TARGET          FSTYPE    SIZE
/proc           proc         0
/sys            sysfs        0
/dev            devtmpfs  3.9G
`-/dev/shm      tmpfs     7.8G
/run            tmpfs     1.6G
`-/run/lock     tmpfs       5M
```

(Çıktı kısaltıldı.)

| Dizin | Tür | İçinde ne var? |
|---|---|---|
| `/proc` | `proc` | Süreçler (`/proc/PID`) ve çekirdek bilgileri, çekirdek parametreleri (`/proc/sys`) |
| `/sys` | `sysfs` | Cihazlar, sürücüler ve ayarları |
| `/dev` | `devtmpfs` | Cihaz dosyaları (`/dev/sda`, `/dev/null`). Çekirdek oluşturur, udev düzenler |
| `/run` | `tmpfs` | Çalışan servislerin PID dosyaları, soketleri. Açılışta boştur |
| `/dev/shm` | `tmpfs` | Süreçler arası paylaşılan bellek |

Bunlar systemd tarafından açılışta otomatik bağlanır, fstab'da yazmazlar.

Sanal dosyaların boyutu yanıltıcıdır:

```bash
ls -l /proc/meminfo
wc -c < /proc/meminfo
du -sh /proc
```
```
-r--r--r-- 1 root root 0 Nov 26 08:03 /proc/meminfo
1503
0	/proc
```

`ls` 0 bayt diyor, ama okuyunca 1503 bayt geliyor. Dosya, okunduğu anda üretiliyor. `du` da `/proc`'u 0 gösterir. Bu yüzden disk doluluğunu araştırırken `/proc` ve `/sys`'i hesaba katmaya gerek yoktur ([Disk Alanı Sorunları](../04-temel-komutlar/15-disk-alani-sorunlari.md)).

---

## 2. /proc: süreçler ve çekirdek

| Dosya veya dizin | Ne gösterir? |
|---|---|
| `/proc/PID/` | Bir sürecin bilgileri: `cmdline`, `status`, `limits`, `fd/`, `cwd` |
| `/proc/meminfo` | Bellek durumu (`free` buradan okur) |
| `/proc/cpuinfo` | İşlemciler |
| `/proc/mounts` | Bağlı dosya sistemleri |
| `/proc/partitions` | Çekirdeğin gördüğü diskler ve bölümler |
| `/proc/swaps` | Etkin swap alanları (`swapon --show` buradan okur) |
| `/proc/filesystems` | Desteklenen dosya sistemleri |
| `/proc/sys/` | Değiştirilebilen çekirdek parametreleri |

`/proc/sys` altındaki her dosya bir `sysctl` parametresidir. `/` yerine `.` koyunca adı çıkar:

```bash
cat /proc/sys/vm/swappiness
echo 30 | sudo tee /proc/sys/vm/swappiness
sysctl vm.swappiness
```
```
60
30
vm.swappiness = 30
```

`/proc/sys/vm/swappiness` ile `vm.swappiness` aynı şeydir. İkisi de yeniden başlatınca kaybolur. Kalıcı yapmak için `/etc/sysctl.d/`: [Kernel Parametreleri](../01-isletim-ve-dagitim/09-kernel-parametreleri.md).

---

## 3. /sys: cihazlar ve sürücüler

`/sys/block/` altında her disk için bir dizin vardır. `queue/` içinde diskin G/Ç ayarları durur:

```bash
cat /sys/block/loop0/queue/scheduler
cat /sys/block/loop0/queue/read_ahead_kb
cat /sys/block/loop0/queue/rotational
cat /sys/block/loop0/size
```
```
[none] mq-deadline kyber bfq
2560
1
6291456
```

| Dosya | Anlamı |
|---|---|
| `queue/scheduler` | G/Ç zamanlayıcısı. Köşeli parantez içindeki etkin olandır |
| `queue/read_ahead_kb` | Sıralı okumada önceden okunan miktar (KB) |
| `queue/rotational` | 1: dönen disk (HDD), 0: SSD |
| `size` | Diskin boyutu, 512 baytlık sektör olarak (6291456 × 512 = 3 GiB) |

Değiştirmek:

```bash
echo mq-deadline | sudo tee /sys/block/loop0/queue/scheduler
cat /sys/block/loop0/queue/scheduler
```
```
none [mq-deadline] kyber bfq
```

> [!WARNING]
> **`sudo echo değer > /sys/...` çalışmaz.** `sudo` sadece `echo`'yu root yapar. `>` yönlendirmesini senin kabuğun, normal kullanıcı olarak yapar:
> ```
> bash: /sys/block/loop0/queue/read_ahead_kb: Permission denied
> ```
> `echo değer | sudo tee dosya` kullan veya `sudo bash -c 'echo değer > dosya'`.

Geçersiz bir değer yazınca:

```
bash: echo: write error: Invalid argument
```

Bazı dosyalar sadece okunabilir ve root bile yazamaz. `size` dosyasına yazmaya çalışınca `Permission denied` alınır. Hangi dosyaların yazılabilir olduğunu `ls -l` gösterir: `-rw-r--r--` yazılabilir, `-r--r--r--` sadece okunur.

`read_ahead_kb` ile `blockdev --getra` aynı ayarı gösterir, ama farklı birimle:

```bash
echo 1024 | sudo tee /sys/block/loop0/queue/read_ahead_kb
sudo blockdev --getra /dev/loop0
```
```
1024
2048
```

1024 KB = 2048 × 512 baytlık sektör.

---

## 4. /sys ayarlarını kalıcı yapmak

`/sys`'e yazılan her şey yeniden başlatınca kaybolur. Kalıcı yapmanın iki yolu var, ikisi de denendi.

### Yol 1: systemd-tmpfiles

`/etc/tmpfiles.d/` içindeki bir dosyada `w` satırı, açılışta bir dosyaya değer yazar:

```bash
echo 'w /sys/block/loop0/queue/read_ahead_kb - - - - 4096' | sudo tee /etc/tmpfiles.d/disk-ayar.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/disk-ayar.conf    # şimdi uygula
cat /sys/block/loop0/queue/read_ahead_kb
```
```
4096
```

Satırdaki `-` işaretleri mod, sahip, grup ve yaş sütunlarıdır. `w` için kullanılmazlar.

### Yol 2: udev kuralı

Diskler için daha uygun yol: disk her göründüğünde (açılışta veya sonradan takılınca) ayar uygulanır.

```bash
echo 'ACTION=="add|change", KERNEL=="loop0", ATTR{queue/scheduler}="kyber"' | sudo tee /etc/udev/rules.d/60-zamanlayici.rules
sudo udevadm control --reload
sudo udevadm trigger --action=change /dev/loop0
cat /sys/block/loop0/queue/scheduler
```
```
none mq-deadline [kyber] bfq
```

`KERNEL=="sd[a-z]"` yazılırsa bütün `sd` diskleri için geçerli olur. `ATTR{...}` yolu, `/sys/block/<disk>/` dizinine göredir.

---

## 5. tmpfs: RAM'de dosya sistemi

tmpfs, dosyaları RAM'de (gerekirse swap'ta) tutar. Çok hızlıdır ve ayrıldığında içindeki her şey kaybolur.

```bash
sudo mkdir -p /mnt/gecici
sudo mount -t tmpfs -o size=50M tmpfs /mnt/gecici
df -h /mnt/gecici
```
```
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            50M     0   50M   0% /mnt/gecici
```

İlk `tmpfs` dosya sistemi türü, ikincisi kaynağın adıdır. Kaynak gerçek bir cihaz olmadığı için oraya istediğin yazılabilir, `df`'de o görünür.

### Boyut sınırı

60 MB yazmaya çalışınca:

```
dd: error writing '/mnt/gecici/buyuk': No space left on device
```

Dosya 50 MB'ta durdu. Kullanılan RAM `/proc/meminfo`'daki `Shmem` satırında görünür: yazmadan önce 27208 kB, sonra 78408 kB oldu (~51 MB arttı).

**`size` vermezsen sınır RAM'in yarısıdır.** 7.8 GB RAM'li makinede `-o size` olmadan bağlanan tmpfs `4.0G` gösterdi. Bu, boyutun ayrılması demek değildir, tmpfs sadece kullanılan kadar RAM tüketir. Ama bir uygulama dizini doldurursa makinenin RAM'inin yarısını yiyebilir.

### Boyutu değiştirmek

```bash
sudo mount -o remount,size=100M /mnt/gecici
df -h /mnt/gecici
```
```
tmpfs           100M   50M   50M  50% /mnt/gecici
```

Bağlıyken, veri kaybolmadan büyüdü. İçindeki veriden küçük bir boyuta (20M) düşürmek reddedildi:

```
mount: /mnt/gecici: mount point not mounted or bad option.
```

### Ayırınca veri kaybolur

```bash
sudo umount /mnt/gecici
sudo mount -t tmpfs -o size=50M tmpfs /mnt/gecici
ls /mnt/gecici          # boş
```

`Shmem` de eski değerine (27208 kB) döndü.

### İzinler

tmpfs varsayılan olarak `drwxrwxrwt` (1777, `/tmp` gibi herkes yazabilir, herkes sadece kendi dosyasını silebilir) izinleriyle oluşur. Değiştirmek için:

```bash
sudo mount -t tmpfs -o size=10M,mode=0750,uid=uygulama tmpfs /mnt/uygulama-gecici
```

Denemede sonuç `drwxr-x---` ve sahibi verilen kullanıcı oldu.

### fstab

```
tmpfs  /mnt/gecici  tmpfs  size=50M,mode=1777  0  0
```

Her açılışta boş olarak bağlanır.

---

## 6. Dosya üzerinde sanal disk: loop cihazları

Bir **loop cihazı** (`/dev/loopN`), sıradan bir dosyayı disk gibi gösterir. Disk imajlarını açmak veya fazladan disk takmadan deneme yapmak için kullanılır. Bu depodaki depolama notlarının hepsi bu şekilde hazırlanmış disklerle denendi.

```bash
sudo truncate -s 64M /srv/sanal.img
sudo mkfs.ext4 /srv/sanal.img
ls -lsh /srv/sanal.img
```
```
4.2M -rw-r--r-- 1 root root 64M Nov 26 09:55 /srv/sanal.img
```

`truncate` boş (sparse) bir dosya oluşturur: boyutu 64 MB görünür, ama diskte sadece yazılan kadar yer kaplar (baştaki 4.2M). `mkfs`'e cihaz yerine dosya vermek çalışır.

### Yol 1: doğrudan mount

```bash
sudo mkdir -p /mnt/sanal
sudo mount /srv/sanal.img /mnt/sanal
findmnt /mnt/sanal
losetup -l -O NAME,BACK-FILE,AUTOCLEAR
```
```
TARGET     SOURCE     FSTYPE OPTIONS
/mnt/sanal /dev/loop6 ext4   rw,relatime
NAME       BACK-FILE       AUTOCLEAR
/dev/loop6 /srv/sanal.img          1
```

`mount`, kaynağın dosya olduğunu görünce boş bir loop cihazı (`loop6`) bulup dosyayı ona bağladı. `-o loop` yazmak gerekmedi. `AUTOCLEAR 1`: dosya sistemi ayrılınca loop cihazı da serbest kalır. Denemede `umount`'tan sonra `losetup -l`'de artık görünmedi.

### Yol 2: losetup ile cihaz oluşturmak

Dosyayı bölümlendirmek, LVM'e vermek veya `mkfs`'ten önce cihaz olarak görmek gerekiyorsa:

```bash
sudo losetup -f --show /srv/sanal.img       # boş ilk cihazı bul, bağla, adını yaz
sudo losetup -l                             # bütün loop cihazları
sudo losetup -d /dev/loop6                  # ayır
```
```
/dev/loop6
```

Bu şekilde oluşturulan cihazda `AUTOCLEAR 0` olur: kendin `losetup -d` ile ayırana kadar kalır. Dosyada bölümler varsa `-P` ile bölümler de görünür (`/dev/loop6p1`, ...).

### fstab

```
/srv/sanal.img  /mnt/sanal  ext4  defaults  0  0
```

`mount -a` bu satırı sorunsuz bağladı. `findmnt --verify` ise bir uyarı verir (`-o loop` eklense de):

```
   [W] non-bind mount source /srv/sanal.img is a directory or regular file
```

Uyarı, kaynağın bir cihaz değil dosya olduğunu söylüyor. Bağlama çalışıyor, uyarı bu durumda göz ardı edilebilir.

---

## 7. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `grep nodev /proc/filesystems` | Disk gerektirmeyen dosya sistemleri |
| `findmnt -t proc,sysfs,tmpfs,devtmpfs` | Bağlı sanal dosya sistemleri |
| `cat /proc/sys/vm/swappiness` | Çekirdek parametresini oku (= `sysctl vm.swappiness`) |
| `cat /sys/block/sda/queue/scheduler` | Diskin G/Ç zamanlayıcısı |
| `echo değer \| sudo tee /sys/...` | `/sys` veya `/proc/sys`'e yaz |
| `/etc/tmpfiles.d/*.conf` + `w yol - - - - değer` | `/sys` ayarını kalıcı yap |
| `/etc/udev/rules.d/*.rules` + `ATTR{...}=` | Disk ayarını kalıcı yap |
| `sudo mount -t tmpfs -o size=50M tmpfs /dizin` | RAM'de dosya sistemi |
| `sudo mount -o remount,size=100M /dizin` | tmpfs boyutunu değiştir |
| `sudo mount dosya.img /dizin` | İmajı bağla (loop otomatik) |
| `sudo losetup -f --show dosya.img` | Dosyayı loop cihazı yap |
| `sudo losetup -l` / `-d /dev/loopN` | Listele / ayır |

---

## 8. Sık yapılan hatalar

- **`sudo echo x > /sys/...` yazmak.** Yönlendirme root olarak yapılmaz. `tee` kullan.
- **`/proc` veya `/sys`'e yazılanın kalıcı olduğunu sanmak.**
- **`/proc`'taki 0 baytlık dosyaların boş olduğunu sanmak.**
- **tmpfs'e `size` vermemek.** Sınır RAM'in yarısı olur.
- **tmpfs'teki verinin yeniden başlatmada veya `umount`'ta kalacağını sanmak.**
- **`losetup` ile oluşturulan loop cihazlarını ayırmamak.** `mount` ile otomatik oluşturulanların aksine kendiliğinden kalkmazlar.
- **`/run` veya `/dev/shm` içine kalıcı olması gereken dosyalar koymak.**

---

## 9. Alıştırmalar

**1.** `/srv/onbellek` dizinine 200 MB sınırlı, RAM'de duran bir dosya sistemi bağla. Açılışta da bağlansın.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /srv/onbellek
echo 'tmpfs  /srv/onbellek  tmpfs  size=200M  0  0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /srv/onbellek
```
</details>

**2.** Aynı dosya sistemini, bağlıyken ve veriyi kaybetmeden 300 MB'a büyüt.

<details>
<summary>Çözüm</summary>

```bash
sudo mount -o remount,size=300M /srv/onbellek
df -h /srv/onbellek
```

Kalıcı olması için fstab'daki `size=200M`'yi de `size=300M` yap.
</details>

**3.** `/dev/vda` diskinin G/Ç zamanlayıcısını `mq-deadline` yap ve açılışta da öyle kalmasını sağla.

<details>
<summary>Çözüm</summary>

```bash
echo mq-deadline | sudo tee /sys/block/vda/queue/scheduler
echo 'ACTION=="add|change", KERNEL=="vda", ATTR{queue/scheduler}="mq-deadline"' | sudo tee /etc/udev/rules.d/60-zamanlayici.rules
sudo udevadm control --reload
cat /sys/block/vda/queue/scheduler
```
</details>

**4.** 100 MB'lık bir imaj dosyası oluştur, XFS ile biçimlendir ve `/mnt/imaj`'a bağla. Hangi loop cihazının kullanıldığını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo truncate -s 100M /srv/imaj.img
sudo mkfs.xfs /srv/imaj.img
sudo mkdir -p /mnt/imaj
sudo mount /srv/imaj.img /mnt/imaj
findmnt /mnt/imaj
losetup -l
```
</details>

**5.** Sistemde `vm.dirty_ratio` parametresinin değerini iki farklı yoldan oku.

<details>
<summary>Çözüm</summary>

```bash
cat /proc/sys/vm/dirty_ratio
sysctl vm.dirty_ratio
```
</details>

---

Önceki konu: [Otomatik Bağlama](09-otomatik-baglama.md) · Sonraki konu: [Depolama Performansı](11-depolama-performansi.md)
