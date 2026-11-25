# Hard ve Soft Link

Linux'ta bir dosyaya birden fazla yerden ulaşmanın iki yolu var:

- **Hard link:** Aynı dosyaya ikinci bir **ad** vermek.
- **Soft link (sembolik link):** Başka bir dosyanın **yolunu** taşıyan küçük bir dosya. Windows'taki kısayola benzer.

İkisini anlamak için önce dosyanın diskte nasıl tutulduğunu bilmek gerekiyor.

Senaryo: Ekip sunucusunda `/srv/ekip` altında ortak dosyalar var. Ayşe aylık raporlara kendi ev dizininden de ulaşmak istiyor. Uygulamanın yapılandırma dosyası ise sürüm dizinleri arasında değiştirilebilir olmalı.

> [!NOTE]
> Bütün komutlar ve çıktılar Ubuntu 24.04 üzerinde denenmiştir. Inode numaraları senin sisteminde farklı olacaktır.

## İçindekiler

1. [Inode: dosyanın asıl kimliği](#1-inode-dosyanın-asıl-kimliği)
2. [Hard link oluşturmak](#2-hard-link-oluşturmak)
3. [Hard link ve kopya farkı](#3-hard-link-ve-kopya-farkı)
4. [Hard link silinince ne olur?](#4-hard-link-silinince-ne-olur)
5. [Hard link'in sınırları](#5-hard-linkin-sınırları)
6. [Soft link oluşturmak](#6-soft-link-oluşturmak)
7. [Soft link ve izinler](#7-soft-link-ve-izinler)
8. [Mutlak ve göreli hedef: kırık linkler](#8-mutlak-ve-göreli-hedef-kırık-linkler)
9. [Linki güncellemek ve silmek](#9-linki-güncellemek-ve-silmek)
10. [Hard ve soft link karşılaştırması](#10-hard-ve-soft-link-karşılaştırması)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Inode: dosyanın asıl kimliği

Linux bir dosyayı diske yazarken iki şey tutar:

- **Inode:** Dosyanın numaralı bir kaydı. Sahibi, izinleri, boyutu, tarihleri ve verinin diskte nerede olduğu burada yazar. **Dosyanın adı inode'da yoktur.**
- **Dizin kaydı:** Dizinler aslında "ad → inode numarası" listeleridir.

Yani bir dosya adı, sadece bir inode'a verilmiş bir etikettir.

```bash
stat kasim.txt
```
```
  File: kasim.txt
  Size: 13        	Blocks: 8          IO Block: 4096   regular file
Device: 254,0	Inode: 812007      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
```

`Links: 1`, bu inode'a şu an tek bir adın bağlı olduğunu gösterir. Inode numarasını listelerken görmek için `ls -i`:

```bash
ls -li kasim.txt
```
```
812007 -rw-r--r-- 1 root root 13 Nov 25 15:35 kasim.txt
```

İlk sütun inode numarası. İzinlerden sonraki `1` ise bağlantı sayısı.

---

## 2. Hard link oluşturmak

```bash
ln HEDEF YENI_AD
```

Ayşe raporu kendi dizininde de görsün:

```bash
ln /srv/ekip/kasim.txt /home/ayse/kasim.txt
ls -li /srv/ekip/kasim.txt /home/ayse/kasim.txt
```
```
812007 -rw-r--r-- 2 root root 13 Nov 25 15:35 /home/ayse/kasim.txt
812007 -rw-r--r-- 2 root root 13 Nov 25 15:35 /srv/ekip/kasim.txt
```

- İki adın inode numarası aynı: `812007`.
- Bağlantı sayısı ikisinde de `2` oldu.

Hiçbir veri kopyalanmadı. Aynı dosyaya ikinci bir ad eklendi. İkisi arasında "asıl" ve "kopya" yoktur, ikisi de eşit derecede o dosyanın adıdır.

Bir taraftan yapılan değişiklik diğerinde de görünür:

```bash
echo "ek satir" >> /home/ayse/kasim.txt
cat /srv/ekip/kasim.txt
```
```
Kasim raporu
ek satir
```

İzinler de inode'da tutulduğu için iki adın izinleri her zaman aynıdır. `chmod 600 /srv/ekip/kasim.txt` sonrası:

```
-rw------- 2 root root 22 Nov 25 15:35 /home/ayse/kasim.txt
```

---

## 3. Hard link ve kopya farkı

```bash
cp /home/ayse/kasim.txt kopya.txt
ls -li /home/ayse/kasim.txt kopya.txt
```
```
812007 -rw------- 1 root root 22 Nov 25 15:35 /home/ayse/kasim.txt
812008 -rw------- 1 root root 22 Nov 25 15:35 kopya.txt
```

| | `cp` (kopya) | `ln` (hard link) |
|---|---|---|
| Inode | Yeni bir inode (`812008`) | Aynı inode |
| Disk alanı | İki kat | Ek alan yok |
| Biri değişince | Diğeri etkilenmez | İkisi de değişir |
| İzinler | Ayrı ayrı değiştirilebilir | Her zaman aynı |

> [!WARNING]
> Hard link bir yedek **değildir**. Dosyanın içeriği bozulursa veya silinirse bütün adlarda bozulmuş olur. Yedek için `cp`.

---

## 4. Hard link silinince ne olur?

`rm` aslında dosyayı silmez, **bir adı** siler ve bağlantı sayısını bir azaltır. Veri, bağlantı sayısı sıfıra inince silinir.

```bash
rm /srv/ekip/kasim.txt
stat -c '%n inode=%i links=%h' /home/ayse/kasim.txt
cat /home/ayse/kasim.txt
```
```
/home/ayse/kasim.txt inode=812007 links=1
Kasim raporu
ek satir
```

Bir ad silindi, bağlantı sayısı 1'e indi, veri yerinde duruyor. Ayşe'nin adını da silersek sayı 0 olur ve veri gerçekten silinir.

> [!NOTE]
> Bir dosya silindiği halde bir program tarafından hala açık tutuluyorsa, veri program dosyayı kapatana kadar diskte kalır. "Dosyayı sildim ama disk yeri açılmadı" durumunun en sık sebebi budur ([Disk Alanı](15-disk-alani-sorunlari.md)).

---

## 5. Hard link'in sınırları

### Dizinlere hard link yapılamaz

```bash
ln /srv/ekip/dizin /srv/ekip/dizin-link
```
```
ln: dizin: hard link not allowed for directory
```

Dizin ağacında döngü oluşmasını önlemek için yasak. Dizinin bağlantı sayısı yine de 1'den büyüktür: her dizinin kendi adı ve içindeki `.` iki bağlantı sayılır, her alt dizinin `..`'sı bir tane daha ekler. Denemede boş bir dizinde `2`, iki alt dizin eklenince `4` göründü.

### Farklı dosya sistemleri arasında yapılamaz

Inode numaraları sadece kendi dosya sistemi içinde anlamlıdır. Kök dosya sisteminden (ext4) `/dev/shm`'ye (tmpfs) link denedik:

```
ln: failed to create hard link '/dev/shm/kasim.txt' => '/home/ayse/kasim.txt': Invalid cross-device link
```

### Başkasının dosyasına hard link yapılamaz

Ubuntu'da `fs.protected_hardlinks = 1` ayarı açıktır. Normal bir kullanıcı, sahibi olmadığı ve okuma/yazma izni olmayan bir dosyaya hard link oluşturamaz. Mert, root'a ait `600` izinli bir dosyaya link oluşturmayı denedi:

```
ln: failed to create hard link '/home/mert/link.txt' => '/srv/ekip/root-dosyasi.txt': Operation not permitted
```

Bu sınırlar soft link'te yoktur.

---

## 6. Soft link oluşturmak

```bash
ln -s HEDEF LINK_ADI
```

```bash
ln -s /srv/ekip/ayarlar/uygulama.conf /srv/ekip/uygulama.conf
ls -l /srv/ekip/uygulama.conf
```
```
lrwxrwxrwx 1 root root 31 Nov 25 15:35 /srv/ekip/uygulama.conf -> /srv/ekip/ayarlar/uygulama.conf
```

- Satırın başındaki `l`, bunun bir sembolik link olduğunu gösterir.
- `->` sonrası linkin gösterdiği yol.
- Boyut `31`: linkin içinde tutulan yolun karakter sayısı.

Soft link'in **kendi inode'u vardır**:

```bash
ls -li /srv/ekip/uygulama.conf /srv/ekip/ayarlar/uygulama.conf
```
```
812012 lrwxrwxrwx 1 root root 31 Nov 25 15:35 /srv/ekip/uygulama.conf -> /srv/ekip/ayarlar/uygulama.conf
811986 -rw-r--r-- 1 root root 10 Nov 25 15:35 ayarlar/uygulama.conf
```

Linki okumak ve yazmak hedefe yönlenir:

```bash
echo "log=on" >> /srv/ekip/uygulama.conf
cat /srv/ekip/ayarlar/uygulama.conf
```
```
port=8080
log=on
```

### Linkin hedefini görmek

```bash
readlink uygulama.conf        # linkin içindeki yol
readlink -f uygulama.conf     # bütün linkler çözüldükten sonraki gerçek yol
stat uygulama.conf            # linkin kendisi
stat -L uygulama.conf         # hedef dosya
```

`readlink -f`, link başka bir linki gösteriyorsa zinciri sonuna kadar takip eder.

### Soft link'in yapabildikleri

Hard link'in yapamadığı iki şeyi yapar:

```bash
ln -s /srv/ekip/ayarlar /home/ayse/ayarlar              # dizine link
ln -s /srv/ekip/ayarlar/uygulama.conf /dev/shm/uygulama.conf   # başka dosya sistemine
```

İkisi de denemede çalıştı.

---

## 7. Soft link ve izinler

Soft link'in izinleri her zaman `lrwxrwxrwx` görünür. Bu izinler **kullanılmaz**. Erişim kararı hedef dosyanın izinlerine göre verilir.

Hedefi `600` yapınca Mert link üzerinden okuyamadı:

```bash
sudo chmod 600 /srv/ekip/ayarlar/uygulama.conf
sudo su - mert -c 'cat /srv/ekip/uygulama.conf'
```
```
cat: /srv/ekip/uygulama.conf: Permission denied
```

Linke `chmod` uygulamak da linkin kendisini değil **hedefi** değiştirir:

```bash
sudo chmod 600 /srv/ekip/uygulama.conf
ls -l /srv/ekip/uygulama.conf /srv/ekip/ayarlar/uygulama.conf
```
```
-rw------- 1 root root 17 Nov 25 15:35 ayarlar/uygulama.conf
lrwxrwxrwx 1 root root 31 Nov 25 15:35 uygulama.conf -> /srv/ekip/ayarlar/uygulama.conf
```

Link hala `lrwxrwxrwx`, hedefin izni değişti.

---

## 8. Mutlak ve göreli hedef: kırık linkler

Soft link sadece bir yol taşıdığı için, hedef silinir veya taşınırsa link **kırılır**.

```bash
ln -s olmayan-dosya.txt kirik.txt
cat kirik.txt
```
```
cat: kirik.txt: No such file or directory
```

`ls -l` linki gösterir ama dosyanın kendisi yoktur. Renkli terminallerde kırık linkler genelde kırmızı görünür.

### Göreli yollar linkin bulunduğu dizine göredir

Göreli bir hedef, **linkin bulunduğu dizine göre** çözülür, linki oluşturduğun dizine göre değil. Denemede `/srv/ekip` içindeyken `/tmp` altına link oluşturduk:

```bash
cd /srv/ekip
ln -s proje-yeni/surum1/app.conf /tmp/yanlis.conf
cat /tmp/yanlis.conf
```
```
cat: /tmp/yanlis.conf: No such file or directory
```

Link `/tmp/proje-yeni/surum1/app.conf`'u arıyor.

### Hangisi daha dayanıklı?

İki link oluşturup hedeflerin bulunduğu dizinin adını değiştirdik:

```bash
ln -s /srv/ekip/proje/surum1/app.conf /srv/ekip/mutlak.conf   # mutlak yol
cd /srv/ekip/proje && ln -s surum1/app.conf goreli.conf        # göreli yol
mv /srv/ekip/proje /srv/ekip/proje-yeni
```

| Link | Sonuç |
|---|---|
| `mutlak.conf -> /srv/ekip/proje/surum1/app.conf` | Kırıldı |
| `proje-yeni/goreli.conf -> surum1/app.conf` | Çalışıyor, çünkü link ve hedef birlikte taşındı |

| | Mutlak yol | Göreli yol |
|---|---|---|
| Link taşınırsa | Çalışmaya devam eder | Kırılabilir |
| Link ve hedef birlikte taşınırsa | Kırılır | Çalışmaya devam eder |

Sistem genelinde sabit yerlere (`/etc/...`) link veriyorsan mutlak, bir proje dizininin içindeki dosyaları birbirine bağlıyorsan göreli yol kullan.

### Kırık linkleri bulmak

```bash
find /srv/ekip -xtype l
```
```
/srv/ekip/mutlak.conf
```

`-type l` bütün sembolik linkleri, `-xtype l` ise hedefi olmayanları (kırıkları) bulur.

```bash
find /srv/ekip -type l -ls      # bütün linkler ve hedefleri
```

---

## 9. Linki güncellemek ve silmek

### Hedefi değiştirmek

Var olan bir linkin üzerine yeniden `ln -s` yazınca hata alırsın:

```
ln: failed to create symbolic link 'proje-yeni/goreli.conf': File exists
```

`-f` (force) eski linki silip yenisini oluşturur. Ama link bir **dizini** gösteriyorsa dikkat:

```bash
ln -s surumler/v1 guncel
ln -sf surumler/v2 guncel
ls -l guncel surumler/v1/
```
```
guncel -> surumler/v1
surumler/v1/:
v2 -> surumler/v2
```

`guncel` hala `v1`'i gösteriyor ve `v1` dizininin içine `v2` adında yanlış bir link oluştu. `ln`, `guncel`'i bir dizin olarak takip etti. Dizin linklerini güncellerken `-n` de ekle:

```bash
ln -sfn surumler/v2 guncel
```
```
guncel -> surumler/v2
```

Sürüm değiştirirken bu yöntem çok kullanılır: uygulama hep `guncel`'i kullanır, yeni sürüme geçmek için sadece link değiştirilir.

### Silmek

```bash
rm uygulama.conf         # sadece linki siler, hedef yerinde kalır
unlink uygulama.conf     # aynı iş
```

> [!CAUTION]
> Bir dizini gösteren linki silerken **sonuna `/` koyma**.
>
> ```bash
> ln -s /srv/ekip/proje-yeni kisayol
> rm -rf kisayol/
> ls -la proje-yeni
> ```
> ```
> total 8
> drwxr-xr-x 2 root root 4096 Nov 25 15:35 .
> drwxr-xr-x 5 root root 4096 Nov 25 15:35 ..
> ```
>
> Link yerinde kaldı ama **hedef dizinin içindeki her şey silindi**. Sondaki `/`, "linkin gösterdiği dizin" demektir. Linki silmek için `rm kisayol` yaz (sonda `/` olmadan, `-r` olmadan).

---

## 10. Hard ve soft link karşılaştırması

| | Hard link (`ln`) | Soft link (`ln -s`) |
|---|---|---|
| Ne oluşturur? | Aynı inode'a yeni bir ad | Yolu taşıyan ayrı bir dosya |
| Inode | Hedefle aynı | Kendi inode'u var |
| `ls -l`'de | Normal dosya, bağlantı sayısı artar | `l` ile başlar, `->` hedefi gösterir |
| Dizinler | Yapılamaz | Yapılabilir |
| Farklı dosya sistemleri | Yapılamaz | Yapılabilir |
| Hedef silinirse | Veri kalır (son ad silinene kadar) | Link kırılır |
| İzinler | Hedefle aynı (aynı inode) | Hedefin izinleri geçerli |
| Tipik kullanım | Aynı dosya sisteminde veriyi iki yerden paylaşmak | Kısayollar, sürüm değiştirme, `/etc` altındaki ayarlar |

Günlük yönetimde çok daha sık soft link kullanılır. Örneğin `/etc/localtime` bir soft link ([Saat Senkronizasyonu](../02-ag-yonetimi/03-saat-senkronizasyonu.md#elle-etclocaltime)), Nginx'te sitelerin etkinleştirilmesi `sites-enabled` altındaki soft linklerle yapılır ([Ters Proxy](../02-ag-yonetimi/09-ters-proxy-ve-yuk-dengeleme.md#1-nginxin-dosya-düzeni)).

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `ls -li` | Inode numaralarıyla listeler |
| `stat dosya` | Inode, bağlantı sayısı, izinler |
| `stat -L link` | Linkin hedefinin bilgileri |
| `ln hedef ad` | Hard link oluşturur |
| `ln -s hedef link` | Soft link oluşturur |
| `ln -sf hedef link` | Var olan linkin üzerine yazar |
| `ln -sfn hedef link` | Dizin linkini güncellerken |
| `readlink link` | Linkin içindeki yol |
| `readlink -f link` | Bütün linkler çözüldükten sonraki gerçek yol |
| `find DIZIN -type l` | Bütün soft linkler |
| `find DIZIN -xtype l` | Kırık soft linkler |
| `rm link` / `unlink link` | Linki siler (sonunda `/` olmadan) |

---

## 12. Sık yapılan hatalar

- **Hard link'i yedek sanmak.** Aynı veridir. İçerik bozulursa bütün adlarda bozuk olur.
- **Dizine hard link yapmaya çalışmak.** Soft link kullan.
- **Farklı dosya sistemleri arasında hard link.** `Invalid cross-device link`, soft link kullan.
- **Göreli soft link'i yanlış dizine göre yazmak.** Hedef, linkin bulunduğu dizine göre çözülür.
- **Hedefi taşıyıp mutlak linki unutmak.** Link kırılır, `find -xtype l` ile kontrol et.
- **Linkin `rwxrwxrwx` izinleriyle herkesin erişebildiğini sanmak.** Hedefin izinleri geçerli.
- **Linke `chmod` uygulayıp linkin değiştiğini sanmak.** Hedef değişir.
- **Dizin linkini `-n`'siz `ln -sf` ile güncellemek.** Hedef dizinin içine yeni link oluşur.
- **Dizin linkini `rm -rf link/` ile silmek.** Hedefin içi silinir.

---

## 13. Alıştırmalar

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

Önceki konu: [Dosya ve Dizin İşlemleri](02-dosya-ve-dizin-islemleri.md) · Sonraki konu: [İzinler ve Özel Bitler](04-izinler-ve-ozel-bitler.md)
