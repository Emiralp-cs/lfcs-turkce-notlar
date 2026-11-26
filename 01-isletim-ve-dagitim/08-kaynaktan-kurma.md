# Kaynaktan Yazılım Kurmak

> **Resmi sınav maddesi:** Search for, install, validate, and maintain software packages or repositories

Her yazılım Ubuntu depolarında bulunmaz, bulunsa bile ihtiyacın olan sürüm olmayabilir. O zaman yazılımı **kaynak kodundan derleyip** kurarsın. C ile yazılmış Linux yazılımlarının çoğu aynı adımları izler: kodu indir, bağımlılıkları kur, `configure`, `make`, `make install`.

Senaryo: Sunucuya `htop`'un depodakinden daha yeni bir sürümü gerekiyor. GitHub'dan kodunu indirip derleyeceğiz, depodaki sürümle nasıl yan yana durduğuna bakacağız ve kaldıracağız.

> [!NOTE]
> Bütün adımlar Ubuntu 24.04 üzerinde, htop'un GitHub'daki güncel koduyla denenmiştir. Yazılımın sürümüne göre adımlar biraz değişebilir, her zaman projenin kendi `README` veya `INSTALL` dosyasına bak.

## İçindekiler

1. [Genel akış](#1-genel-akış)
2. [Kodu indirmek](#2-kodu-indirmek)
3. [Derleme araçları ve bağımlılıklar](#3-derleme-araçları-ve-bağımlılıklar)
4. [configure betiğini üretmek: autogen.sh](#4-configure-betiğini-üretmek-autogensh)
5. [Yapılandırmak: ./configure](#5-yapılandırmak-configure)
6. [Derlemek: make](#6-derlemek-make)
7. [Kurmak: make install](#7-kurmak-make-install)
8. [Depodaki sürümle çakışma](#8-depodaki-sürümle-çakışma)
9. [Kaldırmak: make uninstall](#9-kaldırmak-make-uninstall)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Genel akış

```
git clone / tar xf   →   apt install ...-dev   →   ./autogen.sh   →   ./configure   →   make   →   sudo make install
  kodu indir            bağımlılıkları kur         configure'u     ayarlar,          derle       sisteme kopyala
                                                    üret (gerekirse) kontroller
```

| Adım | Root gerekir mi? | Ne üretir? |
|---|---|---|
| `./autogen.sh` | Hayır | `configure` betiği |
| `./configure` | Hayır | `Makefile` |
| `make` | Hayır | Çalıştırılabilir program, kaynak dizininde |
| `make install` | Evet (`/usr/local`'e yazacaksa) | Programı sisteme kopyalar |

Sadece son adım `sudo` ister. Derlemeyi root olarak yapma.

---

## 2. Kodu indirmek

Projeler kodu genelde iki şekilde dağıtır:

```bash
git clone https://github.com/htop-dev/htop.git     # en güncel geliştirme kodu
cd htop
```

veya sürüm arşivi olarak:

```bash
wget https://.../htop-3.3.0.tar.xz
tar -xf htop-3.3.0.tar.xz
cd htop-3.3.0
```

`git` kullanımı: [Git ile Sürüm Kontrolü](../04-temel-komutlar/12-git.md). Arşivi açmak: [Arşivleme ve Sıkıştırma](../04-temel-komutlar/08-arsivleme-ve-sikistirma.md).

İlk iş `README` ve varsa `INSTALL` dosyasını okumaktır. Hangi bağımlılıkların gerektiği ve özel adımlar orada yazar.

---

## 3. Derleme araçları ve bağımlılıklar

```bash
sudo apt install build-essential autoconf automake autotools-dev libncurses-dev
```

| Paket | Ne için? |
|---|---|
| `build-essential` | `gcc`, `make` ve temel derleme araçları |
| `autoconf`, `automake`, `autotools-dev` | `configure` betiğini üretmek için (git'ten indirilen kodda gerekir) |
| `libncurses-dev` | htop'un ekran çizmek için kullandığı kütüphanenin geliştirme dosyaları |

### -dev paketleri

Çalışan bir programın kütüphaneye ihtiyacı vardır (`libncurses6`). Onu **derlemek** için ise kütüphanenin başlık dosyaları ve bağlantı dosyaları gerekir, bunlar `-dev` ile biten ayrı pakettedir (`libncurses-dev`). Derleme sırasında `fatal error: curses.h: No such file or directory` gibi bir hata, ilgili `-dev` paketinin eksik olduğunu gösterir.

> [!NOTE]
> Eski belgelerde htop için `libncursesw5-dev` paketi geçer. Ubuntu 24.04'te bu adda gerçek bir paket yok (`apt-cache policy libncursesw5-dev` → `Candidate: (none)`). Bu bir **sanal paket**: `apt install libncursesw5-dev` yazınca apt onun yerine `libncurses-dev`'i kurdu. Çalışır, ama doğrudan `libncurses-dev` yazmak daha açıktır. Bir sanal paketi kimin sağladığını görmek için: `apt-cache showpkg libncursesw5-dev`, `Reverse Provides` bölümü.

---

## 4. configure betiğini üretmek: autogen.sh

Sürüm arşivleri (`.tar.gz`) genelde hazır bir `configure` betiğiyle gelir. Git'ten indirilen kodda ise `configure` yoktur, önce üretilmesi gerekir:

```bash
./autogen.sh
```
```
Makefile.am: installing 'build-aux/depcomp'
autoreconf: Leaving directory '.'
```

Bazı projelerde bu betiğin adı `bootstrap` olabilir, bazıları doğrudan `autoreconf -fi` ister. Dizinde zaten `configure` varsa bu adımı atla.

---

## 5. Yapılandırmak: ./configure

`configure`, sistemde gereken her şeyin olup olmadığını kontrol eder ve derleme ayarlarını içeren `Makefile`'ı oluşturur:

```bash
./configure --help        # bütün seçenekler
./configure               # varsayılan ayarlarla
```

Çıktı, kontrol edilen her şeyi satır satır yazar ve sonunda bir özet verir:

```
  hwloc:                     no
  debug:                     no
  static:                    no
```

### En önemli seçenek: --prefix

Programın nereye kurulacağını belirler. Varsayılan `/usr/local`:

| Prefix | Program nereye gider? |
|---|---|
| (varsayılan) `/usr/local` | `/usr/local/bin/htop` |
| `--prefix=/opt/htop` | `/opt/htop/bin/htop` |
| `--prefix=$HOME/.local` | `~/.local/bin/htop`, `sudo` gerekmez |

`/usr` prefix'ini kullanma: paket yöneticisinin dosyalarının üzerine yazar.

### Bir bağımlılık eksikse

`configure` eksik bir şey bulursa durur ve neyin eksik olduğunu yazar. Kurulu olmayan bir özelliği istediğimizde:

```bash
./configure --enable-hwloc
```
```
checking for hwloc_get_proc_cpubind in -lhwloc... no
configure: error: cannot find required library libhwloc
```

Çözüm, eksik kütüphanenin `-dev` paketini kurmak (`apt search hwloc` → `libhwloc-dev`) ve `configure`'u tekrar çalıştırmaktır. Hata yoksa son satırlarda bir hata mesajı olmaz ve dizinde `Makefile` oluşur.

---

## 6. Derlemek: make

```bash
make
make -j$(nproc)           # bütün çekirdeklerle, daha hızlı
```

Derleme bitince program kaynak dizininde durur ve kurulmadan çalıştırılabilir:

```bash
ls -l htop
./htop --version
```
```
-rwxr-xr-x 1 root root 1741176 Nov 26 09:11 htop
htop 3.6.0-dev-b9dc8aa
```

`make` hata verirse çıktının sonundaki ilk `error:` satırına bak. Genelde eksik bir başlık dosyasıdır, yani bir `-dev` paketi eksik. Kurduktan sonra baştan derlemek için `make clean && make`.

---

## 7. Kurmak: make install

```bash
sudo make install
```
```
 /usr/bin/mkdir -p '/usr/local/bin'
  /usr/bin/install -c htop '/usr/local/bin'
 /usr/bin/mkdir -p '/usr/local/share/icons/hicolor/scalable/apps'
```

Program `--prefix` ile belirlenen yere kopyalandı. `/usr/local/bin` zaten PATH'te olduğu için artık her yerden `htop` yazılabilir.

> [!IMPORTANT]
> `make install` ile kurulan dosyaları **dpkg bilmez**:

```bash
dpkg -S /usr/local/bin/htop
```
```
dpkg-query: no path found matching pattern /usr/local/bin/htop
```

Bu yüzden `apt remove` ile kaldırılamaz, `apt upgrade` ile güncellenmez, `dpkg -V` ile doğrulanamaz. Güvenlik güncellemelerini takip etmek ve yeni sürümü derlemek senin sorumluluğundadır. Mümkünse önce depodaki paketi tercih et.

---

## 8. Depodaki sürümle çakışma

Sistemde apt ile kurulmuş htop da vardı. `make install`'dan sonra:

```bash
type -a htop
```
```
htop is /usr/local/bin/htop
htop is /usr/bin/htop
```

İkisi de duruyor, ama PATH'te `/usr/local/bin` `/usr/bin`'den **önce** geldiği için `htop` yazınca derlediğin sürüm çalışır. Depodaki sürümü çalıştırmak için tam yolunu yaz: `/usr/bin/htop`.

Kurulumdan sonra kabuk eski yolu hatırlıyorsa (`htop` hala eskisini açıyorsa) `hash -r` ile önbelleği temizle.

---

## 9. Kaldırmak: make uninstall

Çoğu proje, kurduğu dosyaları silen bir `uninstall` hedefi sunar:

```bash
cd htop                  # kaynak dizini
sudo make uninstall
```
```
 ( cd '/usr/local/share/icons/hicolor/scalable/apps' && rm -f htop.svg )
```

Denemede `/usr/local/bin/htop` silindi.

`make uninstall` için iki şart var:

1. **Kaynak dizini duruyor olmalı.** Derledikten sonra dizini sildiysen, kodu tekrar indirip aynı ayarlarla `configure` etmen gerekir.
2. **`configure` aynı seçeneklerle çalıştırılmış olmalı.** `--prefix=/opt/htop` ile kurduysan, `uninstall` de o prefix'le yapılandırılmış dizinde çalıştırılmalı, yoksa yanlış yerdeki dosyaları silmeye çalışır.

Bu yüzden kaynaktan kurduğun yazılımları `--prefix=/opt/yazilim-adi` ile ayrı bir dizine kurmak, kaldırmayı kolaylaştırır: dizini silmek yeterli olur.

---

## 10. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `git clone URL` / `tar -xf arsiv.tar.gz` | Kodu indir |
| `sudo apt install build-essential` | Derleyici ve make |
| `sudo apt install paket-dev` | Kütüphanenin geliştirme dosyaları |
| `apt-cache showpkg paket` | Sanal paketi kimin sağladığı |
| `./autogen.sh` | `configure`'u üret (git kodunda) |
| `./configure --help` | Seçenekler |
| `./configure --prefix=/opt/ad` | Kurulum yerini belirle |
| `make -j$(nproc)` | Derle |
| `./program` | Kurmadan dene |
| `sudo make install` | Sisteme kur |
| `sudo make uninstall` | Kaldır (kaynak dizininde) |
| `make clean` | Derleme çıktılarını sil |
| `type -a program` | Aynı adlı bütün programlar ve sırası |
| `hash -r` | Kabuğun komut yolu önbelleğini temizle |

---

## 11. Sık yapılan hatalar

- **`README`/`INSTALL` dosyasını okumadan başlamak.**
- **Bütün süreci root olarak yapmak.** Sadece `make install` `sudo` ister.
- **`configure` hata verdiği halde `make`'e geçmek.** `Makefile` oluşmaz veya eksik olur.
- **Eksik `-dev` paketi yerine kütüphanenin kendisini kurmak.** Derleme için `-dev` gerekir.
- **`--prefix=/usr` kullanmak.** Paket yöneticisinin dosyalarıyla çakışır.
- **Kaynak dizinini silip sonra `make uninstall`'a ihtiyaç duymak.**
- **`make install` ile kurulanın `apt` ile güncelleneceğini sanmak.**
- **Depodaki ve derlenmiş sürüm yan yanayken hangisinin çalıştığını kontrol etmemek.** `type -a`.

---

## 12. Alıştırmalar

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

Önceki konu: [Paket Depoları](07-paket-depolari.md) · Sonraki konu: [Kernel Parametreleri](09-kernel-parametreleri.md)
