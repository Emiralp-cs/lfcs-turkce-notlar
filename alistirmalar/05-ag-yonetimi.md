# 05 - Ağ Yönetimi: Alıştırmalar

Ağ Yönetimi bölümündeki 9 konunun bütün alıştırmaları ve çözümleri, toplam **45 soru**. Sorular konu sırasıyla diziliyor.

Çözümler kapalı gelir. Önce kendi sanal makinende dene, sonra **Çözüm**'e tıklayıp karşılaştır. Takıldığın bir konuyu tekrar okumak için başlığın altındaki bağlantıyı kullan.

| # | Konu | Soru |
|---|---|---|
| 01 | [IP Adresleme: IPv4, IPv6 ve CIDR](../05-ag-yonetimi/01-ip-adresleme.md) | 5 |
| 02 | [Ağ Yapılandırması: ip ve Netplan](../05-ag-yonetimi/02-ag-yapilandirmasi.md) | 5 |
| 03 | [Saat Dilimi ve NTP ile Saat Senkronizasyonu](../05-ag-yonetimi/03-saat-senkronizasyonu.md) | 5 |
| 04 | [Ağ İzleme ve Sorun Giderme](../05-ag-yonetimi/04-ag-izleme.md) | 5 |
| 05 | [SSH Sunucusu ve İstemcisi](../05-ag-yonetimi/05-ssh.md) | 5 |
| 06 | [UFW ile Paket Filtreleme](../05-ag-yonetimi/06-ufw.md) | 5 |
| 07 | [NAT ve Port Yönlendirme](../05-ag-yonetimi/07-nat-ve-port-yonlendirme.md) | 5 |
| 08 | [Bridge ve Bonding](../05-ag-yonetimi/08-bridge-ve-bonding.md) | 5 |
| 09 | [Ters Proxy ve Yük Dengeleme (Nginx)](../05-ag-yonetimi/09-ters-proxy-ve-yuk-dengeleme.md) | 5 |

---

## 01 - IP Adresleme: IPv4, IPv6 ve CIDR

Konu notu: [IP Adresleme: IPv4, IPv6 ve CIDR](../05-ag-yonetimi/01-ip-adresleme.md)

Önce kağıt kalemle çöz, sonra `ipcalc` ile kontrol et.

**1.** `192.168.10.20/24` ile `192.168.10.200/24` aynı ağda mı? Peki `192.168.10.20/25` ile `192.168.10.200/25`?

<details>
<summary>Çözüm</summary>

- `/24`: İkisinin de ağ kısmı `192.168.10`, aynı ağdalar.
- `/25`: Blok büyüklüğü 256 − 128 = 128. Ağlar `.0` ve `.128`. `.20` ilk ağda (`192.168.10.0/25`), `.200` ikinci ağda (`192.168.10.128/25`). **Farklı ağdalar.**

```bash
ipcalc 192.168.10.20/25 | grep Network
ipcalc 192.168.10.200/25 | grep Network
```
</details>

**2.** `172.20.5.9/12` adresinin ağ adresini ve broadcast adresini bul.

<details>
<summary>Çözüm</summary>

`/12` = `255.240.0.0`. 255 olmayan oktet 2. oktet, değeri 240. Blok büyüklüğü 16. 2. oktette ağlar 0, 16, 32... 20, 16 ile 32 arasında.

- Ağ adresi: `172.16.0.0`
- Broadcast: `172.31.255.255`

Bu da özel `172.16.0.0/12` aralığının ta kendisi.
</details>

**3.** Bir ofis ağında 50 cihaz var. Hepsine yetecek en küçük prefix hangisi?

<details>
<summary>Çözüm</summary>

`/27` 30 kullanılabilir adres verir, yetmez. `/26` 62 kullanılabilir adres verir, yeter. Cevap: **`/26`**.
</details>

**4.** Şu IPv6 adresini en kısa haliyle yaz: `2001:0db8:0000:0000:0000:0abc:0000:0001`

<details>
<summary>Çözüm</summary>

Baştaki sıfırlar atılınca: `2001:db8:0:0:0:abc:0:1`. En uzun sıfır bloğu (3 grup) `::` olur:

```
2001:db8::abc:0:1
```

Python ile kontrol:

```bash
python3 -c "import ipaddress; print(ipaddress.ip_address('2001:0db8:0000:0000:0000:0abc:0000:0001'))"
```
</details>

