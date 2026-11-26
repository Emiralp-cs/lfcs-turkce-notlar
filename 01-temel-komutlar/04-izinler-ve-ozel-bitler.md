# Dosya İzinleri, Sahiplik ve Özel Bitler

Linux'ta her dosyanın bir sahibi, bir grubu ve bu ikisi dışındakiler için ayrı izinleri vardır. Bu dosyada standart izinleri, sahipliği değiştirmeyi, yeni dosyaların izinlerini belirleyen `umask`'ı ve üç özel biti (SUID, SGID, sticky bit) ele alıyoruz.

Senaryo: `/srv/izin` altında ekibin ortak dosyaları var. `gelistirici` grubunda **ayse** ve **mert** var.

> [!NOTE]
> Bütün komutlar ve çıktılar Ubuntu 24.04 üzerinde denenmiştir. Kullanıcıya özel izinler (ACL) [Kullanıcılar ve Gruplar](../02-kullanicilar-ve-gruplar/06-acl.md) bölümünde anlatılıyor.

## İçindekiler

1. [Dosya izinlerini okumak](#1-dosya-izinlerini-okumak)
2. [Dosyada ve dizinde izinlerin anlamı](#2-dosyada-ve-dizinde-izinlerin-anlamı)
3. [Kontrol sırası: hangi izin uygulanır?](#3-kontrol-sırası-hangi-izin-uygulanır)
4. [chmod: sembolik yöntem](#4-chmod-sembolik-yöntem)
5. [chmod: sayısal (octal) yöntem](#5-chmod-sayısal-octal-yöntem)
6. [Sahipliği değiştirmek: chown ve chgrp](#6-sahipliği-değiştirmek-chown-ve-chgrp)
7. [Yeni dosyaların izinleri: umask](#7-yeni-dosyaların-izinleri-umask)
8. [SUID](#8-suid)
9. [SGID](#9-sgid)
10. [Sticky bit](#10-sticky-bit)
11. [Özel bitlerin gösterimi ve sayısal değerleri](#11-özel-bitlerin-gösterimi-ve-sayısal-değerleri)
12. [Özel izinli dosyaları bulmak](#12-özel-izinli-dosyaları-bulmak)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Dosya izinlerini okumak

```bash
ls -l plan.txt
```
```
-rw-rw-r-- 1 ayse gelistirici 5 Nov 25 15:38 plan.txt
```

İlk 10 karakter:

```
-  rw-  rw-  r--
│   │    │    └─ diğerleri (others)
│   │    └────── grup (group)
│   └─────────── sahip (user/owner)
└─────────────── tür: - dosya, d dizin, l link
```

Her üçlü sırasıyla **r** (okuma), **w** (yazma), **x** (çalıştırma) izinlerini gösterir. Harf yerine `-` varsa o izin yok.

İzinleri sayısal olarak da görmek için:

```bash
stat -c '%A %a %U:%G %n' plan.txt
```
```
-rw-rw-r-- 664 ayse:gelistirici plan.txt
```

---

## 2. Dosyada ve dizinde izinlerin anlamı

Aynı harf dosyada ve dizinde farklı anlama gelir:

| İzin | Dosyada | Dizinde |
|---|---|---|
| `r` | İçeriği okuyabilir | İçindeki **adları** listeleyebilir |
| `w` | İçeriği değiştirebilir | İçinde dosya **oluşturabilir, silebilir, adını değiştirebilir** |
| `x` | Program olarak çalıştırabilir | İçine girebilir ve içindeki dosyalara **erişebilir** |

Dizinlerdeki farkı denedik. Dizinin içinde Mert'in okuyabildiği bir dosya vardı:

| Dizin izni (diğerleri) | `ls dizin` | `cat dizin/dosya.txt` |
|---|---|---|
| `r--` (744) | Adları listeledi | `Permission denied` |
| `--x` (711) | `Permission denied` | Okudu |
| `r-x` (755) | Çalışır | Çalışır |

- `r` olup `x` yoksa adları görürsün ama dosyalara ulaşamazsın.
- `x` olup `r` yoksa listeleyemezsin, ama adını bildiğin dosyayı açabilirsin.

### Dizinde yazma izni = silme izni

Bir dosyayı silmek, **dizindeki** kaydı silmektir. Bu yüzden silme yetkisini dosyanın değil, dizinin izni belirler. `gelistirici` grubunun yazabildiği (`775`) bir dizinde root'a ait, Mert'in **yazamadığı** (`644`) bir dosya vardı:

```
Dosyaya yazmak:  Permission denied
Dosyayı silmek:  silindi
```

Mert dosyanın içeriğini değiştiremedi ama dosyayı tamamen sildi. Bunu engellemenin yolu [sticky bit](#10-sticky-bit).

---

## 3. Kontrol sırası: hangi izin uygulanır?

Linux bir erişimde şu sırayla bakar ve **ilk eşleşen** kategoriyi uygular:

1. Dosyanın **sahibi** misin? Öyleyse sadece sahip izinleri geçerli.
2. Değilse, dosyanın **grubunda** mısın? Öyleyse sadece grup izinleri geçerli.
3. Hiçbiri değilse **diğerleri** izinleri geçerli.

Sonuç biraz şaşırtıcı olabilir. Ayşe'nin sahibi olduğu dosyada sahibe sadece okuma, gruba okuma-yazma verdik:

```
-r--rw---- 1 ayse gelistirici 5 Nov 25 15:38 plan.txt
```

| Kullanıcı | Hangi izin uygulandı? | Yazma denemesi |
|---|---|---|
| ayse (sahip, ayrıca grupta) | Sahip: `r--` | `Permission denied` |
| mert (grupta) | Grup: `rw-` | Yazdı |

Ayşe de `gelistirici` grubunda ama sahip olduğu için grup izinlerine hiç bakılmadı. İzinler "en geniş olan" mantığıyla birleşmez. Root bu kontrollerin dışındadır, okuma ve yazmada izinlerden etkilenmez.

---

## 4. chmod: sembolik yöntem

```
chmod [kim][işlem][izin] dosya
```

| Kim | | İşlem | | İzin |
|---|---|---|---|---|
| `u` | sahip | `+` | ekle | `r` `w` `x` |
| `g` | grup | `-` | kaldır | `X` (aşağıda) |
| `o` | diğerleri | `=` | tam olarak bu yap | `s` `t` (özel bitler) |
| `a` | hepsi | | | |

`644` izinli bir dosyada sırayla denedik:

| Komut | Sonuç |
|---|---|
| `chmod u+x s.txt` | `-rwxr--r--` (744) |
| `chmod g+w,o-r s.txt` | `-rwxrw----` (760) |
| `chmod g=r s.txt` | `-rwxr-----` (740): grubun diğer izinleri silindi |
| `chmod a+r s.txt` | `-rwxr--r--` (744) |
| `chmod +x s.txt` | `-rwxr-xr-x` (755) |
| `chmod o= s.txt` | `-rwxr-x---` (750): diğerlerinin bütün izinleri kaldırıldı |
| `chmod u=rwx,g=rx,o= s.txt` | `-rwxr-x---` (750) |

- `+` ve `-` sadece belirtilen izne dokunur, `=` o kategorinin bütün izinlerini baştan yazar.
- Birden fazla değişiklik virgülle ayrılır, aralarında boşluk olmaz.
- Kim yazılmadığında (`chmod +x`) değişiklik herkese uygulanır, ama `umask`'ın kapattığı izinler eklenmez.

### Büyük X: sadece dizinlere çalıştırma izni

Bir dizin ağacına toplu izin verirken `chmod -R 640` gibi bir komut dizinlerin `x` iznini de kaldırır ve içlerine girilemez hale gelirler. `X`, `x` iznini sadece dizinlere (ve zaten çalıştırılabilir olan dosyalara) verir:

```bash
chmod -R u=rwX,g=rX,o= agac
```
```
-rw-r----- 1 root root    0 Nov 25 15:38 a.txt
drwxr-x--- 2 root root 4096 Nov 25 15:38 alt
```

Dosyalar `640`, dizinler `750` oldu.

---

## 5. chmod: sayısal (octal) yöntem

Her izinin bir değeri var ve bir kategorinin değeri, izinlerinin toplamıdır:

| İzin | Değer |
|---|---|
| `r` | 4 |
| `w` | 2 |
| `x` | 1 |

| Üçlü | Hesap | Sayı |
|---|---|---|
| `rwx` | 4+2+1 | 7 |
| `rw-` | 4+2 | 6 |
| `r-x` | 4+1 | 5 |
| `r--` | 4 | 4 |
| `---` | 0 | 0 |

Üç rakam sırasıyla sahip, grup ve diğerleridir:

```bash
chmod 640 dosya
```

Sık kullanılan değerler:

| Değer | İzin | Kullanım |
|---|---|---|
| `644` | `rw-r--r--` | Normal dosyalar |
| `600` | `rw-------` | Gizli dosyalar (SSH anahtarı gibi) |
| `640` | `rw-r-----` | Grubun okuyabildiği dosyalar |
| `755` | `rwxr-xr-x` | Programlar, script'ler ve normal dizinler |
| `750` | `rwxr-x---` | Sadece grubun girebildiği dizinler |
| `700` | `rwx------` | Sadece sahibin girebildiği dizinler (`~/.ssh` gibi) |
| `777` | `rwxrwxrwx` | Herkese her şey. Neredeyse hiçbir zaman doğru cevap değil |

| | Sembolik | Sayısal |
|---|---|---|
| Neyi değiştirir? | Sadece belirttiğin izinleri | Bütün izinleri baştan yazar |
| Ne zaman? | "Gruba yazma iznini ekle" gibi tek değişiklikler | "İzinler tam olarak şu olsun" |

---

## 6. Sahipliği değiştirmek: chown ve chgrp

```bash
sudo chown ayse dosya                  # sadece sahibi değiştir
sudo chown ayse:gelistirici dosya      # sahip ve grup
sudo chown :gelistirici dosya          # sadece grup
sudo chown ayse: dosya                 # sahip ayse, grup ayse'nin birincil grubu
sudo chgrp gelistirici dosya           # sadece grup
sudo chown -R ayse:gelistirici dizin   # dizin ve içindeki her şey
```

Kim neyi değiştirebilir? Ayşe kendi dosyasında denedi:

| Komut | Sonuç |
|---|---|
| `chgrp gelistirici ~/a.txt` (üyesi olduğu grup) | Çalıştı |
| `chgrp mert ~/a.txt` (üyesi olmadığı grup) | `Operation not permitted` |
| `chown mert ~/a.txt` | `Operation not permitted` |

- **Sahibi değiştirmek** sadece root'a açık. Normal bir kullanıcı kendi dosyasını bile başkasına veremez.
- **Grubu değiştirmek**: dosyanın sahibi, sadece üyesi olduğu gruplara değiştirebilir. Hangi gruplarda olduğunu `groups` ile görürsün.

---

## 7. Yeni dosyaların izinleri: umask

Yeni oluşturulan dosyaların izinleri, `umask`'ın **kapattığı** izinler çıkarılarak belirlenir:

| | Başlangıç | umask `022` ile | umask `002` ile | umask `027` ile |
|---|---|---|---|---|
| Dosya | `666` | `644` | `664` | `640` |
| Dizin | `777` | `755` | `775` | `750` |

Dosyalar `666`'dan başlar: yeni oluşturulan bir dosya hiçbir zaman çalıştırma izniyle oluşmaz.

```bash
umask
```

Ubuntu'da root için `0022`, normal kullanıcılar için `0002`. Denemede Ayşe'nin oluşturduğu dosya `664`, dizin `775` geldi. Sebebi `/etc/login.defs`'teki `USERGROUPS_ENAB yes` ayarı: her kullanıcının kendi adıyla bir grubu olduğu için grup yazma iznini açık bırakmak güvenli kabul ediliyor.

Geçici olarak değiştirmek için `umask 027`. Kalıcı yapmak için kullanıcının `~/.profile`'ına veya sistem geneli için `/etc/login.defs`'teki `UMASK` satırına yazılır ([Ortam Profilleri](../02-kullanicilar-ve-gruplar/04-ortam-profilleri.md)).

---

## 8. SUID

Normalde bir program, onu **çalıştıran** kullanıcının yetkileriyle çalışır. **SUID** (Set User ID) biti olan bir program ise dosyanın **sahibinin** yetkileriyle çalışır.

En bilinen örnek `passwd`:

```bash
ls -l /usr/bin/passwd
```
```
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
```

Sahibin `x` yerinde `s` var. Şifreler sadece root'un yazabildiği `/etc/shadow`'da tutulur. Normal bir kullanıcı `passwd` çalıştırdığında program root yetkisiyle çalışır ve kullanıcının kendi şifresini değiştirebilmesine izin verir.

### Denemek

`id` komutunun bir kopyasını Ayşe'ye ait yapıp Mert'e çalıştırttık:

```bash
sudo cp /usr/bin/id /srv/izin/kimim
sudo chown ayse:ayse /srv/izin/kimim
sudo chmod u+s /srv/izin/kimim
ls -l /srv/izin/kimim
```
```
-rwsr-xr-x 1 ayse ayse 39432 Nov 25 15:38 kimim
```

Mert çalıştırınca:

```
etkin: ayse, gercek: mert
```

Program Mert tarafından başlatıldı (gerçek kullanıcı) ama Ayşe'nin yetkileriyle çalıştı (etkin kullanıcı). SUID biti yokken iki değer de `mert`'ti.

### Bilinmesi gerekenler

- **Script'lerde çalışmaz.** Aynı denemeyi `4755` izinli, Ayşe'ye ait bir bash script'iyle yaptık. Script `mert` olarak çalıştı. Linux, güvenlik nedeniyle script'lerdeki SUID'i yok sayar, sadece derlenmiş programlarda geçerlidir.
- **`chown` SUID'i kaldırır.** `6755` izinli dosyanın sahibini değiştirince izinler `755`'e düştü. Böylece SUID'li bir dosyanın sahibi değiştirilerek yetki aktarılamaz.
- **Dizinlerde anlamı yoktur.**

> [!CAUTION]
> SUID'li ve sahibi root olan her program, bir güvenlik açığı varsa saldırgana root yetkisi verebilir. Kendi programlarına veya script'lerine SUID verme. Sistemdeki SUID'li dosyaları düzenli olarak kontrol et ([12. bölüm](#12-özel-izinli-dosyaları-bulmak)).

---

## 9. SGID

**SGID** (Set Group ID) dosyada ve dizinde farklı çalışır.

### Çalıştırılabilir dosyada

Program dosyanın **grubunun** yetkileriyle çalışır. SUID'in grup karşılığıdır. Grubun `x` yerinde `s` görünür: `-rwxr-sr-x`.

### Dizinde (daha sık kullanılan)

SGID'li bir dizinde oluşturulan dosya ve dizinler, oluşturanın birincil grubunu değil **dizinin grubunu** alır:

```bash
sudo mkdir /srv/izin/ortak
sudo chgrp gelistirici /srv/izin/ortak
sudo chmod 2775 /srv/izin/ortak
ls -ld /srv/izin/ortak
```
```
drwxrwsr-x 2 root gelistirici 4096 Nov 25 15:38 ortak
```

Ayşe bir dosya ve bir alt dizin oluşturdu:

```
-rw-rw-r-- 1 ayse gelistirici    0 Nov 25 15:38 a.txt
drwxrwsr-x 2 ayse gelistirici 4096 Nov 25 15:38 altdizin
```

İkisi de `gelistirici` grubuna ait. Yeni alt dizin SGID bitini de miras aldı (`drwxrwsr-x`), yani kural aşağı doğru yayılıyor. Ekip dizinlerinde bu sayede herkes birbirinin dosyalarına grup izniyle erişebilir ([Gruplar](../02-kullanicilar-ve-gruplar/02-gruplar.md#10-pratik-ortak-proje-dizini) bölümünde ayrıntılı).

> [!WARNING]
> Dizinlerde `chmod 755` gibi üç basamaklı bir sayı **SGID'i kaldırmaz**. Denemede `chmod 755 ortak` sonrası izinler `2755` (`drwxr-sr-x`) kaldı. Dizinin SGID'ini kaldırmak için `chmod g-s ortak` veya beş basamaklı `chmod 00755 ortak` kullan. Normal dosyalarda ise `chmod 755` özel bitleri temizler.

---

## 10. Sticky bit

Sticky bit, herkesin yazabildiği ortak dizinlerde kullanılır: dizine herkes dosya bırakabilir ama bir dosyayı sadece **sahibi** silebilir veya adını değiştirebilir. (Dizinin sahibi ve root da silebilir.)

En bilinen örnek `/tmp`:

```bash
ls -ld /tmp
```
```
drwxrwxrwt 12 root root 12288 Nov 25 15:38 /tmp
```

Diğerlerinin `x` yerinde `t` var.

### Denemek

```bash
sudo mkdir /srv/izin/paylasim
sudo chmod 1777 /srv/izin/paylasim
```

Ayşe dizine bir dosya bıraktı ve herkesin yazabilmesi için `666` izni verdi. Mert dosyaya bir şey eklemeyi ve dosyayı silmeyi denedi:

| | Sticky bit var | Sticky bit yok |
|---|---|---|
| Dosyaya yazmak (`666` izni sayesinde) | Yazdı | Yazdı |
| Dosyayı silmek | `Operation not permitted` | Sildi |

Sticky bit dosyanın içeriğini korumaz, onun için dosya izinleri var. Dosyanın **silinmesini ve yerinin değiştirilmesini** engeller. [2. bölümdeki](#dizinde-yazma-izni--silme-izni) "dizine yazabilen herkes her şeyi silebilir" sorununun çözümü budur.

---

## 11. Özel bitlerin gösterimi ve sayısal değerleri

Özel bitler `x`'in yerinde gösterilir:

| Bit | Nerede görünür? | Küçük harf | Büyük harf |
|---|---|---|---|
| SUID | Sahibin `x` yeri | `s`: SUID var, `x` de var | `S`: SUID var, `x` **yok** |
| SGID | Grubun `x` yeri | `s`: SGID var, `x` de var | `S`: SGID var, `x` yok |
| Sticky | Diğerlerinin `x` yeri | `t`: sticky var, `x` de var | `T`: sticky var, `x` yok |

Denemede SUID'li dosyadan `x`'i kaldırınca `-rwSr-xr-x`, `1770` izinli dizinde `drwxrwx--T` göründü. Büyük harf genelde bir hata işaretidir: SUID'li bir dosya çalıştırılamıyorsa SUID'in bir anlamı yoktur.

### Sayısal değerler

Özel bitler, üç basamaklı iznin **önüne** dördüncü bir rakam olarak eklenir:

| Bit | Değer |
|---|---|
| SUID | 4 |
| SGID | 2 |
| Sticky | 1 |

| Komut | Sonuç |
|---|---|
| `chmod 4755 kimim` | `-rwsr-xr-x` |
| `chmod 2755 kimim` | `-rwxr-sr-x` (önceki SUID kalktı) |
| `chmod 6755 kimim` | `-rwsr-sr-x` (4+2) |
| `chmod 2775 ortak` | `drwxrwsr-x` |
| `chmod 1777 paylasim` | `drwxrwxrwt` |

Sembolik yöntemle:

```bash
chmod u+s dosya      # SUID
chmod g+s dizin      # SGID
chmod +t dizin       # sticky (o+t de olur)
chmod u-s,g-s dosya  # kaldır
```

---

## 12. Özel izinli dosyaları bulmak

`find`'ın `-perm` seçeneği üç şekilde kullanılır:

| Biçim | Anlamı |
|---|---|
| `-perm 664` | İzinler **tam olarak** 664 |
| `-perm -4000` | **En az** bu bitler açık (SUID var, gerisi önemli değil) |
| `-perm /6000` | Bu bitlerden **herhangi biri** açık (SUID **veya** SGID) |

```bash
find / -perm -4000 -type f 2>/dev/null         # SUID'li dosyalar
find / -perm -2000 -type f 2>/dev/null         # SGID'li dosyalar
find / -perm /6000 -type f 2>/dev/null         # SUID veya SGID
find / -xdev -perm -1000 -type d 2>/dev/null   # sticky bit'li dizinler
```

Denemede `/usr/bin` altında SUID'li `sudo`, `chfn`, `chsh`, `gpasswd` gibi programlar, sticky bit'li dizin olarak `/tmp` ve `/var/tmp` bulundu. `2>/dev/null` erişilemeyen dizinlerden gelen hataları gizler, `-xdev` aramayı tek dosya sisteminde tutar.

`find` ayrıntılı olarak [find](05-find.md) dosyasında.

---

## 13. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `ls -l` / `ls -ld dizin` | İzinleri gösterir |
| `stat -c '%A %a' dosya` | İzinleri harf ve sayı olarak gösterir |
| `chmod u+x dosya` | Sembolik: izin ekler |
| `chmod g=r,o= dosya` | Sembolik: izni baştan yazar |
| `chmod 640 dosya` | Sayısal |
| `chmod -R u=rwX,g=rX,o= dizin` | Dizinlere x, dosyalara x vermeden toplu izin |
| `chown kullanici:grup dosya` | Sahip ve grup (root) |
| `chgrp grup dosya` | Grup |
| `umask` / `umask 027` | Yeni dosyaların izin maskesi |
| `chmod u+s` / `4xxx` | SUID |
| `chmod g+s` / `2xxx` | SGID |
| `chmod +t` / `1xxx` | Sticky bit |
| `chmod g-s dizin` / `chmod 00755 dizin` | Dizinin SGID'ini kaldırır |
| `find / -perm -4000 -type f` | SUID'li dosyalar |
| `find / -perm /6000 -type f` | SUID veya SGID'li dosyalar |

---

## 14. Sık yapılan hatalar

- **Sahibin grup izinlerinden yararlandığını sanmak.** Sahip için sadece sahip izinleri geçerli.
- **Dizinde `r`'yi yeterli sanmak.** İçindeki dosyalara erişmek için `x` gerekir.
- **Dosya izinleriyle silmeyi engellediğini sanmak.** Silme, dizinin yazma iznine bağlı. Sticky bit kullan.
- **`chmod -R 644 dizin`.** Dizinlerin `x` izni gider. `X` kullan.
- **`=` ile `+`'yı karıştırmak.** `g=r` grubun diğer izinlerini siler.
- **Normal kullanıcının `chown` yapabileceğini sanmak.**
- **Script'e SUID vermek.** Linux yok sayar.
- **Dizinde `chmod 755`'in SGID'i kaldırdığını sanmak.** `g-s` kullan.
- **Büyük `S` veya `T`'yi gözden kaçırmak.** `x` izni eksik demektir.
- **`find -perm 4000` yazmak.** Tam eşleşme arar ve neredeyse hiçbir şey bulmaz. `-4000` kullan.

---

## 15. Alıştırmalar

**Hazırlık:**

```bash
sudo groupadd ekip
sudo useradd -m -s /bin/bash elif
sudo useradd -m -s /bin/bash can
sudo usermod -aG ekip elif
sudo usermod -aG ekip can
sudo mkdir /srv/alistirma
```

**1.** `/srv/alistirma/rapor.txt` oluştur. Sahibi `elif`, grubu `ekip` olsun. Sahip okuyup yazabilsin, grup sadece okuyabilsin, diğerleri hiçbir şey yapamasın. Önce sayısal, sonra aynı sonucu sembolik yöntemle ayarla.

<details>
<summary>Çözüm</summary>

```bash
sudo touch /srv/alistirma/rapor.txt
sudo chown elif:ekip /srv/alistirma/rapor.txt
sudo chmod 640 /srv/alistirma/rapor.txt
sudo chmod u=rw,g=r,o= /srv/alistirma/rapor.txt
ls -l /srv/alistirma/rapor.txt     # -rw-r----- elif ekip
```
</details>

**2.** `rapor.txt`'nin izinleri `-r--rw----` olsaydı, `elif` (sahip ve `ekip` üyesi) dosyaya yazabilir miydi? `can` yazabilir miydi?

<details>
<summary>Çözüm</summary>

`elif` yazamaz: sahip olduğu için sadece sahip izinleri (`r--`) geçerli, grup izinlerine bakılmaz. `can` yazabilir: sahip değil, grup üyesi, grup izinleri `rw-`.
</details>

**3.** `/srv/alistirma/ortak` dizinini oluştur. `ekip` grubu dizine yazabilsin, dizinde oluşturulan dosyalar otomatik olarak `ekip` grubuna ait olsun ve herkes sadece kendi dosyasını silebilsin. Diğerleri dizine giremesin.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir /srv/alistirma/ortak
sudo chgrp ekip /srv/alistirma/ortak
sudo chmod 3770 /srv/alistirma/ortak       # 2 (SGID) + 1 (sticky) = 3
ls -ld /srv/alistirma/ortak                # drwxrws--T
```

Diğerlerinin `x` izni olmadığı için sticky bit büyük `T` olarak görünür. Bu durumda normal, çünkü diğerlerinin zaten erişimi yok.

Doğrulama:

```bash
sudo su - elif -c 'touch /srv/alistirma/ortak/elif.txt; ls -l /srv/alistirma/ortak'
sudo su - can -c 'rm /srv/alistirma/ortak/elif.txt'    # Operation not permitted
```
</details>

**4.** Sistemdeki SUID veya SGID biti olan bütün dosyaları listele.

<details>
<summary>Çözüm</summary>

```bash
sudo find / -perm /6000 -type f 2>/dev/null
```
</details>

**5.** `-rwSr--r--` izinli bir dosya gördün. Bu ne anlama geliyor? Sayısal değeri nedir? Sonra alıştırmada oluşturduğun her şeyi temizle.

<details>
<summary>Çözüm</summary>

SUID biti açık ama sahibin çalıştırma izni yok, bu yüzden SUID'in bir etkisi yok. Sayısal değer `4644`.

```bash
sudo rm -r /srv/alistirma
sudo userdel -r elif
sudo userdel -r can
sudo groupdel ekip
```
</details>

---

Önceki konu: [Hard ve Soft Link](03-hard-ve-soft-link.md) · Sonraki konu: [find ile Dosya Arama](05-find.md)
