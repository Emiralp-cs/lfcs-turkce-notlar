# 04 - Depolama: Alıştırmalar

Depolama bölümündeki 11 konunun bütün alıştırmaları ve çözümleri, toplam **55 soru**. Sorular konu sırasıyla diziliyor.

Çözümler kapalı gelir. Önce kendi sanal makinende dene, sonra **Çözüm**'e tıklayıp karşılaştır. Takıldığın bir konuyu tekrar okumak için başlığın altındaki bağlantıyı kullan.

| # | Konu | Soru |
|---|---|---|
| 01 | [Disk Bölümleri](../04-depolama/01-disk-bolumleri.md) | 5 |
| 02 | [Dosya Sistemleri: XFS ve ext4](../04-depolama/02-dosya-sistemleri.md) | 5 |
| 03 | [Mount ve /etc/fstab](../04-depolama/03-mount-ve-fstab.md) | 5 |
| 04 | [Mount Seçenekleri](../04-depolama/04-mount-secenekleri.md) | 5 |
| 05 | [Swap](../04-depolama/05-swap.md) | 5 |
| 06 | [LVM](../04-depolama/06-lvm.md) | 5 |
| 07 | [NFS](../04-depolama/07-nfs.md) | 5 |
| 08 | [NBD (Network Block Device)](../04-depolama/08-nbd.md) | 5 |
| 09 | [Otomatik Bağlama (autofs)](../04-depolama/09-otomatik-baglama.md) | 5 |
| 10 | [Sanal Dosya Sistemleri](../04-depolama/10-sanal-dosya-sistemleri.md) | 5 |
| 11 | [Depolama Performansı](../04-depolama/11-depolama-performansi.md) | 5 |

---

## 01 - Disk Bölümleri

Konu notu: [Disk Bölümleri](../04-depolama/01-disk-bolumleri.md)

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

## 02 - Dosya Sistemleri: XFS ve ext4

Konu notu: [Dosya Sistemleri: XFS ve ext4](../04-depolama/02-dosya-sistemleri.md)

[Disk Bölümleri](../04-depolama/01-disk-bolumleri.md) alıştırmalarındaki gibi iki bölümlü bir test diski hazırla (ör. `/dev/loop0p1` ve `/dev/loop0p2`).

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

## 03 - Mount ve /etc/fstab

Konu notu: [Mount ve /etc/fstab](../04-depolama/03-mount-ve-fstab.md)

[Dosya Sistemleri](../04-depolama/02-dosya-sistemleri.md) alıştırmalarındaki gibi bir XFS (`/dev/loop0p1`) ve bir ext4 (`/dev/loop0p2`) bölümün olsun.

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

## 04 - Mount Seçenekleri

Konu notu: [Mount Seçenekleri](../04-depolama/04-mount-secenekleri.md)

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

## 05 - Swap

Konu notu: [Swap](../04-depolama/05-swap.md)

**1.** Sistemde etkin swap alanlarını ve toplam swap boyutunu göster.

<details>
<summary>Çözüm</summary>

```bash
swapon --show
free -h
```
</details>

**2.** Bir test diskindeki bölümü (ör. `/dev/loop0p2`) `HizliSwap` etiketiyle swap olarak biçimlendir ve etkinleştir.

<details>
<summary>Çözüm</summary>

```bash
sudo mkswap -L HizliSwap /dev/loop0p2
sudo swapon /dev/loop0p2
swapon --show
```
</details>

**3.** `/swapfile2` adında 300 MB'lık bir swap dosyası oluştur, güvenli izinler ver ve etkinleştir.

<details>
<summary>Çözüm</summary>

```bash
sudo fallocate -l 300M /swapfile2
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
swapon --show
```
</details>

**4.** 2. ve 3. alıştırmadaki swap'ların açılışta etkin olmasını sağla. Dosyanın önceliği bölümden yüksek olsun. Yeniden başlatmadan kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo cp /etc/fstab /etc/fstab.yedek
echo "LABEL=HizliSwap none swap defaults 0 0" | sudo tee -a /etc/fstab
echo "/swapfile2 none swap pri=10 0 0" | sudo tee -a /etc/fstab
sudo swapoff -a
sudo swapon -a
swapon --show          # /swapfile2'nin PRIO değeri 10
```
</details>

**5.** `/swapfile2`'yi tamamen kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo swapoff /swapfile2
sudo sed -i '\#^/swapfile2 #d' /etc/fstab
sudo rm /swapfile2
swapon --show
```
</details>

---

## 06 - LVM

Konu notu: [LVM](../04-depolama/06-lvm.md)

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

## 07 - NFS

Konu notu: [NFS](../04-depolama/07-nfs.md)

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

## 08 - NBD (Network Block Device)

Konu notu: [NBD (Network Block Device)](../04-depolama/08-nbd.md)

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

## 09 - Otomatik Bağlama (autofs)

Konu notu: [Otomatik Bağlama (autofs)](../04-depolama/09-otomatik-baglama.md)

**1.** `/dev/vdb1` (ext4) diski, `/yedek/gunluk` dizinine erişildiğinde bağlansın ve 60 saniye kullanılmayınca ayrılsın.

<details>
<summary>Çözüm</summary>

