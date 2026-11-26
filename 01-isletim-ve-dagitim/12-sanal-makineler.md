# Sanal Makineler

> **Resmi sınav maddesi:** Manage Virtual Machines (libvirt)

Bir sunucuda birden fazla işletim sistemini, birbirinden yalıtılmış sanal makineler (VM) olarak çalıştırmak için Linux'ta **KVM** (çekirdekteki sanallaştırma desteği), **QEMU** (donanımı taklit eden program) ve **libvirt** (bunları yöneten katman) birlikte kullanılır. libvirt'i komut satırından `virsh` ile yönetirsin, yeni makineyi `virt-install` ile kurarsın.

| | Container | Sanal makine |
|---|---|---|
| Çekirdek | Sunucunun çekirdeğini paylaşır | Kendi çekirdeği ve işletim sistemi var |
| Açılış | Saniyenin altında | Normal bir bilgisayar gibi açılır |
| Farklı işletim sistemi | Hayır (Linux üzerinde Linux) | Evet |
| Yalıtım | Süreç düzeyinde | Donanım düzeyinde |

Senaryo: Test ortamı için hazır bir Ubuntu bulut imajından sanal makine kuracağız, kaynaklarını değiştireceğiz, açılışta otomatik başlamasını sağlayacağız ve işimiz bitince diskiyle birlikte sileceğiz.

> [!NOTE]
> `virsh` ve `virt-install` komutları Ubuntu 24.04, libvirt 10.0 ve QEMU 8.2 ile denenmiştir. Deneme ortamında KVM yoktu, sanal makineler QEMU'nun yazılımsal emülasyonuyla (`--virt-type qemu`) çalıştırıldı, ve bulut imajı sitesine erişim engelliydi. Bu yüzden VM'ler boş bir diskle oluşturuldu: içlerinde işletim sistemi yoktu, ama tanımlama, başlatma, durdurma, kaynak değiştirme ve silme komutlarının hepsi gerçekten çalıştırıldı. İmaj indirme ve cloud-init kısımları denenmedi.

## İçindekiler

