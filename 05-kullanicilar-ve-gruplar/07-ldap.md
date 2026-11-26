# LDAP ile Merkezi Kullanıcı Hesapları

> **Resmi sınav maddesi:** Configure the system to use LDAP user and group accounts

Şimdiye kadar oluşturduğumuz kullanıcılar sadece kendi sunucularında, `/etc/passwd` dosyasında var. Ekip büyüyüp sunucu sayısı onlara çıkınca her kişiyi her sunucuda ayrı ayrı açmak, şifre değişikliklerini ve işten ayrılanları takip etmek zorlaşır. Ansible gibi araçlar bu işi otomatikleştirse de hesaplar yine her sunucuda ayrı ayrı durur.

**LDAP** (Lightweight Directory Access Protocol) bu sorunu merkezi bir dizinle çözer. Kullanıcılar ve gruplar tek bir LDAP sunucusunda tutulur, bütün Linux sunucuları hesapları oradan okur. Bir kişiyi eklemek, şifresini değiştirmek ya da silmek tek bir yerden yapılır.

Bu bölümün asıl konusu **istemci tarafı**, yani bir Linux sunucusunu LDAP'taki hesapları kullanacak şekilde ayarlamak. Gerçek ortamlarda LDAP sunucusu genelde ayrı bir makinede hazır durur. Burada kendi başına pratik yapabilmen için önce aynı makineye küçük bir deneme sunucusu kuruyoruz.

Senaryo: Ekibe uzaktan çalışan iki kişi katılıyor, **elif** ve **kerem**. Hesapları LDAP'ta tutulacak, hiçbir sunucuda yerel hesapları olmayacak.

> [!NOTE]
> Bütün adımlar Ubuntu 24.04 üzerinde denenmiştir. Deneme sunucusu `ekip.local` alan adını, yani `dc=ekip,dc=local` tabanını kullanıyor.

## İçindekiler

