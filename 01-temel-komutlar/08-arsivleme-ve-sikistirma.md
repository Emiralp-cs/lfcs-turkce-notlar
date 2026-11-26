# Arşivleme ve Sıkıştırma

**Arşivlemek**, çok sayıda dosya ve dizini tek bir dosyada toplamaktır. **Sıkıştırmak** ise bir dosyanın kapladığı yeri küçültmektir. Linux'ta bu iki iş genelde ayrı araçlarla yapılır: `tar` arşivler, `gzip`, `bzip2` ve `xz` sıkıştırır. `.tar.gz` gibi uzantılar ikisinin birlikte kullanıldığını gösterir.

Senaryo: Ekibin web sitesi `site/` dizininde duruyor. Güncellemeden önce siteyi tek bir dosyada yedekleyeceğiz, yedeği küçülteceğiz, gerektiğinde içinden tek bir dosyayı geri alacağız.

```
site/
├── index.html
├── kayit.log
├── css/
│   └── stil.css
└── img/
    └── logo.png
```

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve GNU tar 1.35 ile denenmiştir. Boyutlar dosya içeriğine göre değişir, karşılaştırma için verilmiştir.

## İçindekiler

1. [tar nedir?](#1-tar-nedir)
2. [Arşiv oluşturmak: -c](#2-arşiv-oluşturmak--c)
3. [Arşivin içeriğini listelemek: -t](#3-arşivin-içeriğini-listelemek--t)
4. [Arşivden çıkarmak: -x](#4-arşivden-çıkarmak--x)
5. [Arşive dosya eklemek: -r](#5-arşive-dosya-eklemek--r)
6. [-f seçeneğinin yeri](#6--f-seçeneğinin-yeri)
7. [Mutlak yollar ve -C](#7-mutlak-yollar-ve--c)
8. [Sahiplik ve izinler](#8-sahiplik-ve-izinler)
9. [Tek başına sıkıştırma: gzip, bzip2, xz](#9-tek-başına-sıkıştırma-gzip-bzip2-xz)
10. [tar ile sıkıştırma: -z, -j, -J, -a](#10-tar-ile-sıkıştırma--z--j--j--a)
11. [zip ve unzip](#11-zip-ve-unzip)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. tar nedir?

`tar` (tape archive), adını manyetik teyplere yedek almak için kullanılmasından alır. Bugün dosyaları paketlemek, sunucular arasında taşımak ve yedeklemek için kullanılıyor.

Tek başına `tar` **sıkıştırmaz**. Dosyaları uç uca ekleyip her birinin adını, izinlerini, sahibini ve tarihini kaydeder. Bizim denememizde `site/` dizininden çıkan `.tar` dosyası, içindeki dosyaların toplamından bile biraz büyüktü.

Her `tar` komutu bir **işlem** seçeneği ister:

| Kısa | Uzun | İşlem |
|---|---|---|
| `c` | `--create` | Yeni arşiv oluştur |
| `t` | `--list` | İçeriği listele |
| `x` | `--extract` | Arşivden çıkar |
| `r` | `--append` | Var olan arşivin sonuna ekle |

Yanına genelde şunlar eklenir:

| Kısa | Uzun | Anlamı |
|---|---|---|
| `f` | `--file` | Arşiv dosyasının adı |
| `v` | `--verbose` | İşlenen dosyaları ekrana yaz |
| `C` | `--directory` | Bu dizine geçip öyle çalış |

---

## 2. Arşiv oluşturmak: -c

```bash
tar --create --file yedek.tar site/   # uzun hali
tar -cf yedek.tar site/               # kısa hali
tar cf yedek.tar site/                # tiresiz eski yazım, aynı sonuç
```

`-v` eklersen arşive giren her dosyayı yazar:

```bash
tar -cvf yedek.tar site/
```
```
site/
site/img/
site/img/logo.png
site/index.html
site/kayit.log
site/css/
site/css/stil.css
```

Birden fazla dosya ve dizin aynı anda verilebilir:

```bash
tar -cf yedek.tar site/ notlar.txt /etc/hosts
```

### Bazı dosyaları dışarıda bırakmak

Log dosyalarını yedeğe almak istemiyorsan:

```bash
tar -cf yedek.tar --exclude='*.log' site/
```

`kayit.log` arşive girmez.

---

## 3. Arşivin içeriğini listelemek: -t

Bir arşivi açmadan önce içinde ne olduğuna bakmak iyi bir alışkanlıktır. Özellikle arşiv nereye açılacak, bir dizin mi oluşturacak yoksa dosyaları bulunduğun yere mi saçacak, bunu önceden görürsün.

```bash
tar --list --file yedek.tar
tar -tf yedek.tar
tar tf yedek.tar
```

Üçü de aynı listeyi verir. `-v` ile izinler, sahip, boyut ve tarih de görünür:

```bash
tar -tvf yedek.tar
```
```
drwxr-xr-x root/root         0 2026-11-25 15:59 site/
drwxr-xr-x root/root         0 2026-11-25 15:59 site/img/
-rw-r--r-- root/root      2000 2026-11-25 15:59 site/img/logo.png
...
```

### Arşivi diskle karşılaştırmak: -d

Yedeği aldıktan sonra bir dosya değiştiyse `-d` (`--diff`) bunu gösterir:

```bash
echo "yeni satir" >> site/index.html
tar -df yedek.tar
```
```
site/index.html: Mod time differs
site/index.html: Size differs
```

Bir şey değişmediyse hiçbir çıktı vermez.

---

## 4. Arşivden çıkarmak: -x

```bash
tar --extract --file yedek.tar
tar -xf yedek.tar
```

Arşiv **bulunduğun dizine** açılır. Başka bir dizine açmak için `-C`:

```bash
mkdir /tmp/geri
tar -xf yedek.tar -C /tmp/geri
```

Sonuç `/tmp/geri/site/index.html`, `/tmp/geri/site/css/stil.css` gibi olur. Arşivdeki yollar korunur, `-C` sadece başlangıç noktasını değiştirir.

> [!WARNING]
> `-C` ile verilen dizin önceden var olmalı. Yoksa `tar` `Cannot open: No such file or directory` hatası verir ve çıkar. Önce `mkdir -p` ile oluştur.

### Tek bir dosyayı geri almak

Bütün arşivi açmak zorunda değilsin. Dosyanın yolunu, `tar -tf` çıktısında göründüğü gibi yaz:

```bash
tar -xf yedek.tar site/css/stil.css
```

Sadece `site/css/stil.css` (ve onu taşıyan dizinler) oluşturulur.

### Var olan dosyaların üzerine yazma

`tar -x`, aynı adda dosya varsa **sormadan üzerine yazar**. Bunu denemede gördük: değiştirdiğimiz `index.html`, arşivdeki eski haliyle değişti. Var olan dosyalara dokunmamasını istiyorsan `-k` (`--keep-old-files`):

```bash
tar -xkf yedek.tar
```

Bu durumda var olan her dosya için `Cannot open: File exists` uyarısı verir ve onları atlar.

---

## 5. Arşive dosya eklemek: -r

```bash
tar --append --file yedek.tar notlar.txt
tar -rf yedek.tar notlar.txt
```

Dosya arşivin sonuna eklenir. Aynı dosyayı iki kez eklersen arşivde **iki kopya** olur. Denemede `tar -tf yedek.tar | grep -c notlar` sonucu `2` çıktı. Çıkarırken sondaki kopya öncekinin üzerine yazar.

> [!CAUTION]
> `-r` sadece sıkıştırılmamış `.tar` dosyalarında çalışır. `.tar.gz` gibi bir dosyaya eklemeye çalışınca `tar: Cannot update compressed archives` hatası alırsın. Sıkıştırılmış arşivi güncellemek için açıp yeniden oluşturman gerekir.

---

## 6. -f seçeneğinin yeri

`-f`, arşiv dosyasının adını **argüman olarak alır**. Tireli yazımda (`-cvf`), seçeneklerin hemen arkasından gelen kelime `f`'nin argümanıdır, bu yüzden `f` sonda olmalı:

```bash
tar -cvf yedek.tar site/index.html    # doğru: arşivin adı yedek.tar
tar -cfv yedek.tar site/index.html    # yanlış
```

İkinci komutta denemede şu oldu: `f`'den sonraki `v` arşivin adı sanıldı, `v` adında bir arşiv oluştu ve `yedek.tar` da arşive konacak dosya sanıldığı için `Cannot stat: No such file or directory` hatası verdi.

Tiresiz eski yazımda (`tar cfv yedek.tar ...`) kural farklı, argümanlar sırayla dağıtıldığı için orada çalışıyor. Karışıklık olmasın diye en kolayı: **`f`'yi her zaman en sona koy**, yani `-cvf`, `-xvf`, `-tvf`.

---

## 7. Mutlak yollar ve -C

### Baştaki / kaldırılır

`/` ile başlayan bir yolu arşivlediğinde:

```bash
tar -cf mutlak.tar /tmp/deneme/site
```
```
tar: Removing leading `/' from member names
```
```bash
tar -tf mutlak.tar
```
```
tmp/deneme/site/
tmp/deneme/site/img/
...
```

`tar`, güvenlik için baştaki `/`'yi siler. Böylece arşivi açtığında dosyalar sistemdeki asıl yerlerinin üzerine değil, bulunduğun dizinin altına (`./tmp/deneme/site/...`) açılır. Bu davranışı kapatmak için `-P` (`--absolute-names`) var, ama arşivi açarken gerçek sistem dosyalarının üzerine yazabileceği için dikkatli kullanılmalı.

### Arşivde gereksiz üst dizinler olmasın

`/var/www/site`'yi doğrudan arşivlersen arşivde `var/www/site/...` yolları olur. Sadece `site/` ile başlamasını istiyorsan, arşivlerken de `-C` kullan:

```bash
tar -czf site.tar.gz -C /var/www site
```

`tar` önce `/var/www`'ye geçer, oradan `site`'yi arşivler. Arşivdeki yollar `site/...` olur.

---

## 8. Sahiplik ve izinler

`tar` her dosyanın izinlerini ve sahibini (UID/GID) arşive kaydeder. Açarken ne olacağı **kimin açtığına** bağlıdır. Sahibi UID 1234 olan, izinleri `777` olan bir dizini ve içindeki `600` izinli dosyayı arşivleyip açtık:

| Kim açtı? | Sahip | Dizin izni |
|---|---|---|
| Normal kullanıcı, `tar -xf` | Açan kullanıcı | `755` (umask uygulandı) |
| Normal kullanıcı, `tar -xpf` | Açan kullanıcı | `777` (arşivdeki izin korundu) |
| root, `tar -xf` | `1234` (arşivdeki sahip korundu) | `777` |
| root, `tar -xf --no-same-owner` | root | `777` |

Özetle:

- **Sahipliği** sadece root geri yükleyebilir. Normal kullanıcı hangi arşivi açarsa açsın dosyalar onun olur.
- **İzinler**, root'ta olduğu gibi korunur. Normal kullanıcıda umask uygulanır, aynen korunması için `-p` (`--preserve-permissions`) gerekir.

Bu yüzden sistem dosyalarının yedeğini geri yüklerken `sudo tar -xf ...` kullanılır.

---

## 9. Tek başına sıkıştırma: gzip, bzip2, xz

Üç yaygın sıkıştırma aracı, aynı mantıkla çalışır:

| Araç | Uzantı | Sıkıştır | Aç | Açmadan oku |
|---|---|---|---|---|
| gzip | `.gz` | `gzip dosya` | `gunzip dosya.gz` veya `gzip -d` | `zcat` |
| bzip2 | `.bz2` | `bzip2 dosya` | `bunzip2 dosya.bz2` veya `bzip2 -d` | `bzcat` |
| xz | `.xz` | `xz dosya` | `unxz dosya.xz` veya `xz -d` | `xzcat` |

`kayit.log` (108.894 bayt, 1'den 20.000'e kadar sayılar) üzerinde denedik:

| Araç | Sonuç boyutu |
|---|---|
| gzip | 45.010 bayt |
| bzip2 | 25.147 bayt |
| xz | 4.948 bayt |

Genel kural: gzip en hızlısı ama en az küçülteni, xz en çok küçülteni ama en yavaşı, bzip2 ikisinin arasında. Oranlar dosyanın içeriğine göre çok değişir.

### Orijinal dosya silinir

```bash
gzip kayit.log
ls
```
```
kayit.log.gz
```

`gzip`, `bzip2` ve `xz` varsayılan olarak orijinali **siler**, yerine sıkıştırılmış halini bırakır. Açınca da tersi olur. Orijinali korumak için `-k` (`--keep`):

```bash
gzip -k kayit.log      # kayit.log ve kayit.log.gz ikisi de kalır
```

Diğer işe yarar seçenekler:

```bash
gzip -l kayit.log.gz          # sıkıştırma oranını göster
zcat kayit.log.gz | tail -1   # açmadan içine bak
gzip -c kayit.log > yedek.gz  # sonucu ekrana (stdout) yaz, orijinal yerinde kalır
```

`gzip -l` çıktısı:

```
         compressed        uncompressed  ratio uncompressed_name
              45010              108894  58.7% kayit.log
```

> [!IMPORTANT]
> Bu araçlar **tek bir dosyayı** sıkıştırır. `gzip site` denediğimizde `gzip: site is a directory -- ignored` dedi. Bir dizini sıkıştırmak için önce `tar` ile tek dosyaya çevirmek gerekir. Bir sonraki bölüm bunu tek komutta yapıyor.

---

## 10. tar ile sıkıştırma: -z, -j, -J, -a

`tar`'a bir seçenek ekleyerek arşivleme ve sıkıştırmayı tek adımda yapabilirsin:

| Seçenek | Sıkıştırıcı | Yaygın uzantı |
|---|---|---|
| `-z` | gzip | `.tar.gz` veya `.tgz` |
| `-j` | bzip2 | `.tar.bz2` |
| `-J` | xz | `.tar.xz` |
| `-a` | Uzantıya bakıp kendisi seçer | Hepsi |

```bash
tar -czf site.tar.gz site/
tar -cjf site.tar.bz2 site/
tar -cJf site.tar.xz site/
tar -caf site.tar.xz site/     # uzantı .xz olduğu için xz kullanır
```

Denemedeki boyutlar:

```
122880  yedek.tar        (sıkıştırmasız)
 48648  site.tar.gz
 29603  site.tar.bz2
  8132  site.tar.xz
```

`-a` ile oluşturduğumuz dosya, `-J` ile oluşturduğumuzla aynı boyutta çıktı. `file` komutu ikisini de `XZ compressed data` olarak gösterdi.

### Açarken sıkıştırmayı belirtmek gerekmez

GNU tar okurken dosyanın türünü kendisi anlar:

```bash
tar -xf site.tar.bz2      # -j yazmaya gerek yok
tar -tf site.tar.xz       # -J yazmaya gerek yok
```

Yanlış seçeneği zorla verirsen hata alırsın:

```bash
tar -tjf site.tar.gz
```
```
bzip2: (stdin) is not a bzip2 file.
tar: Child returned status 2
```

Kısacası oluştururken `-z`/`-j`/`-J` (veya `-a`) ver, açarken ve listelerken `-xf` ve `-tf` yeterli.

---

## 11. zip ve unzip

Windows kullanıcılarıyla dosya paylaşırken `.zip` daha yaygındır. `zip` hem arşivler hem sıkıştırır:

```bash
zip -r site.zip site/          # -r: dizini alt dizinleriyle birlikte
unzip -l site.zip              # içeriği listele
unzip site.zip -d /tmp/geri    # belirli bir dizine aç
```

Ubuntu'nun minimal kurulumlarında `zip` ve `unzip` yüklü olmayabilir: `sudo apt install zip unzip`.

---

## 12. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `tar -cvf arsiv.tar dizin/` | Arşiv oluştur |
| `tar -czf arsiv.tar.gz dizin/` | gzip ile sıkıştırılmış arşiv |
| `tar -cjf arsiv.tar.bz2 dizin/` | bzip2 ile |
| `tar -cJf arsiv.tar.xz dizin/` | xz ile |
| `tar -caf arsiv.tar.xz dizin/` | Uzantıya göre otomatik |
| `tar -cf arsiv.tar --exclude='*.log' dizin/` | Bazı dosyaları hariç tut |
| `tar -czf arsiv.tar.gz -C /ust/dizin hedef` | Üst dizinler arşive girmesin |
| `tar -tf arsiv.tar` | İçeriği listele |
| `tar -tvf arsiv.tar` | Ayrıntılı listele |
| `tar -xf arsiv.tar` | Bulunduğun dizine aç |
| `tar -xf arsiv.tar -C /hedef` | Başka dizine aç |
| `tar -xf arsiv.tar yol/dosya` | Tek dosyayı çıkar |
| `tar -xkf arsiv.tar` | Var olanların üzerine yazma |
| `tar -xpf arsiv.tar` | İzinleri aynen koru |
| `tar -rf arsiv.tar dosya` | Sona ekle (sıkıştırılmamışta) |
| `tar -df arsiv.tar` | Arşivi diskle karşılaştır |
| `gzip` / `bzip2` / `xz dosya` | Sıkıştır (orijinal silinir) |
| `gunzip` / `bunzip2` / `unxz` | Aç |
| `-k` | Orijinali koru |
| `zcat` / `bzcat` / `xzcat` | Açmadan oku |
| `gzip -l dosya.gz` | Sıkıştırma oranı |
| `zip -r a.zip dizin/` / `unzip a.zip` | zip oluştur / aç |

---

## 13. Sık yapılan hatalar

- **`tar`'ın tek başına sıkıştırdığını sanmak.** `.tar` sıkıştırılmamıştır, `-z`, `-j` veya `-J` gerekir.
- **`-f`'yi ortaya koymak.** `-cfv arsiv.tar` ile `v` adında bir arşiv oluşur. `f` en sonda olsun.
- **Arşivi listelemeden bulunduğun dizine açmak.** Dosyalar bir alt dizin yerine doğrudan bulunduğun yere dağılabilir veya var olan dosyaların üzerine yazılabilir.
- **`-C` dizinini önceden oluşturmamak.**
- **`gzip`'in orijinali sildiğini unutmak.** Lazımsa `-k`.
- **`gzip` ile dizin sıkıştırmaya çalışmak.** Önce `tar`.
- **Sıkıştırılmış arşive `-r` ile eklemeye çalışmak.**
- **Sistem dosyalarını normal kullanıcıyla geri yüklemek.** Sahiplikler kaybolur, `sudo` kullan.

---

## 14. Alıştırmalar

Alıştırmalar için bir `site/` dizini oluştur:

```bash
mkdir -p site/css site/img
echo '<h1>Merhaba</h1>' > site/index.html
echo 'body{}' > site/css/stil.css
seq 1 20000 > site/kayit.log
```

**1.** `site/` dizinini xz ile sıkıştırılmış `site-yedek.tar.xz` dosyasına arşivle. Log dosyaları arşive girmesin. Sonra arşivin içeriğini listele.

<details>
<summary>Çözüm</summary>

```bash
tar -cJf site-yedek.tar.xz --exclude='*.log' site/
tar -tf site-yedek.tar.xz
```

Listede `kayit.log` olmamalı.
</details>

**2.** `site/css/stil.css` dosyasını yanlışlıkla sildin. 1. alıştırmadaki arşivden sadece bu dosyayı geri al.

<details>
<summary>Çözüm</summary>

```bash
rm site/css/stil.css
tar -xf site-yedek.tar.xz site/css/stil.css
cat site/css/stil.css
```

Açarken `-J` yazmak gerekmez. Dosya yolu, `tar -tf` çıktısındakiyle aynı olmalı.
</details>

**3.** Arşivin tamamını `/tmp/kontrol` dizinine aç. Dizin henüz yok.

<details>
<summary>Çözüm</summary>

```bash
mkdir -p /tmp/kontrol
tar -xf site-yedek.tar.xz -C /tmp/kontrol
ls /tmp/kontrol/site
```
</details>

**4.** `site/kayit.log` dosyasını gzip ile sıkıştır ama orijinali silinmesin. Sonra sıkıştırma oranına bak ve dosyayı açmadan son satırını göster.

<details>
<summary>Çözüm</summary>

```bash
gzip -k site/kayit.log
gzip -l site/kayit.log.gz
zcat site/kayit.log.gz | tail -1
```

Son satır `20000` olmalı.
</details>

**5.** `$(pwd)/site` gibi mutlak bir yol vererek, ama arşivdeki yollar `site/` ile başlayacak şekilde `site.tar.gz` oluştur.

<details>
<summary>Çözüm</summary>

```bash
tar -czf site.tar.gz -C "$(pwd)" site
tar -tf site.tar.gz | head -3
```
```
site/
site/img/
site/index.html
```

`-C` olmadan `tar -czf site.tar.gz "$(pwd)/site"` yazsaydın, arşivdeki yollar `home/kullanici/site/...` gibi olurdu.
</details>

---

Önceki konu: [grep ve Düzenli İfadeler](07-grep-ve-regex.md) · Sonraki konu: [Yedekleme: rsync ve dd](09-yedekleme-rsync-ve-dd.md)
