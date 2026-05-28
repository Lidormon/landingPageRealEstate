## חיבור הטופס ל-Google Sheets

הטופס באתר כבר מוכן לשלוח ב-`POST` ל-Google Apps Script.
צריך להשלים רק את ההגדרה בגוגל ולהדביק את ה-URL בקובץ `limor.html`.

### 1) יצירת גיליון

1. פתחי Google Sheets וצרי קובץ חדש.
2. בשורה הראשונה צרי כותרות עמודות:
   - `timestamp`
   - `firstName`
   - `lastName`
   - `phone`
   - `email`
   - `city`
   - `message`
   - `interest`

### 2) Apps Script

0. העתיקי את **מזהה הגיליון** מה-URL:
   - `https://docs.google.com/spreadsheets/d/XXXXXXXXXXXX/edit`
   - החלק `XXXXXXXXXXXX` הוא ה-`SPREADSHEET_ID`

1. מתוך הגיליון: `Extensions` -> `Apps Script`.
2. מחקי את התוכן הקיים והדביקי את הקוד הבא:

```javascript
// ====== הגדרות ======
var NOTIFY_EMAIL = "limormondel@gmail.com"; // אפשר לשנות לכתובת אחרת

// מזהה הגיליון – מעתיקים מה-URL של Google Sheets:
// https://docs.google.com/spreadsheets/d/XXXXXXXXXXXX/edit
var SPREADSHEET_ID = "הדביקי_כאן_את_מזהה_הגיליון";

function getSheet() {
  return SpreadsheetApp.openById(SPREADSHEET_ID).getSheets()[0];
}

// תאריך ושעה לפי שעון ישראל, פורמט קריא: 27/05/2026 22:30
function getIsraelDateTime() {
  return Utilities.formatDate(new Date(), "Asia/Jerusalem", "dd/MM/yyyy HH:mm");
}

function parseRequestData(e) {
  // טופס רגיל (POST) – הנתונים מגיעים ב-e.parameter
  if (e && e.parameter && Object.keys(e.parameter).length > 0) {
    return e.parameter;
  }
  // JSON (אם נשלח בעתיד)
  if (e && e.postData && e.postData.contents) {
    try {
      return JSON.parse(e.postData.contents || "{}");
    } catch (err) {}
  }
  return {};
}

function doGet() {
  return ContentService
    .createTextOutput("Web App פעיל")
    .setMimeType(ContentService.MimeType.TEXT);
}

function doPost(e) {
  try {
    if (!SPREADSHEET_ID || SPREADSHEET_ID.indexOf("הדביקי") !== -1) {
      throw new Error("חסר SPREADSHEET_ID בקוד Apps Script");
    }

    var sheet = getSheet();
    var data = parseRequestData(e);
    data.timestamp = getIsraelDateTime();

    sheet.appendRow([
      data.timestamp,
      data.firstName || "",
      data.lastName || "",
      data.phone || "",
      data.email || "",
      data.city || "",
      data.message || "",
      data.interest || ""
    ]);

    try {
      sendEmailNotification(data);
    } catch (mailErr) {
      // שומר את הליד בגיליון גם אם שליחת המייל נכשלה
    }

    return ContentService
      .createTextOutput(JSON.stringify({ ok: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ ok: false, error: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function sendEmailNotification(data) {
  var subject = "ליד חדש מהאתר - " + (data.interest || "");
  var body = [
    "התקבלה פנייה חדשה מהאתר:",
    "",
    "תאריך: " + (data.timestamp || ""),
    "שם פרטי: " + (data.firstName || ""),
    "שם משפחה: " + (data.lastName || ""),
    "טלפון: " + (data.phone || ""),
    "אימייל: " + (data.email || ""),
    "עיר: " + (data.city || ""),
    "סוג פנייה: " + (data.interest || ""),
    "הערות: " + (data.message || "")
  ].join("\n");

  MailApp.sendEmail({
    to: NOTIFY_EMAIL,
    subject: subject,
    body: body
  });
}

// הרצה חד-פעמית מ-Apps Script (▶ Run) כדי לאשר שליחת מייל
function testEmailNotification() {
  sendEmailNotification({
    timestamp: getIsraelDateTime(),
    firstName: "בדיקה",
    lastName: "מערכת",
    phone: "050-0000000",
    email: "test@example.com",
    city: "בדיקה",
    message: "זו הודעת בדיקה מהאתר",
    interest: "מכירה"
  });
}
```

### 3) Deploy כ-Web App

1. לחצי `Deploy` -> `New deployment`.
2. בחרי סוג `Web app`.
3. `Execute as`: `Me`.
4. `Who has access`: `Anyone`.
5. לחצי `Deploy` והעתיקי את ה-`Web app URL`.

### 4) הדבקת ה-URL באתר

בקובץ `limor.html` חפשי:

```js
const GOOGLE_SCRIPT_URL = "PASTE_YOUR_GOOGLE_APPS_SCRIPT_WEB_APP_URL_HERE";
```

והחליפי ל-URL האמיתי שקיבלת.

### 5) אחרי שינוי קוד – Deploy מחדש

אם עדכנת את הקוד ב-Apps Script, חובה:
1. `Deploy` -> `Manage deployments`
2. `Edit` (עיפרון) -> `Version` -> `New version`
3. `Deploy`

ואז ודאי שה-URL ב-`limor.html` עדיין מעודכן.

### 6) התראה במייל לכל ליד חדש

- המייל נשלח אוטומטית ל-`limormondel@gmail.com` בכל שליחת טופס.
- **חובה פעם אחת:** ב-Apps Script בחרי `testEmailNotification` מהרשימה למעלה → לחצי **Run (▶)** → **אשרי הרשאות** (Gmail / שליחת מייל).
- אחרי אישור, עשי **Deploy מחדש** (גרסה חדשה).
- בדקי גם בתיקיית **ספאם/דואר זבל**.

### 7) פתרון בעיות נפוצות

- **באתר כתוב הצלחה אבל אין שורה בגיליון:**
  - בדקו שה-URL מסתיים ב-`/exec` (בלי כפילות או תווים מיותרים).
  - עשו Deploy מחדש לגרסה חדשה אחרי שינוי קוד.
  - ודאו `Who has access` = `Anyone`.
- **בדיקה מהירה:** פתחו בדפדפן את כתובת ה-Web App – אמור להופיע הודעה (לא שגיאת 404).

### 8) התראה במייל בלבד (בחינם)

- ההתראה במייל דרך `MailApp` היא ללא עלות נוספת.
- יש מכסה יומית של גוגל לשליחת מיילים (מספיק לרוב האתרים הקטנים/בינוניים).
- ההתראה נשלחת לכתובת שמוגדרת ב-`NOTIFY_EMAIL`.

---

אחרי זה כל שליחת טופס:
- תיכנס כשורה חדשה בגיליון
- תשלח התראת מייל
