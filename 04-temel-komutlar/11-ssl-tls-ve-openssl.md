# SSL/TLS ve OpenSSL

Bir tarayıcı `https://` ile başlayan bir siteye bağlandığında iki şeyden emin olmak ister: karşısındaki sunucu gerçekten iddia ettiği sunucu mu, ve aradaki trafiği başkası okuyabilir mi? Bunu sağlayan protokol **TLS**, sunucunun kimliğini kanıtlayan belge ise **sertifika**dır. Linux'ta anahtar üretmek, sertifika istemek, oluşturmak ve incelemek için kullanılan araç `openssl`'dir.

Senaryo: Şirketin iç ağında `panel.sirket.lan` adında bir izleme paneli kuruluyor. Önce hızlıca test etmek için kendi kendine imzalı (self-signed) bir sertifika oluşturacağız. Sonra şirket içi küçük bir sertifika otoritesi (CA) kurup panelin sertifikasını onunla imzalayacağız ve bağlantıyı test edeceğiz.

> [!NOTE]
> Bütün komutlar Ubuntu 24.04 ve OpenSSL 3.0.13 ile denenmiştir. Sunucu tarafı `openssl s_server` ile aynı makinede çalıştırıldı. Anahtarlar her seferinde rastgele üretildiği için seri numarası, parmak izi gibi değerler sende farklı çıkar.

## İçindekiler

