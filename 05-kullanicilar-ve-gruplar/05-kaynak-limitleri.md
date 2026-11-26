# Kaynak Limitleri

> **Resmi sınav maddesi:** Configure user resource limits

Bir sunucuyu birden fazla kişi kullanıyorsa, tek bir kullanıcının yanlış yazılmış bir script'le binlerce süreç açması veya diski dev bir dosyayla doldurması herkesi etkiler. Kaynak limitleriyle kullanıcı ve grup başına üst sınırlar koyarız.

Senaryomuz:

- `stajyer` grubundakiler en fazla 20 süreç çalıştırabilsin ve 1 MB'tan büyük dosya oluşturamasın.
- Mert için süreç limiti daha da düşük olsun.
- Mert'in denediği test uygulaması çok sayıda dosya açıyor, onun için açık dosya limiti 4096 olsun.

> [!NOTE]
> Komutlar Ubuntu 24.04 üzerinde denenmiştir. Varsayılan limit değerleri sistemin belleğine ve ayarlarına göre farklı olabilir.

## İçindekiler

1. [Limitler nerede tanımlanır?](#1-limitler-nerede-tanımlanır)
2. [Satır yapısı](#2-satır-yapısı)
3. [Kimin için: kullanıcı, grup, herkes](#3-kimin-için-kullanıcı-grup-herkes)
4. [soft ve hard limit](#4-soft-ve-hard-limit)
5. [Sık kullanılan kaynaklar](#5-sık-kullanılan-kaynaklar)
6. [Limitler ne zaman uygulanır?](#6-limitler-ne-zaman-uygulanır)
7. [Uygulamalı: süreç limiti](#7-uygulamalı-süreç-limiti)
8. [Uygulamalı: dosya boyutu limiti](#8-uygulamalı-dosya-boyutu-limiti)
9. [ulimit ile limitleri görmek ve değiştirmek](#9-ulimit-ile-limitleri-görmek-ve-değiştirmek)
10. [Çalışan bir sürecin limitleri](#10-çalışan-bir-sürecin-limitleri)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Limitler nerede tanımlanır?

Kalıcı limitler iki yerde tanımlanır:

- `/etc/security/limits.conf`: ana dosya
- `/etc/security/limits.d/*.conf`: ayrı dosyalar

Kendi kurallarını `limits.d` altında konuya göre ayrı dosyalarda tutmak daha düzenlidir. Örneğin `/etc/security/limits.d/stajyer.conf`.

> [!WARNING]
> `limits.d` altındaki dosyaların adı **`.conf` ile bitmeli**. Denemede aynı kuralı uzantısız `stajyer` adlı dosyaya koyduğumuzda hiç uygulanmadı.

---

## 2. Satır yapısı

Her satır dört alandan oluşur:

```
<kim>      <tür>   <kaynak>   <değer>
@stajyer   hard    nproc      20
```

| Alan | Örnekte | Anlamı |
|---|---|---|
| Kim? | `@stajyer` | Kullanıcı adı, `@grup` veya herkes için `*` |
| Tür | `hard` | `soft`, `hard` veya ikisi birden için `-` |
| Kaynak | `nproc` | Sınırlanan kaynak (süreç sayısı, dosya boyutu...) |
| Değer | `20` | İzin verilen üst sınır. Sınırsız için `unlimited` |

---

## 3. Kimin için: kullanıcı, grup, herkes

Aynı kaynak için birden fazla satır kişiyi kapsıyorsa hangisinin geçerli olduğunu test ettik. Dosyaya sırayla şu satırları ekledik:

```
*         hard   nproc   50
@stajyer  hard   nproc   20
mert      hard   nproc   15
```

| Kullanıcı | Kapsayan satırlar | Geçerli limit |
|---|---|---|
| ayse (stajyer değil) | `*` | 50 |
| mert, sadece `*` ve `@stajyer` varken | `*`, `@stajyer` | 20 |
| mert, kendi satırı eklenince | `*`, `@stajyer`, `mert` | 15 |
| root | `*` | Etkilenmedi |

Sonuç:

- **Kullanıcıya özel satır** grup satırından, **grup satırı** da `*` satırından önce gelir.
- **`*` ve grup satırları root'a uygulanmaz.** Root için limit koymak istiyorsan satırda açıkça `root` yazmalısın.

---

## 4. soft ve hard limit

| Tür | Anlamı |
|---|---|
| `hard` | Kesin üst sınır. Normal kullanıcı bunu aşamaz ve yükseltemez |
| `soft` | Oturum açıldığında geçerli olan sınır. Kullanıcı kendisi `hard` değerine kadar yükseltebilir |
| `-` | `soft` ve `hard` aynı değere ayarlanır |

```
mert   soft   nproc   10
mert   hard   nproc   15
```

Bu ayarla Mert'in oturumu 10 süreç limitiyle açılır. Gerekirse kendisi 15'e kadar çıkarabilir, daha fazlasına çıkaramaz. [9. bölümde](#9-ulimit-ile-limitleri-görmek-ve-değiştirmek) bunu deniyoruz.

Sadece `hard` yazıp `soft` yazmazsan ve varsayılan soft değer yeni hard değerden büyükse, soft değer de hard değere iner. Denemede sadece `mert hard nproc 15` yazdığımızda Mert'in iki limiti de 15 oldu.

---

## 5. Sık kullanılan kaynaklar

| Kaynak | Ne sınırlanır? | Birim |
|---|---|---|
| `nproc` | Kullanıcının aynı anda çalışan süreç sayısı | Adet |
| `nofile` | Bir sürecin aynı anda açabileceği dosya sayısı | Adet |
| `fsize` | Oluşturulabilecek en büyük dosya | KB (1024 = 1 MB) |
| `cpu` | Bir sürecin kullanabileceği toplam işlemci süresi | **Dakika** |
| `maxlogins` | Kullanıcının aynı anda açabileceği oturum sayısı | Adet |

Bütün kaynakların listesi ve açıklamaları için:

```bash
man limits.conf
```

> [!TIP]
> Birimlere dikkat et. `limits.conf`'taki `cpu` **dakika**, `ulimit -t` ise **saniye** cinsindendir. `fsize` ise KB'tır: `fsize 1024` 1 MB demektir.

---

## 6. Limitler ne zaman uygulanır?

Limitleri `pam_limits` modülü **oturum açılırken** uygular. Ubuntu'da bu modül `login`, `su`, `sudo` ve `sudo -i` için tanımlıdır:

```bash
grep -l pam_limits /etc/pam.d/*
```

Denemede `su - mert`, `sudo -u mert` ve `sudo -iu mert` ile açılan oturumların üçünde de yeni limitler geçerliydi.

Bunun sonuçları:

- Dosyayı değiştirmek **açık oturumları etkilemez**. Kullanıcının yeniden giriş yapması gerekir.
- Değişikliği test etmek için yeni bir oturum aç: `sudo su - mert` veya `sudo -iu mert`.
- Servis olarak çalışan programlar (systemd servisleri) PAM'den geçmez. Onların limitleri servis dosyasında ayarlanır (`LimitNOFILE=` gibi).

---

## 7. Uygulamalı: süreç limiti

Mert'e 10 süreç limiti koyalım:

```bash
sudo tee /etc/security/limits.d/mert.conf > /dev/null <<'EOF'
mert   soft   nproc   10
mert   hard   nproc   15
EOF
```

Yeni bir oturumda kontrol:

```bash
sudo su - mert -c 'echo "soft=$(ulimit -Su) hard=$(ulimit -Hu)"'
```
```
soft=10 hard=15
```

Limiti aşmayı deneyelim. Mert'in oturumunda arka planda 12 tane `sleep` başlatıyoruz:

```bash
for i in $(seq 1 12); do sleep 30 & done
```
```
-bash: fork: retry: Resource temporarily unavailable
-bash: fork: retry: Resource temporarily unavailable
```

Sadece 8 tane `sleep` başlayabildi. `nproc`, kullanıcının **bütün** süreçlerini sayar. Mert'in shell'i ve oturum süreçleri de bu 10'un içinde.

> [!NOTE]
> Limit kullanıcının bütün oturumları için toplamdır. Mert'in iki ayrı SSH oturumu varsa iki oturumdaki süreçler birlikte sayılır.

---

## 8. Uygulamalı: dosya boyutu limiti

Stajyerler 1 MB'tan büyük dosya oluşturamasın. `-` ile soft ve hard'ı birlikte ayarlıyoruz:

```bash
echo '@stajyer   -   fsize   1024' | sudo tee /etc/security/limits.d/stajyer.conf
```

Mert'in oturumunda 2 MB'lık dosya oluşturmayı deneyelim:

```bash
dd if=/dev/zero of=/tmp/buyuk.dosya bs=1K count=2048
ls -l /tmp/buyuk.dosya
```
```
File size limit exceeded   dd if=/dev/zero of=/tmp/buyuk.dosya bs=1K count=2048
-rw-rw-r-- 1 mert mert 1048576 Nov 25 13:02 /tmp/buyuk.dosya
```

İşlem durduruldu ve dosya tam olarak 1 MB'ta (1048576 bayt) kaldı. 1000 KB'lık bir dosya ise sorunsuz oluştu.

---

## 9. ulimit ile limitleri görmek ve değiştirmek

`ulimit` shell'e yerleşik bir komuttur ve **sadece o anki oturumu** etkiler.

### Limitleri görmek

```bash
ulimit -a      # bütün soft limitler
ulimit -Su     # süreç limiti, soft
ulimit -Hu     # süreç limiti, hard
ulimit -n      # açık dosya limiti
```

`ulimit -a` çıktısından bir kısım:

```
file size                   (blocks, -f) unlimited
open files                          (-n) 4096
cpu time                   (seconds, -t) unlimited
max user processes                  (-u) 32017
```

Her satırın sonundaki harf, o limiti değiştiren bayraktır.

### Limitleri değiştirmek

Mert'in limitleri soft 10, hard 15 iken denedik:

```bash
ulimit -S -u 15    # soft'u hard'a kadar yükselt: çalışır
ulimit -S -u 16    # hard'ın üstü: "cannot modify limit: Invalid argument"
```

> [!CAUTION]
> `-S` veya `-H` yazmadan kullanılan `ulimit -u 12`, **soft ve hard'ı birlikte** 12 yapar:
> ```
> ulimit -u 12
> soft=12 hard=12
> ulimit -S -u 14   # artık çalışmaz, hard da 12 oldu
> ```
> Normal kullanıcı hard limiti bir kez düşürdüğünde o oturumda tekrar yükseltemez:
> ```
> ulimit -H -u 11
> ulimit -H -u 15   # "cannot modify limit: Operation not permitted"
> ```
> Sadece soft limiti değiştirmek istiyorsan her zaman `-S` yaz.

Kurallar:

- Normal kullanıcı soft limiti **hard'a kadar** yükseltebilir.
- Normal kullanıcı hard limiti **sadece düşürebilir**.
- Bu değişiklikler oturum kapanınca kaybolur. Kalıcı limit için `limits.conf` kullanılır.

---

## 10. Çalışan bir sürecin limitleri

Her sürecin geçerli limitleri `/proc/<PID>/limits` dosyasında görünür:

```bash
grep -E 'Max processes|Max open files' /proc/<PID>/limits
```
```
Max processes             15                   15                   processes
Max open files            4096                 4096                 files
```

Sütunlar sırasıyla soft limit, hard limit ve birimdir. Bir kullanıcının limitlerinin gerçekten uygulanıp uygulanmadığını anlamak için o kullanıcının çalışan bir sürecine buradan bakabilirsin.

---

## 11. Komut özeti

| Komut / Dosya | Ne işe yarar? |
|---|---|
| `/etc/security/limits.conf` | Kalıcı limitlerin ana dosyası |
| `/etc/security/limits.d/isim.conf` | Ayrı limit dosyaları (`.conf` ile bitmeli) |
| `kim tür kaynak değer` | Satır yapısı, örn. `@stajyer hard nproc 20` |
| `man limits.conf` | Bütün kaynakların listesi |
| `sudo su - kullanici` | Limitleri test etmek için yeni oturum |
| `ulimit -a` | Oturumun bütün soft limitleri |
| `ulimit -Su` / `ulimit -Hu` | Süreç limiti, soft / hard |
| `ulimit -S -u N` | Sadece soft süreç limitini değiştirir |
| `ulimit -u N` | Soft ve hard'ı birlikte değiştirir (dikkat) |
| `ulimit -n` | Açık dosya limiti |
| `ulimit -f` | Dosya boyutu limiti (KB) |
| `ulimit -t` | İşlemci süresi limiti (saniye) |
| `cat /proc/PID/limits` | Çalışan bir sürecin limitleri |

---

## 12. Sık yapılan hatalar

- **Değişiklikten sonra eski oturumda test etmek.** Limitler oturum açılırken uygulanır. Yeni bir oturum aç.
- **`limits.d` dosyasına `.conf` uzantısı vermemek.** Dosya okunmaz.
- **`*` satırının root'u da kapsadığını sanmak.** Root için açıkça `root` yazılmalı.
- **Birimleri karıştırmak.** `cpu` dakika, `ulimit -t` saniye, `fsize` KB.
- **`ulimit`'i `-S` olmadan kullanmak.** Hard limit de düşer ve o oturumda geri alınamaz.
- **`nproc`'u oturum başına sanmak.** Kullanıcının bütün oturumlarındaki bütün süreçler birlikte sayılır.
- **systemd servislerinin de `limits.conf`'tan etkilendiğini sanmak.** Servislerin limitleri servis dosyasında ayarlanır.

---

## 13. Alıştırmalar

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** `elif` kullanıcısını oluştur. Oturumu en fazla 30 süreçle açılsın, gerekirse kendisi 40'a kadar çıkarabilsin. Kuralı `/etc/security/limits.d/` altında ayrı bir dosyaya yaz ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash elif
sudo tee /etc/security/limits.d/elif.conf > /dev/null <<'EOF'
elif   soft   nproc   30
elif   hard   nproc   40
EOF
sudo su - elif -c 'echo "soft=$(ulimit -Su) hard=$(ulimit -Hu)"'
```
</details>

**2.** `tasarim` grubundaki herkes 5 MB'tan büyük dosya oluşturamasın. `elif`'i gruba ekleyip 6 MB'lık bir dosya oluşturmayı dene.

<details>
<summary>Çözüm</summary>

```bash
sudo groupadd tasarim
sudo usermod -aG tasarim elif
echo '@tasarim   -   fsize   5120' | sudo tee /etc/security/limits.d/tasarim.conf
sudo su - elif -c 'dd if=/dev/zero of=/tmp/deneme bs=1K count=6144; ls -l /tmp/deneme'
```

`dd` "File size limit exceeded" ile durur ve dosya 5 MB'ta kalır.
</details>

**3.** `elif`'in oturumunda açık dosya limitini görüntüle. Soft limiti, hard limiti değiştirmeden 512'ye düşür.

<details>
<summary>Çözüm</summary>

```bash
sudo su - elif
ulimit -Sn
ulimit -Hn
ulimit -S -n 512
ulimit -Sn
exit
```
</details>

**4.** Aşağıdaki satırların her birindeki sorunu bul.

```
*        hard   nproc   100     # root'u da 100 süreçle sınırlamak istiyorum
@tasarim hard   cpu     30      # 30 saniye işlemci süresi
```

Ayrıca kural `/etc/security/limits.d/ozel` adlı dosyaya yazıldı.

<details>
<summary>Çözüm</summary>

1. `*` root'a uygulanmaz. Root için ayrı bir `root hard nproc 100` satırı gerekir.
2. `cpu` dakika cinsindendir. `30` yazmak 30 dakika demektir. 30 saniyelik bir sınır `limits.conf` ile verilemez, en küçük birim 1 dakikadır.
3. Dosya adı `.conf` ile bitmediği için okunmaz. `ozel.conf` olmalı.
</details>

**5.** Alıştırmada yaptığın her şeyi geri al.

<details>
<summary>Çözüm</summary>

```bash
sudo rm /etc/security/limits.d/elif.conf /etc/security/limits.d/tasarim.conf
sudo rm -f /tmp/deneme
sudo userdel -r elif
sudo groupdel tasarim
```
</details>

---

Önceki konu: [Ortam Profilleri](04-ortam-profilleri.md) · Sonraki konu: [ACL](06-acl.md)
