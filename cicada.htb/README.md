1. Mərhələ: Başlanğıc Kəşfiyyat (Enumeration)
İlk addım olaraq hədəf sistemdə hansı istifadəçilərin olduğunu öyrənməyə çalışdıq. Heç bir şifrəmiz olmadığı üçün Anonymous Login boşluğundan istifadə etdik.

İşlədilən Komanda:
crackmapexec smb 10.129.231.149 -u anonymous -p '' --rid-brute

Məntiq: Windows-da hər istifadəçinin bir RID (Relative Identifier) nömrəsi olur. Biz sistemə tək-tək nömrələr göndərərək ("500 kimdir?", "1104 kimdir?") adları sızdırdıq.

Nəticə: john.smoulder, michael.wrightson, david.orelious kimi istifadəçi adlarını əldə etdik.

2. Mərhələ: Giriş Nöqtəsi (Initial Access)
Əlimizdəki adlara qarşı Password Spraying (bir şifrəni çox ad üçün yoxlamaq) etdik.

Nəticə: michael.wrightson istifadəçisinin şifrəsini tapdıq.

Dərinləşmə: Michael-ın səlahiyyəti ilə sistemdəki bütün istifadəçiləri və onların Description (Təsvir) bölmələrini oxuduq:
crackmapexec smb 10.129.231.149 -u 'michael.wrightson' -p '...' --users

Kritik Tapıntı: Təsvir bölməsində digər istifadəçi david.orelious üçün müvəqqəti şifrə tapıldı.

3. Mərhələ: Fayl Sisteminin Tədqiqi (SMB Share Hunting)
David-in şifrəsi ilə daxil olub, sistemdəki paylaşılan qovluqlara baxdıq.

İşlədilən Komanda:
smbclient //10.129.231.149/DEV -U 'david.orelious'

Tapıntı: Backup_script.ps1 adlı PowerShell skriptini tapdıq və get komandası ilə özümüzə yüklədik.

Analiz: Skriptin içində emily.oscars adlı üçüncü bir istifadəçinin şifrəsi açıq şəkildə yazılmışdı (Hardcoded Credentials).

4. Mərhələ: Sistemin İçinə Giriş (Foothold via WinRM)
Emily-nin şifrəsini yoxladıq və onun WinRM (Remote Management) icazəsinin olduğunu gördük. Bu, bizə uzaqdan terminal bağlantısı verdi.

İşlədilən Komanda:
evil-winrm -i 10.129.231.149 -u 'emily.oscars' -p '...'

5. Mərhələ: Səlahiyyət Artırılması (Privilege Escalation)
İçəri girdikdən sonra öz imtiyazlarımızı yoxladıq (whoami /priv).

Kritik Zəiflik: SeBackupPrivilege və SeRestorePrivilege aktiv idi.

Məntiq: Bu imtiyaz bizə sistemdəki istənilən faylı, hətta şifrə bazalarını belə oxumağa imkan verir. Biz bu səlahiyyətlə SAM və SYSTEM registry fayllarının nüsxəsini çıxardıq.

6. Mərhələ: Final - Administrator Hash Dumping
Çıxardığımız registry fayllarını öz Kali maşınımıza çəkib, içindəki şifrə hash-lərini oxuduq.

İşlədilən Komanda:
python3 secretsdump.py -sam sam.bak -system system.bak LOCAL

Nəticə: Administratorun NTLM Hash-ini əldə etdik: 2b87e7c93a3e8a0ea4a581937016f341

7. Mərhələ: Pass-The-Hash (Tam İdarəetmə)
Administratorun əsl şifrəsini bilməyə ehtiyac duymadan, sadəcə hash ilə sistemə daxil olduq.

Final Komanda:
evil-winrm -i 10.129.231.149 -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341
