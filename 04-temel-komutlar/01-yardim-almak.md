# Yardım Almak: --help, man ve apropos

LFCS sınavı sırasında internete erişim yok. Bir komutun seçeneğini veya bir dosyanın sözdizimini unuttuğunda tek kaynağın sistemin kendisi. Bu yüzden yardım araçlarını hızlı kullanabilmek, komutları ezberlemekten daha değerli.

Üç araç var ve her biri farklı bir soruya cevap verir:

| Soru | Araç |
|---|---|
| "Bu komutun seçenekleri nelerdi?" | `komut --help` |
| "Bu komutu ayrıntılı öğrenmek istiyorum." | `man komut` |
| "Bu işi yapan komutun adı neydi?" | `apropos anahtar-kelime` |

> [!NOTE]
> Örnekler Ubuntu 24.04'te denenmiştir. Ubuntu'nun "minimal" kurulumlarında ve bazı bulut imajlarında man sayfaları kaldırılmış olabilir. Bu durumda `man` yerine "This system has been minimized..." mesajı çıkar ve `sudo unminimize` komutuyla belgeler geri yüklenir.

## İçindekiler

1. [--help: hızlı hatırlatma](#1---help-hızlı-hatırlatma)
2. [man: komutun tam kılavuzu](#2-man-komutun-tam-kılavuzu)
3. [man içinde gezinmek ve arama yapmak](#3-man-içinde-gezinmek-ve-arama-yapmak)
4. [man bölümleri](#4-man-bölümleri)
5. [apropos: adını bilmediğin komutu bulmak](#5-apropos-adını-bilmediğin-komutu-bulmak)
6. [Shell'e yerleşik komutlar: help](#6-shelle-yerleşik-komutlar-help)
7. [Tab tamamlama](#7-tab-tamamlama)
8. [Başka yardım kaynakları](#8-başka-yardım-kaynakları)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. --help: hızlı hatırlatma

Komutların çoğu `--help` ile kısa bir kullanım özeti yazar:

```bash
ls --help
```
```
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.
...
```

Seçeneğin adını hatırlıyorsun ama tam yazılışından emin değilsen en hızlı yol bu.

Çıktı uzunsa `less` ile sayfa sayfa oku, ya da aradığın kelimeyi `grep` ile süz:

```bash
ls --help | less
ls --help | grep -i size
```

> [!WARNING]
> `-h` her komutta "yardım" demek değildir. `ls -h` yardım göstermez, boyutları "insan okunabilir" (`4.0K`, `2M`) yazar. Denemede `ls -h` normal bir dizin listesi verdi. Yardım için her zaman uzun hali `--help`'i kullan.

---

## 2. man: komutun tam kılavuzu

```bash
man ls
```

Man sayfaları hep aynı düzendedir. `ls` sayfasının başlıkları:

```
LS(1)                            User Commands                           LS(1)

NAME
       ls - list directory contents

SYNOPSIS
       ls [OPTION]... [FILE]...

DESCRIPTION
       List  information  about  the FILEs (the current directory by default).
...
```

| Başlık | İçerik |
|---|---|
| `NAME` | Komutun adı ve tek satırlık açıklaması |
| `SYNOPSIS` | Sözdizimi: `[...]` isteğe bağlı kısımlar, `...` tekrar edilebilir |
| `DESCRIPTION` | Ayrıntılı açıklama, çoğu zaman seçenekler de burada |
| `OPTIONS` | Bazı sayfalarda seçenekler ayrı başlıkta |
| `EXAMPLES` | Örnekler (her sayfada yoktur) |
| `FILES` | Komutun kullandığı yapılandırma dosyaları |
| `SEE ALSO` | İlgili komutlar ve sayfalar |

Başlıktaki `LS(1)`, sayfanın **1. bölümde** olduğunu gösterir ([4. bölüm](#4-man-bölümleri)).

> [!TIP]
> Bir man sayfasında `EXAMPLES` bölümü varsa oradan başla, genelde aradığın kullanımın hazır örneği vardır. Coreutils komutlarında (`ls`, `cp`, `dd` gibi) bu bölüm yok, ama `tar`, `find`, `rsync` gibi araçların sayfalarında genellikle var.

---

## 3. man içinde gezinmek ve arama yapmak

Man sayfaları `less` ile açılır, gezinme tuşları da onunkiler:

| Tuş | Ne yapar? |
|---|---|
| `↑` `↓` veya `j` `k` | Satır satır kaydır |
| `Space` / `b` | Bir sayfa ileri / geri |
| `g` / `G` | En başa / en sona git |
| `/kelime` + Enter | İleri doğru ara |
| `n` / `N` | Sonraki / önceki eşleşme |
| `q` | Çık |
| `h` | Bütün tuşların listesi |

Arama, sınavda en çok işine yarayacak beceri. Örneğin `ls` sayfasında boyuta göre sıralama seçeneğini bulmak için `/size` yazıp `n` ile eşleşmeler arasında gezersin. Bir başlığa doğrudan atlamak için başlığı aratabilirsin: `/EXAMPLES` veya `/SEE ALSO`.

Bazı aramalar man sayfasındaki girintiden dolayı işe yaramaz. Seçenekler satır başında birkaç boşlukla yazıldığı için, `-r` seçeneğinin açıklamasına atlamak için `/^ *-r` gibi bir arama daha isabetli olur (`^` satır başı, ` *` herhangi sayıda boşluk).

---

## 4. man bölümleri

Man sayfaları numaralı bölümlere ayrılmıştır:

| Bölüm | İçerik | Örnek |
|---|---|---|
| 1 | Kullanıcı komutları | `ls`, `cp`, `printf` |
| 2 | Sistem çağrıları (çekirdek) | `open`, `chdir` |
| 3 | C kütüphanesi fonksiyonları | `printf`, `malloc` |
| 4 | Aygıt dosyaları | `/dev/null` |
| 5 | Dosya biçimleri ve yapılandırma dosyaları | `hosts`, `group`, `crontab` |
| 7 | Çeşitli konular, kavramlar | `regex`, `signal` |
| 8 | Sistem yönetimi komutları (genelde root) | `sysctl`, `mandb` |

Aynı ada sahip birden fazla sayfa olabilir. `printf` hem bir komut (bölüm 1) hem bir C fonksiyonu (bölüm 3):

```bash
man -f printf      # veya: whatis printf
```
```
printf (1)           - format and print data
printf (3)           - formatted output conversion
```

`man printf` yazınca ilk bulunan, yani bölüm 1 açılır. Diğerini açmak için bölüm numarasını ver:

```bash
man 3 printf
```

Bölüm 5 yönetim işlerinde çok işe yarar, çünkü yapılandırma dosyalarının sözdizimi oradadır:

```bash
man 5 hosts          # /etc/hosts dosyasının biçimi
man 5 crontab        # crontab satırlarının biçimi
man sshd_config      # SSH sunucusu ayarları (bölüm 5)
```

Bir sayfanın hangi dosyada durduğunu görmek için:

```bash
man -w 3 printf
```
```
/usr/share/man/man3/printf.3.gz
```

---

## 5. apropos: adını bilmediğin komutu bulmak

"Dizin oluşturan komut neydi?" gibi durumlarda `apropos`, bütün man sayfalarının `NAME` satırlarında anahtar kelime arar:

```bash
apropos director
```
```
alphasort (3)        - scan a directory for matching entries
basename (1)         - strip directory and suffix from filenames
chdir (2)            - change working directory
chroot (8)           - run command or interactive shell with special root dir...
cp (1)               - copy files and directories
...
```

Sonuçlar arasında C fonksiyonları ve sistem çağrıları da var. Sadece komutları görmek için bölümleri sınırla:

```bash
apropos -s 1,8 director
```
```
basename (1)         - strip directory and suffix from filenames
chroot (8)           - run command or interactive shell with special root dir...
cp (1)               - copy files and directories
dir (1)              - list directory contents
ls (1)               - list directory contents
mkdir (1)            - make directories
...
```

`director` yazmak hem "directory" hem "directories" kelimelerini yakalar. Birden fazla kelimeyi tırnak içinde aramak da mümkün:

```bash
apropos 'list directory'
```

`man -k` ile `apropos` aynı işi yapar.

### "nothing appropriate"

`apropos` hiçbir şey bulamazsa:

```
xyzabcqq: nothing appropriate.
```

Bu mesaj gerçekten bir şey olmadığını da gösterebilir, ama sistemi yeni kurduysan sebep **man veritabanının henüz oluşturulmamış** olması olabilir. `apropos` sayfaları tek tek okumaz, önceden oluşturulmuş bir dizinde arar. Dizini güncellemek için:

```bash
sudo mandb
```

Ubuntu'da bu dizin bir zamanlayıcıyla düzenli olarak güncellenir, ama yeni kurulan bir sistemde veya yeni bir paket kurduktan hemen sonra elle çalıştırmak gerekebilir.

---

## 6. Shell'e yerleşik komutlar: help

`cd`, `export`, `alias` gibi bazı komutlar ayrı programlar değil, bash'in kendi içindedir. Bunların ayrı bir man sayfası yoktur:

```bash
type cd
```
```
cd is a shell builtin
```

Yerleşik komutlar için bash'in `help` komutu kullanılır:

```bash
help cd
```
```
cd: cd [-L|[-P [-e]] [-@]] [dir]
    Change the shell working directory.
```

Bir komutun ne olduğunu bilmiyorsan önce `type komut` ile bak: `builtin` ise `help`, normal bir programsa (`ls is /usr/bin/ls`) `man` ve `--help`.

---

## 7. Tab tamamlama

Tab tuşu yazmayı tamamlar ve aynı zamanda bir hatırlatma aracıdır:

| Yazdığın | Tab'a basınca |
|---|---|
| `system` + Tab | Tek seçenek varsa tamamlar: `systemctl` |
| `systemctl ` + Tab Tab | Alt komutları listeler: `start`, `stop`, `status`, `enable`... |
| `ls /usr/` + Tab Tab | Dizindeki dosya ve alt dizinleri listeler |
| `cat /etc/host` + Tab Tab | `host.conf`, `hostname`, `hosts` gibi eşleşenleri listeler |

Tek seçenek varsa bir Tab yeter, birden fazla seçenek varsa ikinci Tab listeyi gösterir. Alt komutların ve servis adlarının tamamlanması `bash-completion` paketiyle gelir. Normal Ubuntu kurulumlarında bulunur, minimal kurulumlarda yoksa `sudo apt install bash-completion` ile eklenir.

Tab'ı alışkanlık haline getir. Hem yazım hatalarını önler hem de bir dosyanın veya komutun gerçekten var olup olmadığını anında gösterir.

---

## 8. Başka yardım kaynakları

| Kaynak | Ne zaman? |
|---|---|
| `/usr/share/doc/PAKET/` | Paketle gelen belgeler ve örnek yapılandırmalar. Örneğin `/usr/share/doc/netplan/examples/` ([Ağ Yapılandırması](../02-ag-yonetimi/02-ag-yapilandirmasi.md#11-yardım-kaynakları)) |
| Yapılandırma dosyalarının içindeki açıklamalar | `/etc/ssh/sshd_config` gibi dosyalarda varsayılan değerler yorum satırı olarak yazılı |
| `man ufw-framework` gibi ek sayfalar | Bazı araçların ayrı örnek sayfaları vardır ([NAT](../02-ag-yonetimi/07-nat-ve-port-yonlendirme.md#8-ufw-ile-nat-beforerules-ve-route-kuralları)) |
| `info komut` | GNU araçlarının daha uzun belgeleri |

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `komut --help` | Kısa kullanım özeti |
| `komut --help \| grep kelime` | Yardımda belirli bir seçeneği arar |
| `man komut` | Komutun kılavuzu |
| `man 5 dosya` | Belirli bölümdeki sayfayı açar |
| `man -f ad` / `whatis ad` | Adın hangi bölümlerde sayfası olduğunu gösterir |
| `man -w ad` | Sayfanın dosya yolunu gösterir |
| `apropos kelime` / `man -k kelime` | Açıklamasında kelime geçen sayfalar |
| `apropos -s 1,8 kelime` | Sadece komutlarda arar |
| `sudo mandb` | apropos'un kullandığı veritabanını günceller |
| `type komut` | Komutun yerleşik mi program mı olduğunu gösterir |
| `help komut` | Bash'e yerleşik komutun yardımı |
| Tab / Tab Tab | Tamamlar / seçenekleri listeler |
| man içinde `/kelime`, `n`, `q` | Ara, sonraki eşleşme, çık |

---

## 10. Sık yapılan hatalar

- **`-h`'yi yardım sanmak.** Birçok komutta başka anlama gelir (`ls -h`, `df -h`). `--help` kullan.
- **Aynı adlı sayfalardan yanlışını okumak.** `man printf` komutu açar, C fonksiyonu için `man 3 printf`.
- **apropos "nothing appropriate" deyince pes etmek.** Önce `sudo mandb`, sonra daha kısa veya farklı bir kelime dene.
- **Yerleşik komutlar için man aramak.** `cd`, `export` için `help` kullan.
- **Man sayfasını baştan sona okumak.** `/` ile ara, `EXAMPLES` bölümüne atla.
- **Yapılandırma dosyalarının sözdizimini ezberlemeye çalışmak.** `man 5 ...`, dosyanın içindeki yorumlar ve `/usr/share/doc` örnekleri her zaman elinin altında.

---

## 11. Alıştırmalar

**1.** `cp` komutunun, kopyalarken dosya izinlerini ve zaman bilgisini koruyan seçeneğini man sayfasına bakmadan, `--help` çıktısında bul.

<details>
<summary>Çözüm</summary>

```bash
cp --help | grep -i preserve
```

`-p` (`--preserve=mode,ownership,timestamps`).
</details>

**2.** Adını hatırlamadığın, dosya kopyalayan kullanıcı komutlarını listele.

<details>
<summary>Çözüm</summary>

```bash
apropos -s 1 copy
```
```
cp (1)               - copy files and directories
dd (1)               - convert and copy a file
install (1)          - copy files and set attributes
...
```

Sonuç çıkmazsa önce `sudo mandb` çalıştır.
</details>

**3.** `/etc/group` dosyasının hangi alanlardan oluştuğunu öğrenmek için hangi man sayfasını açarsın?

<details>
<summary>Çözüm</summary>

```bash
man -f group      # group (5) - user group file
man 5 group
```
</details>

**4.** `ls` man sayfasını aç, boyuta göre sıralama seçeneğini arama yaparak bul ve çık.

<details>
<summary>Çözüm</summary>

```bash
man ls
```

Sayfa açılınca `/size` yazıp Enter'a bas, `n` ile sonraki eşleşmelere geç. `-S` seçeneğini (`sort by file size, largest first`) bulunca `q` ile çık.
</details>

**5.** `export` komutunun seçeneklerini öğren. `man export` neden işe yaramaz?

<details>
<summary>Çözüm</summary>

```bash
type export      # export is a shell builtin
help export
```

`export` bash'e yerleşik bir komut, ayrı bir program olmadığı için kendi man sayfası yok.
</details>

---

Bölüm ana sayfası: [Temel Komutlar](README.md) · Sonraki konu: [Dosya ve Dizin İşlemleri](02-dosya-ve-dizin-islemleri.md)
