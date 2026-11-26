# Kernel Parametreleri

> **Resmi sınav maddesi:** Configure kernel parameters, persistent and non-persistent

Çekirdeğin bellek yönetimi, ağ davranışı ve dosya sistemi gibi birçok ayarı, sistem çalışırken değiştirilebilir. Bu ayarlara **kernel runtime parametreleri** denir ve `sysctl` ile yönetilir. Sınavda neredeyse her zaman iki parçalı sorulur: parametreyi **şimdi** değiştir (geçici) ve yeniden başlatmadan sonra da **kalıcı** olsun.

Senaryo: Sunucuda bol RAM var ve swap'a gereksiz yere başvurulmasını istemiyoruz, bu yüzden `vm.swappiness` değerini düşüreceğiz. Ayrıca sunucu bir yönlendirici olarak da kullanılacak, IP yönlendirmeyi açacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denenmiştir. Değer değiştirmek `sudo` gerektirir.

## İçindekiler

1. [Parametreleri görmek](#1-parametreleri-görmek)
2. [Parametre adları ve /proc/sys](#2-parametre-adları-ve-procsys)
3. [Geçici değişiklik: sysctl -w](#3-geçici-değişiklik-sysctl--w)
4. [Kalıcı değişiklik: /etc/sysctl.d](#4-kalıcı-değişiklik-etcsysctld)
5. [Dosyaları uygulamak: sysctl -p ve sysctl --system](#5-dosyaları-uygulamak-sysctl--p-ve-sysctl---system)
6. [Dosyaların sırası ve çakışmalar](#6-dosyaların-sırası-ve-çakışmalar)
7. [Sık kullanılan parametreler](#7-sık-kullanılan-parametreler)
8. [Açılış parametreleri ile farkı](#8-açılış-parametreleri-ile-farkı)
9. [Komut özeti](#9-komut-özeti)
10. [Sık yapılan hatalar](#10-sık-yapılan-hatalar)
11. [Alıştırmalar](#11-alıştırmalar)

---

## 1. Parametreleri görmek

```bash
sysctl -a                       # bütün parametreler
sysctl -a | grep swappiness     # aramak için
sysctl vm.swappiness            # tek bir parametre
sysctl -n vm.swappiness         # sadece değer (script'ler için)
```
```
vm.swappiness = 60
```

Normal kullanıcı `sysctl -a` çalıştırınca bazı parametreler için `permission denied` satırları da görür (denemede 10 tane). Bunlar sadece root'un okuyabildiği parametrelerdir. Bütün listeyi görmek için `sudo sysctl -a`.

---

## 2. Parametre adları ve /proc/sys

Her parametre aslında `/proc/sys/` altında bir dosyadır. Ad, dosyanın yolundan gelir: noktalar dizin ayırıcısıdır.

| Parametre | Dosya |
|---|---|
| `vm.swappiness` | `/proc/sys/vm/swappiness` |
| `net.ipv4.ip_forward` | `/proc/sys/net/ipv4/ip_forward` |
| `fs.file-max` | `/proc/sys/fs/file-max` |

```bash
cat /proc/sys/vm/swappiness
```

Adın ilk kısmı hangi alana ait olduğunu söyler. Sınavda tanımadığın bir parametre görürsen buradan neyle ilgili olduğunu tahmin edebilirsin:

| Önek | Alan |
|---|---|
| `kernel.` | Çekirdeğin genel davranışı |
| `vm.` | Sanal bellek, swap |
| `net.` | Ağ |
| `fs.` | Dosya sistemleri |
| `dev.` | Cihazlar |

Parametrelerin ne işe yaradığı çekirdek belgelerinde yazar. Kısa açıklamalar için `man 5 sysctl.d` ve `man 8 sysctl`.

---

## 3. Geçici değişiklik: sysctl -w

```bash
sudo sysctl -w vm.swappiness=20
```
```
vm.swappiness = 20
```

Değişiklik **anında** geçerli olur, ama sadece yeniden başlatmaya kadar. Açılışta değer varsayılana (veya kalıcı ayara) döner.

Aynı şey `/proc/sys` dosyasına yazarak da yapılır:

```bash
echo 30 | sudo tee /proc/sys/vm/swappiness
```

(`sudo echo 30 > /proc/...` çalışmaz, yönlendirmeyi kabuk yapar: [Yönlendirme](../04-temel-komutlar/10-yonlendirme.md#11-sudo-ile-yönlendirme).)

### Yazım hataları

| Komut | Sonuç |
|---|---|
| `sysctl -w vm.swappiness=10` | Doğru |
| `sysctl -w vm/swappiness=10` | Doğru, `/` da kabul edilir |
| `sysctl -w vm.swappiness = 10` | `invalid syntax`, `=` etrafında boşluk olmamalı |
| `sysctl vm.swappinesss` | `cannot stat /proc/sys/vm/swappinesss: No such file or directory` |

---

## 4. Kalıcı değişiklik: /etc/sysctl.d

Kalıcı ayarlar, açılışta okunan dosyalara yazılır. Kendi ayarların için `/etc/sysctl.d/` altına `.conf` uzantılı yeni bir dosya oluştur:

```bash
echo 'vm.swappiness=20' | sudo tee /etc/sysctl.d/90-swap.conf
```

Dosyanın biçimi basittir: her satırda `parametre=değer`. `#` ile başlayan satırlar açıklamadır. Burada `=` etrafında boşluk olabilir.

```
# Sunucuda bol RAM var, swap'a gec basvur
vm.swappiness = 20
```

> [!TIP]
> `/etc/sysctl.conf` dosyası da çalışır, ama ayarları konularına göre `sysctl.d/` altında ayrı dosyalarda tutmak daha düzenlidir. Hangi ayarı kimin neden eklediği dosya adından anlaşılır.

Dosyayı kaydetmek **ayarı şimdi değiştirmez.** Kaydettikten sonra hala:

```
vm.swappiness = 60
```

Ayar bir sonraki açılışta uygulanır. Şimdi de geçerli olması için bir sonraki bölümdeki komutlardan birini çalıştır.

---

## 5. Dosyaları uygulamak: sysctl -p ve sysctl --system

| Komut | Hangi dosyaları okur? |
|---|---|
| `sysctl -p` | Sadece `/etc/sysctl.conf` |
| `sysctl -p dosya` | Sadece verilen dosya |
| `sysctl --system` | Bütün dizinlerdeki bütün dosyalar, açılıştaki sırayla |

Bir fark denemede ortaya çıktı: `/etc/sysctl.d/90-swap.conf`'u yazdıktan sonra argümansız `sysctl -p` çalıştırdık, değer **değişmedi**. Çünkü `-p` dosya verilmezse `sysctl.d`'ye bakmaz.

```bash
sudo sysctl -p /etc/sysctl.d/90-swap.conf     # sadece bu dosya
```
```
vm.swappiness = 20
```

veya hepsini, açılıştakiyle aynı şekilde:

```bash
sudo sysctl --system
```
```
* Applying /etc/sysctl.d/90-swap.conf ...
* Applying /usr/lib/sysctl.d/99-protect-links.conf ...
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.conf ...
vm.swappiness = 20
```

`sysctl --system`, açılışta ne olacağını en doğru gösteren komuttur. Kalıcı ayarı yazdıktan sonra onu çalıştırıp değeri kontrol etmek, ayarın açılışta da çalışacağının en iyi kanıtıdır.

---

## 6. Dosyaların sırası ve çakışmalar

`sysctl --system` dosyaları şu dizinlerden okur:

| Dizin | Kimin? |
|---|---|
| `/etc/sysctl.d/` | Yöneticinin |
| `/run/sysctl.d/` | Geçici |
| `/usr/lib/sysctl.d/` | Paketlerin |
| `/etc/sysctl.conf` | Eski tek dosya (`99-sysctl.conf` adıyla bağlantılı) |

Bütün dosyalar **adlarına göre** sıralanıp okunur ve aynı parametre birden fazla dosyada varsa **en son okunan kazanır**. Aynı ada sahip dosya birden fazla dizinde varsa sadece `/etc`'deki okunur.

Denedik:

| Dosyalar | Sonuç |
|---|---|
| `90-swap.conf` (20) ve `95-baska.conf` (40) | 40, çünkü 95 sonra okundu |
| `90-swap.conf` (20) ve `80-baska.conf` (40) | 20, çünkü 90 sonra okundu |

Ayarın etkili olmuyorsa, başka bir dosyanın aynı parametreyi sonradan değiştirip değiştirmediğine bak:

```bash
grep -r swappiness /etc/sysctl.conf /etc/sysctl.d/ /usr/lib/sysctl.d/ /run/sysctl.d/ 2>/dev/null
```

Ubuntu'da `/etc/sysctl.d/99-sysctl.conf`, `/etc/sysctl.conf`'a bir bağlantıdır. Yani `/etc/sysctl.conf`'taki ayarlar her zaman en sonlarda okunur ve `90-` gibi dosyaları ezer.

---

## 7. Sık kullanılan parametreler

| Parametre | Ne yapar? | Örnek |
|---|---|---|
| `vm.swappiness` | Swap'a ne kadar istekli gidileceği (0–100+, varsayılan 60) | Sunucuda 10–20 |
| `net.ipv4.ip_forward` | Sunucunun paketleri bir arayüzden diğerine yönlendirmesi | NAT ve yönlendirici için `1` ([NAT](../02-ag-yonetimi/07-nat-ve-port-yonlendirme.md)) |
| `net.ipv6.conf.all.disable_ipv6` | IPv6'yı kapatır | `1` |
| `net.ipv4.icmp_echo_ignore_all` | Ping'e cevap vermez | `1` |
| `fs.file-max` | Sistem genelinde açık dosya sınırı | |
| `kernel.hostname` | Makinenin adı (geçici) | |
| `vm.dirty_ratio` | Belleğin yüzde kaçı diske yazılmayı bekleyebilir | |
| `net.core.somaxconn` | Bekleyen bağlantı kuyruğu | Yoğun web sunucularında artırılır |

> [!NOTE]
> Deneme ortamında IPv6 olmadığı için `net.ipv6.*` parametreleri yoktu, onlarla ilgili örnekler denenmedi.

---

## 8. Açılış parametreleri ile farkı

`sysctl` ayarları sistem **açıldıktan sonra** uygulanır. Bazı ayarlar ise çekirdek açılırken verilmelidir (`init=`, `systemd.unit=`, bazı donanım ayarları). Bunlar GRUB'dan, açılış parametresi olarak verilir: [Arızadan Kurtarma](02-arizadan-kurtarma.md#2-grub-menüsüne-girmek-ve-açılış-parametresi-vermek).

| | sysctl | Açılış parametresi |
|---|---|---|
| Ne zaman? | Sistem çalışırken | Çekirdek açılırken |
| Geçici | `sysctl -w` | GRUB'da `e` ile |
| Kalıcı | `/etc/sysctl.d/*.conf` | `/etc/default/grub` + `update-grub` |
| Mevcut değer | `sysctl ad` | `cat /proc/cmdline` |

Çekirdek modüllerinin parametreleri ise ayrı bir konudur (`/etc/modprobe.d/`, `modprobe`), örneği: [NBD](../03-depolama/08-nbd.md).

---

## 9. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `sysctl -a` | Bütün parametreler |
| `sysctl ad` / `sysctl -n ad` | Tek parametre / sadece değeri |
| `sudo sysctl -w ad=değer` | Geçici değiştir |
| `echo değer \| sudo tee /proc/sys/...` | Geçici değiştir, dosya yoluyla |
| `/etc/sysctl.d/90-ad.conf` | Kalıcı ayar dosyası |
| `sudo sysctl -p dosya` | Dosyayı şimdi uygula |
| `sudo sysctl -p` | Sadece `/etc/sysctl.conf`'u uygula |
| `sudo sysctl --system` | Bütün dosyaları açılıştaki sırayla uygula |
| `man 5 sysctl.d` | Dosya biçimi ve sıralama kuralları |

---

## 10. Sık yapılan hatalar

- **Sadece `sysctl -w` yapıp kalıcı olduğunu sanmak.** Yeniden başlatınca gider.
- **Sadece dosyaya yazıp şimdi de geçerli olduğunu sanmak.** `sysctl -p dosya` veya `--system` gerekir.
- **`sysctl.d` altına yazıp argümansız `sysctl -p` çalıştırmak.** O sadece `/etc/sysctl.conf`'u okur.
- **Dosya uzantısını `.conf` yapmamak.** Okunmaz.
- **`sysctl -w`'de `=` etrafına boşluk koymak.**
- **Aynı parametreyi daha sonra okunan başka bir dosyanın ezdiğini fark etmemek.**
- **`sudo echo değer > /proc/sys/...` yazmak.** `Permission denied`, `tee` kullan.

---

## 11. Alıştırmalar

**1.** `vm.swappiness`'in şu anki değerini iki farklı yolla göster.

<details>
<summary>Çözüm</summary>

```bash
sysctl vm.swappiness
cat /proc/sys/vm/swappiness
```
</details>

**2.** IP yönlendirmeyi şimdi aç, ama yeniden başlatınca kapanmasına izin ver. Değeri kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward
```
</details>

**3.** `vm.swappiness`'i kalıcı olarak 10 yap ve yeniden başlatmadan da geçerli olmasını sağla.

<details>
<summary>Çözüm</summary>

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/90-swappiness.conf
sudo sysctl -p /etc/sysctl.d/90-swappiness.conf
sysctl vm.swappiness
```

`sudo sysctl --system` da olur ve açılıştaki sırayı da kontrol etmiş olursun.
</details>

**4.** `/etc/sysctl.d/90-swappiness.conf`'ta değer 10, ama `sudo sysctl --system`'dan sonra değer 10 olmuyor. Sebebini nasıl bulursun?

<details>
<summary>Çözüm</summary>

```bash
sudo sysctl --system | grep -E 'Applying|swappiness'
grep -r swappiness /etc/sysctl.conf /etc/sysctl.d/ /usr/lib/sysctl.d/ /run/sysctl.d/
```

Büyük ihtimalle `90-swappiness.conf`'tan sonra okunan bir dosya (`95-...conf` veya `/etc/sysctl.conf`) aynı parametreyi başka bir değere ayarlıyor. Son okunan kazanır.
</details>

**5.** `kernel.` ile başlayan ve adında `panic` geçen bütün parametreleri listele.

<details>
<summary>Çözüm</summary>

```bash
sudo sysctl -a | grep '^kernel\..*panic'
```
</details>

---

Önceki konu: [Kaynaktan Yazılım Kurmak](08-kaynaktan-kurma.md) · Sonraki konu: [SELinux](10-selinux.md)
