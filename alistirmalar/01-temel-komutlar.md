# 01 - Temel Komutlar: Alıştırmalar

Temel Komutlar bölümündeki 16 konunun bütün alıştırmaları ve çözümleri, toplam **80 soru**. Sorular konu sırasıyla diziliyor.

Çözümler kapalı gelir. Önce kendi sanal makinende dene, sonra **Çözüm**'e tıklayıp karşılaştır. Takıldığın bir konuyu tekrar okumak için başlığın altındaki bağlantıyı kullan.

| # | Konu | Soru |
|---|---|---|
| 01 | [Yardım Almak: --help, man ve apropos](../01-temel-komutlar/01-yardim-almak.md) | 5 |
| 02 | [Dosya ve Dizin İşlemleri](../01-temel-komutlar/02-dosya-ve-dizin-islemleri.md) | 5 |
| 03 | [Hard ve Soft Link](../01-temel-komutlar/03-hard-ve-soft-link.md) | 5 |
| 04 | [Dosya İzinleri, Sahiplik ve Özel Bitler](../01-temel-komutlar/04-izinler-ve-ozel-bitler.md) | 5 |
| 05 | [find ile Dosya Arama](../01-temel-komutlar/05-find.md) | 5 |
| 06 | [Metin İşleme: head, tail, sed, cut, sort, uniq, diff](../01-temel-komutlar/06-metin-isleme.md) | 5 |
| 07 | [grep ve Düzenli İfadeler](../01-temel-komutlar/07-grep-ve-regex.md) | 5 |
| 08 | [Arşivleme ve Sıkıştırma](../01-temel-komutlar/08-arsivleme-ve-sikistirma.md) | 5 |
| 09 | [Yedekleme: rsync ve dd](../01-temel-komutlar/09-yedekleme-rsync-ve-dd.md) | 5 |
| 10 | [Girdi/Çıktı Yönlendirme](../01-temel-komutlar/10-yonlendirme.md) | 5 |
| 11 | [SSL/TLS ve OpenSSL](../01-temel-komutlar/11-ssl-tls-ve-openssl.md) | 5 |
| 12 | [Git ile Sürüm Kontrolü](../01-temel-komutlar/12-git.md) | 5 |
| 13 | [Servis Oluşturma ve Sorun Giderme](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md) | 5 |
| 14 | [Performans İzleme](../01-temel-komutlar/14-performans-izleme.md) | 5 |
| 15 | [Disk Alanı Sorunları](../01-temel-komutlar/15-disk-alani-sorunlari.md) | 5 |
| 16 | [Uygulama ve Servis Kısıtlamaları](../01-temel-komutlar/16-uygulama-ve-servis-kisitlamalari.md) | 5 |

---

## 01 - Yardım Almak: --help, man ve apropos

Konu notu: [Yardım Almak: --help, man ve apropos](../01-temel-komutlar/01-yardim-almak.md)

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

## 02 - Dosya ve Dizin İşlemleri

Konu notu: [Dosya ve Dizin İşlemleri](../01-temel-komutlar/02-dosya-ve-dizin-islemleri.md)

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

## 03 - Hard ve Soft Link

Konu notu: [Hard ve Soft Link](../01-temel-komutlar/03-hard-ve-soft-link.md)