**5.** `fd12:3456::/64`, `fe80::1` ve `2001:db8::1` adreslerinin her birinin türünü yaz.

<details>
<summary>Çözüm</summary>

- `fd12:3456::/64`: unique local, özel bir ağ
- `fe80::1`: link-local, sadece aynı ağ parçasında geçerli
- `2001:db8::1`: belgelerde örnek olarak kullanılmak için ayrılmış aralık
</details>

---

## 02 - Ağ Yapılandırması: ip ve Netplan

Konu notu: [Ağ Yapılandırması: ip ve Netplan](../05-ag-yonetimi/02-ag-yapilandirmasi.md)

**1.** `enp0s8` arayüzüne geçici olarak `192.168.10.20/24` ve `192.168.10.21/24` adreslerini ver. Sonra sadece ikincisini sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ip link set dev enp0s8 up
sudo ip addr add 192.168.10.20/24 dev enp0s8
sudo ip addr add 192.168.10.21/24 dev enp0s8
ip -br addr show enp0s8
sudo ip addr del 192.168.10.21/24 dev enp0s8
ip -br addr show enp0s8
```
</details>

**2.** `192.168.50.0/24` ağına `192.168.10.254` üzerinden giden geçici bir route ekle. `192.168.50.7` adresine giden paketin bu route'u kullandığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo ip route add 192.168.50.0/24 via 192.168.10.254
ip route get 192.168.50.7
```

Çıktıda `via 192.168.10.254` görünmeli.
</details>

**3.** `enp0s8` için kalıcı bir Netplan dosyası yaz: adres `192.168.10.20/24`, varsayılan ağ geçidi `192.168.10.1`, DNS sunucuları `192.168.10.1` ve `9.9.9.9`. Dosyayı uygulamadan önce kontrol et.

<details>
<summary>Çözüm</summary>

`/etc/netplan/99-ofis.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.10.20/24
      nameservers:
        addresses: [192.168.10.1, 9.9.9.9]
      routes:
        - to: default
          via: 192.168.10.1
```

```bash
sudo chmod 600 /etc/netplan/99-ofis.yaml
sudo netplan generate         # hata vermemeli
sudo netplan get ethernets.enp0s8
sudo netplan try
```
</details>

**4.** Bu makinede `nas01` ismi `192.168.10.40` adresine gitsin. DNS'e dokunmadan yap ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo "192.168.10.40  nas01" | sudo tee -a /etc/hosts
getent hosts nas01
```
</details>

**5.** Aşağıdaki Netplan dosyasında üç hata var. Bul.

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
	  dhcp4: false
      addresses:
        192.168.10.20/24
      routes:
        - to: default
          via: 10.0.0.1
```

<details>
<summary>Çözüm</summary>

1. `dhcp4` satırı tab ile girintilenmiş. Sadece boşluk kullanılmalı.
2. `addresses` altındaki adres liste elemanı olmalı: `- 192.168.10.20/24`. Tire olmadan `netplan generate` şu hatayı verir: `Error in network definition: expected sequence`.
3. Ağ geçidi `10.0.0.1`, arayüzün ağında (`192.168.10.0/24`) değil. Ulaşılamayan bir geçit route olarak çalışmaz. Doğrusu `192.168.10.1` gibi aynı ağdan bir adres.
</details>

---

## 03 - Saat Dilimi ve NTP ile Saat Senkronizasyonu

Konu notu: [Saat Dilimi ve NTP ile Saat Senkronizasyonu](../05-ag-yonetimi/03-saat-senkronizasyonu.md)

**1.** Sunucunun saat dilimini `Europe/Istanbul` yap ve doğrula. Sistem ayarını değiştirmeden Tokyo'da saatin kaç olduğunu göster.

<details>
<summary>Çözüm</summary>

```bash
sudo timedatectl set-timezone Europe/Istanbul
timedatectl | grep 'Time zone'
TZ=Asia/Tokyo date
```
</details>

**2.** Buenos Aires'in saat dilimi adını bul.

<details>
<summary>Çözüm</summary>

```bash
timedatectl list-timezones | grep -i buenos
```

Sonuç: `America/Argentina/Buenos_Aires`.
</details>

