# 05 - Kullanıcılar ve Gruplar

LFCS sınavında bu alanın ağırlığı **%10**. Resmi konu başlıkları:

- Yerel kullanıcı ve grup hesapları oluşturma ve yönetme
- Kişisel ve sistem geneli ortam profillerini yönetme
- Kullanıcı kaynak limitlerini yapılandırma
- ACL yapılandırma ve yönetme
- Sistemi LDAP kullanıcı ve grup hesaplarını kullanacak şekilde yapılandırma

## Konular

| # | Konu | İçerik |
|---|---|---|
| 01 | [Kullanıcı Hesapları](01-kullanici-hesaplari.md) | `useradd`, `usermod`, `userdel`, `passwd`, `chage`, `/etc/passwd`, `/etc/shadow` |
| 02 | [Gruplar](02-gruplar.md) | `groupadd`, `gpasswd`, `usermod -aG`, birincil ve ek gruplar, ortak proje dizini |
| 03 | [sudo ve su](03-sudo-ve-su.md) | `sudo`, `su -`, kilitli root, `visudo`, `/etc/sudoers.d/` kuralları |
| 04 | [Ortam Profilleri](04-ortam-profilleri.md) | PATH, `~/.profile`, `~/.bashrc`, `/etc/environment`, `/etc/profile.d/`, `/etc/skel` |
| 05 | [Kaynak Limitleri](05-kaynak-limitleri.md) | `/etc/security/limits.conf`, soft/hard, `ulimit` |
| 06 | [ACL ve Dosya Öznitelikleri](06-acl.md) | `setfacl`, `getfacl`, mask, varsayılan ACL, `chattr`, `lsattr` |
| 07 | [LDAP](07-ldap.md) | Deneme sunucusu, `libnss-ldapd`, `nsswitch.conf`, `nslcd.conf`, otomatik ev dizini |

## Nasıl çalışmalı?

Konular birbirinin üzerine kurulu, sırayla gitmen iyi olur. Bütün dosyalarda aynı senaryo devam ediyor: küçük bir yazılım ekibinin sunucusunu yönetiyoruz. Her dosyanın sonunda çözümlü alıştırmalar var. Önce kendin dene, sonra çözüme bak.

Bütün komutlar Ubuntu 24.04 üzerinde denenmiştir.

---

[Ana sayfaya dön](../README.md)
