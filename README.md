# Study-report
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>写真撮影時刻グラフ</title>
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Kaku Gothic ProN",
      "Yu Gothic", Meiryo, sans-serif;
    background: #f5f5f5;
    color: #222;
  }
  .wrap {
    max-width: 1000px;
    margin: 0 auto;
    padding: 20px;
  }
  h1 { font-size: 24px; margin: 0 0 16px; }
  .panel {
    background: white;
    border-radius: 14px;
    padding: 16px;
    box-shadow: 0 2px 12px rgba(0,0,0,.08);
  }
  .controls {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 14px;
  }
  button, .file-label {
    appearance: none;
    border: 0;
    border-radius: 9px;
    padding: 10px 14px;
    background: #222;
    color: white;
    font-size: 14px;
    cursor: pointer;
  }
  .file-label { display: inline-block; }
  .file-label input { display: none; }
  button.secondary { background: #777; }
  button:disabled { opacity: .45; cursor: default; }
  .status {
    font-size: 13px;
    color: #666;
    margin: 6px 0 12px;
    white-space: pre-wrap;
  }
  .chart-box {
    width: 100%;
    overflow-x: auto;
    border: 1px solid #ddd;
    border-radius: 10px;
    background: white;
  }
  canvas {
    display: block;
    width: 100%;
    min-width: 700px;
    height: 430px;
  }
  .note {
    margin-top: 12px;
    color: #777;
    font-size: 12px;
    line-height: 1.6;
  }
</style>
</head>
<body>
<div class="wrap">
  <h1>写真撮影時刻グラフ</h1>

  <div class="panel">
    <div class="controls">
      <label class="file-label">
        写真を選択
        <input id="files" type="file" accept="image/*" multiple>
      </label>
      <button id="save" disabled>グラフをPNG保存</button>
      <button id="clear" class="secondary">クリア</button>
    </div>

    <div id="status" class="status">写真を選択してください。</div>

    <div class="chart-box">
      <canvas id="chart" width="1000" height="430"></canvas>
    </div>

    <div class="note">
      EXIFの撮影日時（DateTimeOriginal）を優先します。取得できない場合はファイルの更新日時を使用します。
    </div>
  </div>
</div>

<script>
const fileInput = document.getElementById("files");
const saveBtn = document.getElementById("save");
const clearBtn = document.getElementById("clear");
const statusEl = document.getElementById("status");
const canvas = document.getElementById("chart");
const ctx = canvas.getContext("2d");

let rows = [];

function readAscii(view, offset, length) {
  let s = "";
  for (let i = 0; i < length; i++) {
    const c = view.getUint8(offset + i);
    if (c === 0) break;
    s += String.fromCharCode(c);
  }
  return s;
}

function exifDateToDate(s) {
  const m = /^(\d{4}):(\d{2}):(\d{2})[ T](\d{2}):(\d{2}):(\d{2})/.exec(s || "");
  if (!m) return null;
  const d = new Date(
    Number(m[1]), Number(m[2]) - 1, Number(m[3]),
    Number(m[4]), Number(m[5]), Number(m[6])
  );
  return isNaN(d.getTime()) ? null : d;
}

function parseExifDate(buffer) {
  const v = new DataView(buffer);
  if (v.byteLength < 4 || v.getUint16(0, false) !== 0xFFD8) return null;

  let p = 2;
  while (p + 4 <= v.byteLength) {
    if (v.getUint8(p) !== 0xFF) { p++; continue; }

    const marker = v.getUint8(p + 1);
    if (marker === 0xDA || marker === 0xD9) break;

    const len = v.getUint16(p + 2, false);
    if (len < 2 || p + 2 + len > v.byteLength) break;

    if (marker === 0xE1 && len >= 8) {
      const exifStart = p + 4;

      if (readAscii(v, exifStart, 6) === "Exif\u0000\u0000") {
        const tiff = exifStart + 6;
        if (tiff + 8 > v.byteLength) return null;

        const endianMark = v.getUint16(tiff, false);
        let little;
        if (endianMark === 0x4949) little = true;
        else if (endianMark === 0x4D4D) little = false;
        else return null;

        const u16 = o => v.getUint16(o, little);
        const u32 = o => v.getUint32(o, little);

        function valueOffset(entry, type, count) {
          const sizes = {1:1, 2:1, 3:2, 4:4, 5:8, 7:1, 9:4, 10:8};
          const size = sizes[type];
          if (!size) return null;
          const bytes = size * count;
          if (bytes <= 4) return entry + 8;
          const off = u32(entry + 8);
          const pos = tiff + off;
          if (pos < 0 || pos + bytes > v.byteLength) return null;
          return pos;
        }

        function readDateFromIfd(ifdOffset, visited = new Set()) {
          if (visited.has(ifdOffset)) return null;
          visited.add(ifdOffset);
          if (ifdOffset < tiff || ifdOffset + 2 > v.byteLength) return null;

          const count = u16(ifdOffset);

          // Prefer DateTimeOriginal (0x9003), then fall back to DateTime (0x0132).
          let fallback = null;

          for (let i = 0; i < count; i++) {
            const e = ifdOffset + 2 + i * 12;
            if (e + 12 > v.byteLength) break;

            const tag = u16(e);
            const type = u16(e + 2);
            const countValue = u32(e + 4);

            if (tag !== 0x9003 && tag !== 0x0132 && tag !== 0x8769) continue;

            if (tag === 0x8769 && type === 4 && countValue >= 1) {
              const exifIfd = tiff + u32(e + 8);
              const d = readDateFromIfd(exifIfd, new Set(visited));
              if (d) return d;
              continue;
            }

            if ((tag === 0x9003 || tag === 0x0132) && type === 2) {
              const pos = valueOffset(e, type, countValue);
              if (pos == null || pos + Math.min(countValue, 64) > v.byteLength) continue;

              const s = readAscii(v, pos, Math.min(countValue, 64));
              const d = exifDateToDate(s);
              if (d) {
                if (tag === 0x9003) return d;
                fallback = d;
              }
            }
          }
          return fallback;
        }

        const ifd0 = tiff + u32(tiff + 4);
        const result = readDateFromIfd(ifd0);
        if (result) return result;
      }
    }

    p += 2 + len;
  }
  return null;
}

async function getCaptureTime(file) {
  try {
    const buffer = await file.arrayBuffer();
    const exifDate = parseExifDate(buffer);
    if (exifDate) return { time: exifDate, source: "EXIF" };
  } catch (_) {}
  return { time: new Date(file.lastModified), source: "ファイル更新日時" };
}

function fmtTime(d) {
  return String(d.getHours()).padStart(2, "0") + ":" +
         String(d.getMinutes()).padStart(2, "0");
}

function draw() {
  const W = canvas.width, H = canvas.height;
  ctx.clearRect(0, 0, W, H);

  ctx.fillStyle = "#fff";
  ctx.fillRect(0, 0, W, H);

  const left = 78, right = 25, top = 25, bottom = 55;
  const pw = W - left - right;
  const ph = H - top - bottom;

  if (!rows.length) {
    ctx.fillStyle = "#888";
    ctx.font = "16px sans-serif";
    ctx.textAlign = "center";
    ctx.fillText("写真を選択すると撮影時刻が表示されます", W / 2, H / 2);
    return;
  }

  const mins = rows.map(r => r.time.getHours() * 60 + r.time.getMinutes() + r.time.getSeconds() / 60);
  let minM = Math.floor(Math.min(...mins) / 60) * 60;
  let maxM = Math.ceil(Math.max(...mins) / 60) * 60;
  if (maxM <= minM) { minM -= 30; maxM += 30; }
  minM = Math.max(0, minM);
  maxM = Math.min(1440, maxM);
  if (maxM - minM < 60) {
    const mid = (minM + maxM) / 2;
    minM = Math.max(0, Math.floor((mid - 30) / 10) * 10);
    maxM = Math.min(1440, minM + 60);
  }

  // grid / Y labels
  ctx.font = "12px sans-serif";
  ctx.textAlign = "right";
  ctx.textBaseline = "middle";

  const step = Math.max(10, Math.ceil((maxM - minM) / 8 / 10) * 10);
  for (let m = Math.ceil(minM / step) * step; m <= maxM; m += step) {
    const y = top + (maxM - m) / (maxM - minM) * ph;
    ctx.strokeStyle = "#e5e5e5";
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(left, y);
    ctx.lineTo(left + pw, y);
    ctx.stroke();

    const hh = String(Math.floor(m / 60)).padStart(2, "0");
    const mm = String(m % 60).padStart(2, "0");
    ctx.fillStyle = "#666";
    ctx.fillText(`${hh}:${mm}`, left - 10, y);
  }

  // axes
  ctx.strokeStyle = "#555";
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.moveTo(left, top);
  ctx.lineTo(left, top + ph);
  ctx.lineTo(left + pw, top + ph);
  ctx.stroke();

  // X labels and dots
  const n = rows.length;
  rows.forEach((r, i) => {
    // ドットは常に左端から1枚目、2枚目、3枚目…の順に配置
    const xStep = n > 1 ? pw / Math.max(n - 1, 1) : 0;
    const x = left + i * xStep;
    const m = r.time.getHours() * 60 + r.time.getMinutes() + r.time.getSeconds() / 60;
    const y = top + (maxM - m) / (maxM - minM) * ph;

    ctx.strokeStyle = "#eee";
    ctx.beginPath();
    ctx.moveTo(x, top + ph);
    ctx.lineTo(x, top + ph + 6);
    ctx.stroke();

    ctx.fillStyle = "#555";
    ctx.font = "11px sans-serif";
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    ctx.fillText(String(i + 1), x, top + ph + 10);

    ctx.beginPath();
    ctx.arc(x, y, 6, 0, Math.PI * 2);
    ctx.fillStyle = "#222";
    ctx.fill();

    ctx.fillStyle = "#555";
    ctx.font = "11px sans-serif";
    ctx.textBaseline = "bottom";
    ctx.fillText(fmtTime(r.time), x, y - 9);
  });

  ctx.fillStyle = "#333";
  ctx.font = "13px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "alphabetic";
  ctx.fillText("何枚目", left + pw / 2, H - 10);

  ctx.save();
  ctx.translate(17, top + ph / 2);
  ctx.rotate(-Math.PI / 2);
  ctx.fillText("撮影時刻", 0, 0);
  ctx.restore();
}

fileInput.addEventListener("change", async () => {
  const files = Array.from(fileInput.files || []);
  if (!files.length) return;

  statusEl.textContent = "撮影時刻を読み込んでいます…";

  const importedRows = await Promise.all(
    files.map(async (file) => {
      const result = await getCaptureTime(file);
      return {
        time: new Date(result.time.getTime()),
        source: String(result.source),
        name: String(file.name)
      };
    })
  );

  // 既に入っている写真を残したまま、追加した順番を維持する
  // 横軸は「何枚目」なので、撮影時刻では並べ替えない
  rows = rows.concat(importedRows);
  draw();
  saveBtn.disabled = false;

  const exifCount = rows.filter(r => r.source === "EXIF").length;
  const fallbackCount = rows.length - exifCount;
  statusEl.textContent =
    `${rows.length}枚を読み込みました。\n` +
    `EXIF撮影日時: ${exifCount}枚 / ファイル更新日時: ${fallbackCount}枚`;

  // 次回も同じファイルを選択できるようにする
  fileInput.value = "";
});

saveBtn.addEventListener("click", () => {
  const a = document.createElement("a");
  a.download = "photo-time-graph.png";
  a.href = canvas.toDataURL("image/png");
  a.click();
});

clearBtn.addEventListener("click", () => {
  rows = [];
  fileInput.value = "";
  saveBtn.disabled = true;
  statusEl.textContent = "写真を選択してください。";
  draw();
});

draw();
</script>
</body>
</html>says 
