# Ortam Profilleri

> **Resmi sınav maddesi:** Manage personal and system-wide environment profiles

Kullanıcı bir terminal açtığında ya da giriş yaptığında, shell bazı dosyaları okuyarak ortamı hazırlar: hangi dizinlerde program aranacak, hangi değişkenler tanımlı olacak, yeni kullanıcının ev dizininde hangi dosyalar bulunacak. Bu bölümde bu dosyaları ve hangisinin ne zaman kullanılacağını ele alıyoruz.

Senaryomuz:

- Ekibin kendi yazdığı araçlar `/opt/ekip/bin` altında. Herkes bunları tam yol yazmadan çalıştırabilmeli.
- Sunucunun test sunucusu olduğunu belirten `EKIP_ORTAMI=test` değişkeni bütün kullanıcılarda tanımlı olmalı.
- Ekibe yeni katılan herkesin ev dizininde kuralları anlatan bir `OKU-BENI.txt` dosyası olmalı.

> [!NOTE]
> Komutlar Ubuntu 24.04 ve bash üzerinde denenmiştir.

## İçindekiler

1. [Ortam değişkenleri ve export](#1-ortam-değişkenleri-ve-export)
2. [PATH nasıl çalışır?](#2-path-nasıl-çalışır)
3. [Hangi dosya ne zaman okunur?](#3-hangi-dosya-ne-zaman-okunur)
4. [Kişisel ayarlar: ~/.profile ve ~/.bashrc](#4-kişisel-ayarlar-profile-ve-bashrc)
5. [Sistem geneli: /etc/environment](#5-sistem-geneli-etcenvironment)
6. [Sistem geneli: /etc/profile.d/](#6-sistem-geneli-etcprofiled)
7. [Yeni kullanıcılar için şablon: /etc/skel](#7-yeni-kullanıcılar-için-şablon-etcskel)
8. [Değişikliği hemen uygulamak: source](#8-değişikliği-hemen-uygulamak-source)
9. [Karar tablosu: hangi dosyayı kullanmalıyım?](#9-karar-tablosu-hangi-dosyayı-kullanmalıyım)
10. [Komut özeti](#10-komut-özeti)
11. [Sık yapılan hatalar](#11-sık-yapılan-hatalar)
12. [Alıştırmalar](#12-alıştırmalar)

---

## 1. Ortam değişkenleri ve export

Shell'de iki tür değişken var:

- **Shell değişkeni:** Sadece o anki shell'de yaşar.
- **Ortam değişkeni:** `export` edilmiş değişkendir. O shell'den başlatılan programlar ve alt shell'ler de görür.

```bash
X=1
export Y=2
bash -c 'echo "X=$X Y=$Y"'
```
```
X= Y=2
```

Alt shell `X`'i görmedi çünkü `X` export edilmemişti.

Değişkenlere bakmak için:

```bash
echo $Y          # tek bir değişkenin değeri
printenv Y       # sadece ortam değişkenlerini gösterir
printenv         # bütün ortam değişkenleri
env              # printenv ile aynı listeyi verir
set              # shell değişkenleri dahil hepsi
```

`printenv`, export edilmemiş bir değişken için hiçbir şey yazmaz ve hata koduyla döner. Bir değişkenin gerçekten export edilip edilmediğini anlamanın kolay yolu budur.

---

## 2. PATH nasıl çalışır?

`ls` yazdığında shell, `ls` adlı programı `PATH` değişkenindeki dizinlerde **soldan sağa** arar ve ilk bulduğunu çalıştırır:

```bash
echo $PATH
```
```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

Hiçbir dizinde bulamazsa `command not found` hatası verir.

Bir komutun nereden çalıştığını görmek için:

```bash
type -P ls      # çalışacak olan dosya
which -a ls     # PATH'te bulunan bütün kopyalar
```

### Dizini PATH'e ekleme

Ekip araçlarını PATH'e eklemenin iki yolu var:

```bash
export PATH="$PATH:/opt/ekip/bin"   # sona ekle
export PATH="/opt/ekip/bin:$PATH"   # başa ekle
```

Fark önemli. Başa eklenen dizin, sistemdeki aynı adlı programların **önüne geçer**. Denemede `/opt/deneme` dizinine `ls` adında sahte bir program koyduk:

```
PATH başına eklenince:   type -P ls  ->  /opt/deneme/ls     ("Bu gercek ls degil!")
PATH sonuna eklenince:   type -P ls  ->  /usr/bin/ls
```

Özel bir sebep yoksa dizinleri **sona** ekle. Böylece sistem komutları yanlışlıkla gölgelenmez.

> [!TIP]
> Ubuntu'da kullanıcının ev dizininde `~/bin` veya `~/.local/bin` dizini varsa, `~/.profile` bunları giriş sırasında kendiliğinden PATH'in **başına** ekler. Kendi script'lerini oraya koyabilirsin, ama sistem komutlarıyla aynı adı vermemeye dikkat et.

---

## 3. Hangi dosya ne zaman okunur?

Bir bash oturumu iki şekilde açılabilir:

- **Giriş shell'i (login shell):** SSH ile bağlandığında, konsoldan giriş yaptığında, `su -` veya `sudo -i` kullandığında.
- **Giriş olmayan shell:** Masaüstünde yeni bir terminal sekmesi açtığında, `bash` yazıp alt shell açtığında veya tiresiz `su` kullandığında.

Giriş sırasında dosyalar şu sırayla okunur:

```
/etc/environment      (PAM okur, shell'den önce)
      |
/etc/profile
      |
/etc/profile.d/*.sh
      |
~/.profile
      |
~/.bashrc             (~/.profile tarafından çağrılır)
```

Sonra okunan dosya öncekini ezer. Yani aynı değişken hem `/etc/profile.d/` altında hem `~/.profile`'da tanımlıysa kullanıcının değeri geçerli olur. Denemede sistemde `EKIP_PROFIL=var` tanımlıyken Ayşe'nin `~/.profile`'ına `EKIP_PROFIL=kullanici_degeri` yazdık ve Ayşe'nin oturumunda `kullanici_degeri` göründü.

Hangi dosyanın hangi durumda etkili olduğunu test ettik:

| Oturum | `/etc/environment` | `/etc/profile.d/*.sh` |
|---|---|---|
| `su - ayse` (giriş shell'i) | Okunur | Okunur |
| `su ayse` (tiresiz) | Okunur | Okunmaz |
| Giriş shell'inden açılan alt `bash` | Değişkenler miras kalır | Değişkenler miras kalır |
| PAM'den geçmeyen yeni bir `bash` | Okunmaz | Okunmaz |

---

## 4. Kişisel ayarlar: ~/.profile ve ~/.bashrc

Sadece bir kullanıcıyı etkileyen ayarlar o kullanıcının ev dizinindeki dosyalara yazılır.

Ubuntu'daki `~/.bashrc`'nin en başında şu satırlar var:

```bash
case $- in
    *i*) ;;
      *) return;;
esac
```

Bu satırlar, shell etkileşimli değilse (yani bir komut veya script çalıştırılıyorsa) dosyanın geri kalanını **okumadan çıkar**.

Bunun sonucunu denedik. Ayşe'nin `~/.bashrc` dosyasının sonuna `export BENIM=bashrc_sonu` ekledik:

```
Terminal açıp elle yazınca:              BENIM=[bashrc_sonu]
su - ayse -c 'echo $BENIM' ile:          BENIM=[]
```

Yani `~/.bashrc`'ye yazılan değişkenler script'lerde, `ssh sunucu komut` şeklinde çalıştırılan komutlarda ve cron görevlerinde **görünmez**. Aynı satırı `~/.profile`'a yazınca iki durumda da görünüyor.

| Dosya | Ne için kullan? |
|---|---|
| `~/.profile` | Ortam değişkenleri ve PATH eklemeleri (her girişte, script'lerde de geçerli) |
| `~/.bashrc` | Alias'lar, prompt ayarları gibi sadece elle kullanılan terminal için gerekenler |

Ayşe'nin kendi PATH'ine ekleme:

```bash
echo 'export PATH="$PATH:/opt/ekip/bin"' >> ~/.profile
```

> [!NOTE]
> `~/.bashrc` her yeni etkileşimli terminalde okunur, giriş shell'inde de `~/.profile` tarafından çağrılır. Ayarların elle açılan terminalde çalışması yetiyorsa `~/.bashrc` da kullanılabilir. Sınavda "kullanıcının PATH'ine ekle" deniyorsa ikisi de genelde kabul edilir. Ama script'lerde de çalışması gereken bir şeyse `~/.profile` doğru yerdir.

---

## 5. Sistem geneli: /etc/environment

`/etc/environment` bütün kullanıcılar için ortam değişkenlerini tanımlar. Bu dosyayı shell değil, giriş sırasında **PAM** (`pam_env` modülü) okur. Ubuntu'da dosyada zaten bir satır vardır:

```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
```

Bu, herkesin başlangıç PATH'idir. Ekip değişkenimizi ekleyelim:

```bash
echo 'EKIP_ORTAMI=test' | sudo tee -a /etc/environment
```

Dosya bir script değil, sadece `DEGISKEN=deger` satırlarından oluşan bir listedir. Kuralları denedik:

| Satır | Sonuç |
|---|---|
| `EKIP_ORTAMI=test` | `test` |
| `MESAJ=birden fazla kelime` | `birden fazla kelime`, boşluklar korunur |
| `MESAJ2="tirnakli deger"` | `tirnakli deger`, tırnaklar kaldırılır |
| `YOL_DENEME=$HOME/bin` | `$HOME/bin`, **değişken açılmaz** |
| `export DISA=1` | `1`, `export` kabul ediliyor ama gerekmez |

Bu tablodan çıkan iki önemli sonuç:

- **Değişkenler açılmaz.** `PATH=$PATH:/opt/ekip/bin` yazarsan PATH tam olarak `$PATH:/opt/ekip/bin` metni olur ve sistem komutları çalışmaz. PATH'e buradan ekleme yapacaksan, mevcut satırın tamamını yeni dizinle birlikte yeniden yazmalısın.
- **Komut çalıştırılamaz.** `if`, `echo` gibi komutlar burada çalışmaz.

Değişiklik **bir sonraki girişte** geçerli olur. Test etmek için yeni bir giriş yap ya da `su - kullanici` kullan. Dosyayı `source` ile okutmak doğru bir test değildir: bash dosyayı bir script gibi okur, değişkenler sadece o shell'de ve export edilmeden tanımlanır. PAM'in dosyayı nasıl okuduğunu ancak yeni bir giriş gösterir.

---

## 6. Sistem geneli: /etc/profile.d/

Bütün kullanıcılar için komut çalıştırman gerekiyorsa (örneğin PATH'e değişkenle ekleme yapmak gibi) `/etc/profile.d/` altına bir dosya koy. Giriş sırasında `/etc/profile` bu dizindeki dosyaları okur.

```bash
sudo tee /etc/profile.d/ekip.sh > /dev/null <<'EOF'
export PATH="$PATH:/opt/ekip/bin"
EOF
```

Ekip aracını `/opt/ekip/bin/sunucu-bilgi` olarak oluşturup denedik:

```bash
su - ayse -c 'sunucu-bilgi'
```
```
Sunucu: vm, kullanici: ayse
```

Denemelerde öğrendiklerimiz:

- **Dosya adı `.sh` ile bitmeli.** `/etc/profile` sadece `*.sh` dosyalarını okur. Uzantısız bir dosya denedik ve hiç okunmadı.
- **Çalıştırma izni gerekmez.** Dosyanın okunabilir olması yeterli (`644`). Dosya çalıştırılmaz, mevcut shell'in içine okunur (`source` edilir). Bu yüzden başına `#!/bin/bash` satırı yazmak da gerekmez.
- **Sadece giriş shell'lerinde okunur.** Tiresiz `su` ile açılan oturumda değişken yoktu.
- **Değişkenler miras kalır.** Giriş shell'inden açılan alt shell'ler, export edilmiş değerleri görür.

> [!TIP]
> Sistem genelindeki ayarlar için `/etc/profile` dosyasını doğrudan düzenleme. Paket güncellemelerinde değişebilir. Kendi ayarlarını `/etc/profile.d/` altında ayrı, anlaşılır isimli bir dosyada tut.

---

## 7. Yeni kullanıcılar için şablon: /etc/skel

`useradd -m` ile yeni bir hesap açıldığında `/etc/skel` dizinindeki her şey yeni ev dizinine kopyalanır:

```bash
ls -a /etc/skel
```
```
.  ..  .bash_logout  .bashrc  .profile
```

Yeni gelenler için bir bilgi dosyası ekleyelim:

```bash
echo "Proje dosyalari /srv/proje altinda tutulur." | sudo tee /etc/skel/OKU-BENI.txt
```

Ayşe'nin hesabı önceden açılmıştı, Mert'in hesabını dosyayı ekledikten sonra açtık:

```bash
sudo useradd -m -s /bin/bash mert
ls /home/mert/OKU-BENI.txt /home/ayse/OKU-BENI.txt
```
```
ls: cannot access '/home/ayse/OKU-BENI.txt': No such file or directory
/home/mert/OKU-BENI.txt
```

`/etc/skel`'deki değişiklikler **sadece bundan sonra açılan hesapları** etkiler. Kopyalama hesap açılırken bir kere yapılır. Aynı mantıkla `/etc/skel/.bashrc`'yi değiştirmek de mevcut kullanıcıların `.bashrc` dosyalarını değiştirmez.

Mevcut bir kullanıcıya dosyayı elle vermek istersen, kopyaladıktan sonra sahipliği de düzelt:

```bash
sudo cp /etc/skel/OKU-BENI.txt /home/ayse/
sudo chown ayse: /home/ayse/OKU-BENI.txt
```

---

## 8. Değişikliği hemen uygulamak: source

`~/.profile` veya `~/.bashrc`'yi düzenledikten sonra değişiklik açık terminale kendiliğinden yansımaz. Oturumu kapatıp açmak yerine dosyayı mevcut shell'e tekrar okutabilirsin:

```bash
echo 'export HEMEN=1' >> ~/.profile
echo $HEMEN        # boş
source ~/.profile  # veya: . ~/.profile
echo $HEMEN        # 1
```

- `source` sadece **o anki shell'i** etkiler. Diğer açık terminaller ve diğer kullanıcılar etkilenmez.
- `/etc/environment` bir script değil, PAM tarafından okunan bir liste. Değişikliği doğru şekilde görmek için yeni bir giriş gerekir: `su - kullanici`.

> [!WARNING]
> `source script.sh` ile `./script.sh` farklı şeylerdir. `./script.sh` script'i ayrı bir alt shell'de çalıştırır ve içinde tanımlanan değişkenler script bitince kaybolur. `source` ise script'i mevcut shell'in içinde çalıştırır, değişkenler kalır.

---

## 9. Karar tablosu: hangi dosyayı kullanmalıyım?

| İhtiyaç | Dosya | Komut çalıştırabilir mi? | sudo gerekir mi? | Ne zaman geçerli olur? |
|---|---|---|---|---|
| Sadece bir kullanıcı, script'lerde de geçerli | `~/.profile` | Evet | Hayır | Sonraki girişte veya `source ~/.profile` ile |
| Sadece bir kullanıcı, elle açılan terminal | `~/.bashrc` | Evet | Hayır | Yeni terminalde veya `source ~/.bashrc` ile |
| Bütün kullanıcılar, sabit değer | `/etc/environment` | Hayır | Evet | Sonraki girişte |
| Bütün kullanıcılar, komut veya değişken gerekiyor | `/etc/profile.d/isim.sh` | Evet | Evet | Sonraki girişte |
| Sadece bundan sonra açılacak hesaplar | `/etc/skel/` | Kopyalanan dosyaya göre | Evet | Hesap açılırken |

Karar verirken iki soru sor:

1. **Kimi etkilemeli?** Tek kullanıcı mı, herkes mi, yoksa sadece yeni gelenler mi?
2. **Sabit bir değer mi, yoksa komut mu?** `EKIP_ORTAMI=test` sabit bir değer, `/etc/environment` yeter. `PATH="$PATH:/opt/ekip/bin"` mevcut değeri kullanıyor, bu yüzden `/etc/profile.d/` gerekir.

> [!IMPORTANT]
> `/etc/environment` ve `/etc/profile.d/` hem mevcut hem de yeni kullanıcıları etkiler. Sadece yeni hesapları etkileyen tek yer `/etc/skel`'dir.

---

## 10. Komut özeti

| Komut / Dosya | Ne işe yarar? |
|---|---|
| `export DEGISKEN=deger` | Değişkeni tanımlar ve alt süreçlere aktarır |
| `echo $DEGISKEN` | Değişkenin değerini gösterir |
| `printenv` / `env` | Ortam değişkenlerini listeler |
| `printenv DEGISKEN` | Tek bir ortam değişkenini gösterir |
| `type -P komut` | Çalışacak programın yolunu gösterir |
| `which -a komut` | PATH'teki bütün kopyaları gösterir |
| `source dosya` / `. dosya` | Dosyayı mevcut shell'e okutur |
| `su - kullanici` | Giriş shell'iyle yeni oturum açar (ayarları test etmek için) |
| `~/.profile` | Kullanıcının ortam değişkenleri |
| `~/.bashrc` | Kullanıcının terminal ayarları (alias, prompt) |
| `/etc/environment` | Herkes için sabit değişkenler |
| `/etc/profile.d/*.sh` | Herkes için giriş sırasında çalışan ayarlar |
| `/etc/skel/` | Yeni hesaplara kopyalanan şablon dosyalar |

---

## 11. Sık yapılan hatalar

- **`export`'u unutmak.** Değişken sadece o shell'de kalır, çalıştırdığın programlar görmez.
- **PATH'in başına ekleme yapmak.** Sistem komutları aynı adlı başka programlar tarafından gölgelenebilir.
- **Değişkenleri `~/.bashrc`'nin sonuna yazıp script'lerde beklemek.** Etkileşimsiz shell'de `.bashrc` baştan döner. Ortam değişkenleri için `~/.profile` kullan.
- **`/etc/environment`'ta `$PATH` kullanmak.** Değişkenler açılmaz, PATH bozulur.
- **`/etc/profile.d/` dosyasına `.sh` uzantısı vermemek.** Dosya hiç okunmaz.
- **`/etc/skel`'i değiştirince mevcut kullanıcıların da etkileneceğini sanmak.** Sadece yeni hesapları etkiler.
- **`source` ile `./script.sh`'yi karıştırmak.** Sadece `source` değişkenleri mevcut shell'de bırakır.

---

## 12. Alıştırmalar

Kendi sanal makinende dene, sonra çözümlere bak.

**1.** Bütün kullanıcılar için `SIRKET=acme` değişkenini tanımla. Değişkenin `elif` kullanıcısının yeni bir oturumunda göründüğünü doğrula (kullanıcı yoksa oluştur).

<details>
<summary>Çözüm</summary>

```bash
sudo useradd -m -s /bin/bash elif
echo 'SIRKET=acme' | sudo tee -a /etc/environment
sudo su - elif -c 'echo $SIRKET'
```
</details>

**2.** `/opt/araclar/bin` dizini bütün kullanıcıların PATH'inin **sonuna** eklensin. Dizine `merhaba` adında küçük bir script koyup `elif` ile tam yol yazmadan çalıştır.

<details>
<summary>Çözüm</summary>

```bash
sudo mkdir -p /opt/araclar/bin
printf '#!/bin/bash\necho "Merhaba $(whoami)"\n' | sudo tee /opt/araclar/bin/merhaba
sudo chmod 755 /opt/araclar/bin/merhaba
echo 'export PATH="$PATH:/opt/araclar/bin"' | sudo tee /etc/profile.d/araclar.sh
sudo su - elif -c 'merhaba'
```

`/etc/environment` burada uygun değil, çünkü mevcut `$PATH` değerini kullanmamız gerekiyor.
</details>

**3.** Bundan sonra açılacak her hesabın ev dizininde `KURALLAR.txt` dosyası olsun. `can` kullanıcısını oluşturup dosyanın geldiğini, `elif`'te ise olmadığını doğrula.

<details>
<summary>Çözüm</summary>

```bash
echo "Sunucuda yogun islem yapmadan once ekibe haber ver." | sudo tee /etc/skel/KURALLAR.txt
sudo useradd -m -s /bin/bash can
ls /home/can/KURALLAR.txt /home/elif/KURALLAR.txt
```
</details>

**4.** Sadece `elif` için `EDITOR=nano` değişkenini tanımla. Değişken, `elif` olarak çalıştırılan komutlarda da (`su - elif -c ...`) görünsün.

<details>
<summary>Çözüm</summary>

```bash
sudo su - elif
echo 'export EDITOR=nano' >> ~/.profile
exit
sudo su - elif -c 'echo $EDITOR'
```

`~/.bashrc`'nin sonuna yazsaydık `su - elif -c` ile çalıştırılan komutta görünmezdi.
</details>

**5.** Alıştırmada yaptığın her şeyi geri al.

<details>
<summary>Çözüm</summary>

```bash
sudo sed -i '/^SIRKET=acme$/d' /etc/environment
sudo rm /etc/profile.d/araclar.sh /etc/skel/KURALLAR.txt
sudo rm -rf /opt/araclar
sudo userdel -r elif
sudo userdel -r can
```
</details>

---

Önceki konu: [sudo ve su](03-sudo-ve-su.md) · Sonraki konu: [Kaynak Limitleri](05-kaynak-limitleri.md)