**3.** timesyncd, `0.tr.pool.ntp.org` ve `1.tr.pool.ntp.org` sunucularını kullansın. Ana yapılandırma dosyasına dokunma. Ayarın okunduğunu doğrula ve uygula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /etc/systemd/timesyncd.conf.d
printf '[Time]\nNTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org\n' | sudo tee /etc/systemd/timesyncd.conf.d/ekip.conf
systemd-analyze cat-config systemd/timesyncd.conf
sudo systemctl restart systemd-timesyncd
timedatectl show-timesync
```
</details>

**4.** `timedatectl` çıktısında `NTP service: inactive` ve `System clock synchronized: no` görüyorsun. Ne yaparsın?

<details>
<summary>Çözüm</summary>

```bash
sudo timedatectl set-ntp true
systemctl status systemd-timesyncd
timedatectl
```

Birkaç saniye sonra hala `synchronized: no` ise güvenlik duvarında UDP 123'ün açık olup olmadığını ve sunucu adlarının çözümlenip çözümlenmediğini kontrol et.
</details>

**5.** Aşağıdaki dosyada iki hata var. Bul.

Dosya adı: `/etc/systemd/timesyncd.conf.d/ntp`

```ini
[Time]
NTP=0.tr.pool.ntp.org,1.tr.pool.ntp.org
```

<details>
<summary>Çözüm</summary>

1. Dosya adı `.conf` ile bitmiyor, okunmaz. `ntp.conf` olmalı.
2. Sunucular virgülle değil boşlukla ayrılmalı: `NTP=0.tr.pool.ntp.org 1.tr.pool.ntp.org`.
</details>

---

## 04 - Ağ İzleme ve Sorun Giderme

Konu notu: [Ağ İzleme ve Sorun Giderme](../05-ag-yonetimi/04-ag-izleme.md)

**1.** Sunucuda TCP 22. portu hangi sürecin dinlediğini bul. Sürecin hangi kullanıcıyla ve hangi komut satırıyla çalıştığını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo ss -ltnp 'sport = :22'
ps -fp PID        # ss çıktısındaki pid= değeri
```

Ubuntu'da süreç `systemd` olarak görünüyorsa SSH socket activation ile çalışıyordur.
</details>

**2.** Sunucuda sadece yerel bağlantıları kabul eden (dışarıdan erişilemeyen) bütün TCP portlarını listele.

<details>
<summary>Çözüm</summary>

```bash
sudo ss -ltnp | grep '127.0.0.1'
```

IPv6 yerel adresleri de dahil etmek için `grep -E '127\.0\.0\.1|\[::1\]'`.
</details>

**3.** `apache2` servisini durdur ve açılışta da başlamamasını sağla. Sonra 80. portun artık dinlenmediğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl disable --now apache2
systemctl is-active apache2      # inactive
systemctl is-enabled apache2     # disabled
sudo ss -ltn 'sport = :80'       # boş
```
</details>

**4.** Bir istemciden `nc -zv 192.168.10.20 8080` çalıştırdın ve `timed out` aldın. Sunucuda `ss` 8080'in `0.0.0.0` üzerinde dinlendiğini gösteriyor. Sorun büyük ihtimalle nerede?

<details>
<summary>Çözüm</summary>

Servis çalışıyor ve her adresi dinliyor. `timed out`, paketlerin cevapsız kaldığını gösteriyor. En olası sebep sunucuda veya aradaki bir cihazda paketi `DROP` eden bir güvenlik duvarı kuralı. `sudo ufw status` veya `sudo iptables -L -n` ile kontrol et.
</details>

**5.** `192.168.50.10` adresine ulaşılamıyor. Paketin hangi route'u kullandığını ve yolda nerede kaybolduğunu bul.

<details>
<summary>Çözüm</summary>

```bash
ip route get 192.168.50.10
tracepath -n 192.168.50.10
```

`tracepath` çıktısında `*` görünmeye başladığı yerden önceki son adres, paketin ulaşabildiği son noktadır.
</details>

---

## 05 - SSH Sunucusu ve İstemcisi

Konu notu: [SSH Sunucusu ve İstemcisi](../05-ag-yonetimi/05-ssh.md)

**1.** Sunucuda root girişini tamamen kapat ve şifreyle girişi engelle. Ana dosyaya dokunma. Ayarın gerçekten geçerli olduğunu doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo tee /etc/ssh/sshd_config.d/10-guvenlik.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t
sudo sshd -T | grep -iE '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication) '
sudo systemctl reload ssh
```

Önce kendi anahtarınla girebildiğinden emin ol.
</details>

