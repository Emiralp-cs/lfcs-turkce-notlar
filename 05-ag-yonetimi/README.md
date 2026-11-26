# 05 - Ağ Yönetimi

LFCS sınavında bu alanın ağırlığı **%25**. Resmi konu başlıkları:

- IPv4 ve IPv6 ağ yapılandırması ve isim çözümleme
- Saati zaman sunucularıyla ayarlama ve senkronize etme
- Ağı izleme ve sorun giderme
- OpenSSH sunucu ve istemcisini yapılandırma
- Paket filtreleme, port yönlendirme ve NAT
- Statik route yapılandırması
- Bridge ve bonding cihazları
- Ters proxy ve yük dengeleyiciler

## Konular

| # | Konu | İçerik |
|---|---|---|
| 01 | [IP Adresleme](01-ip-adresleme.md) | IPv4, IPv6, CIDR, alt ağ hesabı, özel adresler |
| 02 | [Ağ Yapılandırması](02-ag-yapilandirmasi.md) | `ip`, statik route, Netplan, DNS, `/etc/hosts` |
| 03 | [Saat Senkronizasyonu](03-saat-senkronizasyonu.md) | Saat dilimi, `timedatectl`, systemd-timesyncd, chrony |
| 04 | [Ağ İzleme ve Sorun Giderme](04-ag-izleme.md) | `ss`, `lsof`, `systemctl`, `ping`, `nc`, `curl`, `tracepath` |
| 05 | [SSH](05-ssh.md) | `sshd_config`, anahtarla giriş, `Match`, `~/.ssh/config`, `known_hosts` |
| 06 | [UFW ile Paket Filtreleme](06-ufw.md) | Kural sırası, `insert`, deny/reject, giden trafik, `limit` |
| 07 | [NAT ve Port Yönlendirme](07-nat-ve-port-yonlendirme.md) | IP yönlendirme, DNAT, MASQUERADE, nftables, UFW `before.rules` |
| 08 | [Bridge ve Bonding](08-bridge-ve-bonding.md) | Bridge, bonding modları, Netplan ile yapılandırma |
| 09 | [Ters Proxy ve Yük Dengeleme](09-ters-proxy-ve-yuk-dengeleme.md) | Nginx `proxy_pass`, `proxy_params`, `upstream`, `weight`, `backup` |

## Nasıl çalışmalı?

Sırayla gitmen iyi olur: 01'deki adresleme bilgisi diğer bütün dosyalarda kullanılıyor. Her dosyanın sonunda çözümlü alıştırmalar var. Önce kendin dene, sonra çözüme bak.

Komutlar Ubuntu 24.04 üzerinde denenmiştir. Bazı dosyalarda deneme ortamında çalıştırılamayan kısımlar (systemd servisleri, IPv6, bonding) dosyanın başında belirtilmiştir.

---

Önceki bölüm: [Depolama](../04-depolama/README.md) · [Ana sayfaya dön](../README.md)
