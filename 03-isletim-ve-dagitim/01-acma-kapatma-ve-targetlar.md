# Açma, Kapatma ve Target'lar

> **İlgili resmi sınav maddeleri:** Diagnose, identify, manage, and troubleshoot processes and services · Recover from hardware, operating system, or filesystem failures

Bir sunucuyu yeniden başlatmak veya kapatmak basit görünür, ama üzerinde kullanıcılar çalışırken bunu habersiz yapmak veri kaybına yol açar. Bu dosya sistemi güvenle kapatmayı, yeniden başlatmayı zamanlamayı ve kullanıcıları uyarmayı, sonra da sistemin hangi **modda** (target) açılacağını değiştirmeyi anlatıyor.

Senaryo: Sunucuya çekirdek güncellemesi geldi, yeniden başlatılması gerekiyor. Gece 02:00'de, kullanıcılara önceden haber vererek yapacağız. Ayrıca sunucuda gereksiz yere grafik arayüz yükleniyor, açılış modunu değiştireceğiz.

> [!NOTE]
> Komutlar Ubuntu 24.04 ve systemd 255 ile denenmiştir. Kapatma ve yeniden başlatma zamanlanıp hemen iptal edilerek denendi, sistem gerçekten kapatılmadı. Rescue ve emergency modlarına girmek denenmedi.

## İçindekiler

