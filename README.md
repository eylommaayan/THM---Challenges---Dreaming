

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


<img width="996" height="317" alt="image" src="https://github.com/user-attachments/assets/b7fbc386-93b4-4f45-92c1-0db84e47ff9e" />


