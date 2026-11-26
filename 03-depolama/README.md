# 03 - Depolama

LFCS sınavında bu alanın ağırlığı **%20**. Resmi konu başlıkları:

- LVM depolamayı yapılandırma ve yönetme
- Sanal dosya sistemini yönetme ve yapılandırma
- Dosya sistemleri oluşturma, yönetme ve sorun giderme
- Uzak dosya sistemlerini ve ağ blok cihazlarını kullanma
- Swap alanını yapılandırma ve yönetme
- Dosya sistemi otomatik bağlayıcılarını yapılandırma
- Depolama performansını izleme

## Konular

| # | Konu | İçerik | Resmi madde |
|---|---|---|---|
| 01 | [Disk Bölümleri](01-disk-bolumleri.md) | `lsblk`, `fdisk`, `sfdisk`, `parted`, GPT/MBR, bölüm türleri, `partprobe` | Dosya sistemleri |
| 02 | [Dosya Sistemleri](02-dosya-sistemleri.md) | `mkfs.ext4`, `mkfs.xfs`, etiketler, inode sayısı, `tune2fs`, `xfs_admin`, `blkid` | Dosya sistemleri |
| 03 | [Mount ve fstab](03-mount-ve-fstab.md) | `mount`, `umount`, UUID/LABEL, fstab sütunları, `findmnt --verify`, `.mount` birimleri | Dosya sistemleri |
| 04 | [Mount Seçenekleri](04-mount-secenekleri.md) | `ro`, `noexec`, `nosuid`, `nodev`, `remount`, dosya sistemine özgü seçenekler | Dosya sistemleri |
| 05 | [Swap](05-swap.md) | Swap bölümü ve dosyası, `mkswap`, `swapon`, öncelik, fstab | Swap |
| 06 | [LVM](06-lvm.md) | PV, VG, LV, PE, `vgextend`, `lvresize -r`, `xfs_growfs`, `resize2fs`, silme sırası | LVM |
| 07 | [NFS](07-nfs.md) | `/etc/exports`, seçenekler, `exportfs`, boşluk tuzağı, istemcide mount ve fstab | Uzak dosya sistemleri |
| 08 | [NBD](08-nbd.md) | `nbd-server` yapılandırması, izin tuzağı, `nbd-client`, `/etc/nbdtab` | Ağ blok cihazları |
| 09 | [Otomatik Bağlama](09-otomatik-baglama.md) | autofs, dolaylı ve doğrudan harita, `*` ve `&`, `--ghost`, reload ve restart | Otomatik bağlayıcılar |
| 10 | [Sanal Dosya Sistemleri](10-sanal-dosya-sistemleri.md) | `/proc`, `/sys`, ayarları kalıcı yapmak, `tmpfs`, loop cihazları | Sanal dosya sistemi |
| 11 | [Depolama Performansı](11-depolama-performansi.md) | `iostat -y`, `pidstat -d`, süreci ve dosyayı bulmak, "target is busy", `umount -l`, `dd` ile ölçüm | Performans |

## Nasıl çalışmalı?

01–04 sırayla okunmalı: bir diskin bölümlendirilip, biçimlendirilip, bağlanmasının adımları. Diğer bütün dosyalar bunları kullanıyor. 05–11 birbirinden bağımsız, istediğin sırayla okuyabilirsin. 07 ve 08 ağ üzerinden depolama, 09 da çoğu zaman NFS ile birlikte kullanılıyor. Her dosyanın sonunda çözümlü alıştırmalar var. Önce kendin dene, sonra çözüme bak.

Komutlar Ubuntu 24.04 üzerinde, dosyadan oluşturulmuş disklerde (loop cihazları) denenmiştir. Deneme ortamının çekirdeğinde device-mapper, NFS ve NBD desteği yoktu. LVM'in dosya sistemiyle ilgili kısımları, NFS ve NBD istemci tarafı bu yüzden denenmedi. Hangi kısımların denenmediği ilgili dosyanın başında yazıyor.

Disk doluluğu sorunları (silinmiş ama açık dosyalar, inode tükenmesi, bağlama noktası altında kalan veri) Temel Komutlar bölümünde: [Disk Alanı Sorunları](../04-temel-komutlar/15-disk-alani-sorunlari.md).

---

[Ana sayfaya dön](../README.md)
