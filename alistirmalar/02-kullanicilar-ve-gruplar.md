# 02 - Kullanıcılar ve Gruplar: Alıştırmalar

Kullanıcılar ve Gruplar bölümündeki 7 konunun bütün alıştırmaları ve çözümleri, toplam **35 soru**. Sorular konu sırasıyla diziliyor.

Çözümler kapalı gelir. Önce kendi sanal makinende dene, sonra **Çözüm**'e tıklayıp karşılaştır. Takıldığın bir konuyu tekrar okumak için başlığın altındaki bağlantıyı kullan.

| # | Konu | Soru |
|---|---|---|
| 01 | [Kullanıcı Hesapları](../02-kullanicilar-ve-gruplar/01-kullanici-hesaplari.md) | 5 |
| 02 | [Gruplar](../02-kullanicilar-ve-gruplar/02-gruplar.md) | 5 |
| 03 | [sudo ve su](../02-kullanicilar-ve-gruplar/03-sudo-ve-su.md) | 5 |
| 04 | [Ortam Profilleri](../02-kullanicilar-ve-gruplar/04-ortam-profilleri.md) | 5 |
| 05 | [Kaynak Limitleri](../02-kullanicilar-ve-gruplar/05-kaynak-limitleri.md) | 5 |
| 06 | [ACL ve Dosya Öznitelikleri](../02-kullanicilar-ve-gruplar/06-acl.md) | 5 |
| 07 | [LDAP ile Merkezi Kullanıcı Hesapları](../02-kullanicilar-ve-gruplar/07-ldap.md) | 5 |

---

## 01 - Kullanıcı Hesapları

Konu notu: [Kullanıcı Hesapları](../02-kullanicilar-ve-gruplar/01-kullanici-hesaplari.md)

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

## 02 - Gruplar

Konu notu: [Gruplar](../02-kullanicilar-ve-gruplar/02-gruplar.md)

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** GID'si 3000 olan `tasarim` adında bir grup oluştur. `elif` ve `can` kullanıcılarını oluşturup ikisini de bu gruba ek üye olarak ekle. Biri için `gpasswd`, diğeri için `usermod` kullan.

<details>
<summary>Çözüm</summary>

```bash
sudo groupadd -g 3000 tasarim
sudo useradd -m -s /bin/bash elif
sudo useradd -m -s /bin/bash can
sudo gpasswd -a elif tasarim
sudo usermod -aG tasarim can
getent group tasarim
```
</details>

**2.** `can` kullanıcısını `tasarim` grubunda tutarak `stajyer` grubuna da ekle (grup yoksa oluştur). Sonra `stajyer` grubundan çıkar ve `tasarim` grubunda kaldığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo groupadd stajyer
sudo usermod -aG stajyer can
id can
sudo gpasswd -d can stajyer
id can
```
</details>

**3.** `/srv/tasarim` adında, sadece `tasarim` grubunun erişebildiği bir ortak dizin oluştur. Dizinde oluşturulan dosyalar otomatik olarak `tasarim` grubuna ait olsun. `elif` ile bir dosya oluşturarak doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /srv/tasarim
sudo chgrp tasarim /srv/tasarim
sudo chmod 2770 /srv/tasarim
sudo su - elif -c 'touch /srv/tasarim/logo.svg; ls -l /srv/tasarim/logo.svg'
```

Dosyanın grubu `tasarim` olmalı.
</details>

**4.** `tasarim` grubunun adını `ui` yap. `elif`'in birincil grubunu `ui` olarak değiştir.

<details>
<summary>Çözüm</summary>

```bash
sudo groupmod -n ui tasarim
sudo usermod -g ui elif
id elif
```
</details>

**5.** `ui` grubunu silmeyi dene. Hata alırsan sebebini düzelt ve sil. Sonra alıştırmada oluşturduğun her şeyi temizle.

<details>
<summary>Çözüm</summary>

```bash
sudo groupdel ui
# groupdel: cannot remove the primary group of user 'elif'
sudo usermod -g elif elif
sudo groupdel ui
sudo userdel -r elif
sudo userdel -r can
sudo groupdel stajyer
sudo rm -rf /srv/tasarim
```
</details>

---

## 03 - sudo ve su

Konu notu: [sudo ve su](../02-kullanicilar-ve-gruplar/03-sudo-ve-su.md)

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

## 04 - Ortam Profilleri

