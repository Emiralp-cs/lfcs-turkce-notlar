# Ters Proxy ve Yük Dengeleme (Nginx)

> **Resmi sınav maddesi:** Implement reverse proxies and load balancers

**Ters proxy** (reverse proxy), istemciler ile asıl uygulama sunucuları arasında duran bir aracıdır. İstemci sadece proxy'yi görür, proxy isteği arkadaki uygun sunucuya iletir, cevabı da istemciye geri verir.

```
İstemci  ->  Nginx (ters proxy)  ->  Uygulama sunucusu
         <-                      <-
```

Faydaları:

- Uygulama sunucusunu değiştirdiğinde veya taşıdığında sadece proxy ayarını güncellersin, istemciler ve DNS etkilenmez.
- Uygulama sunucuları doğrudan internete açılmaz.
- Proxy, önbellekleme ve TLS gibi işleri uygulamanın yerine yapabilir.

**Yük dengeleme** (load balancing) ise gelen istekleri birden fazla sunucuya dağıtmaktır. Böylece tek bir sunucu aşırı yüklenmez, biri çökerse diğerleri hizmet vermeye devam eder. Nginx ikisini de yapar.

Senaryo: Ekibin sunucusunda Nginx çalışıyor. Arkada uygulamayı çalıştıran sunucular var:

| Sunucu | Adres | Rolü |
|---|---|---|
| app1 | `10.0.0.11:8000` | Uygulama |
| app2 | `10.0.0.12:8000` | Uygulama |
| app3 | `10.0.0.13:8081` | API servisi (farklı port) |
| yedek | `10.0.0.14:8000` | Diğerleri çökerse devreye girecek yedek |

> [!NOTE]
> Bütün yapılandırmalar Ubuntu 24.04 ve Nginx 1.24 ile denendi. Arka uç sunucular, kendi adlarını ve gelen başlıkları yazan küçük test uygulamalarıydı. Böylece her isteğin hangi sunucuya gittiği görülebildi. Denemede arka uçlar aynı makinede `127.0.0.11`–`14` adreslerinde, istemci `127.0.0.50` adresindeydi. Dosyada okunabilirlik için bunlar `10.0.0.11`–`14` ve `192.168.10.50` olarak yazıldı. Deneme ortamı systemd ile çalışmadığı için Nginx doğrudan başlatıldı ve `systemctl reload nginx` yerine `nginx -s reload` kullanıldı.

## İçindekiler

