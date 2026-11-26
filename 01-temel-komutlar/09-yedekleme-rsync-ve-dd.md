# Yedekleme: rsync ve dd

Dosyaları başka bir yere yedeklemenin iki temel aracı var. İkisi farklı işler yapar:

| | rsync | dd |
|---|---|---|
| Ne kopyalar? | Dosya ve dizinleri | Bir disk veya bölümün tamamını, bayt bayt |
| İkinci çalıştırmada | Sadece değişeni gönderir | Her şeyi baştan kopyalar |
| Dosya sistemini anlar mı? | Evet | Hayır, boş alanı bile kopyalar |
| Tipik kullanım | Dizin yedeği, sunucular arası eşitleme | Disk imajı almak, diski klonlamak |

Kısacası: belirli dosya ve dizinleri yedekliyorsan rsync, bir diskin birebir kopyasını istiyorsan dd.

Senaryo: Web sitesi `site/` dizininde duruyor. Her gece yedek sunucusuna (`yedekci@192.0.2.20`) eşitlenecek. Ayrıca sunucuya takılı ikinci bir veri diskinin (`/dev/vdb`) imajı alınacak.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04, rsync 3.2.7 ve coreutils 9.4 ile denenmiştir. Uzak sunucu yerine aynı makinede ayrı bir portta çalışan bir SSH sunucusu, gerçek disk yerine loop cihazı (`/dev/loop0`) kullanıldı. Komutlar gerçek sunucu ve diskte aynı şekilde çalışır.

## İçindekiler