1. [Kurulum](#1-kurulum)
2. [Makineleri listelemek](#2-makineleri-listelemek)
3. [Disk imajları: qemu-img](#3-disk-imajları-qemu-img)
4. [Hazır bulut imajından VM: virt-install --import](#4-hazır-bulut-imajından-vm-virt-install---import)
5. [XML ile tanımlamak: virsh define](#5-xml-ile-tanımlamak-virsh-define)
6. [Başlatmak, durdurmak, yeniden başlatmak](#6-başlatmak-durdurmak-yeniden-başlatmak)
7. [Konsola bağlanmak](#7-konsola-bağlanmak)
8. [Açılışta başlatmak: autostart](#8-açılışta-başlatmak-autostart)
9. [CPU ve bellek değiştirmek](#9-cpu-ve-bellek-değiştirmek)
10. [Silmek: undefine](#10-silmek-undefine)
11. [Komut özeti](#11-komut-özeti)
12. [Sık yapılan hatalar](#12-sık-yapılan-hatalar)
13. [Alıştırmalar](#13-alıştırmalar)

---

## 1. Kurulum

```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst
sudo systemctl enable --now libvirtd
```

| Paket | İçerik |
|---|---|
| `qemu-kvm` (veya `qemu-system-x86`) | Sanal donanım |
| `libvirt-daemon-system` | `libvirtd` servisi |
| `libvirt-clients` | `virsh` |
| `virtinst` | `virt-install` |
| `virt-manager` | Grafik arayüz (sunucuda gerekmez) |

KVM için işlemcinin sanallaştırma desteği olmalı:

```bash
grep -c -E 'vmx|svm' /proc/cpuinfo     # 0'dan büyük olmalı
ls -l /dev/kvm
```

KVM yoksa QEMU yine çalışır ama her şeyi yazılımla taklit ettiği için çok yavaştır. Normal kullanıcının `sudo` olmadan VM yönetebilmesi için `libvirt` grubuna eklenmesi gerekir: `sudo usermod -aG libvirt ayse`.

---

## 2. Makineleri listelemek

```bash
virsh list           # sadece çalışanlar
virsh list --all     # kapalılar dahil
```
```
 Id   Name     State
-------------------------
 -    deneme   shut off
```

`virsh list` tek başına boş görünebilir, tanımlı ama kapalı makineler sadece `--all` ile görünür.

| Durum | Anlamı |
|---|---|
| `running` | Çalışıyor |
| `shut off` | Kapalı |
| `paused` | Duraklatılmış (`virsh suspend`) |

```bash
virsh dominfo deneme
```
```
State:          running
CPU(s):         1
Max memory:     262144 KiB
Used memory:    262144 KiB
Persistent:     yes
Autostart:      disable
```

`virsh`, root olarak çalıştırıldığında sistem genelindeki makineleri (`qemu:///system`) yönetir. Normal kullanıcı `sudo` olmadan çalıştırırsa kendi oturumuna ait, ayrı bir listeye (`qemu:///session`) bakar ve sistemdeki makineleri göremez. Makinelerin "kaybolduğunu" düşünürsen `sudo virsh list --all` dene.

---

## 3. Disk imajları: qemu-img

Sanal makinelerin diskleri genelde **qcow2** biçiminde dosyalardır. Varsayılan yerleri `/var/lib/libvirt/images/`.

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/deneme.qcow2 2G
sudo qemu-img info /var/lib/libvirt/images/deneme.qcow2
```
```
image: /var/lib/libvirt/images/deneme.qcow2
file format: qcow2
virtual size: 2 GiB (2147483648 bytes)
disk size: 196 KiB
```

**virtual size**, VM'in gördüğü disk boyutu. **disk size** ise dosyanın sunucuda gerçekten kapladığı yer. qcow2 "ince" (thin) ayrılır: VM diske yazdıkça büyür. `ls -lh` de dosyayı 193K gösterdi.

Büyütmek:

```bash
sudo qemu-img resize /var/lib/libvirt/images/deneme.qcow2 5G
```
```
Image resized.
```

Sanal boyut 5 GiB oldu, dosya hala 200 KiB. Büyütme VM kapalıyken yapılmalı, sonrasında VM'in içinde bölümün ve dosya sisteminin de büyütülmesi gerekir. Küçültmek veri kaybına yol açabilir.

libvirt bu dizini bir **storage pool** olarak yönetir:

```bash
virsh pool-list --all
virsh vol-list images      # pool'daki diskler
```

---

## 4. Hazır bulut imajından VM: virt-install --import

Bir işletim sistemini sıfırdan kurmak yerine, önceden kurulmuş **bulut imajını** (cloud image) kullanmak en hızlı yoldur.

**1. İmajı indir:**

```bash
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
```

**2. Bozulmadan indiğini doğrula:** Yayıncının verdiği özet dosyasıyla karşılaştır.

```bash
wget https://cloud-images.ubuntu.com/minimal/releases/noble/release/SHA256SUMS
sha256sum -c SHA256SUMS 2>&1 | grep OK
```

`ubuntu-24.04-minimal-cloudimg-amd64.img: OK` görmelisin. `sha256sum -c`, dosyadaki bütün satırları kontrol eder, indirmediğin dosyalar için hata verir, bu yüzden `grep OK`.

**3. İncele, büyüt, yerine kopyala:**

```bash
qemu-img info ubuntu-24.04-minimal-cloudimg-amd64.img
qemu-img resize ubuntu-24.04-minimal-cloudimg-amd64.img 10G
sudo cp ubuntu-24.04-minimal-cloudimg-amd64.img /var/lib/libvirt/images/ubuntu1.qcow2
```

Bulut imajlarının sanal boyutu küçüktür (birkaç GB), içine bir şey kurmadan önce büyütmek gerekir. Gerçek boyutu `qemu-img info` çıktısındaki `virtual size` satırı gösterir.

**4. VM'i oluştur:**

```bash
sudo virt-install --osinfo ubuntu24.04 --name ubuntu1 --memory 1024 --vcpus 1 \
  --import --disk /var/lib/libvirt/images/ubuntu1.qcow2 --graphics none
```

| Seçenek | Anlamı |
|---|---|
| `--osinfo ubuntu24.04` | İşletim sistemi türü, libvirt uygun sanal donanımı seçer |
| `--name` | VM'in adı |
| `--memory 1024` | Bellek (MiB) |
| `--vcpus 1` | Sanal işlemci sayısı |
| `--import` | **Kurulum yapma**, diskte zaten kurulu bir sistem var |
| `--disk yol` | Disk |
| `--graphics none` | Grafik ekran yok, seri konsol kullan |
| `--noautoconsole` | Oluşturduktan sonra konsola bağlanma, komut satırına dön |

Denemede çıktı:

```
WARNING  Requested memory 1024 MiB is less than the recommended 3072 MiB for OS ubuntu24.04

Starting install...
Creating domain...
Domain creation completed.
```

Uyarı engel değildir, test için yeterli.

> [!IMPORTANT]
> `--import` unutulursa virt-install bir kurulum ortamı (ISO) arar. `--import` yalnızca diskte hazır bir sistem varsa kullanılır.

### --osinfo verilmezse

```
ERROR
--os-variant/--osinfo OS name is required, but no value was ...
   virt-install --osinfo list
```

Geçerli adları görmek için `virt-install --osinfo list | grep ubuntu` (`ubuntu24.04`, `ubuntu22.04` ...). Listede olmayan bir sistem için `--osinfo detect=on,require=off` veya `--osinfo generic`.

### Ağ

virt-install varsayılan olarak libvirt'in `default` ağını kullanır (NAT'lı, `virbr0` köprüsü). Bu ağ çalışmıyorsa:

```
ERROR    Requested operation is not valid: network 'default' is not active
```

```bash
sudo virsh net-list --all
sudo virsh net-start default
sudo virsh net-autostart default
```

### cloud-init ile ilk parola

Bulut imajlarında hazır bir parola yoktur. virt-install, cloud-init aracılığıyla rastgele bir root parolası üretebilir:

```bash
sudo virt-install --osinfo ubuntu24.04 --name ubuntu1 --memory 1024 --vcpus 1 \
  --import --disk /var/lib/libvirt/images/ubuntu1.qcow2 --graphics none \
  --cloud-init root-password-generate=on
```

`man virt-install`'a göre üretilen parola, komutu çalıştırdığın terminale bir kez yazılır ve fark etmen için 10 saniye beklenir. Not al. Sadece `--cloud-init` yazmak `root-password-generate=on,disable=on` ile aynıdır: `disable=on`, sonraki açılışlarda cloud-init'i kapatır, yoksa cloud-init giriş ayarlarını her açılışta sıfırlayabilir. Kendi parolanı vermek için `root-password-file=dosya`, SSH anahtarı için `root-ssh-key=dosya`.

> [!NOTE]
> İmaj indirme ve cloud-init adımları, deneme ortamı bulut imajı sitesine erişemediği için denenmedi. Komutlar `man virt-install` ve Ubuntu'nun imaj sayfasına göre yazıldı.

---

## 5. XML ile tanımlamak: virsh define

libvirt her VM'i bir XML dosyasıyla tanımlar. virt-install bu dosyayı senin için oluşturur, ama elle de yazılabilir:

```xml
<domain type="kvm">
  <name>deneme</name>
  <memory unit="MiB">256</memory>
  <vcpu>1</vcpu>
  <os><type arch="x86_64">hvm</type></os>
  <devices>
    <disk type="file" device="disk">
      <source file="/var/lib/libvirt/images/deneme.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <driver name="qemu" type="qcow2"/>
    </disk>
    <graphics type="vnc"/>
  </devices>
</domain>
```

KVM yoksa `type="qemu"` yazılır.

```bash
sudo virsh define deneme.xml
```
```
Domain 'deneme' defined from deneme.xml
```

`define`, makineyi kalıcı olarak **tanımlar ama başlatmaz**. Tanımlar `/etc/libvirt/qemu/deneme.xml`'de saklanır.

```bash
sudo virsh dumpxml deneme     # geçerli tanımı göster
sudo virsh edit deneme        # tanımı düzenle (sonraki açılışta geçerli)
```

`/etc/libvirt/qemu/` altındaki dosyaları elle düzenleme, `virsh edit` kullan: hem doğrular hem libvirt'e haber verir.

---

## 6. Başlatmak, durdurmak, yeniden başlatmak

```bash
sudo virsh start deneme
sudo virsh shutdown deneme     # düzgün kapat
sudo virsh reboot deneme       # düzgün yeniden başlat
sudo virsh destroy deneme      # fişi çek
sudo virsh reset deneme        # reset tuşuna bas
sudo virsh suspend deneme      # duraklat
sudo virsh resume deneme       # devam ettir
```

| Komut | Nasıl? | Benzetme |
|---|---|---|
| `shutdown` | VM'in işletim sistemine "kapan" sinyali gönderir (ACPI) | Güç tuşuna kısa basmak |
| `destroy` | VM'i anında durdurur | Fişi çekmek |
| `reboot` | İşletim sistemine "yeniden başla" der | |
| `reset` | Anında yeniden başlatır | Reset tuşu |

`shutdown` ve `reboot`, VM'in içindeki işletim sisteminin bu sinyali anlamasına bağlıdır. Denemedeki VM'de işletim sistemi olmadığı için:

```
Domain 'deneme' is being shutdown
```

dedi, ama 5 saniye sonra hala `running` durumdaydı. `destroy` ile anında kapandı:

```
Domain 'deneme' destroyed
```

Gerçek bir sistemde de takılan veya cevap vermeyen bir VM'de durum aynıdır: önce `shutdown`, bir süre bekle, kapanmazsa `destroy`. `destroy`, adına rağmen VM'i silmez, sadece kapatır. Kaydedilmemiş veriler kaybolabilir.

---

## 7. Konsola bağlanmak

`--graphics none` ile oluşturulan VM'lere seri konsoldan bağlanılır:

```bash
sudo virsh console ubuntu1
```

Bağlanınca bir şey görünmezse Enter'a bas. **Çıkmak için `Ctrl+]`.** Bu tuş VM'i kapatmaz, sadece konsoldan ayrılırsın.

---

## 8. Açılışta başlatmak: autostart

Sunucu yeniden başladığında VM'in de açılması için:

```bash
sudo virsh autostart deneme
sudo virsh dominfo deneme | grep Autostart
```
```
Domain 'deneme' marked as autostarted
Autostart:      enable
```

Bu, `/etc/libvirt/qemu/autostart/` altında bir bağlantı oluşturur:

```
deneme.xml -> /etc/libvirt/qemu/deneme.xml
```

Kapatmak: `sudo virsh autostart --disable deneme`. `libvirtd` servisinin de açılışta başlıyor olması gerekir.

---

## 9. CPU ve bellek değiştirmek

### İşlemci

```bash
sudo virsh setvcpus deneme 2 --config
```
```
error: invalid argument: requested vcpus is greater than max allowable vcpus for the persistent domain: 2 > 1
```

Her VM'in bir **en fazla** işlemci sayısı vardır. Önce onu artırmak gerekir:

```bash
sudo virsh setvcpus deneme 2 --config --maximum
sudo virsh setvcpus deneme 2 --config
sudo virsh dominfo deneme | grep 'CPU(s)'
```
```
CPU(s):         2
```

### Bellek

Aynı mantık:

```bash
sudo virsh setmem deneme 512M --config
```
```
error: invalid argument: cannot set memory higher than max memory
```
```bash
sudo virsh setmaxmem deneme 512M --config
sudo virsh setmem deneme 512M --config
```
```
Max memory:     524288 KiB
Used memory:    524288 KiB
```

| Seçenek | Ne zaman etkili? |
|---|---|
| `--config` | VM bir sonraki kez **kapatılıp açıldığında** (VM'in içinden yapılan reboot yetmeyebilir) |
| `--live` | Hemen, çalışan VM'de. İşletim sistemi ve sanal donanım desteklemezse hata verir |

Denemede çalışan VM'de `setvcpus deneme 1 --live` hata verdi (`failed to find appropriate hotpluggable vcpus`). Sınavda güvenli yol: `--config` ile değiştir, sonra `virsh shutdown` (gerekirse `destroy`) ve `virsh start`. Kontrol için `virsh dumpxml deneme | grep -E '<memory|<vcpu'`.

---

## 10. Silmek: undefine

```bash
sudo virsh destroy ubuntu1                        # çalışıyorsa önce kapat
sudo virsh undefine ubuntu1 --remove-all-storage
```
```
Domain 'ubuntu1' has been undefined
Volume 'vda'(/var/lib/libvirt/images/ubuntu1.qcow2) removed.
```

| Komut | VM tanımı | Disk dosyası |
|---|---|---|
| `virsh undefine ad` | Silinir | **Kalır** |
| `virsh undefine ad --remove-all-storage` | Silinir | Silinir |

`--remove-all-storage` olmadan disk dosyası `/var/lib/libvirt/images/` altında kalır ve yer kaplamaya devam eder. Denemede `deneme` makinesini bu seçenek olmadan sildik, `deneme.qcow2` yerinde kaldı.

| İstediğin | Komut |
|---|---|
| Kapat | `virsh shutdown` / `destroy` |
| Tanımı sil, disk kalsın | `virsh undefine` |
| Tamamen sil | `virsh undefine --remove-all-storage` |

---

## 11. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `virsh list --all` | Bütün VM'ler |
| `virsh dominfo ad` | Durum, CPU, bellek, autostart |
| `qemu-img create -f qcow2 dosya 10G` | Disk oluştur |
| `qemu-img info dosya` | Disk bilgisi |
| `qemu-img resize dosya 10G` | Diski büyüt |
| `sha256sum -c SHA256SUMS` | İndirilen imajı doğrula |
| `virt-install --osinfo X --name ad --memory M --vcpus N --import --disk yol --graphics none` | Hazır imajdan VM oluştur |
| `virt-install --osinfo list` | İşletim sistemi adları |
| `--cloud-init root-password-generate=on` | Rastgele root parolası |
| `virsh define dosya.xml` | XML'den tanımla |
| `virsh dumpxml ad`, `virsh edit ad` | Tanımı göster / düzenle |
| `virsh start` / `shutdown` / `reboot` | Başlat / kapat / yeniden başlat |
| `virsh destroy` / `reset` | Zorla kapat / yeniden başlat |
| `virsh suspend` / `resume` | Duraklat / devam |
| `virsh console ad` | Konsol (çıkış: `Ctrl+]`) |
| `virsh autostart [--disable] ad` | Açılışta başlat |
| `virsh setvcpus ad N --config --maximum` | En fazla CPU |
| `virsh setvcpus ad N --config` | CPU sayısı |
| `virsh setmaxmem ad 2G --config` | En fazla bellek |
| `virsh setmem ad 2G --config` | Bellek |
| `virsh undefine ad --remove-all-storage` | Diskiyle birlikte sil |
| `virsh net-list --all`, `net-start default` | Sanal ağlar |

---

## 12. Sık yapılan hatalar

- **`virsh list`'te göremediği için VM'in silindiğini sanmak.** `--all` ile bak, `sudo` ile çalıştır.
- **Hazır imajla kurarken `--import`'u unutmak.**
- **İndirilen imajı doğrulamamak.**
- **İşlemci veya belleği en fazla değerin üstüne çıkarmaya çalışmak.** Önce `--maximum` / `setmaxmem`.
- **`--config` ile değiştirip VM'i kapatıp açmadan sonucu beklemek.**
- **`destroy`'un VM'i sildiğini sanmak.** Sadece zorla kapatır.
- **`undefine`'ın diski de sildiğini sanmak.** `--remove-all-storage` gerekir.
- **Konsoldan çıkmak için Ctrl+C veya `exit` kullanmak.** `Ctrl+]`.
- **`/etc/libvirt/qemu/*.xml` dosyalarını elle düzenlemek.** `virsh edit` kullan.

---

## 13. Alıştırmalar

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

Önceki konu: [Container'lar](11-containerlar.md) · Sonraki konu: [Bash Script Temelleri](13-bash-script.md)
