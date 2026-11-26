# Git ile Sürüm Kontrolü

Git, dosyalardaki değişiklikleri kaydeden bir **sürüm kontrol sistemi**dir. Her kayıtta (commit) kimin, ne zaman, neyi, neden değiştirdiği saklanır. Bir şey bozulduğunda eski bir hale dönebilir, farklı fikirleri ayrı dallarda deneyebilir, aynı dosyalar üzerinde birden fazla kişi birlikte çalışabilirsiniz. Sadece kod için değil, sunucu yapılandırmaları ve script'ler için de kullanılır.

Git **dağıtık** çalışır: herkesin bilgisayarında projenin tüm geçmişiyle birlikte tam bir kopyası bulunur. Ortak bir sunucudaki (GitHub, GitLab veya şirketin kendi sunucusu) **uzak repo**, kopyaların birbirleriyle eşitlendiği buluşma noktasıdır.

Senaryo: Sistem ekibi sunucu ayar dosyalarını ve yedekleme script'lerini `ayarlar` adında bir repoda tutmaya karar veriyor. Ayşe repoyu oluşturup ortak sunucuya gönderecek, Mert kopyalayıp üzerinde çalışacak, ikisi aynı satırı değiştirince çıkan çakışmayı çözeceğiz.

> [!NOTE]
> Bütün örnekler Ubuntu 24.04 ve git 2.43 ile denenmiştir. Uzak repo olarak aynı makinede bir dizin kullanıldı. GitHub veya SSH üzerindeki bir repoyla komutlar aynıdır, sadece adres değişir. Commit numaraları (hash) her seferinde farklı üretilir.

## İçindekiler