1. [rsync ile yerel kopyalama](#1-rsync-ile-yerel-kopyalama)
2. [Sondaki / işareti](#2-sondaki--işareti)
3. [Uzak sunucuya yedekleme](#3-uzak-sunucuya-yedekleme)
4. [Sadece değişenler gider](#4-sadece-değişenler-gider)
5. [Uzaktan yerele: geri yükleme](#5-uzaktan-yerele-geri-yükleme)
6. [Silinen dosyalar: --delete ve -n](#6-silinen-dosyalar---delete-ve--n)
7. [Diğer işe yarar seçenekler](#7-diğer-işe-yarar-seçenekler)
8. [dd ile disk imajı almak](#8-dd-ile-disk-imajı-almak)
9. [Disk imajını geri yüklemek](#9-disk-imajını-geri-yüklemek)
10. [dd ile dosya oluşturmak](#10-dd-ile-dosya-oluşturmak)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. rsync ile yerel kopyalama

rsync, aynı makinedeki iki dizin arasında da çalışır. Ağ veya SSH gerekmez:

```bash
rsync -a site/ /yedek/site/
```

`-a` (archive), yedekleme için gereken seçeneklerin kısaltmasıdır: alt dizinlere iner (`-r`), izinleri (`-p`), zaman damgalarını (`-t`), sembolik bağlantıları (`-l`), sahibi ve grubu (`-o`, `-g`) ve özel dosyaları korur. Neredeyse her rsync komutunda bulunur.

`-v` eklersen aktarılan dosyaları yazar:

```bash
rsync -av site/ /yedek/site/
```

Ubuntu'nun minimal kurulumlarında rsync yüklü gelmeyebilir: `sudo apt install rsync`.

---

## 2. Sondaki / işareti

rsync'te **kaynağın** sonundaki `/` sonucu değiştirir:

```bash
rsync -a site/ hedef1/     # site'ın İÇERİĞİNİ kopyalar
rsync -a site  hedef2/     # site dizininin KENDİSİNİ kopyalar
```

Denemedeki sonuç:

```
hedef1/                 hedef2/
├── css/                └── site/
├── img/                    ├── css/
├── index.html              ├── img/
└── kayit.log               ├── index.html
                            └── kayit.log
```

Hedefin sonundaki `/` ise sonucu değiştirmez. `rsync -a site/ hedef4` ile `rsync -a site/ hedef4/` aynı şeyi yaptı.

> [!TIP]
> Kolay hatırlamak için: `site/` "site'ın içindekiler" demek. Bir dizini aynı adla eşitliyorsan en güvenli yazım, iki tarafa da aynı adı ve `/` koymak: `rsync -a site/ yedek/site/`.

---

## 3. Uzak sunucuya yedekleme

Hedefin önüne `kullanıcı@sunucu:` yazınca rsync aktarımı SSH üzerinden yapar:

```bash
rsync -av site/ yedekci@192.0.2.20:/srv/yedek/site/
```

İlk çalıştırmanın çıktısı:

```
sending incremental file list
./
index.html
kayit.log
css/
img/
img/buyuk.bin
img/yeni.bin

sent 9,291,588 bytes  received 149 bytes  18,583,474.00 bytes/sec
total size is 9,288,932  speedup is 1.00
```

İlk seferde her şey gönderildiği için `speedup` 1.00.

Bunun çalışması için:

- Uzak sunucuda **SSH sunucusu** çalışmalı ve sana giriş izni vermeli. SSH anahtarı kuruluysa şifre sormaz, bu da yedeği zamanlanmış bir görev olarak çalıştırmayı mümkün kılar.
- Uzak sunucuda da **rsync yüklü olmalı.** rsync karşı tarafta kendi kopyasını çalıştırır. Yüklü değilse şuna benzer bir hata alırsın:

```
sh: 1: rsync: not found
rsync: connection unexpectedly closed (0 bytes received so far) [sender]
rsync error: error in rsync protocol data stream (code 12)
```

SSH farklı bir portta çalışıyorsa: `rsync -av -e 'ssh -p 2222' site/ yedekci@192.0.2.20:/srv/yedek/site/`.

### Hedefin üst dizinleri var olmalı

rsync hedefte sadece **son** dizini oluşturur. `/srv/yedek/` yoksa `site/`'ı oluşturamaz:

```
rsync: [Receiver] mkdir "/srv/yedek/site" failed: No such file or directory (2)
rsync error: error in file IO (code 11)
```

Ya önce dizini oluştur (`ssh yedekci@192.0.2.20 mkdir -p /srv/yedek`) ya da rsync 3.2.3 ve sonrasında bulunan `--mkpath` seçeneğini kullan:

```bash
rsync -av --mkpath site/ yedekci@192.0.2.20:/srv/yedek/site/
```

---

## 4. Sadece değişenler gider

rsync'i asıl değerli yapan budur. Aynı komutu hiçbir şey değiştirmeden ikinci kez çalıştırdık:

```
sending incremental file list

sent 203 bytes  received 14 bytes  144.67 bytes/sec
total size is 4,288,929  speedup is 19,764.65
```

Hiçbir dosya listelenmedi, birkaç yüz bayt gitti. rsync dosyaların boyutuna ve değişiklik zamanına bakıp aynı olanları atladı.

Sonra `index.html`'e bir satır ekledik:

```
sending incremental file list
index.html
```

Sadece o dosya gitti.

### Değişen dosyanın da sadece değişen kısmı gider

Uzak aktarımda rsync, değişen bir dosyanın bile tamamını göndermez. 1,2 MB'lık `kayit.log`'un sonuna tek satır ekleyip `--stats` ile baktık:

```
Number of regular files transferred: 1
Literal data: 723 bytes
Matched data: 1,288,176 bytes
```

Dosyanın 1,28 MB'ı karşı tarafta zaten vardı (`Matched`), sadece 723 bayt gönderildi (`Literal`).

Yerel kopyalamada bu hesaplama yapılmaz, değişen dosya bütün olarak kopyalanır. Diskten okumak hesaplamaktan zaten hızlı olduğu için rsync bunu bilerek yapar (`Matched data: 0 bytes`). Değişmeyen dosyaların atlanması ise yerelde de geçerlidir.

---

## 5. Uzaktan yerele: geri yükleme

Kaynak ve hedefin yerini değiştirmen yeterli:

```bash
rsync -av yedekci@192.0.2.20:/srv/yedek/site/ site/
```

Kural aynı: önce kaynak, sonra hedef. Sondaki `/` kuralı da aynı şekilde kaynağa uygulanır.

---

## 6. Silinen dosyalar: --delete ve -n

Kaynaktan bir dosya silersen, rsync onu hedeften **silmez**. `css/stil.css`'i silip rsync çalıştırdık, yedekte durmaya devam etti. Bu, yanlışlıkla silinen dosyaları yedekten geri almayı mümkün kılar.

Hedefin kaynağın tam bir aynası olmasını istiyorsan `--delete`:

```bash
rsync -av --delete site/ yedekci@192.0.2.20:/srv/yedek/site/
```
```
sending incremental file list
deleting css/stil.css
```

> [!CAUTION]
> `--delete`, hedefte olup kaynakta olmayan her şeyi siler. Kaynak ve hedefi karıştırırsan veya kaynakta sondaki `/`'yi unutursan, hedefteki dosyaların büyük kısmı gidebilir. `--delete` kullanmadan önce her zaman `-n` ile dene.

### -n: deneme modu

`-n` (`--dry-run`), rsync'in ne yapacağını gösterir ama hiçbir şeyi değiştirmez:

```bash
rsync -avn --delete site/ yedekci@192.0.2.20:/srv/yedek/site/
```
```
sending incremental file list
deleting css/stil.css
...
total size is 4,288,932  speedup is 18,977.58 (DRY RUN)
```

Çıktının sonunda `(DRY RUN)` yazar. Denemeden sonra dosya hala yerindeydi. Listede beklemediğin bir şey yoksa `-n`'yi çıkarıp tekrar çalıştır.

---

## 7. Diğer işe yarar seçenekler

| Seçenek | Ne yapar? |
|---|---|
| `-z` | Aktarım sırasında sıkıştırır. Denemede 1,2 MB'lık metin dosyası için giden veri 1.289.313 bayttan 107.432 bayta düştü. Zaten sıkıştırılmış dosyalarda (resim, video, `.gz`) faydası yok. |
| `-P` | `--progress` ve `--partial` birlikte: ilerleme gösterir, yarıda kesilen aktarımı sonra kaldığı yerden sürdürür. |
| `--exclude='*.log'` | Eşleşen dosyaları atla. |
| `-e 'ssh -p 2222'` | SSH için farklı port veya seçenek. |
| `--mkpath` | Hedefteki eksik üst dizinleri oluştur. |
| `--stats` | Aktarım istatistikleri. |

### Sahiplik

`-a` sahibi korumaya çalışır, ama bunu ancak **hedefte root** olarak yazılıyorsa yapabilir. Denemede UID 1234'e ait bir dosyayı normal bir kullanıcıyla uzak sunucuya gönderdik, karşıda o kullanıcıya ait oldu. Aynı dosyayı root olarak yerel kopyaladığımızda UID 1234 korundu. Bu, [tar'da gördüğümüz](08-arsivleme-ve-sikistirma.md#8-sahiplik-ve-izinler) davranışın aynısı.

---

## 8. dd ile disk imajı almak

dd, bir kaynaktan bayt bayt okuyup bir hedefe yazar. Dosya sistemini, dosyaları, boş alanı ayırt etmez. Bu yüzden bir diskin birebir kopyasını, yani **imajını** almak için kullanılır.

```bash
sudo dd if=/dev/vdb of=vdb-imaj.raw bs=1M status=progress
```

| Parça | Anlamı |
|---|---|
| `if=` | Input file: okunacak kaynak (disk, bölüm veya dosya) |
| `of=` | Output file: yazılacak hedef |
| `bs=1M` | Her seferde 1 MiB oku/yaz. Varsayılan 512 bayttır ve çok yavaştır. |
| `status=progress` | Kopyalama sürerken ilerlemeyi göster |

64 MiB'lık bir diskte çıktı:

```
64+0 records in
64+0 records out
67108864 bytes (67 MB, 64 MiB) copied, 0.0332994 s, 2.0 GB/s
```

`64+0 records`, 1 MiB'lık 64 bloğun tamamının okunup yazıldığı anlamına gelir. İmaj dosyası diskle tam aynı boyutta oldu (67.108.864 bayt), `sha256sum` ile ikisinin özeti de aynı çıktı.

Diske okuma ve yazma root yetkisi gerektirdiği için komut `sudo` ile çalıştırılır. Hangi diskin hangisi olduğundan emin değilsen önce `lsblk` ile bak.

### İmaj almadan önce bağlantıyı kaldır

Bağlı (mount edilmiş) bir disk, sen imajını alırken değişmeye devam edebilir. Sonuçta yarısı eski, yarısı yeni, tutarsız bir imaj çıkar. Önce bağlantıyı kaldır:

```bash
findmnt /dev/vdb          # bağlı mı?
sudo umount /dev/vdb
```

Disk sadece bağlanıp çözülse bile değişir. Denemede imajı aldıktan sonra diski bir kez bağlayıp çözdük, ardından `cmp` ile karşılaştırdık:

```
/dev/loop0 diskimage.raw differ: char 1077, line 1
```

Hiçbir dosyaya dokunmadık, ama ext4 bağlanırken kendi kayıtlarını (son bağlanma zamanı, sayaç) güncelledi. Yani imajın diskle aynı olup olmadığını kontrol edeceksen, bunu diski tekrar bağlamadan yap.

### İmajı sıkıştırmak

Ham imaj, diskin boş alanı dahil tam boyutundadır. Boru ile sıkıştırarak kaydedebilirsin:

```bash
sudo dd if=/dev/vdb bs=1M status=progress | gzip > vdb-imaj.raw.gz
```

Denemede 64 MiB'lık disk 8,5 MB'a indi. `of=` yazılmayınca dd çıktıyı ekrana (stdout) verir, boru onu `gzip`'e aktarır.

---

## 9. Disk imajını geri yüklemek

`if` ve `of`'un yerini değiştirmen yeterli:

```bash
sudo dd if=vdb-imaj.raw of=/dev/vdb bs=1M status=progress
```

Sıkıştırılmış imajdan:

```bash
gunzip -c vdb-imaj.raw.gz | sudo dd of=/dev/vdb bs=1M status=progress
```

Denemede diskin ilk 2 MiB'ını sıfırlarla ezip dosya sistemini bozduk. Bağlamaya çalışınca `wrong fs type, bad option, bad superblock` hatası verdi. İmajı geri yazınca disk bağlandı ve bütün dosyalar yerindeydi.

> [!CAUTION]
> dd **soru sormaz.** `of=` ile yanlış bir disk verirsen, o diskteki her şey anında üzerine yazılır ve geri dönüşü yoktur. Komutu çalıştırmadan önce `lsblk` ile `of=` değerini iki kez kontrol et. Bağlı ve kullanımdaki bir diske, özellikle sistemin kendi diskine asla yazma.

---

## 10. dd ile dosya oluşturmak

dd, belirli boyutta dosya oluşturmak için de sık kullanılır. `count=`, kaç blok kopyalanacağını belirler:

```bash
dd if=/dev/zero of=bos.img bs=1M count=10        # sıfırlarla dolu 10 MiB
dd if=/dev/urandom of=rastgele.bin bs=1M count=3  # rastgele veriyle 3 MiB
```
```
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.00320389 s, 3.3 GB/s
```

`/dev/zero` sonsuz sıfır, `/dev/urandom` sonsuz rastgele veri üretir. `count` verilmezse dd durmaz, disk dolana kadar yazar. Bu tür dosyalar test için, swap dosyası oluşturmak için veya loop cihazı olarak denemelerde disk gibi kullanmak için işe yarar.

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `rsync -av kaynak/ hedef/` | Kaynağın içeriğini hedefe eşitle |
| `rsync -av kaynak hedef/` | Kaynak dizinin kendisini hedefin içine kopyala |
| `rsync -av dizin/ kullanici@sunucu:/yol/` | Uzak sunucuya |
| `rsync -av kullanici@sunucu:/yol/ dizin/` | Uzak sunucudan |
| `rsync -avn ...` | Deneme modu, hiçbir şey değişmez |
| `rsync -av --delete ...` | Hedefi kaynağın aynası yap |
| `rsync -avz ...` | Aktarırken sıkıştır |
| `rsync -aP ...` | İlerleme göster, yarıda kalanı sürdür |
| `rsync -av --mkpath ...` | Eksik üst dizinleri oluştur |
| `rsync -av -e 'ssh -p 2222' ...` | Farklı SSH portu |
| `sudo dd if=/dev/vdb of=imaj.raw bs=1M status=progress` | Disk imajı al |
| `sudo dd if=imaj.raw of=/dev/vdb bs=1M status=progress` | İmajı diske geri yükle |
| `sudo dd if=/dev/vdb bs=1M \| gzip > imaj.raw.gz` | Sıkıştırarak imaj al |
| `dd if=/dev/zero of=dosya bs=1M count=10` | 10 MiB'lık dosya oluştur |
| `lsblk`, `findmnt` | Diskleri ve bağlı olanları gör |

---

## 12. Sık yapılan hatalar

- **Kaynaktaki sondaki `/`'yi karıştırmak.** Hedefte bir fazla veya bir eksik dizin seviyesi oluşur.
- **`--delete`'i `-n` ile denemeden kullanmak.**
- **Uzak sunucuda rsync'in yüklü olduğunu kontrol etmemek.** Sadece SSH yetmez.
- **Hedefin üst dizinlerinin var olduğunu varsaymak.** `--mkpath` veya önce `mkdir -p`.
- **dd'de `bs` vermemek.** Varsayılan 512 bayt, büyük disklerde çok yavaş.
- **Bağlı bir diskin imajını almak.** İmaj tutarsız olabilir, önce `umount`.
- **dd'de `if` ve `of`'u karıştırmak.** İmaj alacakken yedeği diskin üzerine yazarsın. Komutu çalıştırmadan önce `of=` değerini kontrol et.

---

## 13. Alıştırmalar

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

Önceki konu: [Arşivleme ve Sıkıştırma](08-arsivleme-ve-sikistirma.md) · Sonraki konu: [Girdi/Çıktı Yönlendirme](10-yonlendirme.md)