1. [Temel kavramlar](#1-temel-kavramlar)
2. [Deneme için LDAP sunucusu kurmak](#2-deneme-için-ldap-sunucusu-kurmak)
3. [Ön kontrol: kullanıcılar henüz yok](#3-ön-kontrol-kullanıcılar-henüz-yok)
4. [LDAP istemcisini kurmak: libnss-ldapd](#4-ldap-istemcisini-kurmak-libnss-ldapd)
5. [Yapılandırma dosyalarını kontrol etmek](#5-yapılandırma-dosyalarını-kontrol-etmek)
6. [LDAP kullanıcılarını doğrulamak](#6-ldap-kullanıcılarını-doğrulamak)
7. [LDAP şifresiyle giriş ve otomatik ev dizini](#7-ldap-şifresiyle-giriş-ve-otomatik-ev-dizini)
8. [Sorun giderme](#8-sorun-giderme)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Temel kavramlar

Bir program "elif kimdir?" diye sorduğunda cevap birkaç katmandan geçer:

```
id elif
   |
NSS  ->  /etc/nsswitch.conf: "önce files'a, sonra ldap'a bak"
   |
files: /etc/passwd   (bulamadı)
   |
ldap: nslcd servisi  ->  LDAP sunucusu  (buldu)
```

| Kavram | Ne işe yarar? |
|---|---|
| **NSS** (Name Service Switch) | Kullanıcı ve grup bilgisinin **hangi kaynaklardan, hangi sırayla** aranacağını belirler. Ayarı `/etc/nsswitch.conf`'ta |
| **nslcd** | NSS ile LDAP sunucusu arasındaki köprü servisi. Bağlantı ayarları `/etc/nslcd.conf`'ta |
| **PAM** | Giriş ve şifre kontrolü. LDAP şifresiyle girişi ve ev dizininin otomatik oluşmasını sağlar |
| **DN** (Distinguished Name) | LDAP'taki her kaydın tam adı, örneğin `uid=elif,ou=people,dc=ekip,dc=local` |
| **Base** | Aramaların başladığı en üst nokta, örneğin `dc=ekip,dc=local` |

LDAP kullanıcıya **bilgi** sağlar: UID, grup, ev dizini yolu, shell, şifre. Ev dizinini oluşturmak gibi dosya sistemi işlerini yapmaz. Onu [7. bölümde](#7-ldap-şifresiyle-giriş-ve-otomatik-ev-dizini) PAM'e yaptıracağız.

---

## 2. Deneme için LDAP sunucusu kurmak

> [!TIP]
> Bu bölüm sadece pratik yapabilmen için. Elinde hazır bir LDAP sunucusu varsa doğrudan [3. bölüme](#3-ön-kontrol-kullanıcılar-henüz-yok) geç ve oradaki adres ile base değerlerini kullan.

### Sunucuyu kurmak

```bash
sudo apt update
sudo apt install slapd ldap-utils
sudo dpkg-reconfigure slapd
```

`dpkg-reconfigure slapd` şunları sorar:

| Soru | Cevap |
|---|---|
| Omit OpenLDAP server configuration? | No |
| DNS domain name | `ekip.local` |
| Organization name | `Ekip` |
| Administrator password | Bir şifre belirle, `ldapadd` gibi komutlarda bu şifre sorulacak |
| Do you want the database to be removed when slapd is purged? | No |
| Move old database? | Yes |

Kurulumu doğrula:

```bash
ldapsearch -x -LLL -H ldap://localhost -b dc=ekip,dc=local dn
```
```
dn: dc=ekip,dc=local
```

### Kullanıcıları ve grubu eklemek

LDAP'a kayıt eklemek için LDIF denen metin formatı kullanılır. Önce kullanıcılar için bir şifre hash'i üretelim:

```bash
slappasswd
```

Şifreyi iki kez sorar ve `{SSHA}...` ile başlayan bir hash verir. Bu hash'i aşağıda `SIFRE_HASHI` yazan yerlere koy.

`ekip.ldif` dosyası oluştur:

```ldif
dn: ou=people,dc=ekip,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=ekip,dc=local
objectClass: organizationalUnit
ou: groups

dn: cn=ldapekip,ou=groups,dc=ekip,dc=local
objectClass: posixGroup
cn: ldapekip
gidNumber: 10000
memberUid: elif
memberUid: kerem

dn: uid=elif,ou=people,dc=ekip,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: elif
cn: Elif Yildiz
sn: Yildiz
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/elif
loginShell: /bin/bash
userPassword: SIFRE_HASHI

dn: uid=kerem,ou=people,dc=ekip,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: kerem
cn: Kerem Aydin
sn: Aydin
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/kerem
loginShell: /bin/bash
userPassword: SIFRE_HASHI
```

- `posixAccount` ve `posixGroup`, kaydın Linux hesabı olarak kullanılabilmesi için gereken alanları (UID, GID, ev dizini, shell) tanımlar.
- UID ve GID'leri yerel kullanıcılarla çakışmaması için 10000'den başlattık.

Dosyayı yönetici hesabıyla ekle (`-W` şifreyi sorar):

```bash
ldapadd -x -H ldap://localhost -D cn=admin,dc=ekip,dc=local -W -f ekip.ldif
```
```
adding new entry "ou=people,dc=ekip,dc=local"
adding new entry "ou=groups,dc=ekip,dc=local"
adding new entry "cn=ldapekip,ou=groups,dc=ekip,dc=local"
adding new entry "uid=elif,ou=people,dc=ekip,dc=local"
adding new entry "uid=kerem,ou=people,dc=ekip,dc=local"
```

> [!WARNING]
> Deneme sunucusu `ldap://` ile şifrelenmemiş bağlantı kullanıyor. Gerçek ortamlarda LDAP trafiği `ldaps://` veya StartTLS ile şifrelenir.

---

## 3. Ön kontrol: kullanıcılar henüz yok

İstemciyi ayarlamadan önce kullanıcıların sistemde tanınmadığını gör. Ayardan sonra "birdenbire" ortaya çıkmaları, kurulumun çalıştığının kanıtı olacak.

```bash
id elif
id kerem
```
```
id: 'elif': no such user
id: 'kerem': no such user
```

---

## 4. LDAP istemcisini kurmak: libnss-ldapd

```bash
sudo apt install libnss-ldapd
```

Bu paket kurulurken `nslcd` (köprü servisi), `libpam-ldapd` (LDAP şifresiyle giriş) ve `nscd` (önbellek servisi) de birlikte kurulur. Kurulum sırasında bir sihirbaz açılır:

| Soru | Cevap |
|---|---|
| LDAP server URI | `ldap://127.0.0.1/` (sunucu başka makinedeyse onun adresi: `ldap://10.0.0.5/`) |
| LDAP server search base | `dc=ekip,dc=local` |
| Name services to configure | `passwd`, `group`, `shadow` (boşluk tuşuyla işaretle) |

Adresin sonundaki `/` için denedik: `ldap://127.0.0.1` ve `ldap://127.0.0.1/` ikisi de çalıştı.

Sihirbazı daha sonra tekrar çalıştırmak istersen:

```bash
sudo dpkg-reconfigure nslcd          # sunucu adresi ve base
sudo dpkg-reconfigure libnss-ldapd   # nsswitch'e eklenecek servisler
```

---

## 5. Yapılandırma dosyalarını kontrol etmek

### /etc/nsswitch.conf

```bash
grep -E '^(passwd|group|shadow)' /etc/nsswitch.conf
```
```
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files systemd ldap
```

Kullanıcı bilgisi soldan sağa aranır: önce yerel dosyalar (`files`, yani `/etc/passwd`), sonra systemd, en son LDAP. Sistemine göre `systemd` kelimesi olmayabilir, önemli olan `ldap`'ın listede olması.

`files` en başta olduğu için yerel bir kullanıcı ile LDAP kullanıcısının adı aynıysa yerel olan kazanır.

### /etc/nslcd.conf

```bash
sudo grep -vE '^\s*(#|$)' /etc/nslcd.conf
```
```
uid nslcd
gid nslcd
uri ldap://127.0.0.1/
base dc=ekip,dc=local
tls_cacertfile /etc/ssl/certs/ca-certificates.crt
```

`uri` ve `base` satırları sihirbazdaki cevaplarından geldi. Bu dosyayı elle değiştirirsen servisi yeniden başlat:

```bash
sudo systemctl restart nslcd
```

---

## 6. LDAP kullanıcılarını doğrulamak

```bash
id elif
id kerem
```
```
uid=10001(elif) gid=10000(ldapekip) groups=10000(ldapekip)
uid=10002(kerem) gid=10000(ldapekip) groups=10000(ldapekip)
```

Kullanıcılar artık tanınıyor. Kaynaklarına göre listelemek için:

```bash
getent passwd                 # yerel ve LDAP, hepsi
getent passwd -s ldap         # sadece LDAP'tan gelenler (uzun hali: --service ldap)
getent group -s ldap          # sadece LDAP grupları
getent passwd -s files        # sadece /etc/passwd
```
```
elif:x:10001:10000:Elif Yildiz:/home/elif:/bin/bash
kerem:x:10002:10000:Kerem Aydin:/home/kerem:/bin/bash

ldapekip:*:10000:kerem,elif
```

> [!IMPORTANT]
> LDAP kullanıcıları `/etc/passwd`'da **yoktur**. Denemede `grep elif /etc/passwd` hiçbir şey bulmadı. Bu yüzden kullanıcı kontrol ederken `cat /etc/passwd` yerine her zaman `getent passwd` veya `id` kullan. Bu iki komut NSS'ten geçer ve bütün kaynaklara bakar.

Ev dizinleri ise henüz yok:

```bash
ls /home
```

LDAP sadece "Elif'in ev dizini `/home/elif`" bilgisini veriyor, dizini oluşturmuyor.

---

## 7. LDAP şifresiyle giriş ve otomatik ev dizini

`libpam-ldapd` kurulduğu için LDAP kullanıcıları LDAP'taki şifreleriyle giriş yapabilir. Ev dizini olmadan giriş yapmayı denedik:

```bash
su - elif
```
```
Password:
su: warning: cannot change directory to /home/elif: No such file or directory
```

Giriş başarılı oldu ama ev dizini olmadığı için Elif bir önceki dizinde kaldı. Bu durumda `.bashrc` gibi dosyalar okunmaz ve kullanıcı dosya kaydedecek bir yer bulamaz. Yanlış şifreyle ise `su: Authentication failure` aldık.

Yüzlerce LDAP kullanıcısının dizinini her sunucuda elle açmak yerine, PAM'e "kullanıcının dizini yoksa ilk girişte oluştur" dedirtiriz:

```bash
sudo pam-auth-update
```

Açılan listede **Create home directory on login** seçeneğini boşluk tuşuyla işaretle ve onayla. Aynı işi sorusuz yapmak için:

```bash
sudo pam-auth-update --enable mkhomedir
```

Ayar `/etc/pam.d/common-session` dosyasına bir satır ekler:

```bash
grep mkhomedir /etc/pam.d/common-session
```
```
session	optional			pam_mkhomedir.so
```

Tekrar giriş yapalım:

```bash
su - elif
pwd
ls -la
```
```
/home/elif
drwxr-x--- 2 elif ldapekip 4096 Nov 25 13:15 .
-rw-r--r-- 1 elif ldapekip  220 Nov 25 13:15 .bash_logout
-rw-r--r-- 1 elif ldapekip 3771 Nov 25 13:15 .bashrc
-rw-r--r-- 1 elif ldapekip  807 Nov 25 13:15 .profile
```

Ev dizini ilk girişte oluştu, içine `/etc/skel` dosyaları kopyalandı ve sahibi Elif oldu. Root olarak yapılan `su - kerem` gibi şifresiz geçişlerde de dizin oluştu.

---

## 8. Sorun giderme

### nslcd çalışmıyorsa kullanıcılar kaybolur

Denemede `nslcd` servisini durdurduk:

```
id elif               ->  id: 'elif': no such user
getent passwd -s ldap ->  (boş)
```

Servis tekrar başlayınca kullanıcılar geri geldi. LDAP kullanıcıları görünmüyorsa ilk bakılacak yer:

```bash
systemctl status nslcd
```

Sunucuya erişilebildiğini doğrudan test etmek için:

```bash
ldapsearch -x -LLL -H ldap://127.0.0.1/ -b dc=ekip,dc=local '(uid=elif)'
```

`ldapsearch` sonuç veriyor ama `id elif` vermiyorsa sorun istemci tarafındadır: `nslcd.conf`, `nsswitch.conf` veya nslcd servisi.

### nscd önbelleği eski bilgi gösterebilir

`nscd`, kullanıcı bilgilerini bir süre önbellekte tutar. Denemede Kerem'in shell'ini LDAP'ta `/bin/sh` yaptık:

```
getent passwd kerem   ->  ... :/bin/bash     (önbellekteki eski bilgi)
sudo nscd -i passwd
getent passwd kerem   ->  ... :/bin/sh       (güncel)
```

LDAP'ta yaptığın değişiklik istemcide görünmüyorsa önbelleği temizle:

```bash
sudo nscd -i passwd
sudo nscd -i group
```

### LDAP'tan silinen kullanıcının dizini kalır

LDAP'tan bir kullanıcıyı sildiğimizde ev dizini sunucuda kaldı ve sahibi numarayla görünmeye başladı:

```
drwxr-x--- 2 10003 ldapekip 4096 Nov 25 13:16 /home/selin
```

Bu, `userdel`'i `-r` olmadan kullanmakla aynı durum ([Kullanıcı Hesapları](01-kullanici-hesaplari.md#7-hesabı-silme-userdel) bölümüne bak). Dizinleri ayrıca temizlemen gerekir.

---

## 9. Komut özeti

| Komut / Dosya | Ne işe yarar? |
|---|---|
| `apt install slapd ldap-utils` | Deneme sunucusunu ve LDAP araçlarını kurar |
| `dpkg-reconfigure slapd` | Sunucunun alan adını ve yönetici şifresini ayarlar |
| `slappasswd` | LDIF için şifre hash'i üretir |
| `ldapadd -x -D cn=admin,... -W -f dosya.ldif` | LDAP'a kayıt ekler |
| `ldapmodify -x -D cn=admin,... -W -f dosya.ldif` | Kaydı değiştirir |
| `ldapdelete -x -D cn=admin,... -W DN` | Kaydı siler |
| `ldappasswd -x -D cn=admin,... -W -S DN` | Kullanıcının LDAP şifresini değiştirir |
| `ldapsearch -x -LLL -H URI -b BASE 'filtre'` | LDAP'ta arama yapar |
| `apt install libnss-ldapd` | İstemciyi kurar (nslcd ve libpam-ldapd dahil) |
| `dpkg-reconfigure nslcd` | İstemcinin sunucu adresini ve base'ini değiştirir |
| `/etc/nsswitch.conf` | Kullanıcı bilgisinin aranacağı kaynaklar ve sıra |
| `/etc/nslcd.conf` | LDAP sunucusunun adresi ve base'i |
| `systemctl restart nslcd` | nslcd ayarlarını yeniden yükler |
| `id kullanici` | Kullanıcıyı bütün kaynaklarda arar |
| `getent passwd -s ldap` | Sadece LDAP kullanıcılarını listeler |
| `getent group -s ldap` | Sadece LDAP gruplarını listeler |
| `pam-auth-update --enable mkhomedir` | İlk girişte ev dizini oluşturmayı açar |
| `nscd -i passwd` | Kullanıcı önbelleğini temizler |

---

## 10. Sık yapılan hatalar

- **LDAP kullanıcısını `/etc/passwd`'da aramak.** Orada yoktur. `getent passwd` veya `id` kullan.
- **Ev dizininin kendiliğinden oluşmasını beklemek.** `pam-auth-update` ile **Create home directory on login** açılmalı.
- **`nsswitch.conf`'ta `ldap` olup olmadığını kontrol etmemek.** Sihirbazda servisler işaretlenmezse LDAP'a hiç bakılmaz.
- **`nslcd.conf`'u değiştirip servisi yeniden başlatmamak.** Yeni ayar devreye girmez.
- **LDAP'taki değişikliğin hemen görünmesini beklemek.** `nscd` önbelleği eski bilgiyi gösterebilir: `nscd -i passwd`.
- **Sorunun sunucuda mı istemcide mi olduğunu ayırmamak.** Önce `ldapsearch` ile sunucuyu, sonra `id` ile istemciyi test et.
- **LDAP ve yerel kullanıcılara aynı UID'leri vermek.** Dosya sahiplikleri karışır. LDAP UID'leri için ayrı bir aralık kullan.

---

## 11. Alıştırmalar

Bu alıştırmalar 2. bölümdeki deneme sunucusunun ve 4–7. bölümlerdeki istemci ayarlarının yapılmış olduğunu varsayar.

**1.** LDAP'a `selin` adında, UID'si 10003, birincil grubu `ldapekip` olan bir kullanıcı ekle. Şifresini `ldappasswd` ile belirle. İstemcide `id` ile göründüğünü doğrula.

<details>
<summary>Çözüm</summary>

`selin.ldif`:

```ldif
dn: uid=selin,ou=people,dc=ekip,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: selin
cn: Selin Kara
sn: Kara
uidNumber: 10003
gidNumber: 10000
homeDirectory: /home/selin
loginShell: /bin/bash
```

```bash
ldapadd -x -H ldap://localhost -D cn=admin,dc=ekip,dc=local -W -f selin.ldif
ldappasswd -x -H ldap://localhost -D cn=admin,dc=ekip,dc=local -W -S uid=selin,ou=people,dc=ekip,dc=local
id selin
getent passwd -s ldap selin
```

`ldappasswd`'deki `-S` yeni şifreyi sorar, `-W` ise yönetici şifresini.
</details>

**2.** `selin`'in `/etc/passwd`'da olmadığını ama `getent passwd` çıktısında olduğunu göster. Sebebini açıkla.

<details>
<summary>Çözüm</summary>

```bash
grep selin /etc/passwd     # çıktı yok
getent passwd selin        # selin:x:10003:10000:...
```

`grep` sadece dosyaya bakar. `getent` ise NSS'ten geçer ve `nsswitch.conf`'taki bütün kaynakları (`files` ve `ldap`) sorar.
</details>

**3.** `selin` olarak giriş yap ve ev dizininin otomatik oluştuğunu doğrula.

<details>
<summary>Çözüm</summary>

```bash
su - selin      # LDAP şifresi
pwd             # /home/selin
ls -la
exit
```
</details>

**4.** `selin`'i LDAP'taki `ldapekip` grubuna ek üye olarak ekle ve istemcide doğrula. Değişiklik görünmüyorsa sebebini bul.

<details>
<summary>Çözüm</summary>

`grup.ldif`:

```ldif
dn: cn=ldapekip,ou=groups,dc=ekip,dc=local
changetype: modify
add: memberUid
memberUid: selin
```

```bash
ldapmodify -x -H ldap://localhost -D cn=admin,dc=ekip,dc=local -W -f grup.ldif
getent group -s ldap ldapekip
```

Eski liste görünüyorsa `nscd` önbelleği yüzündendir: `sudo nscd -i group` ile temizleyip tekrar bak.
</details>

**5.** `nslcd` servisini durdur ve `id selin` sonucunu gör. Servisi tekrar başlat. Sonra `selin`'i LDAP'tan sil ve ev dizinini temizle.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl stop nslcd
id selin                    # no such user
sudo systemctl start nslcd
id selin

ldapdelete -x -H ldap://localhost -D cn=admin,dc=ekip,dc=local -W uid=selin,ou=people,dc=ekip,dc=local
sudo nscd -i passwd
id selin                    # no such user
sudo rm -rf /home/selin
```
</details>

---

Önceki konu: [ACL ve Dosya Öznitelikleri](06-acl.md) · Bölüm ana sayfası: [Kullanıcılar ve Gruplar](README.md)
