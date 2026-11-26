# find ile Dosya Arama

`find`, dosyaları adına, türüne, boyutuna, tarihine, izinlerine ve sahibine göre arar, bulduklarına bir komut da uygulayabilir. Sınavda "son 10 dakikada değişen dosyaları bul", "100 MB'tan büyük logları listele" veya "SUID'li dosyaları bul" gibi görevlerin hepsi `find` ile çözülür.

Senaryo: `/srv/arama` altında raporlar, loglar ve yedekler var. Disk dolmak üzere ve neyin yer kapladığını, neyin yakın zamanda değiştiğini bulmamız gerekiyor:

```
/srv/arama
├── loglar/        uygulama.log, orta.log (1,5 KB), buyuk.log (600 KB), bos.log
├── loglar-link -> /srv/arama/loglar
├── raporlar/      Kasim.txt, kasim.TXT, ozet.pdf (sahibi ayse)
│   └── 2026/      ocak.txt (3 gün önce değişti)
└── yedek/         arsiv.tar (2 MB)
```

> [!NOTE]
> Bütün örnekler Ubuntu 24.04'te bu dizin yapısıyla denenmiş, çıktılar gerçek sonuçlardır.

## İçindekiler

1. [Sözdizimi](#1-sözdizimi)
2. [Ada göre: -name ve -iname](#2-ada-göre--name-ve--iname)
3. [Türe ve derinliğe göre](#3-türe-ve-derinliğe-göre)
4. [Boyuta göre: -size](#4-boyuta-göre--size)
5. [Zamana göre: -mmin, -mtime, -newer](#5-zamana-göre--mmin--mtime--newer)
6. [Sahibe ve izne göre](#6-sahibe-ve-izne-göre)
7. [Koşulları birleştirmek: ve, veya, değil](#7-koşulları-birleştirmek-ve-veya-değil)
8. [Bulunanlara komut uygulamak: -exec ve -delete](#8-bulunanlara-komut-uygulamak--exec-ve--delete)
9. [Hatalar ve sembolik linkler](#9-hatalar-ve-sembolik-linkler)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Sözdizimi

```
find [NEREDE] [KOŞULLAR] [EYLEM]
find /srv/arama -name "*.log" -exec ls -l {} \;
```

- **Nerede:** Aramanın başlayacağı dizin(ler). Yazılmazsa bulunduğun dizin (`.`).
- **Koşullar:** `-name`, `-size`, `-mtime` gibi testler. Yan yana yazılanların hepsi sağlanmalı.
- **Eylem:** Bulunanlarla ne yapılacağı. Yazılmazsa ekrana yazdırır (`-print`).

`find` alt dizinlerin hepsine iner. Önce yolu, sonra koşulları yaz. `find -name x /srv` yanlış sıradır.

---

## 2. Ada göre: -name ve -iname

```bash
find /srv/arama -name "kasim.txt"
find /srv/arama -iname "kasim.txt"
```

`-name` büyük-küçük harfe duyarlı olduğu için hiçbir şey bulmadı. `-iname` ikisini de buldu:

```
/srv/arama/raporlar/Kasim.txt
/srv/arama/raporlar/kasim.TXT
```

Joker karakterler:

| Desen | Uyan |
|---|---|
| `"*.log"` | `.log` ile biten |
| `"rapor*"` | `rapor` ile başlayan |
| `"*2026*"` | İçinde `2026` geçen |
| `"rapor?.txt"` | `rapor` + tek karakter + `.txt` (`rapor1.txt`) |

`-name` sadece dosyanın **adına** bakar, yoluna değil. Yolda arama yapmak için `-path "*/2026/*"`.

> [!WARNING]
> Desenleri **her zaman tırnak içine al.** Tırnak olmazsa shell, `*.log`'u `find` çalışmadan önce bulunduğun dizindeki dosya adlarıyla değiştirir. `loglar` dizinindeyken tırnaksız denedik:
> ```
> find: paths must precede expression: `buyuk.log'
> find: possible unquoted pattern after predicate `-name'?
> ```
> Bulunduğun dizinde tek bir `.log` dosyası olsaydı hata bile çıkmazdı. `find` sadece o dosyanın adını arardı ve diğerlerini sessizce atlardı.

---

## 3. Türe ve derinliğe göre

| Koşul | Anlamı |
|---|---|
| `-type f` | Normal dosya |
| `-type d` | Dizin |
| `-type l` | Sembolik link |

```bash
find /srv/arama -type d
find /srv/arama -type l
```
```
/srv/arama/loglar-link
```

Ne kadar derine inileceğini sınırlamak için:

```bash
find /srv/arama -maxdepth 1          # sadece /srv/arama'nın kendisi ve hemen içi
find /srv/arama -mindepth 2 -type f  # en az iki seviye aşağıdakiler
```

`-type f` eklemek iyi bir alışkanlık. Eklemezsen dizinler de sonuçlara karışır ([4. bölümde](#4-boyuta-göre--size) örneği var).

---

## 4. Boyuta göre: -size

| Birim | Anlamı |
|---|---|
| `c` | Bayt |
| `k` | Kibibayt (1024 bayt), **küçük harf** |
| `M` | Mebibayt |
| `G` | Gibibayt |

| Önek | Anlamı |
|---|---|
| `+` | Büyük |
| `-` | Küçük |
| (yok) | Tam bu boyutta |

```bash
find /srv/arama -size +500k
```
```
/srv/arama/loglar/buyuk.log
/srv/arama/yedek/arsiv.tar
```

```bash
find /srv/arama -size +1M
```
```
/srv/arama/yedek/arsiv.tar
```

Boş dosyalar için `-size 0` veya `-empty`.

`k` büyük yazılırsa hata alırsın: `find: invalid -size type 'K'`.

> [!CAUTION]
> `find`, boyutu verdiğin birime **yukarı yuvarlar**. 600 KB'lık bir dosya `-size` için "1 M" sayılır. Bu yüzden `-size -1M` ("1 MB'tan küçük") **sadece boş dosyaları** bulur:
> ```bash
> find /srv/arama -type f -size -1M       # sadece 0 baytlık dosyalar
> find /srv/arama -type f -size -1024k    # 1 MB'tan küçük bütün dosyalar
> ```
> Denemede ilk komut 600 KB'lık ve 1,5 KB'lık dosyaları bulmadı, ikincisi buldu. "Küçüktür" araması yaparken daha küçük bir birim kullan.

Aralık aramak için iki koşulu yan yana yaz:

```bash
find /srv/arama -type f -size +1k -size -100k
```

`-type f`'yi unutursak dizinler de çıkar, çünkü her dizin kaydı 4 KB'tır. Denemede `-type f` olmadan `/srv/arama`, `raporlar`, `loglar` gibi dizinler de listelendi.

---

## 5. Zamana göre: -mmin, -mtime, -newer

Her dosyanın üç zamanı vardır:

| | Ne zaman değişir? | Dakika | Gün |
|---|---|---|---|
| **m** (modify) | İçerik değişince | `-mmin` | `-mtime` |
| **c** (change) | İçerik veya **meta veri** (izin, sahip) değişince | `-cmin` | `-ctime` |
| **a** (access) | Dosya okununca | `-amin` | `-atime` |

`-` "daha yakın", `+` "daha eski" demektir:

```bash
find /srv/arama -type f -mmin -5             # son 5 dakikada değişenler
find /srv/arama -type f -mmin +5 -mmin -15   # 5 ile 15 dakika önce arasında
```
```
/srv/arama/loglar/orta.log
```

### -mtime'ın günleri

`-mtime`, geçen süreyi 24 saatlik dilimlere böler ve **küsuratı atar**:

| Dosyanın yaşı | `-mtime` değeri |
|---|---|
| 10 dakika | 0 |
| 30 saat | 1 |
| 50 saat | 2 |
| 3 gün | 3 |

Denemede:

| Komut | Bulduğu |
|---|---|
| `-mtime 0` | Son 24 saatte değişenler |
| `-mtime 1` | Sadece 30 saatlik `ozet.pdf` (24–48 saat arası) |
| `-mtime +1` | 50 saatlik `buyuk.log` ve 3 günlük `ocak.txt` (48 saatten eski) |
| `-mtime -2` | 48 saatten yeni olanlar |

`-mtime +1`'in "1 günden eski" değil, "**2 günden** eski" anlamına gelmesi sık karıştırılır. "7 günden eski" için `-mtime +6`, ya da daha kesin olmak için `-mmin +10080` kullanılabilir.

### mtime ve ctime farkı

Denemede `ocak.txt`'nin içerik zamanını 3 gün önceye çektik, ama `touch` dosyanın meta verisini de değiştirdiği için `-cmin -2` dosyayı yine buldu. İzin değiştirmek de ctime'ı günceller, mtime'ı değiştirmez. "Son zamanlarda **izni** veya **sahibi** değişmiş dosyalar" aranıyorsa `-ctime`/`-cmin`.

### Başka bir dosyaya göre

```bash
find /srv/arama -type f -newer loglar/orta.log
```

`orta.log`'dan sonra değiştirilen her dosyayı bulur. Bir işlemden önce `touch /tmp/isaret` yapıp sonra `find / -newer /tmp/isaret` ile işlemin değiştirdiği dosyaları bulmak kullanışlı bir yöntem.

---

## 6. Sahibe ve izne göre

### -perm

| Biçim | Anlamı | Örnek |
|---|---|---|
| `-perm 664` | İzinler **tam olarak** 664 | Sadece `rw-rw-r--` |
| `-perm -664` | **En az** bu izinler açık | 664, 666, 775... |
| `-perm /022` | Bu bitlerden **herhangi biri** açık | Grup **veya** diğerleri yazabiliyor |

Sembolik biçim de kullanılabilir:

```bash
find /srv/arama -type f -perm 600            # tam olarak rw-------
find /srv/arama -type f -perm /o=w           # diğerlerinin yazabildiği dosyalar
find /srv/arama -type f ! -perm -o=r         # diğerlerinin OKUYAMADIĞI dosyalar
```
```
/srv/arama/yedek/arsiv.tar
```

SUID, SGID ve sticky bit aramaları [İzinler](04-izinler-ve-ozel-bitler.md#12-özel-izinli-dosyaları-bulmak) dosyasında.

### Sahip ve grup

```bash
find /srv/arama -user ayse              # sahibi ayse
find /srv/arama -group root -name "*.pdf"
find / -nouser 2>/dev/null              # sahibi silinmiş dosyalar
find / -nogroup 2>/dev/null             # grubu silinmiş dosyalar
```

`-nouser`, `userdel` ile silinmiş kullanıcılardan kalan dosyaları bulmak için kullanılır ([Kullanıcı Hesapları](../05-kullanicilar-ve-gruplar/01-kullanici-hesaplari.md#7-hesabı-silme-userdel)).

---

## 7. Koşulları birleştirmek: ve, veya, değil

| Operatör | Anlamı | Yazımı |
|---|---|---|
| Ve | İkisi de sağlanmalı | Yan yana yazmak (veya `-a`) |
| Veya | Biri yeterli | `-o` |
| Değil | Sağlanmamalı | `!` veya `-not` |

```bash
find /srv/arama -type f -name "*.log" -size +1k       # ve
find /srv/arama -name "*.pdf" -o -name "*.tar"        # veya
find /srv/arama -type f ! -name "*.log"               # değil
```

### Parantez gerekiyor

"Ve", "veya"dan önce işlenir. Bu yüzden `-o` ile başka koşulları birlikte kullanırken parantez gerekir:

```bash
find /srv/arama -type f -name "*.pdf" -o -name "*.tar" -o -name "loglar*"
```
```
/srv/arama/loglar-link
/srv/arama/raporlar/ozet.pdf
/srv/arama/loglar
/srv/arama/yedek/arsiv.tar
```

`-type f` sadece ilk koşula bağlandı: sonuçlarda bir dizin (`loglar`) ve bir link (`loglar-link`) çıktı. Doğrusu:

```bash
find /srv/arama -type f \( -name "*.pdf" -o -name "*.tar" \)
```

Parantezler shell'e özel karakter olduğu için `\(` ve `\)` diye kaçırılır, parantezlerin iki yanında boşluk olmalıdır.

Aynı sorun `-exec` ile de olur:

```bash
find /srv/arama -name "*.pdf" -o -name "*.tar" -exec ls -l {} \;
```

Sadece `arsiv.tar` listelendi, çünkü `-exec` sadece son koşula bağlandı. Parantezle ikisi de listelendi.

---

## 8. Bulunanlara komut uygulamak: -exec ve -delete

### -exec

```bash
find /srv/arama -type f -name "*.log" -size +100k -exec ls -lh {} \;
```

- `{}` bulunan dosyanın yoluyla değiştirilir.
- `\;` komutun bittiğini gösterir. `;` shell'e özel olduğu için kaçırılır.

İki bitiş biçimi var:

| Bitiş | Ne olur? |
|---|---|
| `-exec komut {} \;` | Komut **her dosya için ayrı ayrı** çalışır |
| `-exec komut {} +` | Komut dosyaların hepsiyle **bir kez** çalışır |

Denemede:

```
\; ile:
cagri: /srv/arama/loglar/orta.log
cagri: /srv/arama/loglar/buyuk.log
cagri: /srv/arama/loglar/bos.log
cagri: /srv/arama/loglar/uygulama.log

+ ile:
cagri: /srv/arama/loglar/orta.log /srv/arama/loglar/buyuk.log /srv/arama/loglar/bos.log /srv/arama/loglar/uygulama.log
```

Çok sayıda dosyada `+` çok daha hızlıdır. `{}`'nin komutun sonunda olması gereken durumlarda (`rm`, `chmod`, `ls` gibi) `+` kullanılabilir.

Örnekler:

```bash
find /srv/arama -name "*.log" -exec grep -l "ERROR" {} +   # içinde ERROR geçen loglar
find /srv/web -type f -exec chmod 644 {} +                 # bütün dosyalara 644
find /srv/web -type d -exec chmod 755 {} +                 # bütün dizinlere 755
find /srv/arama -name "*.tmp" -exec cp {} /yedek/ \;       # bulunanları kopyala
```

Her dosya için onay istemek için `-exec` yerine `-ok`:

```bash
find /srv/arama -name "*.TXT" -ok rm {} \;
```
```
< rm ... /srv/arama/raporlar/kasim.TXT > ?
```

### -delete

```bash
find /srv/arama -type f -empty -delete
```

Bulunanları siler. Denemede bütün boş dosyalar silindi.

> [!CAUTION]
> `-delete` ve `-exec rm` geri alınamaz. Önce aynı komutu **eylem olmadan** çalıştırıp listeye bak, emin olunca eylemi ekle. `-delete`'i koşullardan önce yazma: `find /srv -delete -name "*.tmp"` önce siler, sonra adına bakar, yani **her şeyi** siler.

---

## 9. Hatalar ve sembolik linkler

### Permission denied satırları

Normal bir kullanıcı erişemediği dizinlerde arama yaparken her biri için bir hata görür:

```
find: '/root': Permission denied
```

Bu satırlar sonuçların arasına karışır. Hataları gizlemek için:

```bash
find / -name "*.conf" 2>/dev/null
```

`2>` hata çıktısını yönlendirir ([Girdi/Çıktı Yönlendirme](10-yonlendirme.md)). Hatayı gizlemek, o dizinlerde arama yapıldığı anlamına gelmez. Bütün sistemde arama gerekiyorsa `sudo` kullan.

### Sembolik linkler

`find` varsayılan olarak sembolik linklerin içine girmez:

```bash
find /srv/arama/loglar-link -name "*.log"      # hiçbir şey bulmaz
find -L /srv/arama/loglar-link -name "*.log"   # linki takip eder
find /srv/arama/loglar-link/ -name "*.log"     # sondaki / da linki açar
```

---

## 10. Komut özeti

| Koşul / Eylem | Anlamı |
|---|---|
| `-name "*.log"` / `-iname` | Ad (büyük-küçük harfe duyarlı / değil) |
| `-path "*/2026/*"` | Yolda arama |
| `-type f` / `d` / `l` | Dosya / dizin / link |
| `-maxdepth N` / `-mindepth N` | Derinlik sınırı |
| `-size +100M` / `-size -1024k` | Boyut (`c`, `k`, `M`, `G`) |
| `-empty` | Boş dosya veya dizin |
| `-mmin -10` / `-mtime +6` | İçerik değişim zamanı |
| `-cmin` / `-ctime` | Meta veri değişim zamanı |
| `-amin` / `-atime` | Erişim zamanı |
| `-newer dosya` | Dosyadan yeni |
| `-perm 644` / `-644` / `/022` | Tam / en az / herhangi biri |
| `-user ad` / `-group ad` | Sahip / grup |
| `-nouser` / `-nogroup` | Sahibi / grubu olmayan |
| `! koşul` / `-not` | Değil |
| `koşul -o koşul` | Veya |
| `\( ... \)` | Gruplama |
| `-exec komut {} \;` | Her dosya için komut |
| `-exec komut {} +` | Hepsi için tek komut |
| `-ok komut {} \;` | Onay sorarak |
| `-delete` | Siler |
| `-L` | Sembolik linkleri takip et |
| `2>/dev/null` | Hataları gizle |

---

## 11. Sık yapılan hatalar

- **Deseni tırnaksız yazmak.** Shell genişletir, sonuç yanlış veya hata olur.
- **`-name`'in büyük-küçük harfe duyarlı olduğunu unutmak.** `-iname` kullan.
- **`-size -1M` ile "1 MB'tan küçük" aramak.** Yuvarlama yüzünden sadece boş dosyalar çıkar.
- **`-size` için büyük `K` yazmak.** Kilobayt küçük `k`.
- **`-mtime +1`'i "1 günden eski" sanmak.** 48 saatten eski demek.
- **`-o` ile parantez kullanmamak.** Diğer koşullar ve `-exec` sadece bir tarafa bağlanır.
- **`-type f` yazmayı unutmak.** Dizinler de sonuçlara karışır.
- **`-delete`'i test etmeden çalıştırmak** veya koşullardan önce yazmak.
- **`-perm 4000` yazmak.** Tam eşleşme arar, `-4000` kullan.
- **Hataları gizleyince aramanın tam olduğunu sanmak.** Erişilemeyen dizinler atlanır.

---

## 12. Alıştırmalar

**1.** `/etc` altında adında büyük-küçük harf fark etmeksizin `ssh` geçen bütün dosyaları bul. Sonra sadece adı `_config` ile bitenleri bul.

<details>
<summary>Çözüm</summary>

```bash
sudo find /etc -type f -iname "*ssh*"
sudo find /etc -type f -iname "*ssh*_config"
```

İkinci komut `/etc/ssh/sshd_config` ve `/etc/ssh/ssh_config` dosyalarını bulur.
</details>

**2.** `/var/log` altında 1 MB'tan büyük dosyaları boyutlarıyla birlikte listele.

<details>
<summary>Çözüm</summary>

```bash
sudo find /var/log -type f -size +1M -exec ls -lh {} +
```
</details>

**3.** `/etc` altında son 60 dakikada içeriği değişen dosyaları bul. Sonra izni veya sahibi son 60 dakikada değişenleri bul.

<details>
<summary>Çözüm</summary>

```bash
sudo find /etc -type f -mmin -60
sudo find /etc -type f -cmin -60
```
</details>

**4.** `/tmp` altında 7 günden uzun süredir değişmemiş ve sahibi sen olan dosyaları önce listele, sonra sil.

<details>
<summary>Çözüm</summary>

```bash
find /tmp -type f -user "$USER" -mtime +6
find /tmp -type f -user "$USER" -mtime +6 -delete
```

`-mtime +6` en az 7 tam gün demek. Önce listeleyip sonra `-delete` eklemek iyi alışkanlık.
</details>

**5.** `/srv/web` altında `.html` veya `.css` olan dosyaların izinlerini `644` yap. Komutun neden parantez istediğini açıkla.

<details>
<summary>Çözüm</summary>

```bash
find /srv/web -type f \( -name "*.html" -o -name "*.css" \) -exec chmod 644 {} +
```

Parantez olmadan `-type f` sadece `.html` koşuluna, `-exec` sadece `.css` koşuluna bağlanır. Sonuçta `.html` dosyalarının izni hiç değişmez.
</details>

---

Önceki konu: [İzinler ve Özel Bitler](04-izinler-ve-ozel-bitler.md) · Sonraki konu: [Metin İşleme](06-metin-isleme.md)