1. [Kapatmak ve yeniden başlatmak](#1-kapatmak-ve-yeniden-başlatmak)
2. [Yanıt vermeyen sistem: --force](#2-yanıt-vermeyen-sistem---force)
3. [Zamanlamak ve kullanıcıları uyarmak: shutdown](#3-zamanlamak-ve-kullanıcıları-uyarmak-shutdown)
4. [Target nedir?](#4-target-nedir)
5. [Varsayılan target: get-default ve set-default](#5-varsayılan-target-get-default-ve-set-default)
6. [Anlık geçiş: isolate](#6-anlık-geçiş-isolate)
7. [Rescue ve emergency](#7-rescue-ve-emergency)
8. [Komut özeti](#8-komut-özeti)
9. [Sık yapılan hatalar](#9-sık-yapılan-hatalar)
10. [Alıştırmalar](#10-alıştırmalar)

---

## 1. Kapatmak ve yeniden başlatmak

```bash
sudo systemctl reboot       # yeniden başlat
sudo systemctl poweroff     # kapat
```

Bu komutlar servisleri sırayla, düzgünce durdurur, diskleri ayırır, sonra sistemi kapatır. Root olarak giriş yaptıysan `sudo` gerekmez.

`reboot`, `poweroff`, `shutdown` ve `halt` komutları da var. Ubuntu'da bunların hepsi aslında `systemctl`'e bağlantıdır:

```bash
ls -l /sbin/reboot /sbin/poweroff
```
```
/sbin/reboot -> ../bin/systemctl
/sbin/poweroff -> ../bin/systemctl
```

Yani `sudo reboot` ile `sudo systemctl reboot` aynı işi yapar.

> [!CAUTION]
> Uzak bir sunucuda `poweroff` dersen sunucuyu tekrar açmak için fiziksel erişim veya bir yönetim paneli gerekir. Yeniden başlatmak istiyorsan `reboot` yaz.

---

## 2. Yanıt vermeyen sistem: --force

Bazen bir servis durmayı reddeder ve kapatma takılır. O zaman `--force`:

| Komut | Ne yapar? |
|---|---|
| `systemctl reboot --force` | Servisleri tek tek kapatmayı beklemez, bütün süreçleri öldürür, diskleri ayırıp yeniden başlatır |
| `systemctl reboot --force --force` | Hiçbir şeyi beklemez, diskleri bile ayırmaz. Fişi çekip takmak gibi |

`poweroff` için de aynısı geçerli.

> [!WARNING]
> `--force` verilerin diske yazılmasını beklemeyebilir, çift `--force` dosya sistemini bozabilir. Sadece normal kapatma gerçekten takıldığında, son çare olarak kullan.

---

## 3. Zamanlamak ve kullanıcıları uyarmak: shutdown

`shutdown`, kapatmayı veya yeniden başlatmayı ileri bir zamana kurar ve bağlı kullanıcılara mesaj gönderir:

```bash
sudo shutdown -r 02:00 "Cekirdek guncellemesi icin yeniden baslatma"
```
```
Reboot scheduled for Fri 2026-11-27 02:00:00 UTC, use 'shutdown -c' to cancel.
```

| Komut | Ne yapar? |
|---|---|
| `shutdown 02:00` | Saat 02:00'de kapat (24 saat biçimi) |
| `shutdown +15` | 15 dakika sonra kapat |
| `shutdown now` | Hemen kapat |
| `shutdown -r 02:00` | Kapatmak yerine yeniden başlat |
| `shutdown -r +1 "mesaj"` | 1 dakika sonra, kullanıcılara mesaj göndererek |
| `shutdown -c` | Zamanlanmış kapatmayı iptal et |

Dikkat edilecek iki şey denemede ortaya çıktı:

- Saat **geçmişse ertesi güne** kurulur. Sabah 08:54'te `shutdown 02:00` yazdığımızda ertesi günün 02:00'sine kuruldu.
- Zaman **hiç verilmezse** `shutdown` hemen değil, **1 dakika sonra** kapatır: `Shutdown scheduled for ... 08:54:43` (komut 08:53:43'te yazıldı).

Mesaj, o an terminalde oturumu açık olan herkese gönderilir (`wall` mesajı). Kapatmadan birkaç dakika önce de giriş yapılmasını engeller.

Zamanlanmış bir işlem olup olmadığını görmek için:

```bash
cat /run/systemd/shutdown/scheduled
```
```
USEC=1795685023722019
MODE=reboot
WALL_MESSAGE=Cekirdek\x20guncellemesi\x20icin\x20yeniden\x20baslatma
```

`shutdown -c`'den sonra bu dosya silindi.

| | `systemctl reboot/poweroff` | `shutdown` |
|---|---|---|
| Ne zaman? | Hemen | Zamanlanabilir |
| Kullanıcıları uyarır mı? | Hayır | Evet |
| İptal edilebilir mi? | Hayır | `shutdown -c` |

---

## 4. Target nedir?

systemd'de **target**, birlikte başlatılan unit'lerden oluşan bir gruptur. Sistemin açılışta hangi target'a ulaşacağı, hangi servislerin çalışacağını belirler. Eski sistemlerdeki "runlevel" kavramının karşılığıdır.

| Target | Ne çalışır? | Eski runlevel |
|---|---|---|
| `poweroff.target` | Sistem kapanır | 0 |
| `rescue.target` | Sadece temel sistem, tek kullanıcı, root kabuğu. Ağ yok | 1 |
| `multi-user.target` | Bütün servisler ve ağ, metin konsol. **Sunucular için normal mod** | 3 |
| `graphical.target` | `multi-user` + grafik arayüz | 5 |
| `reboot.target` | Sistem yeniden başlar | 6 |
| `emergency.target` | En az mod: kök dosya sistemi salt okunur, neredeyse hiçbir servis yok | - |

Eski runlevel adları hala bağlantı olarak duruyor:

```
/usr/lib/systemd/system/runlevel3.target -> multi-user.target
/usr/lib/systemd/system/runlevel5.target -> graphical.target
```

Şu an etkin target'lar:

```bash
systemctl list-units --type=target --state=active
```

`graphical.target` etkinse `multi-user.target` da etkindir, çünkü biri diğerinin üzerine kuruludur.

---

## 5. Varsayılan target: get-default ve set-default

```bash
systemctl get-default
```
```
graphical.target
```

Grafik arayüze ihtiyacı olmayan bir sunucuda, açılışta daha az kaynak kullanması için:

```bash
sudo systemctl set-default multi-user.target
```
```
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/multi-user.target.
```

Daha önce bir varsayılan ayarlanmışsa önce onu kaldırır:

```
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
```

Varsayılan target aslında `/etc/systemd/system/default.target` adında bir sembolik bağlantıdır.

> [!IMPORTANT]
> `set-default` **bir sonraki açılışı** etkiler, şu an çalışan sisteme dokunmaz. Değişiklikten sonra sistem hala eski modda çalışır. Hemen geçmek için `isolate` ([6. bölüm](#6-anlık-geçiş-isolate)) veya yeniden başlatma gerekir.

---

## 6. Anlık geçiş: isolate

`isolate`, varsayılanı değiştirmeden **şu an** başka bir target'a geçer. Yeni target'ta olmayan bütün servisleri durdurur:

```bash
sudo systemctl isolate multi-user.target    # grafik arayüzü kapat
sudo systemctl isolate graphical.target     # grafik arayüzü aç
```

Her target'a `isolate` ile geçilemez, sadece `AllowIsolate=yes` olanlara:

```bash
systemctl show multi-user.target -p AllowIsolate    # AllowIsolate=yes
systemctl show network.target -p AllowIsolate       # AllowIsolate=no
sudo systemctl isolate network.target
```
```
Failed to start network.target: Operation refused, unit network.target may be requested by dependency only (it is configured to refuse manual start/stop).
```

| | `set-default` | `isolate` |
|---|---|---|
| Ne zaman etkili? | Bir sonraki açılışta | Hemen |
| Kalıcı mı? | Evet | Hayır, yeniden başlatınca varsayılana döner |

---

## 7. Rescue ve emergency

Sistem düzgün açılmıyorsa, örneğin `/etc/fstab`'daki bir hata yüzünden, bu iki mod sorunu düzeltmek için en az servisle bir root kabuğu verir:

| | rescue | emergency |
|---|---|---|
| Kök dosya sistemi | Okuma-yazma | **Salt okunur** |
| Diğer diskler | Bağlanır | Bağlanmaz |
| Temel servisler | Çalışır | Çalışmaz |
| Ne zaman? | Bir servis veya yapılandırma sorunu | Disk veya dosya sistemi sorunu, rescue bile açılmıyor |

Çalışan sistemden geçmek için:

```bash
sudo systemctl rescue        # = systemctl isolate rescue.target
sudo systemctl emergency
```

Açılmayan bir sistemde ise bu modlara açılış menüsünden (GRUB) girilir: [Arızadan Kurtarma](02-arizadan-kurtarma.md).

> [!WARNING]
> İki mod da girişte **root parolası** ister. Ubuntu'da root hesabı varsayılan olarak kilitlidir (`passwd -S root` çıktısında `L`). Bu durumda rescue ve emergency modlarına giremezsin, root parolası olmadan kabuk açılmaz. Bunun çözümü de [Arızadan Kurtarma](02-arizadan-kurtarma.md) dosyasında.

Uzak bir sunucuda `rescue` veya `emergency`'ye geçmek ağı ve SSH'ı kapatır, bağlantın kopar ve konsol erişimin yoksa geri dönemezsin.

---

## 8. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `systemctl reboot` | Yeniden başlat |
| `systemctl poweroff` | Kapat |
| `systemctl reboot --force` | Servisleri beklemeden yeniden başlat |
| `systemctl reboot --force --force` | Hiçbir şeyi beklemeden, son çare |
| `shutdown -r 02:00 "mesaj"` | 02:00'de yeniden başlat, kullanıcıları uyar |
| `shutdown +15` | 15 dakika sonra kapat |
| `shutdown -c` | Zamanlanmış kapatmayı iptal et |
| `systemctl get-default` | Varsayılan target |
| `systemctl set-default multi-user.target` | Varsayılanı değiştir (sonraki açılış) |
| `systemctl isolate multi-user.target` | Şimdi bu target'a geç |
| `systemctl list-units --type=target` | Etkin target'lar |
| `systemctl rescue` / `emergency` | Kurtarma modlarına geç |

---

## 9. Sık yapılan hatalar

- **Uzak sunucuda `reboot` yerine `poweroff` yazmak.**
- **Üzerinde kullanıcılar varken habersiz yeniden başlatmak.** `shutdown -r +5 "mesaj"` kullan.
- **`set-default`'tan sonra sistemin hemen değişmesini beklemek.** Sonraki açılışta etkili olur.
- **`isolate` ile yapılan geçişin kalıcı olduğunu sanmak.**
- **`--force`'u normal kapatma yöntemi gibi kullanmak.**
- **Root parolası yokken rescue/emergency'ye güvenmek.**
- **`shutdown 02:00`'nin bugün çalışacağını sanmak.** Saat geçtiyse ertesi gün çalışır.

---

## 10. Alıştırmalar

**1.** Sistemi 45 dakika sonra yeniden başlatacak şekilde ayarla, kullanıcılara "Bakim icin yeniden baslatma" mesajı gönderilsin. Zamanlandığını kontrol et, sonra iptal et.

<details>
<summary>Çözüm</summary>

```bash
sudo shutdown -r +45 "Bakim icin yeniden baslatma"
cat /run/systemd/shutdown/scheduled
sudo shutdown -c
```
</details>

**2.** Sistemin varsayılan olarak hangi target'la açıldığını bul. Bir sonraki açılışta grafik arayüz olmadan açılmasını sağla.

<details>
<summary>Çözüm</summary>

```bash
systemctl get-default
sudo systemctl set-default multi-user.target
systemctl get-default
ls -l /etc/systemd/system/default.target
```
</details>

**3.** 2. alıştırmadan sonra sistem şu an hangi target'larda? Varsayılanı değiştirmeden, şu an grafik arayüzü kapat.

<details>
<summary>Çözüm</summary>

```bash
systemctl list-units --type=target --state=active
sudo systemctl isolate multi-user.target
```

`set-default` şu anki sistemi değiştirmediği için `graphical.target` ilk komutta hala etkin görünür.
</details>

**4.** Hangi target'lara `isolate` ile geçilebildiğini nasıl anlarsın? `rescue.target` ve `sockets.target` için kontrol et.

<details>
<summary>Çözüm</summary>

```bash
systemctl show rescue.target -p AllowIsolate     # yes
systemctl show sockets.target -p AllowIsolate    # no
```
</details>

**5.** Eski "runlevel 3" hangi target'a karşılık geliyor? Dosya sisteminden kontrol et.

<details>
<summary>Çözüm</summary>

```bash
ls -l /usr/lib/systemd/system/runlevel3.target
```

`multi-user.target`'a bağlıdır.
</details>

---

Sonraki konu: [Arızadan Kurtarma](02-arizadan-kurtarma.md)