**1.** `/tmp/linkler/veri.txt` dosyasını oluştur. Aynı dizinde ona `ikinci.txt` adında bir hard link oluştur. İki adın aynı inode'u kullandığını ve bağlantı sayısının 2 olduğunu göster. Sonra `veri.txt`'yi sil ve içeriğin hala okunabildiğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
mkdir -p /tmp/linkler
echo "onemli veri" > /tmp/linkler/veri.txt
ln /tmp/linkler/veri.txt /tmp/linkler/ikinci.txt
ls -li /tmp/linkler/
rm /tmp/linkler/veri.txt
cat /tmp/linkler/ikinci.txt
stat -c '%h' /tmp/linkler/ikinci.txt     # 1
```
</details>

**2.** `/tmp/linkler/kisayol` adında, `/etc/ssh` dizinini gösteren bir soft link oluştur. Linkin gösterdiği gerçek yolu yazdır.

<details>
<summary>Çözüm</summary>

```bash
ln -s /etc/ssh /tmp/linkler/kisayol
ls -l /tmp/linkler/kisayol
readlink -f /tmp/linkler/kisayol
```
</details>

**3.** `/tmp/linkler` içinde kırık bir soft link oluştur ve `find` ile bul.

<details>
<summary>Çözüm</summary>

```bash
ln -s /tmp/linkler/yok.txt /tmp/linkler/kirik
find /tmp/linkler -xtype l
```
</details>

**4.** `/tmp/linkler/surum1` ve `/tmp/linkler/surum2` dizinlerini oluştur. `aktif` adlı bir link önce `surum1`'i göstersin, sonra `surum2`'yi gösterecek şekilde güncelle. Sonucu doğrula.

<details>
<summary>Çözüm</summary>

```bash
cd /tmp/linkler
mkdir surum1 surum2
ln -s surum1 aktif
ln -sfn surum2 aktif
ls -l aktif          # aktif -> surum2
ls surum1            # içinde yanlışlıkla "surum2" linki olmamalı
```
</details>

**5.** Aşağıdaki komutlardan hangisi `kisayol` linkinin gösterdiği dizinin içini siler, hangisi sadece linki siler? Sonra bütün alıştırma dizinini temizle.

a) `rm kisayol`
b) `rm -rf kisayol/`

<details>
<summary>Çözüm</summary>

a) Sadece linki siler.
b) Hedef dizinin (`/etc/ssh`!) içindeki her şeyi silmeye çalışır. **Bunu deneme.**

```bash
rm /tmp/linkler/kisayol
rm -r /tmp/linkler
```
</details>

---

## 04 - Dosya İzinleri, Sahiplik ve Özel Bitler

Konu notu: [Dosya İzinleri, Sahiplik ve Özel Bitler](../01-temel-komutlar/04-izinler-ve-ozel-bitler.md)

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

## 05 - find ile Dosya Arama

Konu notu: [find ile Dosya Arama](../01-temel-komutlar/05-find.md)

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

## 06 - Metin İşleme: head, tail, sed, cut, sort, uniq, diff

Konu notu: [Metin İşleme: head, tail, sed, cut, sort, uniq, diff](../01-temel-komutlar/06-metin-isleme.md)

Alıştırmalar için [konu notunun](../01-temel-komutlar/06-metin-isleme.md) başındaki `personel.csv` dosyasını oluştur.

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

`grep -v` ile aynı iş [grep](../01-temel-komutlar/07-grep-ve-regex.md) dosyasında.
</details>

---

## 07 - grep ve Düzenli İfadeler

Konu notu: [grep ve Düzenli İfadeler](../01-temel-komutlar/07-grep-ve-regex.md)

Alıştırmalar için [konu notunun](../01-temel-komutlar/07-grep-ve-regex.md) başındaki `uygulama.log` dosyasını oluştur.

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

## 08 - Arşivleme ve Sıkıştırma

Konu notu: [Arşivleme ve Sıkıştırma](../01-temel-komutlar/08-arsivleme-ve-sikistirma.md)

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

## 09 - Yedekleme: rsync ve dd

Konu notu: [Yedekleme: rsync ve dd](../01-temel-komutlar/09-yedekleme-rsync-ve-dd.md)

Alıştırmalar için bir `site/` dizini oluştur:

```bash
mkdir -p site/css
echo '<h1>Merhaba</h1>' > site/index.html
echo 'body{}' > site/css/stil.css
seq 1 200000 > site/kayit.log
```

**1.** `site/` dizininin içeriğini `/tmp/yedek/site/` dizinine eşitle. `/tmp/yedek` henüz yok. Sonra `/tmp/yedek/site/` altında `css`, `index.html` ve `kayit.log`'un bulunduğunu, fazladan bir `site/` dizini oluşmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
rsync -av --mkpath site/ /tmp/yedek/site/
ls /tmp/yedek/site
```

`--mkpath` yerine önce `mkdir -p /tmp/yedek` de olur. `site/` yerine `site` yazsaydın sonuç `/tmp/yedek/site/site/` olurdu.
</details>

**2.** `site/index.html` dosyasına bir satır ekle ve 1. alıştırmadaki komutu tekrar çalıştır. Hangi dosyalar aktarıldı?

<details>
<summary>Çözüm</summary>

