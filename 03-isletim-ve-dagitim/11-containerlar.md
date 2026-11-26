# Container'lar

> **Resmi sınav maddesi:** Configure container engines, create and manage containers

Bir uygulamayı normal kurduğunda dosyaları sisteme dağılır: program `/usr/bin`'de, ayarlar `/etc`'de, verisi `/var/lib`'de, bağımlılıkları sistemin kütüphanelerinde. **Container**, uygulamayı ihtiyaç duyduğu her şeyle birlikte paketleyip sistemin geri kalanından yalıtılmış çalıştırır. Aynı container her makinede aynı şekilde çalışır, silindiğinde arkasında iz bırakmaz.

| Kavram | Anlamı |
|---|---|
| **Image** | Container'ın şablonu: dosya sistemi ve çalışacak komut. Değişmez |
| **Container** | Bir image'dan oluşturulmuş, çalışan (veya durmuş) örnek. Aynı image'dan çok sayıda container oluşturulabilir |
| **Registry** | Image'ların saklandığı sunucu (Docker Hub, şirketin kendi registry'si) |
| **Dockerfile** | Kendi image'ını nasıl oluşturacağını tarif eden dosya |

Senaryo: Ekibin durum paneli için küçük bir web sunucusu image'ı hazırlayacağız, container olarak çalıştırıp porta bağlayacağız, içeriğini diskten vereceğiz ve sunucu yeniden başladığında kendiliğinden açılmasını sağlayacağız.

> [!NOTE]
> Örnekler Ubuntu 24.04 ve Docker 29 ile denenmiştir. Deneme ortamının ağ kısıtlaması Docker Hub'a erişime izin vermediği için `docker pull` ve `docker search` çalıştırılamadı. Yerine, Ubuntu'nun `busybox-static` paketinden küçük bir temel image oluşturulup (`docker import`) diğer bütün komutlar onunla denendi. Gerçek bir sistemde `nginx`, `ubuntu` gibi hazır image'ları Docker Hub'dan çekersin, komutlar aynıdır.

## İçindekiler

