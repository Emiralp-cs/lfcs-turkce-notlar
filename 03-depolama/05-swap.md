# Swap

> **Resmi sınav maddesi:** Configure and manage swap space

**Swap**, RAM dolduğunda çekirdeğin az kullanılan bellek sayfalarını taşıdığı disk alanıdır. RAM'den çok yavaştır, ama bellek tamamen bittiğinde süreçlerin öldürülmesini (OOM) önler ve uzun süre dokunulmayan verileri RAM'den çıkararak asıl işe yer açar. Swap iki şekilde olabilir: ayrı bir **bölüm** veya normal bir dosya sisteminde bir **dosya**.

Senaryo: Sunucunun hiç swap'ı yok. Test diskindeki 512 MB'lık bölümü swap yapacağız, ayrıca 256 MB'lık bir swap dosyası ekleyeceğiz ve ikisinin de açılışta etkin olmasını sağlayacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde, test diskinin bölümü (`/dev/loop0p2`) ve kök dosya sistemindeki dosyalarla denenmiştir.

## İçindekiler

1. [Mevcut swap'ı görmek](#1-mevcut-swapı-görmek)
2. [Swap bölümü](#2-swap-bölümü)
3. [Swap dosyası](#3-swap-dosyası)
4. [Açılışta etkinleştirmek: fstab](#4-açılışta-etkinleştirmek-fstab)
5. [Öncelik](#5-öncelik)
6. [Swap'ı kapatmak ve kaldırmak](#6-swapı-kapatmak-ve-kaldırmak)
7. [Swap ne kadar kullanılıyor?](#7-swap-ne-kadar-kullanılıyor)
8. [Komut özeti](#8-komut-özeti)
9. [Sık yapılan hatalar](#9-sık-yapılan-hatalar)
10. [Alıştırmalar](#10-alıştırmalar)

---

## 1. Mevcut swap'ı görmek

```bash
swapon --show
free -h
```

Başlangıçta `swapon --show` hiçbir şey yazmadı ve:

```
Swap:             0B          0B          0B
```

`cat /proc/swaps` da aynı bilgiyi verir.

---

## 2. Swap bölümü

Bölümün türünü `Linux swap` yapmak ([Disk Bölümleri](01-disk-bolumleri.md)) iyi bir alışkanlıktır, ama yetmez: önce bölüm swap olarak **biçimlendirilir**, sonra **etkinleştirilir**.

**1. Biçimlendir:**

```bash
sudo mkswap /dev/loop0p2
```
```
Setting up swapspace version 1, size = 512 MiB (536866816 bytes)
no label, UUID=51ef8613-46e4-465f-ae9e-61dd0b293963
```

Çıktıdaki UUID fstab'da kullanılacak. Etiket vermek için `mkswap -L ad`.

**2. Etkinleştir:**

```bash
sudo swapon --verbose /dev/loop0p2
```
```
swapon: /dev/loop0p2: found signature [pagesize=4096, signature=swap]
swapon: /dev/loop0p2: pagesize=4096, swapsize=536870912, devsize=536870912
swapon /dev/loop0p2
```

**3. Kontrol et:**

```bash
swapon --show
free -h | grep Swap
```
```
NAME         TYPE      SIZE USED PRIO
/dev/loop0p2 partition 512M   0B   -2
Swap:          511Mi          0B       511Mi
```

`swapon` ile açılan swap, yeniden başlatınca kapanır. Kalıcı olması için [4. bölüm](#4-açılışta-etkinleştirmek-fstab).

> [!CAUTION]
> `mkswap`, bölümdeki dosya sistemini ve veriyi siler. Doğru bölümde olduğundan `lsblk -f` ile emin ol.

---

## 3. Swap dosyası

Boş bölüm yoksa veya swap'ı sonradan büyütmek istiyorsan, swap bir dosya da olabilir.

**1. Dosyayı oluştur:**

```bash
sudo fallocate -l 256M /swapfile
```

`fallocate` alanı anında ayırır. Bazı dosya sistemlerinde (`fallocate`'i desteklemeyen veya delikli dosya oluşturanlarda) swap dosyası çalışmaz, o zaman `dd` kullanılır:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=256
```

İkisini de denedik, ext4 üzerinde ikisi de çalıştı.

**2. İzinleri ayarla:**

```bash
sudo chmod 600 /swapfile
```

Bu adımı atladığımızda:

```
mkswap: /swapfile: insecure permissions 0644, fix with: chmod 0600 /swapfile
swapon: /swapfile: insecure permissions 0644, 0600 suggested.
```

İki komut da sadece **uyardı**, swap yine de etkinleşti. Ama swap dosyası RAM'den taşınan içeriği (parolalar, anahtarlar dahil) tutar: herkes okuyabilirse herkes bu içeriğe ulaşabilir. `chmod 600` zorunlu değil, ama atlanmamalı.

**3. Biçimlendir ve etkinleştir:**

```bash
sudo mkswap /swapfile
sudo swapon /swapfile
swapon --show
```
```
NAME         TYPE      SIZE USED PRIO
/dev/loop0p2 partition 512M   0B   -2
/swapfile    file      256M   0B   -3
```

Zaten etkin bir swap'ı tekrar açmaya çalışınca: `swapon: /swapfile: swapon failed: Device or resource busy`.

| | Swap bölümü | Swap dosyası |
|---|---|---|
| Kurulum | Boş bir bölüm gerekir | Herhangi bir dosya sisteminde |
| Boyut değiştirmek | Zor (bölümü yeniden boyutlandırmak) | Kolay (kapat, yeniden oluştur) |
| fstab'da | `UUID=...` | Dosyanın yolu |
| Performans | Pratikte aynı | Pratikte aynı |

---

## 4. Açılışta etkinleştirmek: fstab

```bash
sudo cp /etc/fstab /etc/fstab.yedek
echo "UUID=51ef8613-46e4-465f-ae9e-61dd0b293963 none swap defaults 0 0" | sudo tee -a /etc/fstab
echo "/swapfile none swap defaults 0 0" | sudo tee -a /etc/fstab
```

| Alan | Değer |
|---|---|
| 1. Cihaz | Bölüm için `UUID=...`, dosya için yolu |
| 2. Bağlama noktası | `none` |
| 3. Tür | `swap` |
| 4. Seçenekler | `defaults` (veya `pri=10` gibi) |
| 5., 6. | `0 0` |

fstab'ı denemek için swap'ları kapatıp fstab'dan açtık:

```bash
sudo swapoff -a
sudo swapon -a
swapon --show
```

Swap'lar fstab'dan açıldıysa açılışta da açılır. Bölümün UUID'sini unuttuysan: `sudo blkid /dev/loop0p2`.

fstab'ın genel biçimi: [Mount ve fstab](03-mount-ve-fstab.md).

---

## 5. Öncelik

Birden fazla swap varsa çekirdek önce **önceliği yüksek** olanı kullanır. `swapon --show`'daki `PRIO` sütunu. Varsayılan öncelikler negatif sayılardır ve her yeni swap bir öncekinden düşük öncelik alır (-2, -3, -4).

fstab'da `pri=` ile:

```
/swapfile none swap pri=10 0 0
```

`swapon -a` sonrası:

```
NAME         TYPE      SIZE USED PRIO
/dev/loop0p2 partition 512M   0B   -2
/swapfile    file      256M   0B   10
```

Swap dosyası artık önce kullanılır. Hızlı bir diskteki (SSD) swap'a daha yüksek öncelik vermek mantıklıdır. Aynı önceliği alan swap'lar dönüşümlü kullanılır. Elle açarken: `sudo swapon -p 10 /swapfile`.

---

## 6. Swap'ı kapatmak ve kaldırmak

```bash
sudo swapoff /swapfile      # tek bir swap
sudo swapoff -a             # hepsi
```

Swap'taki veri önce RAM'e geri taşınır. RAM'de yeterli yer yoksa `swapoff` başarısız olur.

Swap dosyasını tamamen kaldırmak:

1. `sudo swapoff /swapfile`
2. `/etc/fstab`'daki satırını sil.
3. `sudo rm /swapfile`

Swap dosyasının boyutunu değiştirmek için de aynı yol: kapat, sil, yeni boyutla oluştur, `chmod 600`, `mkswap`, `swapon`.

---

## 7. Swap ne kadar kullanılıyor?

```bash
free -h
swapon --show
vmstat 1 5          # si / so sütunları
```

`free`'deki swap kullanımının yüksek olması tek başına sorun değildir: çekirdek uzun süredir kullanılmayan sayfaları swap'a taşıyıp RAM'i önbellek için kullanmış olabilir. Asıl sorun, `vmstat`'taki `si` ve `so` sütunlarının **sürekli** sıfırdan büyük olmasıdır: bu, sistemin sürekli swap'a yazıp okuduğu, yani RAM'in yetmediği anlamına gelir ([Performans İzleme](../04-temel-komutlar/14-performans-izleme.md#6-hepsine-birden-bakmak-vmstat)).

Çekirdeğin swap'a ne kadar istekli gideceğini `vm.swappiness` belirler ([Kernel Parametreleri](../01-isletim-ve-dagitim/09-kernel-parametreleri.md)).

---

## 8. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `swapon --show`, `free -h`, `cat /proc/swaps` | Swap durumu |
| `sudo mkswap [-L etiket] /dev/X` | Bölümü swap olarak biçimlendir |
| `sudo fallocate -l 1G /swapfile` | Swap dosyası için alan ayır |
| `sudo dd if=/dev/zero of=/swapfile bs=1M count=1024` | Aynısı, dd ile |
| `sudo chmod 600 /swapfile` | Sadece root okuyabilsin |
| `sudo swapon [-p N] /dev/X` | Etkinleştir |
| `sudo swapon -a` | fstab'daki bütün swap'ları aç |
| `sudo swapoff /dev/X`, `swapoff -a` | Kapat |
| `UUID=... none swap defaults 0 0` | fstab satırı (bölüm) |
| `/swapfile none swap pri=10 0 0` | fstab satırı (dosya, öncelikli) |

---

## 9. Sık yapılan hatalar

- **Bölümü `mkswap` olmadan `swapon` ile açmaya çalışmak.**
- **Swap dosyasına `chmod 600` vermemek.**
- **Sadece `swapon` yapıp kalıcı olduğunu sanmak.** fstab gerekir.
- **fstab'da bağlama noktasına bir dizin yazmak.** `none`.
- **fstab'ı `mount -a` ile denemek.** Swap için `swapon -a`.
- **Swap dosyasını `swapoff` yapmadan silmek.**
- **Yanlış bölümde `mkswap` çalıştırmak.**
- **Swap kullanımının yüksek olmasını hemen sorun saymak.** `si`/`so`'ya bak.

---

## 10. Alıştırmalar

**1.** Sistemde etkin swap alanlarını ve toplam swap boyutunu göster.

<details>
<summary>Çözüm</summary>

```bash
swapon --show
free -h
```
</details>

**2.** Bir test diskindeki bölümü (ör. `/dev/loop0p2`) `HizliSwap` etiketiyle swap olarak biçimlendir ve etkinleştir.

<details>
<summary>Çözüm</summary>

```bash
sudo mkswap -L HizliSwap /dev/loop0p2
sudo swapon /dev/loop0p2
swapon --show
```
</details>

**3.** `/swapfile2` adında 300 MB'lık bir swap dosyası oluştur, güvenli izinler ver ve etkinleştir.

<details>
<summary>Çözüm</summary>

```bash
sudo fallocate -l 300M /swapfile2
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
swapon --show
```
</details>

**4.** 2. ve 3. alıştırmadaki swap'ların açılışta etkin olmasını sağla. Dosyanın önceliği bölümden yüksek olsun. Yeniden başlatmadan kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo cp /etc/fstab /etc/fstab.yedek
echo "LABEL=HizliSwap none swap defaults 0 0" | sudo tee -a /etc/fstab
echo "/swapfile2 none swap pri=10 0 0" | sudo tee -a /etc/fstab
sudo swapoff -a
sudo swapon -a
swapon --show          # /swapfile2'nin PRIO değeri 10
```
</details>

**5.** `/swapfile2`'yi tamamen kaldır.

<details>
<summary>Çözüm</summary>

```bash
sudo swapoff /swapfile2
sudo sed -i '\#^/swapfile2 #d' /etc/fstab
sudo rm /swapfile2
swapon --show
```
</details>

---

Önceki konu: [Mount Seçenekleri](04-mount-secenekleri.md) · Sonraki konu: [LVM](06-lvm.md)
