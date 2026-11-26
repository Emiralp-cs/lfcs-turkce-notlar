# SELinux

> **Resmi sınav maddesi:** Create and enforce MAC using SELinux

Normal Linux izinleri (`rwx`, sahip, grup) **DAC** (Discretionary Access Control) modelidir: dosyanın sahibi kimin erişeceğine kendisi karar verir. root her şeye erişebilir. **SELinux** bunun üstüne bir katman daha ekler: **MAC** (Mandatory Access Control). Her dosyaya ve sürece bir **etiket** verilir ve hangi etiketin hangisine ne yapabileceğini, sahibinden bağımsız olarak, merkezi bir **politika** belirler. Bir web sunucusu ele geçirilse bile, root olarak çalışsa dahi, politikanın izin vermediği dosyalara dokunamaz.

Senaryo: Bir Ubuntu sunucusunda AppArmor yerine SELinux'u etkinleştireceğiz, önce sadece izleyen modda çalıştırıp ne engelleyeceğini göreceğiz, sonra zorlayıcı moda geçeceğiz. Ardından web sitesini standart olmayan bir dizinden (`/srv/site`) yayınlamak ve SSH'ı 2222 portuna taşımak için SELinux'a gerekli izinleri vereceğiz.

> [!WARNING]
> Bu dosyadaki komutlar **denenmedi.** Deneme ortamının çekirdeği SELinux desteği olmadan derlenmiş (`getenforce` → `Disabled`, `/sys/fs/selinux` yok) ve sistem yeniden başlatılamıyor. Paket adları Ubuntu 24.04 depolarından, komutların sözdizimi kurulu araçların yardım çıktıları ve man sayfalarından kontrol edildi. `selinux-activate` betiğinin ne yaptığı, betiğin kendisi okunarak doğrulandı. Kendi sanal makinende dene; bir hata yaparsan sistem açılmayabilir, önce bir anlık görüntü (snapshot) al.

## İçindekiler

