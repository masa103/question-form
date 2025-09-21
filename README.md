# question-form
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>質問フォーム</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: sans-serif; padding: 1em; background: #f9f9f9; }
    h2 { color: #007aff; }
    label { display: block; margin-top: 1em; font-weight: bold; }
    input, textarea, select, button {
      width: 100%; padding: 0.5em; margin-top: 0.5em;
      font-size: 1em; box-sizing: border-box;
    }
    button {
      background: #007aff; color: white; border: none;
      border-radius: 4px; margin-top: 1.5em;
    }
  </style>
</head>
<body>
  <h2>質問フォーム</h2>
  <form id="questionForm">
    <label>質問件名</label>
    <input type="text" name="title" required>

    <label>質問内容</label>
    <textarea name="question" required></textarea>

    <label>ステータス</label>
    <select name="status" required>
      <option value="">選択してください</option>
      <option value="新規">新規</option>
      <option value="未着手">未着手</option>
      <option value="着手中">着手中</option>
      <option value="再質問">再質問</option>
    </select>

    <label>過去の質問選択</label>
    <select name="selectedTitle" id="titleSelect">
      <option value="">選択してください</option>
    </select>

    <label>メールアドレス</label>
    <input type="email" name="askerEmail">

    <button type="submit">送信</button>
  </form>

  <script src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
  <script>
    const SCRIPT_URL = "https://script.google.com/macros/s/AKfycb.../exec"; // ← GASのWeb App URLに置き換えてください
    const LIFF_ID = "2008085800-85LEKJG6"; // ← 質問者LIFF ID

    async function initLIFF() {
      await liff.init({ liffId: LIFF_ID });
      const profile = await liff.getProfile();
      window.uid = profile.userId;
      loadTitles();
    }

    async function loadTitles() {
      try {
        const res = await fetch(SCRIPT_URL + "?mode=titles");
        const text = await res.text();
        const select = document.getElementById("titleSelect");
        text.split("\n").forEach(line => {
          const [label, value] = line.split("|");
          if (label && value) {
            const option = document.createElement("option");
            option.textContent = label;
            option.value = line;
            select.appendChild(option);
          }
        });
      } catch (err) {
        alert("過去の質問一覧の取得に失敗しました");
      }
    }

    document.getElementById("questionForm").addEventListener("submit", async (e) => {
      e.preventDefault();
      const form = e.target;
      const data = new FormData(form);
      data.append("uid", window.uid);
      data.append("role", "質問者");

      try {
        const res = await fetch(SCRIPT_URL, {
          method: "POST",
          body: data
        });
        const result = await res.text();
        alert(result);
        liff.closeWindow();
      } catch (err) {
        alert("送信に失敗しました");
      }
    });

    initLIFF();
  </script>
</body>
</html>