```bash
echo '<p>guncelleme</p>' >> site/index.html
rsync -av site/ /tmp/yedek/site/
```

Listede sadece `index.html` görünür. Diğer dosyalar değişmediği için atlanır.
</details>

**3.** `site/css/stil.css`'i sil. Önce yedekten de silinecek dosyaları hiçbir şeyi değiştirmeden göster, sonra gerçekten eşitle.

<details>
<summary>Çözüm</summary>

```bash
rm site/css/stil.css
rsync -avn --delete site/ /tmp/yedek/site/   # deleting css/stil.css ... (DRY RUN)
rsync -av --delete site/ /tmp/yedek/site/
ls /tmp/yedek/site/css                        # boş
```
</details>

**4.** dd ile sıfırlarla dolu 20 MiB'lık `disk.img` dosyası oluştur. Sonra bu dosyanın bayt bayt kopyasını `disk-kopya.img` olarak al ve ikisinin aynı olduğunu doğrula.

<details>
<summary>Çözüm</summary>

```bash
dd if=/dev/zero of=disk.img bs=1M count=20
dd if=disk.img of=disk-kopya.img bs=1M status=progress
cmp disk.img disk-kopya.img && echo AYNI
sha256sum disk.img disk-kopya.img
```

`cmp` hiçbir şey yazmaz ve `AYNI` çıkar, iki özet de aynıdır.
</details>

**5.** `disk.img` dosyasının gzip ile sıkıştırılmış imajını tek komutta `disk.img.gz` olarak al, sonra bu imajı `geri.img` olarak aç ve orijinalle karşılaştır.

<details>
<summary>Çözüm</summary>

```bash
dd if=disk.img bs=1M status=none | gzip > disk.img.gz
ls -lh disk.img disk.img.gz
gunzip -c disk.img.gz | dd of=geri.img bs=1M status=none
cmp disk.img geri.img && echo AYNI
```

Sıfırlarla dolu dosya çok iyi sıkışır: 20 MiB, birkaç on KB'a iner.
</details>

---

## 10 - Girdi/Çıktı Yönlendirme

Konu notu: [Girdi/Çıktı Yönlendirme](../01-temel-komutlar/10-yonlendirme.md)

**1.** Normal kullanıcıyla `/etc` altında adı `.conf` ile biten dosyaları ara. Sonuçları `confs.txt`'ye, hataları `izin-hatalari.txt`'ye yaz. Ekrana hiçbir şey basılmasın. Sonra iki dosyanın satır sayısına bak.

<details>
<summary>Çözüm</summary>

```bash
find /etc -name '*.conf' > confs.txt 2> izin-hatalari.txt
wc -l confs.txt izin-hatalari.txt
```
</details>

**2.** Aynı aramayı yap, ama bu sefer sonuç ve hatalar tek bir `tum.txt` dosyasına gitsin. Sonra `tum.txt`'de kaç tane `Permission denied` satırı olduğunu say.

<details>
<summary>Çözüm</summary>

```bash
find /etc -name '*.conf' > tum.txt 2>&1
grep -c 'Permission denied' tum.txt
```

`find /etc -name '*.conf' &> tum.txt` de olur.
</details>

**3.** Ekrana hiçbir şey basmadan, sadece izin hatalarının sayısını tek komutta bul.

<details>
<summary>Çözüm</summary>

```bash
find /etc -name '*.conf' 2>&1 >/dev/null | wc -l
```

Sıra burada önemli: önce stderr boruya bağlanır, sonra stdout `/dev/null`'a gider.
</details>

**4.** Tek komutta, heredoc ile `/tmp/uygulama.conf` dosyasını şu iki satırla oluştur. `$HOME` olduğu gibi kalsın, açılmasın:

```
log_dizini=$HOME/log
seviye=info
```

<details>
<summary>Çözüm</summary>

```bash
cat > /tmp/uygulama.conf <<'EOF'
log_dizini=$HOME/log
seviye=info
EOF
cat /tmp/uygulama.conf
```

Tırnaksız `<<EOF` yazsaydın ilk satır `log_dizini=/home/kullanici/log` olurdu.
</details>

**5.** `sudo` yetkin olan normal bir kullanıcıyla, `/etc/hosts` dosyasının sonuna `192.0.2.50 yedek` satırını ekle. Sonra son satıra bakarak kontrol et.