1. [Kurulum ve kimlik](#1-kurulum-ve-kimlik)
2. [Repo oluşturmak: git init](#2-repo-oluşturmak-git-init)
3. [Üç alan: çalışma dizini, hazırlık alanı, repo](#3-üç-alan-çalışma-dizini-hazırlık-alanı-repo)
4. [Durum, ekleme ve kayıt: status, add, commit](#4-durum-ekleme-ve-kayıt-status-add-commit)
5. [Farkları görmek: git diff](#5-farkları-görmek-git-diff)
6. [Geçmiş: git log ve git show](#6-geçmiş-git-log-ve-git-show)
7. [Değişikliği geri almak: git restore](#7-değişikliği-geri-almak-git-restore)
8. [Takip edilmeyecek dosyalar: .gitignore](#8-takip-edilmeyecek-dosyalar-gitignore)
9. [Dallar: branch, switch, merge](#9-dallar-branch-switch-merge)
10. [Uzak repo: remote, push, clone](#10-uzak-repo-remote-push-clone)
11. [Başkalarının değişikliklerini almak: fetch ve pull](#11-başkalarının-değişikliklerini-almak-fetch-ve-pull)
12. [Çakışma çözmek](#12-çakışma-çözmek)
13. [Komut özeti](#13-komut-özeti)
14. [Sık yapılan hatalar](#14-sık-yapılan-hatalar)
15. [Alıştırmalar](#15-alıştırmalar)

---

## 1. Kurulum ve kimlik

Ubuntu'da genelde yüklü gelir, değilse:

```bash
sudo apt install git
git --version     # git version 2.43.0
```

### Kimlik

Her commit'e yazarın adı ve e-postası yazılır. Bunlar ayarlanmadan commit yapmaya çalışınca git reddetti:

```
Author identity unknown

*** Please tell me who you are.

Run

  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
```

Kurulumdan sonraki ilk iş:

```bash
git config --global user.name "Ayse Yilmaz"
git config --global user.email "ayse@sirket.lan"
```

### --global ile repo ayarı farkı

| Komut | Nereye yazılır? | Neyi etkiler? |
|---|---|---|
| `git config --global ...` | `~/.gitconfig` | Bu kullanıcının bütün repolarını |
| `git config ...` (repo içinde) | `.git/config` | Sadece o repoyu |

Repo ayarı genel ayarı ezer. Denemede `~/.gitconfig`'te `ayse@sirket.lan` varken bir repoda `git config user.email "ayse.ozel@ornek.lan"` yazdık, o repoda bu adres kullanıldı. `--unset` ile kaldırınca tekrar genel adrese döndü.

Hangi ayarın nereden geldiğini görmek için:

```bash
git config --list --show-origin
```
```
file:/home/ayse/.gitconfig	user.name=Ayse Yilmaz
file:/home/ayse/.gitconfig	user.email=ayse@sirket.lan
```

### Varsayılan dal adı

`git init` ilk dalı `master` adıyla oluşturur ve bununla ilgili uzun bir ipucu basar. GitHub ve çoğu yeni proje `main` kullanır. Bir kez ayarlamak yeter:

```bash
git config --global init.defaultBranch main
```

---

## 2. Repo oluşturmak: git init

```bash
mkdir ayarlar
cd ayarlar
git init
```
```
Initialized empty Git repository in /home/ayse/ayarlar/.git/
```

`git init ayarlar` yazarsan dizini de kendisi oluşturur.

Git, repoya ait her şeyi (bütün geçmiş, dallar, ayarlar) gizli `.git` dizininde tutar:

```bash
ls .git
```
```
HEAD  branches  config  description  hooks  info  objects  refs
```

> [!CAUTION]
> `.git` dizinini silersen bütün geçmiş gider, geriye sadece dosyaların son hali kalır. Bir dizinin repo olmaktan çıkması için tek gereken de budur. Uzak repoya gönderilmemiş commit'ler kalıcı olarak kaybolur.

---

## 3. Üç alan: çalışma dizini, hazırlık alanı, repo

Git'i anlamanın anahtarı, bir değişikliğin üç yerden geçtiğini bilmek:

```
çalışma dizini  ──git add──>  hazırlık alanı  ──git commit──>  repo (geçmiş)
(dosyaları         (staging,           (kalıcı kayıt)
 düzenlediğin yer)  bir sonraki commit'e
                    girecekler)
```

Git dosyalardaki değişiklikleri **kendiliğinden kaydetmez.** Bir dosyayı değiştirmek, onu geçmişe eklemez. Önce `git add` ile hazırlık alanına koyar, sonra `git commit` ile kaydedersin. Bu ara adım sayesinde, değiştirdiğin beş dosyadan sadece ikisini bir commit'e, kalanları başka bir commit'e koyabilirsin.

---

## 4. Durum, ekleme ve kayıt: status, add, commit

İki dosya oluşturduk:

```bash
printf '#!/bin/bash\ntar -czf /yedek/site.tar.gz /var/www/site\n' > yedekle.sh
printf 'server_name panel.sirket.lan;\nlisten 443 ssl;\n' > panel.conf
git status
```
```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	panel.conf
	yedekle.sh
```

`Untracked`: git bu dosyaları görüyor ama takip etmiyor. `git status -s` daha kısa bir çıktı verir. Sol sütun hazırlık alanını, sağ sütun çalışma dizinini gösterir:

| Adım | `git status -s` |
|---|---|
| Dosyalar yeni oluşturuldu | `?? panel.conf`<br>`?? yedekle.sh` |
| `git add yedekle.sh` | `A  yedekle.sh`<br>`?? panel.conf` |
| `git add .` | `A  panel.conf`<br>`A  yedekle.sh` |
| Commit'ten sonra `panel.conf` değişti | ` M panel.conf` |
| `git add panel.conf` | `M  panel.conf` |

`??` takip edilmiyor, `A` eklendi, `M` değişti. Harfin solda mı sağda mı olduğu, değişikliğin hazırlık alanında mı yoksa sadece çalışma dizininde mi olduğunu gösterir.

`git add .` bulunduğun dizindeki bütün değişiklikleri ekler.

### Commit

```bash
git commit -m "Yedekleme scripti ve panel ayari eklendi"
```
```
[main (root-commit) 06bc982] Yedekleme scripti ve panel ayari eklendi
 2 files changed, 4 insertions(+)
 create mode 100644 panel.conf
 create mode 100644 yedekle.sh
```

`-m` olmadan yazarsan git, mesajı yazman için bir metin düzenleyici açar. Mesaj, değişikliğin **neden** yapıldığını anlatmalı. "düzeltme" gibi mesajlar altı ay sonra hiçbir şey ifade etmez.

### git commit -a

`-a`, takip edilen ve değişmiş dosyaları `git add` yapmadan doğrudan commit'e katar:

```bash
git commit -am "gzip ayari eklendi"
```

Ama **yeni dosyaları eklemez.** Yeni bir `notlar.txt` varken `git commit -am` denediğimizde `nothing added to commit but untracked files present` dedi. Yeni dosyalar için `git add` şart.

### Dosya silmek ve taşımak

```bash
git rm eski.conf                       # sil ve silmeyi hazırla
git mv panel.conf nginx-panel.conf     # yeniden adlandır
git status -s                          # R  panel.conf -> nginx-panel.conf
```

`R`, yeniden adlandırma (rename) demek.

---

## 5. Farkları görmek: git diff

`panel.conf`'a bir satır ekledik:

```bash
git diff
```
```diff
diff --git a/panel.conf b/panel.conf
index 2b77ca6..06db681 100644
--- a/panel.conf
+++ b/panel.conf
@@ -1,2 +1,3 @@
 server_name panel.sirket.lan;
 listen 443 ssl;
+gzip_level=6
```

`+` ile başlayan satır eklendi, `-` ile başlayan silindi. Biçim, [Metin İşleme](06-metin-isleme.md) dosyasındaki `diff -u` ile aynıdır.

| Komut | Neyi karşılaştırır? |
|---|---|
| `git diff` | Çalışma dizini ile hazırlık alanı (henüz eklenmemiş değişiklikler) |
| `git diff --staged` | Hazırlık alanı ile son commit (commit'e girecek olanlar) |
| `git diff HEAD~1` | Çalışma dizini ile bir önceki commit |

Değişikliği `git add` ile ekledikten sonra `git diff` hiçbir şey göstermedi, `git diff --staged` gösterdi. Commit'ten önce `git diff --staged` ile neyi kaydettiğine bakmak iyi bir alışkanlıktır.

---

## 6. Geçmiş: git log ve git show

```bash
git log
```
```
commit 1b6f92e08ae3e36ce6fcf8fc53f934877f4a1227
Author: Ayse Yilmaz <ayse@sirket.lan>
Date:   Thu Nov 26 08:16:10 2026 +0000

    gzip ayari eklendi

commit 06bc9825d5c7c7f642f5fc9c6a474766f3d5966b
Author: Ayse Yilmaz <ayse@sirket.lan>
Date:   Thu Nov 26 08:16:10 2026 +0000

    Yedekleme scripti ve panel ayari eklendi
```

Her commit'in uzun bir kimliği (hash) vardır. Komutlarda ilk 7 karakteri yeterlidir.

| Komut | Ne gösterir? |
|---|---|
| `git log --oneline` | Her commit tek satırda |
| `git log -3` | Son 3 commit |
| `git log --oneline --graph` | Dalları çizgilerle |
| `git log --author=Mert` | Belirli bir kişinin commit'leri |
| `git log -- yedekle.sh` | Sadece bu dosyayı değiştiren commit'ler |
| `git log -p` | Her commit'in farkıyla birlikte |
| `git show 1b6f92e` | Bir commit'in ayrıntısı ve farkı |
| `git show --stat HEAD` | Son commit'te hangi dosyalar değişti |

`HEAD`, şu an üzerinde bulunduğun commit'i gösterir. `HEAD~1` bir öncekini, `HEAD~2` iki öncekini.

---

## 7. Değişikliği geri almak: git restore

| Durum | Komut |
|---|---|
| Dosyayı bozdun, henüz `add` yapmadın, son commit'teki haline dönmek istiyorsun | `git restore dosya` |
| `add` yaptın, hazırlık alanından çıkarmak istiyorsun (değişiklik dosyada kalır) | `git restore --staged dosya` |

Denemede `panel.conf`'un içine `YANLIS` yazıp `git restore panel.conf` dedik, dosya son commit'teki haline döndü.

> [!WARNING]
> `git restore dosya`, dosyadaki commit edilmemiş değişiklikleri **geri dönüşsüz** siler. Git onları hiç kaydetmediği için kurtaramaz.

Eski kaynaklarda aynı işler için `git checkout -- dosya` ve `git reset HEAD dosya` görürsün. Hala çalışırlar, `git restore` daha yeni ve daha açık bir komuttur.

---

## 8. Takip edilmeyecek dosyalar: .gitignore

Log dosyaları, geçici dosyalar ve özellikle **şifre veya anahtar içeren dosyalar** repoya girmemeli. Repo kökünde bir `.gitignore` dosyası oluştur:

```bash
cat > .gitignore <<EOF
*.log
sifreler.txt
EOF
```

Sonra `hata.log` ve `sifreler.txt` oluşturduk, `git status -s` sadece `?? .gitignore` gösterdi. İki dosya da görmezden gelindi. `.gitignore`'un kendisi ise repoya eklenir, böylece bütün ekip aynı kuralları kullanır.

> [!IMPORTANT]
> `.gitignore`, sadece henüz takip edilmeyen dosyaları etkiler. Bir dosya bir kez commit edildiyse sonradan `.gitignore`'a eklemek onu geçmişten silmez. Bir parola yanlışlıkla commit edilip uzak repoya gönderildiyse, onu değiştirmek gerekir.

---

## 9. Dallar: branch, switch, merge

Bir **dal** (branch), geçmişte ayrı bir çizgidir. Ana dalı (`main`) bozmadan bir değişikliği denemek için yeni bir dal açar, orada çalışır, hazır olunca ana dala birleştirirsin.

```bash
git branch                      # dalları listele, * olduğun dal
git switch -c gzip-kapat        # yeni dal oluştur ve ona geç
```

Yeni dalda bir değişiklik yapıp commit ettik. Sonra ana dala döndük:

```bash
git switch main
grep gzip nginx-panel.conf      # gzip_level=6, değişiklik burada yok
```

Her dal kendi dosya halini taşır. Dal değiştirince çalışma dizinindeki dosyalar da o dalın haline geçer.

### Birleştirmek: git merge

Değişikliği ana dala almak için önce ana dala geç, sonra diğer dalı birleştir:

```bash
git switch main
git merge gzip-kapat
```
```
Updating a60097a..46143a6
Fast-forward
 nginx-panel.conf | 2 +-
```

**Fast-forward**: `main` dalı ayrıldığından beri hiç değişmemişti, git sadece `main`'i ileri taşıdı, yeni bir commit oluşturmadı.

Her iki dalda da ayrı commit'ler varsa git ikisini birleştiren yeni bir **merge commit** oluşturur:

```
Merge made by the 'ort' strategy.
 kontrol.sh | 1 +
```
```bash
git log --oneline --graph -4
```
```
*   28ac8a4 yeni-script birlestirildi
|\  
| * 5f48655 kontrol scripti
* | 0565be5 yedekleme notu
|/  
```

İki dal farklı dosyalarda veya aynı dosyanın farklı yerlerinde değişiklik yaptıysa git bunları kendisi birleştirir. Aynı satırı değiştirdilerse çakışma olur ([12. bölüm](#12-çakışma-çözmek)).

### Dal silmek

```bash
git branch -d gzip-kapat
```

Birleştirilmemiş bir dalı silmeye çalışınca git korur:

```
error: the branch 'yarim' is not fully merged.
If you are sure you want to delete it, run 'git branch -D yarim'
```

`-D` (büyük harf) zorla siler, o daldaki commit'ler kaybolur.

Eski kaynaklarda dal değiştirmek için `git checkout dal` ve `git checkout -b yeni-dal` görürsün. Bunlar da çalışır, `git switch` sadece dallarla ilgili daha yeni bir komuttur.

---

## 10. Uzak repo: remote, push, clone

### Uzak repoyu oluşturmak

GitHub'da bir repo açmak buna denk gelir. Kendi sunucunda ise çalışma dizini olmayan, sadece geçmişi tutan bir **bare** repo oluşturulur:

```bash
git init --bare /srv/git/ayarlar.git
```

Sonundaki `.git` bir alışkanlıktır, bare repo olduğunu belli eder.

### Yerel repoyu uzak repoya bağlamak ve göndermek

```bash
git remote add origin /srv/git/ayarlar.git
git remote -v
```
```
origin	/srv/git/ayarlar.git (fetch)
origin	/srv/git/ayarlar.git (push)
```

`origin`, uzak repoya verilen kısa addır, gelenek olarak böyle adlandırılır. Adres bir dizin, SSH adresi veya HTTPS adresi olabilir:

```bash
git remote add origin kullanici@git.sirket.lan:/srv/git/ayarlar.git   # SSH
git remote add origin https://github.com/kullanici/ayarlar.git           # HTTPS
```

İlk gönderim:

```bash
git push -u origin main
```
```
To /srv/git/ayarlar.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

`-u`, yerel `main` dalını uzaktaki `origin/main` dalına bağlar. Bir kez yapılır, sonrasında sadece `git push` ve `git pull` yazmak yeter.

### Kopyalamak: git clone

Mert repoyu kendi bilgisayarına kopyalıyor:

```bash
git clone /srv/git/ayarlar.git
```
```
Cloning into 'ayarlar'...
done.
```

`clone`, repo adıyla bir dizin oluşturur, bütün geçmişi indirir ve `origin`'i kendisi ayarlar. Ayrıca `git init`, `git remote add` ve `git pull` yapmaya gerek kalmaz:

```bash
cd ayarlar
git log --oneline | head -2
git branch -a
```
```
46143a6 gzip kapatildi
a60097a Dosya adi degisti
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
```

Farklı bir dizin adı istiyorsan: `git clone /srv/git/ayarlar.git sunucu-ayarlari`.

Mert bir değişiklik yapıp gönderdi:

```bash
echo 'listen 80;' >> nginx-panel.conf
git commit -am "80 portu eklendi"
git push
```

---

## 11. Başkalarının değişikliklerini almak: fetch ve pull

Mert gönderdikten sonra Ayşe kendi reposunda `git status` çalıştırdı:

```
On branch main
Your branch is up to date with 'origin/main'.
```

"Güncel" diyor, ama değil. `git status`, uzak repoyu sorgulamaz, sadece en son indirilen `origin/main` bilgisiyle karşılaştırır. Önce `git fetch` gerekiyor:

```bash
git fetch
git status
```
```
Your branch is behind 'origin/main' by 1 commit, and can be fast-forwarded.
```

Gelen commit'leri birleştirmeden önce görmek için:

```bash
git log --oneline main..origin/main
```
```
5cb19f8 80 portu eklendi
```

| Komut | Ne yapar? |
|---|---|
| `git fetch` | Uzak repodaki yenilikleri indirir, senin dosyalarına dokunmaz |
| `git pull` | `fetch` + indirileni bulunduğun dala birleştirir |

```bash
git pull
```
```
Fast-forward
 nginx-panel.conf | 1 +
```

> [!TIP]
> Çalışmaya başlamadan önce `git pull`, bitirince `git push` yap. Ne kadar uzun süre `pull` yapmadan çalışırsan, başkalarının değişiklikleriyle çakışma ihtimali o kadar artar.

---

## 12. Çakışma çözmek

Mert ve Ayşe aynı satırı farklı değerlere değiştirdi. Mert önce gönderdi. Ayşe göndermeye çalışınca:

```
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to '/srv/git/ayarlar.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally.
```

Uzak repoda Ayşe'nin bilmediği bir commit var, git onu ezmeye izin vermez. Önce `pull` gerek:

```bash
git pull
```
```
hint: You have divergent branches and need to specify how to reconcile them.
...
hint:   git config pull.rebase false  # merge
hint:   git config pull.rebase true   # rebase
hint:   git config pull.ff only       # fast-forward only
...
fatal: Need to specify how to reconcile divergent branches.
```

Git 2.27 ve sonrasında, iki tarafta da yeni commit'ler varsa ve nasıl birleştirileceği ayarlanmamışsa `git pull` bunu sorar ve hiçbir şey yapmaz. Bu notta merge yolunu kullanıyoruz:

```bash
git pull --no-rebase
```
```
Auto-merging nginx-panel.conf
CONFLICT (content): Merge conflict in nginx-panel.conf
Automatic merge failed; fix conflicts and then commit the result.
```

Her seferinde yazmamak için bir kez `git config --global pull.rebase false` ayarlanabilir.

### Çakışan dosya

```bash
cat nginx-panel.conf
```
```
server_name panel.sirket.lan;
listen 443 ssl;
<<<<<<< HEAD
gzip_level=9
=======
gzip_level=4
>>>>>>> 96f209cfff262a2d1f1eac6246696ac7b4a5ea2e
listen 80;
```

| İşaret | Anlamı |
|---|---|
| `<<<<<<< HEAD` ile `=======` arası | Senin değişikliğin |
| `=======` ile `>>>>>>>` arası | Gelen değişiklik |

Git hangisinin doğru olduğuna karar veremez. Çözmek senin işin:

1. Dosyayı düzenle, istediğin son hali bırak, **işaret satırlarını sil**.
2. `git add dosya` ile çözüldüğünü bildir.
3. `git commit` ile birleştirmeyi tamamla.
4. `git push`.

```bash
nano nginx-panel.conf              # ikisi de değil, gzip_level=6 kararı verildi
git add nginx-panel.conf
git commit -m "Cakisma cozuldu: gzip 6"
git push
```

Sonuç:

```
*   a5ff86f Cakisma cozuldu: gzip 6
|\  
| * 96f209c gzip 4
* | ad55069 gzip 9
|/  
* 5cb19f8 80 portu eklendi
```

Çözmekten vazgeçip birleştirme öncesine dönmek istersen: `git merge --abort`. Çakışma sürerken `git status` hangi dosyaların çözülmediğini `Unmerged paths` altında gösterir.

---

## 13. Komut özeti

| Komut | Ne yapar? |
|---|---|
| `git config --global user.name "Ad"` | Kimlik (tüm repolar) |
| `git config --global init.defaultBranch main` | Varsayılan dal adı |
| `git config --list --show-origin` | Ayarlar ve geldikleri dosya |
| `git init` / `git init --bare` | Repo / uzak repo oluştur |
| `git status` / `git status -s` | Durum |
| `git add dosya` / `git add .` | Hazırlık alanına ekle |
| `git commit -m "mesaj"` | Kaydet |
| `git commit -am "mesaj"` | Takip edilenleri ekleyip kaydet |
| `git rm` / `git mv` | Sil / taşı |
| `git diff` / `git diff --staged` | Eklenmemiş / eklenmiş farklar |
| `git log --oneline --graph` | Geçmiş |
| `git show HASH` | Bir commit'in ayrıntısı |
| `git restore dosya` | Dosyayı son commit'e döndür |
| `git restore --staged dosya` | Hazırlık alanından çıkar |
| `git branch` / `git branch -d dal` | Dalları listele / sil |
| `git switch dal` / `git switch -c dal` | Dala geç / yeni dal aç |
| `git merge dal` | Dalı bulunduğun dala birleştir |
| `git merge --abort` | Çakışmalı birleştirmeden vazgeç |
| `git remote add origin ADRES` | Uzak repo ekle |
| `git remote -v` | Uzak repoları listele |
| `git clone ADRES` | Repoyu kopyala |
| `git push -u origin main` | İlk gönderim, dalı bağla |
| `git push` / `git pull` | Gönder / al ve birleştir |
| `git fetch` | Sadece indir |

---

## 14. Sık yapılan hatalar

- **Kimliği ayarlamadan commit denemek.** `Author identity unknown`.
- **`git commit -a`'nın yeni dosyaları da eklediğini sanmak.**
- **`git status` "up to date" dediği için uzak repoyu güncel sanmak.** Önce `git fetch`.
- **Uzun süre `pull` yapmadan çalışmak.** Çakışma ihtimali artar.
- **Çakışma işaretlerini (`<<<<<<<`, `=======`, `>>>>>>>`) dosyada bırakmak.** Git buna engel olmaz, bozuk dosya commit edilir.
- **Şifre ve anahtar dosyalarını commit etmek.** Önce `.gitignore`. Gönderildiyse şifreyi değiştir.
- **`git restore`'un geri alınabileceğini sanmak.** Commit edilmemiş değişiklikler kaybolur.
- **`.git` dizinini silmek.**

---

## 15. Alıştırmalar

**1.** Kimliğini ayarla. Sonra `~/deneme-repo` adında, ilk dalı `main` olan bir repo oluştur. İçine `README.md` dosyası koy ve "Ilk commit" mesajıyla kaydet.

<details>
<summary>Çözüm</summary>

```bash
git config --global user.name "Ad Soyad"
git config --global user.email "ad@ornek.lan"
git config --global init.defaultBranch main

git init ~/deneme-repo
cd ~/deneme-repo
echo "# Deneme" > README.md
git add README.md
git commit -m "Ilk commit"
git log --oneline
```
</details>

**2.** `README.md`'ye bir satır ekle ve `notlar.txt` adında yeni bir dosya oluştur. Sadece `README.md`'deki değişikliği commit et, `notlar.txt` dışarıda kalsın. Commit'ten önce neyin kaydedileceğine bak.

<details>
<summary>Çözüm</summary>

```bash
echo "Bu bir deneme reposudur." >> README.md
echo "taslak" > notlar.txt
git status -s           #  M README.md   ?? notlar.txt
git add README.md
git diff --staged
git commit -m "README aciklamasi eklendi"
git status -s           # ?? notlar.txt
```

`git commit -am` de bu durumda aynı sonucu verirdi, çünkü `notlar.txt` takip edilmiyor.
</details>

**3.** `*.tmp` uzantılı dosyalar ve `notlar.txt` repoya hiç girmesin. Ayarla, bir `a.tmp` oluşturup `git status`'un bunları göstermediğini kontrol et.

<details>
<summary>Çözüm</summary>

```bash
printf '*.tmp\nnotlar.txt\n' > .gitignore
touch a.tmp
git status -s           # sadece ?? .gitignore
git add .gitignore
git commit -m ".gitignore eklendi"
```
</details>

**4.** `ozellik` adında bir dal aç, içinde `yeni.sh` oluşturup commit et. Ana dala dön, `yeni.sh`'ın orada olmadığını gör. Sonra dalı ana dala birleştir ve dalı sil.

<details>
<summary>Çözüm</summary>

```bash
git switch -c ozellik
echo 'echo merhaba' > yeni.sh
git add yeni.sh
git commit -m "yeni.sh eklendi"

git switch main
ls yeni.sh              # No such file or directory

git merge ozellik       # Fast-forward
ls yeni.sh
git branch -d ozellik
```
</details>

**5.** `/tmp/merkez.git` adında bir bare repo oluştur, `deneme-repo`'yu oraya gönder. Sonra onu `/tmp/kopya` dizinine klonla, klonda bir commit yapıp gönder. Son olarak `deneme-repo`'ya dönüp önce gelen değişikliği gör, sonra al.

<details>
<summary>Çözüm</summary>

```bash
git init --bare /tmp/merkez.git
cd ~/deneme-repo
git remote add origin /tmp/merkez.git
git push -u origin main

git clone /tmp/merkez.git /tmp/kopya
cd /tmp/kopya
echo "kopyadan eklendi" >> README.md
git commit -am "Kopyadan degisiklik"
git push

cd ~/deneme-repo
git fetch
git log --oneline main..origin/main    # Kopyadan degisiklik
git pull
```
</details>

---

Önceki konu: [SSL/TLS ve OpenSSL](11-ssl-tls-ve-openssl.md) · Sonraki konu: [Servis Oluşturma ve Sorun Giderme](13-servis-olusturma-ve-sorun-giderme.md)
