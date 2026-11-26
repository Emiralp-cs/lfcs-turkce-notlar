# Dosya ve Dizin İşlemleri

Bu dosyada dizinler arasında gezinmeyi, dosya ve dizin oluşturmayı, kopyalamayı, taşımayı ve silmeyi ele alıyoruz. Komutlar basit görünse de her birinin sınavda ve gerçek hayatta sorun çıkaran birkaç davranışı var. Onlara özellikle dikkat edeceğiz.

Senaryo: Ekibin sunucusunda aylık raporlar tutuluyor. Rapor dizinini düzenleyecek, eski raporları arşivleyecek ve yedekleyeceğiz.

> [!NOTE]
> Bütün komutlar ve çıktılar Ubuntu 24.04 üzerinde denenmiştir.

## İçindekiler

1. [Dizin ağacı ve yollar](#1-dizin-ağacı-ve-yollar)
2. [Gezinmek: pwd ve cd](#2-gezinmek-pwd-ve-cd)
3. [Listelemek: ls](#3-listelemek-ls)
4. [Oluşturmak: touch ve mkdir](#4-oluşturmak-touch-ve-mkdir)
5. [Kopyalamak: cp](#5-kopyalamak-cp)
6. [Taşımak ve yeniden adlandırmak: mv](#6-taşımak-ve-yeniden-adlandırmak-mv)
7. [Silmek: rm ve rmdir](#7-silmek-rm-ve-rmdir)
8. [Özel karakterli dosya adları](#8-özel-karakterli-dosya-adları)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Dizin ağacı ve yollar

Linux'ta bütün dosyalar tek bir ağacın altındadır. Ağacın kökü `/`'dur. Diskler, USB bellekler ve ağ paylaşımları bile bu ağacın bir dalına bağlanır.

```
/
├── etc/          sistem yapılandırma dosyaları
├── home/         kullanıcıların ev dizinleri
│   └── ayse/
├── srv/          sunucunun sunduğu veriler
│   └── raporlar/
├── tmp/          geçici dosyalar
└── var/
    └── log/      log dosyaları
```

Bir dosyaya iki şekilde ulaşılır:

| | Mutlak yol | Göreli yol |
|---|---|---|
| Nasıl başlar? | `/` ile | Başka bir şeyle |
| Neye göre? | Kökten | Bulunduğun dizinden |
| Örnek | `/srv/raporlar/kasim.txt` | `raporlar/kasim.txt` (sen `/srv`'deyken) |
| Nerede çalışır? | Her yerde | Sadece doğru dizindeyken |

Yollarda kullanılan kısaltmalar:

| Sembol | Anlamı |
|---|---|
| `.` | Bulunduğun dizin |
| `..` | Bir üst dizin |
| `~` | Kendi ev dizinin (`/home/kullanici`) |
| `~ayse` | Ayşe'nin ev dizini |
| `-` | `cd -` ile kullanıldığında bir önceki dizin |

Örneğin `/home/ayse/belgeler` dizinindeyken `../indirilenler/fatura.pdf`, `/home/ayse/indirilenler/fatura.pdf` demektir.

> [!TIP]
> Script'lerde ve cron görevlerinde **mutlak yol** kullan. Script'in hangi dizinden çalıştırılacağı belli olmadığı için göreli yollar yanlış yeri gösterebilir.

---

## 2. Gezinmek: pwd ve cd

```bash
pwd              # neredeyim?
cd /var/log      # mutlak yolla git
cd raporlar      # göreli yolla git
cd ..            # bir üst dizine
cd               # ev dizinine (cd ~ ile aynı)
cd -             # bir önceki dizine dön
```

`cd -`, gittiği dizini ekrana da yazar. İki dizin arasında gidip gelirken çok kullanışlıdır:

```bash
cd /tmp/lab/proje
cd /var/log
cd -
```
```
/tmp/lab/proje
```

---

## 3. Listelemek: ls

```bash
ls               # bulunduğun dizin
ls /var/log      # belirli bir dizin
```

Sık kullanılan seçenekler:

| Seçenek | Ne yapar? |
|---|---|
| `-l` | Uzun biçim: izinler, sahip, boyut, tarih |
| `-a` | `.` ile başlayan gizli dosyaları da göster |
| `-A` | Gizlileri göster ama `.` ve `..`'yı gösterme |
| `-h` | Boyutları okunabilir yaz (`4.0K`, `2M`), `-l` ile birlikte |
| `-t` | Değiştirilme zamanına göre sırala, en yeni üstte |
| `-r` | Sıralamayı ters çevir |
| `-S` | Boyuta göre sırala, en büyük üstte |
| `-R` | Alt dizinleri de listele |
| `-d` | Dizinin içini değil, dizinin kendisini göster |

Seçenekler birleştirilebilir:

```bash
ls -lah          # her şey, okunabilir boyutla
ls -ltr          # en eski üstte, en yeni en altta (son değişen dosyayı bulmak için)
ls -ld /srv/raporlar   # dizinin kendi izinleri
```

`ls -l /srv/raporlar` dizinin **içindekileri** listeler. Dizinin kendi izinlerini görmek için `-d` gerekir.

### ls -l çıktısını okumak

```
drwxr-xr-x 3 root root 4096 Nov 25 15:31 raporlar
-rw-r--r-- 1 root root    5 Nov 25 15:31 notlar.txt
```

| Sütun | Örnek | Anlamı |
|---|---|---|
| 1. karakter | `d` / `-` / `l` | Tür: dizin, normal dosya, sembolik link |
| Sonraki 9 karakter | `rwxr-xr-x` | Sahip, grup ve diğerlerinin izinleri ([İzinler](04-izinler-ve-ozel-bitler.md)) |
| Sayı | `3` | Bağlantı (hard link) sayısı ([Linkler](03-hard-ve-soft-link.md)) |
| Sahip | `root` | Dosyanın sahibi |
| Grup | `root` | Dosyanın grubu |
| Boyut | `4096` | Bayt (`-h` ile okunabilir) |
| Tarih | `Nov 25 15:31` | Son değiştirilme zamanı |
| Ad | `raporlar` | Dosya veya dizin adı |

Bir dizin için boyut sütunu, içindeki dosyaların toplam boyutu **değildir**, dizin kaydının kendi boyutudur (genelde 4096). Bir dizinin içindekilerle birlikte ne kadar yer kapladığını görmek için `du -sh dizin` kullanılır ([Disk Alanı](15-disk-alani-sorunlari.md)).

---

## 4. Oluşturmak: touch ve mkdir

### touch

```bash
touch kasim.txt
touch /srv/raporlar/kasim.txt aralik.txt ocak.txt
```

Dosya yoksa boş bir dosya oluşturur. Dosya **varsa içeriğine dokunmaz**, sadece değiştirilme zamanını şimdiki zaman yapar. Denemede içinde "veri" yazan bir dosyaya `touch` uyguladık, içerik aynı kaldı, saat güncellendi:

```
-rw-r--r-- 1 root root 0 15:31:12 notlar.txt
(1 saniye sonra touch)
-rw-r--r-- 1 root root 0 15:31:13 notlar.txt
```

Belirli bir tarih vermek için `touch -d '2026-01-15 10:00' dosya`.

### mkdir

```bash
mkdir raporlar
mkdir ocak subat mart
```

İç içe dizinler oluştururken ara dizinler yoksa hata alırsın:

```bash
mkdir raporlar/2026/kasim
```
```
mkdir: cannot create directory 'raporlar/2026/kasim': No such file or directory
```

`-p` (parents), eksik ara dizinleri de oluşturur:

```bash
mkdir -p raporlar/2026/kasim
```

`-p`'nin bir faydası daha var: dizin zaten varsa hata vermez. Bu yüzden script'lerde `mkdir -p` tercih edilir.

---

## 5. Kopyalamak: cp

```bash
cp kaynak hedef
```

| Komut | Sonuç |
|---|---|
| `cp kasim.txt kasim-yedek.txt` | Aynı dizinde, farklı adla kopya |
| `cp kasim.txt arsiv/` | `arsiv` dizinine aynı adla kopya |
| `cp kasim.txt arsiv/2026-kasim.txt` | Başka dizine farklı adla kopya |
| `cp ocak.txt subat.txt arsiv/` | Birden fazla dosyayı bir dizine kopyala (son argüman hedef dizin) |

### Dizin kopyalamak: -r

```bash
cp raporlar yedek
```
```
cp: -r not specified; omitting directory 'raporlar'
```

Dizin kopyalamak için `-r` (recursive) gerekir:

```bash
cp -r raporlar yedek
```

> [!WARNING]
> `cp -r` hedefin var olup olmamasına göre farklı davranır:
> - `yedek` **yoksa**: `raporlar` dizininin bir kopyası `yedek` adıyla oluşur. İçerik `yedek/2026/...` olur.
> - `yedek` **varsa**: `raporlar` dizini, `yedek`'in **içine** kopyalanır. İçerik `yedek/raporlar/2026/...` olur.
>
> Denemede aynı komutu iki kez çalıştırdık. İkinci seferde `yedek/raporlar` diye iç içe bir dizin oluştu. Hedefin önceden var olup olmadığını kontrol et.

### İzinleri ve zamanları korumak: -p ve -a

Normal `cp`, kopyanın sahibini kopyalayan kişi yapar ve tarihini şimdiki zaman yapar. Ayşe'ye ait, `640` izinli, Ocak tarihli bir dosyayı root olarak kopyaladık:

```
-rw-r----- 1 ayse ayse 7 Jan 15  sahip.txt        (orijinal)
-rw-r----- 1 root root 7 Nov 25  kopya-root.txt   (cp)
-rw-r----- 1 ayse ayse 7 Jan 15  kopya-p.txt      (cp -p)
-rw-r----- 1 ayse ayse 7 Jan 15  kopya-a.txt      (cp -a)
```

| Seçenek | Korunanlar |
|---|---|
| `-p` | İzinler, sahiplik, zaman bilgisi |
| `-a` (archive) | Hepsi: `-p` + alt dizinler (`-r`) + sembolik linkler + ACL gibi ek özellikler |

Yedek alırken `cp -a kaynak hedef` kullan. Sahipliğin korunabilmesi için kopyalayanın root olması gerekir. Normal bir kullanıcı başkasına ait bir kopya oluşturamaz.

### Üzerine yazma

`cp`, hedefte aynı adlı dosya varsa **sormadan üzerine yazar**:

```bash
echo eski > hedef.txt
echo yeni > kaynak.txt
cp kaynak.txt hedef.txt
cat hedef.txt
```
```
yeni
```

Sormasını istiyorsan `-i`, hiç üzerine yazmamasını istiyorsan `-n` kullan:

```bash
cp -i kaynak.txt hedef.txt
```
```
cp: overwrite 'hedef.txt'?
```

---

## 6. Taşımak ve yeniden adlandırmak: mv

Linux'ta ayrı bir "yeniden adlandır" komutu yoktur. Yeniden adlandırmak, aynı dizin içinde taşımaktır:

```bash
mv kasim.txt kasim-2026.txt          # yeniden adlandır
mv kasim-2026.txt arsiv/             # taşı
mv kasim.txt arsiv/2026-kasim.txt    # taşı ve adını değiştir
mv ocak.txt subat.txt arsiv/         # birden fazla dosyayı taşı
mv raporlar arsiv                    # dizini yeniden adlandır
```

Dizinler için `-r` **gerekmez**. `mv` dizini olduğu gibi yerinden alır, içindekileri tek tek kopyalamaz. Aynı dosya sistemi içinde taşıma bu yüzden boyuttan bağımsız olarak anlık gerçekleşir.

`cp` gibi `mv` de hedefte aynı adlı dosya varsa **sormadan üzerine yazar**:

```bash
echo 1 > x.txt
echo 2 > y.txt
mv x.txt y.txt
cat y.txt
```
```
1
```

`y.txt`'nin eski içeriği gitti. `mv -i` sorar, `mv -n` hiç üzerine yazmaz:

```
mv: not replacing 'y.txt'
```

---

## 7. Silmek: rm ve rmdir

```bash
rm kasim.txt
rm ocak.txt subat.txt
```

> [!CAUTION]
> Terminalde silinen dosya **çöp kutusuna gitmez**, doğrudan silinir. Geri getirmenin kolay bir yolu yoktur.

### Dizinleri silmek

`rm` dizinleri tek başına silmez:

```
rm: cannot remove 'arsiv': Is a directory
```

İki yol var:

| Komut | Ne zaman? |
|---|---|
| `rmdir dizin` | Sadece **boş** dizinler. Doluysa `Directory not empty` hatası verir |
| `rm -r dizin` | Dizini ve içindeki her şeyi siler |

`rmdir`'in boş olmayan dizinde hata vermesi bir güvenlik özelliği: içinde bir şey kaldığını fark etmeni sağlar.

```bash
rm -i dosya.txt      # silmeden önce sor
rm -ri arsiv         # her dosya için sor
rm -rf gecici/       # sormadan, hata vermeden zorla sil
```

> [!CAUTION]
> `rm -rf` ile yazım hatası çok pahalıya patlar. `rm -rf / tmp/eski` (fazladan bir boşluk) kök dizini silmeye çalışır. Silmeden önce:
> - Önce aynı yolu `ls` ile listele ve gerçekten silmek istediğin şey mi bak.
> - Değişkenli yollarda dikkatli ol: `rm -rf "$DIZIN/"*` komutunda `$DIZIN` boşsa komut `rm -rf /*` olur.

---

## 8. Özel karakterli dosya adları

### Boşluk içeren adlar

Boşluk, shell için argüman ayıracıdır:

```bash
rm dosya adi bosluklu.txt
```
```
rm: cannot remove 'dosya': No such file or directory
rm: cannot remove 'adi': No such file or directory
rm: cannot remove 'bosluklu.txt': No such file or directory
```

`rm` üç ayrı dosya silmeye çalıştı. Adı tırnak içine al veya boşluğu `\` ile kaçır:

```bash
rm "dosya adi bosluklu.txt"
rm dosya\ adi\ bosluklu.txt
```

Tab tamamlama bunu kendisi yapar.

### - ile başlayan adlar

```bash
rm -tehlikeli
```
```
rm: invalid option -- 't'
Try 'rm ./-tehlikeli' to remove the file '-tehlikeli'.
```

Komut, adı bir seçenek sandı. İki çözüm var:

```bash
rm ./-tehlikeli      # başına yol ekle
rm -- -tehlikeli     # "--" sonrası her şey dosya adıdır
```

`--` kuralı neredeyse bütün komutlarda geçerlidir.

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `pwd` | Bulunduğun dizini gösterir |
| `cd dizin` / `cd ..` / `cd` / `cd -` | Dizine git / üste / eve / öncekine |
| `ls -la` | Gizliler dahil uzun liste |
| `ls -lh` | Okunabilir boyutlarla |
| `ls -ltr` | En yeni en altta |
| `ls -ld dizin` | Dizinin kendisini gösterir |
| `touch dosya` | Boş dosya oluşturur veya tarihini günceller |
| `mkdir -p a/b/c` | İç içe dizin oluşturur |
| `cp kaynak hedef` | Dosya kopyalar |
| `cp -r kaynak hedef` | Dizin kopyalar |
| `cp -a kaynak hedef` | Her şeyi koruyarak kopyalar |
| `cp -i` / `cp -n` | Üzerine yazmadan önce sor / hiç yazma |
| `mv kaynak hedef` | Taşır veya yeniden adlandırır |
| `mv -i` / `mv -n` | Üzerine yazmadan önce sor / hiç yazma |
| `rm dosya` | Dosya siler |
| `rm -r dizin` | Dizini içindekilerle siler |
| `rm -i` | Silmeden önce sorar |
| `rmdir dizin` | Boş dizini siler |
| `rm -- -ad` / `rm ./-ad` | `-` ile başlayan dosyayı siler |

---

## 10. Sık yapılan hatalar

- **`cp` ve `mv`'nin sormadan üzerine yazdığını unutmak.** Önemli dosyalarda `-i` veya `-n` kullan.
- **`cp -r`'de hedefin var olup olmamasına dikkat etmemek.** Varsa kaynak hedefin içine kopyalanır.
- **Yedek alırken düz `cp` kullanmak.** Sahiplik ve tarihler kaybolur, `cp -a` kullan.
- **`mv` için `-r` aramak.** Gerekmez.
- **Silinen dosyanın çöp kutusunda olduğunu sanmak.**
- **`rm -rf`'yi kontrol etmeden çalıştırmak.** Önce `ls` ile aynı yolu listele.
- **Ara dizinler yokken `-p`'siz `mkdir`.** Hata verir.
- **Dizin izinlerine bakarken `-d`'yi unutmak.** `ls -l dizin` içindekileri listeler.
- **Boşluklu dosya adlarını tırnaksız yazmak.**

---

## 11. Alıştırmalar

Alıştırmaları `/tmp/alistirma` gibi deneme amaçlı bir dizinde yap.

**1.** `/tmp/alistirma/raporlar/2026/ocak` dizin yapısını tek komutla oluştur. İçine `butce.txt` ve `plan.txt` adlı iki boş dosya oluştur.

<details>
<summary>Çözüm</summary>

```bash
mkdir -p /tmp/alistirma/raporlar/2026/ocak
touch /tmp/alistirma/raporlar/2026/ocak/butce.txt /tmp/alistirma/raporlar/2026/ocak/plan.txt
ls -R /tmp/alistirma
```
</details>

**2.** `raporlar` dizininin tamamını, izinleri ve tarihleri koruyarak `/tmp/alistirma/yedek` adıyla kopyala (`yedek` henüz yok). Aynı komutu bir kez daha çalıştırırsan ne olur?

<details>
<summary>Çözüm</summary>

```bash
cd /tmp/alistirma
cp -a raporlar yedek
ls yedek        # 2026
cp -a raporlar yedek
ls yedek        # 2026 raporlar
```

İkinci seferde `yedek` var olduğu için `raporlar`, `yedek`'in içine kopyalanır.
</details>

**3.** `plan.txt`'nin adını `plan-v1.txt` yap ve `/tmp/alistirma` dizinine taşı. Tek komut kullan.

<details>
<summary>Çözüm</summary>

```bash
mv /tmp/alistirma/raporlar/2026/ocak/plan.txt /tmp/alistirma/plan-v1.txt
```
</details>

**4.** `/tmp/alistirma` içinde `-v2.txt` adlı bir dosya oluştur ve sonra sil.

<details>
<summary>Çözüm</summary>

```bash
cd /tmp/alistirma
touch -- -v2.txt
ls
rm -- -v2.txt
```

`touch -v2.txt` de hata verir, çünkü `touch` de adı seçenek sanar.
</details>

**5.** `raporlar/2026/ocak` dizinini önce `rmdir` ile silmeyi dene. Neden olmadı? Sonra bütün alıştırma dizinini sil.

<details>
<summary>Çözüm</summary>

```bash
rmdir /tmp/alistirma/raporlar/2026/ocak
# rmdir: failed to remove ...: Directory not empty   (içinde butce.txt var)
ls /tmp/alistirma
rm -r /tmp/alistirma
```
</details>

---

Önceki konu: [Yardım Almak](01-yardim-almak.md) · Sonraki konu: [Hard ve Soft Link](03-hard-ve-soft-link.md)
