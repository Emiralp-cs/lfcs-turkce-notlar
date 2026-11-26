# 03 - İşletim ve Dağıtım: Alıştırmalar

İşletim ve Dağıtım bölümündeki 13 konunun bütün alıştırmaları ve çözümleri, toplam **65 soru**. Sorular konu sırasıyla diziliyor.

Çözümler kapalı gelir. Önce kendi sanal makinende dene, sonra **Çözüm**'e tıklayıp karşılaştır. Takıldığın bir konuyu tekrar okumak için başlığın altındaki bağlantıyı kullan.

| # | Konu | Soru |
|---|---|---|
| 01 | [Açma, Kapatma ve Target'lar](../03-isletim-ve-dagitim/01-acma-kapatma-ve-targetlar.md) | 5 |
| 02 | [Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md) | 5 |
| 03 | [Süreçler](../03-isletim-ve-dagitim/03-surecler.md) | 5 |
| 04 | [Log Dosyaları](../03-isletim-ve-dagitim/04-log-dosyalari.md) | 5 |
| 05 | [Zamanlanmış Görevler](../03-isletim-ve-dagitim/05-zamanlanmis-gorevler.md) | 5 |
| 06 | [Paket Yönetimi](../03-isletim-ve-dagitim/06-paket-yonetimi.md) | 5 |
| 07 | [Paket Depoları](../03-isletim-ve-dagitim/07-paket-depolari.md) | 5 |
| 08 | [Kaynaktan Yazılım Kurmak](../03-isletim-ve-dagitim/08-kaynaktan-kurma.md) | 5 |
| 09 | [Kernel Parametreleri](../03-isletim-ve-dagitim/09-kernel-parametreleri.md) | 5 |
| 10 | [SELinux](../03-isletim-ve-dagitim/10-selinux.md) | 5 |
| 11 | [Container'lar](../03-isletim-ve-dagitim/11-containerlar.md) | 5 |
| 12 | [Sanal Makineler](../03-isletim-ve-dagitim/12-sanal-makineler.md) | 5 |
| 13 | [Bash Script Temelleri](../03-isletim-ve-dagitim/13-bash-script.md) | 5 |

---

## 01 - Açma, Kapatma ve Target'lar

Konu notu: [Açma, Kapatma ve Target'lar](../03-isletim-ve-dagitim/01-acma-kapatma-ve-targetlar.md)

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

## 02 - Arızadan Kurtarma

Konu notu: [Arızadan Kurtarma](../03-isletim-ve-dagitim/02-arizadan-kurtarma.md)

Alıştırmalar için gerçek disklerine dokunmamak adına bir test diski oluştur:

```bash
dd if=/dev/zero of=~/test.img bs=1M count=300
sudo losetup -f --show ~/test.img      # çıktıdaki cihazı (ör. /dev/loop0) kullan
sudo mkfs.ext4 /dev/loop0
```

**1.** Test diskini `/mnt`'e bağla, içine bir dosya yaz, sonra dosya sistemini **değiştirmeden** kontrol et. Bağlıyken ne oluyor?

<details>
<summary>Çözüm</summary>

```bash
sudo mount /dev/loop0 /mnt
echo deneme | sudo tee /mnt/dosya.txt
sudo e2fsck -n /dev/loop0      # Warning: /dev/loop0 is mounted.
sudo umount /mnt
sudo e2fsck -f -n /dev/loop0
```
</details>

**2.** Test diskinin superblock yedeklerinin nerede olduğunu, diske hiçbir şey yazmadan bul.

<details>
<summary>Çözüm</summary>

```bash
sudo dumpe2fs /dev/loop0 | grep -i 'superblock at'
sudo mke2fs -n /dev/loop0 | grep -A1 -i backups
```
</details>

**3.** Test diskinin asıl superblock'unu boz, bağlanmadığını gör, yedekten onar ve dosyanın yerinde olduğunu kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo dd if=/dev/zero of=/dev/loop0 bs=1024 seek=1 count=1 conv=notrunc
sudo mount /dev/loop0 /mnt          # bad superblock
sudo e2fsck -f -y -b 32768 /dev/loop0
sudo mount /dev/loop0 /mnt
cat /mnt/dosya.txt
sudo umount /mnt
```
</details>

**4.** Aşağıdaki satırları içeren bir test dosyası oluştur ve hataları açılıştan önce bul:

```
UUID=11111111-2222-3333-4444-555555555555 /arsiv ext4 defaults 0 2
/dev/loop0 /mnt ext4 defaults 0 2
```

<details>
<summary>Çözüm</summary>

```bash
findmnt --verify --tab-file test.fstab
```

İlk satırda UUID bulunamadığı ve `/arsiv` dizini olmadığı için hata verir. Bu satır gerçek fstab'da olsaydı ve `nofail` olmasaydı sistem emergency moduna düşerdi.
</details>

**5.** Root parolasını unuttuğun bir Ubuntu sunucusunda yapacağın adımları sırayla yaz. (Kendi sanal makinende dene.)

<details>
<summary>Çözüm</summary>

1. Açılışta Shift/Esc ile GRUB menüsünü aç.
2. Normal satırda `e`'ye bas.
3. `linux` satırının sonuna `init=/bin/bash` ekle, Ctrl+X.
4. `mount -o remount,rw /`
5. `passwd root`
6. `sync` ve `exec /sbin/init` (veya `reboot -f`).
</details>

Bitirince: `sudo losetup -d /dev/loop0 && rm ~/test.img`

---

## 03 - Süreçler

Konu notu: [Süreçler](../03-isletim-ve-dagitim/03-surecler.md)

**1.** Arka planda `sleep 1000` ve `sleep 2000` başlat. İkisinin PID'lerini, komut satırlarıyla birlikte tek komutta bul.

<details>
<summary>Çözüm</summary>

```bash
sleep 1000 &
sleep 2000 &
pgrep -a sleep
```
</details>

**2.** `sleep 1000` sürecinin nice değerini 12 yap. Sonra normal kullanıcıyla 5'e indirmeyi dene, ne oluyor? `sudo` ile tekrar dene.

<details>
<summary>Çözüm</summary>

```bash
renice -n 12 -p $(pgrep -f 'sleep 1000')
renice -n 5 -p $(pgrep -f 'sleep 1000')        # Permission denied
sudo renice -n 5 -p $(pgrep -f 'sleep 1000')
ps -o pid,ni,cmd -p $(pgrep -f 'sleep 1000')
```
</details>

**3.** `sleep 2000` sürecini durdur, durumunun `T` olduğunu gör, sonra devam ettir.

<details>
<summary>Çözüm</summary>

```bash
P=$(pgrep -f 'sleep 2000')
kill -STOP $P
ps -o pid,stat,cmd -p $P      # T
kill -CONT $P
ps -o pid,stat,cmd -p $P      # S
```
</details>

**4.** Terminalde `sleep 3000` başlat (sonuna `&` koymadan). Ctrl+Z ile durdur, arka planda devam ettir, `jobs` ile kontrol et, sonra ön plana alıp Ctrl+C ile bitir.

<details>
<summary>Çözüm</summary>

```bash
sleep 3000
# Ctrl+Z
bg
jobs           # [1]+  Running   sleep 3000 &
fg %1
# Ctrl+C
```
</details>

**5.** Kendi kullanıcının bütün `sleep` süreçlerini önce listele, sonra hepsine `SIGTERM` gönder ve kalmadığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
pgrep -a -u $USER sleep
pkill -u $USER sleep
pgrep -u $USER sleep || echo "kalmadi"
```
</details>

---

## 04 - Log Dosyaları

Konu notu: [Log Dosyaları](../03-isletim-ve-dagitim/04-log-dosyalari.md)

**1.** `logger` ile `deneme` etiketli üç kayıt yaz: biri `err`, biri `warning`, biri `info` öncelikli. Sonra sadece warning ve daha önemli olanları, ardından sadece info olanı göster.

<details>
<summary>Çözüm</summary>

```bash
logger -t deneme -p user.err "veritabani baglantisi koptu"
logger -t deneme -p user.warning "disk %85 dolu"
logger -t deneme -p user.info "yedek tamamlandi"

journalctl -t deneme -p warning
journalctl -t deneme -p info..info
```
</details>

**2.** `deneme` etiketli kayıtlardan, mesajı **küçük** "d" harfiyle başlayanları bul. Büyük "D" ile başlayan bir kayıt da yaz ve sonuca karışmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
logger -t deneme "Disk kontrolu basladi"
journalctl -t deneme -g '^d'                          # iki kayıt: disk... ve Disk...
journalctl -t deneme --case-sensitive=true -g '^d'    # sadece disk...
```
</details>

**3.** Bu açılıştan beri `err` ve daha önemli seviyedeki bütün logları, sayfalayıcı açmadan, sadece mesaj metniyle `/tmp/hatalar.txt` dosyasına kaydet.

<details>
<summary>Çözüm</summary>

```bash
sudo journalctl -b -p err --no-pager -o cat > /tmp/hatalar.txt
wc -l /tmp/hatalar.txt
```
</details>

**4.** `sudo` programının son 1 saatteki loglarını göster. Programın yolunu bilmediğini varsay.

<details>
<summary>Çözüm</summary>

```bash
which sudo
sudo journalctl /usr/bin/sudo --since "1 hour ago"
```

`journalctl _COMM=sudo --since "1 hour ago"` de olur.
</details>

**5.** `/var/log` altında "Failed password" geçen bütün kayıtları dosya adlarıyla bul. Journal'da da aynısını `ssh` servisi için ara.

<details>
<summary>Çözüm</summary>

```bash
sudo grep -r "Failed password" /var/log/ 2>/dev/null
sudo journalctl -u ssh -g "Failed password"
```

İkinci komut harfe duyarlıdır, çünkü desende büyük harf var.
</details>

---

## 05 - Zamanlanmış Görevler

Konu notu: [Zamanlanmış Görevler](../03-isletim-ve-dagitim/05-zamanlanmis-gorevler.md)

**1.** `ayse` kullanıcısı için, her hafta içi saat 18:15'te `/home/ayse/rapor` dizinini `/tmp/rapor.tar.gz` olarak arşivleyen bir cron işi kur. Kurulduğunu kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo crontab -u ayse -e
# 15 18 * * 1-5 /usr/bin/tar -czf /tmp/rapor.tar.gz /home/ayse/rapor

sudo crontab -u ayse -l
```
</details>

**2.** Root olarak, her 10 dakikada bir `/usr/bin/df -h /` çıktısını `/var/log/disk.log` dosyasına **ekleyen** bir sistem işi tanımla. `/etc/crontab`'ı değiştirme.

<details>
<summary>Çözüm</summary>

```bash
echo '*/10 * * * * root /usr/bin/df -h / >> /var/log/disk.log 2>&1' | sudo tee /etc/cron.d/disk-izleme
```

`/etc/cron.d/` altında 6. alan kullanıcı adıdır.
</details>

**3.** `/etc/cron.daily/` altına `temizle.sh` adında bir script koymak istiyorsun. Çalışacağından nasıl emin olursun?

<details>
<summary>Çözüm</summary>

```bash
sudo cp temizle.sh /etc/cron.daily/temizle      # nokta olmadan
sudo chmod +x /etc/cron.daily/temizle
run-parts --test /etc/cron.daily                # listede görünmeli
```
</details>

**4.** Bugün saat 22:00'de `/usr/sbin/reboot` çalıştıracak tek seferlik bir iş oluştur, bekleyen işlerde gör, içeriğini kontrol et, sonra iptal et.

<details>
<summary>Çözüm</summary>

```bash
echo '/usr/sbin/reboot' | sudo at 22:00
sudo atq
sudo at -c NUMARA | tail -2
sudo atrm NUMARA
```
</details>

**5.** `/usr/local/bin/temizle` script'ini her gün 03:00'te çalıştıran ve sistem o saatte kapalıysa açılınca çalıştıran bir systemd timer kur. Bir sonraki çalışma zamanını göster.

<details>
<summary>Çözüm</summary>

```bash
sudo tee /etc/systemd/system/temizle.service > /dev/null <<'EOF'
[Unit]
Description=Gunluk temizlik

[Service]
Type=oneshot
ExecStart=/usr/local/bin/temizle
EOF

sudo tee /etc/systemd/system/temizle.timer > /dev/null <<'EOF'
[Unit]
Description=Gunluk temizlik zamanlayicisi

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now temizle.timer
systemctl list-timers temizle.timer
```
</details>

---

## 06 - Paket Yönetimi

Konu notu: [Paket Yönetimi](../03-isletim-ve-dagitim/06-paket-yonetimi.md)

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

## 07 - Paket Depoları

Konu notu: [Paket Depoları](../03-isletim-ve-dagitim/07-paket-depolari.md)

**1.** Sistemin sürüm kod adını ve hangi component'lerin etkin olduğunu bul.

<details>
<summary>Çözüm</summary>

```bash
lsb_release -cs                                  # veya: grep CODENAME /etc/os-release
grep -E '^(Suites|Components)' /etc/apt/sources.list.d/ubuntu.sources
```
</details>

**2.** Sistemdeki bütün depo tanım dosyalarını ve hangi anahtar dosyalarını kullandıklarını listele.

<details>
<summary>Çözüm</summary>

```bash
ls /etc/apt/sources.list.d/
grep -ri 'signed-by' /etc/apt/sources.list.d/
```
</details>

**3.** Aşağıdaki tek satırlık depo tanımını deb822 (`.sources`) biçimine çevir:

```
deb [signed-by=/etc/apt/keyrings/pg.asc] https://apt.postgresql.org/pub/repos/apt noble-pgdg main
```

<details>
<summary>Çözüm</summary>

```
Types: deb
URIs: https://apt.postgresql.org/pub/repos/apt
Suites: noble-pgdg
Components: main
Signed-By: /etc/apt/keyrings/pg.asc
```

Bu içerik `/etc/apt/sources.list.d/pgdg.sources` adıyla kaydedilir.
</details>

**4.** Elinde `anahtar.asc` adında ASCII bir anahtar var, ama depo tanımı `signed-by=/etc/apt/keyrings/ornek.gpg` istiyor. Anahtarı doğru biçimde yerleştir.

<details>
<summary>Çözüm</summary>

```bash
sudo gpg --dearmor -o /etc/apt/keyrings/ornek.gpg anahtar.asc
file /etc/apt/keyrings/ornek.gpg       # OpenPGP Public Key
```

Alternatif: anahtarı `.asc` olarak kopyalayıp depo tanımındaki `signed-by` yolunu `.asc` yap.
</details>

**5.** `apt update` sırasında bir depo için `NO_PUBKEY` hatası alıyorsun. Ne kontrol edersin?

<details>
<summary>Çözüm</summary>

1. Depo dosyasında `Signed-By` (veya `signed-by=`) var mı ve yolu doğru mu?
2. O yolda anahtar dosyası var mı, herkes okuyabiliyor mu? (`ls -l /etc/apt/keyrings/`)
3. Dosya ASCII ise adı `.asc` ile mi bitiyor, ikiliyse `.gpg` ile mi?
4. Anahtar, deponun güncel anahtarı mı? Depo anahtarını değiştirmiş olabilir, yazılımın belgesinden yenisini indir.
</details>

---

## 08 - Kaynaktan Yazılım Kurmak

Konu notu: [Kaynaktan Yazılım Kurmak](../03-isletim-ve-dagitim/08-kaynaktan-kurma.md)

**1.** htop'u derlemek için gereken araçları ve kütüphaneyi kur. `libncursesw5-dev` paketini kurmaya çalışınca ne olduğuna bak.

<details>
<summary>Çözüm</summary>

```bash
sudo apt install build-essential autoconf automake autotools-dev
apt-cache policy libncursesw5-dev                   # Candidate: (none)
apt-cache showpkg libncursesw5-dev | grep -A2 'Reverse Provides'
sudo apt install libncurses-dev
```
</details>

**2.** htop'un kodunu GitHub'dan indir, `/opt/htop` altına kurulacak şekilde yapılandır ve derle. Kurmadan önce sürümünü kontrol et.

<details>
<summary>Çözüm</summary>

```bash
git clone https://github.com/htop-dev/htop.git
cd htop
./autogen.sh
./configure --prefix=/opt/htop
make -j$(nproc)
./htop --version
```
</details>

**3.** 2. alıştırmadaki derlemeyi kur. Programın nereye kurulduğunu ve dpkg'nin onu tanıyıp tanımadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo make install
ls /opt/htop/bin/
dpkg -S /opt/htop/bin/htop        # no path found
```

`/opt/htop/bin` PATH'te olmadığı için tam yolla çalıştırılır: `/opt/htop/bin/htop`.
</details>

**4.** `./configure --enable-hwloc` çalıştır ve hata mesajından hangi paketin eksik olduğunu bul. Paketi kurup `configure`'un geçtiğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
./configure --prefix=/opt/htop --enable-hwloc     # cannot find required library libhwloc
apt search --names-only hwloc
sudo apt install libhwloc-dev
./configure --prefix=/opt/htop --enable-hwloc
```
</details>

**5.** Kurduğun htop'u kaldır ve sistemde kalmadığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
cd htop
sudo make uninstall
ls /opt/htop/bin/          # boş
sudo rm -r /opt/htop       # boş kalan dizinleri temizle
```
</details>

---

## 09 - Kernel Parametreleri

Konu notu: [Kernel Parametreleri](../03-isletim-ve-dagitim/09-kernel-parametreleri.md)

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

## 10 - SELinux

Konu notu: [SELinux](../03-isletim-ve-dagitim/10-selinux.md)

> Bu alıştırmalar SELinux'un etkin olduğu bir sanal makine gerektirir (Rocky Linux, Fedora veya konu notunun 4. bölümündeki gibi hazırlanmış bir Ubuntu).

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

## 11 - Container'lar

Konu notu: [Container'lar](../03-isletim-ve-dagitim/11-containerlar.md)

> Alıştırmalar için Docker Hub erişimi olan bir makine gerekir.

**1.** `nginx` image'ının `1.27` etiketli sürümünü indir ve yerel image'ları listele.

<details>
<summary>Çözüm</summary>

```bash
docker pull nginx:1.27
docker images
```
</details>

**2.** Bu image'dan `web1` adında, arka planda çalışan, sunucunun 8080 portunu container'ın 80 portuna bağlayan bir container başlat. Çalıştığını ve sayfanın geldiğini doğrula.

<details>
<summary>Çözüm</summary>

```bash
docker run -d --name web1 -p 8080:80 nginx:1.27
docker ps
curl http://localhost:8080
```
</details>

**3.** `web1`'i durdur, durmuş container'lar arasında gör, sonra yeni bir container oluşturmadan tekrar başlat.

<details>
<summary>Çözüm</summary>

```bash
docker stop web1
docker ps -a
docker start web1
```
</details>

**4.** Sunucu yeniden başladığında da açılan, `/srv/site` dizinini nginx'in içerik dizinine (`/usr/share/nginx/html`) salt okunur bağlayan, 8090 portunda yayın yapan `web2` container'ını oluştur.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /srv/site
echo '<h1>web2</h1>' | sudo tee /srv/site/index.html
docker run -d --name web2 --restart unless-stopped -p 8090:80 \
  -v /srv/site:/usr/share/nginx/html:ro nginx:1.27
curl http://localhost:8090
```
</details>

**5.** Kendi `index.html`'ini içeren `benim-nginx:1.0` image'ını bir Dockerfile ile oluştur ve çalıştır. Sonra bütün container'ları ve bu image'ı temizle.

<details>
<summary>Çözüm</summary>

```bash
mkdir benim-nginx && cd benim-nginx
echo '<h1>Merhaba</h1>' > index.html
cat > Dockerfile <<'EOF'
FROM nginx:1.27
COPY index.html /usr/share/nginx/html/index.html
EOF
docker build -t benim-nginx:1.0 .
docker run -d --name web3 -p 8091:80 benim-nginx:1.0
curl http://localhost:8091

docker rm -f web1 web2 web3
docker rmi benim-nginx:1.0
```
</details>

---

## 12 - Sanal Makineler

Konu notu: [Sanal Makineler](../03-isletim-ve-dagitim/12-sanal-makineler.md)

> Alıştırmalar için KVM destekli bir makine ve internet erişimi gerekir.

**1.** Ubuntu 24.04 minimal bulut imajını ve `SHA256SUMS` dosyasını indir, imajın bozulmadığını doğrula, sanal boyutunu öğren ve 10 GB'a büyüt.

<details>
<summary>Çözüm</summary>

```bash
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/SHA256SUMS
sha256sum -c SHA256SUMS 2>&1 | grep OK
qemu-img info ubuntu-24.04-minimal-cloudimg-amd64.img
qemu-img resize ubuntu-24.04-minimal-cloudimg-amd64.img 10G
```
</details>

**2.** İmajı `/var/lib/libvirt/images/test1.qcow2` olarak kopyala ve 1 GB bellek, 1 işlemciyle, grafik ekransız, rastgele root parolalı `test1` adında bir VM oluştur.

<details>
<summary>Çözüm</summary>

```bash
sudo cp ubuntu-24.04-minimal-cloudimg-amd64.img /var/lib/libvirt/images/test1.qcow2
sudo virt-install --osinfo ubuntu24.04 --name test1 --memory 1024 --vcpus 1 \
  --import --disk /var/lib/libvirt/images/test1.qcow2 --graphics none \
  --cloud-init root-password-generate=on
```
</details>

**3.** `test1`'in sunucu açıldığında otomatik başlamasını sağla ve kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo virsh autostart test1
sudo virsh dominfo test1 | grep Autostart
```
</details>

**4.** `test1`'e 2 işlemci ve 2 GB bellek ver. Değişikliğin geçerli olduğunu doğrula.

<details>
<summary>Çözüm</summary>

```bash
sudo virsh setvcpus test1 2 --config --maximum
sudo virsh setvcpus test1 2 --config
sudo virsh setmaxmem test1 2G --config
sudo virsh setmem test1 2G --config
sudo virsh shutdown test1          # kapanmazsa: sudo virsh destroy test1
sudo virsh start test1
sudo virsh dominfo test1
```
</details>

**5.** `test1`'i diskiyle birlikte tamamen sil ve disk dosyasının kalmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
sudo virsh destroy test1
sudo virsh undefine test1 --remove-all-storage
sudo virsh list --all
ls /var/lib/libvirt/images/
```
</details>

---

## 13 - Bash Script Temelleri

Konu notu: [Bash Script Temelleri](../03-isletim-ve-dagitim/13-bash-script.md)

**1.** Çalıştırıldığında tarih, saat ve sistemin çalışma süresini (`uptime -p`) `/tmp/durum.log` dosyasına **ekleyen** `durum.sh` adında bir script yaz ve çalıştır.

<details>
<summary>Çözüm</summary>

```bash
cat > durum.sh <<'EOF'
#!/bin/bash
echo "$(date '+%F %T') $(uptime -p)" >> /tmp/durum.log
EOF
chmod +x durum.sh
./durum.sh
cat /tmp/durum.log
```
</details>

**2.** Argüman olarak bir kullanıcı adı alan, kullanıcı sistemde varsa "var", yoksa "yok" yazan ve buna göre 0 veya 1 ile çıkan `kullanici-var-mi.sh` yaz.

<details>
<summary>Çözüm</summary>

```bash
cat > kullanici-var-mi.sh <<'EOF'
#!/bin/bash
if id "$1" > /dev/null 2>&1; then
    echo "var"
    exit 0
else
    echo "yok"
    exit 1
fi
EOF
chmod +x kullanici-var-mi.sh
./kullanici-var-mi.sh root; echo $?
./kullanici-var-mi.sh boyleyok; echo $?
```
</details>

**3.** `/etc/apt` dizinini `/tmp/apt-yedek.tar.gz` olarak arşivleyen, dosya zaten varsa önce onu `.eski` uzantısıyla saklayan bir script yaz. Arşivin içeriğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
cat > apt-yedek.sh <<'EOF'
#!/bin/bash
HEDEF=/tmp/apt-yedek.tar.gz
[ -f "$HEDEF" ] && mv "$HEDEF" "$HEDEF.eski"
tar -czf "$HEDEF" /etc/apt 2>/dev/null
EOF
chmod +x apt-yedek.sh
./apt-yedek.sh; ./apt-yedek.sh
ls /tmp/apt-yedek*
tar -tzf /tmp/apt-yedek.tar.gz | head
```
</details>

**4.** `/etc/ssh/sshd_config` içinde yorum olmayan bir `PermitRootLogin` satırı varsa onu yazdıran, yoksa "varsayilan ayar kullaniliyor" yazan bir script yaz.

<details>
<summary>Çözüm</summary>

```bash
cat > root-kontrol.sh <<'EOF'
#!/bin/bash
if grep -q '^PermitRootLogin' /etc/ssh/sshd_config; then
    grep '^PermitRootLogin' /etc/ssh/sshd_config
else
    echo "varsayilan ayar kullaniliyor"
fi
EOF
chmod +x root-kontrol.sh
./root-kontrol.sh
```
</details>

**5.** Bir script'in sözdiziminde hata olup olmadığını çalıştırmadan nasıl kontrol edersin? `then` kelimesini sildiğin bir `if` ile dene.

<details>
<summary>Çözüm</summary>

```bash
printf '#!/bin/bash\nif [ -f /etc/passwd ]\n  echo var\nfi\n' > hatali.sh
bash -n hatali.sh       # syntax error near unexpected token `fi'
```
</details>

---

Önceki bölüm: [Kullanıcılar ve Gruplar](02-kullanicilar-ve-gruplar.md) · Sonraki bölüm: [Depolama](04-depolama.md) · [Alıştırmalar ana sayfası](README.md)