<details>
<summary>Çözüm</summary>

```bash
echo "192.0.2.50 yedek" | sudo tee -a /etc/hosts > /dev/null
tail -1 /etc/hosts
```

`sudo echo "192.0.2.50 yedek" >> /etc/hosts` çalışmaz, `Permission denied` verir.
</details>

---

## 11 - SSL/TLS ve OpenSSL

Konu notu: [SSL/TLS ve OpenSSL](../01-temel-komutlar/11-ssl-tls-ve-openssl.md)

Bütün alıştırmaları boş bir dizinde yap: `mkdir ~/ssl-deneme && cd ~/ssl-deneme`.

**1.** `wiki.sirket.lan` için 90 gün geçerli, şifresiz anahtarlı bir self-signed sertifika oluştur. SAN'da `wiki.sirket.lan` ve `wiki` adları olsun. Soru sormadan, tek komutta.

<details>
<summary>Çözüm</summary>

```bash
openssl req -x509 -newkey rsa:2048 -noenc \
  -keyout wiki.key -out wiki.crt -days 90 \
  -subj "/O=Ornek Sirket/CN=wiki.sirket.lan" \
  -addext "subjectAltName=DNS:wiki.sirket.lan,DNS:wiki"
```
</details>

**2.** 1. alıştırmadaki sertifikanın geçerlilik tarihlerini, SAN alanını ve self-signed olup olmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl x509 -in wiki.crt -noout -dates
openssl x509 -in wiki.crt -noout -ext subjectAltName
openssl x509 -in wiki.crt -noout -subject -issuer
```

`notAfter` 90 gün sonrası olmalı. Subject ve Issuer aynıysa self-signed.
</details>

**3.** Bir iç CA oluştur (`ca.key`, `ca.crt`, 10 yıl). Sonra `git.sirket.lan` için bir CSR üret (SAN'lı, şifresiz anahtar) ve bu CSR'ı CA ile 1 yıllığına imzala. SAN'ın sertifikaya geçtiğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl req -x509 -newkey rsa:4096 -noenc -keyout ca.key -out ca.crt \
  -days 3650 -subj "/O=Ornek Sirket/CN=Ornek Sirket Ic CA"

openssl req -newkey rsa:2048 -noenc -keyout git.key -out git.csr \
  -subj "/O=Ornek Sirket/CN=git.sirket.lan" \
  -addext "subjectAltName=DNS:git.sirket.lan"

openssl x509 -req -in git.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out git.crt -days 365 -copy_extensions copy

openssl x509 -in git.crt -noout -issuer -ext subjectAltName
```
</details>

**4.** `git.crt`'nin CA tarafından imzalandığını doğrula. Sonra `git.key`'in `git.crt` ile, `wiki.key`'in ise `git.crt` ile eşleşip eşleşmediğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl verify -CAfile ca.crt git.crt          # git.crt: OK

openssl x509 -in git.crt -noout -pubkey | sha256sum
openssl pkey -in git.key -pubout | sha256sum    # aynı
openssl pkey -in wiki.key -pubout | sha256sum   # farklı
```
</details>

**5.** `wiki.crt` önümüzdeki 30 gün içinde dolacak mı? Peki 120 gün içinde? Her ikisinin çıkış kodunu da göster.

<details>
<summary>Çözüm</summary>

```bash
openssl x509 -in wiki.crt -noout -checkend $((30*86400)); echo $?
openssl x509 -in wiki.crt -noout -checkend $((120*86400)); echo $?
```

30 gün için `Certificate will not expire` ve `0`, 120 gün için `Certificate will expire` ve `1`.
</details>

---

## 12 - Git ile Sürüm Kontrolü

Konu notu: [Git ile Sürüm Kontrolü](../01-temel-komutlar/12-git.md)

**1.** Kimliğini ayarla. Sonra `~/deneme-repo` adında, ilk dalı `main` olan bir repo oluştur. İçine `README.md` dosyası koy ve "Ilk commit" mesajıyla kaydet.

<details>
<summary>Çözüm</summary>

```bash
git config --global user.name "Ad Soyad"
git config --global user.email "ad@ornek.lan"
git config --global init.defaultBranch main

