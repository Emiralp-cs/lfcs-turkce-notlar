# Sınav İpuçları

LFCS'de çoktan seçmeli soru yoktur. Sana bir terminal ve bir görev listesi verilir. Her görevde bir sunucuya bağlanıp bir şeyi kurar, yapılandırır veya düzeltirsin. Sonuçta önemli olan, sınav bittiğinde sistemlerin hangi durumda olduğudur. Bu yüzden komutu bilmek yetmez: görevin **tam olarak ne istediğini** anlamak ve sonucu **doğrulamak** gerekir.

> [!NOTE]
> Aşağıdaki resmi bilgiler Kasım 2026'da Linux Foundation'ın [LFCS sayfasından](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/) ve [sınav talimatlarından](https://docs.linuxfoundation.org/tc-docs/certification/instructions-lfcs-and-lfce) alındı. Kurallar değişebilir, sınavdan önce bu iki sayfayı tekrar oku.

## İçindekiler

1. [Sınav hakkında resmi bilgiler](#1-sınav-hakkında-resmi-bilgiler)
2. [Sınav ortamı](#2-sınav-ortamı)
3. [Yapmaman gerekenler](#3-yapmaman-gerekenler)
4. [Görev metnini okumak](#4-görev-metnini-okumak)
5. [Görev türleri](#5-görev-türleri)
6. [Her görevden sonra doğrula](#6-her-görevden-sonra-doğrula)
7. [Sınavda belge bulmak](#7-sınavda-belge-bulmak)
8. [Zaman yönetimi](#8-zaman-yönetimi)
9. [Sınavdan önce](#9-sınavdan-önce)

---

## 1. Sınav hakkında resmi bilgiler

| | |
|---|---|
| Süre | 2 saat |
| Görev sayısı | 17-20 uygulamalı görev |
| Geçme notu | %67 |
| Biçim | Çevrim içi, gözetmenli |
| Tekrar hakkı | 1 |
| Geçerlilik | 2 yıl |
| Deneme sınavı | Killer.sh simülatörü, kayıt başına 2 deneme |

Resmi sayfa sınavın dağıtımdan bağımsız olduğunu söylüyor. Bu depodaki bütün notlar Ubuntu 24.04 üzerinde denendi.

---

## 2. Sınav ortamı

- Sınav `base` adlı bir makinede başlar. **Görevleri `base` üzerinde yapmazsın.**
- Her görevin başındaki kutuda hangi sunucuya `ssh` ile bağlanacağın yazar. Görevi o sunucuda yaparsın.
- Görev bitince `exit` ile `base`'e dön, sonraki görevin sunucusuna oradan bağlan. **İç içe ssh desteklenmez**, bir sunucudan diğerine bağlanma.
- Root yetkisi için `sudo -i` veya tek komutlar için `sudo`.
- `base`'de vim, nano, git ve sudo yoktur. Görev sunucularında vardır. `base`'de dosya düzenlemeye çalışma.
- Tarayıcıda `Ctrl+W` sekmeyi kapatır. Terminalde kelime silmek için `Ctrl+Alt+W` kullan.
- `Insert` tuşu yasak. vim'de ekleme moduna `i` ile geç, `Esc` ile çık.

> [!TIP]
> Bağlandıktan sonra ilk iş `hostname` ile doğru sunucuda olduğunu kontrol et. Doğru yapılmış ama yanlış sunucudaki bir görev puan getirmez.

---

## 3. Yapmaman gerekenler

Resmi talimatlarda açıkça yazanlar:

- **`base` makinesini yeniden başlatma.** Sınav ortamın geri gelmez.
- **8080/tcp, 4505/tcp ve 4506/tcp portlarına gelen bağlantıyı engelleme.** Güvenlik duvarı görevlerinde (`ufw default deny incoming` gibi) bunu unutmak kolaydır.
- **`certerminal` sürecine dokunma.** Sınavın biter.

Bunlara ek olarak kendi önerim: Bir görev sunucusunu yeniden başlatman gerekiyorsa (ör. kernel parametresi veya fstab sonrası doğrulama için), önce yaptığın değişikliğin açılışı bozmayacağından emin ol. fstab'ı `findmnt --verify` ile denetle ([Mount ve fstab](04-depolama/03-mount-ve-fstab.md)). Açılmayan bir sunucuda kalan görevleri de kaybedersin.

---

## 4. Görev metnini okumak

Görevler kısa yazılır ve her kelime bir şey ister. Uydurma bir örnek:

> On host **srv2**, create a 500M logical volume named **arsiv** in volume group **data**, format it with **XFS** and make sure it is **mounted at /arsiv after every boot**.

İçinden çıkan kontrol listesi:

| İfade | Yapılacak | Unutulursa |
|---|---|---|
| on host srv2 | `ssh srv2` | Görev yanlış makinede |
| 500M | `-L 500M`, `-L +500M` değil | Boyut yanlış |
| named arsiv, in data | `-n arsiv data` | Ad yanlış |
| XFS | `mkfs.xfs` | Tür yanlış |
| mounted at /arsiv | `mkdir /arsiv` ve mount | |
| after every boot | fstab satırı | Yeniden başlatınca kaybolur |

Sık geçen ifadeler ve anlamları:

| Görevde | Anlamı |
|---|---|
| persistent, after reboot, permanently | **Kalıcı:** fstab, `/etc/sysctl.d/`, `systemctl enable`, `netplan` dosyası |
| immediately, currently, now | **Şimdi de** etkin olsun: `--now`, `mount -a`, `sysctl --system` |
| persistent **and** immediately | İkisi birden. En sık kaçırılan durum |
| only, exclusively | Başka hiçbir şey olmasın: başka kullanıcı, başka ağ, başka port yok |
| save the output to /path/file | Dosya tam olarak o yolda ve beklenen içerikle. Fazladan satır olmasın |
| owned by, group | `chown`, `chgrp`. Sahipliği de kontrol et |
| should be able to | O kullanıcının gerçekten yapabildiğini dene (`sudo -u kullanici ...`) |

> [!IMPORTANT]
> Adları ve yolları görev metninden **kopyala**, elle yazma. `/opt/backups` ile `/opt/backup` farklı yollardır. Kontrol edilen şey görevde yazan tam yoldur.

---

## 5. Görev türleri

Görevler genelde dört kalıptan birine uyar:

**Kur ve yapılandır.** Boş bir durumdan bir şey oluştur: kullanıcı, servis, LV, NFS paylaşımı, cron görevi. Doğru komutu bilmek çoğu zaman yeter. Kalıcılık kısmını atlama.

**Bozuk olanı düzelt.** Bir servis başlamıyor, bir disk bağlanmıyor, bir kullanıcı giriş yapamıyor. Sebebi bulmak asıl iştir. Önce günlüğe bak: `systemctl status`, `journalctl -u servis -n 30`, `journalctl -p err -b`. Notlardaki "Sık yapılan hatalar" bölümleri bu tür görevlere hazırlık içindir. Örnekler: [Servis Oluşturma ve Sorun Giderme](01-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md), [Arızadan Kurtarma](03-isletim-ve-dagitim/02-arizadan-kurtarma.md).

**Bul ve kaydet.** Bir bilgiyi bulup bir dosyaya yaz: en çok bellek kullanan sürecin PID'i, belirli izinlere sahip dosyalar, bir paketin hangi dosyayı içerdiği. Çıktının biçimine dikkat: görev sadece PID istiyorsa dosyada başlık satırı olmasın (`ps -o pid= ...`, `find ... > dosya`).

**Değiştir.** Var olan bir şeyi değiştir: LV'yi büyüt, bir servisin limitini artır, bir kullanıcının grubunu değiştir. Var olanı bozmadan yap: LV büyütürken dosya sistemini de büyüt, kullanıcıyı gruba eklerken `-aG` kullan.

---

## 6. Her görevden sonra doğrula

Bir sonraki göreve geçmeden önce otuz saniyelik bir kontrol, en çok puan kurtaran alışkanlıktır:

| Konu | Doğrulama |
|---|---|
| Servis | `systemctl is-active servis` ve `systemctl is-enabled servis` |
| fstab | `findmnt --verify`, sonra `umount` ve `mount -a`, `findmnt /bağlama` |
| LV ve dosya sistemi | `lvs`, `df -h /bağlama` |
| Kernel parametresi | `sysctl --system` sonrası `sysctl parametre` |
| Kullanıcı ve grup | `id kullanici`, `getent group grup` |
| Dosya izinleri | `ls -l`, `getfacl` |
| Ağ | `ip a`, `ip r`, `ss -tlnp`, `resolvectl status` |
| Cron | `crontab -l -u kullanici`, `/etc/cron.d/` dosyasının izinleri |
| Güvenlik duvarı | `ufw status verbose`, 8080/4505/4506'nın açık olduğu |
| Kaydedilen çıktı | `cat dosya`: içerik ve biçim beklendiği gibi mi? |

"Açılışta da çalışsın" diyen bir görevde, sunucuyu yeniden başlatmadan yapabileceğin en iyi kontrol, değişikliği yapılandırma dosyasından tekrar yüklemektir: `mount -a`, `sysctl --system`, `netplan apply`, `systemctl daemon-reload`.

---

## 7. Sınavda belge bulmak

Sınavda internet yoktur, ama `man` sayfaları ve `/usr/share/doc` altındaki belgeler kullanılabilir. Bunları sınavdan önce kullanmaya alış:

```bash
man -k logical volume          # anahtar kelimeyle man sayfası ara (= apropos)
man 5 fstab                    # dosya biçimi (bölüm 5)
man exports                    # NFS seçenekleri ve örnekler
man systemd.exec               # servis birimlerindeki Limit..., Protect... seçenekleri
man systemd.time               # OnCalendar biçimi
man 5 crontab                  # cron satırı biçimi
ls /usr/share/doc/paket/       # paket belgeleri, bazen examples/ dizini
```

`man` içinde `/kelime` ile ara, `n` ile sonrakine geç. Çoğu man sayfasının sonunda **EXAMPLES** bölümü vardır: `/EXAMPLES` ile doğrudan oraya atla.

Komutun kendi yardımı da çoğu zaman yeter: `lvresize --help | grep -i fs`, `ip route help`.

---

## 8. Zaman yönetimi

120 dakika ve 17-20 görev: görev başına ortalama 6-7 dakika.

- **İlk turda bildiklerini yap.** Takıldığın bir görevde 10 dakikadan fazla kalma, not al ve geç. Sona kalan zamanda geri dön.
- Görevlerin ağırlıkları resmi sayfada açıklanmıyor. Bir görevin ne kadar puan getirdiğini bilmediğin için, kolay görevleri bitirmek en güvenli yoldur.
- Uzun süren işlemleri (paket kurulumu, büyük dosya kopyalama) başlatıp beklerken görevin geri kalanını oku.
- Son 10 dakikayı, atladığın görevlere ve doğrulamaya ayır.

---

## 9. Sınavdan önce

- **Killer.sh simülatörünü kullan.** Kayıtla birlikte iki deneme hakkı gelir. Gerçek sınavdan daha zor olduğu söylenir. Amaç puan değil, ortama ve zaman baskısına alışmak. İlkini erken çöz, eksiklerini çalış, ikincisini sınavdan birkaç gün önce çöz.
- **Her şeyi terminalde, kendi elinle yap.** Bu depodaki alıştırmaların hepsi bir sanal makinede yapılabilir. Çözüme bakmadan dene.
- **vim'de rahat ol.** Sınavın büyük kısmı dosya düzenlemektir: `i`, `Esc`, `:wq`, `:q!`, `/arama`, `dd`, `u`.
- **Kalıcılık refleksi kazan.** Her değişiklikten sonra kendine "yeniden başlatınca kalır mı?" diye sor.
- Resmi sayfadaki sistem gereksinimlerini (tarayıcı, kamera, internet) sınav gününden önce kontrol et.

---

[Ana sayfaya dön](README.md)