```bash
echo '/yedek  /etc/auto.yedek  --timeout=60' | sudo tee /etc/auto.master.d/yedek.autofs
echo 'gunluk  -fstype=ext4,rw  :/dev/vdb1' | sudo tee /etc/auto.yedek
sudo systemctl enable autofs
sudo systemctl restart autofs
ls /yedek/gunluk
findmnt -R /yedek
```
</details>

**2.** NFS sunucusu `192.168.50.10`'un `/srv/ev/<kullanıcı>` dizinleri, istemcide `/ev/<kullanıcı>`'ya otomatik bağlansın. Her kullanıcı için ayrı satır yazma.

<details>
<summary>Çözüm</summary>

```bash
echo '/ev  /etc/auto.ev' | sudo tee /etc/auto.master.d/ev.autofs
echo '*  -fstype=nfs4,rw  192.168.50.10:/srv/ev/&' | sudo tee /etc/auto.ev
sudo systemctl restart autofs
ls /ev/ayse
```
</details>

**3.** `/srv/belgeler` dizini tam olarak `/kurum/belgeler` yoluna salt okunur bağlansın.

<details>
<summary>Çözüm</summary>

```bash
echo '/-  /etc/auto.kurum' | sudo tee /etc/auto.master.d/kurum.autofs
echo '/kurum/belgeler  -fstype=bind,ro  :/srv/belgeler' | sudo tee /etc/auto.kurum
sudo systemctl restart autofs
ls /kurum/belgeler
```
</details>

**4.** Alıştırma 1'deki `/yedek` dizinine bakan kullanıcılar `gunluk`'u görebilsin, ama içine girilmeden bağlanmasın.

<details>
<summary>Çözüm</summary>

`/etc/auto.master.d/yedek.autofs`:

```
/yedek  /etc/auto.yedek  --timeout=60 --ghost
```

```bash
sudo systemctl restart autofs
ls /yedek            # gunluk görünür
findmnt -R /yedek    # ama bağlı değil
```
</details>

**5.** Bir haritadaki satır çalışmıyor ve `ls` sadece "No such file or directory" diyor. Sebebi nasıl bulursun?

<details>
<summary>Çözüm</summary>

```bash
sudo automount -m                     # satır okunmuş mu?
sudo sed -i 's/^#\?logging = .*/logging = verbose/' /etc/autofs.conf
sudo systemctl restart autofs
ls /yol/anahtar
sudo journalctl -u autofs -n 20       # mount hatası
sudo dmesg | tail                     # çekirdek tarafındaki hata
```
</details>

---

## 10 - Sanal Dosya Sistemleri

Konu notu: [Sanal Dosya Sistemleri](../04-depolama/10-sanal-dosya-sistemleri.md)

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

## 11 - Depolama Performansı

Konu notu: [Depolama Performansı](../04-depolama/11-depolama-performansi.md)

Hazırlık: Bir dosya sistemi bağla ve arka planda üzerine sürekli yazan bir döngü başlat.

```bash
sudo truncate -s 1G /srv/test.img && sudo mkfs.xfs -q /srv/test.img
sudo mkdir -p /mnt/test && sudo mount /srv/test.img /mnt/test
sudo mkdir /mnt/test/is
sudo bash -c 'cd /mnt/test/is && while :; do dd if=/dev/zero of=veri bs=1M count=300 oflag=direct status=none; done' &
```

**1.** Hangi diskin meşgul olduğunu bul. İlk raporun yanıltmamasına dikkat et.

<details>
<summary>Çözüm</summary>

```bash
iostat -dxyz 1 3
losetup -l          # loop cihazının hangi dosyaya bağlı olduğu
```

`%util` ve `wkB/s` yüksek olan loop cihazı.
</details>

**2.** Diske en çok yazan süreci ve kullanıcısını bul.

<details>
<summary>Çözüm</summary>

```bash
pidstat -d -U 1 3
```

`bash` (döngü) ve kısa ömürlü `dd`'ler. `ps -o pid,ppid,cmd -p <dd PID>` ile `dd`'nin ebeveyninin `bash` olduğunu doğrula.
</details>

**3.** Yazılan dosyanın tam yolunu ve dosya sistemini bul.

<details>
<summary>Çözüm</summary>

```bash
P=$(pgrep -x dd)
pwdx $P
sudo ls -l /proc/$P/fd
df -h /mnt/test/is/veri
```
</details>

**4.** Yazmayı tamamen durdur ve diskin sakinleştiğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo kill <bash PID>
sudo pkill -x dd
iostat -dxyz 1 2        # loop satırı görünmemeli
```
</details>

**5.** Bir terminalde `cd /mnt/test/is` yap. Başka bir terminalden `/mnt/test`'i ayırmayı dene, engelleyeni bul ve doğru şekilde ayır.

<details>
<summary>Çözüm</summary>

```bash
sudo umount /mnt/test          # target is busy
sudo fuser -vm /mnt/test       # cwd (c) olan bash
# ilk terminalde: cd /
sudo umount /mnt/test
```
</details>

---

Önceki bölüm: [İşletim ve Dağıtım](03-isletim-ve-dagitim.md) · Sonraki bölüm: [Ağ Yönetimi](05-ag-yonetimi.md) · [Alıştırmalar ana sayfası](README.md)