**2.** 1. alıştırmadaki ayarlar varken `sshd -T` hala `passwordauthentication yes` gösteriyor. Sebebi ne olabilir?

<details>
<summary>Çözüm</summary>

`sshd_config.d/` altında, senin dosyandan **önce** okunan başka bir dosya (örneğin `05-...conf`) `PasswordAuthentication yes` içeriyor olabilir. İlk değer kazandığı için o geçerli olur.

```bash
grep -ri passwordauthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Ya da kendi dosyanın adı `.conf` ile bitmiyordur ve hiç okunmuyordur.
</details>

**3.** Şifreyle giriş herkes için kapalıyken sadece `yedek` kullanıcısı şifreyle girebilsin.

<details>
<summary>Çözüm</summary>

Dosyanın **sonuna**:

```
Match User yedek
    PasswordAuthentication yes
```

```bash
sudo sshd -t
sudo sshd -T -C user=yedek,host=x,addr=127.0.0.1 | grep -i passwordauth
sudo systemctl reload ssh
```
</details>

**4.** Kendi bilgisayarında bir anahtar oluştur, `192.168.10.20` sunucusundaki `elif` hesabına ekle ve `lab` takma adıyla şifresiz bağlan.

<details>
<summary>Çözüm</summary>

```bash
ssh-keygen -t ed25519 -C "elif@laptop"
ssh-copy-id elif@192.168.10.20
cat >> ~/.ssh/config <<'EOF'
Host lab
    HostName 192.168.10.20
    User elif
EOF
chmod 600 ~/.ssh/config
ssh lab
```
</details>

**5.** Anahtarın sunucuya eklendiği halde `Permission denied (publickey)` alıyorsun. Neleri kontrol edersin?

<details>
<summary>Çözüm</summary>

Sunucuda:

```bash
ls -ld ~/.ssh                     # 700 olmalı, başkası yazamamalı
ls -l ~/.ssh/authorized_keys      # 600 olmalı
sudo journalctl -u ssh | tail     # "bad ownership or modes" var mı?
sudo sshd -T | grep -i pubkeyauthentication
```

İstemcide:

```bash
ls -l ~/.ssh/id_ed25519           # 600 olmalı
ssh -v kullanici@sunucu           # hangi anahtarın denendiğini gösterir
```
</details>

---

## 06 - UFW ile Paket Filtreleme

Konu notu: [UFW ile Paket Filtreleme](../05-ag-yonetimi/06-ufw.md)

**1.** UFW kapalı bir sunucuda SSH'ı (sadece TCP) açık bırakarak güvenlik duvarını etkinleştir ve varsayılan politikaları doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

`Default: deny (incoming), allow (outgoing), disabled (routed)` görmelisin.
</details>

**2.** SSH sadece `10.20.30.0/24` ağından erişilebilir olsun. 1. alıştırmadaki genel kuralı kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw allow from 10.20.30.0/24 to any port 22 proto tcp
sudo ufw delete allow 22/tcp
sudo ufw status numbered
```

Kısıtlayıcı kuralı **önce** ekle, genel kuralı **sonra** sil. Tersini yaparsan arada SSH'ın kapalı kaldığı bir an olur.
</details>

**3.** `10.20.30.0/24` ağında izin var ama `10.20.30.66` adresi tamamen engellensin. Kuralın gerçekten çalışacağından emin ol.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw insert 1 deny from 10.20.30.66
sudo ufw status numbered
```

`deny` kuralının, ağın tamamına izin veren kuralın **üstünde** olduğunu kontrol et.
</details>

**4.** Aşağıdaki listede `10.20.30.66` engellenmiyor. Neden? Nasıl düzeltirsin?

```
[ 1] 22/tcp                     ALLOW IN    10.20.30.0/24
[ 2] Anywhere                   DENY IN     10.20.30.66
```

<details>
<summary>Çözüm</summary>

`10.20.30.66`, `10.20.30.0/24` ağının içinde. Paketler 1. kurala uyuyor ve izin alıyor, 2. kurala hiç bakılmıyor.

```bash
sudo ufw delete 2
sudo ufw insert 1 deny from 10.20.30.66
```
</details>

**5.** Sunucu `enp0s3` arayüzünden `1.1.1.1` adresine bağlanamasın. Diğer giden trafik etkilenmesin. Sonra kuralı sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ufw deny out on enp0s3 to 1.1.1.1
ping -c 2 1.1.1.1          # cevap gelmemeli
ping -c 2 8.8.8.8          # çalışmalı
sudo ufw delete deny out on enp0s3 to 1.1.1.1
```
</details>

