# Girdi/Çıktı Yönlendirme

Komutların çoğu girdisini klavyeden okur, çıktısını ve hatalarını ekrana yazar. **Yönlendirme**, bu akışların yönünü değiştirmektir: çıktıyı bir dosyaya kaydetmek, hataları ayrı bir dosyaya almak veya tamamen susturmak, bir dosyayı komuta girdi olarak vermek, bir komutun çıktısını başka bir komuta aktarmak.

Senaryo: Sunucuda normal bir kullanıcıyla yapılandırma dosyalarını tarıyorsun. Sonuçları bir rapora yazacaksın, ama ekran izin hatalarıyla doluyor. Ayrıca `/etc/hosts`'a `sudo` ile bir satır eklemen gerekiyor.

Örneklerin çoğunda şu komutu kullanacağız. Bir dosya var, biri yok, bu yüzden hem normal çıktı hem hata üretiyor:

```bash
ls /etc/hostname /etc/yok
```
```
ls: cannot access '/etc/yok': No such file or directory
/etc/hostname
```

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve bash 5.2 ile denenmiştir. `&>` ve `|&` bash'e özgüdür, `sh` (dash) içinde çalışmaz.

## İçindekiler

1. [Üç akış: stdin, stdout, stderr](#1-üç-akış-stdin-stdout-stderr)
2. [Çıktıyı dosyaya yazmak: > ve >>](#2-çıktıyı-dosyaya-yazmak--ve-)
3. [Hataları yönlendirmek: 2> ve 2>>](#3-hataları-yönlendirmek-2-ve-2)
4. [Görmek istemediğini atmak: /dev/null](#4-görmek-istemediğini-atmak-devnull)
5. [Tek dosyada toplamak: 2>&1 ve &>](#5-tek-dosyada-toplamak-21-ve-)
6. [Girdi vermek: <](#6-girdi-vermek-)
7. [Heredoc: <<](#7-heredoc-)
8. [Here-string: <<<](#8-here-string-)
9. [Boru: |](#9-boru-)
10. [tee: hem ekrana hem dosyaya](#10-tee-hem-ekrana-hem-dosyaya)
11. [sudo ile yönlendirme](#11-sudo-ile-yönlendirme)
12. [Dosyanın üzerine yazma tuzakları](#12-dosyanın-üzerine-yazma-tuzakları)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Üç akış: stdin, stdout, stderr

Her program üç standart akışla başlar. Her birinin bir numarası (**dosya tanımlayıcısı**) vardır:

| No | Ad | Ne taşır? | Varsayılan |
|---|---|---|---|
| 0 | stdin | Girdi | Klavye |
| 1 | stdout | Normal çıktı | Ekran |
| 2 | stderr | Hata ve uyarı mesajları | Ekran |

```
            ┌─────────┐ ──> stdout (1)
stdin (0) ──>│  komut  │
            └─────────┘ ──> stderr (2)
```

stdout ve stderr ikisi de varsayılan olarak ekrana gittiği için ekranda karışık görünürler. Ama ayrı akışlardır ve ayrı ayrı yönlendirilebilirler. Yukarıdaki `ls` çıktısında `/etc/hostname` stdout'tan, `cannot access` satırı stderr'den geldi.

Komutların çoğu hem dosya adı alır hem de dosya verilmezse stdin'den okur. `sort sayilar.txt` dosyayı okur, `sort` tek başına klavyeden okur (Ctrl+D ile bitirilir). Yönlendirme bu yüzden işe yarar.

---

## 2. Çıktıyı dosyaya yazmak: > ve >>

```bash
ls /etc/hostname /etc/yok > cikti.txt
```

Ekranda sadece hata kaldı, normal çıktı dosyaya gitti:

```
ls: cannot access '/etc/yok': No such file or directory
```
```bash
cat cikti.txt
```
```
/etc/hostname
```

`>` aslında `1>`'nin kısaltmasıdır, yani sadece stdout'u yönlendirir.

| Operatör | Dosya varsa | Dosya yoksa |
|---|---|---|
| `>` | İçeriği **silinir**, yenisi yazılır | Oluşturulur |
| `>>` | Yeni çıktı **sonuna eklenir** | Oluşturulur |

```bash
date > zaman.txt
date > zaman.txt     # zaman.txt'de sadece bu satır kalır
date >> zaman.txt    # iki satır olur
```

---

## 3. Hataları yönlendirmek: 2> ve 2>>

```bash
ls /etc/hostname /etc/yok 2> hata.txt
```

Bu sefer ekranda normal çıktı kaldı, hata dosyaya gitti:

```
/etc/hostname
```

`2>` üzerine yazar, `2>>` ekler. İkisini birlikte kullanıp çıktıyla hataları ayrı dosyalara alabilirsin. Normal bir kullanıcıyla `/etc` altında arama yaptığımızda:

```bash
grep -r PermitRootLogin /etc/ > sonuc.txt 2> hatalar.txt
wc -l sonuc.txt hatalar.txt
```
```
   2 sonuc.txt
  30 hatalar.txt
```

Ekrana hiçbir şey yazılmadı. 2 satırlık sonuç ve okuma izni olmayan dosyalar için 30 satırlık `Permission denied` hatası ayrı dosyalara gitti.

---

## 4. Görmek istemediğini atmak: /dev/null

`/dev/null`, yazılan her şeyi yok eden özel bir dosyadır. Hataları susturmak için:

```bash
grep -r PermitRootLogin /etc/ 2>/dev/null
```
```
/etc/ssh/sshd_config:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config:# the setting of "PermitRootLogin prohibit-password".
```

30 satırlık hata gitti, sadece sonuç kaldı. Tersini de yapabilirsin: `komut > /dev/null` sadece hataları gösterir.

> [!WARNING]
> `/dev/null`'a giden çıktı geri gelmez. Hataları susturmak, bir sorunun belirtisini de gizleyebilir. Bir şey beklediğin gibi çalışmıyorsa önce `2>/dev/null`'u kaldırıp hatalara bak.

---

## 5. Tek dosyada toplamak: 2>&1 ve &>

```bash
ls /etc/hostname /etc/yok > hepsi.txt 2>&1
cat hepsi.txt
```
```
ls: cannot access '/etc/yok': No such file or directory
/etc/hostname
```

`2>&1`, "stderr'i, stdout'un **şu anda** gittiği yere gönder" demektir. `&1`'deki `&`, `1`'in bir dosya adı değil tanımlayıcı numarası olduğunu belirtir. `2>1` yazsaydın `1` adında bir dosya oluşurdu.

### Sıra önemli

Bash yönlendirmeleri **soldan sağa** işler:

```bash
ls /etc/hostname /etc/yok > hepsi.txt 2>&1    # doğru
ls /etc/hostname /etc/yok 2>&1 > ters.txt     # hata ekranda kalır
```

İkinci komutta hata yine ekrana yazıldı, `ters.txt`'ye sadece `/etc/hostname` gitti. Nedeni:

1. `2>&1`: stderr, stdout'un o anda gittiği yere bağlanır. stdout henüz ekranda, yani stderr de ekrana bağlanır.
2. `> ters.txt`: stdout dosyaya yönlenir. stderr'in bağlantısı değişmez, ekranda kalır.

### Kısa yol: &>

Bash'te aynı işi daha kısa yazabilirsin:

```bash
ls /etc/hostname /etc/yok &> hepsi.txt     # > hepsi.txt 2>&1 ile aynı
ls /etc/hostname /etc/yok &>> hepsi.txt    # ekleyerek
```

### Aynı dosyayı iki kez açmak

```bash
ls /etc/hostname /etc/yok > ayni.txt 2> ayni.txt
cat ayni.txt
```
```
/etc/hostname
ess '/etc/yok': No such file or directory
```

İki akış dosyayı ayrı ayrı açtı ve birbirinin üzerine yazdı. Hata mesajının başı silindi. İki akışı aynı dosyaya istiyorsan her zaman `> dosya 2>&1` veya `&> dosya` kullan.

---

## 6. Girdi vermek: <

`<`, bir dosyayı komutun stdin'ine bağlar:

```bash
sort < sayilar.txt
wc -l < /etc/passwd
```

`sort sayilar.txt` ile `sort < sayilar.txt` aynı sonucu verir. Farkı şurada görürsün:

```bash
wc -l /etc/passwd
wc -l < /etc/passwd
```
```
62 /etc/passwd
62
```

İkincisinde `wc` dosyanın adını bilmez, sadece içeriği görür, bu yüzden adı yazmaz. Script'lerde sayıyı tek başına almak için kullanışlıdır.

Bazı komutlar dosya adı kabul etmez, sadece stdin'den okur. Örneğin `tr`:

```bash
tr a-z A-Z < ad.txt
```

---

## 7. Heredoc: <<

Birden fazla satırı komuta girdi olarak vermek için, metni ayrı bir dosyaya yazmadan:

```bash
sort <<EOF
6
3
2
5
EOF
```
```
2
3
5
6
```

`<<EOF` ile başlar, tek başına bir satırdaki `EOF` ile biter. `EOF` sadece bir alışkanlık, `SON` gibi herhangi bir kelime olabilir.

En sık kullanım, bir dosyayı içeriğiyle birlikte oluşturmak:

```bash
cat > ayar.conf <<EOF
port=8080
mod=uretim
EOF
```

### Değişkenler açılır, tırnakla kapatılır

```bash
AD=ayse
cat <<EOF
Merhaba $AD, bugun $(date +%Y)
EOF
```
```
Merhaba ayse, bugun 2026
```

Bitiş kelimesini tırnak içine alırsan metin olduğu gibi kalır:

```bash
cat <<'EOF'
Merhaba $AD, bugun $(date +%Y)
EOF
```
```
Merhaba $AD, bugun $(date +%Y)
```

İçinde `$` geçen bir script veya yapılandırma dosyası yazıyorsan `<<'EOF'` kullan.

### Bitiş satırı

Bitiş kelimesi satırda **tek başına** olmalı, önünde ve arkasında boşluk olmamalı. Bir script'te `  SON` gibi önünde boşluk bırakınca bash bitişi bulamadı:

```
sp.sh: line 4: warning: here-document at line 1 delimited by end-of-file (wanted `SON')
```

Dosyanın sonuna kadar her şeyi heredoc'un içeriği saydı, arkasından gelen `echo bitti` komutu çalışmadı, metin olarak ekrana basıldı. Etkileşimli terminalde ise bash `>` istemiyle girdi beklemeye devam eder.

Script'te girintili yazmak istiyorsan `<<-` kullanılır. Bu, satır başındaki **sekmeleri** (Tab) siler, boşlukları silmez:

```bash
	cat <<-SON
		sekmeli satir
	SON
```

---

## 8. Here-string: <<<

Tek bir satırı stdin olarak vermek için:

```bash
bc <<< "1+2+3+4"              # 10
wc -w <<< "bir iki uc"        # 3
tr a-z A-Z <<< "merhaba"      # MERHABA
read ad soyad <<< "Ayse Yilmaz"; echo "$soyad"   # Yilmaz
```

`echo "1+2+3+4" | bc` ile aynı işi yapar ama ek bir işlem başlatmaz. Here-string bash'e özgüdür.

---

## 9. Boru: |

Boru (pipe), bir komutun **stdout**'unu sonraki komutun **stdin**'ine bağlar. Küçük araçları birleştirip tek satırda iş yapmanın yolu budur:

```bash
grep -Ev '^(#|$)' /etc/login.defs | sort | column -t | head -3
```
```
CHFN_RESTRICT     rwh
DEFAULT_HOME      yes
ENCRYPT_METHOD    SHA512
```

Yorum ve boş satırları at ([grep ve Düzenli İfadeler](07-grep-ve-regex.md)), sırala, sütunları hizala, ilk üçünü al.

### Boru sadece stdout'u taşır

```bash
ls /etc/hostname /etc/yok | wc -l
```
```
ls: cannot access '/etc/yok': No such file or directory
1
```

Hata borudan geçmedi, doğrudan ekrana yazıldı. `wc` sadece bir satır saydı. Hataları da boruya katmak için:

```bash
ls /etc/hostname /etc/yok 2>&1 | wc -l    # 2
ls /etc/hostname /etc/yok |& wc -l        # 2, bash kısaltması
```

Tersine, **sadece hataları** boruya göndermek de mümkün. Bu kalıp ilk bakışta tuhaf görünür ama sırayı hatırlarsan mantıklıdır:

```bash
find /etc -name '*.conf' 2>&1 >/dev/null | head -2
```
```
find: '/etc/redis': Permission denied
find: '/etc/ldap/slapd.d/cn=config': Permission denied
```

Önce stderr, stdout'un o anki yerine, yani boruya bağlandı. Sonra stdout `/dev/null`'a gönderildi. Boruda sadece hatalar kaldı.

### Borunun çıkış kodu

Bir borunun çıkış kodu, **son** komutun çıkış kodudur:

```bash
ls /yok 2>/dev/null | wc -l
echo $?                      # 0, çünkü wc başarılı oldu
echo "${PIPESTATUS[@]}"      # 2 0: her komutun ayrı ayrı çıkış kodu
```

Script'lerde borudaki herhangi bir komut başarısız olursa bunu fark etmek için `set -o pipefail` kullanılır. O zaman `echo $?` `2` verir.

---

## 10. tee: hem ekrana hem dosyaya

`tee`, stdin'den okuduğunu hem ekrana (stdout) hem bir dosyaya yazar. Adı, borudaki T şeklindeki ek parçadan gelir:

```bash
ls /etc/hostname /etc/yok 2>&1 | tee kayit.txt | wc -l
```

Burada çıktı hem `kayit.txt`'ye kaydedildi hem de bir sonraki komuta (`wc`) aktarıldı. Uzun süren bir komutu izlerken çıktısını kaydetmek için idealdir.

`tee -a` dosyanın üzerine yazmak yerine sonuna ekler:

```bash
echo "ikinci" | tee -a kayit.txt > /dev/null
```

Sonundaki `> /dev/null`, ekrana tekrar basılmasını engeller.

---

## 11. sudo ile yönlendirme

`/etc/hosts`'a satır eklemek istiyorsun, aklına gelen ilk komut şu:

```bash
sudo echo "10.0.0.5 db" >> /etc/hosts
```
```
-bash: line 1: /etc/hosts: Permission denied
```

`sudo` vardı, neden olmadı? Çünkü yönlendirmeyi (`>>`) `echo` değil, **senin kabuğun** yapıyor ve kabuk normal kullanıcı yetkisiyle çalışıyor. `sudo` sadece `echo`'yu root yaptı, dosyayı açan kabuk ise normal kullanıcıydı.

Çözüm, dosyaya yazma işini de root olarak yaptırmak:

```bash
echo "10.0.0.5 db" | sudo tee -a /etc/hosts > /dev/null
```

Burada dosyayı açan `tee`, ve o `sudo` ile root olarak çalışıyor. İkinci bir yol, bütün komutu root yetkisiyle başlatılan bir kabuğa vermek:

```bash
sudo sh -c 'echo "10.0.0.6 cache" >> /etc/hosts'
```

İkisini de denedik, satırlar dosyaya eklendi.

> [!CAUTION]
> `sudo tee` kullanırken `-a`'yı unutursan dosyanın tüm içeriği silinip yerine tek satır yazılır. `/etc/hosts` veya `/etc/fstab` gibi dosyalarda bu ciddi bir sorundur.

---

## 12. Dosyanın üzerine yazma tuzakları

### Okuduğun dosyaya yazmak

```bash
sort kopya.txt > kopya.txt
wc -c kopya.txt
```
```
0 kopya.txt
```

Dosya boşaldı. Kabuk `> kopya.txt`'yi komut çalışmadan **önce** işler ve dosyayı sıfırlar. `sort` çalıştığında okuyacak bir şey kalmamıştır. Aynı dosyaya yazmak için:

```bash
sort -o kopya.txt kopya.txt              # sort'un kendi seçeneği
sort kopya.txt > gecici && mv gecici kopya.txt
```

`sed -i` de aynı nedenle var ([Metin İşleme](06-metin-isleme.md)).

### noclobber: > ile üzerine yazmayı engellemek

```bash
set -o noclobber
echo a > sirali.txt
```
```
bash: sirali.txt: cannot overwrite existing file
```

Bu açıkken var olan bir dosyanın üzerine bilerek yazmak için `>|` kullanılır. `set +o noclobber` ile kapatılır. Yanlışlıkla `>` yerine `>>` yazmaktan korkuyorsan `~/.bashrc`'ye eklenebilir.

---

## 13. Komut özeti

| Yazım | Ne yapar? |
|---|---|
| `komut > dosya` | stdout'u dosyaya yaz (üzerine) |
| `komut >> dosya` | stdout'u dosyaya ekle |
| `komut 2> dosya` | stderr'i dosyaya yaz |
| `komut 2>> dosya` | stderr'i dosyaya ekle |
| `komut > a.txt 2> b.txt` | Çıktı ve hata ayrı dosyalara |
| `komut > dosya 2>&1` | İkisi aynı dosyaya |
| `komut &> dosya` | Aynısı, bash kısaltması |
| `komut &>> dosya` | İkisini ekleyerek |
| `komut 2>/dev/null` | Hataları sustur |
| `komut > /dev/null` | Normal çıktıyı sustur |
| `komut < dosya` | stdin'i dosyadan al |
| `komut <<EOF ... EOF` | Çok satırlı girdi |
| `komut <<'EOF' ... EOF` | Değişkenler açılmadan |
| `komut <<< "metin"` | Tek satırlık girdi |
| `k1 \| k2` | k1'in stdout'u k2'nin stdin'ine |
| `k1 2>&1 \| k2` veya `k1 \|& k2` | Hatalar da boruya |
| `k1 2>&1 >/dev/null \| k2` | Sadece hatalar boruya |
| `k1 \| tee dosya` | Hem ekrana hem dosyaya |
| `k1 \| sudo tee -a dosya` | Root yetkisiyle dosyaya ekle |
| `set -o noclobber`, `>\|` | Üzerine yazmayı engelle, yine de yaz |

---

## 14. Sık yapılan hatalar

- **`2>&1`'i `>`'den önce yazmak.** Hatalar ekranda kalır. Doğrusu `> dosya 2>&1`.
- **`2>1` yazmak.** `1` adında bir dosya oluşur. `&` gerekir.
- **`> dosya 2> dosya` ile iki akışı aynı dosyaya göndermek.** Birbirinin üzerine yazarlar.
- **`>>` yerine `>` yazmak.** Dosyanın eski içeriği gider.
- **Okuduğun dosyaya `>` ile yazmak.** Dosya boşalır.
- **`sudo echo ... > /etc/dosya` denemek.** Yönlendirmeyi kabuk yapar, `sudo tee` kullan.
- **`sudo tee`'de `-a`'yı unutmak.**
- **Borunun hataları da taşıdığını sanmak.**
- **Heredoc bitiş kelimesinin önüne boşluk koymak.**

---

## 15. Alıştırmalar

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

Önceki konu: [Yedekleme: rsync ve dd](09-yedekleme-rsync-ve-dd.md) · Sonraki konu: [SSL/TLS ve OpenSSL](11-ssl-tls-ve-openssl.md)