1. [Temel kavramlar](#1-temel-kavramlar)
2. [SSL ve TLS](#2-ssl-ve-tls)
3. [Sertifikada ne var?](#3-sertifikada-ne-var)
4. [Dosya türleri ve PEM biçimi](#4-dosya-türleri-ve-pem-biçimi)
5. [openssl alt komutları](#5-openssl-alt-komutları)
6. [Gizli anahtar üretmek](#6-gizli-anahtar-üretmek)
7. [CSR oluşturmak](#7-csr-oluşturmak)
8. [Self-signed sertifika](#8-self-signed-sertifika)
9. [Sertifikayı incelemek](#9-sertifikayı-incelemek)
10. [Kendi CA'n ile imzalamak](#10-kendi-can-ile-imzalamak)
11. [Anahtar ve sertifika eşleşiyor mu?](#11-anahtar-ve-sertifika-eşleşiyor-mu)
12. [Bağlantıyı test etmek](#12-bağlantıyı-test-etmek)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Temel kavramlar

**Anahtar çifti.** Her sertifikanın arkasında matematiksel olarak bağlı iki anahtar vardır:

| | Gizli anahtar (private key) | Açık anahtar (public key) |
|---|---|---|
| Nerede durur? | Sadece sunucuda | Sertifikanın içinde, herkese gösterilir |
| Ne yapar? | İmza atar | O imzanın gerçekten gizli anahtarla atıldığını doğrular |
| Paylaşılır mı? | **Asla** | Evet |

Açık anahtardan gizli anahtar hesaplanamaz. TLS bağlantısında sunucu, gizli anahtarıyla bir şey imzalayarak "bu sertifikadaki açık anahtarın sahibi benim" der. Bağlantının geri kalanını şifreleyecek oturum anahtarları ise her bağlantıda ayrıca, iki taraf arasında anlaşılarak üretilir. Yani gizli anahtar asıl olarak **kimliği kanıtlamak** için kullanılır.

**Sertifika.** Bir açık anahtarı bir isme (`panel.sirket.lan`) bağlayan ve bunu birinin imzasıyla onaylayan belge.

**CA (Certificate Authority).** Sertifikaları imzalayan kurum. Tarayıcılar ve işletim sistemleri, güvendikleri CA'ların listesiyle gelir. Ubuntu'da bu liste `/etc/ssl/certs/` altındadır. Bir sertifika bu CA'lardan biri tarafından imzalanmışsa güvenilir sayılır.

**CSR (Certificate Signing Request).** Bir CA'dan sertifika istemek için hazırlanan başvuru. İçinde açık anahtar ve kimlik bilgileri (isim, kurum, ülke) vardır, gizli anahtar yoktur. CA bunu kontrol edip imzalar ve sana sertifikayı verir.

**Self-signed sertifika.** Bir CA'ya gitmeden, kendi gizli anahtarınla imzaladığın sertifika. Çalışır, trafiği şifreler, ama kimse onu onaylamadığı için tarayıcılar ve `curl` güvenmez, uyarı verir. Test ortamları ve iç kullanım içindir.

| Durum | Ne kullanılır? |
|---|---|
| İnternete açık gerçek site | Güvenilir bir CA'dan sertifika (örneğin Let's Encrypt) |
| Şirket içi sistemler | Şirketin kendi iç CA'sı, CA sertifikası şirket makinelerine yüklenir |
| Kısa süreli test | Self-signed |

---

## 2. SSL ve TLS

SSL, protokolün eski adıdır. SSL 2.0 ve 3.0 güvensiz bulunup bırakıldı, yerine TLS geldi. Bugün kullanılan sürümler TLS 1.2 ve TLS 1.3. Denememizdeki bağlantı da TLS 1.3 ile kuruldu:

```
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```

"SSL sertifikası" ifadesi alışkanlıktan hala kullanılıyor, ama kastedilen TLS'te kullanılan sertifikadır. Sertifikanın kendisi zaten protokolden bağımsız bir X.509 belgesidir. `openssl` adındaki SSL de aynı tarihten kalma.

---

## 3. Sertifikada ne var?

| Alan | Anlamı | Örnek |
|---|---|---|
| Subject | Sertifika kime ait? | `CN = panel.sirket.lan` |
| Issuer | Kim imzaladı? | `CN = Ornek Sirket Ic CA` |
| Validity | Geçerlilik başlangıcı ve bitişi (Not Before / Not After) | 1 yıl |
| Subject Public Key | Açık anahtar | RSA 2048 bit |
| Subject Alternative Name (SAN) | Sertifikanın geçerli olduğu bütün adlar ve IP'ler | `DNS:panel.sirket.lan, DNS:panel` |
| Serial Number | CA'nın verdiği benzersiz numara | |
| Signature | Issuer'ın imzası | `sha256WithRSAEncryption` |

Subject içindeki kısaltmalar:

| Kısaltma | Anlamı |
|---|---|
| `C` | Ülke, iki harf (`TR`) |
| `ST` | İl veya eyalet |
| `L` | Şehir/ilçe |
| `O` | Kurum |
| `OU` | Birim |
| `CN` | Common Name, genelde sunucunun adı |

> [!IMPORTANT]
> Tarayıcılar ve `curl`, sunucunun adını **SAN** alanında arar, CN'e bakmaz. Sadece CN'i dolu, SAN'ı boş bir sertifika, adı doğru yazılmış olsa bile "isim eşleşmiyor" hatası verir. Sertifika oluştururken SAN'ı mutlaka ekle ([7. bölüm](#7-csr-oluşturmak)).

---

## 4. Dosya türleri ve PEM biçimi

OpenSSL'in varsayılan biçimi **PEM**'dir: base64 ile kodlanmış, başında ve sonunda ne olduğunu söyleyen bir satır bulunan metin dosyası.

```
-----BEGIN CERTIFICATE-----
MIIDqzCCApOgAwIBAgIUVxEoyskJNsTAC4lnwi3WvM7roGUwDQYJKoZIhvcNAQEL
...
-----END CERTIFICATE-----
```

Dosyanın ne olduğunu uzantıdan değil, bu ilk satırdan anlarsın:

| İlk satır | İçerik |
|---|---|
| `BEGIN CERTIFICATE` | Sertifika |
| `BEGIN CERTIFICATE REQUEST` | CSR |
| `BEGIN PRIVATE KEY` | Şifresiz gizli anahtar |
| `BEGIN ENCRYPTED PRIVATE KEY` | Parolayla şifrelenmiş gizli anahtar |
| `BEGIN PUBLIC KEY` | Açık anahtar |

Uzantılar sadece alışkanlıktır: sertifika için `.crt`, `.pem`, `.cer`; anahtar için `.key`, `.pem`; CSR için `.csr`, `.req`. İçerik okunabilir değildir, okumak için `openssl` gerekir ([9. bölüm](#9-sertifikayı-incelemek)).

---

## 5. openssl alt komutları

`openssl` tek bir program, işi alt komutlarla yapar:

| Alt komut | Ne yapar? |
|---|---|
| `openssl genpkey` | Gizli anahtar üretir |
| `openssl pkey` | Anahtarı inceler, dönüştürür, açık anahtarı çıkarır |
| `openssl rsa` | Aynı işler, sadece RSA anahtarları için (eski komut) |
| `openssl req` | CSR oluşturur ve inceler; `-x509` ile self-signed sertifika üretir |
| `openssl x509` | Sertifikayı inceler; `-req` ile CSR imzalar |
| `openssl verify` | Sertifika zincirini doğrular |
| `openssl s_client` | Bir TLS sunucusuna bağlanıp sertifikasını gösterir |

Yardım için:

```bash
man openssl-req          # her alt komutun ayrı man sayfası var
man openssl-x509
openssl req -help        # seçeneklerin kısa listesi
```

`openssl req -help | grep days` gibi bir aramayla aradığın seçeneği hızlıca bulabilirsin ([Yardım Almak](01-yardim-almak.md)).

---

## 6. Gizli anahtar üretmek

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out panel.key
```

RSA yerine daha kısa ve hızlı olan modern bir algoritma da seçilebilir:

```bash
openssl genpkey -algorithm ED25519 -out panel-ed.key
```

Anahtarın sağlam olduğunu kontrol etmek ve açık anahtarı çıkarmak:

```bash
openssl pkey -in panel.key -check -noout     # Key is valid
openssl pkey -in panel.key -pubout           # -----BEGIN PUBLIC KEY-----
```

`openssl rsa -in panel.key -check -noout` da çalışır (`RSA key ok`), ama sadece RSA anahtarlarında. OpenSSL 3'te genel komut `pkey`.

OpenSSL gizli anahtar dosyalarını `600` izniyle yazar. Denemede umask `022` olmasına rağmen sertifika `644`, anahtar `600` oluştu. Anahtarı başka bir yere kopyalarken bu izni koru ([İzinler ve Özel Bitler](04-izinler-ve-ozel-bitler.md)).

---

## 7. CSR oluşturmak

Bir CA'dan sertifika istemek için. `openssl req -newkey`, anahtarı ve CSR'ı tek komutta üretir:

```bash
openssl req -newkey rsa:2048 -keyout panel.key -out panel.csr
```

| Parça | Anlamı |
|---|---|
| `-newkey rsa:2048` | Yeni bir 2048 bit RSA anahtarı üret |
| `-keyout panel.key` | Gizli anahtarı bu dosyaya yaz |
| `-out panel.csr` | CSR'ı bu dosyaya yaz |

Komut önce gizli anahtarı şifrelemek için bir **parola** (PEM pass phrase), sonra kimlik bilgilerini sorar:

```
Country Name (2 letter code) [AU]:TR
State or Province Name (full name) [Some-State]:Ankara
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Ornek Sirket
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:panel.sirket.lan
Email Address []:
```

Boş bırakmak istediğin alana Enter ile geçebilirsin.

### Soruları komut satırında cevaplamak: -subj ve -addext

Script'lerde ve sınavda daha pratik olan, her şeyi komutta vermek:

```bash
openssl req -newkey rsa:2048 -keyout panel.key -out panel.csr \
  -subj "/C=TR/O=Ornek Sirket/CN=panel.sirket.lan" \
  -addext "subjectAltName=DNS:panel.sirket.lan,DNS:panel"
```

`-subj` kimlik bilgilerini, `-addext` ise SAN alanını ekler. Birden fazla ad virgülle ayrılır, IP adresi de eklenebilir: `DNS:panel.sirket.lan,IP:10.0.0.20`.

### Şifreli anahtar

Parola verdiysen anahtar dosyası `BEGIN ENCRYPTED PRIVATE KEY` ile başlar. Anahtarı her kullanmak istediğinde parola gerekir. Yanlış parolada:

```
Could not read key from panel.key
```

Bu güvenli, ama bir web sunucusu her açıldığında parola sorarsa servis kendiliğinden başlayamaz. Bu yüzden sunucu anahtarları çoğunlukla şifresiz tutulur ve dosya izinleriyle korunur. Şifresiz anahtar üretmek için `-noenc` eklenir. Var olan şifreli anahtarın şifresini kaldırmak için:

```bash
openssl pkey -in panel.key -out panel-sifresiz.key
```

### CSR'ı kontrol etmek

CA'ya göndermeden önce içeriğine bak:

```bash
openssl req -in panel.csr -noout -text
openssl req -in panel.csr -noout -verify
```
```
Certificate request self-signature verify OK
```

`-text` çıktısında `Subject` satırını ve `Requested Extensions` altında SAN'ı görmelisin.

---

## 8. Self-signed sertifika

CSR'ı bir CA'ya göndermek yerine `-x509` ekleyince `openssl req` doğrudan, kendi anahtarıyla imzalanmış bir sertifika üretir:

```bash
openssl req -x509 -newkey rsa:2048 -noenc \
  -keyout panel.key -out panel.crt -days 365 \
  -subj "/C=TR/ST=Ankara/O=Ornek Sirket/CN=panel.sirket.lan" \
  -addext "subjectAltName=DNS:panel.sirket.lan,DNS:panel,IP:10.0.0.20"
```

| Parça | Anlamı |
|---|---|
| `-x509` | CSR yerine sertifika üret |
| `-newkey rsa:2048` | Yeni anahtar |
| `-noenc` | Anahtarı şifreleme, parola sorma |
| `-days 365` | Geçerlilik süresi |
| `-keyout` / `-out` | Anahtar ve sertifika dosyaları |

Sonuç iki dosya: `panel.key` (gizli anahtar, sunucuda kalır) ve `panel.crt` (sertifika, istemcilere gösterilir).

### -nodes ve -noenc

Eski kaynaklarda ve notlarda `-nodes` ("no DES", şifreleme yok) görürsün. OpenSSL 3'te hala çalışıyor ama yardım çıktısında `deprecated` olarak işaretli, yerine `-noenc` geldi:

```
 -noenc                Don't encrypt private keys
 -nodes                Don't encrypt private keys; deprecated
```

### -days vermezsen

`-days` olmadan oluşturduğumuz sertifika sadece **30 gün** geçerli oldu:

```
notBefore=Nov 26 08:11:58 2026 GMT
notAfter=Dec 26 08:11:58 2026 GMT
```

Test sertifikası bir ay sonra sessizce geçersiz olur. Süreyi her zaman açıkça ver.

### Self-signed olduğunu nasıl anlarsın?

Subject ve Issuer aynıdır, çünkü sertifikayı kendisi imzalamıştır:

```bash
openssl x509 -in panel.crt -noout -subject -issuer
```
```
subject=C = TR, ST = Ankara, O = Ornek Sirket, CN = panel.sirket.lan
issuer=C = TR, ST = Ankara, O = Ornek Sirket, CN = panel.sirket.lan
```

---

## 9. Sertifikayı incelemek

```bash
openssl x509 -in panel.crt -noout -text
```

| Seçenek | Anlamı |
|---|---|
| `-in` | Okunacak dosya |
| `-text` | Okunabilir metin olarak göster |
| `-noout` | Sertifikanın base64 halini tekrar basma |

Çıktının başı:

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            57:11:28:ca:c9:09:36:c4:c0:0b:89:67:c2:2d:d6:bc:ce:eb:a0:65
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = TR, ST = Ankara, O = Ornek Sirket, CN = panel.sirket.lan
        Validity
            Not Before: Nov 26 08:11:58 2026 GMT
            Not After : Nov 26 08:11:58 2027 GMT
        Subject: C = TR, ST = Ankara, O = Ornek Sirket, CN = panel.sirket.lan
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
...
```

Bütün metin yerine belirli alanları almak için:

| Seçenek | Ne gösterir? |
|---|---|
| `-subject` | Sahip |
| `-issuer` | İmzalayan |
| `-dates` | `notBefore` ve `notAfter` |
| `-serial` | Seri numarası |
| `-ext subjectAltName` | SAN alanı |
| `-fingerprint -sha256` | Parmak izi |
| `-pubkey` | Açık anahtar |
| `-checkend SANİYE` | Bu süre içinde dolacak mı? |

```bash
openssl x509 -in panel.crt -noout -ext subjectAltName
```
```
X509v3 Subject Alternative Name: 
    DNS:panel.sirket.lan, DNS:panel, IP Address:10.0.0.20
```

### Süresi dolmak üzere mi?

`-checkend`, sertifikanın verilen saniye içinde dolup dolmayacağına bakar ve çıkış koduyla bildirir. İzleme script'leri için idealdir:

```bash
openssl x509 -in panel.crt -noout -checkend 86400          # 1 gün
```
```
Certificate will not expire
```

30 günlük sertifikayı 60 gün için sorduğumuzda `Certificate will expire` dedi ve çıkış kodu `1` oldu.

---

## 10. Kendi CA'n ile imzalamak

Şirket içinde her sunucu için self-signed sertifika üretmek yerine bir iç CA kurmak daha düzenlidir: CA sertifikasını şirket makinelerine bir kez yüklersin, o CA'nın imzaladığı bütün sertifikalar güvenilir olur.

**1. CA'nın anahtarı ve sertifikası.** CA sertifikası da aslında self-signed bir sertifikadır:

```bash
openssl req -x509 -newkey rsa:4096 -noenc \
  -keyout ca.key -out ca.crt -days 3650 \
  -subj "/O=Ornek Sirket/CN=Ornek Sirket Ic CA"
```

`ca.key` bütün sistemin anahtarıdır. Ele geçen biri istediği adla güvenilir sertifika üretebilir, en sıkı korunması gereken dosya budur.

**2. Panel için CSR** ([7. bölümdeki](#7-csr-oluşturmak) gibi).

**3. CSR'ı CA ile imzala:**

```bash
openssl x509 -req -in panel.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out panel.crt -days 397 -copy_extensions copy
```

| Parça | Anlamı |
|---|---|
| `-req -in panel.csr` | Girdi bir CSR |
| `-CA ca.crt -CAkey ca.key` | İmzalayacak CA ve anahtarı |
| `-CAcreateserial` | Seri numaralarını tutan `ca.srl` dosyasını oluştur |
| `-copy_extensions copy` | CSR'daki eklentileri (SAN) sertifikaya kopyala |

> [!WARNING]
> `-copy_extensions copy` olmadan imzaladığımızda, CSR'da SAN olmasına rağmen sertifikada hiç eklenti yoktu: `No extensions in certificate`. Bu sertifika tarayıcıda ve `curl`'de isim hatası verir. OpenSSL 3.0'dan önceki sürümlerde bu seçenek yok, SAN ayrı bir dosyayla (`-extfile`) verilir.

Sonuç:

```bash
openssl x509 -in panel.crt -noout -subject -issuer
```
```
subject=C = TR, O = Ornek Sirket, CN = panel.sirket.lan
issuer=O = Ornek Sirket, CN = Ornek Sirket Ic CA
```

Issuer artık CA.

**4. Zinciri doğrula:**

```bash
openssl verify -CAfile ca.crt panel.crt
```
```
panel.crt: OK
```

`-CAfile` vermezsen, sistemin güvendiği CA'lar arasında bizim CA'mız olmadığı için `unable to get local issuer certificate` hatası alırsın. 8. bölümdeki self-signed sertifikayı bizim CA ile doğrulamaya çalışınca da `self-signed certificate` hatası verdi, çünkü onu CA değil kendisi imzalamıştı.

---

## 11. Anahtar ve sertifika eşleşiyor mu?

Bir sunucuda çok sayıda `.key` ve `.crt` dosyası birikince hangisinin hangisine ait olduğu karışır. Yanlış anahtar verilen bir web sunucusu başlamaz. Sertifikadaki açık anahtar ile gizli anahtardan çıkan açık anahtar aynı olmalı:

```bash
openssl x509 -in panel.crt -noout -pubkey | sha256sum
openssl pkey -in panel.key -pubout | sha256sum
```
```
9a118a2266c1229cca6656ffd515e8b579c96f7187bb5ef3050f2d9bf9604a90  -
9a118a2266c1229cca6656ffd515e8b579c96f7187bb5ef3050f2d9bf9604a90  -
```

İki özet aynıysa eşleşiyor. Başka bir anahtarla denediğimizde tamamen farklı bir özet çıktı. RSA anahtarlarında eski kaynaklarda `-modulus` ile karşılaştırma görürsün (`openssl x509 -noout -modulus` ve `openssl rsa -noout -modulus`). O da çalışır ama sadece RSA için.

---

## 12. Bağlantıyı test etmek

### openssl s_client

Çalışan bir TLS sunucusunun sertifikasını görmek için:

```bash
openssl s_client -connect panel.sirket.lan:443 </dev/null
```

`</dev/null`, bağlantı kurulduktan sonra beklemeden kapanmasını sağlar ([Yönlendirme](10-yonlendirme.md)). Çıktıdan önemli satırlar:

```
subject=C = TR, O = Ornek Sirket, CN = panel.sirket.lan
issuer=O = Ornek Sirket, CN = Ornek Sirket Ic CA
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Verify return code: 21 (unable to verify the first certificate)
```

`Verify return code` sıfır değil, çünkü `s_client` bizim CA'mızı tanımıyor. CA'yı ve beklenen adı verince:

```bash
openssl s_client -connect panel.sirket.lan:443 -CAfile ca.crt -verify_hostname panel.sirket.lan </dev/null
```

| Durum | Sonuç |
|---|---|
| `-CAfile` yok | `21 (unable to verify the first certificate)` |
| `-CAfile ca.crt`, doğru ad | `0 (ok)` |
| `-CAfile ca.crt`, ad `baska.sirket.lan` | `62 (hostname mismatch)` |

Uzaktaki bir sunucunun sertifikasının ne zaman dolacağını görmek için çıktıyı `openssl x509`'a aktarabilirsin:

```bash
openssl s_client -connect panel.sirket.lan:443 </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

### curl

```bash
curl https://panel.sirket.lan/
```
```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

| Komut | Sonuç |
|---|---|
| `curl https://panel.sirket.lan/` | Hata 60: CA tanınmıyor |
| `curl --cacert ca.crt https://panel.sirket.lan/` | Başarılı |
| `curl --cacert ca.crt https://127.0.0.1/` | Hata 60: `no alternative certificate subject name matches target host name '127.0.0.1'` |
| `curl -k https://panel.sirket.lan/` | Başarılı, ama doğrulama tamamen kapalı |

Üçüncü satır SAN'ın işini gösteriyor: sertifikada `127.0.0.1` adı olmadığı için CA doğru olsa bile bağlantı reddedildi. `-k` (`--insecure`) sadece test içindir, sahte bir sunucuyu da kabul eder.

### CA'yı sisteme tanıtmak

Bütün programların (`curl`, `wget`, paket yöneticisi) bizim CA'mıza güvenmesi için Ubuntu'da:

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/ornek-ic-ca.crt
sudo update-ca-certificates
```
```
1 added, 0 removed; done.
```

Dosya uzantısı `.crt` olmalı. Bundan sonra `--cacert` olmadan `curl https://panel.sirket.lan/` başarılı oldu. Kaldırmak için dosyayı silip `sudo update-ca-certificates --fresh` çalıştırılır.

---

## 13. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out a.key` | RSA anahtarı üret |
| `openssl pkey -in a.key -check -noout` | Anahtarı kontrol et |
| `openssl pkey -in a.key -pubout` | Açık anahtarı çıkar |
| `openssl pkey -in sifreli.key -out sifresiz.key` | Anahtarın şifresini kaldır |
| `openssl req -newkey rsa:2048 -keyout a.key -out a.csr` | Anahtar ve CSR üret |
| `... -subj "/C=TR/O=.../CN=ad"` | Kimlik bilgilerini komutta ver |
| `... -addext "subjectAltName=DNS:ad,IP:1.2.3.4"` | SAN ekle |
| `openssl req -in a.csr -noout -text -verify` | CSR'ı incele |
| `openssl req -x509 -newkey rsa:2048 -noenc -keyout a.key -out a.crt -days 365` | Self-signed sertifika |
| `openssl x509 -in a.crt -noout -text` | Sertifikayı oku |
| `openssl x509 -in a.crt -noout -subject -issuer -dates` | Belirli alanlar |
| `openssl x509 -in a.crt -noout -ext subjectAltName` | SAN |
| `openssl x509 -in a.crt -noout -checkend 86400` | 1 gün içinde doluyor mu? |
| `openssl x509 -req -in a.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out a.crt -days 397 -copy_extensions copy` | CSR'ı CA ile imzala |
| `openssl verify -CAfile ca.crt a.crt` | Zinciri doğrula |
| `openssl s_client -connect sunucu:443 </dev/null` | Sunucunun sertifikasını gör |
| `curl --cacert ca.crt https://...` | Belirli CA ile bağlan |
| `update-ca-certificates` | CA'yı sisteme tanıt |

---

## 14. Sık yapılan hatalar

- **Gizli anahtarı paylaşmak veya git deposuna eklemek.** Anahtar sızarsa sertifika yenilenmeli, anahtar baştan üretilmeli.
- **SAN eklememek.** Sadece CN'i dolu sertifika tarayıcıda ve `curl`'de isim hatası verir.
- **CA ile imzalarken `-copy_extensions copy`'yi unutmak.** CSR'daki SAN sertifikaya geçmez.
- **`-days` vermemek.** Sertifika 30 gün sonra dolar.
- **Sunucu anahtarını parolayla şifrelemek.** Servis her açılışta parola bekler, kendiliğinden başlamaz.
- **Anahtar dosyasının izinlerini gevşetmek.** `600` olmalı, sahibi servis kullanıcısı veya root.
- **`curl -k` ile "çalışıyor" deyip geçmek.** Doğrulama kapalıyken her şey çalışır.
- **`ca.key`'i sunucularla birlikte dağıtmak.** CA anahtarı sadece imza atılan makinede durmalı.

---

## 15. Alıştırmalar

Bütün alıştırmaları boş bir dizinde yap: `mkdir ~/ssl-deneme && cd ~/ssl-deneme`.

**1.** `wiki.sirket.lan` için 90 gün geçerli, şifresiz anahtarlı bir self-signed sertifika oluştur. SAN'da `wiki.sirket.lan` ve `wiki` adları olsun. Soru sormadan, tek komutta.

<details>
<summary>Çözüm</summary>

```bash
openssl req -x509 -newkey rsa:2048 -noenc \
  -keyout wiki.key -out wiki.crt -days 90 \
  -subj "/O=Ornek Sirket/CN=wiki.sirket.lan" \
  -addext "subjectAltName=DNS:wiki.sirket.lan,DNS:wiki"
```
</details>

**2.** 1. alıştırmadaki sertifikanın geçerlilik tarihlerini, SAN alanını ve self-signed olup olmadığını kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl x509 -in wiki.crt -noout -dates
openssl x509 -in wiki.crt -noout -ext subjectAltName
openssl x509 -in wiki.crt -noout -subject -issuer
```

`notAfter` 90 gün sonrası olmalı. Subject ve Issuer aynıysa self-signed.
</details>

**3.** Bir iç CA oluştur (`ca.key`, `ca.crt`, 10 yıl). Sonra `git.sirket.lan` için bir CSR üret (SAN'lı, şifresiz anahtar) ve bu CSR'ı CA ile 1 yıllığına imzala. SAN'ın sertifikaya geçtiğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl req -x509 -newkey rsa:4096 -noenc -keyout ca.key -out ca.crt \
  -days 3650 -subj "/O=Ornek Sirket/CN=Ornek Sirket Ic CA"

openssl req -newkey rsa:2048 -noenc -keyout git.key -out git.csr \
  -subj "/O=Ornek Sirket/CN=git.sirket.lan" \
  -addext "subjectAltName=DNS:git.sirket.lan"

openssl x509 -req -in git.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out git.crt -days 365 -copy_extensions copy

openssl x509 -in git.crt -noout -issuer -ext subjectAltName
```
</details>

**4.** `git.crt`'nin CA tarafından imzalandığını doğrula. Sonra `git.key`'in `git.crt` ile, `wiki.key`'in ise `git.crt` ile eşleşip eşleşmediğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
openssl verify -CAfile ca.crt git.crt          # git.crt: OK

openssl x509 -in git.crt -noout -pubkey | sha256sum
openssl pkey -in git.key -pubout | sha256sum    # aynı
openssl pkey -in wiki.key -pubout | sha256sum   # farklı
```
</details>

**5.** `wiki.crt` önümüzdeki 30 gün içinde dolacak mı? Peki 120 gün içinde? Her ikisinin çıkış kodunu da göster.

<details>
<summary>Çözüm</summary>

```bash
openssl x509 -in wiki.crt -noout -checkend $((30*86400)); echo $?
openssl x509 -in wiki.crt -noout -checkend $((120*86400)); echo $?
```

30 gün için `Certificate will not expire` ve `0`, 120 gün için `Certificate will expire` ve `1`.
</details>

---

Önceki konu: [Girdi/Çıktı Yönlendirme](10-yonlendirme.md) · Sonraki konu: [Git ile Sürüm Kontrolü](12-git.md)
