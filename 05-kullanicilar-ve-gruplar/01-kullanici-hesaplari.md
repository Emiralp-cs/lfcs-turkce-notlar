# Kullanıcı Hesapları

Bu bölümde yerel kullanıcı hesaplarını oluşturmayı, değiştirmeyi ve silmeyi ele alıyoruz. Örnekler boyunca küçük bir yazılım ekibinin sunucusunu yönettiğimizi düşünelim:

- **ayse**: ekipteki geliştirici
- **mert**: altı aylık stajyer, hesabı staj bitince kendiliğinden kapanmalı
- **backupsvc**: gece yedekleme betiğini çalıştıracak servis hesabı

> [!NOTE]
> Komutlar Ubuntu 24.04 üzerinde denenmiştir. Çıktılardaki UID/GID numaraları ve tarihler senin sisteminde farklı olabilir.

## İçindekiler

1. [Hesap bilgileri nerede tutulur?](#1-hesap-bilgileri-nerede-tutulur)
2. [useradd'in varsayılanları](#2-useraddin-varsayılanları)
3. [Kullanıcı oluşturma](#3-kullanıcı-oluşturma)
4. [Hesabı kontrol etme](#4-hesabı-kontrol-etme)
5. [Hesabı değiştirme: usermod](#5-hesabı-değiştirme-usermod)
6. [Şifre süreleri: chage](#6-şifre-süreleri-chage)
7. [Hesabı silme: userdel](#7-hesabı-silme-userdel)
8. [Komut özeti](#8-komut-özeti)
9. [Sık yapılan hatalar](#9-sık-yapılan-hatalar)
10. [Alıştırmalar](#10-alıştırmalar)

---

## 1. Hesap bilgileri nerede tutulur?

Yerel hesaplar üç düz metin dosyasında tutulur:

| Dosya | İçerik | Kim okuyabilir? |
|---|---|---|
| `/etc/passwd` | Hesabın temel bilgileri | Herkes |
| `/etc/shadow` | Şifre hash'i ve şifre süreleri | Sadece root |
| `/etc/group` | Gruplar ve üyeleri | Herkes |

### /etc/passwd

Her satır bir hesaptır ve alanlar `:` ile ayrılır:

```
ayse:x:1001:1001:Ayse Demir:/home/ayse:/bin/bash
```

| Sıra | Örnek | Anlamı |
|---|---|---|
| 1 | `ayse` | Kullanıcı adı |
| 2 | `x` | Şifrenin `/etc/shadow`'da olduğunu gösterir |
| 3 | `1001` | UID (kullanıcı numarası) |
| 4 | `1001` | Birincil grubun GID'si |
| 5 | `Ayse Demir` | Açıklama alanı (GECOS), genelde tam ad |
| 6 | `/home/ayse` | Ev dizini |
| 7 | `/bin/bash` | Giriş yapınca açılan shell |

Tek bir hesaba bakmak için dosyanın tamamını okumak yerine `getent` kullan:

```bash
getent passwd ayse
```

`getent`, sadece `/etc/passwd`'a değil, sistemin tanıdığı bütün kaynaklara (örneğin LDAP) bakar. LDAP bölümünde bunun neden önemli olduğunu göreceğiz.

> [!TIP]
> Bu dosyaları elle düzenlemek yerine `useradd`, `usermod` gibi araçları kullan. Elle düzenlemen gerçekten gerekirse dosyayı kilitleyerek açan `sudo vipw` komutunu kullan.

### /etc/shadow

```bash
sudo getent shadow ayse
```
```
ayse:$y$j9T$...:20782:0:99999:7:::
```

| Sıra | Örnek | Anlamı |
|---|---|---|
| 1 | `ayse` | Kullanıcı adı |
| 2 | `$y$j9T$...` | Şifre hash'i (`$y$` = yescrypt). Başında `!` varsa şifreyle giriş kapalı |
| 3 | `20782` | Şifrenin son değiştiği gün (1 Ocak 1970'ten beri geçen gün sayısı) |
| 4 | `0` | Şifre değiştirdikten sonra tekrar değiştirebilmek için beklenecek en az gün |
| 5 | `99999` | Şifrenin geçerli olduğu en fazla gün |
| 6 | `7` | Şifre dolmadan kaç gün önce uyarı verilecek |
| 7 | (boş) | Şifre dolduktan sonra hesabın kaç gün daha açık kalacağı |
| 8 | (boş) | Hesabın kapanacağı tarih |

3–8 arasındaki alanları [chage](#6-şifre-süreleri-chage) ile yöneteceğiz.

---

## 2. useradd'in varsayılanları

`useradd`, bayrakla belirtmediğin her şey için iki dosyadaki varsayılanları kullanır: `/etc/default/useradd` ve `/etc/login.defs`.

```bash
useradd -D
```
```
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/sh
SKEL=/etc/skel
CREATE_MAIL_SPOOL=no
```

```bash
grep -E '^(UID_MIN|UID_MAX|USERGROUPS_ENAB|HOME_MODE)' /etc/login.defs
```
```
HOME_MODE	0750
UID_MIN			 1000
UID_MAX			60000
USERGROUPS_ENAB yes
```

Bayraksız bir `useradd` ile ne olduğunu görelim:

```bash
sudo useradd deneme
getent passwd deneme
ls -ld /home/deneme
sudo getent shadow deneme
```
```
deneme:x:1002:1002::/home/deneme:/bin/sh
ls: cannot access '/home/deneme': No such file or directory
deneme:!:20782:0:99999:7:::
```

Ubuntu 24.04'te sonuç şöyle:

| Konu | Bayraksız `useradd` |
|---|---|
| Ev dizini | Kayıtta yazıyor ama **oluşturulmuyor** (`-m` gerekir) |
| Shell | `/bin/sh`, yani **bash değil** |
| Kullanıcı grubu | Aynı adla bir grup oluşturuluyor (`USERGROUPS_ENAB yes` sayesinde) |
| Şifre | Yok. Şifre alanı `!` olduğu için şifreyle giriş yapılamıyor |
| UID | 1000'den başlayarak ilk boş numara |

> [!IMPORTANT]
> Bu varsayılanlar dağıtıma göre değişir. "Kullanıcı oluştur" dendiğinde ev dizini ve shell gibi beklenen özellikleri bayraklarla açıkça belirt, varsayılana güvenme.

```bash
sudo userdel deneme   # temizlik
```

---

## 3. Kullanıcı oluşturma

### Eksiksiz bir hesap

```bash
sudo useradd -m -s /bin/bash -c "Ayse Demir" ayse
sudo passwd ayse
```

`useradd` şifre sormaz. Şifre her zaman ayrı bir adımda `passwd` ile verilir.

Sık kullanılan bayraklar:

| Bayrak | Uzun hali | Ne yapar? |
|---|---|---|
| `-m` | `--create-home` | Ev dizinini oluşturur ve `/etc/skel` içeriğini kopyalar |
| `-s` | `--shell` | Giriş shell'ini belirler |
| `-c` | `--comment` | Açıklama alanını (tam ad) doldurur |
| `-u` | `--uid` | UID'yi elle belirler |
| `-d` | `--home-dir` | Ev dizini için farklı bir yol verir |
| `-U` | `--user-group` | Aynı adla bir grup oluşturur. Ubuntu'da zaten varsayılan, yazmanın zararı yok |
| `-g` | `--gid` | Birincil grubu belirler |
| `-G` | `--groups` | Ek gruplara ekler ([Gruplar](02-gruplar.md) bölümünde) |
| `-e` | `--expiredate` | Hesabın kapanacağı tarihi belirler |

Ev dizinine `/etc/skel`'deki dosyalar kopyalanır:

```bash
sudo ls -la /home/ayse
```
```
drwxr-x--- 2 ayse ayse 4096 Nov 25 12:31 .
drwxr-xr-x 5 root root 4096 Nov 25 12:31 ..
-rw-r--r-- 1 ayse ayse  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 ayse ayse 3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 ayse ayse  807 Mar 31  2024 .profile
```

Dizin izni `drwxr-x---` (0750) geliyor, bu da `login.defs`'teki `HOME_MODE` değerinden. `/etc/skel` konusunu [Ortam Profilleri](04-ortam-profilleri.md) bölümünde ayrıntılı ele alacağız.

### Belirli bir UID ve bitiş tarihi

Stajyer Mert'in hesabı 30 Haziran 2027'de kendiliğinden kapansın. UID'si de 1500 olsun, çünkü dosyaları NFS ile paylaşılan başka bir sunucuda da aynı UID'ye sahip:

```bash
sudo useradd -m -s /bin/bash -u 1500 -c "Mert Kaya (stajyer)" -e 2027-06-30 mert
getent passwd mert
sudo chage -l mert | grep 'Account expires'
```
```
mert:x:1500:1500:Mert Kaya (stajyer):/home/mert:/bin/bash
Account expires						: Jun 30, 2027
```

> [!TIP]
> NFS gibi ağ dosya sistemleri dosya sahipliğini isimle değil UID ile eşleştirir. Bu yüzden aynı kişinin farklı sunuculardaki UID'si aynı olmalıdır.

### Sistem hesabı

Servisleri çalıştıran hesaplarla kimse giriş yapmaz. Bunlar için `--system` kullanılır:

```bash
sudo useradd --system -s /usr/sbin/nologin backupsvc
getent passwd backupsvc
```
```
backupsvc:x:996:995::/home/backupsvc:/usr/sbin/nologin
```

- UID, 1000'in altındaki sistem aralığından verilir.
- Ev dizini kayıtta yazar ama oluşturulmaz.
- `/usr/sbin/nologin` shell'i, bu hesapla etkileşimli giriş yapılmasını engeller.

### Peki adduser?

Debian ve Ubuntu'da bir de `adduser` vardır. `useradd`'in üzerine kurulmuş, soru sorarak ilerleyen bir araçtır:

| | `useradd` | `adduser` (Debian/Ubuntu) |
|---|---|---|
| Çalışma şekli | Sadece bayraklarla, soru sormaz | Şifre ve ad gibi bilgileri sorar |
| Ev dizini | `-m` ile | Kendiliğinden |
| Varsayılan shell | `/bin/sh` | `/bin/bash` |
| Bulunduğu yer | Bütün dağıtımlar | Debian/Ubuntu |

RHEL tabanlı dağıtımlarda `adduser`, `useradd`'e bağlanan bir kısayoldur ve aynı şekilde davranır. Script yazarken ve her dağıtımda aynı sonucu almak istediğinde `useradd` kullan.

---

## 4. Hesabı kontrol etme

```bash
whoami              # şu an hangi kullanıcıyım?
id                  # benim UID, GID ve gruplarım
id ayse             # başka bir kullanıcının bilgileri
getent passwd ayse  # hesabın passwd kaydı
sudo passwd -S ayse # şifre durumu
```

`passwd -S` çıktısı:

```
ayse P 2026-11-25 0 99999 7 -1
```

İkinci alan şifre durumunu gösterir: `P` şifre var, `L` kilitli, `NP` şifre yok.

---

## 5. Hesabı değiştirme: usermod

### Ev dizinini taşıma

Ekip dosyaları `/srv/ekip` altında tutulmaya başlandı ve Ayşe'nin ev dizini oraya taşınacak:

```bash
sudo mkdir -p /srv/ekip
sudo usermod -d /srv/ekip/ayse -m ayse
```

- `-d` kayıttaki yolu değiştirir.
- `-m` dosyaları yeni yere taşır. `-m` olmadan sadece kayıt değişir, dosyalar eski yerde kalır ve kullanıcı var olmayan bir dizine giriş yapmaya çalışır.

> [!WARNING]
> Hedefin üst dizini (burada `/srv/ekip`) yoksa taşıma başarısız olur, ama **kayıt yine de değişir**. Denemede `usermod: cannot rename directory` hatasından sonra kaydın yeni yolu gösterdiği, dosyaların ise eski yerde kaldığı görüldü. Hata alırsan `getent passwd kullanici` ile kaydı kontrol et ve gerekirse eski yola geri al.

### Kullanıcı adını değiştirme

```bash
sudo usermod -l aysedemir ayse
id aysedemir
```
```
uid=1001(aysedemir) gid=1001(ayse) groups=1001(ayse)
```

Dikkat: `-l` sadece kullanıcı adını değiştirir. Grubun adı hala `ayse` ve ev dizini hala `/home/ayse`. Hepsini tutarlı hale getirmek için:

```bash
sudo groupmod -n aysedemir ayse
sudo usermod -d /home/aysedemir -m aysedemir
id aysedemir
```
```
uid=1001(aysedemir) gid=1001(aysedemir) groups=1001(aysedemir)
```

### Shell'i değiştirme

```bash
sudo usermod -s /usr/sbin/nologin ayse   # girişi kapat
sudo chsh -s /bin/bash ayse              # aynı işi yapan diğer komut
```

### Hesabı kilitleme

```bash
sudo usermod -L ayse   # kilitle (passwd -l ile aynı)
sudo usermod -U ayse   # kilidi aç (passwd -u ile aynı)
```

Kilitleme, `/etc/shadow`'daki hash'in başına `!` koyar ve böylece şifreyle giriş kapanır.

> [!WARNING]
> `-L` sadece şifreyle girişi kapatır. Kullanıcının SSH anahtarı varsa hala giriş yapabilir. Hesabı tamamen kapatmak için kilitlemeyle birlikte bitiş tarihini de geçmişe al:
> ```bash
> sudo usermod -L -e 1 ayse
> ```

Şifresi hiç olmayan bir hesabın kilidini açmaya çalışırsan `usermod` buna izin vermez, çünkü sonuç şifresiz bir hesap olurdu. Önce `passwd` ile şifre ver.

### Hesap bitiş tarihi

```bash
sudo usermod -e 2027-06-30 mert   # bu tarihte kapanır
sudo usermod -e 2020-01-01 mert   # geçmiş tarih: hesap hemen kapanır
sudo usermod -e "" mert           # bitiş tarihini kaldır
sudo usermod -e -1 mert           # bu da kaldırır
```

---

## 6. Şifre süreleri: chage

İki farklı "süre dolması" var ve sık karıştırılıyor:

| | Hesabın süresi dolarsa | Şifrenin süresi dolarsa |
|---|---|---|
| Ayarlandığı yer | `usermod -e` veya `chage -E` | `chage -M`, `chage -d` |
| Sonuç | Giriş yapılamaz | Giriş yapılır, ama önce yeni şifre istenir |
| Ne zaman kullanılır? | Stajyerin hesabı staj bitince kapansın | Şifreler 90 günde bir yenilensin |

Mevcut ayarları görmek için:

```bash
sudo chage -l mert
```
```
Last password change					: Nov 25, 2026
Password expires					: never
Password inactive					: never
Account expires						: Jun 30, 2027
Minimum number of days between password change		: 0
Maximum number of days between password change		: 99999
Number of days of warning before password expires	: 7
```

Sık kullanılan ayarlar:

```bash
sudo chage -d 0 mert            # bir sonraki girişte şifre değiştirmek zorunda
sudo chage -M 90 -m 1 -W 7 mert # 90 günde bir değişsin, en az 1 gün beklensin, 7 gün önce uyar
sudo chage -M -1 mert           # şifre hiç dolmasın
sudo chage -E 2027-06-30 mert   # hesap bitiş tarihi (usermod -e ile aynı)
sudo chage -E -1 mert           # bitiş tarihini kaldır
sudo chage mert                 # bütün değerleri tek tek sorarak ayarla
```

`chage -d 0` sonrası çıktı şöyle görünür:

```
Last password change					: password must be changed
Password expires					: password must be changed
```

`sudo passwd -e mert` komutu da aynı işi yapar.

Hangi bayrak `/etc/shadow`'daki hangi alanı değiştiriyor:

| `chage` bayrağı | `/etc/shadow` alanı |
|---|---|
| `-d` | 3: son değişim günü |
| `-m` | 4: en az bekleme |
| `-M` | 5: en fazla geçerlilik |
| `-W` | 6: uyarı günü |
| `-I` | 7: dolduktan sonra açık kalma süresi |
| `-E` | 8: hesap bitiş tarihi |

> [!CAUTION]
> `-M -1` "hiç dolmasın" demektir. `-d 0` ise "şifre şimdiden dolmuş" demektir. Sayıları karıştırma.

---

## 7. Hesabı silme: userdel

```bash
sudo userdel mert      # hesabı siler, ev dizini kalır
sudo userdel -r mert   # hesabı, ev dizinini ve mail kutusunu siler
```

- Kullanıcının kendi adıyla oluşturulmuş grubu da silinir (başka üyesi yoksa).
- `-r` ile silerken `mail spool ... not found` uyarısı çıkabilir. Mail kutusu hiç oluşturulmadığı içindir, sorun değil.

Sadece `userdel` ile silersen ev dizini sahipsiz kalır:

```bash
ls -ld /home/mert
```
```
drwxr-x--- 2 1500 1500 4096 Nov 25 12:31 /home/mert
```

Sahip artık isim olarak değil, numara olarak görünüyor.

> [!WARNING]
> İleride UID'si 1500 olan yeni bir kullanıcı açarsan bu dosyaların sahibi o olur. Sahipsiz dosyaları bulmak için:
> ```bash
> sudo find / -nouser -o -nogroup 2>/dev/null
> ```

Aynı isimle tekrar hesap açarsan ve eski ev dizini hala duruyorsa, `useradd -m` uyarı verir ve `/etc/skel` dosyalarını kopyalamaz:

```
useradd: warning: the home directory /home/mert already exists.
useradd: Not copying any file from skel directory into it.
```

---

## 8. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `useradd -D` | useradd varsayılanlarını gösterir |
| `useradd -m -s /bin/bash -c "Ad" kullanici` | Ev dizini ve bash shell'iyle hesap açar |
| `useradd -u 1500 kullanici` | Belirli UID ile açar |
| `useradd -e 2027-06-30 kullanici` | Bitiş tarihli hesap açar |
| `useradd --system -s /usr/sbin/nologin servis` | Servis hesabı açar |
| `passwd kullanici` | Şifre verir veya değiştirir |
| `passwd -S kullanici` | Şifre durumunu gösterir |
| `getent passwd kullanici` | Hesap kaydını gösterir |
| `id kullanici` | UID, GID ve grupları gösterir |
| `usermod -d /yeni/yol -m kullanici` | Ev dizinini taşır |
| `usermod -l yeniad eskiad` | Kullanıcı adını değiştirir |
| `usermod -s /bin/bash kullanici` | Shell'i değiştirir |
| `usermod -L` / `-U kullanici` | Şifreyle girişi kilitler / açar |
| `usermod -e TARIH kullanici` | Hesap bitiş tarihi koyar |
| `chage -l kullanici` | Şifre ve hesap sürelerini gösterir |
| `chage -d 0 kullanici` | Sonraki girişte şifre değişimini zorunlu yapar |
| `chage -M 90 kullanici` | Şifreyi 90 günde bir yeniletir |
| `chage -M -1 kullanici` | Şifre süresini kaldırır |
| `userdel kullanici` | Hesabı siler, ev dizini kalır |
| `userdel -r kullanici` | Hesabı ve ev dizinini siler |

---

## 9. Sık yapılan hatalar

- **`-m`'yi unutmak.** Hesap oluşur ama ev dizini oluşmaz.
- **Shell'i belirtmemek.** Ubuntu'da varsayılan `/bin/sh` gelir. Kullanıcı bash bekliyorsa `-s /bin/bash` yaz.
- **Şifre vermeyi unutmak.** `useradd` sonrası `passwd` ayrı bir adımdır. Şifresiz hesapla şifreyle giriş yapılamaz.
- **`usermod -d`'yi `-m` olmadan kullanmak.** Kayıt değişir, dosyalar taşınmaz.
- **`usermod -l` sonrası grubu ve ev dizinini unutmak.** Sadece kullanıcı adı değişir.
- **Kilitlemeyi yeterli sanmak.** `-L` SSH anahtarıyla girişi engellemez, gerekirse `-e 1` de ekle.
- **Hesap süresiyle şifre süresini karıştırmak.** `usermod -e` girişi kapatır, `chage -M` sadece şifre yenilemeyi zorunlu kılar.
- **`userdel`'i `-r` olmadan kullanıp dizini unutmak.** Geride sahipsiz dosyalar kalır.

---

## 10. Alıştırmalar

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** `elif` adında, ev dizini olan, shell'i `/bin/bash`, açıklaması "Elif Yildiz" olan bir kullanıcı oluştur ve şifresini belirle. İlk girişte şifresini değiştirmek zorunda kalsın.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash -c "Elif Yildiz" elif
sudo passwd elif
sudo chage -d 0 elif
sudo chage -l elif | head -2
```
</details>

**2.** UID'si 2000 olan ve 31 Aralık 2027'de kapanacak `misafir` adında bir hesap aç. Sonra bitiş tarihini kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -u 2000 -e 2027-12-31 misafir
sudo chage -l misafir | grep 'Account expires'
sudo usermod -e "" misafir
```
</details>

**3.** `monitor` adında, giriş yapılamayan bir sistem hesabı oluştur. UID'sinin 1000'in altında olduğunu doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd --system -s /usr/sbin/nologin monitor
id monitor
```
</details>

**4.** `elif` kullanıcısının adını `elify` yap. Grubunun ve ev dizininin adı da değişsin.

<details>
<summary>Çözüm</summary>

```bash
sudo usermod -l elify elif
sudo groupmod -n elify elif
sudo usermod -d /home/elify -m elify
id elify
getent passwd elify
```
</details>

**5.** `elify` hesabını SSH anahtarıyla bile giriş yapılamayacak şekilde kapat. Sonra bütün alıştırma hesaplarını ev dizinleriyle birlikte sil.

<details>
<summary>Çözüm</summary>

```bash
sudo usermod -L -e 1 elify
sudo chage -l elify | grep 'Account expires'
sudo userdel -r elify
sudo userdel -r misafir
sudo userdel monitor
```
</details>

---

Sonraki konu: [Gruplar](02-gruplar.md)
