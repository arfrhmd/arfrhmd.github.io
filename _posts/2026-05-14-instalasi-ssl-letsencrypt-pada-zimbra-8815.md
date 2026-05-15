---
layout: post
title: Instalasi SSL Let's Encrypt pada Zimbra 8.8.15 (Metode DNS)
date: 2026-05-14 13:05:00
description: Artikel ini meliputi tahapan instalasi SSL Let's Encrypt pada layanan Zimbra 8.8.15 di CentOS 8 menggunakan metode DNS.
tags: linux zimbra migration
categories: zimbra
---

SSL atau Secure Socket Layer biasa digunakan pada sistem sebagai keamanan jaringan dasar, lalu lintas jaringan yang sudah terpasang SSL akan meminimalisir kebocoran data mentah (plaintext) pada saat pertukaran data berlangsung.

Pada artikel kali ini, saya akan membahas tentang bagaimana instalasi SSL Let's Encrypt pada Zimbra versi 8.8.15 dengan sistem operasi CentOS 8.

## Update repository

Update repositori CentOS dan instal paket `epel-release`

```bash
sudo yum update -y
sudo yum install epel-release -y
```

## Install `certbot`

**Certbot** adalah tool open-source otomatis yang digunakan untuk mendapatkan dan menginstal sertifikat SSL/TLS gratis dari Let's Encrypt guna mengaktifkan HTTPS di server web (Apache, Nginx)

```bash
sudo yum install certbot -y
```

## Request SSL ke Let's Encrypt (metode DNS)

Disini saya menggunakan metode DNS sebagai metode alternatif dari `--standalone` dikarenakan sering terjadinya konflik pada port 80 saat verifikasi ACME file.

```bash
sudo certbot certonly --manual --preferred-challenges dns -d mail.rahmadi.id
```

## Setting DNS Record

Setelah ini saya diberi sebuah kode acak untuk dimasukkan ke dalam record TXT pada DNS dengan tampilan seperti berikut:

```
Please deploy a DNS TXT record under the name:
_acme-challenge.mail.rahmadi.id
with the following value:
nkE22ZcsbxobX5W56iEDbeN1T7cduyhE7jJrqR6Z7fE
```

Setelah mendapatkan kode acak, tambahkan DNS record dengan tipe TXT pada domain control panel, kira-kira seperti ini:

```
Hostname                            Type    TTL    Value
_acme-challenge.mail.rahmadi.id     TXT     1800   nkE22ZcsbxobX5W56iEDbeN1T7cduyhE7jJrqR6Z7fE
```

Setelah DNS record ditambahkan, verifikasi propagasi menggunakan [DNS Checker](https://dnschecker.org/).

Jika propagasi sudah selesai, kembali ke terminal server dan konfirmasi verifikasi Let's Encrypt dengan menekan tombol `Enter` dan SSL akan segera terbit dengan tampilan seperti ini:

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/mail.rahmadi.id/privkey.pem
   Your cert will expire on 2026-08-12. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:
 
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

## Deploy SSL

Setelah SSL berhasil didapatkan dari Let's Encrypt, tahap selanjutnya saya deploy SSL tersebut ke layanan Zimbra.

Lakukan copy file pada privatekey SSL ke folder SSL Zimbra

```bash
cp /etc/letsencrypt/live/mail.rahmadi.id/privkey.pem /opt/zimbra/ssl/zimbra/commercial/commercial.key
```

Lakukan perubahan akses ke user `zimbra` pada file `commercial.key`

```bash
chown zimbra:zimbra /opt/zimbra/ssl/zimbra/commercial/commercial.key
```

Tahapan berikutnya, buat CA Let's Encrypt yang digabungkan ke file `fullchain.pem`

```bash
wget -O /tmp/ISRG-X1.pem https://letsencrypt.org/certs/isrgrootx1.pem
wget -O /tmp/R3.pem https://letsencrypt.org/certs/lets-encrypt-r3.pem
cat /tmp/R3.pem >> /etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem
cat /tmp/ISRG-X1.pem >> /etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem
```

Lakukan perubahan akses ke user `zimbra` pada folder `/etc/letsencrypt/`

```bash
chown -R zimbra:zimbra /etc/letsencrypt/
```

Jalankan sebagai `zimbra` untuk verifikasi SSL

```bash
su - zimbra
/opt/zimbra/bin/zmcertmgr verifycrt comm /opt/zimbra/ssl/zimbra/commercial/commercial.key /etc/letsencrypt/live/mail.rahmadi.id/cert.pem /etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem
```

*Jika semua valid, maka tampilannya seperti berikut:*

```
** Verifying '/etc/letsencrypt/live/mail.rahmadi.id/cert.pem' against '/opt/zimbra/ssl/zimbra/commercial/commercial.key'
Certificate '/etc/letsencrypt/live/mail.rahmadi.id/cert.pem' and private key '/opt/zimbra/ssl/zimbra/commercial/commercial.key' match.
** Verifying '/etc/letsencrypt/live/mail.rahmadi.id/cert.pem' against '/etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem'
Valid certificate chain: /etc/letsencrypt/live/mail.rahmadi.id/cert.pem: OK
```

Lanjut di tahap terakhir, deploy SSL

```
/opt/zimbra/bin/zmcertmgr deploycrt comm /etc/letsencrypt/live/mail.rahmadi.id/cert.pem /etc/letsencrypt/live/mail.rahmadi.id/fullchain.pem
```

Kemudian, restart layanan Zimbra

```bash
zmcontrol restart
```

## References

- [https://saad.web.id/2021/10/cara-mudah-pasang-ssl-lets-encrypt-zimbra-8-8-15-di-ubuntu-18-04/](https://saad.web.id/2021/10/cara-mudah-pasang-ssl-lets-encrypt-zimbra-8-8-15-di-ubuntu-18-04/)
- [https://www.jisc.ac.uk/security-certificate-automation/automation-options/using-acme-protocol-and-certbot](https://www.jisc.ac.uk/security-certificate-automation/automation-options/using-acme-protocol-and-certbot)
- [https://www.azion.com/en/documentation/products/guides/secure/lets-encrypt-record/](https://www.azion.com/en/documentation/products/guides/secure/lets-encrypt-record/#overview)