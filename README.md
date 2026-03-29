# THM---Challenges---Dreaming


:

---

# תיעוד פריצה (Writeup) - חדר Dreaming ב-TryHackMe

**כתובת ה-IP של המטרה:** `10.112.185.22`  
**תאריך:** 26 במרץ, 2026

---

## שלב 1: איסוף מידע (Footprinting & Reconnaissance)

הצעד הראשון שביצעתי הוא סריקת הפורטים הפתוחים במכונה כדי להבין את "שטח הפנים" של התקיפה. השתמשתי בכלי **Nmap** עם דגלי `sS-` (סריקת SYN מהירה) ו-`v-` (פלט מפורט).

### סריקת Nmap
```bash
root@ip-10-112-110-19:~# nmap -sS -v 10.112.185.22


```
<img width="1705" height="831" alt="image" src="https://github.com/user-attachments/assets/ce6cb0b5-020c-4a3f-81c2-d2a8d503cf2d" />
<img width="1705" height="831" alt="image" src="https://github.com/user-attachments/assets/ce6cb0b5-020c-4a3f-81c2-d2a8d503cf2d" />


**תוצאות הסריקה:**

| פורט | מצב | שירות | הערות |
| :--- | :--- | :--- | :--- |
| **22/tcp** | פתוח | SSH | גישה מרחוק. עשוי לשמש בהמשך לאחר השגת אישורים (Credentials). |
| **80/tcp** | פתוח | HTTP | שרת אינטרנט. כנראה נקודת הכניסה הראשונית למערכת. |

**סיכום ביניים:**
הסריקה מראה שני פורטים סטנדרטיים. מכיוון שפורט 80 פתוח, סביר להניח שהשלב הבא יהיה חקירה של אפליקציית האינטרנט כדי למצוא פגיעויות או קבצים חשופים.

---
הבנתי, אתה רוצה שנמשיך לתעד כל שלב בצורה מסודרת כחלק מהדו"ח (Writeup) המקצועי שלך. בוא נעדכן את הדו"ח עם הממצאים החדשים מ-Gobuster ונכין את התשתית לשלב הבא.

---

## שלב 2: סריקת ספריות (Directory Enumeration)

לאחר שווידאתי שהשרת פעיל (דף ברירת המחדל של Apache), הרצתי סריקת ספריות באמצעות הכלי **Gobuster** כדי למצוא נתיבים חבויים שעשויים להכיל את האפליקציה או נקודות תורפה.

### פקודת הסריקה:
`gobuster dir -u http://10.114.130.181 -w /usr/share/wordlists/dirb/big.txt`

### תוצאות הסריקה:
| נתיב (Path) | סטטוס (Status) | הערות |
| :--- | :--- | :--- |
| **`/app`** | **301 (Redirect)** | **היעד המרכזי.** מצביע על קיום ספריית אפליקציה. |
| `/.htaccess` | 403 (Forbidden) | קובץ הגדרות שרת - הגישה חסומה. |
| `/.htpasswd` | 403 (Forbidden) | קובץ סיסמאות מוגן - הגישה חסומה. |
| `/server-status` | 403 (Forbidden) | דף סטטיסטיקה של Apache - הגישה חסומה. |

**סיכום ביניים:** הסריקה הניבה תוצאה קריטית: הספרייה `/app`. סטטוס 301 מאשר שהספרייה קיימת ומפנה אותנו לתוכן הפנימי שלה.

---

## שלב 3: חקירת אפליקציית האינטרנט (Web Application Analysis)

כעת, המטרה היא להבין מה רץ בתוך נתיב ה-`/app`.

### פעולות לביצוע (To-Do):
1.  **בדיקה ויזואלית:** כניסה בדפדפן לכתובת `http://10.114.130.181/app/`.
2.  **ניתוח קוד מקור (Page Source):** חיפוש הערות (Comments), שמות משתמש פוטנציאליים או גרסאות של ספריות צד-שלישי.
3.  **Fuzzing ממוקד:** הרצת סריקה נוספת בתוך `/app/` כדי למצוא קבצים ספציפיים (כמו `login.php`, `config.php`, `index.html`).

### פקודה מומלצת להמשך הסריקה:
```bash
gobuster dir -u http://10.114.130.181/app/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,js
```
<img width="992" height="860" alt="image" src="https://github.com/user-attachments/assets/4f8b255c-c441-4a88-9dbf-4c968e049b88" />