git init ~/deneme-repo
cd ~/deneme-repo
echo "# Deneme" > README.md
git add README.md
git commit -m "Ilk commit"
git log --oneline
```
</details>

**2.** `README.md`'ye bir satır ekle ve `notlar.txt` adında yeni bir dosya oluştur. Sadece `README.md`'deki değişikliği commit et, `notlar.txt` dışarıda kalsın. Commit'ten önce neyin kaydedileceğine bak.

<details>
<summary>Çözüm</summary>

```bash
echo "Bu bir deneme reposudur." >> README.md
echo "taslak" > notlar.txt
git status -s           #  M README.md   ?? notlar.txt
git add README.md
git diff --staged
git commit -m "README aciklamasi eklendi"
git status -s           # ?? notlar.txt
```

`git commit -am` de bu durumda aynı sonucu verirdi, çünkü `notlar.txt` takip edilmiyor.
</details>

**3.** `*.tmp` uzantılı dosyalar ve `notlar.txt` repoya hiç girmesin. Ayarla, bir `a.tmp` oluşturup `git status`'un bunları göstermediğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
printf '*.tmp\nnotlar.txt\n' > .gitignore
touch a.tmp
git status -s           # sadece ?? .gitignore
git add .gitignore
git commit -m ".gitignore eklendi"
```
</details>

**4.** `ozellik` adında bir dal aç, içinde `yeni.sh` oluşturup commit et. Ana dala dön, `yeni.sh`'ın orada olmadığını gör. Sonra dalı ana dala birleştir ve dalı sil.

<details>
<summary>Çözüm</summary>

```bash
git switch -c ozellik
echo 'echo merhaba' > yeni.sh
git add yeni.sh
git commit -m "yeni.sh eklendi"

git switch main
ls yeni.sh              # No such file or directory

git merge ozellik       # Fast-forward
ls yeni.sh
git branch -d ozellik
```
</details>

**5.** `/tmp/merkez.git` adında bir bare repo oluştur, `deneme-repo`'yu oraya gönder. Sonra onu `/tmp/kopya` dizinine klonla, klonda bir commit yapıp gönder. Son olarak `deneme-repo`'ya dönüp önce gelen değişikliği gör, sonra al.

<details>
<summary>Çözüm</summary>

```bash
git init --bare /tmp/merkez.git
cd ~/deneme-repo
git remote add origin /tmp/merkez.git
git push -u origin main

git clone /tmp/merkez.git /tmp/kopya
cd /tmp/kopya
echo "kopyadan eklendi" >> README.md
git commit -am "Kopyadan degisiklik"
git push

cd ~/deneme-repo
git fetch
git log --oneline main..origin/main    # Kopyadan degisiklik
git pull
```
</details>

---

## 13 - Servis Oluşturma ve Sorun Giderme

Konu notu: [Servis Oluşturma ve Sorun Giderme](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md)

**1.** `/usr/local/bin/saat.sh` adında, her 30 saniyede bir `date` çıktısını yazan bir script oluştur. Bunu `saat.service` adında bir servis yap: çökerse 10 saniye sonra yeniden başlasın, açılışta başlasın. Hem şimdi başlat hem açılışa ekle.

<details>
<summary>Çözüm</summary>

```bash
sudo tee /usr/local/bin/saat.sh > /dev/null <<'EOF'
#!/bin/bash
while true; do date; sleep 30; done
EOF
sudo chmod +x /usr/local/bin/saat.sh

sudo tee /etc/systemd/system/saat.service > /dev/null <<'EOF'
[Unit]
Description=Saat kaydi

[Service]
ExecStart=/usr/local/bin/saat.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now saat
systemctl is-active saat; systemctl is-enabled saat
```
</details>

**2.** `saat.service`'in ana sürecini `kill -9` ile öldür. Servisin yeniden başladığını journal'dan doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo kill -9 $(systemctl show -p MainPID --value saat)
sleep 12
journalctl -u saat -n 5
systemctl show saat -p NRestarts
```

Journal'da `Scheduled restart job` satırını görmelisin, `NRestarts` 1 olmalı.
</details>

**3.** `saat.service`'in dosyasını değiştirmeden, bir drop-in ile `RestartSec` değerini 3 saniye yap. Geçerli değeri kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl edit saat
# Açılan düzenleyiciye:
# [Service]
# RestartSec=3

systemctl show saat -p RestartUSec     # RestartUSec=3s
systemctl cat saat                      # iki dosya görünür
```
</details>

