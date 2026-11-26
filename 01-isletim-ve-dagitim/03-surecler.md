# Süreçler

> **Resmi sınav maddesi:** Diagnose, identify, manage, and troubleshoot processes and services

Çalışan her program bir **süreçtir** (process). Her sürecin bir numarası (PID), sahibi, önceliği ve durumu vardır. Bu dosya süreçleri bulmayı, incelemeyi, önceliklerini değiştirmeyi, sinyal göndererek durdurmayı ve arka planda çalıştırmayı anlatıyor. Süreçlerin servis olarak yönetilmesi ayrı bir konu: [Servis Oluşturma ve Sorun Giderme](../04-temel-komutlar/13-servis-olusturma-ve-sorun-giderme.md).

Senaryo: Sunucuda `ayse` kullanıcısının başlattığı uzun süren işler var. Hangi süreçlerin onun olduğunu, nerede çalıştıklarını, hangi dosyaları kullandıklarını bulacağız. Birinin önceliğini düşürüp diğerini durduracağız.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 üzerinde denenmiştir. PID numaraları her çalıştırmada farklıdır.

## İçindekiler

1. [Süreçleri listelemek: ps](#1-süreçleri-listelemek-ps)
2. [Süreç durumları](#2-süreç-durumları)
3. [Süreç bulmak: pgrep ve pidof](#3-süreç-bulmak-pgrep-ve-pidof)
4. [Bir süreci incelemek ve doğrulamak](#4-bir-süreci-incelemek-ve-doğrulamak)
5. [Canlı izlemek: top](#5-canlı-izlemek-top)
6. [Öncelik: nice ve renice](#6-öncelik-nice-ve-renice)
7. [Sinyaller: kill ve pkill](#7-sinyaller-kill-ve-pkill)
8. [Ön plan ve arka plan: jobs, bg, fg](#8-ön-plan-ve-arka-plan-jobs-bg-fg)
9. [Hangi dosyalar açık: lsof](#9-hangi-dosyalar-açık-lsof)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Süreçleri listelemek: ps

`ps` tek başına sadece o terminalde başlattığın süreçleri gösterir. Bütün sistem için iki yaygın yazım vardır:

```bash
ps aux        # BSD yazımı, tiresiz
ps -ef        # UNIX yazımı
```

```bash
ps aux | head -3
```
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  37124  5724 ?        SLl  08:03   0:01 /sbin/init
root         2  0.0  0.0      0     0 ?        S    08:03   0:00 [kthreadd]
```

| Sütun | Anlamı |
|---|---|
| `USER` | Süreci çalıştıran kullanıcı |
| `PID` | Süreç numarası |
| `%CPU`, `%MEM` | İşlemci ve bellek kullanımı |
| `VSZ` | Ayrılan sanal bellek (KB). Gerçek kullanım değildir |
| `RSS` | Gerçekte RAM'de duran kısım (KB) |
| `TTY` | Bağlı olduğu terminal. `?` terminale bağlı değil (servisler) |
| `STAT` | Durum ([2. bölüm](#2-süreç-durumları)) |
| `START`, `TIME` | Başlama saati, toplam kullandığı işlemci süresi |
| `COMMAND` | Komut. Köşeli parantez içindekiler çekirdek süreçleri |

### İstediğin sütunlar ve sıralama

```bash
ps -u ayse -o pid,ppid,stat,etime,cmd          # ayse'nin süreçleri
ps -eo pid,user,%cpu,%mem,comm --sort=-%mem    # belleğe göre sıralı
ps -ef --forest                                 # süreç ağacı
```
```
  PID  PPID STAT     ELAPSED CMD
12111     1 S          00:00 sleep 500
12112     1 S          00:00 sleep 600
```

`PPID` süreci başlatan üst süreç, `etime` ne zamandır çalıştığı. Ağaç görünümü için `pstree -p` de kullanılabilir.

> [!TIP]
> `ps aux | grep sleep` yazınca `grep` süreci de listede çıkar. Bunu önlemek için `ps aux | grep '[s]leep'` veya doğrudan `pgrep` kullan ([3. bölüm](#3-süreç-bulmak-pgrep-ve-pidof)).

---

## 2. Süreç durumları

`STAT` sütununun ilk harfi:

| Harf | Durum | Anlamı |
|---|---|---|
| `R` | Running | Çalışıyor veya işlemci sırasında bekliyor |
| `S` | Sleeping | Bir olayı bekliyor (çoğu süreç böyledir) |
| `D` | Uninterruptible sleep | Diskten veya ağdan cevap bekliyor, sinyallere cevap vermez |
| `T` | Stopped | Durdurulmuş (Ctrl+Z veya `SIGSTOP`) |
| `Z` | Zombie | Bitmiş ama üst süreci çıkış kodunu henüz almamış |

Sonraki harfler ek bilgidir: `s` oturum lideri, `l` çok iş parçacıklı, `+` ön planda, `<` yüksek öncelikli, `N` düşük öncelikli.

`D` durumunda çok süreç varsa disk veya ağ dosya sistemi (NFS) takılmış olabilir, bunlar `kill -9` ile bile ölmez. Zombi süreçler kaynak tüketmez, sadece süreç tablosunda yer kaplar. Üst süreç çıkış kodunu alınca (veya kapanınca) kaybolurlar.

---

## 3. Süreç bulmak: pgrep ve pidof

```bash
pgrep sleep                  # adı sleep olan süreçlerin PID'leri
pgrep -a -u ayse             # ayse'nin süreçleri, komutlarıyla
pgrep -x sleep -u ayse -l    # adı tam olarak sleep olanlar, adlarıyla
pgrep -f 'sleep 6'           # komut satırının tamamında ara
pidof redis-server           # programın PID'leri
```
```
12111 sleep 500
12112 sleep 600
```

| Seçenek | Anlamı |
|---|---|
| `-a` | Komutun tamamını göster |
| `-l` | Sadece adını göster |
| `-u kullanici` | Belirli bir kullanıcının süreçleri |
| `-x` | Adı tam eşleşsin (`sleep` için `sleepy`'yi alma) |
| `-f` | Sadece adda değil, bütün komut satırında ara |

`pgrep` hiçbir şey bulamazsa çıkış kodu 1 olur, script'lerde "bu program çalışıyor mu?" kontrolü için kullanılır: `pgrep -x nginx > /dev/null || echo "nginx calismiyor"`.

---

## 4. Bir süreci incelemek ve doğrulamak

Bir sürecin gerçekten beklediğin şeyi yaptığını doğrulamak için `/proc/PID/` dizini her bilgiyi içerir:

```bash
P=$(pgrep -f 'sleep 500')
pwdx $P                                          # hangi dizinde çalışıyor
ls -l /proc/$P/cwd /proc/$P/exe                  # çalışma dizini ve programın kendisi
grep -E 'State|VmRSS|Threads' /proc/$P/status
cat /proc/$P/cmdline | tr '\0' ' '; echo         # tam komut satırı
cat /proc/$P/limits                              # limitleri
```
```
12111: /home/ayse
/proc/12111/cwd -> /home/ayse
/proc/12111/exe -> /usr/bin/sleep
State:	S (sleeping)
VmRSS:	    1972 kB
Threads:	1
```

| Soru | Komut |
|---|---|
| Çalışıyor mu? | `pgrep -a ad`, `systemctl is-active servis` |
| Kimin adına? | `ps -o user= -p PID` |
| Hangi dizinde? | `pwdx PID` |
| Hangi dosyaları kullanıyor? | `lsof -p PID` ([9. bölüm](#9-hangi-dosyalar-açık-lsof)) |
| Hangi portu dinliyor? | `ss -tlnp \| grep PID` |
| Ne kadar kaynak kullanıyor? | `ps -o %cpu,%mem,rss -p PID`, `pidstat -p PID 1 3` |
| Ne zamandır çalışıyor? | `ps -o etime= -p PID` |

Sistem genelinde performansı izlemek ve yükün kaynağını bulmak: [Performans İzleme](../04-temel-komutlar/14-performans-izleme.md).

---

## 5. Canlı izlemek: top

`top`, süreçleri birkaç saniyede bir yenileyerek gösterir. Sütunların ve üst bölümün ayrıntılı açıklaması [Performans İzleme](../04-temel-komutlar/14-performans-izleme.md#3-süreçleri-canlı-izlemek-top-ve-htop) dosyasında. `top` içindeyken en sık kullanılan tuşlar:

| Tuş | Ne yapar? |
|---|---|
| `P` / `M` | İşlemciye / belleğe göre sırala |
| `u` | Bir kullanıcının süreçlerini göster |
| `k` | PID ve sinyal sorup süreç öldür |
| `r` | PID sorup nice değerini değiştir |
| `1` | Çekirdekleri ayrı göster |
| `q` | Çık |

---

## 6. Öncelik: nice ve renice

İşlemci için yarışan süreçlerden hangisinin daha çok zaman alacağını **nice değeri** belirler: **-20** (en yüksek öncelik) ile **19** (en düşük) arasında, varsayılan **0**. "Nice" adı buradan gelir: değeri yüksek olan süreç diğerlerine karşı daha "kibar" davranır.

```bash
nice -n 10 ./yedekle.sh            # düşük öncelikle başlat
renice -n 15 -p 12086              # çalışan sürecin önceliğini değiştir
ps -o pid,ni,comm -p 12086         # NI sütunu
```
```
12086 (process ID) old priority 7, new priority 9
  PID  NI PRI COMMAND
12086   9  10 sleep
```

`renice 7 12086` gibi eski yazım da çalışır. `-u kullanici` ile bir kullanıcının bütün süreçleri değiştirilebilir.

### Kim neyi değiştirebilir?

`ayse` kullanıcısıyla denedik:

| İşlem | Sonuç |
|---|---|
| `nice -n 5 sleep 300` | Çalıştı, NI=5 |
| `renice -n 10` (5'ten 10'a, önceliği düşürmek) | Çalıştı |
| `renice -n 2` (10'dan 2'ye, önceliği geri yükseltmek) | `Permission denied` |
| `nice -n -5 true` (negatif değer) | `nice: cannot set niceness: Permission denied` |

Normal kullanıcı kendi süreçlerinin önceliğini **sadece düşürebilir**, bir kez düşürdüğünü bile geri yükseltemez. Negatif değer ve öncelik yükseltmek root gerektirir (`sudo nice -n -5 ...`).

Disk erişiminin önceliği için ayrıca `ionice` var: `ionice -c3 -p PID` süreci sadece disk boştayken çalıştırır.

---

## 7. Sinyaller: kill ve pkill

Bir sürece **sinyal** göndererek ne yapmasını istediğini söylersin. `kill` adına rağmen her zaman öldürmez, sadece sinyal gönderir.

```bash
kill -l          # bütün sinyaller
kill -l 15       # TERM
kill -l TERM     # 15
```

En sık kullanılanlar:

| Numara | Ad | Anlamı | Süreç yakalayabilir mi? |
|---|---|---|---|
| 1 | `SIGHUP` | Terminal kapandı. Birçok servis için "ayarlarını yeniden oku" | Evet |
| 2 | `SIGINT` | Ctrl+C | Evet |
| 9 | `SIGKILL` | Hemen öldür | **Hayır** |
| 15 | `SIGTERM` | Düzgünce kapan (varsayılan) | Evet |
| 18 | `SIGCONT` | Durdurulmuşsa devam et | Hayır |
| 19 | `SIGSTOP` | Durdur | **Hayır** |
| 20 | `SIGTSTP` | Ctrl+Z | Evet |

```bash
kill 12092               # SIGTERM gönder
kill -TERM 12092         # aynısı
kill -15 12092           # aynısı
kill -9 12092            # SIGKILL
kill -STOP 12092         # durdur
kill -CONT 12092         # devam ettir
```

Durdurup devam ettirdiğimiz süreç:

```
  PID STAT COMMAND
12092 T    sleep         # SIGSTOP sonrası
12092 S    sleep         # SIGCONT sonrası
```

### Önce TERM, sonra KILL

`SIGTERM` sürece "kapan" der, süreç dosyalarını kaydedip düzgünce çıkar. Ama süreç bu sinyali görmezden gelebilir. Denemede `SIGTERM`'i yok sayan bir süreç `kill PID`'den sonra çalışmaya devam etti, `kill -9 PID` ile öldü.

Sıra her zaman: önce `kill PID`, birkaç saniye bekle, çalışmaya devam ediyorsa `kill -9 PID`. `-9` süreci temizlik yapma fırsatı vermeden öldürür, yarım yazılmış dosyalar veya kilit dosyaları kalabilir.

### Ada göre sinyal göndermek

```bash
pkill sleep                # adı sleep olan bütün süreçlere SIGTERM
pkill -u ayse sleep        # ayse'nin sleep süreçlerine
pkill -9 -f 'yedekle.sh'   # komut satırında yedekle.sh geçenlere SIGKILL
killall nginx              # adı tam olarak nginx olanlara
```

> [!CAUTION]
> `pkill` eşleşen **her** süreci etkiler. `pkill bash` veya `pkill -9 python` gibi geniş bir desen, senin terminalin dahil, beklemediğin süreçleri de öldürebilir. Önce aynı deseni `pgrep -a` ile çalıştırıp listeye bak, sonra `pkill`.

Servis olarak çalışan bir programı `kill` ile değil, `systemctl stop` / `restart` ile yönet. `Restart=` ayarlı bir servisi `kill` ile öldürürsen systemd onu hemen yeniden başlatır.

---

## 8. Ön plan ve arka plan: jobs, bg, fg

Terminalde başlattığın bir komut bitene kadar terminali meşgul eder. Uzun süren işler arka plana alınabilir:

```bash
sleep 3000 &          # doğrudan arka planda başlat
```

Zaten başlattıysan:

1. **Ctrl+Z** ile durdur (`T` durumuna geçer).
2. `bg` ile arka planda devam ettir.

```bash
jobs                  # bu terminaldeki işler
fg %1                 # 1 numaralı işi ön plana getir
bg %1                 # 1 numaralı işi arka planda devam ettir
kill %1               # 1 numaralı işe sinyal gönder
```

`jobs` çıktısındaki `%1`, `%2` iş numaralarıdır, PID değildir. Sadece o terminalde geçerlidir.

Arka plandaki işler, terminal kapanınca `SIGHUP` alıp kapanır. Terminalden bağımsız çalışmaya devam etmesi için:

```bash
nohup ./uzun-is.sh > is.log 2>&1 &
```

veya zaten çalışan bir iş için `disown %1`. Sürekli çalışması gereken işler için doğru yol bunları servis yapmaktır.

---

## 9. Hangi dosyalar açık: lsof

`lsof` (list open files), süreçlerin açık tuttuğu dosyaları, dizinleri ve ağ bağlantılarını gösterir:

```bash
sudo lsof -p 12111                 # bir sürecin açık dosyaları
sudo lsof /var/log/auth.log        # bu dosyayı kim açık tutuyor
sudo lsof -u ayse                  # bir kullanıcının açık dosyaları
sudo lsof +D /srv/veri             # bir dizin altında açık olanlar
sudo lsof -i :22                   # 22 portunu kullananlar
```
```
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF     NODE NAME
sleep   12111 ayse  cwd    DIR  254,0     4096   812188 /home/ayse
sleep   12111 ayse  rtd    DIR  254,0     4096        2 /
sleep   12111 ayse  txt    REG  254,0    35336  8036541 /usr/bin/sleep
```

`FD` sütunu: `cwd` çalışma dizini, `txt` programın kendisi, `3r` okumak için açılmış 3 numaralı dosya, `4w` yazmak için.

İki sık kullanım:
- Bir disk `umount` sırasında `target is busy` diyorsa, onu kullanan süreci bulmak: `lsof +D /mnt` ([Depolama Performansı](../03-depolama/11-depolama-performansi.md)).
- Silindiği halde yer açmayan dosyalar: `lsof +L1` ([Disk Alanı Sorunları](../04-temel-komutlar/15-disk-alani-sorunlari.md)).

`lsof` yüklü değilse `sudo apt install lsof`. Benzer bir araç: `fuser -vm /mnt`.

> [!TIP]
> Bir komutun adını veya seçeneğini hatırlamıyorsan: `apropos process`, `man -k signal`, `man kill` içinde `/SIGNALS` ile arama. Yardım almanın ayrıntısı: [Yardım Almak](../04-temel-komutlar/01-yardim-almak.md).

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `ps aux`, `ps -ef` | Bütün süreçler |
| `ps -u ayse -o pid,stat,cmd` | Kullanıcının süreçleri, seçilen sütunlar |
| `ps -eo pid,%mem,comm --sort=-%mem` | Belleğe göre sıralı |
| `ps -ef --forest`, `pstree -p` | Süreç ağacı |
| `pgrep -a ad`, `pgrep -u kullanici` | Süreç bul |
| `pidof program` | Programın PID'leri |
| `pwdx PID` | Çalışma dizini |
| `/proc/PID/status`, `/proc/PID/cmdline` | Süreç ayrıntıları |
| `top` | Canlı izle |
| `nice -n 10 komut` | Düşük öncelikle başlat |
| `renice -n 15 -p PID` | Önceliği değiştir |
| `kill PID`, `kill -9 PID` | TERM / KILL gönder |
| `kill -STOP PID`, `kill -CONT PID` | Durdur / devam ettir |
| `kill -l` | Sinyal listesi |
| `pkill -u kullanici ad` | Ada göre sinyal |
| `komut &` | Arka planda başlat |
| Ctrl+Z, `bg`, `fg %1`, `jobs` | İş kontrolü |
| `nohup komut &` | Terminal kapanınca da devam et |
| `lsof -p PID`, `lsof dosya`, `lsof +D dizin` | Açık dosyalar |

---

## 11. Sık yapılan hatalar

- **Hemen `kill -9` kullanmak.** Önce `SIGTERM` ver.
- **`pkill`'i önce `pgrep` ile denemeden kullanmak.**
- **Servisleri `kill` ile durdurmaya çalışmak.** `systemctl stop` kullan, `Restart=` varsa geri gelir.
- **Normal kullanıcıyla negatif nice veya öncelik yükseltmeyi denemek.** Root gerekir.
- **Nice değerinin yüksek olmasını yüksek öncelik sanmak.** Tersi: 19 en düşük öncelik.
- **`ps`'teki `VSZ`'yi bellek kullanımı sanmak.** `RSS`'e bak.
- **`jobs`'taki `%1` ile PID'i karıştırmak.**
- **`D` durumundaki süreçleri `kill -9` ile öldürmeye çalışmak.** Önce beklediği disk veya ağ sorununu çöz.

---

## 12. Alıştırmalar

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

Önceki konu: [Arızadan Kurtarma](02-arizadan-kurtarma.md) · Sonraki konu: [Log Dosyaları](04-log-dosyalari.md)
