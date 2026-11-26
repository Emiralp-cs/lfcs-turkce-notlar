# Paket Yönetimi

> **Resmi sınav maddesi:** Search for, install, validate, and maintain software packages or repositories

Ubuntu'da yazılımlar **paket** olarak gelir: programın dosyaları, yapılandırması ve hangi başka paketlere ihtiyaç duyduğu bilgisi tek bir `.deb` dosyasında. Paketleri iki araç yönetir: **apt** depolardan indirir ve bağımlılıkları çözer, **dpkg** ise paketleri sisteme kurar ve kurulu paketlerin kaydını tutar. Bu dosya paketleri aramayı, kurmayı, güncellemeyi, kaldırmayı ve bozulup bozulmadıklarını doğrulamayı anlatıyor. Depoların kendisi: [Paket Depoları](07-paket-depolari.md).

Senaryo: Sunucuya yeni araçlar kurulacak, bir paketin hangi dosyaları getirdiği incelenecek, bir dosyanın hangi paketten geldiği bulunacak ve değiştirilmiş sistem dosyaları tespit edilecek.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denenmiştir. Paket kurmak ve kaldırmak `sudo` gerektirir, sadece bilgi almak gerektirmez.

## İçindekiler

1. [apt ve dpkg](#1-apt-ve-dpkg)
2. [Paket listesini güncellemek: apt update](#2-paket-listesini-güncellemek-apt-update)
3. [Kurulu paketleri yükseltmek: apt upgrade](#3-kurulu-paketleri-yükseltmek-apt-upgrade)
4. [Paket aramak ve bilgi almak](#4-paket-aramak-ve-bilgi-almak)
5. [Paket kurmak](#5-paket-kurmak)
6. [Paket kaldırmak: remove, purge, autoremove](#6-paket-kaldırmak-remove-purge-autoremove)
7. [Kurulu paketleri incelemek: dpkg](#7-kurulu-paketleri-incelemek-dpkg)
8. [Paketleri doğrulamak: dpkg -V ve debsums](#8-paketleri-doğrulamak-dpkg--v-ve-debsums)
9. [Sürüm sabitlemek: apt-mark hold](#9-sürüm-sabitlemek-apt-mark-hold)
10. [Bozuk paket durumunu düzeltmek](#10-bozuk-paket-durumunu-düzeltmek)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. apt ve dpkg

| | apt | dpkg |
|---|---|---|
| Ne yapar? | Depolardan bulur, indirir, bağımlılıkları çözer | `.deb` dosyasını kurar, kurulu paketleri kaydeder |
| Bağımlılıklar | Kendisi kurar | Kurmaz, eksikse hata verir |
| Ne zaman? | Neredeyse her zaman | Kurulu paketleri sorgulamak, doğrulamak |

`apt`, `apt-get` ve `apt-cache`'in kullanıcı dostu birleşimidir. Script'lerde, çıktısı değişmediği için hala `apt-get` tercih edilir. Komut satırında `apt` kullan.

---

## 2. Paket listesini güncellemek: apt update

```bash
sudo apt update
```

`apt update` **hiçbir yazılımı güncellemez.** Sadece depolardaki paketlerin güncel listesini indirir. Ardından neyin güncellenebileceğini gösterir:

```bash
apt list --upgradable
```
```
apt/noble-updates,noble-security 2.8.4ubuntu0.1 amd64 [upgradable from: 2.8.3]
ca-certificates-java/noble-updates,noble-security 20240118ubuntu0.24.04.2 all [upgradable from: 20240118]
```

Bir paket kurmadan önce her zaman `apt update` yap, yoksa apt eski listeye göre artık depoda olmayan bir sürümü indirmeye çalışabilir (`404 Not Found`).

Bir depo ulaşılamazsa `apt update` uyarı verir ama durmaz:

```
W: Failed to fetch https://download.docker.com/linux/ubuntu/dists/noble/InRelease  ...
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

---

## 3. Kurulu paketleri yükseltmek: apt upgrade

```bash
sudo apt upgrade                  # kurulu paketleri yükselt
sudo apt update && sudo apt upgrade
```

`&&`, ilk komut başarılıysa ikincisini çalıştırır. Liste indirilemezse yükseltme denenmez.

| Komut | Fark |
|---|---|
| `apt upgrade` | Yükseltir, ama bunun için bir paketi **kaldırmak** gerekiyorsa o paketi yükseltmez |
| `apt full-upgrade` | Gerekirse paket kaldırarak da yükseltir |
| `apt install --only-upgrade paket` | Sadece bu paketi yükselt, kurulu değilse kurma |

Çekirdek güncellemesinden sonra yeni çekirdeğin kullanılması için yeniden başlatmak gerekir ([Açma, Kapatma ve Target'lar](01-acma-kapatma-ve-targetlar.md)). Ubuntu bunu `/var/run/reboot-required` dosyasıyla bildirir.

---

## 4. Paket aramak ve bilgi almak

```bash
apt search nginx                    # adında veya açıklamasında nginx geçen paketler
apt search --names-only '^tree$'    # sadece adında, düzenli ifadeyle
apt show tree                       # ayrıntılı bilgi
apt-cache policy tree               # kurulu ve depodaki sürümler
```
```
Package: tree
Version: 2.1.1-2ubuntu3.24.04.2
Depends: libc6 (>= 2.38)
Description: displays an indented directory tree, in color
```
```
tree:
  Installed: 2.1.1-2ubuntu3.24.04.2
  Candidate: 2.1.1-2ubuntu3.24.04.2
```

`apt search` açıklamalarda da aradığı için genelde çok uzun bir liste verir. Paketin adını yaklaşık biliyorsan `--names-only`.

| Soru | Komut |
|---|---|
| Bu iş için hangi paket var? | `apt search anahtar-kelime` |
| Paket ne işe yarıyor, neye bağımlı? | `apt show paket` |
| Hangi sürüm kurulu, depoda hangisi var? | `apt-cache policy paket` |
| Kurulu mu? | `dpkg -l paket`, `apt list --installed paket` |
| Bu dosya hangi paketten geldi? | `dpkg -S /yol/dosya` |
| Kurulu olmayan bir komut hangi pakette? | Komutu yazınca Ubuntu önerir (`command-not-found`) |

---

## 5. Paket kurmak

```bash
sudo apt install tree
sudo apt install nginx curl git        # birden fazla
sudo apt install -y tree               # onay sorma
sudo apt install nginx=1.24.0-2ubuntu7 # belirli bir sürüm
sudo apt install --reinstall tree      # yeniden kur (bozuk dosyaları geri getirir)
```

apt, ek olarak kurulacak bağımlılıkları listeler ve onay ister:

```
The following NEW packages will be installed:
  tree
0 upgraded, 1 newly installed, 0 to remove and 66 not upgraded.
```

### İndirilmiş bir .deb dosyasını kurmak

```bash
sudo apt install ./sl_5.02-1_amd64.deb    # bağımlılıklarıyla birlikte
```

Başındaki `./` önemli: olmazsa apt bunu depodaki bir paket adı sanar. `dpkg -i dosya.deb` de kurar, ama eksik bağımlılıkları kurmaz ve hata verir.

Kurmadan önce bir `.deb`'in içine bakmak:

```bash
dpkg -I sl_5.02-1_amd64.deb      # bilgisi ve bağımlılıkları
dpkg -c sl_5.02-1_amd64.deb      # içindeki dosyalar
```
```
 Package: sl
 Version: 5.02-1
 Depends: libc6 (>= 2.2.5), libncurses6 (>= 6), libtinfo6 (>= 6)
```

Depodaki bir paketi kurmadan sadece indirmek için: `apt download paket`.

---

## 6. Paket kaldırmak: remove, purge, autoremove

| Komut | Programı siler | Yapılandırma dosyalarını siler | Bağımlılıkları siler |
|---|---|---|---|
| `apt remove paket` | Evet | Hayır | Hayır |
| `apt purge paket` | Evet | Evet | Hayır |
| `apt autoremove` | | | Artık hiçbir paketin ihtiyaç duymadığı bağımlılıkları |

`at` paketiyle denedik:

```bash
sudo apt remove at
dpkg -l at
ls /etc/at.deny
```
```
rc  at             3.2.5-2.1ubuntu3 amd64        Delayed job execution and batch processing
/etc/at.deny
```

Durum `rc`: **r**emoved (kaldırılmış), **c**onfig-files (yapılandırma dosyaları duruyor). `/etc/at.deny` hala yerinde. Paket tekrar kurulursa eski ayarlarla gelir.

```bash
sudo apt purge at
```
```
Purging configuration files for at (3.2.5-2.1ubuntu3) ...
```

Bundan sonra `dpkg -l at` paketi hiç bulamadı ve `/etc/at.deny` silindi. Bir yazılımı tamamen ve temiz kaldırmak istiyorsan `purge`, ardından `autoremove`.

> [!CAUTION]
> `apt remove` ve `apt autoremove`, kaldırılacak paketleri listeler ve onay ister. Listeyi **okumadan** `y` deme: önemli bir paketi kaldırmak, ona bağlı başka paketleri de (örneğin ağ yöneticisini veya `sudo`'yu) kaldırabilir.

---

## 7. Kurulu paketleri incelemek: dpkg

```bash
dpkg -l                  # bütün kurulu paketler
dpkg -l 'nginx*'         # desene uyanlar
dpkg -L tree             # paketin kurduğu dosyalar (--listfiles)
dpkg -S /usr/bin/tree    # bu dosya hangi paketten (--search)
```
```
ii  tree           2.1.1-2ubuntu3.24.04.2 amd64        displays an indented directory tree, in color
tree: /usr/bin/tree
```

`dpkg -l` çıktısının başındaki iki harf:

| Harf | Anlamı |
|---|---|
| `ii` | İstenmiş ve kurulu (normal durum) |
| `rc` | Kaldırılmış, yapılandırma dosyaları duruyor |
| `iU`, `iF` | Kurulumu yarım kalmış veya yapılandırma başarısız ([10. bölüm](#10-bozuk-paket-durumunu-düzeltmek)) |
| `hi` | Kurulu ve sürümü sabitlenmiş (hold) |

`dpkg -L` ve `dpkg -S` birbirinin tersidir: "bu paket ne getirdi?" ve "bu dosya nereden geldi?".

```bash
dpkg -S /etc/crontab
```
```
cron-daemon-common: /etc/crontab
```

Bir yapılandırma dosyasını bozduysan, hangi paketten geldiğini bulup o paketi yeniden kurmak dosyayı geri getirebilir. Yapılandırma dosyaları için bu varsayılan olarak geçerli değildir, dpkg senin değişikliğini korur.

---

## 8. Paketleri doğrulamak: dpkg -V ve debsums

Bir paketin dosyalarının kurulduğu andaki halinden değişip değişmediğini kontrol etmek, bir dosyanın bozulduğunu veya izinsiz değiştirildiğini anlamanın yoludur.

```bash
sudo dpkg -V tree        # (--verify)
```

Hiçbir şey değişmemişse çıktı vermez. `/usr/bin/tree` dosyasını bilerek değiştirdik:

```
??5??????   /usr/bin/tree
```

Her sütundaki karakter bir kontroldür. `5`, dosyanın **MD5 özetinin değiştiği** anlamına gelir. `?` o kontrolün yapılamadığını, `.` geçtiğini gösterir. Yapılandırma dosyaları `c` ile işaretlenir:

```bash
sudo dpkg -V cron-daemon-common
```
```
??5?????? c /etc/crontab
```

Argüman verilmezse `dpkg -V` bütün kurulu paketleri kontrol eder.

Bozulan dosyayı geri getirmek için paketi yeniden kur:

```bash
sudo apt install --reinstall tree
sudo dpkg -V tree        # artık çıktı yok
```

> [!NOTE]
> Bu notun denendiği ortamda `dpkg -V` ayrıca `missing /usr/share/doc/...` ve `missing /usr/share/man/...` satırları gösterdi. Ubuntu'nun minimal kurulumları (container'lar, bazı bulut imajları) yer kazanmak için belge dosyalarını kurmaz, bu satırlar bir sorun değildir.

### debsums

`debsums` paketi aynı işi daha okunabilir yapar (`sudo apt install debsums`):

```bash
sudo debsums -s tree       # sadece hataları göster (-s: silent)
sudo debsums -c            # değişmiş dosyaları listele
sudo debsums -ce           # sadece değişmiş yapılandırma dosyaları
```
```
/etc/bash.bashrc
/etc/crontab
/etc/fuse.conf
```

`debsums -ce` bir sistemde hangi yapılandırma dosyalarının varsayılandan farklı olduğunu hızlıca gösterir. Yeni devraldığın bir sunucuda neyin değiştirildiğini anlamak için çok işe yarar.

---

## 9. Sürüm sabitlemek: apt-mark hold

Bir paketin sürümü, güncellemede değişmesin isteniyorsa (yeni sürümde uyumsuzluk varsa):

```bash
sudo apt-mark hold tree
apt-mark showhold
sudo apt-mark unhold tree
```
```
tree set on hold.
tree
Canceled hold on tree.
```

Sabitlenen paketler `apt upgrade` sırasında `The following packages have been kept back` listesinde görünür.

---

## 10. Bozuk paket durumunu düzeltmek

Bir kurulum yarıda kesilirse (bağlantı koptu, elektrik gitti) veya bir paketin kurulum script'i hata verirse, apt sonraki her komutta hata verir:

```
dpkg: error processing package openssh-server (--configure):
 installed openssh-server package post-installation script subprocess returned error exit status 1
Errors were encountered while processing:
 openssh-server
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

Bu notu hazırlarken tam olarak bu oldu: yarım kalmış bir `openssh-server` kurulumu, alakasız bir paketin kaldırılmasını da engelledi. Çözüm sırası:

```bash
sudo dpkg --configure -a     # yarım kalan yapılandırmaları tamamla
sudo apt install -f          # eksik bağımlılıkları kur (--fix-broken)
```

Hata mesajı, script'in neden başarısız olduğunu genelde birkaç satır yukarıda yazar. Önce o sorunu çöz, sonra `dpkg --configure -a`'yı tekrar çalıştır.

Başka bir apt işlemi sürerken ikinci bir tane başlatırsan `Could not get lock /var/lib/dpkg/lock-frontend` hatası alırsın. Kilit dosyasını silme, diğer işlemin (genelde arka planda çalışan otomatik güncelleme) bitmesini bekle: `ps aux | grep -E 'apt|dpkg'`.

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `apt update` | Paket listesini yenile |
| `apt list --upgradable` | Güncellenebilecekler |
| `apt upgrade` / `full-upgrade` | Yükselt / gerekirse kaldırarak yükselt |
| `apt search kelime`, `--names-only` | Ara |
| `apt show paket` | Bilgi |
| `apt-cache policy paket` | Kurulu ve aday sürüm |
| `apt install paket`, `paket=sürüm` | Kur |
| `apt install ./dosya.deb` | Yerel paketi bağımlılıklarıyla kur |
| `apt install --reinstall paket` | Yeniden kur |
| `apt remove` / `purge` / `autoremove` | Kaldır / yapılandırmayla kaldır / gereksizleri kaldır |
| `apt download paket` | Sadece indir |
| `dpkg -l [desen]` | Kurulu paketler |
| `dpkg -L paket` | Paketin dosyaları |
| `dpkg -S /yol/dosya` | Dosyanın paketi |
| `dpkg -I` / `-c dosya.deb` | `.deb`'in bilgisi / içeriği |
| `dpkg -V [paket]` | Değişmiş dosyaları bul |
| `debsums -s`, `debsums -ce` | Değişmiş dosyalar, yapılandırmalar |
| `apt-mark hold` / `unhold` / `showhold` | Sürüm sabitle |
| `dpkg --configure -a`, `apt install -f` | Yarım kalan kurulumu düzelt |

---

## 12. Sık yapılan hatalar

- **`apt update`'in yazılımları güncellediğini sanmak.** Sadece listeyi yeniler.
- **`apt update` yapmadan kurmak.** `404 Not Found`.
- **`remove` ile yapılandırma dosyalarının da silindiğini sanmak.** `purge` gerekir.
- **`apt remove`/`autoremove`'un kaldıracağı listeyi okumadan onaylamak.**
- **`apt install dosya.deb` yazıp `./`'yi unutmak.**
- **`dpkg -i` ile kurup bağımlılık hatasında takılmak.** `apt install -f` veya baştan `apt install ./dosya.deb`.
- **apt kilit dosyalarını elle silmek.**
- **Script'lerde `apt` kullanmak.** Çıktısı kararlı değil, `apt-get` kullan.

---

## 13. Alıştırmalar

**1.** Paket listesini güncelle, güncellenebilecek paketleri listele ve `htop` paketinin kurulu olup olmadığını, depodaki sürümünü kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo apt update
apt list --upgradable
apt-cache policy htop
```
</details>

**2.** `/usr/bin/ssh` dosyasının hangi paketten geldiğini bul. Sonra o paketin kurduğu bütün dosyalardan sadece `/usr/bin` altındakileri listele.

<details>
<summary>Çözüm</summary>

```bash
dpkg -S /usr/bin/ssh                       # openssh-client
dpkg -L openssh-client | grep '^/usr/bin'
```
</details>

**3.** `at` paketini kur, sonra kaldır ama yapılandırma dosyaları kalsın. `dpkg -l` ile durumunu ve `/etc/at.deny`'ın durduğunu gör. Sonra tamamen temizle.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install -y at
sudo apt remove -y at
dpkg -l at              # rc
ls /etc/at.deny         # duruyor
sudo apt purge -y at
dpkg -l at              # no packages found matching at
ls /etc/at.deny         # No such file or directory
```
</details>

**4.** Sistemde, kurulduğu andan bu yana değiştirilmiş yapılandırma dosyalarını listele. Sonra `/etc/crontab`'ın değişip değişmediğini sadece ilgili paket üzerinden kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install -y debsums
sudo debsums -ce
dpkg -S /etc/crontab                       # paketin adını bul
sudo dpkg -V cron-daemon-common            # çıktıda /etc/crontab varsa değişmiş
```
</details>

**5.** `nginx` paketini, güncellemelerde sürümü değişmeyecek şekilde sabitle, sabitlendiğini göster, sonra sabitlemeyi kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo apt-mark hold nginx
apt-mark showhold
sudo apt-mark unhold nginx
```
</details>

---

Önceki konu: [Zamanlanmış Görevler](05-zamanlanmis-gorevler.md) · Sonraki konu: [Paket Depoları](07-paket-depolari.md)
