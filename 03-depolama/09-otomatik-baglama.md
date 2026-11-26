# Otomatik Bağlama (autofs)

> **Resmi sınav maddesi:** Configure filesystem automounters

fstab'daki bir satır, dosya sistemini açılışta bağlar ve kapanana kadar bağlı tutar. Bu, her zaman erişilebilen yerel diskler için uygundur. Ama bir NFS sunucusu kapalıysa açılış yavaşlar veya takılır. Nadiren kullanılan bir disk de boşuna sürekli bağlı kalır.

**autofs** başka türlü çalışır: bir dizine **erişildiği anda** bağlar ve bir süre kullanılmayınca **kendisi ayırır.** Sunucu kapalıysa sadece o dizine erişim başarısız olur, sistemin geri kalanı etkilenmez.

Senaryo: Sunucuda üç şey otomatik bağlanacak:

- Arşiv diski (`/dev/loop5`, ext4): `/arsiv/disk1`'e erişildiğinde bağlansın.
- `/srv/projeler` altındaki her proje: `/proje/<proje-adı>`'na erişildiğinde bağlansın. Her proje için ayrı satır yazmak istemiyoruz.
- `/srv/raporlar`: tam olarak `/veri/raporlar` yoluna, salt okunur bağlansın.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve autofs 5.1.9 üzerinde denendi. Deneme ortamında NFS desteği olmadığı için ([NFS](07-nfs.md)), bağlanan şeyler yerel bir disk ve bind mount'lardı. autofs için hepsi aynı şekilde çalışır, değişen sadece satırın sonundaki kaynak. **NFS kaynaklı satırlar ve `/net` denenmedi.**

## İçindekiler

