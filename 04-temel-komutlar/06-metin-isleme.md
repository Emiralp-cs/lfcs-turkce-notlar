# Metin İşleme: head, tail, sed, cut, sort, uniq, diff

Linux'ta yapılandırma dosyaları, loglar ve komut çıktıları düz metindir. Bu yüzden metni görüntülemek, değiştirmek, süzmek ve karşılaştırmak günlük yönetimin büyük kısmını oluşturur. Bu dosyadaki komutların her biri tek bir işi yapar, asıl gücü ise boru (`|`) ile birbirine bağlandıklarında ortaya çıkar.

Senaryo: Ekibin personel listesi `personel.csv` dosyasında tutuluyor. Her satırda kullanıcı adı, UID, şehir ve rol var:

```
ayse,1001,Izmir,gelistirici
mert,1002,Ankara,stajyer
elif,1003,izmir,gelistirici
can,1004,Istanbul,operasyon
deniz,1005,Izmir,denetci
kerem,1006,Ankara,operasyon
selin,1007,Istanbul,gelistirici
```

Listede bilerek bir tutarsızlık var: `elif`'in şehri küçük harfle `izmir` yazılmış.

> [!NOTE]
> Bütün komutlar ve çıktılar Ubuntu 24.04'te bu dosyayla denenmiştir. Düzenli ifadeler (regex) ayrıntılı olarak [grep ve Düzenli İfadeler](07-grep-ve-regex.md) dosyasında.

## İçindekiler

