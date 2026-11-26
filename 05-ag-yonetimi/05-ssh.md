# SSH Sunucusu ve İstemcisi

> **Resmi sınav maddesi:** Configure the OpenSSH server and client

SSH, uzaktaki bir Linux sunucusunu güvenli şekilde yönetmenin standart yolu. İki tarafı var:

- **Sunucu (`sshd`):** Yönetilecek makinede çalışır, gelen bağlantıları kabul eder.
- **İstemci (`ssh`):** Senin bilgisayarında çalışır, sunucuya bağlanır.

Senaryo: Ekibin sunucusunun iki adresi var: ofis ağında `192.168.10.20`, internete açık `203.0.113.10`. Sunucu şöyle ayarlanacak:

- SSH sadece ofis ağından erişilebilir olacak.
- Root ile doğrudan giriş kapanacak.
- Herkes şifre yerine SSH anahtarıyla girecek. Tek istisna stajyer Mert, onun anahtarı hazır olana kadar şifreyle girebilecek.
- Ayşe kendi bilgisayarından sunucuya kısa bir takma adla bağlanacak.

> [!NOTE]
> Bütün ayarlar Ubuntu 24.04 ve OpenSSH 9.6 ile denenmiştir. `sshd` elle başlatılıp aynı makineden ve ağ namespace'lerinden gerçek bağlantılar kuruldu. Deneme ortamı systemd ile çalışmadığı için `systemctl` komutları çalıştırılamadı, ayar değişiklikleri `sshd`'ye doğrudan sinyal gönderilerek uygulandı.

## İçindekiler

