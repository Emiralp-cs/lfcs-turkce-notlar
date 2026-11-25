# grep ve Düzenli İfadeler

`grep`, dosyalarda belirli bir metni içeren satırları bulur. **Düzenli ifadeler** (regular expressions, kısaca regex) ise "satır başında `#` olan", "bir IP adresi", "`ERROR` veya `WARN`" gibi desenleri tarif etmenin dilidir. İkisi birlikte log incelemenin ve yapılandırma dosyalarında arama yapmanın temel aracıdır.

Senaryo: Ekibin uygulaması `uygulama.log` dosyasına kayıt yazıyor. Hataları, kullanıcı girişlerini ve IP adreslerini bulacağız:

```
# uygulama.log - ornek kayitlar
2026-11-25 09:00:01 INFO  sunucu basladi port=8080
2026-11-25 09:00:05 WARN  disk kullanimi %85
2026-11-25 09:01:12 ERROR veritabani baglantisi kurulamadi host=10.0.0.12
2026-11-25 09:01:15 INFO  yeniden deneniyor (1/3)
2026-11-25 09:01:20 error zaman asimi host=10.0.0.12
2026-11-25 09:02:00 INFO  kullanici ayse giris yapti ip=192.168.10.50
2026-11-25 09:02:30 INFO  kullanici mert giris yapti ip=192.168.10.37
2026-11-25 09:03:00 ERROR yetkisiz erisim denemesi ip=203.0.113.99
2026-11-25 09:04:00 INFO  servis disabled durumda: yedekleme
2026-11-25 09:04:10 INFO  servis enabled: izleme
2026-11-25 09:04:20 INFO  ozellik disables listesinde
```

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve GNU grep 3.11 ile bu dosya üzerinde denenmiştir. Çıktılarda uzun satırlar kısaltılmış olabilir.

## İçindekiler

