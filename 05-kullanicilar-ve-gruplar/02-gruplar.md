# Gruplar

Ekipteki herkesin aynı proje dizinine yazabilmesi gerekiyor. İzni kullanıcılara tek tek vermek yerine, hepsini bir gruba koyup izni gruba veririz. Ekibe biri katıldığında onu gruba eklemek, ayrıldığında da çıkarmak yeterli olur.

Önceki bölümdeki senaryoya devam ediyoruz:

- **ayse**: geliştirici
- **mert**: stajyer
- **gelistirici**: ekip grubu, proje dizinine erişim bu grup üzerinden
- **stajyer**: stajyerlerin grubu

> [!NOTE]
> Komutlar Ubuntu 24.04 üzerinde denenmiştir. Çıktılardaki UID/GID numaraları senin sisteminde farklı olabilir.

## İçindekiler

1. [Birincil grup ve ek gruplar](#1-birincil-grup-ve-ek-gruplar)
2. [/etc/group ve /etc/gshadow](#2-etcgroup-ve-etcgshadow)
3. [Grup oluşturma: groupadd](#3-grup-oluşturma-groupadd)
4. [Kullanıcıyı ek gruba ekleme](#4-kullanıcıyı-ek-gruba-ekleme)
5. [Üyelik ne zaman geçerli olur?](#5-üyelik-ne-zaman-geçerli-olur)
6. [Kullanıcıyı gruptan çıkarma](#6-kullanıcıyı-gruptan-çıkarma)
7. [Birincil grubu değiştirme](#7-birincil-grubu-değiştirme)
8. [Grubu yeniden adlandırma ve GID değiştirme: groupmod](#8-grubu-yeniden-adlandırma-ve-gid-değiştirme-groupmod)
9. [Grubu silme: groupdel](#9-grubu-silme-groupdel)
10. [Pratik: ortak proje dizini](#10-pratik-ortak-proje-dizini)
11. [Özel anlamı olan gruplar](#11-özel-anlamı-olan-gruplar)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. Birincil grup ve ek gruplar

Her kullanıcının bir **birincil** grubu vardır ve istediği kadar **ek** gruba üye olabilir:

| | Birincil grup | Ek grup |
|---|---|---|
| Sayısı | Her zaman tam bir tane | Sıfır veya daha fazla |
| Nerede yazar? | `/etc/passwd`'ın 4. alanı (GID) | `/etc/group`'un 4. alanı |
| Ne işe yarar? | Kullanıcının oluşturduğu dosyaların grubu bu olur | Grubun izinlerinden yararlanmayı sağlar |
| Değiştiren komut | `usermod -g` | `usermod -aG`, `gpasswd -a` |

`id` çıktısında ikisi de görünür:

```bash
id ayse
```
```
uid=1001(ayse) gid=1001(ayse) groups=1001(ayse),1501(gelistirici)
```

`gid=` birincil grubu, `groups=` ise bütün grupları gösterir.

---

## 2. /etc/group ve /etc/gshadow

```bash
grep -E '^(ayse|gelistirici):' /etc/group
```
```
ayse:x:1001:
gelistirici:x:1501:ayse,mert
```

| Sıra | Örnek | Anlamı |
|---|---|---|
| 1 | `gelistirici` | Grup adı |
| 2 | `x` | Grup şifresinin `/etc/gshadow`'da olduğunu gösterir |
| 3 | `1501` | GID |
| 4 | `ayse,mert` | Bu grubu **ek grup** olarak kullananlar |

`ayse` grubunun 4. alanı boş, oysa Ayşe'nin birincil grubu bu. Birincil grup üyeleri burada listelenmez, onlar `/etc/passwd`'daki GID'den bilinir. Bir grubun bütün üyelerini görmek istiyorsan iki yere de bakman gerekir.

Grup şifreleri ve grup yöneticileri `/etc/gshadow`'da tutulur. Günlük işlerde pek gerekmez:

```bash
sudo getent gshadow gelistirici
```
```
gelistirici:!::ayse,mert
```

---

## 3. Grup oluşturma: groupadd

```bash
sudo groupadd gelistirici
getent group gelistirici
```
```
gelistirici:x:1501:
```

GID'yi elle vermek için `-g` kullanılır. Örneğin GID, başka sunuculardaki grupla aynı olsun:

```bash
sudo groupadd -g 2500 stajyer
getent group stajyer
```
```
stajyer:x:2500:
```

Kullanılan bir GID'yi tekrar vermeye çalışırsan hata alırsın:

```
groupadd: GID '2500' already exists
```

Servisler için sistem grubu oluşturmak istersen `sudo groupadd --system grupadi` yaz. GID 1000'in altından verilir.

---

## 4. Kullanıcıyı ek gruba ekleme

Aynı işi yapan iki komut var:

```bash
sudo gpasswd -a ayse gelistirici
sudo usermod -aG gelistirici mert
```

Argümanların sırası farklı:

| Komut | Sıra |
|---|---|
| `gpasswd -a KULLANICI GRUP` | Önce kullanıcı, sonra grup |
| `usermod -aG GRUP KULLANICI` | Önce grup, sonra kullanıcı |

Kontrol etmek için:

```bash
getent group gelistirici
groups mert
```
```
gelistirici:x:1501:ayse,mert
mert : mert gelistirici
```

Birden fazla gruba aynı anda eklemek için grupları virgülle ayır: `sudo usermod -aG gelistirici,stajyer mert`

> [!CAUTION]
> `usermod -G`'yi `-a` olmadan kullanırsan, kullanıcı yazdığın gruplar **dışındaki bütün ek gruplardan çıkarılır**.
>
> ```bash
> id mert
> # groups=1500(mert),1501(gelistirici),2500(stajyer)
> sudo usermod -G stajyer mert
> id mert
> # groups=1500(mert),2500(stajyer)    <- gelistirici gitti
> ```
>
> `-a` (append) "mevcutlara ekle" demektir. Ekleme yaparken her zaman `-aG` yaz.

---

## 5. Üyelik ne zaman geçerli olur?

Grup değişiklikleri **açık oturumları etkilemez**. Kullanıcının grupları, oturum açtığı anda belirlenir.

Denemede Ayşe'nin oturumu açıkken onu `operasyon` grubuna ekledik:

```
Oturum içinde, eklemeden önce:   ayse gelistirici
Oturum içinde, eklemeden sonra:  ayse gelistirici
Yeni oturumda:                   ayse gelistirici operasyon
```

Yani "gruba ekledim ama izin hala yok" durumunun en sık sebebi budur. Çözümü:

- Kullanıcı oturumu kapatıp tekrar açar (SSH'tan çıkıp yeniden bağlanır).
- Ya da o terminalde `newgrp operasyon` yazar. Bu komut, o gruba geçerli olarak yeni bir shell açar. Bu shell'de oluşturulan dosyaların grubu da `operasyon` olur.

> [!TIP]
> `id ayse` gibi kullanıcı adı vererek çalıştırılan komutlar güncel bilgiyi gösterir. Kullanıcının kendi oturumunda yazdığı düz `id` ise oturumun açıldığı andaki grupları gösterir. Bu fark kafa karıştırmasın.

---

## 6. Kullanıcıyı gruptan çıkarma

```bash
sudo gpasswd -d mert stajyer
```
```
Removing user mert from group stajyer
```

`usermod` ile de yapılabilir:

```bash
sudo usermod -r -G stajyer mert
```

Bir grubun üye listesini bir kerede yazmak için `gpasswd -M` kullanılır. Bu komut mevcut listeyi tamamen değiştirir:

```bash
sudo gpasswd -M ayse,mert gelistirici   # üyeler artık sadece ayse ve mert
sudo gpasswd -M "" gelistirici           # bütün ek üyeleri çıkar
```

---

## 7. Birincil grubu değiştirme

```bash
sudo usermod -g gelistirici mert
id mert
```
```
uid=1500(mert) gid=1501(gelistirici) groups=1501(gelistirici)
```

> [!IMPORTANT]
> `-g` (küçük harf) birincil grubu değiştirir. `-G` (büyük harf) ek grupları belirler. Tek harf farkı tamamen farklı bir iş yapar.

Mevcut dosyalara ne olduğuna bakalım. Mert'in biri ev dizininde, biri dışarıda iki dosyası vardı:

```
Önce:
-rw-rw-r-- 1 mert mert 0 Nov 25 12:48 /home/mert/rapor.txt
-rw-rw-r-- 1 mert mert 0 Nov 25 12:48 /tmp/mertdir/disari.txt

usermod -g gelistirici mert sonrası:
-rw-rw-r-- 1 mert gelistirici 0 Nov 25 12:48 /home/mert/rapor.txt
-rw-rw-r-- 1 mert mert        0 Nov 25 12:48 /tmp/mertdir/disari.txt
```

`usermod -g`, sadece **ev dizinindeki** dosyaların grubunu yeni gruba çevirir. Başka yerlerdeki dosyalar eski grupta kalır. Onları da değiştirmek gerekiyorsa `chgrp` ile elle yapılır.

Eski `mert` grubu silinmez, boş olarak durmaya devam eder.

---

## 8. Grubu yeniden adlandırma ve GID değiştirme: groupmod

### Yeniden adlandırma

```bash
sudo groupmod -n devops operasyon
getent group devops
```
```
devops:x:2501:ayse
```

GID ve üyeler aynı kalır, sadece ad değişir. Dosyaların sahiplik bilgisi GID olarak tutulduğu için dosyalar da otomatik olarak yeni adla görünür.

### GID değiştirme

```bash
sudo groupmod -g 2600 devops
```

> [!WARNING]
> GID'yi değiştirdiğinde mevcut dosyalar **eski numarada kalır** ve artık hiçbir gruba ait olmazlar:
> ```
> -rw-r--r-- 1 root 2501 0 Nov 25 12:48 /tmp/g.txt
> ```
> Bunları yeni gruba bağlamak için eski GID'ye sahip dosyaları bulup grubunu değiştir:
> ```bash
> sudo find / -gid 2501 -exec chgrp devops {} + 2>/dev/null
> ```

---

## 9. Grubu silme: groupdel

```bash
sudo groupdel devops
```

Ek üyeleri olan bir grup sorunsuz silinir, üyeler sadece o gruptan çıkmış olur. Ama bir kullanıcının **birincil** grubu silinemez:

```
groupdel: cannot remove the primary group of user 'mert'
```

Önce kullanıcının birincil grubunu başka bir gruba çevir, sonra sil:

```bash
sudo usermod -g mert mert
sudo groupdel gelistirici
```

Silinen grubun dosyaları da GID değişikliğinde olduğu gibi numarayla görünmeye başlar.

---

## 10. Pratik: ortak proje dizini

Grupların en yaygın kullanımı bir ortak çalışma dizini. `gelistirici` grubundakiler `/srv/proje` dizinine yazabilsin, diğerleri dizine hiç giremesin:

```bash
sudo mkdir -p /srv/proje
sudo chgrp gelistirici /srv/proje
sudo chmod 2770 /srv/proje
ls -ld /srv/proje
```
```
drwxrws--- 2 root gelistirici 4096 Nov 25 12:48 /srv/proje
```

- `770`: sahip ve grup her şeyi yapabilir, diğerleri hiçbir şey yapamaz.
- Baştaki `2`: setgid biti. İzinlerde `s` olarak görünür.

setgid'nin farkı şu: dizinde oluşturulan dosyalar, oluşturan kişinin birincil grubunu değil, **dizinin grubunu** alır.

```
setgid olmadan:  -rw-rw-r-- 1 mert mert        0 ... setgidsiz.txt
setgid ile:      -rw-rw-r-- 1 mert gelistirici 0 ... setgidli.txt
```

setgid olmadan Mert'in oluşturduğu dosyanın grubu `mert` olur ve Ayşe o dosyayı düzenleyemez. Ekip dizinlerinde setgid'yi açmayı unutma.

Gruba üye olmayan biri dizine erişmeye çalıştığında:

```
ls: cannot open directory '/srv/proje': Permission denied
```

---

## 11. Özel anlamı olan gruplar

Bazı grupların üyeliği doğrudan yetki verir:

| Grup | Dağıtım | Verdiği yetki |
|---|---|---|
| `sudo` | Debian/Ubuntu | `sudo` ile her komutu root olarak çalıştırabilir |
| `wheel` | RHEL/Fedora | Aynı işi RHEL ailesinde yapar |
| `adm` | Debian/Ubuntu | `/var/log` altındaki çoğu log dosyasını okuyabilir |
| `docker` | Docker kurulu sistemler | Docker'ı sudo olmadan kullanabilir |

```bash
sudo usermod -aG sudo ayse   # Ubuntu'da ayse'yi yönetici yap
```

> [!WARNING]
> `docker` grubu pratikte root yetkisine eşittir. Bu gruptaki biri, ana makinenin dosya sistemini bir konteynere bağlayıp her dosyayı okuyabilir ve değiştirebilir. Bu gruba sadece root yetkisi vereceğin kişileri ekle.

`sudo` ve `wheel` gruplarının nasıl yetki verdiğini [sudo ve su](03-sudo-ve-su.md) bölümünde göreceğiz.

---

## 12. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `groupadd grup` | Grup oluşturur |
| `groupadd -g 2500 grup` | Belirli GID ile oluşturur |
| `getent group grup` | Grubun kaydını ve ek üyelerini gösterir |
| `id kullanici` | Kullanıcının birincil ve ek gruplarını gösterir |
| `groups kullanici` | Kullanıcının gruplarını kısa listeler |
| `gpasswd -a kullanici grup` | Ek gruba ekler |
| `usermod -aG grup kullanici` | Ek gruba ekler |
| `usermod -G grup kullanici` | Ek grupları **sadece** bunlar yapar (dikkat) |
| `gpasswd -d kullanici grup` | Ek gruptan çıkarır |
| `usermod -r -G grup kullanici` | Ek gruptan çıkarır |
| `gpasswd -M k1,k2 grup` | Üye listesini baştan yazar |
| `usermod -g grup kullanici` | Birincil grubu değiştirir |
| `newgrp grup` | Oturumu kapatmadan yeni gruba geçen bir shell açar |
| `groupmod -n yeniad eskiad` | Grubu yeniden adlandırır |
| `groupmod -g GID grup` | GID'yi değiştirir (dosyaları elle düzelt) |
| `groupdel grup` | Grubu siler |
| `chmod 2770 dizin` | Ortak dizin: grup erişimi ve setgid |

---

## 13. Sık yapılan hatalar

- **`-a`'yı unutmak.** `usermod -G` kullanıcıyı diğer bütün ek gruplardan çıkarır.
- **`-g` ile `-G`'yi karıştırmak.** Küçük harf birincil grubu, büyük harf ek grupları değiştirir.
- **Argüman sırasını karıştırmak.** `gpasswd -a` önce kullanıcıyı, `usermod -aG` önce grubu ister.
- **Üyeliğin hemen geçerli olmasını beklemek.** Kullanıcı yeniden giriş yapmalı ya da `newgrp` kullanmalı.
- **`/etc/group`'ta birincil üyeleri aramak.** 4. alanda sadece ek üyeler yazar.
- **GID değiştirip dosyaları unutmak.** Eski GID'li dosyalar sahipsiz kalır.
- **Ortak dizinde setgid'yi açmamak.** Dosyalar oluşturanın grubunu alır ve ekip arkadaşları düzenleyemez.

---

## 14. Alıştırmalar

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

Önceki konu: [Kullanıcı Hesapları](01-kullanici-hesaplari.md) · Sonraki konu: [sudo ve su](03-sudo-ve-su.md)
