```dataviewjs
function extractSecs(fileContent, subjectHeader) {
  const safeHeader = subjectHeader.replace(/\*/g, "\\*");
  const regex = new RegExp(safeHeader + "[\\s\\S]*?```simple-time-tracker\\s*(\\{[\\s\\S]*?\\})\\s*```", "i");
  const match = fileContent.match(regex);
  if (!match) return 0;
  try {
    const data = JSON.parse(match[1]);
    let totalMs = 0;
    if (data.entries) {
      for (let entry of data.entries) {
        if (entry.startTime && entry.endTime) totalMs += (new Date(entry.endTime).getTime() - new Date(entry.startTime).getTime());
        if (entry.subEntries) {
          for (let sub of entry.subEntries) {
            if (sub.startTime && sub.endTime) totalMs += (new Date(sub.endTime).getTime() - new Date(sub.startTime).getTime());
          }
        }
      }
    } return Math.round(totalMs / 1000);
  } catch (e) { return 0; }
}

function formatTime(totalSecs, isBold = false) {
  const sortKey = String(totalSecs).padStart(7, '0');
  if (!totalSecs) {
      const dash = "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
      return `<!--${sortKey}-->${isBold ? `**${dash}**` : dash}`;
  }
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  let str = "";
  if (h > 0) str += `${h}h `;
  if (m > 0 || h > 0) str += `${m}m `;
  str += `${s}s`;
  const span = `<span style='font-size: 0.8em; white-space: nowrap;'>${str.trim()}</span>`;
  return `<!--${sortKey}-->${isBold ? `**${span}**` : span}`;
}

const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc');
const groupedByMonth = {};

for (let p of allPages) {
    const monthName = window.moment(p.file.name).format("MMMM YYYY");
    if (!groupedByMonth[monthName]) groupedByMonth[monthName] = [];
    groupedByMonth[monthName].push(p);
}

for (let month in groupedByMonth) {
    const tableData = [];
    
    for (let p of groupedByMonth[month]) {
        const content = await dv.io.load(p.file.path);
        const m = extractSecs(content, "**Math**");
        const ph = extractSecs(content, "**Physics**");
        const ch = extractSecs(content, "**Chemistry**");
        const total = m + ph + ch;
        
        const dayNum = p.file.name.slice(-2);
        const dayLink = `<!--${dayNum}-->[[${p.file.path}|${dayNum}]]`;
        
        tableData.push([
            dayLink, 
            formatTime(m), 
            formatTime(ph), 
            formatTime(ch), 
            formatTime(total, true)
        ]);
    }
    
    const mdTable = dv.markdownTable(["Day", "Math", "Phy", "Chem", "Tot"], tableData);
    const callout = `> [!info]- 📅 ${month}\n${mdTable.split('\n').map(line => '> ' + line).join('\n')}`;
    dv.paragraph(callout);
}
```
## Recent Study Sessions

```dataviewjs
function extractSecs(fileContent, subjectHeader) {
  const safeHeader = subjectHeader.replace(/\*/g, "\\*");
  const regex = new RegExp(safeHeader + "[\\s\\S]*?```simple-time-tracker\\s*(\\{[\\s\\S]*?\\})\\s*```", "i");
  const match = fileContent.match(regex);
  if (!match) return 0;
  try {
    const data = JSON.parse(match[1]);
    let totalMs = 0;
    if (data.entries) {
      for (let entry of data.entries) {
        if (entry.startTime && entry.endTime) totalMs += (new Date(entry.endTime).getTime() - new Date(entry.startTime).getTime());
        if (entry.subEntries) {
          for (let sub of entry.subEntries) {
            if (sub.startTime && sub.endTime) totalMs += (new Date(sub.endTime).getTime() - new Date(sub.startTime).getTime());
          }
        }
      }
    } return Math.round(totalMs / 1000);
  } catch (e) { return 0; }
}

function formatTime(totalSecs, isBold = false) {
  const sortKey = String(totalSecs).padStart(7, '0'); // Hidden padded number for perfect mathematical sorting
  if (!totalSecs) {
      const dash = "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
      return `<!--${sortKey}-->${isBold ? `**${dash}**` : dash}`;
  }
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  let str = "";
  if (h > 0) str += `${h}h `;
  if (m > 0 || h > 0) str += `${m}m `;
  str += `${s}s`;
  const span = `<span style='font-size: 0.8em; white-space: nowrap;'>${str.trim()}</span>`;
  return `<!--${sortKey}-->${isBold ? `**${span}**` : span}`;
}

const recentPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc').limit(3);
const recentTableData = [];

for (let p of recentPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = extractSecs(fileContent, "**Math**");
  const ph = extractSecs(fileContent, "**Physics**");
  const ch = extractSecs(fileContent, "**Chemistry**");
  const total = m + ph + ch;
  
  const dayNum = p.file.name.slice(-2);
  const dayLink = `<!--${dayNum}-->[[${p.file.path}|${dayNum}]]`;
  
  recentTableData.push([
    dayLink, 
    formatTime(m), 
    formatTime(ph), 
    formatTime(ch), 
    formatTime(total, true)
  ]);
}

dv.table(["Day", "Math", "Phy", "Chem", "Tot"], recentTableData);
```
