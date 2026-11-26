# 🐧 LFCS Türkçe Notlar

![Ziyaret](https://komarev.com/ghpvc/?username=emiralp-cs-lfcs-turkce-notlar&label=ziyaret&color=blueviolet) ![Yıldız](https://img.shields.io/github/stars/Emiralp-cs/lfcs-turkce-notlar?style=social)

Linux Foundation Certified System Administrator (LFCS) sınavına hazırlananlar için Türkçe konu anlatımları ve uygulamalı alıştırmalar.

Ağustos 2026'da LFCS sınavını geçtim. Hazırlanırken Türkçe kaynağın çok az olduğunu gördüm, bu repoyu o eksiği biraz olsun kapatmak için hazırladım.

> [!TIP]
> Notlar işine yaradıysa sağ üstteki **⭐ Star** butonuna basarak destek olabilirsin. Yıldızlar, repoyu GitHub'da daha çok kişinin bulmasını sağlıyor.

## ✨ Bu repoda neler var

- Sınavın resmi konu başlıklarının **tamamını** kapsayan Türkçe anlatımlar. Her notun başında hangi resmi maddeye karşılık geldiği yazıyor.
- Ubuntu 24.04 üzerinde **denenmiş** komutlar ve gerçek çıktılar. Denenemeyen kısımlar her notta açıkça belirtildi.
- Her konunun sonunda **sık yapılan hatalar** ve **çözümlü alıştırmalar**
- [Sınav ipuçları](sinav-ipuclari.md): sınav ortamı, görev metnini okumak, doğrulama ve zaman yönetimi

## 📚 İçindekiler

| Bölüm | Sınav ağırlığı | Durum |
|---|---|---|
| [01 - Temel Komutlar](01-temel-komutlar/) | %20 | Tamamlandı |
| [02 - Kullanıcılar ve Gruplar](02-kullanicilar-ve-gruplar/) | %10 | Tamamlandı |
| [03 - İşletim ve Dağıtım](03-isletim-ve-dagitim/) | %25 | Tamamlandı |
| [04 - Depolama](04-depolama/) | %20 | Tamamlandı |
| [05 - Ağ Yönetimi](05-ag-yonetimi/) | %25 | Tamamlandı |

Bölümler öğrenme sırasına göre dizildi: önce her konuda kullanılan temel komutlar ve izinler, en sonda en karmaşık konular olan ağ yönetimi. Resmi sınav sayfası bölümleri ağırlığa göre sıralar (İşletim ve Dağıtım, Ağ Yönetimi, Depolama, Temel Komutlar, Kullanıcılar ve Gruplar). Konu başlıkları aynıdır, değişen sadece sıra.

## 🎯 Resmi müfredat

Sınavın güncel konu listesi, Linux Foundation'ın LFCS sayfasındaki **Domains & Competencies** bölümündedir. Hazırlanmaya başlamadan önce listeyi oku ve her maddeyi rahatça yapabildiğinden emin olana kadar çalış:

- [LFCS - The Linux Foundation](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)

Sınav belirli bir dağıtıma bağlı değildir. Bu repodaki bütün örnekler Ubuntu 24.04 ile hazırlandı.

## 📋 Linux Foundation belgeleri

Sınav kurallarını, ortamı ve teknik talimatları sınavdan önce mutlaka oku:

- [LFCS sınav talimatları](https://docs.linuxfoundation.org/tc-docs/certification/instructions-lfcs-and-lfce): Sınav ortamı, sunuculara bağlanma, yasak işlemler, kullanılabilecek belgeler
- [Sınav hazırlık kontrol listesi](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/exam-preparation-checklist): Kayıttan sonuca kadar adımlar (aday el kitabı)
- [Sınav kuralları ve politikaları](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/exam-rules-and-policies): Sınav sırasında nelere izin verildiği
- [Sertifika SSS](https://training.linuxfoundation.org/about/faqs/certification-faq/): Sık sorulan sorular

## 📖 Belgeler

Sınavda internet yoktur, ama sunucudaki `man` sayfaları ve `/usr/share/doc` altındaki belgeler kullanılabilir. Bunları sınavdan önce kullanmaya alış. Nasıl hızlı arama yapılacağı [Yardım Almak](01-temel-komutlar/01-yardim-almak.md) notunda ve [sınav ipuçlarında](sinav-ipuclari.md#7-sınavda-belge-bulmak).

Çalışırken işine yarayacak çevrim içi belgeler:

- [Ubuntu man sayfaları](https://manpages.ubuntu.com): Bütün man sayfaları, Ubuntu sürümüne göre
- [Ubuntu Server belgeleri](https://documentation.ubuntu.com/server): Ağ, depolama, servisler ve sanallaştırma için resmi rehberler

## 🧪 Deneme sınavı ve pratik

- [Killer.sh LFCS simülatörü](https://killer.sh/lfcs): Gerçek sınava benzeyen bir arayüzde 20 senaryo ve çözümleri. Linux Foundation'dan alınan LFCS sınavına [iki deneme hakkı dahildir](https://training.linuxfoundation.org/blog/exam-simulators/). Her deneme 36 saat açık kalır. Simülatör genelde gerçek sınavdan daha zor bulunur, puanın düşük çıkarsa moralini bozma.
- Bu repodaki her notun sonundaki alıştırmalar. Önce kendin çöz, sonra çözüme bak.

## 🖥️ Kendi laboratuvarını kur

Bütün notlar kendi bilgisayarında, bir sanal makinede denenebilir. Hiçbir bulut hesabı gerekmez.

[Multipass](https://canonical.com/multipass), Windows, macOS ve Linux'ta tek komutla Ubuntu sanal makinesi açar:

```bash
multipass launch 24.04 --name lfcs --cpus 2 --memory 4G --disk 20G
multipass shell lfcs
```

Disk ve LVM alıştırmaları için ek disk takmana gerek yok. Notlarda kullandığımız yöntemle, dosyadan sanal disk oluşturabilirsin:

```bash
truncate -s 1G ~/disk1.img
sudo losetup -f --show ~/disk1.img      # /dev/loop0 gibi bir cihaz verir
```

Ayrıntılar: [Sanal Dosya Sistemleri - loop cihazları](04-depolama/10-sanal-dosya-sistemleri.md#6-dosya-üzerinde-sanal-disk-loop-cihazları).

Ağ alıştırmalarında (SSH, NFS, ters proxy) birbirine bağlanan iki makine gerekir. Multipass ile ikinci bir makine açman yeterli: `multipass launch 24.04 --name lfcs2`.

## 🎓 Kurslar

- [Introduction to Linux (LFS101)](https://training.linuxfoundation.org/resources/free-courses/introduction-to-linux/): Linux Foundation'ın **ücretsiz** başlangıç kursu. Linux'a yeniysen buradan başla.
- [Linux System Administration Essentials (LFS207)](https://training.linuxfoundation.org/training/linux-system-administration-essentials-lfs207/): Linux Foundation'ın LFCS'ye hazırlayan resmi kursu
- [KodeKloud LFCS kursu](https://kodekloud.com/courses/linux-foundation-certified-system-administrator-lfcs): Tarayıcıda çalışan laboratuvarlarıyla uygulamalı bir kurs. Ben sınava bu kursla hazırlandım.

> [!TIP]
> Linux Foundation sınav ve kurslarında yıl içinde sık sık indirim kampanyası yapar. En büyük indirimler genelde Black Friday ve Cyber Monday döneminde olur. Acelen yoksa sınavı bir kampanya döneminde almak iyi bir fikir.

## 🛠️ Nasıl kullanılır

1. [Resmi müfredatı](#-resmi-müfredat) oku ve neyle karşılaşacağını gör.
2. Bölümleri sırayla çalış. Komutları okumak yerine kendi sanal makinende yaz, çok daha kalıcı oluyor.
3. Her notun sonundaki alıştırmaları çözüme bakmadan dene.
4. Sınavdan birkaç hafta önce [sınav ipuçlarını](sinav-ipuclari.md) oku ve Killer.sh simülatörünü çöz.

## 🤝 Katkı

Hata veya eksik görürsen issue açabilir ya da pull request gönderebilirsin.

## ⭐ Destek ol

Repo işine yaradıysa bir yıldız vermen, LFCS'ye hazırlanan başka birinin de onu bulmasına yardımcı olur. Paylaşmak da aynı şekilde.

## 📄 Lisans

[CC0 1.0](LICENSE): Bu notlar kamu malıdır. İzin almadan, kaynak göstermeden kopyalayabilir, değiştirebilir, paylaşabilir ve ticari dahil her amaçla kullanabilirsin. Kaynak göstermek zorunlu değil, ama yaparsan sevinirim.