1. [Görüntülemek: cat, tac, head, tail](#1-görüntülemek-cat-tac-head-tail)
2. [Saymak: wc](#2-saymak-wc)
3. [Bul ve değiştir: sed](#3-bul-ve-değiştir-sed)
4. [Sütun çıkarmak: cut](#4-sütun-çıkarmak-cut)
5. [Sıralamak: sort](#5-sıralamak-sort)
6. [Tekrarları ayıklamak: uniq](#6-tekrarları-ayıklamak-uniq)
7. [Karakter dönüştürmek: tr](#7-karakter-dönüştürmek-tr)
8. [Karşılaştırmak: diff](#8-karşılaştırmak-diff)
9. [Hepsini birleştirmek](#9-hepsini-birleştirmek)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Görüntülemek: cat, tac, head, tail

```bash
cat personel.csv        # baştan sona
tac personel.csv        # sondan başa (cat'in tersi)
```

`tac`, en yeni kaydın sonda olduğu dosyalarda (loglar gibi) en yenileri üstte görmek için kullanışlıdır.

Uzun dosyaların başı ve sonu:

```bash
head personel.csv           # ilk 10 satır
head -n 3 personel.csv      # ilk 3 satır
tail -n 2 personel.csv      # son 2 satır
```

İki az bilinen kullanım:

| Komut | Ne yapar? | Denemedeki sonuç |
|---|---|---|
| `head -n -5 personel.csv` | Son 5 satır **hariç** hepsi | ayse, mert |
| `tail -n +6 personel.csv` | **6. satırdan itibaren** hepsi | kerem, selin |

`tail -n +2`, başlık satırı olan dosyalarda başlığı atlamak için çok kullanılır.

### tail -f: dosyayı canlı izlemek

```bash
tail -f /var/log/syslog
```

Dosyanın sonunu gösterir ve dosyaya yeni satırlar eklendikçe ekrana yazmaya devam eder. Denemede `tail -f` açıkken dosyaya bir satır ekledik, satır anında göründü. Çıkmak için `Ctrl+C`. Bir servisi yeniden başlatırken logunu izlemek için kullanılır.

Uzun dosyaları sayfa sayfa okumak için `less dosya`. Gezinme tuşları man sayfalarıyla aynı ([Yardım Almak](01-yardim-almak.md#3-man-içinde-gezinmek-ve-arama-yapmak)).

---

## 2. Saymak: wc

```bash
wc personel.csv
```
```
  7   7 194 personel.csv
```

Sırasıyla satır, kelime ve bayt sayısı.

```bash
wc -l personel.csv      # sadece satır sayısı
```

Bir komutun kaç sonuç verdiğini saymak için çok kullanılır: `find /etc -name "*.conf" | wc -l`.

---

## 3. Bul ve değiştir: sed

`sed` (stream editor), metni satır satır okuyup üzerinde değişiklik yapar.

### Değiştirme

```
sed 's/ARANAN/YENİ/' dosya
```

`s` substitute (değiştir) demek. `elif`'in şehrini düzeltelim:

```bash
sed 's/izmir/Izmir/' personel.csv
```

Bu komut **dosyayı değiştirmez**. Sonucu ekrana yazar, dosya olduğu gibi kalır. Önce böyle denemek, değişikliğin doğru olduğunu görmek için iyi bir alışkanlık.

### g bayrağı: her eşleşme

`s` komutu varsayılan olarak her satırdaki **ilk** eşleşmeyi değiştirir:

```bash
echo "Izmir Izmir" | sed 's/Izmir/IZMIR/'
echo "Izmir Izmir" | sed 's/Izmir/IZMIR/g'
```
```
IZMIR Izmir
IZMIR IZMIR
```

Sondaki `g` (global), satırdaki bütün eşleşmeleri değiştirir. `I` bayrağı büyük-küçük harfi yok sayar: `s/izmir/Izmir/Ig` hem `izmir`'i hem `IZMIR`'i yakalar.

### Dosyanın kendisini değiştirmek: -i

```bash
sed -i 's/izmir/Izmir/' personel.csv
```

`-i` (in-place) değişikliği dosyaya yazar. **Geri alınamaz.** Değiştirmeden önce bir yedek almak için `-i`'nin hemen arkasına bir uzantı yaz:

```bash
sed -i.bak 's/stajyer/gelistirici/' personel.csv
```

Denemede `personel.csv` değişti, eski hali `personel.csv.bak` olarak kaldı.

### Satır seçmek ve silmek

```bash
sed -n '2,4p' personel.csv          # sadece 2-4. satırları yazdır
sed -n '/operasyon/p' personel.csv  # sadece "operasyon" geçen satırlar
sed '/stajyer/d' personel.csv       # "stajyer" geçen satırları sil
sed '1d' personel.csv               # ilk satırı sil
```

`-n` normal çıktıyı kapatır, `p` sadece seçilen satırları yazdırır. `d` satırı siler.

### Ayraç değiştirmek

Değiştirilecek metinde `/` varsa (dosya yolları gibi) her birini `\/` diye kaçırmak gerekir. Bunun yerine başka bir ayraç kullanılabilir:

```bash
echo "/srv/eski/yol" | sed 's#/srv/eski#/srv/yeni#'
```
```
/srv/yeni/yol
```

`s`'den hemen sonra gelen karakter ayraç olur.

### Tek tırnak kullan

```bash
echo 'fiyat: $100' | sed "s/$100/200/"
echo 'fiyat: $100' | sed 's/\$100/200/'
```
```
fiyat: $1200
fiyat: 200
```

Çift tırnak içinde shell `$1`'i bir değişken sanıp boşlukla değiştirdi ve `sed` aslında `s/00/200/` komutunu çalıştırdı. `sed` ifadelerini **tek tırnakla** yaz. İçinde shell değişkeni kullanmak istiyorsan çift tırnak kullan ama `$`, `\` gibi karakterlere dikkat et.

---

## 4. Sütun çıkarmak: cut

```
cut -d AYRAÇ -f ALANLAR dosya
```

```bash
cut -d, -f1 personel.csv          # 1. alan: kullanıcı adları
cut -d, -f1,3 personel.csv        # 1. ve 3. alan
cut -d, -f2-3 personel.csv        # 2'den 3'e kadar
cut -c1-4 personel.csv            # her satırın ilk 4 karakteri
```
```
ayse,Izmir
mert,Ankara
...
```

`-d` alanları ayıran karakteri, `-f` alınacak alanları belirtir. `-d` yazılmazsa ayraç tab karakteridir.

Sistem dosyalarında çok işe yarar. `/etc/passwd`'dan kullanıcı adları ve shell'ler:

```bash
cut -d: -f1,7 /etc/passwd
```
```
root:/bin/bash
daemon:/usr/sbin/nologin
```

> [!WARNING]
> `cut` her ayraç karakterini ayrı sayar. Kelimeler arasında **birden fazla boşluk** varsa alanlar kayar:
> ```bash
> echo "ayse  mert" | cut -d' ' -f2      # boş satır
> echo "ayse  mert" | awk '{print $2}'   # mert
> ```
> İki boşluk arasında "boş" bir alan oluştu. `ps`, `df`, `ls -l` gibi sütunları boşlukla hizalanmış çıktılarda `cut` yerine `awk '{print $N}'` kullan.

---

## 5. Sıralamak: sort

```bash
sort dosya          # alfabetik
sort -r dosya       # ters
sort -n dosya       # sayısal
sort -u dosya       # sırala ve tekrarları kaldır
```

### Sayılar: -n gerekli

```bash
printf '10\n9\n100\n2\n' | sort
printf '10\n9\n100\n2\n' | sort -n
printf '10\n9\n100\n2\n' | sort -rn
```
```
10 100 2 9
2 9 10 100
100 10 9 2
```

`-n` olmadan sayılar metin gibi, karakter karakter sıralanır ve `100`, `9`'dan önce gelir.

### Belirli bir alana göre: -t ve -k

```bash
sort -t, -k3,3 -k1,1 personel.csv     # önce şehre, şehir aynıysa ada göre
sort -t, -k2,2nr personel.csv         # 2. alana (UID) göre sayısal ve ters
```

- `-t,` alan ayracı virgül.
- `-k3,3` sadece 3. alana göre. `-k3` yazılırsa 3. alandan satır sonuna kadar olan kısım kullanılır.
- Alan numarasının arkasına `n` veya `r` yazılarak sadece o alan için sayısal veya ters sıralama yapılır.

### Okunabilir boyutlar: -h

```bash
du -sh /usr/* | sort -h
```

`-h`, `2.4G`, `500M`, `12K` gibi değerleri doğru sıralar. Diskte en çok yer kaplayan dizinleri bulmak için kullanılır.

> [!NOTE]
> Harflerin sıralanma şekli dil ayarına (locale) bağlıdır. Deneme ortamı `C` yerel ayarındaydı ve büyük harfler küçüklerden önce geldi (`A B a b`). `en_US.UTF-8` gibi ayarlarda sıra farklı olabilir. Her yerde aynı sonucu almak için `LC_ALL=C sort` kullanılır.

---

## 6. Tekrarları ayıklamak: uniq

`uniq` sadece **art arda gelen** aynı satırları birleştirir:

```bash
cut -d, -f3 personel.csv | uniq
```
```
Izmir
Ankara
izmir
Istanbul
Izmir
Ankara
Istanbul
```

Hiçbir şey birleşmedi, çünkü aynı şehirler yan yana değil. Önce sıralamak gerekir:

```bash
cut -d, -f3 personel.csv | sort | uniq
```
```
Ankara
Istanbul
Izmir
izmir
```

`izmir` ile `Izmir` ayrı satırlar sayıldı, dosyadaki yazım hatası ortaya çıktı.

| Seçenek | Ne yapar? |
|---|---|
| `uniq -c` | Her satırın kaç kez tekrarlandığını yazar |
| `uniq -d` | Sadece tekrar eden satırlar |
| `uniq -u` | Sadece bir kez geçen satırlar |
| `uniq -i` | Büyük-küçük harfi yok sayar (`sort -f` ile birlikte kullan) |

Her şehirde kaç kişi var, en kalabalıktan başlayarak:

```bash
cut -d, -f3 personel.csv | sort | uniq -c | sort -rn
```
```
      2 Izmir
      2 Istanbul
      2 Ankara
      1 izmir
```

Bu kalıp (`sort | uniq -c | sort -rn`) loglarda en sık görülen IP'yi veya hata mesajını bulmak için çok kullanılır.

`sort -u`, `sort | uniq` ile aynı işi tek komutta yapar.

---

## 7. Karakter dönüştürmek: tr

`tr`, karakterleri birebir değiştirir. Dosya almaz, sadece girdi okur:

```bash
echo "merhaba" | tr a-z A-Z          # MERHABA
echo "Ayse,Mert" | tr ',' '\n'       # virgülleri satır sonuna çevir
tr -d '\r' < windows.txt > linux.txt  # Windows satır sonlarındaki \r'yi sil
```

---

## 8. Karşılaştırmak: diff

İki dosya arasındaki farkları gösterir. SSH yapılandırmasının eski ve yeni halini karşılaştıralım:

```
eski.conf                       yeni.conf
Port 22                         Port 2222
PermitRootLogin yes             PermitRootLogin no
PasswordAuthentication yes      PasswordAuthentication yes
X11Forwarding yes               X11Forwarding yes
                                ClientAliveInterval 300
```

### Normal biçim

```bash
diff eski.conf yeni.conf
```
```
1,2c1,2
< Port 22
< PermitRootLogin yes
---
> Port 2222
> PermitRootLogin no
4a5
> ClientAliveInterval 300
```

| Parça | Anlamı |
|---|---|
| `1,2c1,2` | Birinci dosyanın 1-2. satırları, ikinci dosyanın 1-2. satırlarıyla **değişmiş** (change) |
| `4a5` | Birinci dosyanın 4. satırından sonra, ikincideki 5. satır **eklenmiş** (add) |
| `2d1` | Birinci dosyanın 2. satırı **silinmiş** (delete) |
| `<` | Birinci dosyadaki satır |
| `>` | İkinci dosyadaki satır |

### Birleşik biçim: -u

```bash
diff -u eski.conf yeni.conf
```
```
--- eski.conf	2026-11-25 15:46:56.290049052 +0000
+++ yeni.conf	2026-11-25 15:46:56.290049052 +0000
@@ -1,4 +1,5 @@
-Port 22
-PermitRootLogin yes
+Port 2222
+PermitRootLogin no
 PasswordAuthentication yes
 X11Forwarding yes
+ClientAliveInterval 300
```

`-` ile başlayanlar kaldırılmış, `+` ile başlayanlar eklenmiş, boşlukla başlayanlar değişmemiş satırlardır. Git'in gösterdiği fark da bu biçimdedir, okuması en kolay olanı budur.

### Yan yana: -y ve sdiff

```bash
diff -y eski.conf yeni.conf
```
```
Port 22                           |   Port 2222
PermitRootLogin yes               |   PermitRootLogin no
PasswordAuthentication yes            PasswordAuthentication yes
X11Forwarding yes                     X11Forwarding yes
                                  >   ClientAliveInterval 300
```

`|` değişmiş, `>` sadece ikincide olan, `<` sadece birincide olan satırı gösterir. `sdiff eski.conf yeni.conf` aynı çıktıyı verir.

### Diğer kullanışlı seçenekler

| Komut | Ne yapar? |
|---|---|
| `diff -q a b` | Sadece "farklı mı" sorusunu cevaplar: `Files a and b differ` |
| `diff -i a b` | Büyük-küçük harf farkını yok sayar |
| `diff -r dizin1 dizin2` | İki dizini karşılaştırır, sadece birinde olan dosyaları da gösterir |
| `diff -rq dizin1 dizin2` | Dizinlerde sadece hangi dosyaların farklı olduğunu listeler |

`diff`'in çıkış kodu dosyalar aynıysa 0, farklıysa 1'dir. Script'lerde `if diff -q a b >/dev/null; then ...` şeklinde kullanılabilir.

---

## 9. Hepsini birleştirmek

Boru (`|`), bir komutun çıktısını bir sonrakine girdi olarak verir. Küçük komutları zincirleyerek karmaşık işler yapılır:

```bash
# Sistemde hangi shell'i kaç kullanıcı kullanıyor?
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn

# Personel listesinde şehir adlarındaki yazım farklılıklarını düzelt ve tekrarsız listele
sed 's/izmir/Izmir/Ig' personel.csv | cut -d, -f3 | sort -u

# Bir yapılandırma dosyasının yorum ve boş satırlar hariç içeriği
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config
```

Komutun çıktısını dosyaya kaydetmek için `>`:

```bash
cut -d, -f3 personel.csv | sort -u > sehirler.txt
```

`>` dosyanın içeriğini siler ve yeniden yazar, eklemek için `>>`. Ayrıntılar [Girdi/Çıktı Yönlendirme](10-yonlendirme.md) dosyasında.

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `cat` / `tac` | Baştan / sondan görüntüler |
| `head -n N` / `tail -n N` | İlk / son N satır |
| `head -n -N` | Son N satır hariç |
| `tail -n +N` | N. satırdan itibaren |
| `tail -f` | Dosyayı canlı izler |
| `wc -l` | Satır sayısı |
| `sed 's/a/b/'` | Her satırda ilk eşleşmeyi değiştirir (ekrana) |
| `sed 's/a/b/g'` | Bütün eşleşmeleri değiştirir |
| `sed -i 's/a/b/g' dosya` | Dosyanın kendisini değiştirir |
| `sed -i.bak ...` | Değiştirmeden önce yedek alır |
| `sed -n '2,4p'` / `sed '/x/d'` | Satır seçer / siler |
| `cut -d, -f1,3` | Belirli alanları alır |
| `sort` / `sort -n` / `sort -r` | Alfabetik / sayısal / ters |
| `sort -t, -k3,3` | Belirli alana göre |
| `sort -h` | Okunabilir boyutlara göre |
| `sort -u` | Sırala ve tekrarları kaldır |
| `uniq` / `uniq -c` | Art arda tekrarları birleştirir / sayar |
| `tr a-z A-Z` | Karakter dönüştürür |
| `diff a b` / `diff -u a b` | Farklar / birleşik biçim |
| `diff -y a b` / `sdiff a b` | Yan yana karşılaştırma |
| `diff -q` / `diff -r` | Sadece farklı mı / dizinler |

---

## 11. Sık yapılan hatalar

- **`sed`'in dosyayı değiştirdiğini sanmak.** `-i` olmadan sadece ekrana yazar.
- **`sed -i`'yi denemeden kullanmak.** Önce `-i`'siz çalıştır, sonuç doğruysa `-i` ekle, ya da `-i.bak` ile yedek al.
- **`g` bayrağını unutmak.** Satırdaki sadece ilk eşleşme değişir.
- **`sed` ifadesini çift tırnakla yazmak.** Shell `$` gibi karakterleri yorumlar, tek tırnak kullan.
- **Birden fazla boşlukla ayrılmış sütunlarda `cut` kullanmak.** `awk` kullan.
- **Sayıları `-n`'siz sıralamak.** `100`, `9`'dan önce gelir.
- **`uniq`'i `sort`'suz kullanmak.** Sadece yan yana olan tekrarlar birleşir.
- **`>` ile mevcut dosyanın üzerine yazmak.** Aynı dosyayı hem okuyup hem `>` ile yazma: `sort dosya > dosya` dosyayı boşaltır.

---

## 12. Alıştırmalar

Alıştırmalar için yukarıdaki `personel.csv` dosyasını oluştur.

**1.** Dosyanın 3. ile 5. satırları arasını iki farklı yöntemle yazdır.

<details>
<summary>Çözüm</summary>

```bash
sed -n '3,5p' personel.csv
head -n 5 personel.csv | tail -n 3
```
</details>

**2.** `elif` satırındaki `izmir`'i `Izmir` yap. Dosyayı değiştirmeden önce yedek al ve iki dosya arasındaki farkı birleşik biçimde göster.

<details>
<summary>Çözüm</summary>

```bash
sed -i.bak 's/izmir/Izmir/' personel.csv
diff -u personel.csv.bak personel.csv
```
</details>

**3.** Her rolde kaç kişi olduğunu en kalabalık rolden başlayarak listele.

<details>
<summary>Çözüm</summary>

```bash
cut -d, -f4 personel.csv | sort | uniq -c | sort -rn
```
</details>

**4.** Personeli UID'ye göre büyükten küçüğe sırala ve sadece ilk üç kişinin kullanıcı adını yazdır.

<details>
<summary>Çözüm</summary>

```bash
sort -t, -k2,2nr personel.csv | head -n 3 | cut -d, -f1
```
</details>

**5.** `/etc/ssh/sshd_config` dosyasını yorum satırları ve boş satırlar olmadan görüntüle. Sonuçta kaç satır kaldığını say.

<details>
<summary>Çözüm</summary>

```bash
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config | wc -l
```

`grep -v` ile aynı iş [grep](07-grep-ve-regex.md) dosyasında.
</details>

---

Önceki konu: [find ile Dosya Arama](05-find.md) · Sonraki konu: [grep ve Düzenli İfadeler](07-grep-ve-regex.md)
