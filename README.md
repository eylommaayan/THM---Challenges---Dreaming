💤 מדריך פתרון למכונת Dreaming
שלב 1: איסוף מידע (Footprinting)
בשלב הראשון נרצה להבין אילו שירותים רצים על המכונה.

הפקודה שביצעתי:

Bash
nmap -sS -v <MACHINE_IP>
מה אמורים לגלות: פורטים פתוחים ושירותים זמינים.

מה גיליתי: רק פורטים 22 (SSH) ו-80 (HTTP) פתוחים. בביקור בדפדפן מופיע דף ברירת המחדל של Apache.

שלב 2: סריקת ספריות (Directory Enumeration)
מכיוון שהדף הראשי ריק, נחפש ספריות חבויות בשרת הווב.

הפקודה שביצעתי:

Bash
gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/big.txt
מה אמורים לגלות: נתיבים או קבצים שלא מופיעים בקישורים רגילים.

מה גיליתי: נמצאה ספרייה בשם /app. בתוכה מצאתי התקנה של מערכת ניהול תוכן (CMS) בשם Pluck בגרסה 4.7.13.

שלב 3: פריצה ראשונית (Initial Access) ודגל Lucien
ננסה להיכנס לממשק הניהול של Pluck ולנצל פגיעות ידועה.

הפקודה שביצעתי:

ניסיון התחברות ב-http://<MACHINE_IP>/app/pluck-4.7.13/login.php עם סיסמת ברירת המחדל password.

הרצת אקספלויט (CVE-2020-29607) להעלאת קובץ זדוני:

Bash
python3 exploit.py <MACHINE_IP> 80 password /app/pluck-4.7.13
מה אמורים לגלות: גישה למערכת הקבצים (Webshell).

מה גיליתי: בתוך /opt מצאתי קובץ בשם test.py המכיל את הסיסמה של המשתמש Lucien בטקסט גלוי. השתמשתי בה להתחברות ב-SSH וקראתי את הדגל הראשון.

שלב 4: העלאת הרשאות למשתמש Death
נבדוק מה Lucien יכול להריץ בהרשאות גבוהות.

הפקודה שביצעתי:

Bash
sudo -l
cat ~/.bash_history
מה אמורים לגלות: פקודות sudo מותרות או סיסמאות בהיסטוריה.

מה גיליתי: Lucien יכול להריץ סקריפט בשם getDreams.py כמשתמש death. בנוסף, מצאתי סיסמת MySQL בהיסטוריית הפקודות.

הזרקת פקודה (Command Injection):
הסקריפט getDreams.py משתמש ב-subprocess בצורה לא מאובטחת. נכנסתי ל-MySQL והזרקתי פקודה לטבלה:

SQL
use library;
insert into dreams (dreamer, dream) VALUES ("'flag'", "'; cat /home/death/death_flag.txt; # ");
לאחר מכן הרצתי את הסקריפט: sudo -u death /opt/scripts/getDreams.py וקיבלתי את דגל ה-Death.

שלב 5: העלאת הרשאות למשתמש Morpheus (Root/Final Flag)
כעת כשיש לנו את הסיסמה של Death (מהסקריפט), נעבור אליו ונחפש דרך להפוך ל-Morpheus.

הפקודה שביצעתי:

Bash
find / -type f -group morpheus 2> /dev/null
ls -al /usr/lib/python3.8/shutil.py
מה אמורים לגלות: קבצים ששייכים לקבוצת morpheus או הרשאות כתיבה בספריות מערכת.

מה גיליתי: ישנו סקריפט גיבוי בשם restore.py שרץ כנראה כ-Cron Job על ידי Morpheus. גיליתי שלמשתמש death יש הרשאת כתיבה לספריית הפייתון shutil.py.

הביצוע:
ערכתי את הקובץ /usr/lib/python3.8/shutil.py והוספתי לפונקציה copy2 את השורה הבאה:

Python
os.system("chmod 777 /home/morpheus/morpheus_flag.txt")
התוצאה: כשהסקריפט restore.py רץ אוטומטית, הוא קרא לספריית shutil, והפקודה שלי שינתה את ההרשאות של הדגל. כעת יכולתי לקרוא את הדגל האחרון של Morpheus.