1. [grep'in temel seçenekleri](#1-grepin-temel-seçenekleri)
2. [Dizinlerde arama: -r](#2-dizinlerde-arama--r)
3. [Düzenli ifadeler: neden gerekli?](#3-düzenli-ifadeler-neden-gerekli)
4. [Satır başı ve sonu: ^ ve $](#4-satır-başı-ve-sonu--ve-)
5. [Herhangi bir karakter: .](#5-herhangi-bir-karakter-)
6. [Karakter kümeleri: [...] ve [^...]](#6-karakter-kümeleri--ve-)
7. [Tekrar: *, +, ?, {n,m}](#7-tekrar-----nm)
8. [Temel ve genişletilmiş regex: grep -E](#8-temel-ve-genişletilmiş-regex-grep--e)
9. [Seçenek ve gruplama: | ve ( )](#9-seçenek-ve-gruplama--ve--)
10. [Deseni adım adım kurmak: IP adresi](#10-deseni-adım-adım-kurmak-ip-adresi)
11. [Düz metin araması: grep -F](#11-düz-metin-araması-grep--f)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. grep'in temel seçenekleri

```bash
grep ERROR uygulama.log
```

`ERROR` geçen satırları yazar. Sık kullanılan seçenekler:

| Seçenek | Ne yapar? | Denemedeki sonuç |
|---|---|---|
| (yok) | Eşleşen satırlar | `ERROR`: 2 satır |
| `-i` | Büyük-küçük harfi yok say | `error`: 3 satır (küçük harfli `error` da geldi) |
| `-c` | Satır yerine eşleşen satır **sayısı** | `-ci error`: `3` |
| `-v` | Eşleş**meyen** satırlar | `-v '^#'`: yorum hariç 11 satır |
| `-n` | Satır numaralarını da yaz | `4:2026-11-25 09:01:12 ERROR ...` |
| `-w` | Sadece **tam kelime** olarak geçenler | Aşağıda |
| `-x` | Sadece **satırın tamamı** eşleşenler | |
| `-o` | Satırın tamamı yerine sadece eşleşen kısım | Aşağıda |
| `-l` | Satırlar yerine eşleşme olan **dosya adları** | |
| `-A N` / `-B N` / `-C N` | Eşleşmenin sonrasındaki / öncesindeki / iki yanındaki N satır | |
| `-q` | Hiçbir şey yazma, sadece çıkış koduyla bildir | |

### -o: sadece eşleşen kısım

```bash
grep -o 'ip=[0-9.]*' uygulama.log
```
```
ip=192.168.10.50
ip=192.168.10.37
ip=203.0.113.99
```

### -A, -B, -C: bağlam

Hatanın hemen arkasından ne olduğunu görmek için:

```bash
grep -A1 'ERROR veritabani' uygulama.log
```
```
2026-11-25 09:01:12 ERROR veritabani baglantisi kurulamadi host=10.0.0.12
2026-11-25 09:01:15 INFO  yeniden deneniyor (1/3)
```

### -w: tam kelime

```bash
grep 'disable' uygulama.log      # disabled ve disables satırları
grep -w 'disabled' uygulama.log  # sadece "disabled" kelimesi
```

`-w` olmadan `disable`, `disabled` ve `disables` kelimelerinin içinde de bulunur.

### Çıkış kodu

`grep` bir şey bulursa `0`, bulamazsa `1` ile çıkar. Script'lerde:

```bash
if grep -q ERROR uygulama.log; then echo "hata var"; fi
```

### Birden fazla süzgeç

`grep`'ler boruyla zincirlenebilir:

```bash
grep -i error uygulama.log | grep -v zaman     # hatalar, "zaman" geçenler hariç
```

---

## 2. Dizinlerde arama: -r

```bash
grep -r 'Port' /etc/ssh/              # alt dizinler dahil bütün dosyalarda
grep -rl 'Port' /etc/ssh/             # sadece eşleşme olan dosyaların adları
grep -rn '^Port' /etc/ssh/            # satır numaralarıyla
grep -r --include='*.conf' 'listen' /etc/   # sadece .conf dosyalarında
```

`-r` ile her satırın başında dosya adı da yazar. `grep -rn 'Port' /etc/ssh/` çıktısının bir kısmı:

```
/etc/ssh/sshd_config:23:#Port 22
/etc/ssh/sshd_config:98:#GatewayPorts no
/etc/ssh/ssh_config:40:#   Port 22
```

`GatewayPorts` da geldi, çünkü içinde `Port` geçiyor. Sadece `Port` ile başlayan satırlar için `^Port` kullanılır ([4. bölüm](#4-satır-başı-ve-sonu--ve-)).

Erişim izni olmayan dosyalar için `Permission denied` hataları çıkarsa `2>/dev/null` ekle veya `sudo` kullan.

---

## 3. Düzenli ifadeler: neden gerekli?

Düz bir arama sadece aynı karakterleri bulur. `grep '7'`, içinde `7` olan **her** satırı getirir: `kod 7`, `surum 17`, `7 numara`. Ama genelde istenen daha belirgindir: "sonu 7 ile biten", "sadece tek başına 7", "üç basamaklı bir sayı". Düzenli ifadeler bu kuralları tarif eder.

Temel operatörler:

| Operatör | Anlamı |
|---|---|
| `^` | Satır başı |
| `$` | Satır sonu |
| `.` | Herhangi tek bir karakter |
| `[abc]` | Köşeli parantezdeki karakterlerden biri |
| `[^abc]` | Köşeli parantezdekiler **dışında** bir karakter |
| `*` | Önceki öğe 0 veya daha fazla kez |
| `+` | Önceki öğe 1 veya daha fazla kez |
| `?` | Önceki öğe 0 veya 1 kez |
| `{n,m}` | Önceki öğe en az n, en fazla m kez |
| `\|` | Veya |
| `( )` | Gruplama |

Bu operatörlerin bazıları `grep`'te doğrudan, bazıları ancak `grep -E` ile çalışır ([8. bölüm](#8-temel-ve-genişletilmiş-regex-grep--e)).

> [!TIP]
> Desenleri **tek tırnak** içinde yaz. `*`, `$`, `?` gibi karakterler shell için de özeldir ve tırnak olmazsa `grep`'e ulaşmadan değiştirilebilirler.

---

## 4. Satır başı ve sonu: ^ ve $

```bash
grep '^#' uygulama.log           # # ile başlayan satırlar (yorumlar)
grep -v '^#' uygulama.log        # yorum OLMAYAN satırlar
grep '^Port' /etc/ssh/sshd_config  # tam olarak "Port" ile başlayan (yorumdaki #Port hariç)
grep 'listesinde$' uygulama.log  # "listesinde" ile biten
grep -v '^$' dosya               # boş olmayan satırlar
```

`^$`, başı ile sonu arasında hiçbir şey olmayan, yani boş satır demektir. Yorumları ve boş satırları birlikte atmak için:

```bash
grep -v -e '^#' -e '^$' /etc/ssh/sshd_config
```

### $ ile -w farkı

Üç satırlık bir dosyada denedik: `kod 7`, `surum 17`, `7 numara`.

| Desen | Bulduğu | Neden? |
|---|---|---|
| `grep '7'` | Üçü | İçinde 7 olan her satır |
| `grep '7$'` | `kod 7`, `surum 17` | 7 ile biten satırlar |
| `grep -w '7$'` | `kod 7` | 7 ile biten **ve** 7 tek başına bir kelime |
| `grep -w '7'` | `kod 7`, `7 numara` | 7'nin tek başına kelime olduğu satırlar |

---

## 5. Herhangi bir karakter: .

`.`, tam olarak **bir** karakterin yerini tutar:

```bash
grep 'c.t' kelimeler.txt
```

`cat`, `cut`, `CRT`, `C1T`, `CT`, `coat` kelimelerinin olduğu bir dosyada:

| Desen | Bulduğu |
|---|---|
| `c.t` | `cat`, `cut` |
| `-i 'c.t'` | `cat`, `cut`, `CRT`, `C1T` |
| `c..t` | `coat` |

`CT`'de `c` ile `t` arasında karakter olmadığı için `c.t`'ye uymaz. `coat`'ta iki karakter var, `c..t` gerekir.

`.` gerçek bir nokta değil, herhangi bir karakter olduğu için `grep '1.2.3'` hem `1.2.3`'ü hem `1x2x3`'ü bulur. Gerçek noktayı aramak için `\.` yaz: `grep '1\.2\.3'`.

---

## 6. Karakter kümeleri: [...] ve [^...]

| Küme | Anlamı |
|---|---|
| `[ou]` | `o` veya `u` |
| `[a-z]` | Küçük harf |
| `[A-Z]` | Büyük harf |
| `[0-9]` | Rakam |
| `[a-zA-Z0-9]` | Harf veya rakam |
| `[^0-9]` | Rakam **olmayan** bir karakter |

```bash
grep -E 'c[ou]t' kelimeler.txt       # cut (cat değil, çünkü a kümede yok)
grep -E '^[A-Z]+$' kelimeler.txt     # sadece büyük harflerden oluşan satırlar: CRT, CT
```

### Olumsuz küme: [^...]

Köşeli parantezin **içindeki** `^`, "bunlar dışında" demektir (satır başı anlamındaki `^` ile karıştırma):

```bash
grep -E 'https[^:]' <<< $'https://a\nhttps.txt\nhttps'
```
```
https.txt
```

`https` sonrasında `:` **olmayan** bir karakter isteniyor. `https://a` elendi. Tek başına `https` da elendi, çünkü arkasında hiç karakter yok ve `[^:]` bir karakter ister.

---

## 7. Tekrar: *, +, ?, {n,m}

Tekrar operatörleri **hemen önlerindeki** öğeye uygulanır.

### * : sıfır veya daha fazla

`le`, `let`, `lett`, `lattice` satırlarında:

```bash
grep 'let*' liste.txt
```
```
le
let
lett
```

`t*`, "sıfır veya daha fazla `t`" demek. `le` de eşleşti çünkü sıfır `t` de kabul.

> [!WARNING]
> Regex'teki `*`, shell'deki `*` ile aynı değil. Shell'de (`ls *.log`) "herhangi bir şey", regex'te "önceki karakterin tekrarı" demektir. Regex'te "herhangi bir şey" `.*` ile yazılır.

### + ve ?

| Operatör | Anlamı | Örnek | Eşleşenler |
|---|---|---|---|
| `+` | 1 veya daha fazla | `^let+$` | `let`, `lett` (ama `le` değil) |
| `?` | 0 veya 1 (isteğe bağlı) | `colou?r` | `color`, `colour` (ama `colouur` değil) |

`disabled?`, `disable` ve `disabled`'ı bulur. Ama `disables`'ın içinde de `disable` geçtiği için onu da bulur. Kelimenin tam olarak bunlardan biri olmasını istiyorsan `-w` ekle.

### {n,m}: belirli sayıda

| Desen | Anlamı |
|---|---|
| `0{3}` | Tam 3 tane `0` |
| `0{2,}` | En az 2 tane |
| `0{,2}` | En fazla 2 tane |
| `0{1,2}` | 1 veya 2 tane |

`100`, `1000`, `10`, `105` satırlarında:

```bash
grep -E '0{3}' sayilar.txt        # 1000
grep -E '^10{1,2}$' sayilar.txt   # 10, 100
grep -E '^10{2,}$' sayilar.txt    # 100, 1000
```

`^` ve `$` olmadan `0{1,2}` satırın herhangi bir yerinde bir veya iki sıfır arar, `1000` de eşleşirdi.

---

## 8. Temel ve genişletilmiş regex: grep -E

`grep` iki regex dili bilir:

| | Temel (BRE): `grep` | Genişletilmiş (ERE): `grep -E` |
|---|---|---|
| `^ $ . * [ ]` | Doğrudan çalışır | Doğrudan çalışır |
| `+ ? \| { } ( )` | Önlerine ters bölü (`\`) gerekir | Doğrudan çalışır |

Aynı aramaları iki şekilde denedik (`sayilar.txt` içinde `100`, `1000`, `10`, `105` var):

```bash
grep '0+' sayilar.txt                    # 0 satır: + düz karakter olarak arandı
grep '0\+' sayilar.txt                   # 4 satır
grep -E '0+' sayilar.txt                 # 4 satır

grep 'enabled|disabled' uygulama.log     # 0 satır: | düz karakter olarak arandı
grep 'enabled\|disabled' uygulama.log    # 2 satır
grep -E 'enabled|disabled' uygulama.log  # 2 satır
```

`+`, `?`, `|`, `{}`, `()` kullanacaksan `grep -E` ile yazmak hem daha okunur hem daha az hataya açıktır.

### egrep

`egrep`, `grep -E`'nin eski kısayoludur. Ubuntu 24.04'te `/usr/bin/egrep` sadece `grep -E`'yi çağıran küçük bir script:

```bash
cat $(which egrep)
```
```
#!/bin/sh
cmd=${0##*/}
exec grep -E "$@"
```

GNU grep 3.8'den itibaren `egrep` resmi olarak eskimiş sayılıyor ve bazı dağıtımlarda kullanınca uyarı veriyor. Yeni yazdığın komutlarda `grep -E` kullan.

---

## 9. Seçenek ve gruplama: | ve ( )

```bash
grep -E 'ERROR|WARN' uygulama.log
grep -E 'enabled|disabled' uygulama.log
```

Parantez, birkaç öğeyi tek bir birim yapar. Tekrar operatörü artık bütün gruba uygulanır:

```bash
grep -E '^[0-9-]+ [0-9:]+ (ERROR|WARN)' uygulama.log
```
```
2026-11-25 09:00:05 WARN  disk kullanimi %85
2026-11-25 09:01:12 ERROR veritabani baglantisi ...
2026-11-25 09:03:00 ERROR yetkisiz erisim denemesi ...
```

Desen şunu diyor: satır başında rakam ve tirelerden oluşan bir tarih, bir boşluk, rakam ve iki noktalardan oluşan bir saat, bir boşluk, sonra `ERROR` veya `WARN`. Küçük harfli `error` satırı eşleşmedi.

---

## 10. Deseni adım adım kurmak: IP adresi

Karmaşık bir deseni tek seferde yazmaya çalışma. Genel başla, her adımda biraz daralt ve sonucu kontrol et. Logdaki IP adreslerini bulalım:

**1. Rakam ve noktalardan oluşan her şey:**

```bash
grep -oE '[0-9.]+' <<< "surum 1.2 ip=10.0.0.12"
```
```
1.2
10.0.0.12
```

Sürüm numarası da yakalandı. Çok geniş.

**2. Nokta ile ayrılmış dört sayı grubu:**

```bash
grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' <<< "surum 1.2 ip=10.0.0.12"
```
```
10.0.0.12
```

`\.` gerçek nokta. Sürüm numarası elendi.

**3. Tekrar eden kısmı grupla, her grup en fazla 3 basamaklı olsun:**

```bash
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' uygulama.log | sort | uniq -c
```
```
      2 10.0.0.12
      1 192.168.10.37
      1 192.168.10.50
      1 203.0.113.99
```

`([0-9]{1,3}\.){3}`, "1-3 basamak ve bir nokta" grubunu üç kez tekrarlar. Sondaki `[0-9]{1,3}` dördüncü sayı. Sonucu [Metin İşleme](06-metin-isleme.md#6-tekrarları-ayıklamak-uniq) dosyasındaki `sort | uniq -c` kalıbıyla saydık.

Bu desen `999.999.999.999` gibi geçersiz adresleri de kabul eder, ama log incelemek için genelde yeterlidir. Deseni denemek için regex101.com gibi siteler kullanılabilir, sınavda ise adım adım `grep` ile denemek en güvenli yol.

---

## 11. Düz metin araması: grep -F

Aradığın metinde `.`, `*`, `[` gibi karakterler varsa ve bunların regex olarak yorumlanmasını istemiyorsan `-F` kullan:

```bash
grep '1.2.3' <<< $'1x2x3\n1.2.3'       # ikisini de bulur (. herhangi bir karakter)
grep -F '1.2.3' <<< $'1x2x3\n1.2.3'    # sadece 1.2.3
```

`-F` (fixed strings) hiçbir karakteri özel saymaz.

---

## 12. Komut özeti

| Komut / Desen | Anlamı |
|---|---|
| `grep desen dosya` | Eşleşen satırlar |
| `grep -i` / `-v` / `-c` / `-n` | Harf duyarsız / tersi / sayı / satır no |
| `grep -w` / `-x` | Tam kelime / tam satır |
| `grep -o` | Sadece eşleşen kısım |
| `grep -A N` / `-B N` / `-C N` | Sonraki / önceki / çevredeki satırlar |
| `grep -l` | Eşleşen dosya adları |
| `grep -r` / `-rl` | Alt dizinler dahil |
| `grep -q` | Sessiz, sadece çıkış kodu |
| `grep -E` | Genişletilmiş regex |
| `grep -F` | Düz metin, regex yok |
| `grep -e A -e B` | Birden fazla desen |
| `^` / `$` | Satır başı / sonu |
| `^$` | Boş satır |
| `.` / `\.` | Herhangi bir karakter / gerçek nokta |
| `[a-z]` / `[^0-9]` | Küme / küme dışı |
| `*` / `+` / `?` | 0+ / 1+ / 0-1 kez |
| `{n}` / `{n,}` / `{n,m}` | Tam / en az / aralık |
| `A\|B` | Veya (`grep -E` ile) |
| `( )` | Gruplama (`grep -E` ile) |

---

## 13. Sık yapılan hatalar

- **Deseni tırnaksız veya çift tırnakla yazmak.** Shell özel karakterleri değiştirebilir, tek tırnak kullan.
- **`+`, `?`, `|`'yi düz `grep`'te kaçırmadan kullanmak.** Sonuç sıfır çıkar, `grep -E` kullan.
- **Regex'teki `*`'ı shell'deki gibi sanmak.** "Herhangi bir şey" `.*`'dır.
- **`.`'nın gerçek nokta olduğunu sanmak.** IP ve sürüm numaralarında `\.` veya `grep -F`.
- **`[^...]` içindeki `^`'yi satır başı sanmak.** Köşeli parantez içinde "değil" demek.
- **`^` ve `$` olmadan tekrar sayısı aramak.** `0{1,2}`, `1000`'in içinde de bulunur.
- **`-i`'yi unutup `ERROR` ile `error`'ı ayrı saymak.**
- **Karmaşık deseni tek seferde yazmak.** Adım adım daralt.

---

## 14. Alıştırmalar

Alıştırmalar için yukarıdaki `uygulama.log` dosyasını oluştur.

**1.** Büyük-küçük harf fark etmeksizin hata satırlarını satır numaralarıyla göster. Kaç tane olduğunu da say.

<details>
<summary>Çözüm</summary>

```bash
grep -in error uygulama.log
grep -ic error uygulama.log     # 3
```
</details>

**2.** `/etc/ssh/sshd_config` dosyasında yorum satırları ve boş satırlar dışındaki satırları göster.

<details>
<summary>Çözüm</summary>

```bash
grep -v -e '^#' -e '^$' /etc/ssh/sshd_config
```

`grep -Ev '^(#|$)' /etc/ssh/sshd_config` de aynı işi yapar.
</details>

**3.** Logda giriş yapan kullanıcıların sadece adlarını yazdır.

<details>
<summary>Çözüm</summary>

```bash
grep -o 'kullanici [a-z]*' uygulama.log | cut -d' ' -f2
```
```
ayse
mert
```
</details>

**4.** `WARN` veya `ERROR` seviyesindeki satırları bul, ama küçük harfli `error` satırı gelmesin.

<details>
<summary>Çözüm</summary>

```bash
grep -E ' (WARN|ERROR) ' uygulama.log
```

`-i` kullanılmadığı için küçük harfli satır gelmez. Boşluklar, kelimenin başka bir kelimenin içinde geçmesini önler.
</details>

**5.** `/etc/ssh/` altında `PermitRootLogin` geçen bütün satırları dosya adı ve satır numarasıyla bul. Sonra sadece yorum olmayan (satır başında geçen) satırları göster.

<details>
<summary>Çözüm</summary>

```bash
sudo grep -rn 'PermitRootLogin' /etc/ssh/
sudo grep -rn '^PermitRootLogin' /etc/ssh/
```

Ubuntu 24.04'ün varsayılan kurulumunda ilk komut `#PermitRootLogin prohibit-password` gibi yorum satırlarını bulur, ikincisi ise hiçbir şey bulmaz: ayar değiştirilmemiş, varsayılan değer geçerli demektir.
</details>

---

Önceki konu: [Metin İşleme](06-metin-isleme.md) · Sonraki konu: [Arşivleme ve Sıkıştırma](08-arsivleme-ve-sikistirma.md)