1. [Docker'ı kurmak ve yetki](#1-dockerı-kurmak-ve-yetki)
2. [Image bulmak ve indirmek](#2-image-bulmak-ve-indirmek)
3. [Container çalıştırmak: docker run](#3-container-çalıştırmak-docker-run)
4. [Port yayınlamak: -p](#4-port-yayınlamak--p)
5. [Container'ları yönetmek](#5-containerları-yönetmek)
6. [run ile start farkı](#6-run-ile-start-farkı)
7. [Container'ın içine bakmak: logs, exec, inspect](#7-containerın-içine-bakmak-logs-exec-inspect)
8. [Silmek: rm ve rmi](#8-silmek-rm-ve-rmi)
9. [Kalıcı veri: volume ve bind mount](#9-kalıcı-veri-volume-ve-bind-mount)
10. [Otomatik yeniden başlatma: --restart](#10-otomatik-yeniden-başlatma---restart)
11. [Kendi image'ını oluşturmak: Dockerfile](#11-kendi-imageını-oluşturmak-dockerfile)
12. [Podman](#12-podman)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Docker'ı kurmak ve yetki

Ubuntu deposundan:

```bash
sudo apt install docker.io
sudo systemctl enable --now docker
```

Docker'ın kendi deposundan en güncel sürüm (`docker-ce`) de kurulabilir: [Paket Depoları, Docker örneği](07-paket-depolari.md#6-örnek-docker-deposu).

Docker, arka planda çalışan `dockerd` servisiyle bir soket üzerinden konuşur. Normal kullanıcı bu sokete erişemez:

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

İki çözüm var: her komutun başına `sudo` veya kullanıcıyı `docker` grubuna eklemek:

```bash
sudo usermod -aG docker ayse      # yeniden giriş gerekir
```

> [!WARNING]
> `docker` grubundaki bir kullanıcı, kök dizini bir container'a bağlayarak sistemde istediği dosyayı okuyup yazabilir. Yani bu grup **root'a eşdeğerdir**. Sadece root yetkisi vereceğin kişileri ekle.

---

## 2. Image bulmak ve indirmek

```bash
docker search nginx          # Docker Hub'da ara
docker pull nginx            # en güncel sürüm (nginx:latest)
docker pull nginx:1.27       # belirli bir sürüm
docker images                # indirilmiş image'lar
```

`nginx:1.27`'deki `1.27` bir **etikettir** (tag). Etiket verilmezse `latest` kullanılır. `latest` "en yeni" anlamına gelmek zorunda değildir, sadece varsayılan etikettir. Üretimde belirli bir sürüm etiketi kullanmak, bir gün beklenmedik bir sürüme geçmeyi önler.

`docker run` ile kullanılan image yerelde yoksa önce kendisi indirir, `docker pull` yapmak şart değildir.

---

## 3. Container çalıştırmak: docker run

```bash
docker run nginx
```

Bu container'ı **ön planda** çalıştırır: terminale bağlı kalır, Ctrl+C container'ı da durdurur. Genelde arka planda, bir adla ve bir porta bağlı çalıştırılır:

```bash
docker run -d --name panel -p 8081:8080 panel:1.0
```

| Seçenek | Uzun hali | Anlamı |
|---|---|---|
| `-d` | `--detach` | Arka planda çalıştır |
| `--name panel` | | Container'a ad ver (verilmezse rastgele bir ad atanır) |
| `-p 8081:8080` | `--publish` | Port yayınla ([4. bölüm](#4-port-yayınlamak--p)) |
| `-e ORTAM=test` | `--env` | Ortam değişkeni |
| `-v kaynak:hedef` | `--volume` | Dizin bağla ([9. bölüm](#9-kalıcı-veri-volume-ve-bind-mount)) |
| `--restart` | | Yeniden başlatma politikası ([10. bölüm](#10-otomatik-yeniden-başlatma---restart)) |
| `--rm` | | Container bitince kendiliğinden silinsin |
| `-it` | | Etkileşimli terminal (`docker run -it ubuntu bash`) |

Image adından sonra yazılan her şey, image'ın varsayılan komutunun yerine çalışır:

```bash
docker run --rm yerel/busybox:1.0 echo "gecici konteyner"
docker run --rm -e ORTAM=test yerel/busybox:1.0 sh -c 'echo $ORTAM'
```
```
gecici konteyner
test
```

`--rm` sayesinde iki container da işi bitince silindi, `docker ps -a`'da görünmediler.

---

## 4. Port yayınlamak: -p

Container kendi yalıtılmış ağında çalışır. İçindeki web sunucusu 8080'i dinlese bile dışarıdan erişilemez. `-p` sunucunun bir portunu container'ın bir portuna bağlar:

```
-p 8081:8080
   │     │
   │     └── container'ın içindeki port
   └──────── sunucunun (host) portu
```

Sıra her zaman **host:container**.

```bash
docker ps
```
```
CONTAINER ID   IMAGE       COMMAND                  CREATED   STATUS    PORTS                    NAMES
89ad74b8455a   panel:1.0   "httpd -f -p 8080 -h…"   ...       Up ...    0.0.0.0:8081->8080/tcp   panel
```
```bash
curl http://127.0.0.1:8081/
```
```
<h1>Durum paneli</h1>
```

`curl` yoksa `nc localhost 8081` yazıp bağlantı açılınca `GET /` ve iki kez Enter da olur.

Aynı host portunu ikinci bir container'a vermeye çalışınca:

```
docker: Error response from daemon: ... Bind for 0.0.0.0:8081 failed: port is already allocated
```

Sadece bu makineden erişilsin istiyorsan: `-p 127.0.0.1:8081:8080`.

---

## 5. Container'ları yönetmek

```bash
docker ps                  # çalışanlar
docker ps -a               # durmuşlar dahil hepsi (--all)
docker stop panel          # durdur
docker start panel         # durmuş container'ı başlat
docker restart panel
```

`docker ps` sadece çalışanları gösterir. Durdurduğun bir container'ı arıyorsan `-a` gerekir:

```
CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS                                PORTS     NAMES
89ad74b8455a   panel:1.0   "httpd -f -p 8080 -h…"   11 seconds ago   Exited (137) Less than a second ago             panel
```

### Exited (137) ne demek?

`docker stop`, container'ın ana sürecine önce `SIGTERM` gönderir ve **10 saniye** bekler. Süreç kapanmazsa `SIGKILL` ile öldürür. Çıkış kodu `137` = 128 + 9 (SIGKILL), yani süreç düzgünce kapanmadı ve zorla öldürüldü. Denemedeki web sunucusu `SIGTERM`'e cevap vermediği için `stop` 10 saniye sürdü ve 137 ile bitti. Düzgün kapanan bir uygulamada `Exited (0)` görürsün. Sinyallerin ayrıntısı: [Süreçler](03-surecler.md#7-sinyaller-kill-ve-pkill).

Bekleme süresi değiştirilebilir: `docker stop -t 30 panel`.

---

## 6. run ile start farkı

| | `docker run` | `docker start` |
|---|---|---|
| Ne yapar? | Image'dan **yeni** bir container oluşturur ve başlatır | **Var olan**, durmuş bir container'ı başlatır |
| Argümanı | Image adı | Container adı veya ID'si |
| Ayarlar (`-p`, `-v`...) | Burada verilir | Container oluşturulurken verilenler kullanılır |

Durmuş bir container'ı tekrar çalıştırmak için `run` yazarsan yeni bir container oluşturulur. Aynı adı verirsen `Conflict. The container name "/panel" is already in use` hatası alırsın. Doğrusu `docker start panel`.

Bir container'ın ayarlarını (port, volume) değiştirmek için ise silip aynı image'dan yeni ayarlarla `run` gerekir. Container'ın ayarları oluşturulduktan sonra değişmez.

---

## 7. Container'ın içine bakmak: logs, exec, inspect

```bash
docker logs panel              # container'ın çıktısı
docker logs -f panel           # canlı izle
docker exec panel ls /www      # çalışan container'da komut çalıştır
docker exec -it panel sh       # içinde kabuk aç (exit ile çık)
docker inspect panel           # bütün ayrıntılar (JSON)
docker inspect panel --format '{{.HostConfig.RestartPolicy.Name}}'
```

`exec` sadece **çalışan** container'larda çalışır ve komutun container'ın içinde bulunması gerekir. Denemede `kill` komutu image'da olmadığı için `exec: "kill": executable file not found in $PATH` hatası verdi. Küçük image'larda `bash` olmayabilir, `sh` dene.

`docker inspect` ile container'ın IP'si, bağlı dizinleri, ortam değişkenleri, ana sürecin host'taki PID'i (`{{.State.Pid}}`) gibi her bilgi alınabilir.

---

## 8. Silmek: rm ve rmi

| Komut | Neyi siler? |
|---|---|
| `docker rm panel` | Container'ı |
| `docker rmi panel:1.0` | Image'ı (sondaki **i** = image) |

Çalışan bir container silinemez:

```
Error response from daemon: cannot remove container "panel": container is running: stop the container before removing or force remove
```

Önce `docker stop`, sonra `docker rm`, ya da tek adımda `docker rm -f panel`.

Bir container'ın kullandığı image da silinemez:

```
Error response from daemon: conflict: unable to delete panel:1.0 (must be forced) - container 89ad74b8455a is using its referenced image eb78b0a49e71
```

Bu kural **durmuş** container'lar için de geçerlidir: `docker ps -a`'da o image'dan oluşturulmuş bir container durdukça image silinmez. Sıra: `docker stop` → `docker rm` → `docker rmi`.

Kullanılmayan her şeyi temizlemek için: `docker system prune` (durmuş container'lar, kullanılmayan ağlar, etiketsiz image'lar). Silmeden önce ne gideceğini listeler ve onay ister.

---

## 9. Kalıcı veri: volume ve bind mount

Container silindiğinde içinde yazılan her şey de silinir. Kalıcı olması gereken veri (veritabanı dosyaları, yüklenen dosyalar, web içeriği) container'ın dışında tutulur:

```bash
docker run -d --name panel -p 8081:8080 -v /srv/panel-icerik:/www panel:1.0
echo '<h1>Diskten gelen icerik</h1>' > /srv/panel-icerik/index.html
curl http://127.0.0.1:8081/
```
```
<h1>Diskten gelen icerik</h1>
```

Host'taki `/srv/panel-icerik` dizini container'da `/www` olarak göründü. Dosyayı host'ta değiştirmek, container'ı yeniden oluşturmadan içeriği değiştirdi. Container silinse bile dosyalar host'ta kalır.

| Tür | Yazım | Veri nerede? |
|---|---|---|
| Bind mount | `-v /srv/veri:/www` (`/` ile başlayan yol) | Host'ta senin seçtiğin dizinde |
| Volume | `-v panel-veri:/www` (sadece ad) | Docker'ın yönettiği `/var/lib/docker/volumes/` altında |

```bash
docker volume create panel-veri
docker volume ls
docker volume rm panel-veri
```

Salt okunur bağlamak için sonuna `:ro`: `-v /srv/panel-icerik:/www:ro`.

---

## 10. Otomatik yeniden başlatma: --restart

Varsayılan olarak bir container çöktüğünde veya sunucu yeniden başladığında **kendiliğinden geri gelmez**.

| Politika | Ne zaman yeniden başlatır? |
|---|---|
| `no` (varsayılan) | Hiçbir zaman |
| `on-failure` | Hata koduyla çıkarsa |
| `always` | Her zaman. Elle durdurulmuş olsa bile Docker servisi yeniden başlayınca başlatır |
| `unless-stopped` | Her zaman, **elle durdurulmadıysa** |

```bash
docker run -d --name panel --restart unless-stopped -p 8081:8080 panel:1.0
```

Denemeler:

| Durum | `unless-stopped` | Politikasız |
|---|---|---|
| Container'ın süreci `kill -9` ile öldürüldü | Hemen yeniden başladı (`RestartCount` 1) | |
| Docker servisi yeniden başlatıldı | Geri geldi | |
| `docker stop` sonrası Docker yeniden başlatıldı | **Durmuş kaldı** | Durmuş kaldı |

Sunucu yeniden başladığında container'ların açılması için Docker servisinin de açılışta başlaması gerekir: `systemctl is-enabled docker`.

Var olan bir container'ın politikasını değiştirmek: `docker update --restart unless-stopped panel`.

---

## 11. Kendi image'ını oluşturmak: Dockerfile

Proje dizini:

```
panel-img/
├── Dockerfile
└── index.html
```

`Dockerfile`:

```dockerfile
FROM yerel/busybox:1.0
COPY index.html /www/index.html
EXPOSE 8080
CMD ["httpd", "-f", "-p", "8080", "-h", "/www"]
```

Gerçek bir sistemde ilk satır genelde `FROM nginx:1.27` gibi hazır bir image olur ve `COPY index.html /usr/share/nginx/html/` yeterli olur, çünkü nginx image'ının `CMD`'si zaten tanımlıdır.

| Talimat | Ne yapar? | Ne zaman çalışır? |
|---|---|---|
| `FROM` | Temel image | |
| `COPY kaynak hedef` | Proje dizininden image'a dosya kopyala | Build sırasında |
| `RUN komut` | Komut çalıştır, sonucu image'a kaydet (paket kurmak gibi) | Build sırasında |
| `WORKDIR /yol` | Sonraki komutların çalışma dizini | |
| `ENV AD=değer` | Ortam değişkeni | |
| `EXPOSE 8080` | Hangi portu kullandığını belgeler. **Portu yayınlamaz**, `-p` yine gerekir | |
| `CMD [...]` | Container başlarken çalışacak varsayılan komut. `docker run image komut` ile değiştirilir | Her container başlarken |
| `ENTRYPOINT [...]` | Container başlarken her zaman çalışacak komut. `docker run`'daki argümanlar ona eklenir | Her container başlarken |

```bash
docker build -t panel:1.0 panel-img/
```
```
#7 naming to docker.io/library/panel:1.0 done
```

`-t ad:etiket` image'a ad verir, sondaki dizin `Dockerfile`'ın bulunduğu yerdir (`.` = bulunduğun dizin). Dosyanın adı varsayılan olarak tam `Dockerfile` olmalı, başka bir ad için `-f dosya`.

```bash
docker images
```
```
IMAGE               ID             DISK USAGE   CONTENT SIZE
panel:1.0           eb78b0a49e71       3.32MB         1.15MB
yerel/busybox:1.0   7a6098459e77       3.31MB         1.15MB
```

Bir image'ı registry'ye göndermek için `docker tag` ve `docker push`, bir dosyaya kaydetmek için `docker save -o panel.tar panel:1.0`, başka bir makinede yüklemek için `docker load -i panel.tar`.

---

## 12. Podman

Resmi madde "container engines" der. Docker'ın yanında en yaygın alternatif **Podman**'dır (RHEL ailesinde varsayılan). Komutları neredeyse aynıdır (`podman run`, `podman ps`, `podman build`), farkları:

| | Docker | Podman |
|---|---|---|
| Arka plan servisi | `dockerd` gerekir | Yok, her komut kendi sürecini başlatır |
| Root olmadan | Ek kurulum gerekir | Varsayılan olarak root'suz çalışabilir |
| Açılışta başlatma | `--restart` politikası | systemd unit dosyası (Quadlet) |

```bash
sudo apt install podman
podman run -d --name panel -p 8081:80 docker.io/library/nginx
```

Resmi nginx image'ı container içinde 80. portu dinler, bu yüzden hedef port 80. Bu notun kendi `panel:1.0` image'ı ise 8080'i dinliyordu.

> [!NOTE]
> Podman bu ortamda denenmedi.

---

## 13. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `docker search ad`, `docker pull ad:etiket` | Ara, indir |
| `docker images` | Yerel image'lar |
| `docker run -d --name ad -p H:C image` | Arka planda çalıştır, port yayınla |
| `docker run --rm -it image sh` | Geçici, etkileşimli |
| `docker run -v /host:/cont image` | Dizin bağla |
| `docker run --restart unless-stopped ...` | Otomatik yeniden başlat |
| `docker ps`, `docker ps -a` | Çalışanlar, hepsi |
| `docker stop` / `start` / `restart ad` | Durdur / başlat / yeniden başlat |
| `docker logs -f ad` | Çıktısı |
| `docker exec -it ad sh` | İçinde kabuk |
| `docker inspect ad` | Ayrıntılar |
| `docker rm ad`, `docker rm -f ad` | Container sil |
| `docker rmi image` | Image sil |
| `docker build -t ad:etiket dizin` | Image oluştur |
| `docker update --restart ... ad` | Politikayı değiştir |
| `docker volume ls` / `create` / `rm` | Volume'lar |
| `docker system prune` | Kullanılmayanları temizle |

---

## 14. Sık yapılan hatalar

- **Durmuş container'ı tekrar `run` ile başlatmaya çalışmak.** Yeni container oluşur veya ad çakışır, `start` kullan.
- **`rm` ile `rmi`'yi karıştırmak.**
- **Port sırasını ters yazmak.** `-p host:container`.
- **`EXPOSE`'un portu yayınladığını sanmak.** `-p` gerekir.
- **`docker ps`'te göremediği için container'ın silindiğini sanmak.** `-a` ile bak.
- **Kalıcı veriyi container'ın içinde tutmak.** Container silinince gider, volume veya bind mount kullan.
- **Sunucu yeniden başlayınca container'ların açılmasını beklemek.** `--restart` gerekir.
- **Kullanıcıları düşünmeden `docker` grubuna eklemek.** Root yetkisi demektir.
- **Image'ı silmeden önce ondan oluşturulmuş durmuş container'ları silmemek.**

---

## 15. Alıştırmalar

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

Önceki konu: [SELinux](10-selinux.md) · Sonraki konu: [Sanal Makineler](12-sanal-makineler.md)
