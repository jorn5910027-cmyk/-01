var BOT_TOKEN = "บบบ"; // ใส่ Bot Token ของคุณที่นี่
var CHAT_ID = "บบบ";    // ใส่ Chat ID ของกลุ่มหรือส่วนตัวที่นี่

function onFormSubmit(e) {
  if (!e || !e.values) {
    Logger.log("ไม่พบข้อมูล e.values (หากทดสอบให้กดรัน testRAWdataCommon หรือ testRAWdataRoom)");
    return;
  }

  var values = e.values;
  var sheetName = e.range ? e.range.getSheet().getName() : "RAWdataส่วนกลาง";
  var timestamp = new Date().toLocaleString("th-TH", { timeZone: "Asia/Bangkok" });

  // กำหนดประเภทงานตาม Tab ชีต
  var categoryName = "งานห้องชุด";
  if (sheetName === "RAWdataส่วนกลาง") {
    categoryName = "งานส่วนกลาง";
  }

  // ดึงข้อมูลพื้นฐานที่ตรงกัน
  var workDate = values[2] || "-";               // วันที่เข้างาน (คอลัมน์ C)[cite: 3]
  var building = values[4] || "-";               // อาคาร (คอลัมน์ E)[cite: 3]
  var leaderName = escapeHtml(values[5] || "-");   // ชื่อ-นามสกุล / หัวหน้าชุด (คอลัมน์ F)[cite: 3]
  var company = escapeHtml(values[7] || "-");        // ชื่อบริษัท (คอลัมน์ H)[cite: 3]
  var jobType = escapeHtml(values[8] || "-");        // ประเภทงาน (คอลัมน์ I)[cite: 3]

  var roomNo = "-";
  var floor = "-";
  var unit = "-";
  var equipmentRaw = "";
  var workerCount = "-";

  // จัดการ Index ตามโครงสร้างจริงของคุณ
  if (sheetName === "RAWdataส่วนกลาง") {
    // RAWdataส่วนกลาง[cite: 3]
    floor = values[9] || "-";          // ชั้น (คอลัมน์ J)[cite: 3]
    equipmentRaw = values[10] || "";   // รายการอุปกรณ์ (คอลัมน์ K)[cite: 3]
    workerCount = values[11] || "-";   // จำนวนคนงานทั้งหมด (คอลัมน์ L)[cite: 3]
  } else {
    // RAWdataห้องชุด[cite: 3]
    roomNo = values[9] || "-";         // ห้องทำงานเลขที่ (คอลัมน์ J)[cite: 3]
    floor = values[10] || "-";         // ชั้น (คอลัมน์ K)[cite: 3]
    unit = values[11] || "-";          // ยูนิต (คอลัมน์ L)[cite: 3]
    equipmentRaw = values[13] || "";   // รายการอุปกรณ์ (คอลัมน์ N)[cite: 3]
    workerCount = values[14] || "-";   // จำนวนคนงานทั้งหมด (คอลัมน์ O)[cite: 3]
  }

  // ตรวจสอบและจัดการลิงก์รูปภาพ
  var photoFileId = extractDriveFileId(equipmentRaw);
  var equipmentDisplay = formatEquipmentText(equipmentRaw);

  // สร้างข้อความแจ้งเตือน
  var text = "<b>👷 แจ้งเตือนผู้รับเหมาเข้าทำงาน (" + categoryName + ") 👷</b>\n" +
             "<b>📅 เวลาบันทึกข้อมูล:</b> " + timestamp + "\n" +
             "------------------------------------\n" +
             "<b>🗓️ วันที่เข้างาน:</b> " + workDate + "\n" +
             "<b>🏢 อาคาร:</b> " + building + "\n";

  // แสดงเลขห้อง/ยูนิตเฉพาะงานห้องชุด
  if (sheetName === "RAWdataห้องชุด") {
    text += "<b>🚪 ห้องเลขที่:</b> " + roomNo + " | <b>ยูนิต:</b> " + unit + "\n";
  }

  text += "<b>📶 ชั้น:</b> " + floor + "\n" +
          "<b>👤 ชื่อ-นามสกุล / ผู้ติดต่อ:</b> " + leaderName + "\n" +
          "<b>🏢 บริษัท:</b> " + company + "\n" +
          "<b>👥 จำนวนคนงานทั้งหมด:</b> " + workerCount + " คน\n" +
          "<b>🛠️ ประเภทงาน:</b> " + jobType + "\n" +
          "<b>📦 รายการอุปกรณ์:</b> " + equipmentDisplay;

  // ส่งข้อมูลเข้า Telegram
  if (photoFileId) {
    sendPhotoBlobToTelegram(BOT_TOKEN, CHAT_ID, text, photoFileId);
  } else if (isValidImageUrl(equipmentRaw)) {
    sendPhotoUrlToTelegram(BOT_TOKEN, CHAT_ID, text, equipmentRaw);
  } else {
    sendMessageToTelegram(BOT_TOKEN, CHAT_ID, text);
  }
}

// ค้นหา File ID จากลิงก์ Google Drive
function extractDriveFileId(url) {
  if (!url) return null;
  var fileIdMatch = url.match(/[-\w]{25,}/);
  if (fileIdMatch && (url.indexOf("drive.google.com") !== -1 || url.indexOf("docs.google.com") !== -1)) {
    return fileIdMatch[0];
  }
  return null;
}

// ตรวจสอบว่าเป็นลิงก์รูปภาพทั่วไปหรือไม่
function isValidImageUrl(url) {
  if (!url) return false;
  return url.match(/\.(jpeg|jpg|gif|png)$/i) !== null;
}