**4.** Aşağıdaki servis başlamıyor. `systemd-analyze verify` ve `journalctl` ile sorunları bul ve düzelt.

```ini
[Unit]
Description=Hatali servis
[Service]
ExecStart=/usr/local/bin/hatali.sh
Restrat=always
```

`/usr/local/bin/hatali.sh` dosyası var ama izni `644`.

<details>
<summary>Çözüm</summary>

```bash
systemd-analyze verify /etc/systemd/system/hatali.service
systemctl status hatali          # status=203/EXEC
ls -l /usr/local/bin/hatali.sh   # x izni yok

sudo chmod +x /usr/local/bin/hatali.sh
sudo sed -i 's/Restrat/Restart/' /etc/systemd/system/hatali.service
sudo systemctl daemon-reload
sudo systemctl reset-failed hatali
sudo systemctl start hatali
```
</details>

**5.** `cron` servisinin hiçbir şekilde başlatılamamasını sağla, başlatmayı deneyip hatayı gör, sonra eski haline getir.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl stop cron
sudo systemctl mask cron
sudo systemctl start cron        # Unit cron.service is masked.
sudo systemctl unmask cron
sudo systemctl start cron
```
</details>

---

## 14 - Performans İzleme

Konu notu: [Performans İzleme](../01-temel-komutlar/14-performans-izleme.md)

**1.** Sistemin çekirdek sayısını ve son 1, 5 ve 15 dakikalık load average değerlerini bul. Sistem şu an aşırı yüklü mü?

<details>
<summary>Çözüm</summary>

```bash
nproc
uptime
```

Load average değerleri `nproc` sonucundan küçükse sistem aşırı yüklü değil.
</details>

**2.** Arka planda iki tane `yes > /dev/null` başlat. En çok işlemci kullanan 3 süreci `ps` ile listele, sonra `vmstat` ile `r` ve `id` sütunlarına bak. İşin bitince süreçleri durdur.

<details>
<summary>Çözüm</summary>

```bash
yes > /dev/null &
yes > /dev/null &
ps -eo pid,%cpu,comm --sort=-%cpu | head -4
vmstat 1 3
pkill yes
```

`r` sütunu 2 civarında, `id` sıfıra yakın olmalı.
</details>

**3.** Sistemde yeni programlara ne kadar bellek verilebileceğini ve en çok bellek kullanan 5 süreci bul.

<details>
<summary>Çözüm</summary>

```bash
free -h                                              # available sütunu
ps -eo pid,user,rss,%mem,comm --sort=-%mem | head -6
```
</details>

**4.** `sysstat` paketini kur. Arka planda diske yazan bir `dd` başlat, hangi sürecin saniyede ne kadar yazdığını ve diskin ne kadar meşgul olduğunu bul. Sonra dosyayı sil.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install sysstat
dd if=/dev/zero of=/tmp/test bs=1M count=3000 oflag=direct &
pidstat -d 1 3          # kB_wr/s sütununda dd
iostat -dxz 1 3         # %util
wait
rm /tmp/test
```
</details>

**5.** Sistemde en çok bellek kullanan servisleri listele ve açılışta en uzun süren 3 servisi bul.

<details>
<summary>Çözüm</summary>

```bash
systemd-cgtop -b -n 1 --order=memory | head
systemd-analyze blame | head -3
```
</details>

---

## 15 - Disk Alanı Sorunları

Konu notu: [Disk Alanı Sorunları](../01-temel-komutlar/15-disk-alani-sorunlari.md)

**1.** Sistemdeki bütün diskleri, sanal dosya sistemleri hariç, doluluklarıyla listele. Kök diskin inode doluluğuna da bak.

<details>
<summary>Çözüm</summary>

```bash
df -h -x tmpfs
df -i /
```
</details>

**2.** `/var` altında en çok yer kaplayan 5 dizini bul, sadece kök diski say.

<details>
<summary>Çözüm</summary>

```bash
sudo du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail -5
```
</details>

**3.** Kök diskte 50 MB'tan büyük bütün dosyaları boyutlarına göre sıralı listele.

