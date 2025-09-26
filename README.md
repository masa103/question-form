function doGet(e) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const settingSheet = ss.getSheetByName("設定");
  const qaSheet = ss.getSheetByName("質問・回答");

  if (!settingSheet) return ContentService.createTextOutput("設定シートが見つかりません");

  // LIFF ID取得
  if (e?.parameter?.mode === "getLiffId") {
    const liffId = settingSheet.getRange("B12").getValue();
    return ContentService.createTextOutput(liffId);
  }

  // ステータスが「未着手・着手中」の件名一覧取得
  if (e?.parameter?.mode === "getTitles") {
    if (!qaSheet) return ContentService.createTextOutput("質問・回答シートが見つかりません");

    const values = qaSheet.getDataRange().getValues();
    const titles = new Set();
    const allowedStatus = ["未着手", "着手中"];

    for (let i = 1; i < values.length; i++) {
      const title = values[i][6];     // G列：件名
      const status = values[i][14];   // O列：ステータス

      if (title && allowedStatus.includes(status)) {
        titles.add(title);
      }
    }

    return ContentService.createTextOutput(JSON.stringify([...titles]));
  }

  return HtmlService.createHtmlOutputFromFile("index");
}

function doPost(e) {
  Logger.log("✅ doPost triggered");

  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName("質問・回答");
    const settingSheet = ss.getSheetByName("設定");
    const nameSheet = ss.getSheetByName("質問者名リスト");
    const taskSheet = ss.getSheetByName("タスク一覧表");

    if (!sheet || !settingSheet || !nameSheet || !taskSheet) {
      throw new Error("必要なシートが見つかりません");
    }

    const folderId = settingSheet.getRange("B14").getValue();
    const folder = DriveApp.getFolderById(folderId);

    const now = new Date();
    const qid = "Q" + Utilities.formatDate(now, "Asia/Tokyo", "yyyyMMddHHmmss");
    const groupId = "G" + Utilities.formatDate(now, "Asia/Tokyo", "yyyyMMddHHmmss");
    const data = e.parameter || {};
    Logger.log("📦 受信データ: " + JSON.stringify(data));

    let photo1Url = "", photo2Url = "";
    // 画像は後で処理するため、ここでは空のままでOK

    const uid = data.uid || "";
    const groupName = data.responder || "";
    Logger.log("🆔 UID: " + uid);

    let roomNumber = "", ownerName = "", ownerEmail = "";
    const nameValues = nameSheet.getDataRange().getValues();
    for (let i = 1; i < nameValues.length; i++) {
      const rowRoom = nameValues[i][0];
      const rowUid = nameValues[i][3];

      if (rowUid === uid) {
        roomNumber = rowRoom || "";
        ownerName = nameValues[i][1] || "";
        ownerEmail = nameValues[i][2] || "";
        nameSheet.getRange(i + 1, 5).setValue(groupName);
        nameSheet.getRange(i + 1, 6).setValue(groupId);
        Logger.log("🔗 UID一致: row " + (i + 1));
        break;
      }

      if (!rowUid && rowRoom) {
        nameSheet.getRange(i + 1, 4).setValue(uid);
        nameSheet.getRange(i + 1, 5).setValue(groupName);
        nameSheet.getRange(i + 1, 6).setValue(groupId);
        roomNumber = rowRoom;
        ownerName = nameValues[i][1] || "";
        ownerEmail = nameValues[i][2] || "";
        Logger.log("🔗 部屋番号一致でUID登録: row " + (i + 1));
        break;
      }
    }

    const newRow = [
      qid,
      Utilities.formatDate(now, "Asia/Tokyo", "yyyy/MM/dd HH:mm:ss"),
      uid,
      roomNumber,
      ownerName,
      groupName,
      data.title || "",
      data.question || "",
      photo1Url,
      Utilities.formatDate(now, "Asia/Tokyo", "yyyy/MM/dd HH:mm:ss"),
      data.answer || "",
      data.cause || "",
      photo2Url,
      data.status || "",
      groupName,
      ownerEmail,
      ""
    ];

    const title = data.title || "";
    const sheetValues = sheet.getDataRange().getValues();
    let insertIndex = sheetValues.length + 1;
    for (let i = 1; i < sheetValues.length; i++) {
      if (sheetValues[i][6] === title) {
        insertIndex = i + 1;
      }
    }

    sheet.insertRows(insertIndex, 1);
    sheet.getRange(insertIndex, 1, 1, newRow.length).setValues([newRow]);
    Logger.log("📋 質問・回答シートに挿入: row " + insertIndex);

    const taskRow = [
      qid,
      Utilities.formatDate(now, "Asia/Tokyo", "yyyy/MM/dd HH:mm:ss"),
      data.title || "",
      data.question || "",
      data.status || "",
      uid,
      roomNumber,
      ownerName,
      groupName,
      photo1Url
    ];
    taskSheet.appendRow(taskRow);
    Logger.log("📋 タスク一覧表に記録");

    return ContentService.createTextOutput("記録完了：" + qid);
  } catch (err) {
    Logger.log("❌ Error: " + err.message);
    Logger.log("🪵 Stack: " + err.stack);
    return ContentService.createTextOutput("エラー：" + err.message);
  }
}