1. [Nginx'in dosya düzeni](#1-nginxin-dosya-düzeni)
2. [Basit bir ters proxy](#2-basit-bir-ters-proxy)
3. [Varsayılan site neden kaldırılmalı?](#3-varsayılan-site-neden-kaldırılmalı)
4. [Belirli bir yolu yönlendirmek: location](#4-belirli-bir-yolu-yönlendirmek-location)
5. [Gerçek istemci bilgisini iletmek: proxy_params](#5-gerçek-istemci-bilgisini-iletmek-proxy_params)
6. [Değişiklikleri test etmek ve uygulamak](#6-değişiklikleri-test-etmek-ve-uygulamak)
7. [Yük dengeleme: upstream](#7-yük-dengeleme-upstream)
8. [Dağıtım yöntemleri: weight ve least_conn](#8-dağıtım-yöntemleri-weight-ve-least_conn)
9. [down ve backup](#9-down-ve-backup)
10. [Bir sunucu çökerse ne olur?](#10-bir-sunucu-çökerse-ne-olur)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Nginx'in dosya düzeni

```bash
sudo apt install nginx
```

Ubuntu'daki Nginx yapılandırması:

| Yol | İçerik |
|---|---|
| `/etc/nginx/nginx.conf` | Ana yapılandırma. `sites-enabled/` ve `conf.d/` dosyalarını içeri alır |
| `/etc/nginx/sites-available/` | Site yapılandırmaları burada yazılır |
| `/etc/nginx/sites-enabled/` | Etkin siteler: `sites-available`'daki dosyalara sembolik bağlantılar |
| `/etc/nginx/proxy_params` | Ters proxy için hazır başlık ayarları |
| `/var/log/nginx/error.log` | Hata logları |

Bir siteyi etkinleştirmek, `sites-available`'daki dosyaya `sites-enabled`'da bağlantı oluşturmak demek. Devre dışı bırakmak için bağlantı silinir, dosya yerinde kalır.

Kurulumdan sonra `sites-enabled/` içinde `default` adlı bir site vardır ve 80. portta "Welcome to nginx!" sayfasını gösterir.

---

## 2. Basit bir ters proxy

Bütün istekler app1'e gitsin:

```bash
sudo vim /etc/nginx/sites-available/proxy.conf
```

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://10.0.0.11:8000;
    }
}
```

| Satır | Anlamı |
|---|---|
| `listen 80;` | 80. portu dinle |
| `location / { ... }` | `/` ile başlayan, yani bütün istekler için |
| `proxy_pass http://10.0.0.11:8000;` | İsteği bu adrese ilet. Port 80 değilse adresin sonuna yazılır |

Etkinleştir, test et ve uygula:

```bash
sudo ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/proxy.conf
sudo nginx -t
sudo systemctl reload nginx
```

Ama bu noktada bir sürpriz var.

---

## 3. Varsayılan site neden kaldırılmalı?

Yukarıdaki adımlardan sonra istek gönderdik:

```bash
curl http://192.168.10.20/
```
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

İstek uygulamaya değil, varsayılan siteye gitti. `nginx -t` hata vermemişti.

Sebebi, varsayılan sitedeki `listen 80 default_server;` satırı. Aynı portu dinleyen birden fazla site varken ve isteğe uyan özel bir `server_name` yokken, istek `default_server` olarak işaretli siteye gider.

Varsayılan sitenin bağlantısını kaldırınca:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
curl http://192.168.10.20/rapor
```
```
app1 | yol=/rapor | Host=10.0.0.11:8000 | X-Real-IP=None | ...
```

İstek app1'e ulaştı. Arka uç, isteğin yolunu (`/rapor`) olduğu gibi aldı.

> [!IMPORTANT]
> Varsayılan siteyi kaldırmak "isteğe bağlı" değil. Kendi sitende `server_name` tanımlamıyorsan ve ikisi de 80'i dinliyorsa, `default` siteyi kaldırmazsan istekler hiç senin siteye gelmez. Alternatif olarak kendi sitende `listen 80 default_server;` yazıp varsayılan sitedekini kaldırabilirsin, ama iki sitede birden `default_server` olamaz.

### İki site aynı anda etkinse

Benzer bir durum iki kendi siten arasında da olur. `server_name`'i olmayan iki site 80'i dinliyorsa `nginx -t` uyarı verir:

```
[warn] conflicting server name "" on 0.0.0.0:80, ignored
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Test "successful" der ama ikinci site **yok sayılır**. Bir yapılandırmadan diğerine geçerken (örneğin ters proxy'den yük dengeleyiciye) eski sitenin bağlantısını kaldır.

---

## 4. Belirli bir yolu yönlendirmek: location

Sadece belirli bir yolu yönlendirmek için `location`'a o yol yazılır:

```nginx
server {
    listen 80;
    location /images {
        proxy_pass http://10.0.0.11:8000;
        include proxy_params;
    }
    location /api/ {
        proxy_pass http://10.0.0.13:8081/;
        include proxy_params;
    }
}
```

Denemedeki sonuçlar:

```
GET /images/logo.png    ->  app1 | yol=/images/logo.png
GET /api/kullanicilar   ->  app3 | yol=/kullanicilar
```

İki location arasındaki fark `proxy_pass`'in sonundaki `/`:

| `proxy_pass` | Arka uca giden yol |
|---|---|
| `http://10.0.0.11:8000` (sonunda yol yok) | İsteğin yolu olduğu gibi: `/images/logo.png` |
| `http://10.0.0.13:8081/` (sonunda `/` var) | Location'a uyan kısım (`/api/`) silinip yerine `/` konur: `/kullanicilar` |

Arka uç uygulama `/api/...` yollarını bekliyorsa sondaki `/`'yu yazma. Uygulama kökten (`/kullanicilar`) çalışıyorsa yaz.

> [!WARNING]
> `location /images` bir **önektir**, `/images` ile başlayan her yola uyar. Denemede `/imagesXYZ` isteği de app1'e gitti. Sadece `/images/` altındaki yolları istiyorsan `location /images/` yaz.

Hiçbir `location`'a uymayan istekler (bu örnekte `/`) proxy'ye gitmez. Nginx onları kendi varsayılan web kök dizininden sunmaya çalışır.

---

## 5. Gerçek istemci bilgisini iletmek: proxy_params

Arka uç sunucu bağlantıyı Nginx'ten aldığı için, istemcinin kim olduğunu doğrudan bilemez. 3. bölümdeki çıktıda arka uç `Host=10.0.0.11:8000` ve `X-Real-IP=None` görüyordu.

Ubuntu'daki `/etc/nginx/proxy_params` dosyası bunu çözen hazır başlıkları içerir:

```nginx
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

`include proxy_params;` ekledikten sonra (`192.168.10.50`'den, `Host: uygulama.ekip.local` başlığıyla):

```
app1 | yol=/images/logo.png | Host=uygulama.ekip.local | X-Real-IP=192.168.10.50 | X-Forwarded-For=192.168.10.50
```

| Başlık | Ne iletir? |
|---|---|
| `Host` | İstemcinin yazdığı alan adı. Yoksa arka uç Nginx'in ona bağlandığı adresi görür |
| `X-Real-IP` | İstemcinin gerçek adresi |
| `X-Forwarded-For` | İsteğin geçtiği adreslerin listesi |
| `X-Forwarded-Proto` | İstemcinin `http` mi `https` mi kullandığı |

`X-Forwarded-For` listeye ekleme yapar. İstek kendi başlığıyla geldiyse Nginx onun arkasına istemcinin adresini ekler. Denemede `X-Forwarded-For: 1.2.3.4` başlığıyla istek gönderdik ve arka uç `1.2.3.4, 192.168.10.50` gördü. Bu yüzden arka uç tarafında güvenilir kaynak olarak `X-Real-IP` daha sağlamdır, çünkü onu Nginx bağlantının kendisinden doldurur.

`proxy_params` yoksa arka uç sunucunun loglarında bütün istekler Nginx'ten gelmiş gibi görünür.

---

## 6. Değişiklikleri test etmek ve uygulamak

Her değişiklikten sonra:

```bash
sudo nginx -t
```
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Bir satırın sonundaki `;`'yu unuttuğumuzda:

```
[emerg] invalid number of arguments in "proxy_pass" directive in /etc/nginx/sites-enabled/lb.conf:10
nginx: configuration file /etc/nginx/nginx.conf test failed
```

Test başarılıysa uygula:

```bash
sudo systemctl reload nginx
```

`reload`, açık bağlantıları koparmadan yeni ayarları okur. Hatalı bir yapılandırmayla `reload` yapılırsa Nginx eski ayarlarla çalışmaya devam eder, ama `restart` yapılırsa hiç açılmayabilir. Bu yüzden önce her zaman `nginx -t`.

---

## 7. Yük dengeleme: upstream

Birden fazla sunucuyu bir **upstream** grubunda toplarsın ve `proxy_pass` bu gruba yönlendirilir:

```bash
sudo rm /etc/nginx/sites-enabled/proxy.conf
sudo vim /etc/nginx/sites-available/lb.conf
```

```nginx
upstream uygulama {
    server 10.0.0.11:8000;
    server 10.0.0.12:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://uygulama;
        include proxy_params;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/lb.conf /etc/nginx/sites-enabled/lb.conf
sudo nginx -t && sudo systemctl reload nginx
```

| Parça | Anlamı |
|---|---|
| `upstream uygulama { ... }` | `uygulama` adında bir sunucu grubu |
| `server ADRES:PORT;` | Gruptaki bir sunucu. Port yazılmazsa 80 |
| `proxy_pass http://uygulama;` | İstekleri gruba gönder |

`upstream` bloğu `server` bloğunun **dışında** yazılır. `proxy_pass`'teki ad, upstream'in adıyla aynı olmalı.

Varsayılan dağıtım **round robin**, yani sırayla. Dört istek gönderdik:

```
app2 app1 app2 app1
```

12 istekte dağılım `7 app1, 5 app2` oldu. Nginx birden fazla işçi süreçle çalıştığı ve her süreç kendi sırasını tuttuğu için dağılım tam eşit olmayabilir, ama genel olarak dengelidir.

Gruptaki sunucuların portları farklı olabilir:

```nginx
upstream uygulama {
    server 10.0.0.11:8000;
    server 10.0.0.13:8081;
}
```

---

## 8. Dağıtım yöntemleri: weight ve least_conn

### weight: güçlü sunucuya daha fazla iş

app1 daha güçlü bir makine olsun:

```nginx
upstream uygulama {
    server 10.0.0.11:8000 weight=3;
    server 10.0.0.12:8000;
}
```

12 istekte: `9 app1, 3 app2`. app1, app2'nin üç katı istek aldı. `weight` yazılmayan sunucunun ağırlığı 1'dir.

### least_conn: en az meşgul sunucuya

```nginx
upstream uygulama {
    least_conn;
    server 10.0.0.11:8000;
    server 10.0.0.12:8000;
}
```

Yeni istek, o an en az aktif bağlantısı olan sunucuya gider. Bazı istekler uzun sürüyorsa (rapor üretme, dosya yükleme gibi) round robin'den daha adil dağıtır. Denemedeki gibi her istek anında bittiğinde aktif bağlantı sayıları hep eşit kaldığı için dağılım round robin'e benzer çıkar.

`least_conn` ile `weight` birlikte de kullanılabilir.

---

## 9. down ve backup

### down: sunucuyu geçici olarak çıkarmak

app1 bakıma alınacak:

```nginx
upstream uygulama {
    server 10.0.0.11:8000 down;
    server 10.0.0.12:8000;
}
```

8 istekte: `8 app2`. app1'e hiç istek gitmedi. Satırı silmek yerine `down` yazmak, bakım bitince sadece bu kelimeyi silerek sunucuyu geri almayı kolaylaştırır.

### backup: yedek sunucu

```nginx
upstream uygulama {
    server 10.0.0.11:8000;
    server 10.0.0.12:8000;
    server 10.0.0.14:8000 backup;
}
```

Bütün sunucular çalışırken 8 istekte: `5 app1, 3 app2`. Yedeğe hiç istek gitmedi. `backup` sunucusu sadece normal sunucuların hiçbirine ulaşılamadığında kullanılır ([10. bölüm](#10-bir-sunucu-çökerse-ne-olur)).

`backup`, `least_conn` ile birlikte kullanılabilir, denemede `nginx -t` hata vermedi.

---

## 10. Bir sunucu çökerse ne olur?

9. bölümdeki `backup`'lı yapılandırmayla sunucuları sırayla kapattık:

| Durum | 8 isteğin dağılımı |
|---|---|
| Hepsi çalışıyor | 5 app1, 3 app2 |
| app1 kapalı | 8 app2 |
| app1 ve app2 kapalı | 6 isteğin hepsi yedek |
| Hepsi kapalı | `502 Bad Gateway` |
| app1 tekrar açıldı (10 saniye sonra) | 6 isteğin hepsi app1 (app2 hala kapalı) |

Gözlemler:

- Bir sunucu çöktüğünde istemciler hata görmedi. Nginx bağlanamadığı sunucuyu atlayıp isteği bir sonrakine gönderdi.
- Çalışan bir normal sunucu kaldığı sürece yedek kullanılmadı.
- Nginx, bağlanamadığı bir sunucuyu varsayılan olarak 10 saniye boyunca devre dışı sayar (`fail_timeout=10s`), sonra tekrar dener. app1 geri gelince trafik yine ona döndü.

Hepsi kapalıyken hata logunda:

```
connect() failed (111: Connection refused) while connecting to upstream, ... upstream: "http://10.0.0.11:...
```

`502 Bad Gateway` gördüğünde ilk bakılacak yer burası: Nginx çalışıyor ama arka uca ulaşamıyor.

---

## 11. Komut özeti

| Komut / Dosya | Ne yapar? |
|---|---|
| `apt install nginx` | Nginx'i kurar |
| `/etc/nginx/sites-available/site.conf` | Site yapılandırması |
| `ln -s /etc/nginx/sites-available/X /etc/nginx/sites-enabled/X` | Siteyi etkinleştirir |
| `rm /etc/nginx/sites-enabled/default` | Varsayılan siteyi devre dışı bırakır |
| `nginx -t` | Yapılandırmayı test eder |
| `systemctl reload nginx` | Ayarları bağlantıları koparmadan uygular |
| `proxy_pass http://ADRES:PORT;` | İsteği arka uca iletir |
| `proxy_pass http://ADRES:PORT/;` | Location önekini silerek iletir |
| `include proxy_params;` | İstemci bilgisini başlıklarla iletir |
| `upstream AD { server ...; }` | Sunucu grubu tanımlar |
| `proxy_pass http://AD;` | İsteği gruba gönderir |
| `weight=N` | Sunucunun ağırlığı |
| `least_conn;` | En az aktif bağlantılı sunucuya gönderir |
| `down` | Sunucuyu geçici olarak devre dışı bırakır |
| `backup` | Sadece diğerleri çökünce kullanılır |
| `/var/log/nginx/error.log` | 502 gibi hataların sebebi |

---

## 12. Sık yapılan hatalar

- **Varsayılan siteyi kaldırmamak.** İstekler `default_server`'a gider ve `nginx -t` bunu hata olarak göstermez.
- **İki siteyi aynı anda etkin bırakmak.** "conflicting server name" uyarısı çıkar ve ikinci site yok sayılır.
- **`proxy_pass` sonundaki `/`'nun etkisini bilmemek.** Arka uca giden yol değişir.
- **`location /images` yazıp sadece `/images/`'ı kapsadığını sanmak.** `/imagesXYZ` de uyar.
- **`include proxy_params;`'ı unutmak.** Arka uç gerçek istemci adresini göremez.
- **Satır sonundaki `;`'yu unutmak.** `nginx -t` hata verir.
- **`nginx -t` çalıştırmadan `restart` yapmak.** Hatalı yapılandırmayla Nginx açılmayabilir.
- **`upstream` bloğunu `server` bloğunun içine yazmak.** `upstream` dışarıda olmalı.
- **`backup` sunucusunun yükü paylaşacağını sanmak.** Sadece diğerleri çökünce devreye girer.
- **502 hatasında Nginx'i suçlamak.** Önce arka uç sunucuların çalışıp çalışmadığına ve `error.log`'a bak.

---

## 13. Alıştırmalar

**1.** Nginx, 80. porta gelen bütün istekleri `10.0.0.21:3000` adresindeki uygulamaya iletsin. Uygulama istemcinin gerçek adresini görebilsin.

<details>
<summary>Çözüm</summary>

`/etc/nginx/sites-available/uygulama.conf`:

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://10.0.0.21:3000;
        include proxy_params;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/uygulama.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
</details>

**2.** Sadece `/docs/` altındaki istekler `10.0.0.22:8080`'e gitsin ve arka uç bu istekleri `/docs/` öneki olmadan alsın. `/docs/kurulum.html` isteği arka uca `/kurulum.html` olarak gitmeli.

<details>
<summary>Çözüm</summary>

```nginx
location /docs/ {
    proxy_pass http://10.0.0.22:8080/;
    include proxy_params;
}
```

`proxy_pass`'in sonundaki `/`, location önekinin (`/docs/`) silinmesini sağlar.
</details>

**3.** `10.0.0.31`, `10.0.0.32` ve `10.0.0.33` sunucuları arasında yük dengeleme yap. `10.0.0.31` diğerlerinin iki katı istek alsın, istekler en az meşgul sunucuya gitsin.

<details>
<summary>Çözüm</summary>

```nginx
upstream uygulama {
    least_conn;
    server 10.0.0.31 weight=2;
    server 10.0.0.32;
    server 10.0.0.33;
}

server {
    listen 80;
    location / {
        proxy_pass http://uygulama;
        include proxy_params;
    }
}
```
</details>

**4.** 3. alıştırmada `10.0.0.32` bakıma alınacak. Ayrıca `10.0.0.40` sadece diğerleri çökerse kullanılacak bir yedek olsun.

<details>
<summary>Çözüm</summary>

```nginx
upstream uygulama {
    least_conn;
    server 10.0.0.31 weight=2;
    server 10.0.0.32 down;
    server 10.0.0.33;
    server 10.0.0.40 backup;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```
</details>

**5.** Yapılandırmayı etkinleştirdin, `nginx -t` başarılı, ama tarayıcıda hala "Welcome to nginx!" sayfası görünüyor. Sebebi ne olabilir? Başka bir durumda da `502 Bad Gateway` alıyorsun. İkisini nasıl incelersin?

<details>
<summary>Çözüm</summary>

"Welcome to nginx!": `sites-enabled/default` hala etkin ve `default_server` olarak istekleri alıyor.

```bash
ls -l /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

502: Nginx arka uca ulaşamıyor.

```bash
sudo tail /var/log/nginx/error.log      # hangi upstream'e bağlanılamadı?
curl http://10.0.0.31:8000/              # arka uç doğrudan cevap veriyor mu?
```
</details>

---

Önceki konu: [Bridge ve Bonding](08-bridge-ve-bonding.md) · Bölüm ana sayfası: [Ağ Yönetimi](README.md)