1. [Temel kavramlar](#1-temel-kavramlar)
2. [Etiketleri görmek: -Z](#2-etiketleri-görmek--z)
3. [Modlar: Enforcing, Permissive, Disabled](#3-modlar-enforcing-permissive-disabled)
4. [Ubuntu'da SELinux'u etkinleştirmek](#4-ubuntuda-selinuxu-etkinleştirmek)
5. [Reddedilenleri okumak: audit2why](#5-reddedilenleri-okumak-audit2why)
6. [Mod değiştirmek: geçici ve kalıcı](#6-mod-değiştirmek-geçici-ve-kalıcı)
7. [Dosya etiketlerini düzeltmek: restorecon ve semanage fcontext](#7-dosya-etiketlerini-düzeltmek-restorecon-ve-semanage-fcontext)
8. [Geçici etiket: chcon](#8-geçici-etiket-chcon)
9. [Hazır anahtarlar: boolean'lar](#9-hazır-anahtarlar-booleanlar)
10. [Port etiketleri: semanage port](#10-port-etiketleri-semanage-port)
11. [Kendi kuralını üretmek: audit2allow](#11-kendi-kuralını-üretmek-audit2allow)
12. [Kullanıcı eşleşmeleri](#12-kullanıcı-eşleşmeleri)
13. [AppArmor ile karşılaştırma](#13-apparmor-ile-karşılaştırma)
14. [Sorun giderme sırası](#14-sorun-giderme-sırası)
15. [Komut özeti](#15-komut-özeti)
16. [Sık yapılan hatalar](#16-sık-yapılan-hatalar)
17. [Alıştırmalar](#17-alıştırmalar)

---

## 1. Temel kavramlar

Her dosyanın ve sürecin bir **güvenlik bağlamı** (security context) vardır. Dört parçadan oluşur:

```
system_u:object_r:httpd_sys_content_t:s0
   │        │             │            │
 kullanıcı  rol          tür         seviye
```

| Parça | Anlamı | Önemi |
|---|---|---|
| Kullanıcı (`_u`) | SELinux kullanıcısı, Linux kullanıcısından farklıdır | Az |
| Rol (`_r`) | Kullanıcının hangi türlere geçebileceği | Az |
| **Tür** (`_t`) | Dosyanın veya sürecin türü | **Kuralların neredeyse hepsi bunu kullanır** |
| Seviye | Çok seviyeli güvenlik (MLS), standart politikada hep `s0` | Az |

Pratikte SELinux ile çalışmak, **türlerle** çalışmak demektir. Bu modele **type enforcement** denir: politika "`httpd_t` türündeki süreç, `httpd_sys_content_t` türündeki dosyaları okuyabilir" gibi kurallardan oluşur.

Süreçlerin türüne **domain** de denir. Bir program çalıştırıldığında, dosyasının türüne göre süreç belirli bir domain'e girer (**type transition**): `/usr/sbin/sshd` dosyası `sshd_exec_t` türündedir, çalışınca süreç `sshd_t` domain'ine geçer.

| Model | Kim karar verir? | root'u sınırlar mı? |
|---|---|---|
| DAC (normal izinler) | Dosyanın sahibi | Hayır |
| MAC (SELinux) | Merkezi politika | Evet |

SELinux, DAC'ın **yerine geçmez**, üstüne eklenir. Bir erişim için önce normal izinler, sonra SELinux kontrol edilir. İkisi de izin vermelidir.

---

## 2. Etiketleri görmek: -Z

Birçok komutun `-Z` seçeneği SELinux bağlamını gösterir:

```bash
ls -Z /usr/sbin/sshd          # dosyanın bağlamı
ls -ldZ /srv/site             # dizinin kendisi
ps -eZ | grep sshd            # süreçlerin bağlamı (ps axZ de olur)
id -Z                         # kendi bağlamın
```

Beklenen çıktılar:

```
system_u:object_r:sshd_exec_t:s0 /usr/sbin/sshd
system_u:system_r:sshd_t:s0     905 ?        00:00:00 sshd
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

`sshd_exec_t` dosya çalışınca `sshd_t` domain'i oluşmuş. Giriş yapan kullanıcıların çoğu `unconfined_t` ile çalışır, yani SELinux onları kısıtlamaz. Asıl kısıtlanan servislerdir.

---

## 3. Modlar: Enforcing, Permissive, Disabled

| Mod | Ne yapar? |
|---|---|
| **Enforcing** | Politikaya aykırı erişimleri engeller ve loglar |
| **Permissive** | Hiçbir şeyi engellemez, ama "bunu engellerdim" diye loglar |
| **Disabled** | SELinux tamamen kapalı, etiket de tutulmaz |

```bash
getenforce     # Enforcing / Permissive / Disabled
sestatus       # ayrıntılı: mod, yapılandırmadaki mod, politika adı
```

Permissive mod, kuralların ne engelleyeceğini sistemi bozmadan görmenin yoludur. Yeni bir SELinux kurulumunda veya yeni bir servis eklerken önce permissive'de çalıştırıp logları incelemek, sonra enforcing'e geçmek doğru sıradır.

Disabled'dan diğer modlara geçmek yeniden başlatma ve bütün dosyaların yeniden etiketlenmesini gerektirir, çünkü disabled iken dosyalar etiketlenmez.

---

## 4. Ubuntu'da SELinux'u etkinleştirmek

RHEL, Fedora ve Rocky Linux'ta SELinux kurulu ve açık gelir, bu bölüm onlarda gerekmez. Ubuntu ise varsayılan olarak **AppArmor** kullanır. İkisi aynı anda çalışamaz.

**1. Hangisinin çalıştığına bak:**

```bash
sudo aa-status          # AppArmor
sestatus                # SELinux (kurulu değilse komut da yoktur)
```

**2. AppArmor'u durdur ve açılışta başlamasını engelle:**

```bash
sudo systemctl disable --now apparmor
```

Sadece `stop` dersen yeniden başlatınca AppArmor geri gelir.

**3. SELinux araçlarını, politikayı ve auditd'yi kur:**

```bash
sudo apt install selinux-basics selinux-policy-default auditd
```

| Paket | Ne için? |
|---|---|
| `selinux-basics` | `selinux-activate`, `check-selinux-installation` |
| `selinux-policy-default` | Hazır politika |
| `auditd` | Reddedilen erişimleri `/var/log/audit/audit.log`'a yazar |
| `policycoreutils-python-utils` | `semanage`, `audit2allow`, `audit2why` (genelde bağımlılık olarak gelir) |

**4. Etkinleştir:**

```bash
sudo selinux-activate
```

Bu betik iki şey yapar (betiğin içeriğinden doğrulandı):
- `/etc/default/grub` dosyasındaki `GRUB_CMDLINE_LINUX` satırına `security=selinux` ekler ve GRUB yapılandırmasını günceller. SELinux çekirdek açılırken devreye girdiği için bir servis gibi `systemctl start` ile açılamaz.
- Kök dizinde `/.autorelabel` dosyası oluşturur. Bu, bir sonraki açılışta bütün dosya sistemine etiket verilmesini sağlar.

**5. Yeniden başlat:**

```bash
sudo reboot
```

İlk açılış uzun sürer: bütün dosyalar etiketlenir, sonra sistem bir kez daha kendiliğinden yeniden başlayabilir.

**6. Kontrol et:**

```bash
sestatus
```

Ubuntu'da ilk etkinleştirmeden sonra mod **Permissive** olur. Bu kasıtlıdır: önce neyin engelleneceğini göreceksin.

---

## 5. Reddedilenleri okumak: audit2why

Permissive modda sistemi normal kullan: SSH ile bağlan, servisleri başlat, web sitesini aç. SELinux engelleyeceği her şeyi `/var/log/audit/audit.log`'a yazar. Bu kayıtlara **AVC** (Access Vector Cache) mesajı denir:

```bash
sudo ausearch -m AVC -ts recent      # son AVC kayıtları
sudo audit2why --all | less          # hepsini, nedeniyle birlikte açıkla
```

Bir AVC kaydı şuna benzer:

```
type=AVC msg=audit(...): avc:  denied  { read } for  pid=1234 comm="nginx"
name="index.html" scontext=system_u:system_r:httpd_t:s0
tcontext=unconfined_u:object_r:var_t:s0 tclass=file permissive=1
```

| Alan | Anlamı |
|---|---|
| `denied { read }` | Engellenen işlem |
| `comm="nginx"` | Hangi program |
| `scontext=...httpd_t` | **Kaynak**: işlemi yapmaya çalışan sürecin türü |
| `tcontext=...var_t` | **Hedef**: erişilmek istenen dosyanın türü |
| `tclass=file` | Hedef bir dosya |
| `permissive=1` | Permissive modda olduğu için gerçekte engellenmedi, sadece loglandı |

Bu örnek tipik bir sorun: web sunucusu (`httpd_t`) `var_t` türündeki bir dosyayı okumak istiyor. Web içeriği `httpd_sys_content_t` türünde olmalıydı. Çözüm kural yazmak değil, dosyanın etiketini düzeltmektir ([7. bölüm](#7-dosya-etiketlerini-düzeltmek-restorecon-ve-semanage-fcontext)).

auditd kurulu değilse AVC kayıtları journal'a gider: `journalctl -g 'avc:  denied'`.

---

## 6. Mod değiştirmek: geçici ve kalıcı

**Geçici** (yeniden başlatmaya kadar):

```bash
sudo setenforce 1      # Enforcing
sudo setenforce 0      # Permissive
getenforce
```

`setenforce`, SELinux disabled iken çalışmaz, sadece Enforcing ile Permissive arasında geçiş yapar.

**Kalıcı:** `/etc/selinux/config` dosyası:

```
SELINUX=enforcing        # enforcing / permissive / disabled
SELINUXTYPE=default      # Ubuntu'da default, RHEL'de targeted
```

Dosya yeniden başlatınca okunur. `setenforce` ile yapılan geçici değişiklik de açılışta bu dosyadaki değere döner.

Önerilen sıra: `setenforce 1` ile dene, servisler normal çalışıyorsa `/etc/selinux/config`'i `enforcing` yap, sonra bir kez yeniden başlatıp `getenforce` ile kontrol et. Bir şey bozulursa `setenforce 0` ile hemen geri dönebilirsin.

| | `setenforce` | `/etc/selinux/config` |
|---|---|---|
| Ne zaman etkili? | Hemen | Sonraki açılışta |
| Kalıcı mı? | Hayır | Evet |
| Disabled'a geçebilir mi? | Hayır | Evet |

Bir sistem SELinux yüzünden açılmıyorsa, GRUB'da açılış satırına `enforcing=0` ekleyerek o açılış için permissive moda alınabilir ([Arızadan Kurtarma](02-arizadan-kurtarma.md#2-grub-menüsüne-girmek-ve-açılış-parametresi-vermek)).

---

## 7. Dosya etiketlerini düzeltmek: restorecon ve semanage fcontext

SELinux sorunlarının çoğu **yanlış etiketli dosyalardan** kaynaklanır. Politika, her yol için hangi etiketin olması gerektiğini bilir (`/var/www` altı `httpd_sys_content_t` gibi). Bir dosya bu kurala uymuyorsa `restorecon` onu düzeltir:

```bash
sudo restorecon -v /var/www/html/index.html
sudo restorecon -Rv /var/www/          # dizin, içindekilerle birlikte
```

`-v` neyi değiştirdiğini yazar. `restorecon` varsayılan olarak sadece **tür** kısmını düzeltir, bütün bağlamı zorla düzeltmek için `-F` eklenir.

### Etiketler nasıl bozulur?

- **`mv` etiketi korur, `cp` korumaz.** Ev dizininde (`user_home_t`) oluşturup `/var/www/html`'e **taşıdığın** dosya `user_home_t` olarak kalır, web sunucusu okuyamaz. Kopyalasaydın hedef dizinin etiketini alırdı.
- Dosyayı politikanın bilmediği bir dizine koymak.
- `chcon` ile elle değiştirmek ([8. bölüm](#8-geçici-etiket-chcon)).

### Standart olmayan bir dizin: semanage fcontext

Web sitesi `/srv/site` dizininde. Politika bu dizini tanımaz, `restorecon` ona varsayılan bir etiket (`var_t` gibi) verir ve web sunucusu erişemez. Politikaya "bu yol da web içeriğidir" kuralını eklemek gerekir:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/site(/.*)?'
sudo restorecon -Rv /srv/site
ls -ldZ /srv/site
```

| Parça | Anlamı |
|---|---|
| `-a` | Kural ekle (`--add`) |
| `-t httpd_sys_content_t` | Verilecek tür |
| `'/srv/site(/.*)?'` | Yol, **düzenli ifade** olarak: `/srv/site` ve altındaki her şey |

`semanage fcontext` sadece **kuralı** yazar, dosyalara dokunmaz. Kuralı uygulamak için ardından `restorecon` gerekir. İkisi her zaman birlikte kullanılır.

```bash
sudo semanage fcontext -l | grep /srv/site     # eklenen kural
sudo semanage fcontext -d '/srv/site(/.*)?'     # kuralı sil
```

Benzer bir dizinin etiketini örnek almak için: `sudo semanage fcontext -a -e /var/www /srv/site` (`/srv/site`'a `/var/www` ile aynı kuralları uygula).

---

## 8. Geçici etiket: chcon

`chcon`, bir dosyanın etiketini doğrudan değiştirir:

```bash
sudo chcon -t httpd_sys_content_t /srv/site/index.html
sudo chcon -R -t httpd_sys_content_t /srv/site
sudo chcon --reference=/var/www/html /srv/site      # başka bir dosyanın etiketini kopyala
```

`-u` kullanıcı, `-r` rol, `-t` tür kısmını değiştirir. En sık `-t` kullanılır.

> [!WARNING]
> `chcon` ile yapılan değişiklik **kalıcı değildir.** Bir sonraki `restorecon` veya tam yeniden etiketleme, dosyayı politikadaki kurala geri döndürür. Hızlı bir deneme için kullan, kalıcı çözüm için `semanage fcontext` + `restorecon`.

| | `chcon` | `semanage fcontext` + `restorecon` |
|---|---|---|
| Neyi değiştirir? | Dosyanın etiketini | Politikadaki kuralı, sonra dosyayı |
| `restorecon` sonrası | Kaybolur | Kalır |
| Ne zaman? | Deneme | Kalıcı çözüm |

---

## 9. Hazır anahtarlar: boolean'lar

Politikada sık ihtiyaç duyulan bazı davranışlar hazır tanımlıdır ama varsayılan olarak kapalıdır. Bunları açıp kapatmak için kural yazmaya gerek yoktur:

```bash
getsebool -a                               # bütün boolean'lar
getsebool -a | grep httpd                  # web sunucusuyla ilgili olanlar
sudo semanage boolean -l | grep httpd      # açıklamalarıyla
```

Örnek: Nginx ters proxy olarak arkadaki bir uygulamaya bağlanacak ([Ters Proxy](../02-ag-yonetimi/09-ters-proxy-ve-yuk-dengeleme.md)). SELinux web sunucusunun ağ bağlantısı açmasını varsayılan olarak engeller:

```bash
sudo setsebool httpd_can_network_connect on       # geçici
sudo setsebool -P httpd_can_network_connect on    # kalıcı
getsebool httpd_can_network_connect
```

**`-P` olmadan** yapılan değişiklik yeniden başlatınca kaybolur. Sınavda kalıcı isteniyorsa `-P` unutulmamalı.

Diğer örnekler: `httpd_enable_homedirs` (kullanıcıların `~/public_html`'i), `virt_use_nfs` (sanal makinelerin NFS kullanması), `samba_enable_home_dirs`.

---

## 10. Port etiketleri: semanage port

SELinux, servislerin hangi portları dinleyebileceğini de kısıtlar. SSH'ı 2222 portuna taşımak için sadece `sshd_config`'i değiştirmek yetmez ([SSH](../02-ag-yonetimi/05-ssh.md)), SELinux 2222'nin SSH'a ait olduğunu bilmediği için sshd başlayamaz.

```bash
sudo semanage port -l | grep ssh              # ssh_port_t   tcp   22
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -l | grep ssh              # ssh_port_t   tcp   2222, 22
```

Sonra `sshd_config`'e `Port 2222` eklenir. Ubuntu 24.04'te SSH `ssh.socket` ile başladığı için port değişikliğinden sonra `sudo systemctl daemon-reload` ve `sudo systemctl restart ssh.socket` gerekir ([SSH](../02-ag-yonetimi/05-ssh.md)).

Kaldırmak için: `sudo semanage port -d -t ssh_port_t -p tcp 2222`.

Web sunucusu için aynı mantık: Nginx'i 8081'de dinletmek için `http_port_t` türüne eklenir. Hangi portların zaten tanımlı olduğuna önce `semanage port -l | grep http` ile bak.

---

## 11. Kendi kuralını üretmek: audit2allow

Etiket düzeltmek, boolean veya port tanımı sorunu çözmüyorsa, AVC kayıtlarından yeni bir politika modülü üretilebilir:

```bash
sudo audit2allow -a -M yerel_panel        # bütün AVC kayıtlarından modül üret
```
```
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i yerel_panel.pp
```

İki dosya oluşur: `yerel_panel.te` (okunabilir kurallar) ve `yerel_panel.pp` (yüklenecek derlenmiş modül).

**Yüklemeden önce `.te` dosyasını oku:**

```
allow httpd_t var_t:file { getattr open read };
```

"Web sunucusu `var_t` türündeki **bütün** dosyaları okuyabilir." Bu, sorunu çözer ama çok geniş bir izindir: `/var` altındaki birçok dosya `var_t` türündedir. Doğru çözüm dosyaların etiketini düzeltmekti. audit2allow, her reddi gerçek bir eksik gibi kurala çevirir, sorunun yanlış etiket olduğunu bilmez.

```bash
sudo semodule -i yerel_panel.pp     # yükle
sudo semodule -l | grep yerel       # yüklü modüller
sudo semodule -r yerel_panel        # kaldır
```

Kural: önce `restorecon` / `semanage fcontext`, sonra boolean, sonra port, en son `audit2allow`.

---

## 12. Kullanıcı eşleşmeleri

Linux kullanıcıları, giriş yaptıklarında bir SELinux kullanıcısıyla eşleşir:

```bash
sudo semanage login -l      # Linux kullanıcısı → SELinux kullanıcısı
sudo semanage user -l       # SELinux kullanıcıları ve rolleri
```

Varsayılan olarak herkes `unconfined_u` ile eşleşir ve SELinux tarafından kısıtlanmaz. Kısıtlı kullanıcılar (`user_u`, `staff_u`) tanımlanabilir ama bu sınav kapsamının ötesindedir.

---

## 13. AppArmor ile karşılaştırma

| | SELinux | AppArmor |
|---|---|---|
| Varsayılan olduğu dağıtımlar | RHEL, Fedora, Rocky (2025'ten beri yeni openSUSE kurulumları da) | Ubuntu, Debian |
| Neye göre kural? | Etiketlere (dosya taşınınca etiketi de gider) | Dosya yollarına |
| Kurallar | Merkezi politika, modüller | Program başına profil (`/etc/apparmor.d/`) |
| Durum | `getenforce`, `sestatus` | `aa-status` |
| Geçici olarak sadece logla | `setenforce 0` | `aa-complain /yol/program` |
| Zorla | `setenforce 1` | `aa-enforce /yol/program` |

Sınav sorusu SELinux istiyorsa ve sistem Ubuntu ise, önce [4. bölümdeki](#4-ubuntuda-selinuxu-etkinleştirmek) kurulum gerekebilir. Sistemde hangisinin çalıştığına her zaman önce bak.

---

## 14. Sorun giderme sırası

Bir servis normal izinler doğru olduğu halde `Permission denied` veriyorsa:

1. SELinux açık mı? `getenforce`
2. SELinux mu engelliyor? Kısa bir test: `sudo setenforce 0`, servisi dene. Çalışıyorsa sorun SELinux'ta. **Sonra hemen `setenforce 1`.**
3. Ne engellendi? `sudo ausearch -m AVC -ts recent` veya `sudo audit2why -a`
4. Hedef dosyanın etiketi doğru mu? `ls -Z`, düzeltmek için `restorecon` veya `semanage fcontext`
5. İlgili bir boolean var mı? `getsebool -a | grep servis`
6. Port mu? `semanage port -l`
7. Hiçbiri değilse: `audit2allow`, `.te`'yi okuyarak.

---

## 15. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `getenforce`, `sestatus` | Mod ve durum |
| `setenforce 1` / `0` | Geçici Enforcing / Permissive |
| `/etc/selinux/config` | Kalıcı mod |
| `selinux-activate` | Ubuntu'da etkinleştir (GRUB + autorelabel) |
| `ls -Z`, `ps -eZ`, `id -Z` | Bağlamlar |
| `ausearch -m AVC -ts recent` | Son reddedilenler |
| `audit2why -a` | Reddedilenleri açıkla |
| `restorecon -Rv yol` | Etiketleri politikaya göre düzelt |
| `semanage fcontext -a -t TÜR 'yol(/.*)?'` | Kalıcı etiket kuralı |
| `semanage fcontext -l` / `-d` | Kuralları listele / sil |
| `chcon -t TÜR dosya` | Geçici etiket |
| `getsebool -a`, `semanage boolean -l` | Boolean'lar |
| `setsebool -P ad on` | Boolean'ı kalıcı aç |
| `semanage port -l` / `-a -t TÜR -p tcp PORT` | Port etiketleri |
| `audit2allow -a -M ad` | Kayıtlardan modül üret |
| `semodule -i ad.pp` / `-l` / `-r ad` | Modül yükle / listele / kaldır |
| `semanage login -l`, `semanage user -l` | Kullanıcı eşleşmeleri |

---

## 16. Sık yapılan hatalar

- **Ubuntu'da AppArmor'u kapatmadan SELinux kurmak.** İkisi birlikte çalışmaz.
- **AppArmor'u sadece `stop` edip `disable` etmemek.** Açılışta geri gelir.
- **Doğrudan enforcing'e geçmek.** Önce permissive'de logları incele.
- **`setenforce`'u kalıcı sanmak.** Kalıcı ayar `/etc/selinux/config`.
- **`setsebool`'da `-P`'yi unutmak.**
- **`semanage fcontext`'ten sonra `restorecon` yapmamak.** Kural yazılır ama dosyalar değişmez.
- **`chcon`'u kalıcı çözüm sanmak.**
- **Dosyaları `mv` ile taşıyıp etiketlerin değiştiğini sanmak.** `mv` etiketi korur.
- **`audit2allow`'un ürettiği modülü okumadan yüklemek.** Çok geniş izin verebilir.
- **Sorunu "çözmek" için SELinux'u disabled yapmak.**
- **`scontext` ile `tcontext`'i karıştırmak.** Kaynak: işlemi yapan, hedef: erişilen.

---

## 17. Alıştırmalar

> Bu alıştırmalar SELinux'un etkin olduğu bir sanal makine gerektirir (Rocky Linux, Fedora veya 4. bölümdeki gibi hazırlanmış bir Ubuntu).

**1.** SELinux'un şu anki modunu, yapılandırma dosyasındaki modu ve `sshd` süreçlerinin hangi domain'de çalıştığını bul.

<details>
<summary>Çözüm</summary>

```bash
getenforce
sestatus
grep ^SELINUX= /etc/selinux/config
ps -eZ | grep sshd
```
</details>

**2.** SELinux'u bu oturum için permissive moda al, sonra kalıcı olarak enforcing olmasını sağla ve şimdi de enforcing'e geçir.

<details>
<summary>Çözüm</summary>

```bash
sudo setenforce 0
getenforce                                              # Permissive
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
sudo setenforce 1
getenforce                                              # Enforcing
```
</details>

**3.** `/srv/site` dizinini oluştur, içine `index.html` koy. Dizinin ve içindekilerin web içeriği olarak etiketlenmesini, `restorecon` sonrasında da kalacak şekilde sağla.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /srv/site
echo merhaba | sudo tee /srv/site/index.html
ls -ldZ /srv/site                                       # var_t gibi bir tür
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/site(/.*)?'
sudo restorecon -Rv /srv/site
ls -lZ /srv/site                                        # httpd_sys_content_t
```
</details>

**4.** Web sunucusunun ağ üzerinden başka sunuculara bağlanmasına kalıcı olarak izin ver ve kontrol et.

<details>
<summary>Çözüm</summary>

```bash
getsebool httpd_can_network_connect
sudo setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect                     # --> on
```
</details>

**5.** SSH'ın 2222 portunu da dinleyebilmesi için SELinux'a gereken tanımı ekle ve doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo semanage port -l | grep ssh_port_t
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -l | grep ssh_port_t                 # 2222, 22
```

Ardından `sshd_config`'e `Port 2222` eklenir, Ubuntu 24.04'te `sudo systemctl daemon-reload` ve `sudo systemctl restart ssh.socket` yapılır.
</details>

---

Önceki konu: [Kernel Parametreleri](09-kernel-parametreleri.md) · Sonraki konu: [Container'lar](11-containerlar.md)