1. [Kurulum ve iki dosya mantığı](#1-kurulum-ve-iki-dosya-mantığı)
2. [Dolaylı harita (indirect map)](#2-dolaylı-harita-indirect-map)
3. [Bağlamayı görmek ve zaman aşımı](#3-bağlamayı-görmek-ve-zaman-aşımı)
4. [Joker karakter: * ve &](#4-joker-karakter--ve-)
5. [Doğrudan harita (direct map)](#5-doğrudan-harita-direct-map)
6. [Dizinleri önceden göstermek: --ghost](#6-dizinleri-önceden-göstermek---ghost)
7. [Değişiklikleri uygulamak: reload mu, restart mı?](#7-değişiklikleri-uygulamak-reload-mu-restart-mı)
8. [NFS ile kullanmak](#8-nfs-ile-kullanmak)
9. [Sorun giderme](#9-sorun-giderme)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Kurulum ve iki dosya mantığı

```bash
sudo apt install autofs
```

autofs iki tür dosya kullanır:

| Dosya | Ne söyler? |
|---|---|
| **Ana harita** (master map): `/etc/auto.master` ve `/etc/auto.master.d/*.autofs` | Hangi dizini autofs yönetecek ve ayrıntılar hangi dosyada |
| **Harita** (map): ör. `/etc/auto.arsiv` | O dizinin altında ne, nereden, hangi seçeneklerle bağlanacak |

Ubuntu'daki `/etc/auto.master` neredeyse tamamen yorum satırıdır. İçinde çalışan tek önemli satır:

```
+dir:/etc/auto.master.d
```

Bu satır, `/etc/auto.master.d/` içindeki `.autofs` uzantılı dosyaları da ana haritaya ekler. Ana dosyayı değiştirmek yerine oraya ayrı bir dosya koymak daha düzenlidir. Dosya adı mutlaka **`.autofs`** ile bitmeli.

---

## 2. Dolaylı harita (indirect map)

Ana haritaya satır ekle:

```bash
echo '/arsiv  /etc/auto.arsiv  --timeout=20' | sudo tee /etc/auto.master.d/arsiv.autofs
```

Satırın anlamı: **`/arsiv` dizinini autofs yönetsin, altındaki her şey `/etc/auto.arsiv`'de yazılı, 20 saniye kullanılmayan bağlamaları ayır.**

Haritayı yaz:

```bash
echo 'disk1   -fstype=ext4,rw   :/dev/loop5' | sudo tee /etc/auto.arsiv
```

Harita satırının üç sütunu var:

```
disk1     -fstype=ext4,rw     :/dev/loop5
  │             │                  │
anahtar      seçenekler          kaynak
```

| Sütun | Anlamı |
|---|---|
| Anahtar | `/arsiv`'in altında oluşacak dizinin adı. Tam yol: `/arsiv/disk1` |
| Seçenekler | `-` ile başlar. `-fstype=` dosya sistemi türü, sonra mount seçenekleri |
| Kaynak | Yerel bir cihaz `:/dev/...` diye, **iki nokta ile** başlar. NFS için `sunucu:/dizin` |

Bu türe **dolaylı** (indirect) denir: ana haritada sadece üst dizin (`/arsiv`) yazılır, alt dizinler haritadan gelir.

Servisi başlat:

```bash
sudo systemctl enable --now autofs
```

`/arsiv` dizinini oluşturmana gerek yok, autofs kendisi oluşturur.

---

## 3. Bağlamayı görmek ve zaman aşımı

Henüz kimse erişmedi:

```bash
ls -la /arsiv
findmnt -R /arsiv
```
```
total 4
drwxr-xr-x  2 root root    0 Nov 26 09:49 .
drwxr-xr-x 26 root root 4096 Nov 26 09:49 ..
TARGET SOURCE          FSTYPE OPTIONS
/arsiv /etc/auto.arsiv autofs rw,relatime,fd=7,pgrp=12061,timeout=20,minproto=5,maxproto=5,indirect,pipe_ino=103966
```

`/arsiv` boş görünüyor ve `disk1` yok. Bağlı olan tek şey autofs'in kendisi. Şimdi tam yola eriş:

```bash
cat /arsiv/disk1/OKUBENI.txt
findmnt -R /arsiv
```
```
arsiv diskindeki dosya
TARGET         SOURCE          FSTYPE OPTIONS
/arsiv         /etc/auto.arsiv autofs rw,relatime,fd=7,pgrp=12061,timeout=20,minproto=5,maxproto=5,indirect,pipe_ino=103966
`-/arsiv/disk1 /dev/loop5      ext4   rw,relatime
```

Erişim anında disk bağlandı. 35 saniye bekledikten sonra `findmnt -R /arsiv` tekrar sadece ilk satırı gösterdi: 20 saniyelik zaman aşımı dolunca disk ayrıldı.

> [!IMPORTANT]
> Bağlama, **`ls /arsiv`** ile değil, **`ls /arsiv/disk1`** ile tetiklenir. Üst dizine bakınca hiçbir şey görmemek normaldir. Bölüm 6'da bunun nasıl değiştirileceği var.

Haritada olmayan bir ada erişince:

```
ls: cannot access '/arsiv/disk9': No such file or directory
```

**Kullanımdaki bir bağlama ayrılmaz.** Bir süreç `/arsiv/disk1` içinde dururken 35 saniye bekledik, disk bağlı kaldı. Süreç çıkıp zaman aşımı dolunca ayrıldı.

Zaman aşımı yazılmazsa `/etc/autofs.conf`'taki değer geçerlidir: `timeout = 300` (5 dakika).

---

## 4. Joker karakter: * ve &

`/srv/projeler` altında `alfa`, `beta` ve sonra eklenecek başka projeler var. Her biri için ayrı satır yazmak yerine:

```
# /etc/auto.master.d/proje.autofs
/proje  /etc/auto.proje  --timeout=20
```
```
# /etc/auto.proje
*   -fstype=bind   :/srv/projeler/&
```

`*` her anahtara uyar, `&` ise erişilen anahtarla değiştirilir. `/proje/beta`'ya erişince `:/srv/projeler/beta` bağlanır:

```bash
cat /proje/beta/not.txt
findmnt -R /proje
```
```
beta notlari
TARGET        SOURCE                       FSTYPE OPTIONS
/proje        /etc/auto.proje              autofs rw,relatime,...,indirect,...
`-/proje/beta /dev/vda[/srv/projeler/beta] ext4   rw,relatime,...
```

Var olmayan bir projeye (`/proje/gama`) erişince bağlama başarısız olur ve dizin yok görünür. Yeni bir proje dizini oluşturulduğunda haritaya dokunmaya gerek yoktur.

`-fstype=bind`, bir dizini başka bir yere bağlar (`mount --bind` ile aynı). NFS'te aynı mantık şöyle yazılır: `* -fstype=nfs4,rw 10.0.5.10:/srv/ev/&`. Kullanıcıların ev dizinleri için sık kullanılır.

---

## 5. Doğrudan harita (direct map)

Dolaylı haritada bağlamalar hep aynı üst dizinin altındadır. Bağlama noktası tam bir yol olacaksa (`/veri/raporlar`) **doğrudan** (direct) harita kullanılır. Ana haritada dizin yerine **`/-`** yazılır:

```
# /etc/auto.master.d/dogrudan.autofs
/-  /etc/auto.dogrudan  --timeout=20
```

Haritada anahtar, **tam yoldur**:

```
# /etc/auto.dogrudan
/veri/raporlar   -fstype=bind,ro   :/srv/raporlar
```

```bash
sudo systemctl reload autofs
cat /veri/raporlar/kasim.txt
touch /veri/raporlar/yeni
```
```
kasim raporu
touch: cannot touch '/veri/raporlar/yeni': Read-only file system
```

`ro` seçeneği uygulandı.

| | Dolaylı | Doğrudan |
|---|---|---|
| Ana haritada | Üst dizin: `/arsiv` | `/-` |
| Haritada anahtar | Alt dizin adı: `disk1` | Tam yol: `/veri/raporlar` |
| Joker (`*`) | Kullanılabilir | Kullanılamaz |
| autofs dizini | Üst dizin tamamen autofs'in olur | Sadece o yol |

> [!WARNING]
> Dolaylı haritada üst dizin (`/arsiv`) tamamen autofs'e ait olur. Bu dizinde önceden dosya varsa görünmez olur. Denemede içinde `eski-dosya.txt` olan bir dizini dolaylı harita yaptık: `ls` sadece haritadaki anahtarları gösterdi, dosya görünmedi. Satırı silip autofs'i yeniden başlatınca geri geldi. `/home` veya `/srv` gibi dolu bir dizini dolaylı harita yapma.

---

## 6. Dizinleri önceden göstermek: --ghost

Varsayılan olarak `ls /arsiv` boş görünür. Kullanıcı neye erişebileceğini bilemez. `--ghost` (veya `--browse`) seçeneği, haritadaki anahtarları bağlamadan dizin olarak gösterir:

```
/arsiv  /etc/auto.arsiv  --timeout=20 --ghost
```

```bash
sudo systemctl restart autofs
ls -l /arsiv
```
```
total 0
drwxr-xr-x 2 root root 0 Nov 26 09:51 disk1
drwxr-xr-x 2 root root 0 Nov 26 09:51 projeler-alfa
```

`ls -l` dizinleri gösterdi ama hiçbirini bağlamadı. Bağlama yine içlerine girildiğinde olur.

`--ghost`, **joker (`*`) haritalarında çalışmaz.** autofs hangi anahtarların var olduğunu bilemez. Denemede `/proje` haritasına `--ghost` ekledik, `ls /proje` yine boş göründü.

Bütün haritalar için `/etc/autofs.conf`'ta `browse_mode = yes` yazılabilir. Ubuntu'da varsayılan `browse_mode = no`.

---

## 7. Değişiklikleri uygulamak: reload mu, restart mı?

Denemede üç durum birbirinden farklı davrandı:

| Değişiklik | Ne gerekti? |
|---|---|
| Var olan bir haritaya yeni satır eklemek (`/etc/auto.arsiv`'e `projeler-alfa`) | **Hiçbir şey.** Hemen erişilebildi |
| Ana haritaya yeni bir satır eklemek (`/proje`) | **`reload`.** Öncesinde `/proje` yoktu |
| Ana haritadaki var olan bir satırın seçeneğini değiştirmek (`--ghost` eklemek) | **`restart`.** `reload` sonrası değişmedi (iki kez denendi) |

Kısacası emin değilsen `sudo systemctl restart autofs`. `restart` bağlı olan her şeyi ayırır, kullanımdaki bağlamalar olmadığı bir anda yap.

---

## 8. NFS ile kullanmak

autofs'in en sık kullanıldığı yer NFS'tir. Satırlar aynı mantıkla yazılır:

```
# /etc/auto.master.d/ofis.autofs
/ofis  /etc/auto.ofis  --timeout=60
```
```
# /etc/auto.ofis
belgeler   -fstype=nfs4,rw   10.0.5.10:/srv/paylasim/belgeler
yedek      -fstype=nfs4,ro   10.0.5.10:/srv/paylasim/yedek
```

NFS kaynağında başta `:` **yoktur**, sunucu adı veya IP vardır. İstemcide `nfs-common` kurulu olmalıdır.

### /net: bütün sunuculara hazır erişim

`/etc/auto.master`'da yorum satırı olarak duran özel bir harita:

```
/net  -hosts
```

Bu satırın yorumu kaldırılırsa `/net/10.0.5.10/` dizinine erişmek, o sunucunun paylaştığı bütün dizinleri `/net/10.0.5.10/srv/paylasim/...` altına bağlar. Haritaya tek tek yazmak gerekmez. Dosyadaki not: bu şekilde yapılan bağlamalara `nosuid` ve `nodev` uygulanır.

---

## 9. Sorun giderme

Haritaya hatalı bir satır ekledik (disk ext4 ama `xfs` yazdık):

```
bozuk  -fstype=xfs  :/dev/loop5
```
```
ls: cannot access '/arsiv/bozuk': No such file or directory
```

Hata mesajı sadece "dizin yok" diyor. Varsayılan ayarda autofs günlüğe de bir şey yazmadı. Sadece çekirdek mesajı vardı: `XFS (loop5): Invalid superblock magic number`.

Ayrıntılı günlük için `/etc/autofs.conf`'ta:

```
logging = verbose
```

```bash
sudo systemctl restart autofs
ls /arsiv/bozuk
sudo journalctl -u autofs -n 5
```
```
automount[12229]: attempting to mount entry /arsiv/bozuk
automount[12229]: mount(generic): failed to mount /dev/loop5 (type xfs) on /arsiv/bozuk
automount[12229]: failed to mount /arsiv/bozuk
```

autofs'in okuduğu bütün haritaları ve satırları görmek için:

```bash
sudo automount -m
```
```
Mount point: /arsiv
source(s):
  instance type(s): file
  map: /etc/auto.arsiv
  disk1 | -fstype=ext4,rw   :/dev/loop5
  projeler-alfa | -fstype=bind  :/srv/projeler/alfa
  bozuk | -fstype=xfs  :/dev/loop5
Mount point: /-
  ...
  /veri/raporlar | -fstype=bind,ro   :/srv/raporlar
Mount point: /proje
  ...
  * | -fstype=bind   :/srv/projeler/&
```

Bir harita bu listede yoksa, ya ana haritadaki satır yanlış ya da dosya adı `.autofs` ile bitmiyor.

---

## 10. Komut özeti

| Komut veya dosya | Ne yapar? |
|---|---|
| `sudo apt install autofs` | Kurulum |
| `/etc/auto.master.d/ad.autofs` | Ana harita satırı: `dizin harita --timeout=N` |
| `/etc/auto.ad` | Harita: `anahtar -fstype=tür,seçenekler kaynak` |
| `:/dev/sdb1` | Yerel cihaz kaynağı |
| `sunucu:/dizin` | NFS kaynağı |
| `-fstype=bind :/dizin` | Bir dizini bağlamak |
| `* ... :/yol/&` | Joker |
| `/-` | Doğrudan harita |
| `--ghost` | Anahtarları bağlamadan göster |
| `sudo systemctl reload autofs` | Yeni ana harita satırlarını oku |
| `sudo systemctl restart autofs` | Her şeyi yeniden oku |
| `sudo automount -m` | Okunan haritaları göster |
| `logging = verbose` | `/etc/autofs.conf`'ta ayrıntılı günlük |

---

## 11. Sık yapılan hatalar

- **Bağlamayı `ls /arsiv` ile kontrol etmek.** Tam yola (`/arsiv/disk1`) erişmek gerekir.
- **Yerel cihaz kaynağında baştaki `:`'yi unutmak.** `:/dev/loop5`, `/dev/loop5` değil.
- **NFS kaynağının başına `:` koymak.**
- **`/etc/auto.master.d/` içindeki dosyayı `.autofs` uzantısı olmadan kaydetmek.** Okunmaz.
- **Seçenekleri `-` olmadan yazmak.**
- **Doğrudan haritada ana harita satırına `/-` yerine bir dizin yazmak veya haritada tam yol yerine kısa ad kullanmak.**
- **Dolu bir dizini (`/home`) dolaylı harita yapmak.** İçindekiler görünmez olur.
- **Aynı yeri hem fstab'da hem autofs'te tanımlamak.**
- **Ana harita satırını değiştirip sadece `reload` yapmak.** `restart` gerekebilir.

---

## 12. Alıştırmalar

**1.** `/dev/vdb1` (ext4) diski, `/yedek/gunluk` dizinine erişildiğinde bağlansın ve 60 saniye kullanılmayınca ayrılsın.

<details>
<summary>Çözüm</summary>

```bash
echo '/yedek  /etc/auto.yedek  --timeout=60' | sudo tee /etc/auto.master.d/yedek.autofs
echo 'gunluk  -fstype=ext4,rw  :/dev/vdb1' | sudo tee /etc/auto.yedek
sudo systemctl enable autofs
sudo systemctl restart autofs
ls /yedek/gunluk
findmnt -R /yedek
```
</details>

**2.** NFS sunucusu `192.168.50.10`'un `/srv/ev/<kullanıcı>` dizinleri, istemcide `/ev/<kullanıcı>`'ya otomatik bağlansın. Her kullanıcı için ayrı satır yazma.

<details>
<summary>Çözüm</summary>

```bash
echo '/ev  /etc/auto.ev' | sudo tee /etc/auto.master.d/ev.autofs
echo '*  -fstype=nfs4,rw  192.168.50.10:/srv/ev/&' | sudo tee /etc/auto.ev
sudo systemctl restart autofs
ls /ev/ayse
```
</details>

**3.** `/srv/belgeler` dizini tam olarak `/kurum/belgeler` yoluna salt okunur bağlansın.

<details>
<summary>Çözüm</summary>

```bash
echo '/-  /etc/auto.kurum' | sudo tee /etc/auto.master.d/kurum.autofs
echo '/kurum/belgeler  -fstype=bind,ro  :/srv/belgeler' | sudo tee /etc/auto.kurum
sudo systemctl restart autofs
ls /kurum/belgeler
```
</details>

**4.** Alıştırma 1'deki `/yedek` dizinine bakan kullanıcılar `gunluk`'u görebilsin, ama içine girilmeden bağlanmasın.

<details>
<summary>Çözüm</summary>

`/etc/auto.master.d/yedek.autofs`:

```
/yedek  /etc/auto.yedek  --timeout=60 --ghost
```

```bash
sudo systemctl restart autofs
ls /yedek            # gunluk görünür
findmnt -R /yedek    # ama bağlı değil
```
</details>

**5.** Bir haritadaki satır çalışmıyor ve `ls` sadece "No such file or directory" diyor. Sebebi nasıl bulursun?

<details>
<summary>Çözüm</summary>

```bash
sudo automount -m                     # satır okunmuş mu?
sudo sed -i 's/^#\?logging = .*/logging = verbose/' /etc/autofs.conf
sudo systemctl restart autofs
ls /yol/anahtar
sudo journalctl -u autofs -n 20       # mount hatası
sudo dmesg | tail                     # çekirdek tarafındaki hata
```
</details>

---

Önceki konu: [NBD](08-nbd.md) · Sonraki konu: [Sanal Dosya Sistemleri](10-sanal-dosya-sistemleri.md)