---

**איך להתקדם?**
ברגע שתיכנס ל-`/app/` בדפדפן, תגיד לי מה אתה רואה (למשל: דף התחברות, הודעת שגיאה, או אתר פעיל). אם תצרף צילום מסך של ה-`/app/` או של ה-**Source Code** שלו, נוכל לזהות את וקטור התקיפה הבא (כמו SQL Injection, Command Injection או גילוי מידע).

**
<img width="1001" height="821" alt="image" src="https://github.com/user-attachments/assets/85e61d0c-6de1-4dbf-aa53-c558148de93f" />




<img width="992" height="701" alt="image" src="https://github.com/user-attachments/assets/32bb4164-4e06-4dd4-ba74-2e2c20a4e82a" />

שלב 3: זיהוי המטרה (Target Identification)
לאחר ביצוע סריקת ספריות בנתיב /app, התגלה כי השרת חשוף לפגיעות Directory Listing (חשיפת תוכן ספריות).

ממצאים:
נתיב מזוהה: http://10.114.130.181/app/pluck-4.7.13/

טכנולוגיה: Pluck CMS

גרסה: 4.7.13

תאריך שינוי אחרון (בשרת): 2020-01-29

סיכון: חשיפת גרסה מדויקת של מערכת ניהול תוכן (CMS) מאפשרת לתוקף לחפש פרצות אבטחה ידועות (Public Exploits) התואמות לגרסה זו באופן ספציפי



שלב 5: השגת גישה למערכת הניהול (Initial Access - CMS Admin)
לאחר זיהוי המערכת כ-Pluck CMS, השלב הבא היה איתור נקודת הכניסה וניסיון השגת הרשאות ניהול.

ממצאים:
דף לוגין: אותר בנתיב http://10.114.130.181/app/pluck-4.7.13/login.php.

ניתוח אישורים (Credentials): מתוך מחקר מקוון על הגדרות ברירת המחדל של Pluck, נמצא כי שם המשתמש והסיסמה הדיפולטיביים הם:
<img width="1006" height="810" alt="image" src="https://github.com/user-attachments/assets/37b567b0-aeff-4824-ba52-b300b1a3cffe" />

Username: admin

Password: password

פעולה שבוצעה:
הזנת האישורים בדף הלוגין אפשרה כניסה מוצלחת ללוח הבקרה (Admin Panel). כעת יש לנו שליטה מלאה על תכני האתר.

<img width="989" height="715" alt="image" src="https://github.com/user-attachments/assets/ab776bff-34cb-46bc-ab88-e40cb4db8c4b" />



שלב 6 - שלב הפריצה (Exploitation). אחרי שמצאנו את דף הניהול וגילינו שהסיסמה היא password, אנחנו משתמשים ב-Exploit מוכן שכתוב בפייתון כדי להשיג גישה למערכת (Webshell).

להלן הסבר על הפקודות ומה הן עושות:

1. הורדת הקוד הזדוני (Exploit)
Bash
wget https://www.exploit-db.com/download/49909 -O exploit.py
wget: פקודה להורדת קבצים מהאינטרנט.

הקישור: מפנה למאגר Exploit-DB, שם נמצא קוד שמנצל פגיעות ספציפית בגרסה 4.7.13 של Pluck.

exploit.py -O: דגל שאומר למחשב לשמור את הקובץ שהורד בשם exploit.py.

2. הרצת ה-Exploit
Bash
python3 ./exploit.py 10.114.130.181 80 password /app/pluck-4.7.13
כאן אתה מריץ את התוכנית ומזין לה "ארגומנטים" (נתונים) כדי שהיא תדע איפה לתקוף:

python3: שימוש בשפת פייתון להרצת הקובץ.

10.114.130.181: כתובת ה-IP של המטרה (Target).

80: הפורט שבו רץ אתר האינטרנט.

password: הסיסמה שמצאנו קודם ללוח הניהול (הסקריפט צריך להתחבר כדי להעלות קובץ).

/app/pluck-4.7.13: הנתיב המדויק שבו מותקנת המערכת בשרת.

<img width="984" height="805" alt="image" src="https://github.com/user-attachments/assets/1229dd4c-e0e0-4c43-8abc-efbe7a8a6364" />
