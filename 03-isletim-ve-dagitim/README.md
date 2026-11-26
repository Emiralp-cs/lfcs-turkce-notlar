# 03 - İşletim ve Dağıtım

LFCS sınavında bu alanın ağırlığı **%25**. Resmi konu başlıkları:

- Kernel parametrelerini yapılandırma (kalıcı ve geçici)
- Süreçleri ve servisleri teşhis etme, yönetme, sorun giderme
- Komutları zamanlanmış görev olarak çalıştırma
- Yazılım paketlerini ve depolarını arama, kurma, doğrulama, güncel tutma
- Donanım, işletim sistemi veya dosya sistemi arızalarından kurtarma
- Sanal makineleri yönetme (libvirt)
- Container motorlarını yapılandırma, container oluşturma ve yönetme
- SELinux ile zorunlu erişim denetimi (MAC)

## Konular

| # | Konu | İçerik | Resmi madde |
|---|---|---|---|
| 01 | [Açma, Kapatma ve Target'lar](01-acma-kapatma-ve-targetlar.md) | `reboot`, `shutdown`, `set-default`, `isolate`, rescue/emergency | Süreç ve servisler, Kurtarma |
| 02 | [Arızadan Kurtarma](02-arizadan-kurtarma.md) | GRUB parametreleri, root parolası sıfırlama, fstab hataları, `e2fsck`, `xfs_repair`, SMART | Kurtarma |
| 03 | [Süreçler](03-surecler.md) | `ps`, `pgrep`, `/proc`, `nice`, sinyaller, `jobs`, `lsof` | Süreç ve servisler |
| 04 | [Log Dosyaları](04-log-dosyalari.md) | `/var/log`, `journalctl` (`-u`, `-p`, `-g`, `-b`), `logger`, `last`, logrotate | Süreç ve servisler |
| 05 | [Zamanlanmış Görevler](05-zamanlanmis-gorevler.md) | cron, `/etc/cron.d`, anacron, `at`, systemd timer | Zamanlanmış görevler |
| 06 | [Paket Yönetimi](06-paket-yonetimi.md) | `apt`, `dpkg`, remove/purge, `dpkg -V`, `debsums`, hold | Paketler |
| 07 | [Paket Depoları](07-paket-depolari.md) | `.sources` ve `.list`, imza anahtarları, PPA, `apt-cache policy` | Paketler |
| 08 | [Kaynaktan Yazılım Kurmak](08-kaynaktan-kurma.md) | `configure`, `make`, `make install`, `--prefix`, `-dev` paketleri | Paketler |
| 09 | [Kernel Parametreleri](09-kernel-parametreleri.md) | `sysctl`, `/proc/sys`, `/etc/sysctl.d`, `--system`, dosya sırası | Kernel parametreleri |
| 10 | [SELinux](10-selinux.md) | Bağlamlar, modlar, Ubuntu'da etkinleştirme, `restorecon`, `semanage`, boolean'lar | SELinux |
| 11 | [Container'lar](11-containerlar.md) | `docker run`, port, volume, restart politikası, Dockerfile, Podman | Container'lar |
| 12 | [Sanal Makineler](12-sanal-makineler.md) | `qemu-img`, `virt-install --import`, `virsh`, autostart, CPU/bellek, `undefine` | Sanal makineler |
| 13 | [Bash Script Temelleri](13-bash-script.md) | Shebang, değişkenler, çıkış kodları, `if`, `for`, hata ayıklama | Temel beceri |

Servis oluşturma, `systemctl` ve servis sorun giderme, Temel Komutlar bölümünde: [Servis Oluşturma ve Sorun Giderme](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md).

## Nasıl çalışmalı?

01–05 sistemin nasıl çalıştığını ve sorunları nasıl bulacağını anlatıyor, diğer dosyalarda sürekli kullanılıyor, oradan başla. 06–08 paketler, 09–12 ise birbirinden bağımsız konular, istediğin sırayla okuyabilirsin. Her dosyanın sonunda çözümlü alıştırmalar var. Önce kendin dene, sonra çözüme bak.

Komutlar Ubuntu 24.04 üzerinde denenmiştir. Deneme ortamında çalıştırılamayan kısımlar (SELinux, GRUB ve açılış, Docker Hub'dan image indirmek, KVM) ilgili dosyanın başında belirtilmiştir.

---

Önceki bölüm: [Kullanıcılar ve Gruplar](../02-kullanicilar-ve-gruplar/README.md) · Sonraki bölüm: [Depolama](../04-depolama/README.md) · [Ana sayfaya dön](../README.md)
