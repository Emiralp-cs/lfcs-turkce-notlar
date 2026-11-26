# Bash Script Temelleri

> Güncel resmi listede ayrı bir madde değil. Ama zamanlanmış görevler, servisler ve yedekleme işleri hep bir script çalıştırır. Basit bir script yazıp okuyabilmek sınavın birçok görevinde işine yarar.

Bir **script**, sırayla çalıştırılacak komutları içeren bir metin dosyasıdır. Elle tekrar tekrar yazdığın komutları bir dosyaya koyup tek komutla çalıştırmanı, cron veya systemd ile zamanlamanı sağlar.

Senaryo: `/etc/ssh` dizinini yedekleyen ve bir önceki yedeği saklayan bir script, bir de diskler belirli bir doluluğu geçince uyaran bir kontrol script'i yazacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve bash 5.2 ile denenmiştir.

## İçindekiler

1. [Basit bir script](#1-basit-bir-script)
2. [Shebang satırı](#2-shebang-satırı)
3. [Çalıştırma izni ve çalıştırma yolları](#3-çalıştırma-izni-ve-çalıştırma-yolları)
4. [Değişkenler ve argümanlar](#4-değişkenler-ve-argümanlar)
5. [Çıkış kodları](#5-çıkış-kodları)
6. [Koşullar: if ve test](#6-koşullar-if-ve-test)
7. [Döngüler: for ve while](#7-döngüler-for-ve-while)
8. [Örnek: yedek döndürmeli yedekleme](#8-örnek-yedek-döndürmeli-yedekleme)
9. [Örnek: disk doluluk kontrolü](#9-örnek-disk-doluluk-kontrolü)
10. [Hata ayıklamak](#10-hata-ayıklamak)
11. [Yerleşik komutlar: help](#11-yerleşik-komutlar-help)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. Basit bir script

```bash
nano ilk.sh
```

```bash
#!/bin/bash
# Sistemin cekirdek surumunu ve tarihi bir log dosyasina ekler
date >> /tmp/sistem.log
uname -r >> /tmp/sistem.log
```

```bash
chmod +x ilk.sh
./ilk.sh
cat /tmp/sistem.log
```

- İlk satır (`#!/bin/bash`) script'i hangi programın çalıştıracağını söyler.
- `#` ile başlayan diğer satırlar açıklamadır, çalıştırılmaz.
- `>>` çıktıyı dosyanın sonuna ekler ([Yönlendirme](../04-temel-komutlar/10-yonlendirme.md)).

`.sh` uzantısı sadece alışkanlıktır, Linux uzantıya bakmaz. Ama script'i `/etc/cron.daily/`'ye koyacaksan uzantısız olmalı ([Zamanlanmış Görevler](05-zamanlanmis-gorevler.md#4-hazır-dizinler-etccrondaily-ve-diğerleri)).

---

## 2. Shebang satırı

`#!` ile başlayan ilk satıra **shebang** denir.

| Shebang | Anlamı |
|---|---|
| `#!/bin/bash` | bash ile çalıştır. bash'e özgü özellikler kullanılabilir |
| `#!/bin/sh` | POSIX uyumlu kabukla çalıştır. Ubuntu'da bu `dash`'tir, bash değil |
| `#!/usr/bin/env bash` | bash'i PATH'te ara. bash'in farklı yerlerde olabileceği sistemlerde |
| `#!/usr/bin/python3` | Python script'i |

### Shebang dosyanın ilk satırında olmalı

Dosyanın başına yanlışlıkla bir boş satır koyduk, shebang ikinci satırda kaldı:

| Script nasıl başlatıldı? | Sonuç |
|---|---|
| bash'ten `./bosluklu.sh` | Çalıştı (bash, shebang'ı geçersiz dosyayı kendisi yorumladı) |
| `sh -c ./bosluklu.sh` (cron böyle çalıştırır) | `[[: not found`: dosya `sh` ile yorumlandı |
| Doğrudan başka bir programdan (systemd gibi) | `Exec format error` |

Yani terminalde çalışan bir script cron'da veya servis olarak çalışmayabilir. Shebang her zaman dosyanın ilk satırının ilk karakterinden başlamalı. `#! /bin/bash` (ünlemden sonra boşluk) ise sorunsuz çalıştı.

---

## 3. Çalıştırma izni ve çalıştırma yolları

```bash
./izinsiz.sh
```
```
bash: ./izinsiz.sh: Permission denied
```

Script'i doğrudan çalıştırmak için **çalıştırma izni** gerekir: `chmod +x script.sh` ([İzinler](../04-temel-komutlar/04-izinler-ve-ozel-bitler.md)).

| Yöntem | Çalıştırma izni gerekir mi? | Shebang kullanılır mı? |
|---|---|---|
| `./script.sh` | Evet | Evet |
| `/tam/yol/script.sh` | Evet | Evet |
| `bash script.sh` | Hayır | Hayır, bash ile çalışır |
| `source script.sh` veya `. script.sh` | Hayır | Hayır, **bulunduğun kabukta** çalışır |

`source` ile çalışan bir script'in tanımladığı değişkenler ve yaptığı `cd` senin kabuğunda kalır. `~/.bashrc` gibi dosyalar böyle okunur ([Ortam Profilleri](../05-kullanicilar-ve-gruplar/04-ortam-profilleri.md)).

Script'i her yerden adıyla çalıştırmak için PATH'teki bir dizine koy: `/usr/local/bin/` (herkes için) veya `~/.local/bin/` (sadece sen).

---

## 4. Değişkenler ve argümanlar

```bash
HEDEF=/tmp/yedek.tar.gz        # = etrafında boşluk YOK
echo "$HEDEF"
echo "${HEDEF}.eski"           # süslü parantez: değişkenin nerede bittiğini belirtir
TARIH=$(date +%F)              # komutun çıktısını değişkene al
```

> [!WARNING]
> `HEDEF = /tmp/...` (boşluklu) yazarsan bash `HEDEF` adında bir komut çalıştırmaya çalışır: `HEDEF: command not found`.

Değişkenleri her zaman **çift tırnak** içinde kullan: `"$HEDEF"`. Değer boşluk içeriyorsa tırnaksız kullanım onu birden fazla parçaya böler.

### Argümanlar

Script'e verilen argümanlar özel değişkenlerle okunur:

| Değişken | Anlamı |
|---|---|
| `$0` | Script'in adı |
| `$1`, `$2` ... | 1., 2. ... argüman |
| `$#` | Argüman sayısı |
| `$@` | Bütün argümanlar |
| `$?` | Son komutun çıkış kodu |

```bash
ESIK=${1:-80}     # 1. argüman verilmediyse 80 kullan
```

---

## 5. Çıkış kodları

Her komut bittiğinde bir **çıkış kodu** döndürür: **0 başarılı**, 0 dışındaki her değer hata demektir.

```bash
grep -q 'PermitRootLogin' /etc/ssh/sshd_config; echo $?    # 0: bulundu
grep -q 'YOKBOYLE' /etc/ssh/sshd_config; echo $?           # 1: bulunamadı
```

`-q` (quiet), `grep`'in hiçbir şey yazmamasını, sadece çıkış koduyla cevap vermesini sağlar. Script'lerde "var mı?" sorusu için idealdir.

Script'in kendi çıkış kodunu `exit` belirler:

```bash
exit 0     # başarılı
exit 1     # hata
```

`exit` yazılmazsa script, son çalışan komutun çıkış kodunu döndürür. Cron, systemd ve diğer script'ler senin script'inin başarılı olup olmadığını bu koddan anlar.

### Komutları koşullu bağlamak

```bash
mkdir -p /yedek && cp dosya /yedek/      # ilki başarılıysa ikincisi
ping -c1 sunucu || echo "sunucuya ulasilamiyor"   # ilki başarısızsa ikincisi
```

---

## 6. Koşullar: if ve test

```bash
if grep -q 'PermitRootLogin yes' /etc/ssh/sshd_config; then
    echo "UYARI: root girisi acik"
else
    echo "root girisi kapali"
fi
```

`if`, ardından gelen **komutun çıkış koduna** bakar: 0 ise `then` kısmı çalışır. En sık kullanılan komut `test`'tir, kısaca `[ ]` olarak yazılır:

```bash
if [ -f /tmp/yedek.tar.gz ]; then
    echo "yedek var"
fi
```

`[` ve `]`'nin içinde, kenarlarında **boşluk** olmalı: `[-f dosya]` çalışmaz.

| Test | Doğruysa |
|---|---|
| `-f dosya` | Normal dosya var |
| `-d dizin` | Dizin var |
| `-e yol` | Herhangi bir şey var |
| `-x dosya` | Çalıştırılabilir |
| `-s dosya` | Var ve boş değil |
| `-z "$a"` | Değişken boş |
| `"$a" = "$b"` | Metinler eşit |
| `"$a" != "$b"` | Metinler farklı |
| `$a -eq $b` | Sayılar eşit |
| `$a -ne $b`, `-lt`, `-le`, `-gt`, `-ge` | Eşit değil, küçük, küçük eşit, büyük, büyük eşit |
| `! test` | Tersi |

Bash'te `[[ ]]` de vardır: daha esnektir (`&&`, `||`, desen eşleşmesi kullanılabilir), ama `sh`'ta çalışmaz. Shebang `#!/bin/bash` ise kullanabilirsin.

`man test` ve `help [[` ile bütün testler görülebilir.

---

## 7. Döngüler: for ve while

```bash
for SUNUCU in web1 web2 db1; do
    ping -c1 -W1 "$SUNUCU" > /dev/null && echo "$SUNUCU ayakta"
done

for DOSYA in /var/log/*.log; do
    echo "$DOSYA: $(wc -l < "$DOSYA") satir"
done

for i in {1..5}; do echo "$i"; done        # bash'e özgü

while read -r KULLANICI; do
    echo "kullanici: $KULLANICI"
done < kullanicilar.txt
```

`continue` döngünün o turunu atlar, `break` döngüden çıkar.

---

## 8. Örnek: yedek döndürmeli yedekleme

Her çalıştığında `/etc/ssh`'ı yedekler, varsa bir önceki yedeği `.eski` olarak saklar:

```bash
#!/bin/bash
# /etc/ssh dizinini yedekler, bir onceki yedegi .eski olarak saklar
HEDEF=/tmp/ssh-yedek.tar.gz

if [ -f "$HEDEF" ]; then
    mv "$HEDEF" "$HEDEF.eski"
fi

tar -czf "$HEDEF" /etc/ssh 2>/dev/null
echo "$(date '+%F %T') yedek alindi: $HEDEF" >> /tmp/yedek.log
```

İki kez çalıştırdık:

```bash
ls /tmp/ssh-yedek*
cat /tmp/yedek.log
```
```
/tmp/ssh-yedek.tar.gz
/tmp/ssh-yedek.tar.gz.eski
2026-11-26 09:26:23 yedek alindi: /tmp/ssh-yedek.tar.gz
2026-11-26 09:26:23 yedek alindi: /tmp/ssh-yedek.tar.gz
```

Bu kontrol olmasaydı, her çalıştırma bir önceki yedeğin üzerine yazardı ve yedek bozuk çıkarsa geri dönecek bir şey kalmazdı. `tar`'ın ayrıntısı: [Arşivleme ve Sıkıştırma](../04-temel-komutlar/08-arsivleme-ve-sikistirma.md). `2>/dev/null`, tar'ın "baştaki `/` kaldırıldı" uyarısını gizler.

Her gece çalıştırmak için script'i `/usr/local/bin/yedek-ssh` olarak kaydedip bir cron satırı veya systemd timer ekle.

---

## 9. Örnek: disk doluluk kontrolü

```bash
#!/bin/bash
# Kullanim: disk-kontrol.sh [esik]   (varsayilan 80)
ESIK=${1:-80}

for BAGLAMA in / /boot /var; do
    [ -d "$BAGLAMA" ] || continue
    KULLANIM=$(df --output=pcent "$BAGLAMA" | tail -1 | tr -dc '0-9')
    if [ "$KULLANIM" -ge "$ESIK" ]; then
        echo "UYARI: $BAGLAMA %$KULLANIM dolu (esik %$ESIK)"
        DURUM=1
    else
        echo "tamam: $BAGLAMA %$KULLANIM"
    fi
done
exit ${DURUM:-0}
```

```bash
./disk-kontrol.sh;    echo "cikis=$?"
./disk-kontrol.sh 20; echo "cikis=$?"
```
```
tamam: / %31
tamam: /boot %31
tamam: /var %31
cikis=0
UYARI: / %31 dolu (esik %20)
UYARI: /boot %31 dolu (esik %20)
UYARI: /var %31 dolu (esik %20)
cikis=1
```

Script hem okunabilir bir çıktı veriyor hem de çıkış koduyla durumu bildiriyor: bir izleme sistemi veya başka bir script sadece koda bakarak karar verebilir. `df --output=pcent` sadece yüzde sütununu verir, `tr -dc '0-9'` rakam dışındaki her şeyi siler.

---

## 10. Hata ayıklamak

```bash
bash -n script.sh     # sadece sözdizimini kontrol et, çalıştırma
bash -x script.sh     # her komutu çalıştırmadan önce ekrana yaz
```

`bash -x` çıktısında her satırın değişkenler yerine konmuş hali `+` ile görünür:

```
+ ESIK=95
+ for BAGLAMA in / /boot /var
+ '[' -d / ']'
```

Script'in belirli bir kısmını izlemek için içine `set -x` (başlat) ve `set +x` (durdur) yazılabilir.

Script'lerde işe yarayan güvenlik ayarları:

| Ayar | Ne yapar? |
|---|---|
| `set -e` | Bir komut hata verirse script'i durdur |
| `set -u` | Tanımlanmamış bir değişken kullanılırsa hata ver |
| `set -o pipefail` | Borudaki herhangi bir komut hata verirse borunun tamamı hatalı sayılsın |

---

## 11. Yerleşik komutlar: help

`cd`, `echo`, `if`, `for`, `test`, `alias` gibi bazı komutlar ayrı programlar değil, **bash'in içindedir** (builtin). Bunların `man` sayfası yoktur veya genel bash sayfasındadır:

```bash
type cd echo ls
```
```
cd is a shell builtin
echo is a shell builtin
ls is /usr/bin/ls
```

Yerleşik komutların listesi ve yardımı:

```bash
help             # bütün yerleşik komutlar
help if          # if'in kullanımı
help test
```

Örnek script bulmak için sistemin kendi script'lerine bakmak öğreticidir: `/etc/cron.daily/` altındakiler gerçek hayatta kullanılan script'lerdir.

---

## 12. Komut özeti

| Yazım | Anlamı |
|---|---|
| `#!/bin/bash` | Shebang, ilk satırın başında |
| `chmod +x script.sh`, `./script.sh` | İzin ver, çalıştır |
| `bash script.sh` | İzin olmadan çalıştır |
| `source script.sh` | Bulunduğun kabukta çalıştır |
| `AD=değer`, `"$AD"`, `${AD}` | Değişken |
| `$(komut)` | Komutun çıktısı |
| `$1`, `$#`, `$@`, `$?` | Argümanlar, sayısı, hepsi, son çıkış kodu |
| `${1:-80}` | Argüman yoksa varsayılan |
| `exit 0` / `exit 1` | Başarılı / hatalı çık |
| `komut1 && komut2`, `komut1 \|\| komut2` | Koşullu çalıştır |
| `if [ -f dosya ]; then ... else ... fi` | Koşul |
| `for X in a b c; do ... done` | Döngü |
| `while read -r X; do ... done < dosya` | Satır satır oku |
| `bash -n`, `bash -x` | Sözdizimi kontrolü, adım adım çalıştır |
| `set -euo pipefail` | Hatada dur |
| `help`, `type` | Yerleşik komutlar |

---

## 13. Sık yapılan hatalar

- **Shebang'ı ilk satıra koymamak.** Terminalde çalışır, cron'da ve serviste çalışmaz.
- **`chmod +x` unutmak.** `Permission denied`.
- **`=` etrafına boşluk koymak.** `AD = değer` bir komut çalıştırmaya çalışır.
- **Değişkenleri tırnaksız kullanmak.** Boşluklu değerler bölünür.
- **`[` ve `]` kenarlarında boşluk bırakmamak.**
- **`#!/bin/sh` yazıp `[[ ]]`, `{1..5}` gibi bash özellikleri kullanmak.**
- **Script'in çıkış kodunu belirlememek.** Hata olsa bile cron "başarılı" sanabilir.
- **Yedekleme script'lerinde eski yedeğin üzerine yazmak.**
- **Script'te göreli yol kullanmak.** Cron'dan çalışınca çalışma dizini farklıdır, tam yol yaz.

---

## 14. Alıştırmalar

**1.** Çalıştırıldığında tarih, saat ve sistemin çalışma süresini (`uptime -p`) `/tmp/durum.log` dosyasına **ekleyen** `durum.sh` adında bir script yaz ve çalıştır.

<details>
<summary>Çözüm</summary>

```bash
cat > durum.sh <<'EOF'
#!/bin/bash
echo "$(date '+%F %T') $(uptime -p)" >> /tmp/durum.log
EOF
chmod +x durum.sh
./durum.sh
cat /tmp/durum.log
```
</details>

**2.** Argüman olarak bir kullanıcı adı alan, kullanıcı sistemde varsa "var", yoksa "yok" yazan ve buna göre 0 veya 1 ile çıkan `kullanici-var-mi.sh` yaz.

<details>
<summary>Çözüm</summary>

```bash
cat > kullanici-var-mi.sh <<'EOF'
#!/bin/bash
if id "$1" > /dev/null 2>&1; then
    echo "var"
    exit 0
else
    echo "yok"
    exit 1
fi
EOF
chmod +x kullanici-var-mi.sh
./kullanici-var-mi.sh root; echo $?
./kullanici-var-mi.sh boyleyok; echo $?
```
</details>

**3.** `/etc/apt` dizinini `/tmp/apt-yedek.tar.gz` olarak arşivleyen, dosya zaten varsa önce onu `.eski` uzantısıyla saklayan bir script yaz. Arşivin içeriğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
cat > apt-yedek.sh <<'EOF'
#!/bin/bash
HEDEF=/tmp/apt-yedek.tar.gz
[ -f "$HEDEF" ] && mv "$HEDEF" "$HEDEF.eski"
tar -czf "$HEDEF" /etc/apt 2>/dev/null
EOF
chmod +x apt-yedek.sh
./apt-yedek.sh; ./apt-yedek.sh
ls /tmp/apt-yedek*
tar -tzf /tmp/apt-yedek.tar.gz | head
```
</details>

**4.** `/etc/ssh/sshd_config` içinde yorum olmayan bir `PermitRootLogin` satırı varsa onu yazdıran, yoksa "varsayilan ayar kullaniliyor" yazan bir script yaz.

<details>
<summary>Çözüm</summary>

```bash
cat > root-kontrol.sh <<'EOF'
#!/bin/bash
if grep -q '^PermitRootLogin' /etc/ssh/sshd_config; then
    grep '^PermitRootLogin' /etc/ssh/sshd_config
else
    echo "varsayilan ayar kullaniliyor"
fi
EOF
chmod +x root-kontrol.sh
./root-kontrol.sh
```
</details>

**5.** Bir script'in sözdiziminde hata olup olmadığını çalıştırmadan nasıl kontrol edersin? `then` kelimesini sildiğin bir `if` ile dene.

<details>
<summary>Çözüm</summary>

```bash
printf '#!/bin/bash\nif [ -f /etc/passwd ]\n  echo var\nfi\n' > hatali.sh
bash -n hatali.sh       # syntax error near unexpected token `fi'
```
</details>

---

Önceki konu: [Sanal Makineler](12-sanal-makineler.md) · Bölüm ana sayfası: [İşletim ve Dağıtım](README.md)