---

## 07 - NAT ve Port Yönlendirme

Konu notu: [NAT ve Port Yönlendirme](../05-ag-yonetimi/07-nat-ve-port-yonlendirme.md)

Aşağıdaki alıştırmalarda ağ geçidinin arayüzleri: internet `enp0s3`, iç ağ `enp0s8` (`192.168.100.0/24`).

**1.** IP yönlendirmeyi kalıcı olarak aç ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/90-yonlendirme.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward
```
</details>

**2.** Ağ geçidinin `2222` portuna internetten gelen bağlantılar iç ağdaki `192.168.100.20` makinesinin `22` portuna gitsin. iptables kullan.

<details>
<summary>Çözüm</summary>

```bash
sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 2222 -j DNAT --to-destination 192.168.100.20:22
sudo iptables -t nat -L PREROUTING -n -v
```

`192.168.100.20`'nin varsayılan ağ geçidi bu sunucu değilse, iç arayüzde masquerade de gerekir.
</details>

**3.** İç ağdaki bütün makineler internete çıkabilsin. Kuralları kalıcı yap.

<details>
<summary>Çözüm</summary>

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
</details>

**4.** 3. alıştırmayı iptables yerine UFW ile yap. UFW zaten açık ve SSH'a izin veriyor.

<details>
<summary>Çözüm</summary>

`/etc/ufw/sysctl.conf`:

```
net/ipv4/ip_forward=1
```

`/etc/ufw/before.rules` dosyasının sonuna:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.100.0/24 -o enp0s3 -j MASQUERADE
COMMIT
```

```bash
sudo ufw route allow in on enp0s8 out on enp0s3 from 192.168.100.0/24
sudo ufw reload
```
</details>

**5.** NAT kuralları yüklü, yönlendirme açık, ama dışarıdan iç web sunucusuna bağlanılamıyor. UFW açık. Neleri kontrol edersin?

<details>
<summary>Çözüm</summary>

```bash
sysctl net.ipv4.ip_forward                   # 1 mi?
sudo iptables -t nat -L PREROUTING -n -v     # DNAT kuralı var mı, sayaç artıyor mu?
sudo ufw status verbose                      # routed: deny mı?
sudo ufw status numbered                     # ALLOW FWD kuralı var mı?
```

En sık sebep `ufw route allow` kuralının eksik olması. Sayaç artıyor ama bağlantı yine olmuyorsa iç sunucunun varsayılan ağ geçidini kontrol et: cevaplar ağ geçidine dönüyor mu?
</details>

---

## 08 - Bridge ve Bonding

Konu notu: [Bridge ve Bonding](../05-ag-yonetimi/08-bridge-ve-bonding.md)

**1.** `enp0s8` ve `enp0s9` arayüzlerini `br1` adlı geçici bir bridge'e ekle, bridge'e `10.50.0.1/24` adresini ver ve portları doğrula. Sonra bridge'i sil.

<details>
<summary>Çözüm</summary>

```bash
sudo ip link add br1 type bridge
sudo ip link set enp0s8 master br1
sudo ip link set enp0s9 master br1
sudo ip link set br1 up
sudo ip addr add 10.50.0.1/24 dev br1
ip link show master br1
bridge link
sudo ip link delete br1
```
</details>

**2.** Aynı bridge'i Netplan ile kalıcı yap. Bridge adresini DHCP'den alsın.

<details>
<summary>Çözüm</summary>

`/etc/netplan/99-kopru.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
    enp0s9:
      dhcp4: false
  bridges:
    br1:
      dhcp4: true
      interfaces: [enp0s8, enp0s9]
```

```bash
sudo chmod 600 /etc/netplan/99-kopru.yaml
sudo netplan generate
sudo netplan try
```
</details>

**3.** `enp0s8` ve `enp0s9`'u `bond0` altında birleştir. Switch'e dokunmadan hem yedeklilik hem gelen ve giden trafik için yük dağılımı istiyorsun. Bond'un adresi `192.168.10.20/24` olsun.

<details>
<summary>Çözüm</summary>

Switch ayarı gerektirmeden iki yönde yük dağıtan mod `balance-alb`.

```yaml
network:
  version: 2
  ethernets:
    enp0s8: {}
    enp0s9: {}
  bonds:
    bond0:
      addresses: [192.168.10.20/24]
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: balance-alb
        mii-monitor-interval: 100
```

