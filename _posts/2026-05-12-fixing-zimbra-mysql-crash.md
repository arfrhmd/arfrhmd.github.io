---
layout: post
title: Mysql.server Not Running pada Zimbra 8.8.15
date: 2026-05-12 09:01:00
description: 
tags: recovery zimbra mysql-error
categories: zimbra
---

Kali ini saya bertemu masalah baru selama saya menggunakan Zimbra. Error kali ini menampilkan keterangan `mysql.server not running` setelah melihat status zimbra dengan `zmcontrol status`.

Arti `mysql.server not running` menandakan database mysql terkendala saat menjalankan servicenya. Umumnya hal ini terjadi karena ada file yang korup atau faktor lain yang menyebabkan kendala ini terjadi.

## Log Analyze

Setelah saya ingat-ingat, error yang saya dapat ini bermula ketika terjadi gangguan listrik pada area server yang menyebabkan komputer server mati secara paksa sehingga beberapa service Zimbra mengalami error.

Sebelum mulai memperbaiki, saya coba untuk melihat bagian log pada file `/opt/zimbra/log/mysql_error.log` dan berikut isi log yang saya duga penyebab masalah:

```log
260511 14:01:27 mysqld_safe Starting mysqld daemon with databases from /opt/zimbra/db/data
2026-05-11 14:01:27 140589145823104 [Note] /opt/zimbra/common/sbin/mysqld (mysqld 10.1.25-MariaDB) starting as process 271254 ...
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Using mutexes to ref count buffer pool pages
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: The InnoDB memory heap is disabled
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Compressed tables use zlib 1.2.3
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Using Linux native AIO
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Using generic crc32 instructions
2026-05-11 14:01:27 140589145823104 [Note] InnoDB: Initializing buffer pool, size = 1.1G
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Completed initialization of buffer pool
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Highest supported file format is Barracuda.
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Starting crash recovery from checkpoint LSN=4768977081
2026-05-11 14:01:28 140589145823104 [ERROR] InnoDB: space header page consists of zero bytes in tablespace ./zimbra/flush_enforcer.ibd (table zimbra/flush_enforcer)
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:1024 Pages to analyze:64
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 1024, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:2048 Pages to analyze:32
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 2048, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:4096 Pages to analyze:16
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 4096, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:8192 Pages to analyze:8
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 8192, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:16384 Pages to analyze:4
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 16384, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:32768 Pages to analyze:2
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 32768, Possible space_id count:0
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size:65536 Pages to analyze:1
2026-05-11 14:01:28 140589145823104 [Note] InnoDB: Page size: 65536, Possible space_id count:0
2026-05-11 14:01:28 7fdd76189780  InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
InnoDB: If you are installing InnoDB, remember that you must create
InnoDB: directories yourself, InnoDB does not create them.
InnoDB: Error: could not open single-table tablespace file ./zimbra/flush_enforcer.ibd
InnoDB: We do not continue the crash recovery, because the table may become
InnoDB: corrupt if we cannot apply the log records in the InnoDB log to it.
InnoDB: To fix the problem and start mysqld:
InnoDB: 1) If there is a permission problem in the file and mysqld cannot
InnoDB: open the file, you should modify the permissions.
InnoDB: 2) If the table is not needed, or you can restore it from a backup,
InnoDB: then you can remove the .ibd file, and InnoDB will do a normal
InnoDB: crash recovery and ignore that table.
InnoDB: 3) If the file system or the disk is broken, and you cannot remove
InnoDB: the .ibd file, you can set innodb_force_recovery > 0 in my.cnf
InnoDB: and force InnoDB to continue crash recovery here.
260511 14:01:28 [ERROR] mysqld got signal 6 ;
This could be because you hit a bug. It is also possible that this binary
or one of the libraries it was linked against is corrupt, improperly built,
or misconfigured. This error can also be caused by malfunctioning hardware.

To report this bug, see https://mariadb.com/kb/en/reporting-bugs

We will try our best to scrape up some info that will hopefully help
diagnose the problem, but since we have already crashed,
something is definitely wrong and this may fail.
```

Dan penyebab error terdapat pada baris ini:

```log
2026-05-11 14:01:28 140589145823104 [ERROR] InnoDB: space header page consists of zero bytes in tablespace ./zimbra/flush_enforcer.ibd (table zimbra/flush_enforcer)
```

Error tersebut mengarah pada dugaan file `./zimbra/flush_enforcer.ibd` kosong atau korup. Jelasnya, ketika InnoDB mencoba untuk membaca file `flush_enforcer.ibd` tapi yang ditemukan adalah file tersebut kosong (zero bytes). Akibatnya, hal ini mencegah database untuk menyelesaikan pemulihannya secara mandiri.

Setelah tahu masalahnya, saya mulai untuk memperbaiki hal tersebut.

## Backup Database

Ini langkah yang penting dilakukan sebelum melakukan perubahan pada sistem manapun. Saya backup terlebih dahulu database saat ini untuk mencegah terjadinya kesalahan pada step berikutnya.

1. Jalankan sebagai user `zimbra` dan stop seluruh service Zimbra

```bash
su - zimbra
zmcontrol stop
```

2. Jalankan sebagai user `root`, copy direktori database saat ini

```bash
cp -rp /opt/zimbra/db/data /opt/zimbra/db/data.backup.$(date +%Y%m%d)
```

Command ini tujuannya untuk ngebuat folder backup menjadi `/opt/zimbra/db/data.backup.20260512`. **Pastikan folder backup ini juga disalin di disk atau server lain biar aman**.

## Recovery Steps

1. Jalankan sebagai user `root` dan arahkan ke direktori database. Lalu pindahkan file `flush_enforcer.ibd`. Menurut keterangan di forum Zimbra, file ini dapat dipulihkan dengan sendirinya ketika Zimbra mendeteksi file ini tidak ada.

```bash
cd /opt/zimbra/db/data/zimbra
mkdir /tmp/zimbra_table_backup
mv flush_enforcer.frm flush_enforcer.ibd /tmp/zimbra_table_backup/
```

2. Hapus Transaction Coordinator Log (`tc.log`), terkadang ini menjadi penyebab startup Zimbra gagal. File ini aman untuk dipindah/dihapus.

```bash
rm -f /opt/zimbra/db/data/tc.log
```

3. Setelah memindahkan file, pastikan permission pada direktori sudah tepat

```bash
chown -R zimbra:zimbra /opt/zimbra/db/data
```

4. Aktifkan lagi service Zimbra

```bash
su - zimbra
zmcontrol start
```

5. Verifikasi status Zimbra

```bash
zmcontrol status
```

Jika berhasil, maka tidak akan menampilkan `mysql.server not running` ketika verifikasi status.

## References

- [https://forums.zimbra.org/viewtopic.php?t=8462](https://forums.zimbra.org/viewtopic.php?t=8462)
- [https://blog.bartlweb.net/2021/12/mysql-server-von-zimbra-startet-nach-einem-crash-nicht-mehr/](https://blog.bartlweb.net/2021/12/mysql-server-von-zimbra-startet-nach-einem-crash-nicht-mehr/)