<details>
<summary>Çözüm</summary>

```bash
sudo find / -xdev -type f -size +50M -printf '%s %p\n' 2>/dev/null | sort -n
```
</details>

**4.** Silinmiş ama açık dosya durumunu kendin oluştur ve çöz:

```bash
dd if=/dev/zero of=/tmp/buyuk.log bs=1M count=100
tail -f /tmp/buyuk.log > /dev/null &
rm /tmp/buyuk.log
```

Dosyayı tutan süreci bul, yer açılmadığını gör, süreci durdurmadan yeri geri kazan, sonra süreci durdur.

<details>
<summary>Çözüm</summary>

```bash
df -h /tmp
sudo lsof +L1 | grep buyuk.log      # PID ve FD (ör. 3r) sütunlarına bak
sudo truncate -s 0 /proc/PID/fd/3   # PID ve 3 yerine kendi değerlerini yaz
df -h /tmp
kill %1
```
</details>

**5.** Journal'ın ne kadar yer kapladığına bak ve 3 günden eski kayıtları temizle.

<details>
<summary>Çözüm</summary>

```bash
journalctl --disk-usage
sudo journalctl --vacuum-time=3d
journalctl --disk-usage
```
</details>

---

## 16 - Uygulama ve Servis Kısıtlamaları

Konu notu: [Uygulama ve Servis Kısıtlamaları](../01-temel-komutlar/16-uygulama-ve-servis-kisitlamalari.md)

**1.** `ssh` servisinin hangi kullanıcıyla çalıştığını, açık dosya limitini ve bellek sınırını bul. Hangi portu dinlediğini de göster.

<details>
<summary>Çözüm</summary>

```bash
systemctl show ssh -p User -p LimitNOFILE -p MemoryMax
sudo ss -tlnp | grep ssh
```

`User=` boşsa servis root olarak çalışıyordur. `MemoryMax=infinity` sınır yok demektir.
</details>

**2.** Normal bir kullanıcıyla `python3 -m http.server 80` çalıştır, hatayı gör. Sonra aynı komutu 8080 portuyla çalıştırıp başka bir terminalden hangi sürecin 8080'i dinlediğini bul.

<details>
<summary>Çözüm</summary>

```bash
python3 -m http.server 80        # PermissionError: [Errno 13] Permission denied
python3 -m http.server 8080 &
sudo ss -tlnp 'sport = :8080'
kill %1
```
</details>

**3.** `systemd-run` ile, en fazla 100 MB bellek ve %30 işlemci kullanabilen, sonsuz döngüde çalışan geçici bir servis başlat. Limitlerin uygulandığını kontrol et ve servisi durdur.

<details>
<summary>Çözüm</summary>

```bash
sudo systemd-run --unit=sinirli -p MemoryMax=100M -p CPUQuota=30% sh -c 'while :; do :; done'
systemctl show sinirli -p MemoryMax -p CPUQuotaPerSecUSec
top -bn1 | grep ' sh$'          # yaklaşık %30
sudo systemctl stop sinirli
```
</details>

**4.** [13. dosyadaki](../01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md#14-alıştırmalar) `saat.service`'e (veya kendi servislerinden birine) kalıcı olarak 4096 açık dosya limiti ver ve bunu çalışan sürecin `/proc` bilgisinden doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /etc/systemd/system/saat.service.d
printf '[Service]\nLimitNOFILE=4096\n' | sudo tee /etc/systemd/system/saat.service.d/limit.conf
sudo systemctl daemon-reload
sudo systemctl restart saat
grep 'open files' /proc/$(systemctl show -p MainPID --value saat)/limits
```
</details>

**5.** `redis-server` (veya sistemindeki başka bir servis) için hangi servislerin ona bağlı olduğunu ve programın hangi kütüphaneleri kullandığını bul. Servisin güvenlik puanına da bak.

<details>
<summary>Çözüm</summary>

```bash
systemctl list-dependencies --reverse redis-server
ldd /usr/bin/redis-server
systemd-analyze security redis-server
```
</details>

---

Sonraki bölüm: [Kullanıcılar ve Gruplar](02-kullanicilar-ve-gruplar.md) · [Alıştırmalar ana sayfası](README.md)