1. [Sunucu ve istemci dosyaları](#1-sunucu-ve-istemci-dosyaları)
2. [sshd_config'in yapısı](#2-sshd_configin-yapısı)
3. [Hangi ayar geçerli? sshd_config.d ve "ilk değer kazanır"](#3-hangi-ayar-geçerli-sshd_configd-ve-ilk-değer-kazanır)
4. [Ayarları kontrol etmek ve uygulamak](#4-ayarları-kontrol-etmek-ve-uygulamak)
5. [Hangi adreste ve portta dinlenecek?](#5-hangi-adreste-ve-portta-dinlenecek)
6. [Root girişi ve şifreyle girişi kapatmak](#6-root-girişi-ve-şifreyle-girişi-kapatmak)
7. [Kullanıcıya özel istisna: Match](#7-kullanıcıya-özel-istisna-match)
8. [Diğer sık kullanılan ayarlar](#8-diğer-sık-kullanılan-ayarlar)
9. [SSH anahtarı oluşturmak ve sunucuya eklemek](#9-ssh-anahtarı-oluşturmak-ve-sunucuya-eklemek)
10. [Dosya izinleri](#10-dosya-izinleri)
11. [SSH istemci ayarları: ssh_config ve ~/.ssh/config](#11-ssh-istemci-ayarları-ssh_config-ve-sshconfig)
12. [known_hosts ve "host key değişti" uyarısı](#12-known_hosts-ve-host-key-değişti-uyarısı)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Sunucu ve istemci dosyaları

İki dosyanın adı neredeyse aynı. Fazladan `d` harfi **daemon**, yani sunucu demek:

| Dosya | Ne için? |
|---|---|
| `/etc/ssh/sshd_config` | Sunucu ayarları (bu makineye gelen bağlantılar) |
| `/etc/ssh/sshd_config.d/*.conf` | Sunucu için ek ayar dosyaları |
| `/etc/ssh/ssh_config` | İstemci ayarları, bu makinedeki herkes için |
| `/etc/ssh/ssh_config.d/*.conf` | İstemci için ek ayar dosyaları |
| `~/.ssh/config` | Bir kullanıcının kendi istemci ayarları |

Sunucunun davranışını değiştirmek istiyorsan `sshd_config`, bağlanma şeklini değiştirmek istiyorsan `ssh_config` veya `~/.ssh/config`.

---

## 2. sshd_config'in yapısı

Ubuntu 24.04'teki dosyanın başı şöyle:

```
Include /etc/ssh/sshd_config.d/*.conf
```

Dosyanın geri kalanında çoğu satır `#` ile başlar:

```
#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
#PermitRootLogin prohibit-password
#PubkeyAuthentication yes
#PasswordAuthentication yes
KbdInteractiveAuthentication no
```

`#` ile başlayan satırlar etkin değildir, sadece **varsayılan değeri** gösterir. Yani `#Port 22` satırı "port değiştirilmezse 22" demek. Bir ayarı değiştirmek için `#`'yi kaldırıp değeri yazarsın.

Sunucunun o an kullandığı gerçek ayarların tamamını görmek için:

```bash
sudo sshd -T
```

Denemede varsayılan kurulumdan bazı satırlar:

```
port 22
addressfamily any
listenaddress [::]:22
listenaddress 0.0.0.0:22
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes
kbdinteractiveauthentication no
x11forwarding yes
```

`without-password`, `prohibit-password`'un eski adıdır, aynı anlama gelir: root sadece anahtarla girebilir, şifreyle giremez.

---

## 3. Hangi ayar geçerli? sshd_config.d ve "ilk değer kazanır"

Bu konu sık karıştırılır ve sınavda tuzak olabilir.

**Kural:** `sshd_config`'te aynı ayar birden fazla kez yazılmışsa **ilk okunan değer** geçerli olur. Sonrakiler yok sayılır.

Ubuntu'da `Include` satırı dosyanın **en başında**. Bu yüzden `sshd_config.d/` altındaki dosyalar ana dosyadan **önce** okunur ve ana dosyadaki ayarları ezer.

Denedik:

| Durum | `sshd -T` sonucu |
|---|---|
| Ana dosyanın sonuna `PasswordAuthentication no` | `no` |
| Ayrıca `sshd_config.d/50-cloud-init.conf` içinde `PasswordAuthentication yes` | `yes`, ek dosya kazandı |
| Aynı ek dosyada önce `no`, sonra `yes` | `no`, ilk değer kazandı |
| Ek dosya `.conf` uzantısız (`ekip`) | Dosya hiç okunmadı |

> [!WARNING]
> Bulut sunucularında `sshd_config.d/50-cloud-init.conf` içinde `PasswordAuthentication yes` olabilir. Ana dosyada şifreyi kapatsan bile bu dosya yüzünden şifreyle giriş açık kalır. Bir ayar beklediğin gibi çalışmıyorsa önce `sudo sshd -T | grep -i ayaradi` ile gerçek değere, sonra `ls /etc/ssh/sshd_config.d/` ile ek dosyalara bak.

Bu yüzden önerilen yöntem, kendi ayarlarını `sshd_config.d/` altında **küçük numaralı** bir dosyaya yazmak. Örneğin `10-ekip.conf`, `50-cloud-init.conf`'tan önce okunur ve onu ezer.

> [!NOTE]
> Bu kural sadece `sshd_config` için geçerli. Netplan'da sonra okunan dosya kazanıyordu ([Ağ Yapılandırması](02-ag-yapilandirmasi.md#dosya-adı-neden-99-ile-başlıyor)). İki aracın mantığı birbirinin tersi.

---

## 4. Ayarları kontrol etmek ve uygulamak

### Sözdizimini kontrol et

Her değişiklikten sonra, servisi yeniden yüklemeden önce:

```bash
sudo sshd -t
```

Sorun yoksa hiçbir şey yazmaz. Yazım hatası varsa satırı söyler:

```
/etc/ssh/sshd_config: line 132: Bad configuration option: PasswordAuthentcation
/etc/ssh/sshd_config: terminating, 1 bad configuration options
```

Hatalı bir ayarla servisi yeniden başlatırsan SSH açılmayabilir ve uzaktan bağlıysan sunucuya erişimi kaybedersin.

### Değişikliği uygula

```bash
sudo systemctl reload ssh      # RHEL ailesinde: sshd
```

`reload`, açık SSH bağlantılarını koparmadan yeni ayarları okutur.

> [!IMPORTANT]
> Ubuntu 22.10 ve sonrasında SSH, `ssh.socket` ile başlatılır: 22. portu systemd dinler. Ubuntu 24.04'ün kendi `sshd_config` dosyasındaki açıklamaya göre `Port`, `AddressFamily` veya `ListenAddress` değiştirdiğinde `reload` yetmez:
> ```bash
> sudo systemctl daemon-reload
> sudo systemctl restart ssh.socket
> ```
> Diğer ayarlar için `reload ssh` yeterli.

> [!TIP]
> SSH ayarlarını uzaktan değiştirirken mevcut oturumu kapatma. Yeni ayarı uyguladıktan sonra **ikinci bir terminalden** bağlanmayı dene. Bir şey bozulduysa açık kalan ilk oturumdan düzeltirsin.

---

## 5. Hangi adreste ve portta dinlenecek?

| Ayar | Değerler |
|---|---|
| `Port` | Dinlenecek port, varsayılan 22 |
| `AddressFamily` | `any` (IPv4 ve IPv6), `inet` (sadece IPv4), `inet6` (sadece IPv6) |
| `ListenAddress` | Dinlenecek adres. Yazılmazsa bütün adresler (`0.0.0.0` ve `::`) |

Senaryoda SSH sadece ofis ağından erişilebilir olmalı. Denemede portu da 2222'ye taşıdık:

```
Port 2222
AddressFamily inet
ListenAddress 192.168.10.20
```

`sshd -T` ve `ss` ile kontrol:

```
port 2222
addressfamily inet
listenaddress 192.168.10.20:2222
```
```
LISTEN 0      128    192.168.10.20:2222      0.0.0.0:*    users:(("sshd",pid=1349,fd=4))
```

İki farklı ağdan bağlanmayı denedik:

```
Ofisten (192.168.10.50 -> 192.168.10.20):   Connection to 192.168.10.20 2222 port [tcp/*] succeeded!
Dışarıdan (203.0.113.99 -> 203.0.113.10):  nc: connect to 203.0.113.10 port 2222 (tcp) failed: Connection refused
```

Sunucu internete açık adreste hiç dinlemediği için dışarıdan bağlantı reddedildi.

> [!WARNING]
> Portu değiştirirsen güvenlik duvarında da yeni portu açmayı unutma ([UFW](06-ufw.md) dosyasında). Önce güvenlik duvarını, sonra SSH'ı değiştir, yoksa kendini dışarıda bırakırsın.

---

## 6. Root girişi ve şifreyle girişi kapatmak

Senaryodaki ayarları `sshd_config.d/10-ekip.conf` dosyasına yazalım:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
```

| Ayar | Değer | Etkisi |
|---|---|---|
| `PermitRootLogin` | `prohibit-password` (varsayılan) | Root sadece anahtarla girebilir |
| | `no` | Root hiçbir şekilde doğrudan giremez |
| | `yes` | Root şifreyle de girebilir (önerilmez) |
| `PasswordAuthentication` | `no` | Şifreyle giriş kapalı |
| `KbdInteractiveAuthentication` | `no` | Klavyeden etkileşimli giriş kapalı. Ubuntu'da zaten `no` |

Şifreyle girişi tamamen kapatmak için ikisinin de `no` olması gerekir, çünkü `KbdInteractiveAuthentication` da PAM üzerinden şifre sorabilir.

Uyguladıktan sonra denedik:

```
mert, şifreyle:        mert@127.0.0.1: Permission denied (publickey).
ayse, anahtarla:       anahtar hala calisiyor
root, anahtarla bile:  root@127.0.0.1: Permission denied (publickey).
```

Hata mesajındaki `(publickey)`, sunucunun artık sadece anahtarı kabul ettiğini gösteriyor.

> [!CAUTION]
> Şifreyle girişi kapatmadan önce kendi anahtarınla girebildiğini mutlaka test et. Aksi halde sunucuya erişimin tamamen kesilir.

---

## 7. Kullanıcıya özel istisna: Match

Genel ayarlar herkese uygulanır. Belirli bir kullanıcı, grup veya adres için farklı davranış istiyorsan `Match` bloğu kullanılır. Mert şifreyle girebilsin:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no

Match User mert
    PasswordAuthentication yes
```

Denemede:

```
mert, şifreyle:  mert sifreyle girdi
```

Ayşe ise hala şifreyle giremiyordu. Bir kullanıcı için hangi ayarın geçerli olacağını bağlanmadan da test edebilirsin:

```bash
sudo sshd -T -C user=mert,host=x,addr=192.168.10.50 | grep -i passwordauth
sudo sshd -T -C user=ayse,host=x,addr=192.168.10.50 | grep -i passwordauth
```
```
passwordauthentication yes
passwordauthentication no
```

`Match` ile kullanılabilecek diğer koşullar: `Match Group stajyer`, `Match Address 192.168.10.0/24`.

> [!WARNING]
> `Match` bloğu bir sonraki `Match` satırına veya dosyanın sonuna kadar devam eder. Arkasına yazdığın her ayar artık sadece o kullanıcıya uygulanır. Denemede `Match User mert` bloğunun altına `X11Forwarding no` yazdık: Mert için `no`, Ayşe için hala `yes` oldu. **`Match` bloklarını her zaman dosyanın en sonuna yaz.**

---

## 8. Diğer sık kullanılan ayarlar

| Ayar | Ubuntu varsayılanı | Anlamı |
|---|---|---|
| `PubkeyAuthentication` | `yes` | Anahtarla girişe izin ver |
| `UsePAM` | `yes` | Girişte PAM modüllerini kullan |
| `X11Forwarding` | `yes` | Grafik uygulamaları SSH üzerinden göster |
| `PrintMotd` | `no` | Girişte `/etc/motd` mesajını SSH göstermesin (Ubuntu'da PAM gösteriyor) |
| `ClientAliveInterval` | `0` | Sunucunun istemciye kaç saniyede bir "orada mısın?" diye soracağı. `0` = sormaz |
| `ClientAliveCountMax` | `3` | Cevap gelmezse kaç sorudan sonra bağlantının kesileceği |
| `AllowUsers` / `AllowGroups` | Yok | Sadece listelenen kullanıcılar/gruplar girebilir |

Örneğin boşta kalan bağlantıları 5 dakika sonra kesmek için: `ClientAliveInterval 100` ve `ClientAliveCountMax 3` (100 × 3 = 300 saniye).

Bütün ayarların açıklaması için `man sshd_config`.

---

## 9. SSH anahtarı oluşturmak ve sunucuya eklemek

Anahtarla girişte iki dosya vardır:

- **Özel anahtar** (`id_ed25519`): Senin bilgisayarında kalır, kimseyle paylaşılmaz.
- **Açık anahtar** (`id_ed25519.pub`): Sunucudaki `~/.ssh/authorized_keys` dosyasına eklenir.

### Anahtar oluşturmak

Ayşe kendi bilgisayarında:

```bash
ssh-keygen -t ed25519 -C "ayse@laptop"
```

Dosya yerini sorduğunda Enter'a basarak varsayılanı kabul et. Passphrase sorulur: boş bırakabilirsin ama bir passphrase koyarsan anahtar dosyası çalınsa bile tek başına işe yaramaz.

```bash
ls -l ~/.ssh
```
```
-rw------- 1 ayse ayse 399 Nov 25 14:20 id_ed25519
-rw-r--r-- 1 ayse ayse  93 Nov 25 14:20 id_ed25519.pub
```

`-C`, anahtarın sonuna eklenen bir açıklamadır. `authorized_keys`'te hangi anahtarın kime ait olduğunu anlamayı kolaylaştırır.

### Açık anahtarı sunucuya kopyalamak

```bash
ssh-copy-id ayse@192.168.10.20
```

Bu komut bir kez şifreyle bağlanır ve açık anahtarı sunucudaki `~/.ssh/authorized_keys` dosyasına ekler. Dizini ve dosyayı doğru izinlerle oluşturur. Denemede sunucuda oluşan dosyalar:

```
drwx------ 2 ayse ayse 4096 Nov 25 14:20 .
-rw------- 1 ayse ayse   93 Nov 25 14:20 authorized_keys
```

Artık şifre sorulmadan girilir:

```bash
ssh ayse@192.168.10.20
```

Port 22 değilse: `ssh-copy-id -p 2222 ...` ve `ssh -p 2222 ...`.

### ssh-copy-id yoksa: elle eklemek

Açık anahtarın içeriğini (`cat ~/.ssh/id_ed25519.pub`, tek satır) kopyala ve sunucuda o kullanıcı olarak:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA... ayse@laptop" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

`>>` kullan, `>` dosyadaki diğer anahtarları siler.

---

## 10. Dosya izinleri

SSH, izinleri gevşek olan dosyaları güvenlik nedeniyle reddeder. Denediğimiz durumlar:

| Dosya | İzin | Sonuç |
|---|---|---|
| Sunucu: `~/.ssh/authorized_keys` | `600` | Çalışır (önerilen) |
| | `644` | Çalışır |
| | `666` (herkes yazabilir) | Reddedildi |
| Sunucu: `~/.ssh` dizini | `700` | Çalışır (önerilen) |
| | `777` | Reddedildi |
| İstemci: `~/.ssh/id_ed25519` | `600` | Çalışır |
| | `644` | Reddedildi |
| İstemci: `~/.ssh/config` | `600` veya `644` | Çalışır |
| | `666` | `Bad owner or permissions on ~/.ssh/config` |

Kural: **başkalarının yazabildiği** dosya ve dizinler reddedilir. Özel anahtar ise başkaları tarafından **okunabiliyorsa** bile reddedilir.

`authorized_keys` reddedildiğinde istemci sadece `Permission denied` görür, sebebini söylemez. Sebep sunucunun loglarındadır:

```
Authentication refused: bad ownership or modes for file /home/ayse/.ssh/authorized_keys
```

Ubuntu'da bu loglar için: `sudo journalctl -u ssh` veya `/var/log/auth.log`.

Özel anahtarın izni açıksa istemci açıkça uyarır:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/home/ayse/.ssh/id_ed25519' are too open.
```

Güvenli değerler: dizin `700`, `authorized_keys` ve özel anahtar `600`, `config` `600`.

---

## 11. SSH istemci ayarları: ssh_config ve ~/.ssh/config

### ~/.ssh/config ile takma ad

Her seferinde `ssh -p 2222 ayse@192.168.10.20` yazmak yerine Ayşe kendi bilgisayarında bir takma ad tanımlayabilir:

```bash
vim ~/.ssh/config
```

```
Host ekip-sunucu
    HostName 192.168.10.20
    Port 2222
    User ayse
```

```bash
chmod 600 ~/.ssh/config
ssh ekip-sunucu
```

Takma ad `scp`, `rsync` ve `ssh-copy-id` gibi SSH kullanan bütün araçlarda çalışır: `scp rapor.txt ekip-sunucu:/tmp/`.

Bir takma ad için hangi ayarların kullanılacağını bağlanmadan görmek için:

```bash
ssh -G ekip-sunucu | grep -E '^(hostname|port|user) '
```
```
user ayse
hostname 192.168.10.20
port 2222
```

### Herkes için istemci ayarları

`/etc/ssh/ssh_config`, o makinedeki bütün kullanıcıların istemci ayarlarını belirler. Ubuntu'daki dosyada `Host *` (her sunucu için) bloğunun altında varsayılan değerler yorum satırı olarak durur:

```
Host *
#   ForwardAgent no
#   ForwardX11 no
#   PasswordAuthentication yes
...
#   StrictHostKeyChecking ask
#   IdentityFile ~/.ssh/id_rsa
#   IdentityFile ~/.ssh/id_dsa
#   IdentityFile ~/.ssh/id_ecdsa
#   IdentityFile ~/.ssh/id_ed25519
#   Port 22
```

Ana dosyayı değiştirmek yerine `/etc/ssh/ssh_config.d/` altına bir `.conf` dosyası eklemek daha güvenli, paket güncellemeleri bu dizine dokunmaz. Örneğin şirketin iç sunucularının hepsi 2222 kullanıyorsa:

```
Host 192.168.10.*
    Port 2222
```

İstemci tarafında da öncelik aynı: ilk bulunan değer geçerli. Ayar sırası `komut satırı (-p)` → `~/.ssh/config` → `/etc/ssh/ssh_config`. Kullanıcının kendi dosyası sistem dosyasının önüne geçer.

---

## 12. known_hosts ve "host key değişti" uyarısı

Her SSH sunucusunun kendi kimlik anahtarları var (`/etc/ssh/ssh_host_*_key`). Bir sunucuya ilk bağlandığında istemci bu anahtarın parmak izini gösterir ve onay ister:

```
The authenticity of host '192.168.10.20 (192.168.10.20)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes` dediğinde anahtar `~/.ssh/known_hosts` dosyasına kaydedilir. Sonraki bağlantılarda istemci, sunucunun aynı anahtarı göstermesini bekler.

Sunucunun parmak izini sunucuda görmek için:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

### Anahtar değişirse

Denemede sunucunun anahtarını değiştirip tekrar bağlandık:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
```

Bağlantı reddedildi. Bu uyarı iki sebepten çıkar: sunucu yeniden kurulmuştur (normal) ya da biri araya girmeye çalışıyordur (tehlikeli). Sunucunun yeniden kurulduğundan **emin olduktan sonra** eski kaydı sil:

```bash
ssh-keygen -R 192.168.10.20
```

Standart dışı bir port kullanılıyorsa kayıt `[adres]:port` biçiminde tutulur ve silerken de öyle yazılmalıdır:

```bash
ssh-keygen -R "[192.168.10.20]:2222"
```
```
# Host [192.168.10.20]:2222 found: line 1
/home/ayse/.ssh/known_hosts updated.
Original contents retained as /home/ayse/.ssh/known_hosts.old
```

Bir sunucunun kaydı olup olmadığına bakmak için `ssh-keygen -F 192.168.10.20`. Ubuntu'da kayıtlar hash'lenmiş tutulduğu için dosyayı `cat` ile okuyup aramak işe yaramaz.

---

## 13. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `/etc/ssh/sshd_config` | Sunucu ayarları |
| `/etc/ssh/sshd_config.d/*.conf` | Ek sunucu ayarları, ana dosyadan önce okunur |
| `sudo sshd -t` | Sözdizimini kontrol eder |
| `sudo sshd -T` | Geçerli bütün ayarları gösterir |
| `sudo sshd -T -C user=X,host=x,addr=IP` | Bir kullanıcı için geçerli ayarları gösterir |
| `sudo systemctl reload ssh` | Ayarları uygular (RHEL'de `sshd`) |
| `systemctl daemon-reload` + `restart ssh.socket` | Ubuntu'da port/adres değişikliğinden sonra |
| `Port`, `AddressFamily`, `ListenAddress` | Dinlenecek port ve adres |
| `PermitRootLogin no` | Root girişini kapatır |
| `PasswordAuthentication no` + `KbdInteractiveAuthentication no` | Şifreyle girişi kapatır |
| `Match User X` | Kullanıcıya özel ayar (dosyanın sonuna) |
| `ssh-keygen -t ed25519` | Anahtar çifti oluşturur |
| `ssh-copy-id kullanici@sunucu` | Açık anahtarı sunucuya ekler |
| `~/.ssh/authorized_keys` | Sunucuda kabul edilen açık anahtarlar |
| `/etc/ssh/ssh_config`, `ssh_config.d/` | Bütün kullanıcılar için istemci ayarları |
| `~/.ssh/config` | Kullanıcının kendi istemci ayarları ve takma adlar |
| `ssh -G takma-ad` | İstemcinin kullanacağı ayarları gösterir |
| `~/.ssh/known_hosts` | Güvenilen sunucu anahtarları |
| `ssh-keygen -R sunucu` | Bir sunucunun kaydını siler |
| `ssh-keygen -F sunucu` | Bir sunucunun kaydını arar |
| `ssh-keygen -lf dosya.pub` | Anahtarın parmak izini gösterir |

---

## 14. Sık yapılan hatalar

- **`sshd_config` ile `ssh_config`'i karıştırmak.** Sunucu ayarı `d`'li dosyada.
- **Ana dosyayı değiştirip `sshd_config.d/`'yi unutmak.** Ek dosyalar önce okunur ve ana dosyayı ezer.
- **Aynı ayarı iki kez yazıp sonuncusunun geçerli olduğunu sanmak.** `sshd_config`'te ilk değer kazanır.
- **`sshd -t` çalıştırmadan servisi yeniden yüklemek.** Hatalı ayar SSH'ı kapatabilir.
- **`Match` bloğunu dosyanın ortasına yazmak.** Arkasındaki bütün ayarlar sadece o kullanıcıya uygulanır.
- **Şifreyi kapatmadan önce anahtarı test etmemek.** Sunucuya erişim kesilir.
- **`KbdInteractiveAuthentication`'ı unutmak.** Şifre sorulmaya devam edebilir.
- **`authorized_keys`'e `>` ile yazmak.** Diğer anahtarlar silinir.
- **İzinleri açık bırakmak.** Dizin `700`, `authorized_keys` ve özel anahtar `600`.
- **"Host key değişti" uyarısını sorgulamadan geçmek.** Önce sunucunun gerçekten yeniden kurulduğundan emin ol.
- **Ubuntu'da port değiştirip sadece `reload` yapmak.** `ssh.socket`'in de yeniden başlatılması gerekir.

---

## 15. Alıştırmalar

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

Önceki konu: [Ağ İzleme ve Sorun Giderme](04-ag-izleme.md) · Sonraki konu: [UFW ile Paket Filtreleme](06-ufw.md)
