

# 💤 Dreaming - TryHackMe Writeup (Personal Walkthrough)

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
python3 exploit.py 10.113.150.246 80 password /app/pluck-4.7.13
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


חשיפת הסיסמה של Lucien:
קראתי את התוכן של הקובץ test.py כדי להבין את תפקידו.

הפקודה שביצעתי:

Bash
cat /opt/test.py
מה גיליתי: הסקריפט הכיל קוד פייתון שנועד לבדוק התחברות למערכת ה-Pluck. בתוך הקוד הופיעה הסיסמה של המשתמש Lucien בטקסט גלוי (Cleartext).

הסיסמה שנחשפה: HeyLucien#@1999!

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

10.113.150.246: כתובת ה-IP של היעד (Dreaming).

💡 למה אני עושה את זה ובשביל מה?
יציבות ואמינות: ה-Webshell (הטרמינל השחור בדפדפן) הוא כלי שברירי ומוגבל. הוא עלול להתנתק בכל רגע, הוא איטי, ולא מאפשר הרצת פקודות אינטראקטיביות מורכבות. חיבור SSH מעניק לי נוכחות יציבה (Persistence) בשרת.

שיפור הרשאות (Lateral Movement): כרגע אני פועל כמשתמש www-data המוגבל לתיקיות האתר בלבד. על ידי התחברות כ-lucien, אני מבצע "תנועה רוחבית" למשתמש מערכת בעל הרשאות גבוהות יותר, שיש לו גישה לתיקיית בית אישית (/home/lucien).

השגת הדגל הראשון: משתמש ה-Web אינו מורשה לקרוא קבצי משתמשים. רק לאחר ההתחברות כ-Lucien, אוכל לגשת לקובץ user.txt ולקבל את הדגל הראשון.

🛡️ התוצאה בשטח:
עם הזנת הסיסמה בטרמינל, קיבלתי גישת Shell מלאה.
ביצעתי את הפקודה:


cat /home/lucien/user.txt

<img width="859" height="314" alt="image" src="https://github.com/user-attachments/assets/ff214155-28ae-4db4-9b63-3fd21b088197" />

"במהלך החיבור הראשוני ב-SSH, אישרתי את ה-ECDSA key fingerprint של השרת. פעולה זו נדרשת כדי להקים ערוץ תקשורת מוצפן (Encrypted Tunnel) בין המכונה התוקפת ליעד. לאחר מכן, הזנתי את פרטי המשתמש Lucien שחולצו קודם לכן."

🚀 מה המטרה שלי עכשיו? 
הסיבה: אישור המפתח הוא הצעד האחרון לפני הכניסה ל"בית" של המשתמש.

המטרה: להשיג Shell אינטראקטיבי מלא.

השלב הבא: ברגע שתהיה בפנים, הפקודה הראשונה שתריץ היא:


cat user.txt
כדי לקבל את ה-Flag הראשון ולהוכיח שפרצת את המשתמש.



"ניסיתי לעבור לחיבור SSH יציב כמשתמש Lucien, אך נתקלתי בשגיאת Broken pipe. הבנתי ששירות ה-SSH בשרת היעד אינו מאפשר התחברות ישירה בשלב זה. עקב כך, חזרתי להשתמש ב-Webshell שהשגתי קודם לכן כדי להמשיך בחקירת המערכת והשגת הדגל.

<img width="757" height="283" alt="image" src="https://github.com/user-attachments/assets/06f046da-bbd8-42d6-8a6c-d83a42b56a56" />

2.4 חקירת מערכת הקבצים והשגת הדגל (Enumeration)
הפעולה: לאחר שחיבור ה-SSH נכשל (Broken pipe), חזרתי להשתמש ב-Webshell בדפדפן כדי להמשיך בחקירת המכונה.

אימות המיקום (pwd):
בטרמינל בדפדפן הרצתי את הפקודה pwd (ראשי תיבות של Print Working Directory)

<img width="740" height="757" alt="image" src="https://github.com/user-attachments/assets/e2745d3e-198f-45e4-b76c-a8a9aaf6c766" />

<img width="793" height="797" alt="image" src="https://github.com/user-attachments/assets/aec6eaeb-6b55-4c11-890b-2dbd66937343" />

<img width="1564" height="842" alt="image" src="https://github.com/user-attachments/assets/4177d7ef-8fc0-4cc1-8362-234a627ca90c" />



