# sudo ve su

> **İlgili resmi sınav maddesi:** Create and manage local user and group accounts

Sistemde root yetkisi gerektiren işler için iki yol var:

- **sudo**: Kendi şifrenle, izin verilen komutları root olarak çalıştırırsın.
- **su**: Başka bir kullanıcının (genelde root'un) şifresiyle onun hesabına geçersin.

Senaryomuz devam ediyor:

- **ayse** sunucunun yöneticisi olacak, her komutu sudo ile çalıştırabilecek.
- **mert** stajyer. Sadece web sunucusunu yeniden başlatabilmeli, başka hiçbir şeyi değil.
- **root** hesabının şifresi kilitli kalacak, kimse doğrudan root olarak giriş yapmayacak.

> [!NOTE]
> Komutlar Ubuntu 24.04 ve sudo 1.9.15 üzerinde denenmiştir.

## İçindekiler

1. [sudo ile tek bir komut çalıştırma](#1-sudo-ile-tek-bir-komut-çalıştırma)
2. [Root shell açma: sudo -i ve sudo -s](#2-root-shell-açma-sudo--i-ve-sudo--s)
3. [su ile kullanıcı değiştirme](#3-su-ile-kullanıcı-değiştirme)
4. [sudo ve su farkı](#4-sudo-ve-su-farkı)
5. [Kilitli root hesabı](#5-kilitli-root-hesabı)
6. [Kimin sudo yetkisi var?](#6-kimin-sudo-yetkisi-var)
7. [sudoers kuralı yazma](#7-sudoers-kuralı-yazma)
8. [sudoers'ın temel kuralları](#8-sudoersın-temel-kuralları)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. sudo ile tek bir komut çalıştırma

`/root` dizinini normal bir kullanıcı okuyamaz:

```bash
ls /root
```
```
ls: cannot open directory '/root': Permission denied
```

Başına `sudo` koyunca komut root olarak çalışır:

```bash
sudo ls /root
```

- sudo **senin kendi şifreni** sorar, root'un şifresini değil.
- Şifreyi bir kez girince 15 dakika boyunca tekrar sormaz. Bu süre `sudo -V` çıktısında "Authentication timestamp timeout" satırında yazar.
- Süreyi beklemeden şifreyi unutturmak için `sudo -k` yazılır.

Root yerine başka bir kullanıcı olarak çalıştırmak için `-u` kullanılır:

```bash
sudo -u mert whoami
```
```
mert
```

---

## 2. Root shell açma: sudo -i ve sudo -s

Arka arkaya çok sayıda root komutu çalıştıracaksan her seferinde `sudo` yazmak yerine bir root shell açabilirsin:

```bash
sudo -i    # veya: sudo --login
```

İşin bitince `exit` ya da `logout` ile çıkarsın.

`sudo -s` de root shell açar ama farklı davranır. Ayşe `/tmp` dizinindeyken ikisini de denedik:

| | `sudo -i` | `sudo -s` |
|---|---|---|
| Shell türü | Giriş shell'i (root'un giriş dosyalarını okur) | Normal shell |
| Açılınca bulunduğun dizin | `/root` | Olduğun yerde kalır (`/tmp`) |
| `whoami` | `root` | `root` |

Root'un ortamının eksiksiz yüklenmesini istiyorsan `sudo -i` kullan.

> [!TIP]
> Root shell'de geçirilen süreyi kısa tut. Uzun süre açık kalan bir root shell'de yanlış bir komut yazmak çok kolay. Tek tek `sudo` ile çalışmak daha güvenli ve her komut `/var/log/auth.log`'a ayrı ayrı kaydedilir.

---

## 3. su ile kullanıcı değiştirme

```bash
su -           # root'a geç (su -l ve su --login ile aynı)
su - mert      # mert'e geç
```

- `su`, geçtiğin hesabın şifresini sorar. `su -` için root'un şifresini bilmen gerekir.
- Root olarak çalışırken `su - mert` şifre sormaz.

### Tire neden önemli?

Root olarak `/tmp` dizinindeyken ve kendi tanımladığımız `DENEME=gizli` değişkeni varken iki şekilde mert'e geçtik:

| | `su mert` | `su - mert` |
|---|---|---|
| Bulunduğun dizin | `/tmp` (değişmez) | `/home/mert` |
| `$HOME` | `/home/mert` | `/home/mert` |
| `$DENEME` | `gizli` (önceki oturumdan taşındı) | boş |
| Giriş dosyaları (`.profile` vb.) | Okunmaz | Okunur |

Tiresiz `su`, `HOME` ve `USER` gibi birkaç temel değişkeni değiştirir. Ama önceki kullanıcının diğer değişkenleri yeni oturuma taşınır ve hedef kullanıcının giriş dosyaları çalışmaz. Bu da beklenmedik davranışlara yol açabilir. Her zaman `su -` kullan.

---

## 4. sudo ve su farkı

| | `sudo` | `su -` |
|---|---|---|
| Hangi şifre? | Kendi şifren | Geçtiğin hesabın şifresi |
| Gereken şey | sudoers'ta yetkin olması | O hesabın şifresini bilmen |
| Yetki kapsamı | Kuralla sınırlanabilir (sadece belirli komutlar) | Hesabın tüm yetkisi |
| Kayıt | Her komut ayrı ayrı loglanır | Sadece geçiş loglanır |

Birden fazla yönetici olan sistemlerde `sudo` tercih edilir. Herkese root şifresini dağıtmak gerekmez, birinin yetkisini kaldırmak için root şifresini değiştirmek gerekmez ve kimin hangi komutu çalıştırdığı kayıt altında kalır.

---

## 5. Kilitli root hesabı

Ubuntu'da root hesabının şifresi varsayılan olarak kilitlidir:

```bash
sudo passwd -S root
```
```
root L 2026-10-24 0 99999 7 -1
```

`L` kilitli demektir. Kilitli root ile şunu denedik:

| Deneme | Sonuç |
|---|---|
| `su -` | `su: Authentication failure` |
| `sudo -i` (sudo yetkisi olan Ayşe ile) | `uid=0(root)`, çalışıyor |

Yani root'un kilitli olması sadece **şifreyle root'a girişi** kapatır. Root hesabı vardır ve sudo yetkisi olanlar root olarak çalışmaya devam eder.

Root şifresiyle ilgili komutlar:

```bash
sudo passwd root      # root'a şifre ver (şifresi hiç yoksa böyle açılır)
sudo passwd -l root   # şifreyle girişi kilitle (hash'in başına ! koyar)
sudo passwd -u root   # kilidi aç (daha önce şifre verilmişse)
```

Root'un hiç şifresi yoksa `passwd -u root` çalışmaz, çünkü sonuç şifresiz bir root hesabı olurdu. Önce `passwd root` ile şifre verilir.

> [!CAUTION]
> Root'u kilitlemeden önce kendi sudo yetkinin çalıştığından emin ol:
> ```bash
> sudo -l     # neleri çalıştırabildiğini gösterir
> groups      # sudo (veya wheel) grubunda mısın?
> ```
> Hem root kilitliyse hem de sudo yetkin bozuksa sisteme yönetici olarak girmenin kolay bir yolu kalmaz. Kurtarma için sistemi kurtarma modunda başlatmak gerekir.

---

## 6. Kimin sudo yetkisi var?

Yetkiler `/etc/sudoers` dosyasında ve `/etc/sudoers.d/` dizinindeki dosyalarda tanımlıdır. Ubuntu'daki varsayılan dosyada şu satır var:

```
%sudo	ALL=(ALL:ALL) ALL
```

Başındaki `%` bunun bir grup olduğunu gösterir: `sudo` grubundaki herkes her komutu çalıştırabilir. RHEL ailesinde aynı iş `wheel` grubuyla yapılır.

Ayşe'yi yönetici yapmak için onu gruba eklemek yeterli:

```bash
sudo usermod -aG sudo ayse
```

Bir kullanıcının neleri çalıştırabildiğini görmek için:

```bash
sudo -l             # kendi yetkilerim
sudo -l -U ayse     # başka bir kullanıcının yetkileri
```
```
User ayse may run the following commands on vm:
    (ALL : ALL) ALL
```

Ayşe gruba eklendikten sonra yetkinin geçerli olması için yeniden giriş yapmalıdır ([Gruplar](02-gruplar.md#5-üyelik-ne-zaman-geçerli-olur) bölümüne bak).

---

## 7. sudoers kuralı yazma

### Kuralın yapısı

```
mert  ALL=(root)  NOPASSWD: /usr/bin/systemctl restart nginx
```

| Parça | Örnekte | Anlamı |
|---|---|---|
| Kim? | `mert` | Kullanıcı adı. Grup için `%grupadi` |
| Hangi sunucuda? | `ALL` | Kural hangi makinede geçerli (tek sunucuda hep `ALL`) |
| Kim olarak? | `(root)` | Komut hangi kullanıcı olarak çalışacak. `(ALL)` herkes olarak demek |
| Seçenek | `NOPASSWD:` | Şifre sorma (isteğe bağlı) |
| Komut | `/usr/bin/systemctl restart nginx` | İzin verilen komut. Virgülle birden fazla yazılabilir |

### Dosyayı nerede ve nasıl düzenleriz?

Kuralları ana dosyaya değil, `/etc/sudoers.d/` altında ayrı bir dosyaya yaz ve dosyayı **her zaman `visudo` ile** aç:

```bash
sudo visudo -f /etc/sudoers.d/mert
```

`visudo`, kaydettiğinde sözdizimini kontrol eder. Hatalı bir satır varsa kaydetmeden önce uyarır:

```
/tmp/s5:1:36: syntax error
mert ALL=(ALL) NOPASWD: /usr/bin/ls
                                   ^
```

Burada hata `NOPASSWD` kelimesinin yanlış yazılması. Dosyayı başka bir editörle açıp böyle bir hatayla kaydetseydin, sudo bütün kuralları okuyamaz hale gelir ve kimse sudo kullanamazdı.

Bütün dosyaları kontrol etmek için:

```bash
sudo visudo -c
```
```
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/mert: parsed OK
```

> [!WARNING]
> `/etc/sudoers.d/` altındaki dosyaların adında **nokta olmamalı**. Denemede kuralı `mert.conf` adlı dosyaya koyduğumuzda sudo dosyayı tamamen görmezden geldi. Dosya adını `mert` yapınca kural çalıştı. Adı `~` ile biten dosyalar da okunmaz.

### Mert'in kuralı

```bash
sudo visudo -f /etc/sudoers.d/mert
```

Dosyaya yazılacak satır:

```
mert ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
```

Kontrol:

```bash
sudo -l -U mert
```
```
User mert may run the following commands on vm:
    (root) NOPASSWD: /usr/bin/systemctl restart nginx
```

Bir komuta izin olup olmadığını, komutu çalıştırmadan da test edebilirsin. Mert'in oturumunda:

```bash
sudo -l /usr/bin/systemctl restart nginx   # izin var: komutun tam yolu yazılır
sudo -l /usr/bin/systemctl stop nginx      # izin yok: hiçbir şey yazmaz, hata koduyla döner
```

### Grup için kural

Mert'i tek başına yetkilendirmek yerine bir `operasyon` grubu açıp yetkiyi gruba verebiliriz. Bu kez şifre sorulsun:

```
%operasyon ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

---

## 8. sudoers'ın temel kuralları

### Kural 1: Komutlar tam yoluyla yazılır

```
mert ALL=(root) NOPASSWD: systemctl restart nginx
```
```
expected a fully-qualified path name
```

Komutun tam yolunu bulmak için `which` kullan:

```bash
which systemctl
```
```
/usr/bin/systemctl
```

Güvenlik sebebi şu: Tam yol yazılmasaydı, kullanıcı kendi dizinine `systemctl` adında başka bir program koyup onu root olarak çalıştırabilirdi.

Ubuntu'da `/bin`, `/usr/bin`'e bir bağlantıdır. Bu yüzden `/bin/bash` ile `/usr/bin/bash` aynı dosyadır ve sudo ikisini de aynı komut olarak kabul eder. Yine de `which`'in gösterdiği yolu yazmak en temizidir.

### Kural 2: Kuralın içine sudo yazılmaz

sudoers zaten "sudo ile neler çalıştırılabilir" listesidir. Kuralın içine tekrar `sudo` yazmak yanlış:

```
mert ALL=(ALL) NOPASSWD: sudo /bin/bash
```
```
expected a fully-qualified path name
```

`NOPASSWD:` kelimesinden hemen sonra doğrudan çalıştırılacak programın tam yolu gelmelidir.

### Kural 3: Komut terminalde yine sudo ile çalıştırılır

Kural eklemek komutu kendiliğinden root yetkisiyle çalışır hale getirmez. Kullanıcının yetkiyi kullanmak için komutun başına `sudo` yazması gerekir:

```bash
ls /root          # Permission denied
sudo ls /root     # kurala göre çalışır
```

### Kural 4: Argüman yazarsan sadece o argümana izin verilir

Kuralda komutun arkasına argüman yazılırsa, sadece o argümanlarla çalıştırmaya izin verilir:

```
mert ALL=(root) NOPASSWD: /usr/bin/ls /root
```
```
sudo ls /root           # izin var
sudo ls /root/.bashrc   # izin yok: "a password is required"
```

Argüman yazılmazsa (`/usr/bin/ls`), komut her argümanla çalıştırılabilir. Yetkiyi daraltmak istediğinde argümanları da yaz.

> [!CAUTION]
> Bazı programlara verilen sudo yetkisi pratikte tam root yetkisi demektir:
> - `/bin/bash` veya başka bir shell: doğrudan root shell açar.
> - `vim`, `less`, `nano` gibi editörler ve görüntüleyiciler: içinden komut çalıştırılabilir (örneğin vim'de `:!bash`).
> - `tar`, `find`, `awk` gibi araçlar: parametreleriyle başka komut çalıştırabilirler.
>
> `NOPASSWD: /bin/bash` gibi bir kural, o kullanıcıya şifresiz root vermekle aynıdır. Kısıtlı yetki vermek istiyorsan belirli komutlara ve argümanlara izin ver.

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sudo komut` | Komutu root olarak çalıştırır (kendi şifrenle) |
| `sudo -u kullanici komut` | Komutu başka bir kullanıcı olarak çalıştırır |
| `sudo -i` | Root için giriş shell'i açar |
| `sudo -s` | Root shell açar, olduğun dizinde kalır |
| `sudo -k` | sudo'nun şifreyi hatırlamasını sonlandırır |
| `sudo -l` | Kendi sudo yetkilerini gösterir |
| `sudo -l -U kullanici` | Başka bir kullanıcının yetkilerini gösterir |
| `su -` | Root'un şifresiyle root'a geçer |
| `su - kullanici` | Başka bir kullanıcıya geçer |
| `exit` / `logout` | Açılan shell'den çıkar |
| `passwd -S root` | Root'un şifre durumunu gösterir |
| `passwd root` | Root'a şifre verir |
| `passwd -l root` / `passwd -u root` | Root'un şifreyle girişini kilitler / açar |
| `visudo` | `/etc/sudoers`'ı kontrollü açar |
| `visudo -f /etc/sudoers.d/dosya` | Ayrı bir kural dosyasını kontrollü açar |
| `visudo -c` | Bütün sudoers dosyalarını kontrol eder |
| `usermod -aG sudo kullanici` | Ubuntu'da kullanıcıyı yönetici yapar (RHEL'de `wheel`) |

---

## 10. Sık yapılan hatalar

- **sudoers'ı `visudo` olmadan düzenlemek.** Tek bir sözdizimi hatası sudo'yu herkes için bozar.
- **Komutu tam yolsuz yazmak.** `systemctl` değil `/usr/bin/systemctl`.
- **Kuralın içine `sudo` yazmak.** `NOPASSWD: sudo ...` hatalıdır.
- **`/etc/sudoers.d/` altındaki dosyaya noktalı ad vermek.** `mert.conf` gibi bir dosya hiç okunmaz.
- **Tiresiz `su` kullanmak.** Önceki kullanıcının ortamı yeni oturuma taşınır.
- **Root'u kilitlemeden önce sudo'yu kontrol etmemek.** Sistemin dışında kalabilirsin.
- **`passwd -l` ile `-u`'yu karıştırmak.** `-l` lock (kilitle), `-u` unlock (aç).
- **Shell veya editöre sudo yetkisi vermek.** Bu, tam root yetkisi vermek demektir.

---

## 11. Alıştırmalar

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** `elif` adında bir kullanıcı oluştur ve onu yönetici yap. Yetkisini doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash elif
sudo passwd elif
sudo usermod -aG sudo elif     # RHEL ailesinde: wheel
sudo -l -U elif
```
</details>

**2.** `can` adında bir kullanıcı oluştur. `can`, şifre girmeden sadece `/usr/bin/apt update` komutunu çalıştırabilsin, `apt install` çalıştıramasın. Kuralı `/etc/sudoers.d/` altında ayrı bir dosyaya yaz.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash can
which apt
sudo visudo -f /etc/sudoers.d/can
```

Dosyaya:

```
can ALL=(root) NOPASSWD: /usr/bin/apt update
```

Kontrol:

```bash
sudo visudo -c
sudo -l -U can
```
</details>

**3.** Aşağıdaki kuralda üç hata var. Bul ve düzelt.

```
elif ALL=(root) NOPASWD: sudo systemctl status ssh
```

<details>
<summary>Çözüm</summary>

1. `NOPASWD` yanlış yazılmış, doğrusu `NOPASSWD`.
2. Kuralın içine `sudo` yazılmaz.
3. `systemctl` tam yoluyla yazılmalı.

```
elif ALL=(root) NOPASSWD: /usr/bin/systemctl status ssh
```
</details>

**4.** Root'un şifre durumunu kontrol et. Root'a şifre ver, `su -` ile giriş yapmayı dene, sonra root'u tekrar kilitle ve `su -`'nun artık çalışmadığını ama `sudo -i`'nin çalıştığını gör.

<details>
<summary>Çözüm</summary>

```bash
sudo passwd -S root
sudo passwd root
su -              # root'un yeni şifresi ile girer
exit
sudo passwd -l root
su -              # su: Authentication failure
sudo -i           # çalışır
exit
```
</details>

**5.** Alıştırmada oluşturduğun her şeyi temizle.

<details>
<summary>Çözüm</summary>

```bash
sudo rm /etc/sudoers.d/can
sudo userdel -r can
sudo userdel -r elif
sudo visudo -c
```
</details>

---

Önceki konu: [Gruplar](02-gruplar.md) · Sonraki konu: [Ortam Profilleri](04-ortam-profilleri.md)