Konu notu: [Ortam Profilleri](../02-kullanicilar-ve-gruplar/04-ortam-profilleri.md)

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** Bütün kullanıcılar için `SIRKET=acme` değişkenini tanımla. Değişkenin `elif` kullanıcısının yeni bir oturumunda göründüğünü doğrula (kullanıcı yoksa oluştur).

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash elif
echo 'SIRKET=acme' | sudo tee -a /etc/environment
sudo su - elif -c 'echo $SIRKET'
```
</details>

**2.** `/opt/araclar/bin` dizini bütün kullanıcıların PATH'inin **sonuna** eklensin. Dizine `merhaba` adında küçük bir script koyup `elif` ile tam yol yazmadan çalıştır.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /opt/araclar/bin
printf '#!/bin/bash\necho "Merhaba $(whoami)"\n' | sudo tee /opt/araclar/bin/merhaba
sudo chmod 755 /opt/araclar/bin/merhaba
echo 'export PATH="$PATH:/opt/araclar/bin"' | sudo tee /etc/profile.d/araclar.sh
sudo su - elif -c 'merhaba'
```

`/etc/environment` burada uygun değil, çünkü mevcut `$PATH` değerini kullanmamız gerekiyor.
</details>

**3.** Bundan sonra açılacak her hesabın ev dizininde `KURALLAR.txt` dosyası olsun. `can` kullanıcısını oluşturup dosyanın geldiğini, `elif`'te ise olmadığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo "Sunucuda yogun islem yapmadan once ekibe haber ver." | sudo tee /etc/skel/KURALLAR.txt
sudo useradd -m -s /bin/bash can
ls /home/can/KURALLAR.txt /home/elif/KURALLAR.txt
```
</details>

**4.** Sadece `elif` için `EDITOR=nano` değişkenini tanımla. Değişken, `elif` olarak çalıştırılan komutlarda da (`su - elif -c ...`) görünsün.

<details>
<summary>Çözüm</summary>

```bash
sudo su - elif
echo 'export EDITOR=nano' >> ~/.profile
exit
sudo su - elif -c 'echo $EDITOR'
```

`~/.bashrc`'nin sonuna yazsaydık `su - elif -c` ile çalıştırılan komutta görünmezdi.
</details>

**5.** Alıştırmada yaptığın her şeyi geri al.

<details>
<summary>Çözüm</summary>

```bash
sudo sed -i '/^SIRKET=acme$/d' /etc/environment
sudo rm /etc/profile.d/araclar.sh /etc/skel/KURALLAR.txt
sudo rm -rf /opt/araclar
sudo userdel -r elif
sudo userdel -r can
```
</details>

---

## 05 - Kaynak Limitleri

Konu notu: [Kaynak Limitleri](../02-kullanicilar-ve-gruplar/05-kaynak-limitleri.md)

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** `elif` kullanıcısını oluştur. Oturumu en fazla 30 süreçle açılsın, gerekirse kendisi 40'a kadar çıkarabilsin. Kuralı `/etc/security/limits.d/` altında ayrı bir dosyaya yaz ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash elif
sudo tee /etc/security/limits.d/elif.conf > /dev/null <<'EOF'
elif   soft   nproc   30
elif   hard   nproc   40
EOF
sudo su - elif -c 'echo "soft=$(ulimit -Su) hard=$(ulimit -Hu)"'
```
</details>

**2.** `tasarim` grubundaki herkes 5 MB'tan büyük dosya oluşturamasın. `elif`'i gruba ekleyip 6 MB'lık bir dosya oluşturmayı dene.

<details>
<summary>Çözüm</summary>

```bash
sudo groupadd tasarim
sudo usermod -aG tasarim elif
echo '@tasarim   -   fsize   5120' | sudo tee /etc/security/limits.d/tasarim.conf
sudo su - elif -c 'dd if=/dev/zero of=/tmp/deneme bs=1K count=6144; ls -l /tmp/deneme'
```

`dd` "File size limit exceeded" ile durur ve dosya 5 MB'ta kalır.
</details>

**3.** `elif`'in oturumunda açık dosya limitini görüntüle. Soft limiti, hard limiti değiştirmeden 512'ye düşür.

<details>
<summary>Çözüm</summary>

```bash
sudo su - elif
ulimit -Sn
ulimit -Hn
ulimit -S -n 512
ulimit -Sn
exit
```
</details>

**4.** Aşağıdaki satırların her birindeki sorunu bul.

```
*        hard   nproc   100     # root'u da 100 süreçle sınırlamak istiyorum
@tasarim hard   cpu     30      # 30 saniye işlemci süresi
```

Ayrıca kural `/etc/security/limits.d/ozel` adlı dosyaya yazıldı.

<details>
<summary>Çözüm</summary>

1. `*` root'a uygulanmaz. Root için ayrı bir `root hard nproc 100` satırı gerekir.
2. `cpu` dakika cinsindendir. `30` yazmak 30 dakika demektir. 30 saniyelik bir sınır `limits.conf` ile verilemez, en küçük birim 1 dakikadır.
3. Dosya adı `.conf` ile bitmediği için okunmaz. `ozel.conf` olmalı.
</details>

**5.** Alıştırmada yaptığın her şeyi geri al.

<details>
<summary>Çözüm</summary>

```bash
sudo rm /etc/security/limits.d/elif.conf /etc/security/limits.d/tasarim.conf
sudo rm -f /tmp/deneme
sudo userdel -r elif
sudo groupdel tasarim
```
</details>

