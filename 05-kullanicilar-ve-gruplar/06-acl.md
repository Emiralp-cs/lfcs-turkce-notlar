# ACL ve Dosya Öznitelikleri

Standart Linux izinleri üç kategoriyle çalışır: sahip, grup ve diğerleri. Çoğu zaman bu yeter, ama bazen bir dosyaya **tek bir kişiye daha** özel izin vermek gerekir. Kişiyi gruba eklemek ona grubun bütün dosyalarına erişim verir, sahipliği değiştirmek de başkalarının işini bozar. ACL (Access Control List) bu durumda devreye girer.

Bu bölümün ikinci yarısında da dosya özniteliklerini (`chattr`) ele alıyoruz. Bunlar izinlerden bağımsız olarak bir dosyanın değiştirilmesini, root için bile, engelleyebilir.

Senaryomuz devam ediyor. `/srv/proje` dizini `gelistirici` grubuna ait ve izinleri `2770` ([Gruplar](02-gruplar.md#10-pratik-ortak-proje-dizini) bölümünde kurmuştuk):

- **deniz** dışarıdan gelen bir denetçi. Projeyi okuyabilmeli ama hiçbir şeyi değiştirememeli. Gruba eklenmemeli.
- **mert** stajyer. Sadece `plan.txt` dosyasını düzenleyebilmeli.
- **can** `gelistirici` grubunda ama gizli bir dosyayı görmemeli.
- `denetim.log` dosyasına sadece ekleme yapılabilmeli, `ayar.conf` ise hiç değiştirilememeli.

> [!NOTE]
> Komutlar Ubuntu 24.04 ve ext4 dosya sistemi üzerinde denenmiştir. `setfacl` bulunamazsa `sudo apt install acl` ile kurabilirsin.

## İçindekiler

1. [ACL'e bakmak: getfacl](#1-acle-bakmak-getfacl)
2. [Kullanıcıya veya gruba izin vermek](#2-kullanıcıya-veya-gruba-izin-vermek)
3. [İzinler hangi sırayla kontrol edilir?](#3-izinler-hangi-sırayla-kontrol-edilir)
4. [mask: etkili izin](#4-mask-etkili-izin)
5. [ACL girişlerini kaldırmak](#5-acl-girişlerini-kaldırmak)
6. [Dizinlere toplu uygulama: -R ve X](#6-dizinlere-toplu-uygulama--r-ve-x)
7. [Yeni dosyalar için varsayılan ACL](#7-yeni-dosyalar-için-varsayılan-acl)
8. [Kopyalama, taşıma ve yedekleme](#8-kopyalama-taşıma-ve-yedekleme)
9. [Dosya öznitelikleri: chattr ve lsattr](#9-dosya-öznitelikleri-chattr-ve-lsattr)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. ACL'e bakmak: getfacl

ACL'i olmayan bir dizin:

```bash
getfacl /srv/proje
```
```
# file: srv/proje
# owner: root
# group: gelistirici
# flags: -s-
user::rwx
group::rwx
other::---
```

- İlk satırlar yorumdur: dosya adı, sahip, grup ve özel bitler (`-s-` setgid demek).
- `user::`, `group::`, `other::` satırları, `ls -l`'de gördüğün standart izinlerin aynısıdır.
- `getfacl` mutlak yol verildiğinde "Removing leading '/'" uyarısı yazar, önemli değil. Yorum satırları olmadan sadece izinleri görmek için `getfacl -c` kullanılır.

Bir dosyada ek ACL girişi varsa `ls -l` çıktısında izinlerin sonunda **`+`** işareti görünür:

```
drwxrws---+ 4 root gelistirici 4096 Nov 25 13:06 /srv/proje
```

`+` işaretini gördüğünde izinlerin tamamını öğrenmek için `getfacl`'e bak.

---

## 2. Kullanıcıya veya gruba izin vermek

`setfacl -m` (modify) giriş ekler veya değiştirir:

```bash
sudo setfacl -m u:deniz:rx /srv/proje            # kullanıcıya
sudo setfacl -m g:denetim:rx /srv/proje          # gruba (denetim grubu varsa)
```

`u:` yerine `user:`, `g:` yerine `group:` de yazılabilir.

Deniz'in dizine girebildiğini kontrol edelim:

```bash
sudo su - deniz -c 'ls /srv/proje'
```
```
docs
plan.txt
src
```

### Dizin izni de gerekir

Mert'e sadece `plan.txt` dosyasında yazma izni verelim:

```bash
sudo setfacl -m u:mert:rw /srv/proje/plan.txt
```

Bu tek başına yetmez. Mert `/srv/proje` dizinine giremiyorsa içindeki dosyaya da ulaşamaz. Bir dosyaya erişmek için yoldaki **her dizinde `x`** (geçiş) iznine ihtiyaç vardır:

```bash
sudo setfacl -m u:mert:rx /srv/proje
sudo su - mert -c 'echo "notum" >> /srv/proje/plan.txt && echo yazdi'
```
```
yazdi
```

> [!TIP]
> ACL verdiğin halde "Permission denied" alıyorsan, önce dosyanın bulunduğu dizinlerde o kişinin `x` izni olup olmadığına bak.

---

## 3. İzinler hangi sırayla kontrol edilir?

Linux bir erişimde girişleri şu sırayla kontrol eder ve **ilk eşleşen** kuralı uygular:

1. Dosyanın sahibi mi? Öyleyse `user::` uygulanır.
2. Adıyla bir `user:isim:` girişi var mı? Varsa o uygulanır.
3. Dosyanın grubunda veya `group:isim:` girişlerinden birinde mi? Öyleyse grup izinleri uygulanır.
4. Hiçbiri değilse `other::` uygulanır.

Sonuç olarak **kişiye özel giriş, grup izninden önce gelir**. Bunu bir kişiyi engellemek için kullanabiliriz. Can `gelistirici` grubunda ve normalde `plan.txt`'yi okuyabiliyor:

```bash
sudo setfacl -m u:can:--- /srv/proje/plan.txt
sudo su - can -c 'cat /srv/proje/plan.txt'
```
```
cat: /srv/proje/plan.txt: Permission denied
```

Grup üyeliği hiç değişmedi, ama Can'a özel `---` girişi grubun önüne geçti.

> [!NOTE]
> Bu engelleme dosyanın **sahibine** işlemez. Sahip için her zaman `user::` satırı uygulanır, adıyla yazılan giriş dikkate alınmaz.

---

## 4. mask: etkili izin

Kullanıcıya veya gruba ACL girişi eklediğinde `getfacl` çıktısında bir `mask::` satırı belirir:

```
user::rw-
user:mert:rw-
group::rw-
mask::rw-
other::r--
```

mask, adıyla yazılan bütün kullanıcı girişleri ve bütün grup girişleri için **üst sınırdır**. Bir girişin etkili izni, kendi izni ile mask'in kesişimidir.

### chmod mask'i değiştirir

ACL'i olan bir dosyada `chmod` ile grup izinlerini değiştirdiğinde, aslında mask'i değiştirmiş olursun:

```bash
sudo chmod g=r /srv/proje/plan.txt
getfacl -c /srv/proje/plan.txt
```
```
user::rw-
user:mert:rw-	#effective:r--
group::rw-	#effective:r--
mask::r--
other::r--
```

Mert'in girişi hala `rw-` görünüyor ama `#effective:r--` notu, gerçekte sadece okuyabildiğini söylüyor. Denemede Mert'in yazma girişimi `Permission denied` ile reddedildi.

> [!WARNING]
> ACL kullanılan dosyalarda `chmod` ile grup iznini kısmak, ACL ile verdiğin izinleri de sessizce kısar. `getfacl`'de `#effective` notlarına dikkat et.

### mask tamamen boşsa

Denemede beklenmedik bir şey gördük. Kerem'e `r` izni verilmiş bir dosyada `chmod g=` ile mask'i `---` yaptık:

```
-rw----r--+ 1 root root 6 Nov 25 13:10 f.txt

user::rw-
user:kerem:r--	#effective:---
group::r--	#effective:---
mask::---
other::r--
```

`#effective:---` yazmasına rağmen Kerem dosyayı **okuyabildi**. Sebebi şu: mask (yani `ls -l`'deki grup bitleri) tamamen boş olduğunda Linux çekirdeği ACL girişlerine hiç bakmaz ve standart izinleri uygular. Kerem de dosyayı "diğerleri" (`other::r--`) iznini kullanarak okudu. Aynı dosyada `other` izni de kapatılınca (`-rw-------+`) Kerem okuyamadı.

> [!CAUTION]
> Birini ACL ile kısıtlamaya veya engellemeye çalışırken mask'i boş bırakma ve `other` iznini de kontrol et. Mask boşken adıyla yazılan girişler, `u:isim:---` engellemesi dahil, devre dışı kalır.

### setfacl mask'i yeniden hesaplar

`setfacl -m` ile yeni bir giriş eklediğinde mask, girişlerin hepsini kapsayacak şekilde **yeniden hesaplanır**. Yukarıdaki durumda Deniz için bir giriş ekleyince mask tekrar `rw-` oldu ve Mert'in yazma izni geri geldi.

mask'i kendin belirlemek istersen:

```bash
sudo setfacl -m m::r /srv/proje/plan.txt
```

---

## 5. ACL girişlerini kaldırmak

```bash
sudo setfacl -x u:can /srv/proje/plan.txt   # tek bir girişi kaldır
sudo setfacl -b /srv/proje/plan.txt         # bütün ek girişleri kaldır
```

- `-x` için izin yazılmaz, sadece `u:isim` veya `g:isim`.
- `-b` sonrası dosya standart izinlerine döner ve `ls -l`'deki `+` kaybolur.

Uzun halleri: `-m` = `--modify`, `-x` = `--remove`, `-b` = `--remove-all`.

---

## 6. Dizinlere toplu uygulama: -R ve X

Deniz projenin tamamını okuyabilsin:

```bash
sudo setfacl -R -m u:deniz:rX /srv/proje
```

- `-R` (`--recursive`): dizindeki mevcut bütün dosya ve alt dizinlere uygular.
- Büyük `X`: `x` iznini **sadece dizinlere** (ve zaten çalıştırılabilir olan dosyalara) verir.

Sonuç:

```bash
getfacl -c /srv/proje/src | grep deniz
getfacl -c /srv/proje/src/app.py | grep deniz
```
```
user:deniz:r-x
user:deniz:r--
```

Dizine `r-x`, dosyaya `r--` verildi. Küçük `x` kullansaydık bütün dosyalar çalıştırılabilir hale gelirdi.

> [!IMPORTANT]
> `-R` sadece **o an var olan** dosyaları etkiler. Sonradan oluşturulan dosyalarda Deniz'in girişi olmaz. Denemede Ayşe `docs` altında "diğerleri" izni kapalı yeni bir dosya oluşturdu (`umask 007` ile) ve Deniz o dosyayı okuyamadı. Yeni dosyalar için bir sonraki bölümdeki varsayılan ACL gerekir.

---

## 7. Yeni dosyalar için varsayılan ACL

Bir dizine **varsayılan (default) ACL** koyarsan, o dizinde oluşturulan her yeni dosya ve dizin bu girişleri otomatik olarak alır:

```bash
sudo setfacl -d -m u:deniz:rX /srv/proje/docs
getfacl -c /srv/proje/docs
```
```
user::rwx
user:deniz:r-x
group::rwx
mask::rwx
other::---
default:user::rwx
default:user:deniz:r-x
default:group::rwx
default:mask::rwx
default:other::---
```

`default:` ile başlayan satırlar dizinin kendisine değil, içinde oluşturulacak dosyalara uygulanır. Ayşe şimdi yeni bir dosya oluşturduğunda Deniz onu okuyabiliyor:

```bash
sudo su - ayse -c 'echo "taslak" > /srv/proje/docs/rapor2.md'
sudo su - deniz -c 'cat /srv/proje/docs/rapor2.md'
```
```
taslak
```

Denemelerde gördüklerimiz:

- **Alt dizinler varsayılanı miras alır.** Yeni oluşturulan bir alt dizinin de aynı `default:` satırları oldu, yani kural aşağı doğru yayılıyor.
- **Dosyalar `x` almaz.** Yeni dosyada `user:deniz:r-x #effective:r--` göründü. Programlar dosyaları genelde çalıştırma izni olmadan oluşturur, mask de buna göre `rw-` olur.
- **Varsayılan ACL umask'in yerine geçer.** Varsayılan ACL'li dizinde oluşan dosyanın `other` izni, kullanıcının umask'inden değil, `default:other::` satırından geldi.

Mevcut dosyalar ve gelecek dosyalar için ikisini birlikte kullan:

```bash
sudo setfacl -R -m u:deniz:rX /srv/proje      # şimdiki dosyalar
sudo setfacl -R -d -m u:deniz:rX /srv/proje   # bundan sonra oluşacaklar
```

Varsayılan ACL'i kaldırmak için:

```bash
sudo setfacl -k /srv/proje/docs
```

---

## 8. Kopyalama, taşıma ve yedekleme

ACL'ler dosya kopyalanırken kaybolabilir. Denedik:

| İşlem | ACL korunur mu? |
|---|---|
| `cp dosya hedef` | Hayır |
| `cp -a dosya hedef` | Evet |
| `mv dosya hedef` | Evet |

Dosyaları başka bir yere kopyalarken ACL'lerin korunmasını istiyorsan `cp -a` (veya `cp --preserve=all`) kullan.

### ACL'leri yedeklemek ve geri yüklemek

Bir dizin ağacının bütün ACL'lerini bir metin dosyasına kaydedebilirsin:

```bash
cd /srv
sudo getfacl -R proje > /tmp/proje-acl.txt
```

Bir şey bozulursa (denemede `setfacl -R -b` ile hepsini sildik) geri yükle:

```bash
cd /srv
sudo setfacl --restore=/tmp/proje-acl.txt
```

Yedek dosyasındaki yollar göreli olduğu için, geri yüklerken yedeği aldığın dizinde (`/srv`) olmalısın.

---

## 9. Dosya öznitelikleri: chattr ve lsattr

Öznitelikler izinlerden farklı bir katmandır. Bir dosyayı `777` izinle bile değiştirilemez hale getirebilirler. İki tanesi özellikle önemli: `a` (append-only, sadece ekleme) ve `i` (immutable, değiştirilemez).

### Sadece ekleme: +a

Denetim kayıtları silinmemeli veya geriye dönük değiştirilmemeli:

```bash
echo "2026-11-25 sunucu basladi" | sudo tee /srv/proje/denetim.log
sudo chattr +a /srv/proje/denetim.log
lsattr /srv/proje/denetim.log
```
```
-----a--------e------- /srv/proje/denetim.log
```

Root olarak denedik:

| İşlem | Sonuç |
|---|---|
| `echo "kayit 2" >> denetim.log` (ekleme) | Çalışır |
| `echo "yeni" > denetim.log` (üzerine yazma) | `Operation not permitted` |
| `truncate -s0 denetim.log` (boşaltma) | `Operation not permitted` |
| `rm denetim.log` | `Operation not permitted` |
| `mv denetim.log x.log` | `Operation not permitted` |

### Değiştirilemez: +i

Kritik bir ayar dosyası yanlışlıkla değiştirilmesin:

```bash
sudo chattr +i /srv/proje/ayar.conf
lsattr /srv/proje/ayar.conf
```
```
----i---------e------- /srv/proje/ayar.conf
```

Yine root olarak denedik:

| İşlem | Sonuç |
|---|---|
| Ekleme veya üzerine yazma | `Operation not permitted` |
| `rm`, `mv` | `Operation not permitted` |
| Hard link oluşturma (`ln`) | `Operation not permitted` |
| `chmod`, `chown` | `Operation not permitted` |
| Okuma (`cat`) | Çalışır |

### Kimler değiştirebilir?

- Öznitelikleri sadece root ekleyip kaldırabilir. Denemede Ayşe kendi ev dizinindeki **kendi dosyasına** bile `+i` koyamadı.
- Kaldırmak için `-` kullanılır:

```bash
sudo chattr -a /srv/proje/denetim.log
sudo chattr -i /srv/proje/ayar.conf
```

> [!TIP]
> Root olduğun halde bir dosyayı silemiyor veya değiştiremiyorsan aklına öznitelikler gelsin: `lsattr dosya` ile `i` veya `a` var mı bak.

`lsattr` çıktısındaki `e` harfi ext4'ün dosyayı "extent" yapısıyla sakladığını gösterir. Normaldir, sen eklemedin. Diğer öznitelikler için `man chattr`'a bakabilirsin. Desteklenenler dosya sistemine göre değişir.

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `getfacl dosya` | ACL'i gösterir |
| `getfacl -c dosya` | Yorum satırları olmadan gösterir |
| `setfacl -m u:isim:rwx dosya` | Kullanıcıya izin verir |
| `setfacl -m g:grup:rx dosya` | Gruba izin verir |
| `setfacl -m u:isim:--- dosya` | Kullanıcıyı engeller (grup izni olsa bile) |
| `setfacl -m m::r dosya` | mask'i ayarlar |
| `setfacl -x u:isim dosya` | Tek bir girişi kaldırır |
| `setfacl -b dosya` | Bütün ek girişleri kaldırır |
| `setfacl -R -m u:isim:rX dizin` | Mevcut bütün içeriğe uygular |
| `setfacl -d -m u:isim:rX dizin` | Yeni oluşacak dosyalar için varsayılan ACL |
| `setfacl -k dizin` | Varsayılan ACL'i kaldırır |
| `getfacl -R dizin > yedek.txt` | ACL'leri yedekler |
| `setfacl --restore=yedek.txt` | ACL'leri geri yükler |
| `cp -a kaynak hedef` | ACL'leri koruyarak kopyalar |
| `chattr +a dosya` / `-a` | Sadece ekleme özniteliğini koyar / kaldırır |
| `chattr +i dosya` / `-i` | Değiştirilemez özniteliğini koyar / kaldırır |
| `lsattr dosya` | Öznitelikleri gösterir |

---

## 11. Sık yapılan hatalar

- **Dizin iznini unutmak.** Dosyaya ACL verdin ama kişi dizine giremiyorsa dosyaya ulaşamaz.
- **ACL'li dosyada `chmod` ile grup iznini kısmak.** mask değişir ve ACL izinleri de kısılır.
- **mask'i tamamen boş bırakmak.** Linux ACL girişlerine bakmaz, `other` izni uygulanır.
- **`-R`'nin yeni dosyaları da kapsayacağını sanmak.** Yeni dosyalar için `-d` (varsayılan ACL) gerekir.
- **Dizinlere toplu uygularken küçük `x` kullanmak.** Bütün dosyalar çalıştırılabilir olur. Büyük `X` kullan.
- **`-x` ile kaldırırken izin yazmak.** Doğrusu `setfacl -x u:isim dosya`.
- **`cp` ile kopyalayıp ACL'lerin geleceğini sanmak.** `cp -a` kullan.
- **Silinemeyen dosyada öznitelikleri kontrol etmemek.** `lsattr` ile `i` veya `a` var mı bak.

---

## 12. Alıştırmalar

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

Önceki konu: [Kaynak Limitleri](05-kaynak-limitleri.md) · Sonraki konu: [LDAP](07-ldap.md)
