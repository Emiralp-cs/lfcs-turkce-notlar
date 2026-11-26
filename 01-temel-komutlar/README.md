# 01 - Temel Komutlar

LFCS sınavında bu alanın ağırlığı **%20**. Resmi konu başlıkları:

- Temel Git işlemleri
- Servis oluşturma, yapılandırma ve sorun giderme
- Sistem performansını ve servisleri izleme, sorun giderme
- Uygulama ve servislere özgü kısıtlamaları belirleme
- Disk alanı sorunlarını giderme
- SSL sertifikalarıyla çalışma

## Resmi sınav maddeleri

| # | Konu | İçerik |
|---|---|---|
| 11 | [SSL/TLS ve OpenSSL](11-ssl-tls-ve-openssl.md) | Anahtar, CSR, self-signed sertifika, iç CA, SAN, `s_client`, `update-ca-certificates` |
| 12 | [Git ile Sürüm Kontrolü](12-git.md) | `add`, `commit`, `diff`, `log`, dallar, `merge`, uzak repo, `clone`, çakışma çözme |
| 13 | [Servis Oluşturma ve Sorun Giderme](13-servis-olusturma-ve-sorun-giderme.md) | `systemctl`, unit dosyası yazma, drop-in, `Restart`, `journalctl`, `systemd-analyze verify` |
| 14 | [Performans İzleme](14-performans-izleme.md) | Load average, `top`, `vmstat`, `iostat`, `pidstat`, `systemd-cgtop`, OOM |
| 15 | [Disk Alanı Sorunları](15-disk-alani-sorunlari.md) | `df`, `du`, silinmiş açık dosyalar, inode, ayrılmış bloklar, journal temizliği |
| 16 | [Uygulama ve Servis Kısıtlamaları](16-uygulama-ve-servis-kisitlamalari.md) | Port izinleri, `LimitNOFILE`, `MemoryMax`, `CPUQuota`, `ProtectSystem`, bağımlılıklar |

## Temel beceriler

Bu konular güncel resmi listede ayrı bir madde olarak yok, ama sınavdaki neredeyse her görevde kullanılıyor: log'da arama yapmak, bir dosyanın iznini düzeltmek, sonucu bir dosyaya kaydetmek gibi. Sınava yeni başlıyorsan buradan başla.

| # | Konu | İçerik |
|---|---|---|
| 01 | [Yardım Almak](01-yardim-almak.md) | `man`, `--help`, `apropos`, `/usr/share/doc` |
| 02 | [Dosya ve Dizin İşlemleri](02-dosya-ve-dizin-islemleri.md) | `ls`, `cp`, `mv`, `rm`, `mkdir`, yollar |
| 03 | [Hard ve Soft Link](03-hard-ve-soft-link.md) | `ln`, inode, kırık linkler |
| 04 | [İzinler ve Özel Bitler](04-izinler-ve-ozel-bitler.md) | `chmod`, `chown`, `umask`, SUID, SGID, sticky bit |
| 05 | [find](05-find.md) | Ada, boyuta, zamana, izne göre arama, `-exec` |
| 06 | [Metin İşleme](06-metin-isleme.md) | `head`, `tail`, `sed`, `cut`, `sort`, `uniq`, `tr`, `diff` |
| 07 | [grep ve Düzenli İfadeler](07-grep-ve-regex.md) | `grep` seçenekleri, BRE ve ERE |
| 08 | [Arşivleme ve Sıkıştırma](08-arsivleme-ve-sikistirma.md) | `tar`, `gzip`, `bzip2`, `xz`, `zip` |
| 09 | [Yedekleme: rsync ve dd](09-yedekleme-rsync-ve-dd.md) | `rsync` ile uzak yedek, `dd` ile disk imajı |
| 10 | [Girdi/Çıktı Yönlendirme](10-yonlendirme.md) | `>`, `2>&1`, `\|`, heredoc, `tee`, `sudo tee` |

## Nasıl çalışmalı?

Temel becerilerden (01–10) başlayıp resmi maddelere (11–16) geçmen iyi olur. 13–16 birbirine bağlı: servis yazmayı öğrendikten sonra performansını izlemeyi, diskini ve kısıtlamalarını yönetmeyi görüyorsun. 13–16 servisler, süreçler ve disklerle ilgili olduğu için yer yer [İşletim ve Dağıtım](../03-isletim-ve-dagitim/README.md) ve [Depolama](../04-depolama/README.md) bölümlerine bağlantı veriyor. Zorlanırsan bu dört dosyayı o bölümlerden sonra okuyabilirsin. Her dosyanın sonunda çözümlü alıştırmalar var. Önce kendin dene, sonra çözüme bak.

Komutlar Ubuntu 24.04 üzerinde denenmiştir. Deneme ortamında çalıştırılamayan kısımlar ilgili dosyada belirtilmiştir.

---

Sonraki bölüm: [Kullanıcılar ve Gruplar](../02-kullanicilar-ve-gruplar/README.md) · [Ana sayfaya dön](../README.md)
