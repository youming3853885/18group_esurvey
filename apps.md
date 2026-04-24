# Google Apps Script — 分析報告寄送

將以下內容整段貼回 Apps Script 編輯器中 `程式碼.gs`（蓋掉原本 `doPost`），存檔後重新部署（管理部署作業 → 編輯 → 版本選「新版本」→ 部署）。

本版用 **Template Literal（反引號 \`）** 取代前版的字串串接，避免引號跳脫引起的 `Invalid or unexpected token` 語法錯誤。資料呈現順序、全部文字、交叉分析判斷邏輯都與網頁端一致。

---

## 完整 `doPost`

```js
function doPost(e) {
  try {
    var data = JSON.parse(e.postData.contents);
    var email = data.email;
    var topGroup = data.topGroup;
    var riasec = data.riasec;
    var finalScores = data.finalScores;
    var interestScores = data.interestScores;
    var abilityScores = data.abilityScores;
    var careerPath = data.careerPath;
    var subjectWeights = data.subjectWeights;

    // === RIASEC 雷達圖 ===
    var chartConfig = {
      type: "radar",
      data: {
        labels: ["R 現實", "I 研究", "A 藝術", "S 社會", "E 企業", "C 傳統"],
        datasets: [{
          label: "性格分佈",
          data: [riasec.R, riasec.I, riasec.A, riasec.S, riasec.E, riasec.C],
          backgroundColor: "rgba(168, 85, 247, 0.2)",
          borderColor: "rgba(168, 85, 247, 1)",
          pointBackgroundColor: "rgba(168, 85, 247, 1)",
          borderWidth: 2
        }]
      },
      options: { legend: { display: false }, scale: { ticks: { min: 0, max: 100, stepSize: 20 } } }
    };
    var chartUrl = "https://quickchart.io/chart?c=" + encodeURIComponent(JSON.stringify(chartConfig));

    // === 取前三名 ===
    var sorted = Object.entries(finalScores).sort(function(a, b) { return b[1] - a[1]; }).slice(0, 3);
    var top1 = sorted[0][0];
    var top1Score = sorted[0][1];

    // === TOP 3 進度條 ===
    var top3BarsHtml = sorted.map(function(item, i) {
      var g = item[0], pct = item[1];
      return `
        <div style="margin-bottom:14px;">
          <div style="display:flex; justify-content:space-between; margin-bottom:6px; font-size:14px;">
            <span style="color:#333; font-weight:bold;">${i + 1}. ${g}</span>
            <span style="color:#a855f7; font-weight:bold;">${pct}%</span>
          </div>
          <div style="width:100%; height:10px; background:#ece9f5; border-radius:5px; overflow:hidden;">
            <div style="width:${pct}%; height:100%; background:linear-gradient(90deg,#6366f1,#a855f7);"></div>
          </div>
        </div>`;
    }).join("");

    // === 興趣 × 能力 交叉分析（建議用原始分數判斷） ===
    var crossRows = sorted.map(function(item) {
      var g = item[0];
      var intScore = Math.round(interestScores[g] || 0);
      var ablScore = Math.round(abilityScores[g] || 0);
      var iS = Math.ceil(intScore / 20), aS = Math.ceil(ablScore / 20);
      var iStars = `<span style="color:#f59e0b;">${"★".repeat(iS)}</span><span style="color:#ddd;">${"★".repeat(Math.max(0, 5 - iS))}</span>`;
      var aStars = `<span style="color:#10b981;">${"★".repeat(aS)}</span><span style="color:#ddd;">${"★".repeat(Math.max(0, 5 - aS))}</span>`;
      var advice, bg;
      if (ablScore >= 70) { advice = "✅ 完美對齊"; bg = "#065f46"; }
      else if (intScore > ablScore) { advice = "⚠️ 強化能力"; bg = "#92400e"; }
      else { advice = "💡 值得探索"; bg = "#1e3a8a"; }
      return `
        <tr style="border-bottom:1px solid #eee;">
          <td style="padding:10px;">${g}</td>
          <td style="text-align:center;">${iStars}</td>
          <td style="text-align:center;">${aStars}</td>
          <td style="text-align:right;"><span style="background:${bg}; color:#fff; padding:3px 8px; border-radius:4px; font-size:11px;">${advice}</span></td>
        </tr>`;
    }).join("");

    var crossHtml = `
      <table style="width:100%; border-collapse:collapse; font-size:13px;">
        <tr style="background:#f3e8ff; border-bottom:2px solid #e9d5ff;">
          <th style="padding:10px; text-align:left;">學群</th>
          <th style="padding:10px; text-align:center;">熱情</th>
          <th style="padding:10px; text-align:center;">能力</th>
          <th style="padding:10px; text-align:right;">建議</th>
        </tr>
        ${crossRows}
      </table>`;

    // === 職涯方向（膠囊 + 趨勢補充行） ===
    var trendRegex = /（趨勢：([^）]*)）/;
    var careerHtml = sorted.map(function(item) {
      var g = item[0];
      var raw = careerPath[g] || "專業人員";
      var match = raw.match(trendRegex);
      var trend = match ? match[1] : null;
      var clean = raw.replace(trendRegex, "");
      var pills = clean.split("、").map(function(r) {
        return `<span style="background:rgba(168,85,247,0.1); border:1px solid rgba(168,85,247,0.3); color:#6b21a8; padding:3px 10px; border-radius:12px; font-size:12px; display:inline-block; margin:3px 3px 3px 0;">${r}</span>`;
      }).join("");
      var trendHtml = trend
        ? `<div style="margin-top:8px; font-size:12px; color:#92400e; background:#fef3c7; border-left:3px solid #f59e0b; padding:6px 10px; border-radius:4px;">📈 <b>趨勢</b>：${trend}</div>`
        : "";
      return `
        <div style="margin-bottom:16px;">
          <div style="color:#a855f7; font-weight:bold; font-size:13px; margin-bottom:6px;">${g}</div>
          <div>${pills}</div>
          ${trendHtml}
        </div>`;
    }).join("");

    // === 參採科目比重 ===
    var subjectsHtml = sorted.map(function(item) {
      var g = item[0], w = subjectWeights[g] || {};
      var rows = Object.keys(w).map(function(s) {
        var pct = w[s];
        var barColor = pct > 70 ? "#f87171" : (pct > 40 ? "#6366f1" : "#4b5563");
        var textColor = pct > 70 ? "#f87171" : "#374151";
        return `
          <tr>
            <td style="width:35px; font-size:12px; color:#6b7280; padding:4px 0;">${s}</td>
            <td style="padding:4px 10px;">
              <div style="width:100%; height:8px; background:#f3f4f6; border-radius:4px; overflow:hidden;">
                <div style="width:${pct}%; height:100%; background:${barColor};"></div>
              </div>
            </td>
            <td style="width:40px; text-align:right; font-size:12px; font-weight:bold; color:${textColor}; padding:4px 0;">${pct}%</td>
          </tr>`;
      }).join("");
      return `
        <div style="background:#fff; border:1px solid #e9d5ff; padding:15px; border-radius:12px; margin-bottom:12px;">
          <div style="color:#333; font-weight:bold; margin-bottom:10px; font-size:13px;">${g}</div>
          <table style="width:100%; border-collapse:collapse;">${rows}</table>
        </div>`;
    }).join("");

    // === 品牌資源 ===
    var logoUrl = "https://raw.githubusercontent.com/youming3853885/18group_esurvey/main/kut_logo.png";
    var mascotUrl = "https://raw.githubusercontent.com/youming3853885/18group_esurvey/main/%E5%B0%8F%E7%B7%A8.webp";
    var subject = `【職群怪物圖鑑】${topGroup}：完整深度分析報告`;

    // === 整體 HTML ===
    var htmlBody = `
<div style="font-family:'Microsoft JhengHei',sans-serif; max-width:650px; margin:auto; border:1px solid #eee; padding:40px; border-radius:20px; color:#333; line-height:1.6; background:#fff;">

  <div style="text-align:center; margin-bottom:20px;">
    <img src="${logoUrl}" style="max-width:160px; height:auto;" alt="名師學院">
  </div>

  <div style="text-align:center; margin-bottom:25px; padding-bottom:18px; border-bottom:2px solid #f3e8ff;">
    <h2 style="color:#6b21a8; margin:0 0 8px 0; font-size:24px;">你的職涯羅盤 — 深度分析報告</h2>
    <div style="font-size:14px; color:#6b7280;">三維度分析完成 • 最高適配指數 ${top1Score}%</div>
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:18px;">
    <h3 style="color:#6b21a8; margin:0 0 14px 0; font-size:16px;">📊 RIASEC 人格輪廓</h3>
    <div style="text-align:center; margin-bottom:14px;"><img src="${chartUrl}" style="max-width:100%; border-radius:8px;"></div>
    <table style="width:100%; border-collapse:collapse; font-size:12px; color:#4b5563;">
      <tr>
        <td style="padding:4px 8px; width:50%;"><b style="color:#111;">R 現實型</b>：動手實作、技術操作</td>
        <td style="padding:4px 8px; width:50%;"><b style="color:#111;">I 研究型</b>：邏輯分析、科學探索</td>
      </tr>
      <tr>
        <td style="padding:4px 8px;"><b style="color:#111;">A 藝術型</b>：創作表達、審美創新</td>
        <td style="padding:4px 8px;"><b style="color:#111;">S 社會型</b>：助人服務、溝通協作</td>
      </tr>
      <tr>
        <td style="padding:4px 8px;"><b style="color:#111;">E 企業型</b>：領導組織、說服影響</td>
        <td style="padding:4px 8px;"><b style="color:#111;">C 傳統型</b>：系統規劃、精確執行</td>
      </tr>
    </table>
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:18px;">
    <h3 style="color:#6b21a8; margin:0 0 14px 0; font-size:16px;">🏆 你的命定學群 TOP 3</h3>
    ${top3BarsHtml}
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:18px;">
    <h3 style="color:#6b21a8; margin:0 0 4px 0; font-size:16px;">🔍 興趣 × 能力 交叉分析</h3>
    <p style="font-size:13px; color:#6b7280; margin:0 0 14px 0;">看看你的熱情和你的能力有沒有對齊</p>
    ${crossHtml}
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:18px;">
    <h3 style="color:#6b21a8; margin:0 0 14px 0; font-size:16px;">💼 適合的職涯方向</h3>
    ${careerHtml}
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:18px;">
    <h3 style="color:#6b21a8; margin:0 0 14px 0; font-size:16px;">📌 雙軌行動計畫 (TOP 1: ${top1})</h3>
    <div style="border-left:3px solid #a855f7; padding-left:14px;">
      <div style="background:rgba(168,85,247,0.08); padding:12px; border-radius:8px; margin-bottom:10px;">
        <div style="color:#a855f7; font-size:12px; font-weight:bold; margin-bottom:4px;">📱 今天可以做的事</div>
        <div style="color:#333; font-size:13px;">搜尋一篇關於「${top1}」的一日工作 Vlog，看看專業人士都在做什麼。</div>
      </div>
      <div style="background:rgba(52,211,153,0.08); padding:12px; border-radius:8px; margin-bottom:10px;">
        <div style="color:#059669; font-size:12px; font-weight:bold; margin-bottom:4px;">🗓️ 這個月可以做的事</div>
        <div style="color:#333; font-size:13px;">找尋相關學群的線上體驗課程，或者參加一場實體的職群講座。</div>
      </div>
      <div style="background:rgba(251,191,36,0.1); padding:12px; border-radius:8px;">
        <div style="color:#d97706; font-size:12px; font-weight:bold; margin-bottom:4px;">🎯 一年後要確認的事</div>
        <div style="color:#333; font-size:13px;">強化與該學群相關的核心科目（如數A、英文），了解大學入學的甄試條件。</div>
      </div>
    </div>
  </div>

  <div style="background:#faf5ff; padding:20px; border-radius:16px; margin-bottom:28px;">
    <h3 style="color:#6b21a8; margin:0 0 14px 0; font-size:16px;">📚 參採科目比重參考 (TOP 3)</h3>
    ${subjectsHtml}
  </div>

  <div style="text-align:center; padding:20px; background:#f8f9fa; border-radius:16px;">
    <img src="${mascotUrl}" style="width:70px; height:auto; margin-bottom:8px;" alt="小編">
    <p style="margin:0; font-weight:bold; color:#6b21a8;">名師學院怪物小編 辛勤製作</p>
    <p style="margin:5px 0; font-size:12px; color:#666;">官方網站：<a href="https://www.kut.com.tw/" style="color:#a855f7;">https://www.kut.com.tw/</a></p>
    <p style="margin:0; font-size:11px; color:#999;">如有問題請洽：kutfb@kut.com.tw</p>
  </div>

</div>`;

    MailApp.sendEmail({
      to: email,
      subject: subject,
      htmlBody: htmlBody,
      name: "名師學院",
      replyTo: "kutfb@kut.com.tw"
    });

    return ContentService.createTextOutput("Success").setMimeType(ContentService.MimeType.TEXT);
  } catch (err) {
    return ContentService.createTextOutput("Error: " + err.toString()).setMimeType(ContentService.MimeType.TEXT);
  }
}
```

---

## 為什麼前一版會壞？

原版用 `'...字串...' + 變數 + '...字串...'` 串接，過程中需要 `\'` 跳脫內嵌引號。當程式碼從網頁複製到 Apps Script 編輯器時，某些環境會把 ASCII 單引號 `'` 自動替換成印刷用的智慧引號 `'` 或 `'`，導致 JS 解析失敗並報 `Invalid or unexpected token`。

本版改用 Template Literal（反引號 \`\`\`\`\`），大段 HTML 字串不需要任何跳脫，複製貼上時也不會被 auto-correct 影響。Apps Script V8 runtime（預設）完整支援。

---

## 部署步驟

1. 打開 Apps Script 編輯器 → 開啟 `程式碼.gs`
2. 全選原本的 `doPost` 函式 → 整段覆蓋貼上
3. 檔案 → 儲存（Ctrl + S）
4. 右上「部署」→「管理部署作業」→ 點鉛筆編輯 → 版本選「新版本」→ 部署
5. 第一次執行若要求授權，按同意即可
6. 重新測試：網頁跑測驗 → 輸入 email → 檢查收信

---

## 功能對應表（與網頁報告一致）

| 順位 | 區塊 | 來源欄位 |
|---|---|---|
| ① | 標題「你的職涯羅盤 — 深度分析報告」+ 最高適配指數 | `finalScores` 最高者 |
| ② | 📊 RIASEC 人格輪廓 + 雷達圖 + 6 型說明 | `riasec`（QuickChart 繪製） |
| ③ | 🏆 你的命定學群 TOP 3 進度條 | `finalScores` 前 3 名 |
| ④ | 🔍 興趣 × 能力 交叉分析表 | `interestScores`、`abilityScores` |
| ⑤ | 💼 適合的職涯方向（膠囊 + 趨勢補充行，**信件特有**） | `careerPath` |
| ⑥ | 📌 雙軌行動計畫（今天／這個月／一年後） | `top1`（依題目帶入學群名稱） |
| ⑦ | 📚 參採科目比重參考 (TOP 3) | `subjectWeights` |

順序、文字、判斷邏輯皆與 `index.html` 的詳細分析區塊一對一對齊。