```bash
sudo chmod 600 /etc/netplan/99-bag.yaml
sudo netplan try
cat /proc/net/bonding/bond0
```
</details>

**4.** Aşağıdaki durumların her biri için bridge mi bond mu kullanırsın?

a) Sunucudaki sanal makineler ofis ağında kendi adreslerini alsın.
b) Sunucunun ağ kartı bozulursa bağlantı kesilmesin.
c) Switch LACP destekliyor, iki 1 Gbit kart birlikte kullanılsın.

<details>
<summary>Çözüm</summary>

a) Bridge. b) Bond (`active-backup` yeterli). c) Bond, `802.3ad` modu.
</details>

**5.** Aşağıdaki Netplan dosyasında iki hata var. Bul.

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses: [192.168.10.20/24]
    enp0s9: {}
  bonds:
    bond0:
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: 1
```

<details>
<summary>Çözüm</summary>

1. Adres bond'a ait karta (`enp0s8`) verilmiş. Adres `bond0`'a verilmeli.
2. `mode: 1` geçersiz. Netplan isim ister: `mode: active-backup`.

Ayrıca `mii-monitor-interval` eklemek iyi olur, yoksa arıza fark edilmeyebilir.
</details>

---

## 09 - Ters Proxy ve Yük Dengeleme (Nginx)

Konu notu: [Ters Proxy ve Yük Dengeleme (Nginx)](../05-ag-yonetimi/09-ters-proxy-ve-yuk-dengeleme.md)

**1.** Nginx, 80. porta gelen bütün istekleri `10.0.0.21:3000` adresindeki uygulamaya iletsin. Uygulama istemcinin gerçek adresini görebilsin.

<details>
<summary>Çözüm</summary>

`/etc/nginx/sites-available/uygulama.conf`:

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://10.0.0.21:3000;
        include proxy_params;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/uygulama.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
</details>

**2.** Sadece `/docs/` altındaki istekler `10.0.0.22:8080`'e gitsin ve arka uç bu istekleri `/docs/` öneki olmadan alsın. `/docs/kurulum.html` isteği arka uca `/kurulum.html` olarak gitmeli.

<details>
<summary>Çözüm</summary>

```nginx
location /docs/ {
    proxy_pass http://10.0.0.22:8080/;
    include proxy_params;
}
```

`proxy_pass`'in sonundaki `/`, location önekinin (`/docs/`) silinmesini sağlar.
</details>

**3.** `10.0.0.31`, `10.0.0.32` ve `10.0.0.33` sunucuları arasında yük dengeleme yap. `10.0.0.31` diğerlerinin iki katı istek alsın, istekler en az meşgul sunucuya gitsin.

<details>
<summary>Çözüm</summary>

```nginx
upstream uygulama {
    least_conn;
    server 10.0.0.31 weight=2;
    server 10.0.0.32;
    server 10.0.0.33;
}

server {
    listen 80;
    location / {
        proxy_pass http://uygulama;
        include proxy_params;
    }
}
```
</details>

**4.** 3. alıştırmada `10.0.0.32` bakıma alınacak. Ayrıca `10.0.0.40` sadece diğerleri çökerse kullanılacak bir yedek olsun.

<details>
<summary>Çözüm</summary>

```nginx
upstream uygulama {
    least_conn;
    server 10.0.0.31 weight=2;
    server 10.0.0.32 down;
    server 10.0.0.33;
    server 10.0.0.40 backup;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```
</details>

**5.** Yapılandırmayı etkinleştirdin, `nginx -t` başarılı, ama tarayıcıda hala "Welcome to nginx!" sayfası görünüyor. Sebebi ne olabilir? Başka bir durumda da `502 Bad Gateway` alıyorsun. İkisini nasıl incelersin?

<details>
<summary>Çözüm</summary>

"Welcome to nginx!": `sites-enabled/default` hala etkin ve `default_server` olarak istekleri alıyor.

```bash
ls -l /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

502: Nginx arka uca ulaşamıyor.

```bash
sudo tail /var/log/nginx/error.log      # hangi upstream'e bağlanılamadı?
curl http://10.0.0.31:8000/              # arka uç doğrudan cevap veriyor mu?
```
</details>

---

Önceki bölüm: [Depolama](04-depolama.md) · [Alıştırmalar ana sayfası](README.md)
