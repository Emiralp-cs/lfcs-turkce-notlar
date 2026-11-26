# Paket Depoları

> **Resmi sınav maddesi:** Search for, install, validate, and maintain software packages or repositories

Bir **depo** (repository), paketlerin indirildiği sunucudur. apt hangi depolara bakacağını `/etc/apt/` altındaki dosyalardan okur. Ubuntu'nun kendi depolarında olmayan bir yazılımı (Docker, PostgreSQL'in en yeni sürümü, şirketin kendi paketleri) kurmak için o yazılımın deposunu eklemen gerekir. Bu dosya depo tanımlarını okumayı, yeni depo eklemeyi, imza anahtarlarını ve PPA'ları anlatıyor.

Senaryo: Şirketin kendi araçlarını dağıttığı bir iç depo var. Bunu sunucuya ekleyip oradan paket kuracağız. Ayrıca Docker'ın resmi deposunun nasıl tanımlandığına bakacağız.

> [!NOTE]
> Örnekler Ubuntu 24.04 üzerinde denenmiştir. İç depo senaryosu için aynı makinede, gerçek bir GPG anahtarıyla imzalanmış küçük bir depo oluşturuldu. Deneme ortamının ağ kısıtlaması yüzünden internetteki üçüncü parti depolara (Docker, PPA'lar) bağlanılamadı. Bu kısımlar dosya biçimleri kontrol edilerek yazılmıştır.

## İçindekiler

1. [Depo tanımları nerede?](#1-depo-tanımları-nerede)
2. [Bir depo tanımının parçaları](#2-bir-depo-tanımının-parçaları)
3. [Depo dosyalarının iki biçimi: .sources ve .list](#3-depo-dosyalarının-iki-biçimi-sources-ve-list)
4. [Depo imza anahtarları](#4-depo-imza-anahtarları)
5. [Yeni depo eklemek: adım adım](#5-yeni-depo-eklemek-adım-adım)
6. [Örnek: Docker deposu](#6-örnek-docker-deposu)
7. [PPA](#7-ppa)
8. [Paket hangi depodan gelecek?](#8-paket-hangi-depodan-gelecek)
9. [Depo kaldırmak](#9-depo-kaldırmak)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Depo tanımları nerede?

| Dosya | İçerik |
|---|---|
| `/etc/apt/sources.list.d/ubuntu.sources` | Ubuntu'nun kendi depoları (Ubuntu 24.04'ten itibaren) |
| `/etc/apt/sources.list.d/*.sources`, `*.list` | Eklenen diğer depolar, her biri ayrı dosyada |
| `/etc/apt/sources.list` | Eski sürümlerde ana dosya, 24.04'te genelde boş veya sadece bir açıklama |
| `/etc/apt/keyrings/` | Eklenen depoların imza anahtarları |
| `/usr/share/keyrings/` | Paketlerle gelen anahtarlar (Ubuntu'nun kendi anahtarı) |

Yeni bir depo eklerken mevcut dosyaları düzenlemek yerine `sources.list.d/` altına **ayrı bir dosya** koy. Depoyu kaldırmak da o dosyayı silmek kadar kolay olur.

---

## 2. Bir depo tanımının parçaları

```bash
cat /etc/apt/sources.list.d/ubuntu.sources
```
```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

| Alan | Anlamı |
|---|---|
| `Types` | `deb`: derlenmiş paketler. `deb-src`: kaynak kod paketleri |
| `URIs` | Deponun adresi |
| `Suites` | Dağıtım sürümü ve kanalı |
| `Components` | Depo içindeki bölümler |
| `Signed-By` | Deponun imzasını doğrulayacak anahtar dosyası |

### Suites

`noble`, Ubuntu 24.04'ün kod adıdır (`lsb_release -cs` veya `/etc/os-release` içindeki `VERSION_CODENAME`).

| Suite | İçerik |
|---|---|
| `noble` | Sürüm çıktığındaki paketler |
| `noble-updates` | Hata düzeltmeleri |
| `noble-security` | Güvenlik güncellemeleri |
| `noble-backports` | Yeni Ubuntu sürümlerinden geri taşınan, daha yeni paketler |

### Components

| Component | İçerik |
|---|---|
| `main` | Ubuntu'nun resmi desteklediği açık kaynak yazılımlar |
| `restricted` | Resmi desteklenen ama kapalı kaynak sürücüler |
| `universe` | Topluluğun bakımını yaptığı açık kaynak yazılımlar |
| `multiverse` | Lisans kısıtlaması olan yazılımlar |

Bir paket `apt install` ile bulunamıyorsa, ilk kontrol edilecek şey `universe`'ün etkin olup olmadığıdır.

---

## 3. Depo dosyalarının iki biçimi: .sources ve .list

Aynı depo iki şekilde tanımlanabilir. Ubuntu 24.04 yeni **deb822** biçimini (`.sources`) kullanır, ama eski tek satırlık biçim (`.list`) de çalışır. Üçüncü parti yazılımların kurulum belgelerinin çoğu hala tek satırlık biçimi verir.

**`.sources` (çok satırlı):**

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```

**`.list` (tek satır):**

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable
```

| Parça | `.list`'teki yeri |
|---|---|
| `deb` | Tür |
| `[arch=amd64 signed-by=...]` | Seçenekler, köşeli parantez içinde |
| `https://...` | Adres |
| `noble` | Suite |
| `stable` | Component(ler) |

İkisini denedik, aynı sonucu verdiler. Dosya uzantısı biçimle uyuşmalı: deb822 içeriği `.list` dosyasına yazarsan apt onu okuyamaz.

---

## 4. Depo imza anahtarları

Depolar paket listelerini bir GPG anahtarıyla imzalar. apt, `Signed-By` ile gösterilen açık anahtarla imzayı doğrular. Böylece biri araya girip sahte paket gönderemez.

Kendi depomuzu önce anahtarsız ekledik:

```
Err:1 file:/srv/depo ./ InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 5F232E045761B7AA
W: GPG error: file:/srv/depo ./ InRelease: The following signatures couldn't be verified ...
```

apt depodan paket listesini almayı reddetti. Anahtarı ekleyip `Signed-By` ile gösterince sorun çözüldü.

### .asc ve .gpg

Anahtar iki biçimde olabilir:

| Biçim | İçerik | Dosya adı |
|---|---|---|
| ASCII (armored) | `-----BEGIN PGP PUBLIC KEY BLOCK-----` ile başlayan metin | `.asc` ile bitmeli |
| İkili (binary) | Okunamaz ikili veri | `.gpg` ile bitmeli |

Üç durumu denedik:

| Anahtar dosyası | Sonuç |
|---|---|
| ASCII anahtar, `sirket.asc` | Çalıştı |
| Aynı ASCII anahtar, `sirket-yanlis.gpg` | **Hata** |
| `gpg --dearmor` ile ikiliye çevrilmiş, `sirket.gpg` | Çalıştı |

Yani Ubuntu 24.04'teki apt, ASCII anahtarı doğrudan kabul eder, yeter ki dosyanın adı `.asc` ile bitsin. Birçok kurulum belgesindeki `gpg --dearmor` adımı, anahtarı `.gpg` adıyla kaydetmek istediğinde gereklidir:

```bash
curl -fsSL https://ornek.com/anahtar.asc | sudo gpg --dearmor -o /etc/apt/keyrings/ornek.gpg
```

Eski kaynaklarda görülen `apt-key add` komutu **kullanımdan kaldırıldı**. Anahtarı bütün depolar için geçerli hale getirdiği için güvensizdir.

---

## 5. Yeni depo eklemek: adım adım

**1. Anahtarı indir** ve `/etc/apt/keyrings/` altına koy:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
sudo curl -fsSL https://depo.sirket.lan/anahtar.asc -o /etc/apt/keyrings/sirket.asc
```

**2. Depo dosyasını oluştur:**

```bash
sudo tee /etc/apt/sources.list.d/sirket.sources > /dev/null <<'EOF'
Types: deb
URIs: https://depo.sirket.lan/ubuntu
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/sirket.asc
EOF
```

**3. Listeyi güncelle ve kur:**

```bash
sudo apt update
apt-cache policy paket-adi      # paket bu depodan mı geliyor?
sudo apt install paket-adi
```

Kendi depomuzla denemede `apt update` çıktısında depo görünür:

```
Get:1 file:/srv/depo ./ InRelease [1481 B]
Get:2 file:/srv/depo ./ Packages [1151 B]
```

---

## 6. Örnek: Docker deposu

Docker'ın güncel kurulum belgeleri de aynı adımları izler ve depoyu `.sources` biçiminde ekler:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Buradaki `EOF` tırnaksız yazıldı, bu yüzden `$(...)` kısımları dosyaya yazılmadan önce çalışır ve sürüm kod adını (`noble`) ve mimariyi (`amd64`) yazar. Denemede oluşan dosya:

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

Eski belgelerde ve birçok blog yazısında aynı depo tek satırlık `.list` biçimiyle verilir. İkisi de çalışır ([3. bölüm](#3-depo-dosyalarının-iki-biçimi-sources-ve-list)), ama aynı depoyu iki dosyada birden tanımlama:

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable
```

Docker'ı kurup kullanmak: [Container'lar](11-containerlar.md).

---

## 7. PPA

**PPA** (Personal Package Archive), Launchpad üzerinde barındırılan, genelde tek bir yazılımın daha yeni sürümlerini sunan küçük depolardır. Eklemek tek komuttur, anahtarı ve depo dosyasını kendisi oluşturur:

```bash
sudo add-apt-repository ppa:kullanici/ppa-adi
sudo apt update
```

```bash
sudo add-apt-repository --list                       # eklenmiş depolar
sudo add-apt-repository --remove ppa:kullanici/ppa-adi
```

`add-apt-repository` komutu `software-properties-common` paketindedir. PPA'lar Ubuntu tarafından denetlenmez, sadece güvendiğin kaynakları ekle.

---

## 8. Paket hangi depodan gelecek?

Aynı paket birden fazla depoda varsa apt en yüksek sürümü seçer. Hangi sürümün hangi depodan geleceğini görmek için:

```bash
apt-cache policy sl
```
```
sl:
  Installed: (none)
  Candidate: 5.02-1
  Version table:
     5.02-1 500
        500 file:/srv/depo ./ Packages
        500 http://archive.ubuntu.com/ubuntu noble/universe amd64 Packages
```

`Candidate`, `apt install`'ın kuracağı sürümdür. Altındaki satırlar her sürümün hangi depolarda olduğunu ve önceliğini (`500` normal) gösterir. Bir depoya daha yüksek öncelik vermek (pinning) `/etc/apt/preferences.d/` altındaki dosyalarla yapılır: `man apt_preferences`.

---

## 9. Depo kaldırmak

```bash
sudo rm /etc/apt/sources.list.d/sirket.sources
sudo rm /etc/apt/keyrings/sirket.asc
sudo apt update
```

Kaldırılan depodan kurulmuş paketler sistemde kalır, sadece artık güncelleme almazlar. Onları da kaldırmak istiyorsan `apt remove`.

---

## 10. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `/etc/apt/sources.list.d/ubuntu.sources` | Ubuntu depoları |
| `/etc/apt/sources.list.d/ad.sources` / `.list` | Eklenen depo |
| `/etc/apt/keyrings/` | Depo anahtarları |
| `curl -fsSL URL -o /etc/apt/keyrings/ad.asc` | Anahtarı indir |
| `gpg --dearmor -o ad.gpg` | ASCII anahtarı ikiliye çevir |
| `apt update` | Depo eklendikten sonra listeyi yenile |
| `apt-cache policy paket` | Paketin hangi depodan geleceği |
| `add-apt-repository ppa:...` | PPA ekle |
| `add-apt-repository --list` / `--remove` | Listele / kaldır |
| `lsb_release -cs` | Sürüm kod adı (noble) |
| `dpkg --print-architecture` | Mimari (amd64) |

---

## 11. Sık yapılan hatalar

- **Depoyu ekleyip `apt update` yapmamak.** Paket bulunamaz.
- **`Signed-By` vermemek veya anahtarı koymamak.** `NO_PUBKEY` hatası.
- **ASCII anahtarı `.gpg` uzantılı dosyaya kaydetmek.** Ya `.asc` adını kullan ya `gpg --dearmor`.
- **`apt-key add` kullanmak.** Kaldırıldı ve güvensiz.
- **Yanlış sürüm kod adı yazmak.** Ubuntu 24.04 için `noble`, başka bir sürümün adı yazılırsa uyumsuz paketler gelebilir veya depo bulunamaz.
- **deb822 içeriğini `.list` dosyasına yazmak (veya tersi).**
- **Ubuntu'nun ana depo dosyasını düzenlemek.** Yeni depo için `sources.list.d/` altına ayrı dosya koy.

---

## 12. Alıştırmalar

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

Önceki konu: [Paket Yönetimi](06-paket-yonetimi.md) · Sonraki konu: [Kaynaktan Yazılım Kurmak](08-kaynaktan-kurma.md)
