# Servis Oluşturma ve Sorun Giderme

> **Resmi sınav maddesi:** Create, configure, and troubleshoot services

Ubuntu'da arka planda sürekli çalışan programları (web sunucusu, veritabanı, SSH) **systemd** yönetir. systemd her servisi bir **unit dosyası**ndan okur: programın nasıl başlatılacağı, çökerse ne yapılacağı, açılışta otomatik başlayıp başlamayacağı bu dosyada yazar. Sınavda hem var olan servisleri yönetmen, hem kendi servisini yazman, hem de açılmayan bir servisin neden açılmadığını bulman beklenir.

Senaryo: Ekip, sunucunun yükünü her 10 saniyede bir kaydeden küçük bir script yazdı. Bunu terminalde açık bırakmak yerine bir servis yapacağız: açılışta kendiliğinden başlasın, çökerse yeniden başlasın, kayıtları journal'a gitsin. Sonra bilerek bozuk bir servis yazıp hatalarını bulacağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve systemd 255 ile denenmiştir. Servis dosyası yazmak root yetkisi gerektirir, komutların başına `sudo` ekle.

## İçindekiler

1. [Unit dosyaları nerede durur?](#1-unit-dosyaları-nerede-durur)
2. [Servisleri yönetmek: systemctl](#2-servisleri-yönetmek-systemctl)
3. [Açılışta başlatmak: enable ve disable](#3-açılışta-başlatmak-enable-ve-disable)
4. [Tamamen engellemek: mask](#4-tamamen-engellemek-mask)
5. [Kendi servisini yazmak](#5-kendi-servisini-yazmak)
6. [Servis dosyasının bölümleri](#6-servis-dosyasının-bölümleri)
7. [Değişiklikten sonra: daemon-reload](#7-değişiklikten-sonra-daemon-reload)
8. [Var olan servisi değiştirmek: drop-in](#8-var-olan-servisi-değiştirmek-drop-in)
9. [Çökünce yeniden başlatma: Restart](#9-çökünce-yeniden-başlatma-restart)
10. [Sorun giderme adımları](#10-sorun-giderme-adımları)
11. [Sık görülen hatalar ve anlamları](#11-sık-görülen-hatalar-ve-anlamları)
12. [Komut özeti](#12-komut-özeti)
13. [Sık yapılan hatalar](#13-sık-yapılan-hatalar)
14. [Alıştırmalar](#14-alıştırmalar)

---

## 1. Unit dosyaları nerede durur?

systemd'nin yönettiği her şey bir **unit**'tir. Türü, dosya uzantısından anlaşılır:

| Uzantı | Ne yönetir? |
|---|---|
| `.service` | Bir program veya servis |
| `.timer` | Zamanlanmış görev ([Zamanlanmış Görevler](../03-isletim-ve-dagitim/05-zamanlanmis-gorevler.md)) |
| `.target` | Birlikte başlatılan unit grupları, çalışma modları ([Açma, Kapatma ve Target'lar](../03-isletim-ve-dagitim/01-acma-kapatma-ve-targetlar.md)) |
| `.socket` | Gelen bağlantıda servisi başlatan soket |
| `.mount` | Dosya sistemi bağlama |

Unit dosyaları iki yerde durur:

| Dizin | Kimin? | Elle düzenlenir mi? |
|---|---|---|
| `/usr/lib/systemd/system/` | Paketlerle gelen dosyalar | Hayır, paket güncellenince üzerine yazılır |
| `/etc/systemd/system/` | Yöneticinin dosyaları ve değişiklikleri | Evet |

Aynı adda dosya ikisinde de varsa `/etc` altındaki geçerlidir. Kendi servislerini ve değişikliklerini her zaman `/etc/systemd/system/` altına koy.

Bir servisin hangi dosyadan okunduğunu ve içeriğini görmek için:

```bash
systemctl cat ssh.service
```

İlk satır dosyanın yolunu gösterir: `# /usr/lib/systemd/system/ssh.service`. Komutlarda `.service` uzantısı yazılmasa da olur, `systemctl status ssh` yeterli.

---

## 2. Servisleri yönetmek: systemctl

```bash
systemctl status nabiz
```
```
● nabiz.service - Sunucu nabiz kaydi
     Loaded: loaded (/etc/systemd/system/nabiz.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-26 08:38:42 UTC; 2s ago
   Main PID: 458 (nabiz.sh)
      Tasks: 2 (limit: 9605)
     Memory: 964.0K ()
     CGroup: /system.slice/nabiz.service
             ├─458 /bin/bash /usr/local/bin/nabiz.sh
             └─460 sleep 10

Nov 26 08:38:42 vm nabiz.sh[458]: nabiz: yuk 0.13
```

| Satır | Anlamı |
|---|---|
| `Loaded` | Dosya nerede, açılışta başlayacak mı (`enabled`/`disabled`) |
| `Active` | Şu an çalışıyor mu (`active (running)`, `inactive (dead)`, `failed`) |
| `Main PID` | Ana sürecin numarası |
| `CGroup` | Servise ait bütün süreçler |
| Son satırlar | Servisin son log kayıtları |

Baştaki işaret de durumu özetler: `●` yeşil çalışıyor, `○` durmuş, `×` hata vermiş.

### Başlatmak, durdurmak, yeniden başlatmak

| Komut | Ne yapar? |
|---|---|
| `systemctl start nabiz` | Başlat |
| `systemctl stop nabiz` | Durdur |
| `systemctl restart nabiz` | Durdur ve tekrar başlat |
| `systemctl reload nabiz` | Durdurmadan ayarları yeniden okut |
| `systemctl reload-or-restart nabiz` | Destekliyorsa reload, desteklemiyorsa restart |

`reload` sadece unit dosyasında `ExecReload=` tanımlı servislerde çalışır. Bizim servisimizde denedik:

```
Failed to reload nabiz.service: Job type reload is not applicable for unit nabiz.service.
```

`reload-or-restart` aynı servisi hatasız yeniden başlattı. Nginx veya SSH gibi bağlantıları koparmadan ayar okuyabilen servislerde `reload` tercih edilir.

### Script'lerde durum sorgulamak

```bash
systemctl is-active nabiz      # active / inactive / failed
systemctl is-enabled nabiz     # enabled / disabled / static / masked
systemctl is-failed nabiz
```

Bunlar tek kelime yazar ve çıkış koduyla da bildirir: `if systemctl is-active -q nabiz; then ...`.

### Listelemek

```bash
systemctl list-units --type=service                  # yüklü ve çalışan servisler
systemctl list-units --type=service --all            # durmuş olanlar dahil
systemctl list-units --type=service --state=failed   # sadece hata verenler
systemctl --failed                                   # aynısı, kısa yol
systemctl list-unit-files --type=service             # bütün servis dosyaları ve enable durumları
```

---

## 3. Açılışta başlatmak: enable ve disable

`start` servisi **şimdi** başlatır, açılışta başlatmaz. `enable` ise açılışta başlatır, **şimdi** başlatmaz. İkisi farklı işlerdir:

| | Şimdi | Açılışta |
|---|---|---|
| `start` | Başlar | Değişmez |
| `enable` | Değişmez | Başlar |
| `enable --now` | Başlar | Başlar |
| `disable --now` | Durur | Başlamaz |

```bash
systemctl enable nabiz
```
```
Created symlink /etc/systemd/system/multi-user.target.wants/nabiz.service → /etc/systemd/system/nabiz.service.
```

`enable` aslında bir sembolik bağlantı oluşturur: servis dosyasında `WantedBy=multi-user.target` yazdığı için `multi-user.target.wants/` dizinine. Sistem `multi-user.target`'a ulaşınca bu dizindeki her şeyi başlatır.

`is-enabled` çıktısında `static` görürsen, dosyada `[Install]` bölümü yok demektir. Bu servis kendi başına enable edilemez, başka bir unit onu çağırınca başlar.

---

## 4. Tamamen engellemek: mask

`disable` edilen bir servis açılışta başlamaz ama elle veya başka bir servis tarafından yine başlatılabilir. `mask` ise servisi hiçbir şekilde başlatılamaz hale getirir:

```bash
systemctl mask redis-server
```
```
Created symlink /etc/systemd/system/redis-server.service → /dev/null.
```
```bash
systemctl start redis-server
```
```
Failed to start redis-server.service: Unit redis-server.service is masked.
```

Geri almak için `systemctl unmask redis-server`.

`mask`, servisin `/etc/systemd/system/` altındaki dosyasını `/dev/null`'a bağlar. Bu yüzden **unit dosyası zaten `/etc/systemd/system/` altında olan bir servis mask edilemez.** Kendi yazdığımız servisi mask etmeye çalışınca:

```
Failed to mask unit: File /etc/systemd/system/nabiz.service already exists.
```

Mask, paketlerle gelen (`/usr/lib/systemd/system/` altındaki) servisler içindir. Kendi servisini engellemek istiyorsan `disable --now` yeterli.

---

## 5. Kendi servisini yazmak

**1. Programı hazırla.** Servis olacak program, terminale bağlı olmadan çalışabilmeli ve çalıştırma izni olmalı:

```bash
sudo tee /usr/local/bin/nabiz.sh > /dev/null <<'EOF'
#!/bin/bash
# Her 10 saniyede bir sunucunun yukunu kaydeder
while true; do
  echo "nabiz: yuk $(cut -d' ' -f1 /proc/loadavg)"
  sleep 10
done
EOF
sudo chmod +x /usr/local/bin/nabiz.sh
```

`echo` ile ekrana yazılan her şey, servis olarak çalışınca kendiliğinden journal'a gider. Ayrı bir log dosyası tutmana gerek yok.

**2. Servis dosyasını yaz:**

```bash
sudo tee /etc/systemd/system/nabiz.service > /dev/null <<'EOF'
[Unit]
Description=Sunucu nabiz kaydi
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/nabiz.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**3. systemd'ye bildir, başlat ve açılışa ekle:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nabiz
systemctl status nabiz
journalctl -u nabiz
```

> [!TIP]
> Bazı kaynaklar, `/usr/lib/systemd/system/` altından `ssh.service` gibi hazır bir dosyayı kopyalayıp düzenlemeyi önerir. Bu durumda kopyalanan dosyanın `After=auditd.service`, `ExecReload=` gibi o servise özel satırları da gelir ve senin programın için anlamsız veya hatalı olur. Yukarıdaki gibi beş-altı satırlık bir dosyayı sıfırdan yazmak hem daha kısa hem daha güvenli. Seçeneklerin tam listesi: `man systemd.service`, `man systemd.unit`, `man systemd.exec`.

---

## 6. Servis dosyasının bölümleri

### [Unit]: tanım ve sıralama

| Seçenek | Anlamı |
|---|---|
| `Description=` | `systemctl status`'ta görünen açıklama |
| `After=` | Bu unit'lerden **sonra** başlat (sadece sıra, onları başlatmaz) |
| `Requires=` | Bu unit'ler de başlasın; onlar başlayamazsa bu da başlamasın |
| `Wants=` | Bu unit'ler de başlasın; başlayamazlarsa yine de devam et |

`After=` ile `Requires=` sık karıştırılır. `After=postgresql.service` tek başına PostgreSQL'i başlatmaz, sadece "o da başlıyorsa önce o başlasın" der. İkisi genelde birlikte yazılır.

### [Service]: nasıl çalışacak

| Seçenek | Anlamı |
|---|---|
| `ExecStart=` | Çalışacak komut, **tam yolla** |
| `ExecStartPre=` | `ExecStart`'tan önce çalışacak komut (ayar kontrolü gibi) |
| `ExecReload=` | `systemctl reload` ile çalışacak komut |
| `Type=` | Programın nasıl davrandığı, aşağıda |
| `Restart=` | Ne zaman yeniden başlatılsın ([9. bölüm](#9-çökünce-yeniden-başlatma-restart)) |
| `RestartSec=` | Yeniden başlatmadan önce bekleme süresi |
| `User=`, `Group=` | Hangi kullanıcıyla çalışsın (varsayılan root) |
| `WorkingDirectory=` | Çalışma dizini |
| `Environment=` | Ortam değişkeni: `Environment=ORTAM=test` |

`Type=` için en sık kullanılanlar:

| Değer | Ne zaman? |
|---|---|
| `simple` (varsayılan) | Program ön planda çalışmaya devam ediyor (bizim script gibi) |
| `forking` | Program kendini arka plana atıp ana süreci kapatıyor (eski tarz daemon'lar) |
| `oneshot` | Bir iş yapıp bitiyor (açılışta bir kez çalışan ayar script'i) |
| `notify` | Program hazır olduğunu systemd'ye kendisi bildiriyor (sshd gibi) |

### [Install]: enable edilince nereye bağlansın

| Seçenek | Anlamı |
|---|---|
| `WantedBy=multi-user.target` | Normal açılışta başlasın. Sunucu servisleri için neredeyse her zaman bu. |
| `WantedBy=graphical.target` | Sadece grafik arayüzle açılışta |

### Güvenlik: servisi root olmadan çalıştırmak

`User=` verilmezse servis root olarak çalışır. Root yetkisi gerektirmeyen bir programı ayrı bir sistem kullanıcısıyla çalıştırmak daha güvenlidir:

```bash
sudo useradd -r -s /usr/sbin/nologin nabizci
```

Servise `User=nabizci` ekledikten sonra:

```
USER         PID CMD
nabizci     1111 /bin/bash /usr/local/bin/nabiz.sh
```

---

## 7. Değişiklikten sonra: daemon-reload

systemd unit dosyalarını bir kez okuyup hafızada tutar. Çalışan bir servisin dosyasını değiştirdik ve doğrudan `restart` dedik:

```
Warning: The unit file, source configuration file or drop-ins of nabiz.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

Servis **eski** ayarlarla yeniden başladı. Değişikliğin geçerli olması için:

```bash
sudo systemctl daemon-reload
sudo systemctl restart nabiz
```

Denemede ilginç bir şey de gördük: **yeni** oluşturduğumuz bir servis dosyası, `daemon-reload` yapmadan da tanındı. systemd daha önce hiç görmediği bir unit istendiğinde onu diskten okur. Ama değiştirilen dosyalarda bu olmaz. Karışıklık yaşamamak için kural basit: unit dosyasına her dokunduğunda `daemon-reload`.

---

## 8. Var olan servisi değiştirmek: drop-in

Paketle gelen bir servisin dosyasını `/usr/lib/systemd/system/` altında düzenlersen, paket güncellenince değişikliğin silinir. Bunun yerine sadece değiştirmek istediğin satırları içeren bir **drop-in** dosyası yazılır:

```bash
sudo systemctl edit redis-server
```

Bu komut bir metin düzenleyici açar ve kaydettiğinde dosyayı `/etc/systemd/system/redis-server.service.d/override.conf` olarak yazar, `daemon-reload`'u da kendisi yapar. İçine sadece değişiklik yazılır:

```ini
[Service]
Environment=ORTAM=test
```

Aynı dosyayı elle de oluşturabilirsin. O zaman `daemon-reload`'u sen yaparsın:

```bash
sudo mkdir -p /etc/systemd/system/redis-server.service.d
sudo nano /etc/systemd/system/redis-server.service.d/override.conf
sudo systemctl daemon-reload
```

Sonuç, `systemctl cat` çıktısında iki dosya olarak görünür:

```
# /usr/lib/systemd/system/redis-server.service
...
# /etc/systemd/system/redis-server.service.d/override.conf
Environment=ORTAM=test
```

`systemctl status` de `Drop-In:` satırında bunu gösterir. Bütün değişiklikleri geri almak için `sudo systemctl revert redis-server`. Sistemdeki bütün drop-in'leri görmek için `systemd-delta`.

`systemctl edit --full servis` ise drop-in yerine dosyanın tam bir kopyasını `/etc/systemd/system/` altına alıp onu düzenletir.

> [!NOTE]
> `systemctl edit` etkileşimli bir terminal ister. Bu notu hazırladığımız ortamda terminal olmadığı için drop-in dosyasını elle oluşturup denedik. İki yöntemin sonucu aynıdır.

---

## 9. Çökünce yeniden başlatma: Restart

| Değer | Ne zaman yeniden başlatır? |
|---|---|
| `no` (varsayılan) | Hiçbir zaman |
| `on-failure` | Hata koduyla çıkarsa, sinyalle öldürülürse, zaman aşımında |
| `always` | Her durumda, düzgün kapansa bile (`systemctl stop` hariç) |

Servisin ana sürecini `kill -9` ile öldürdük:

```
Active: activating (auto-restart) (Result: signal) since ...; 1s ago
```

3 saniye (`RestartSec=3`) sonra tekrar `active (running)` oldu. Journal'da:

```
nabiz.service: Failed with result 'signal'.
nabiz.service: Scheduled restart job, restart counter is at 2.
Started nabiz.service - Sunucu nabiz kaydi.
```

### Sonsuz döngü koruması

Program her başladığında hemen hata veriyorsa ne olur? `Restart=always` ve `RestartSec=1` ile, başlar başlamaz `exit 1` yapan bir script'i servis yaptık:

```
bozuk.service: Scheduled restart job, restart counter is at 5.
bozuk.service: Start request repeated too quickly.
bozuk.service: Failed with result 'exit-code'.
Failed to start bozuk.service - Bozuk servis denemesi.
```

systemd varsayılan olarak **10 saniye içinde 5 başlatma** denemesinden sonra vazgeçer ve servisi `failed` durumuna alır:

```bash
systemctl show bozuk -p StartLimitBurst -p StartLimitIntervalUSec
```
```
StartLimitIntervalUSec=10s
StartLimitBurst=5
```

`Restart=always` bozuk bir programı kurtarmaz, sadece hatayı geciktirir. Sorunu düzelttikten sonra sayacı sıfırlayıp tekrar başlat:

```bash
sudo systemctl reset-failed bozuk
sudo systemctl start bozuk
```

### KillMode=process kullanma

Bazı kaynaklarda servis dosyasında `KillMode=process` görürsün. Bu, servis durdurulurken sadece ana süreci öldürür. Denemede servisi durdurduk, script'in başlattığı `sleep` süreçleri sahipsiz olarak çalışmaya devam etti:

```
1173 sleep 10
1205 sleep 10
```

Varsayılan (`control-group`) servise ait **bütün** süreçleri durdurur. Ne yaptığını bilmiyorsan bu satırı ekleme.

---

## 10. Sorun giderme adımları

Bir servis başlamıyorsa sırayla:

**1. Durumuna bak:**

```bash
systemctl status bozuk
```
```
× bozuk.service - Bozuk servis denemesi
     Loaded: loaded (/etc/systemd/system/bozuk.service; static)
     Active: failed (Result: exit-code) since Thu 2026-11-26 08:39:21 UTC; 1s ago
    Process: 618 ExecStart=/usr/local/bin/yok.sh (code=exited, status=203/EXEC)
```

`status=203/EXEC` bir ipucu ([11. bölüm](#11-sık-görülen-hatalar-ve-anlamları)).

**2. Loglarını oku.** `status` sadece son birkaç satırı gösterir:

```bash
journalctl -u bozuk                    # bu servisin bütün logları
journalctl -u bozuk -n 20              # son 20 satır
journalctl -u bozuk -f                 # canlı izle (Ctrl+C ile çık)
journalctl -u bozuk -b                 # sadece bu açılıştan beri
journalctl -u bozuk --since "10 min ago"
journalctl -u bozuk -p err             # sadece hata ve daha önemlileri
journalctl -xeu bozuk                  # sona git, açıklamalarla (systemd'nin önerdiği)
```

Denemedeki bozuk servisin logunda:

```
/etc/systemd/system/bozuk.service:5: Unknown key name 'Restrat' in section 'Service', ignoring.
bozuk.service: Main process exited, code=exited, status=203/EXEC
```

İki ayrı hata: `Restart` yanlış yazılmış (systemd satırı sessizce **yok saydı**, servis yine de başladı) ve program çalıştırılamadı. Journal'ın daha ayrıntılı kullanımı: [Log Dosyaları](../03-isletim-ve-dagitim/04-log-dosyalari.md).

**3. Unit dosyasını kontrol et.** `systemd-analyze verify` dosyadaki hataları servisi başlatmadan bulur:

```bash
systemd-analyze verify /etc/systemd/system/bozuk.service
```
```
/etc/systemd/system/bozuk.service:5: Unknown key name 'Restrat' in section 'Service', ignoring.
bozuk.service: Command /usr/local/bin/yok.sh is not executable: No such file or directory
```

`status`'un sadece `203/EXEC` dediği yerde bu komut nedenini açıkça yazdı: dosya yok.

**4. Programı elle çalıştır.** Servisin `ExecStart` satırındaki komutu, servisin çalıştığı kullanıcıyla terminalde çalıştır. Hata mesajını doğrudan görürsün:

```bash
sudo -u nabizci /usr/local/bin/nabiz.sh
```

**5. Bağımlılıklara bak.** Servis başka bir şeyin başlamasını bekliyorsa:

```bash
systemctl list-dependencies nabiz
```

**6. Düzelt, `daemon-reload`, `reset-failed`, `start`.**

---

## 11. Sık görülen hatalar ve anlamları

| Belirti | Olası neden | Ne yapmalı? |
|---|---|---|
| `status=203/EXEC` | `ExecStart` yolu yanlış, dosya yok veya çalıştırma izni yok, script'in ilk satırında `#!` eksik | `ls -l` ile dosyaya ve iznine bak, `chmod +x` |
| `status=217/USER` | `User=` ile verilen kullanıcı yok | `id kullanici` |
| `status=1/FAILURE` | Program kendisi hata verdi | `journalctl -u` ile programın mesajına bak |
| `Unknown key name` | Unit dosyasında yazım hatası, satır yok sayıldı | `systemd-analyze verify` |
| `Start request repeated too quickly` | 10 saniyede 5 kez çöktü | Asıl hatayı bul, sonra `reset-failed` |
| `Unit ... is masked` | Servis mask edilmiş | `systemctl unmask` |
| `Unit ... not found` | Ad yanlış veya dosya yanlış dizinde | `systemctl list-unit-files \| grep ad` |
| `changed on disk` uyarısı | Dosya değişti, `daemon-reload` yapılmadı | `daemon-reload` |
| `Job type reload is not applicable` | Serviste `ExecReload` yok | `restart` veya `reload-or-restart` |
| Açılışta başlamıyor | `enable` edilmemiş | `systemctl is-enabled` |

Denemede çalıştırma izni olmayan (`chmod 644`) bir script de aynı `203/EXEC` hatasını verdi. Yani `203` gördüğünde hem yolu hem izni kontrol et. `systemd-analyze verify` ikisini ayırır: dosya yoksa `No such file or directory`, izin yoksa `is not executable: Permission denied` yazar.

---

## 12. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `systemctl status servis` | Durum ve son loglar |
| `systemctl start` / `stop` / `restart` | Başlat / durdur / yeniden başlat |
| `systemctl reload` / `reload-or-restart` | Ayarları yeniden okut |
| `systemctl enable` / `disable` | Açılışta başlat / başlatma |
| `systemctl enable --now` / `disable --now` | Aynısı, şimdi de |
| `systemctl mask` / `unmask` | Tamamen engelle / kaldır |
| `systemctl is-active` / `is-enabled` / `is-failed` | Tek kelimelik durum |
| `systemctl cat servis` | Unit dosyası ve drop-in'ler |
| `systemctl show servis -p Ayar` | Geçerli bir ayarın değeri |
| `systemctl edit servis` | Drop-in oluştur |
| `systemctl edit --full servis` | Tam kopya oluşturup düzenle |
| `systemctl revert servis` | Yerel değişiklikleri geri al |
| `systemctl daemon-reload` | Unit dosyalarını yeniden oku |
| `systemctl reset-failed servis` | Hata durumunu ve başlatma sayacını sıfırla |
| `systemctl list-units --type=service --all` | Bütün servisler |
| `systemctl --failed` | Hata verenler |
| `systemctl list-unit-files` | Bütün unit dosyaları ve enable durumları |
| `systemctl list-dependencies servis` | Bağımlılıklar |
| `systemd-analyze verify dosya` | Unit dosyasını denetle |
| `journalctl -u servis` | Servisin logları |
| `journalctl -xeu servis` | Son loglar, açıklamalı |

---

## 13. Sık yapılan hatalar

- **`enable`'ın servisi şimdi de başlattığını sanmak.** Başlatmaz, `--now` gerekir.
- **Unit dosyasını değiştirip `daemon-reload` yapmamak.** Servis eski ayarlarla çalışır.
- **`ExecStart`'ta tam yol yazmamak.** `ExecStart=nabiz.sh` çalışmaz.
- **Script'e çalıştırma izni vermemek.** `203/EXEC`.
- **Paket servislerini `/usr/lib/systemd/system/` altında düzenlemek.** Güncellemede silinir, drop-in kullan.
- **`After=`'ın bağımlılığı başlattığını sanmak.** Sadece sırayı belirler.
- **Çöken servisi `Restart=always` ile "düzeltmek".** 5 denemeden sonra yine `failed` olur.
- **Hatayı düzeltip `reset-failed` yapmadan başlatmaya çalışmak.**
- **Uzak sunucuda `ssh` servisini durdurmak veya mask etmek.** Bağlantın kopar ve geri giremezsin.

---

## 14. Alıştırmalar

**1.** `/usr/local/bin/saat.sh` adında, her 30 saniyede bir `date` çıktısını yazan bir script oluştur. Bunu `saat.service` adında bir servis yap: çökerse 10 saniye sonra yeniden başlasın, açılışta başlasın. Hem şimdi başlat hem açılışa ekle.

<details>
<summary>Çözüm</summary>

```bash
sudo tee /usr/local/bin/saat.sh > /dev/null <<'EOF'
#!/bin/bash
while true; do date; sleep 30; done
EOF
sudo chmod +x /usr/local/bin/saat.sh

sudo tee /etc/systemd/system/saat.service > /dev/null <<'EOF'
[Unit]
Description=Saat kaydi

[Service]
ExecStart=/usr/local/bin/saat.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now saat
systemctl is-active saat; systemctl is-enabled saat
```
</details>

**2.** `saat.service`'in ana sürecini `kill -9` ile öldür. Servisin yeniden başladığını journal'dan doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo kill -9 $(systemctl show -p MainPID --value saat)
sleep 12
journalctl -u saat -n 5
systemctl show saat -p NRestarts
```

Journal'da `Scheduled restart job` satırını görmelisin, `NRestarts` 1 olmalı.
</details>

**3.** `saat.service`'in dosyasını değiştirmeden, bir drop-in ile `RestartSec` değerini 3 saniye yap. Geçerli değeri kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl edit saat
# Açılan düzenleyiciye:
# [Service]
# RestartSec=3

systemctl show saat -p RestartUSec     # RestartUSec=3s
systemctl cat saat                      # iki dosya görünür
```
</details>

**4.** Aşağıdaki servis başlamıyor. `systemd-analyze verify` ve `journalctl` ile sorunları bul ve düzelt.

```ini
[Unit]
Description=Hatali servis
[Service]
ExecStart=/usr/local/bin/hatali.sh
Restrat=always
```

`/usr/local/bin/hatali.sh` dosyası var ama izni `644`.

<details>
<summary>Çözüm</summary>

```bash
systemd-analyze verify /etc/systemd/system/hatali.service
systemctl status hatali          # status=203/EXEC
ls -l /usr/local/bin/hatali.sh   # x izni yok

sudo chmod +x /usr/local/bin/hatali.sh
sudo sed -i 's/Restrat/Restart/' /etc/systemd/system/hatali.service
sudo systemctl daemon-reload
sudo systemctl reset-failed hatali
sudo systemctl start hatali
```
</details>

**5.** `cron` servisinin hiçbir şekilde başlatılamamasını sağla, başlatmayı deneyip hatayı gör, sonra eski haline getir.

<details>
<summary>Çözüm</summary>

```bash
sudo systemctl stop cron
sudo systemctl mask cron
sudo systemctl start cron        # Unit cron.service is masked.
sudo systemctl unmask cron
sudo systemctl start cron
```
</details>

---

Önceki konu: [Git ile Sürüm Kontrolü](12-git.md) · Sonraki konu: [Performans İzleme](14-performans-izleme.md)