// จัดการข้อความแสดงในช่องรายการอุปกรณ์
function formatEquipmentText(rawText) {
  if (!rawText || rawText.trim() === "") return "-";
  if (rawText.indexOf("http://") === 0 || rawText.indexOf("https://") === 0) {
    return "<a href='" + rawText + "'>📎 ดูรูปภาพ/รายการอุปกรณ์แนบ</a>";
  }
  return escapeHtml(rawText);
}

// ส่งแบบข้อความปกติ
function sendMessageToTelegram(token, chatId, text) {
  var url = "https://api.telegram.org/bot" + token + "/sendMessage";
  var payload = {
    "chat_id": chatId,
    "text": text,
    "parse_mode": "HTML",
    "disable_web_page_preview": false
  };
  var options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "muteHttpExceptions": true
  };
  try {
    UrlFetchApp.fetch(url, options);
  } catch (err) {
    Logger.log("เกิดข้อผิดพลาดในการส่งข้อความ: " + err.toString());
  }
}

// ส่งรูปภาพโดยดึงไฟล์จาก Google Drive โดยตรง
function sendPhotoBlobToTelegram(token, chatId, captionText, fileId) {
  var url = "https://api.telegram.org/bot" + token + "/sendPhoto";
  try {
    var file = DriveApp.getFileById(fileId);
    var blob = file.getBlob();
    
    var payload = {
      "chat_id": chatId,
      "photo": blob,
      "caption": captionText,
      "parse_mode": "HTML"
    };
    var options = {
      "method": "post",
      "payload": payload,
      "muteHttpExceptions": true
    };
    
    var response = UrlFetchApp.fetch(url, options);
    var resJson = JSON.parse(response.getContentText());
    if (!resJson.ok) {
      sendMessageToTelegram(token, chatId, captionText);
    }
  } catch (err) {
    sendMessageToTelegram(token, chatId, captionText);
  }
}

// ส่งรูปภาพผ่าน URL ปกติ
function sendPhotoUrlToTelegram(token, chatId, captionText, photoUrl) {
  var url = "https://api.telegram.org/bot" + token + "/sendPhoto";
  var payload = {
    "chat_id": chatId,
    "photo": photoUrl,
    "caption": captionText,
    "parse_mode": "HTML"
  };
  var options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "muteHttpExceptions": true
  };
  try {
    var response = UrlFetchApp.fetch(url, options);
    var resJson = JSON.parse(response.getContentText());
    if (!resJson.ok) {
      sendMessageToTelegram(token, chatId, captionText);
    }
  } catch (err) {
    sendMessageToTelegram(token, chatId, captionText);
  }
}

// ป้องกันตัวอักษรพิเศษชนกับ HTML Tags
function escapeHtml(text) {
  return String(text)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;");
}

// --- ฟังก์ชันสำหรับทดสอบรันในระบบ (มีครบถ้วนแล้ว) ---
function testRAWdataCommon() {
  var dummyEvent = {
    range: {
      getSheet: function() {
        return { getName: function() { return "RAWdataส่วนกลาง"; } };
      }
    },
    values: [
      "29/10/2026 23:35:00", // [0] คอลัมน์ A[cite: 3]
      "รับทราบ PDPA",         // [1] คอลัมน์ B[cite: 3]
      "05/11/2026",          // [2] คอลัมน์ C[cite: 3]
      "09:30:00",            // [3] คอลัมน์ D[cite: 3]
      "อาคาร B",              // [4] คอลัมน์ E[cite: 3]
      "สมคิด งานช่าง",         // [5] คอลัมน์ F[cite: 3]
      "082-111-2233",        // [6] คอลัมน์ G[cite: 3]
      "บจก. ส่วนกลางดูแลดี",    // [7] คอลัมน์ H[cite: 3]
      "ซ่อมแซมปั๊มน้ำ",         // [8] คอลัมน์ I[cite: 3]
      "ชั้น B1",             // [9] คอลัมน์ J[cite: 3]
      "https://drive.google.com/open?id=1A1Q-5ROHkTHJAnuFpbb1LwvwX9fEIguO", // [10] คอลัมน์ K[cite: 3]
      "4"                    // [11] คอลัมน์ L[cite: 3]
    ]
  };
  onFormSubmit(dummyEvent);
}

function testRAWdataRoom() {
  var dummyEvent = {
    range: {
      getSheet: function() {
        return { getName: function() { return "RAWdataห้องชุด"; } };
      }
    },
    values: [
      "29/10/2026 23:35:00", // [0] คอลัมน์ A[cite: 3]
      "รับทราบ PDPA",         // [1] คอลัมน์ B[cite: 3]
      "06/11/2026",          // [2] คอลัมน์ C[cite: 3]
      "10:00:00",            // [3] คอลัมน์ D[cite: 3]
      "อาคาร A",              // [4] คอลัมน์ E[cite: 3]
      "สุรชัย ช่างไฟ",         // [5] คอลัมน์ F[cite: 3]
      "089-999-8877",        // [6] คอลัมน์ G[cite: 3]
      "บริษัท ติดตั้งเน็ตไว",   // [7] คอลัมน์ H[cite: 3]
      "ติดตั้งเน็ต",           // [8] คอลัมน์ I[cite: 3]
      "101/55",              // [9] คอลัมน์ J[cite: 3]
      "5",                   // [10] คอลัมน์ K[cite: 3]
      "A0501",               // [11] คอลัมน์ L[cite: 3]
      "-",                   // [12] คอลัมน์ M[cite: 3]
      "https://drive.google.com/open?id=1A1Q-5ROHkTHJAnuFpbb1LwvwX9fEIguO", // [13] คอลัมน์ N[cite: 3]
      "5"                    // [14] คอลัมน์ O[cite: 3]
    ]
  };
  onFormSubmit(dummyEvent);
}