---

## 06 - ACL ve Dosya Öznitelikleri

Konu notu: [ACL ve Dosya Öznitelikleri](../02-kullanicilar-ve-gruplar/06-acl.md)

Kendi sanal makinende dene, sonra çözümlere bak.

**Hazırlık:**

```bash
sudo groupadd ekip
sudo useradd -m -s /bin/bash elif
sudo useradd -m -s /bin/bash kerem
sudo usermod -aG ekip elif
sudo mkdir -p /srv/paylasim
sudo chgrp ekip /srv/paylasim
sudo chmod 2770 /srv/paylasim
sudo su - elif -c 'echo "butce" > /srv/paylasim/butce.txt'
```

**1.** `kerem` gruba eklenmeden `/srv/paylasim/butce.txt` dosyasını okuyabilsin ama değiştiremesin. Doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo setfacl -m u:kerem:rx /srv/paylasim
sudo setfacl -m u:kerem:r /srv/paylasim/butce.txt
sudo su - kerem -c 'cat /srv/paylasim/butce.txt'
sudo su - kerem -c 'echo x >> /srv/paylasim/butce.txt'   # Permission denied
```
</details>

**2.** `/srv/paylasim` altında bundan sonra oluşturulacak bütün dosyaları da `kerem` okuyabilsin. `elif` ile yeni bir dosya oluşturup doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo setfacl -d -m u:kerem:rX /srv/paylasim
sudo su - elif -c 'echo "yeni" > /srv/paylasim/plan.txt'
sudo su - kerem -c 'cat /srv/paylasim/plan.txt'
getfacl -c /srv/paylasim/plan.txt
```
</details>

**3.** `kerem`'e `plan.txt` üzerinde okuma ve yazma izni ver ve yazabildiğini doğrula. Sonra `chmod g=r` çalıştır. Kerem hala yazabiliyor mu? `getfacl` ile sebebini gör ve durumu düzelt.

<details>
<summary>Çözüm</summary>

```bash
sudo setfacl -m u:kerem:rw /srv/paylasim/plan.txt
sudo su - kerem -c 'echo ek >> /srv/paylasim/plan.txt && echo yazdi'

sudo chmod g=r /srv/paylasim/plan.txt
getfacl -c /srv/paylasim/plan.txt     # user:kerem:rw- #effective:r--
sudo su - kerem -c 'echo ek >> /srv/paylasim/plan.txt'   # Permission denied

sudo chmod g=rw /srv/paylasim/plan.txt
sudo su - kerem -c 'echo ek >> /srv/paylasim/plan.txt && echo yazdi'
```

`chmod g=r` mask'i `r--` yaptı ve Kerem'in `rw-` girişi etkili olarak `r--`'ye indi.
</details>

**4.** `/srv/paylasim/islem.log` dosyası oluştur. Dosyaya ekleme yapılabilsin ama root dahil kimse silemesin veya üzerine yazamasın. Doğrula, sonra özniteliği kaldır.

<details>
<summary>Çözüm</summary>

```bash
echo "ilk kayit" | sudo tee /srv/paylasim/islem.log
sudo chattr +a /srv/paylasim/islem.log
lsattr /srv/paylasim/islem.log
echo "ikinci kayit" | sudo tee -a /srv/paylasim/islem.log   # çalışır
sudo rm /srv/paylasim/islem.log                             # Operation not permitted
sudo chattr -a /srv/paylasim/islem.log
```
</details>

**5.** `/srv/paylasim` ağacının ACL'lerini yedekle, hepsini sil ve yedekten geri yükle. Sonra alıştırmada oluşturduğun her şeyi temizle.

<details>
<summary>Çözüm</summary>

```bash
cd /srv
sudo getfacl -R paylasim | sudo tee /root/paylasim-acl.txt > /dev/null
sudo setfacl -R -b paylasim
getfacl -c paylasim/butce.txt
sudo setfacl --restore=/root/paylasim-acl.txt
getfacl -c paylasim/butce.txt

sudo rm -rf /srv/paylasim /root/paylasim-acl.txt
sudo userdel -r elif
sudo userdel -r kerem
sudo groupdel ekip
```
</details>

---

## 07 - LDAP ile Merkezi Kullanıcı Hesapları

Konu notu: [LDAP ile Merkezi Kullanıcı Hesapları](../02-kullanicilar-ve-gruplar/07-ldap.md)

Bu alıştırmalar konu notunun 2. bölümündeki deneme sunucusunun ve 4–7. bölümlerindeki istemci ayarlarının yapılmış olduğunu varsayar.

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

Önceki bölüm: [Temel Komutlar](01-temel-komutlar.md) · Sonraki bölüm: [İşletim ve Dağıtım](03-isletim-ve-dagitim.md) · [Alıştırmalar ana sayfası](README.md)
