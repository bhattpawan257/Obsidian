
---

## Total Study Activity

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
  const safeHeader = subjectHeader.replace(/\*/g, "\\*");
  const regex = new RegExp(safeHeader + "[\\s\\S]*?```simple-time-tracker\\s*(\\{[\\s\\S]*?\\})\\s*```", "i");
  const match = fileContent.match(regex);
  if (!match) return 0;
  
  try {
    const data = JSON.parse(match[1]);
    let totalMs = 0;
    if (data.entries) {
      for (let entry of data.entries) {
        if (entry.startTime && entry.endTime) {
          totalMs += (new Date(entry.endTime).getTime() - new Date(entry.startTime).getTime());
        }
        if (entry.subEntries) {
          for (let sub of entry.subEntries) {
            if (sub.startTime && sub.endTime) {
              totalMs += (new Date(sub.endTime).getTime() - new Date(sub.startTime).getTime());
            }
          }
        }
      }
    } 
    return Math.round(totalMs / 60000);
  } catch (e) {
    return 0;
  }
}

const totalData = [];
const totalPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of totalPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = extractMinsForSubject(fileContent, "**Math**");
  const ph = extractMinsForSubject(fileContent, "**Physics**");
  const ch = extractMinsForSubject(fileContent, "**Chemistry**");
  const totalMins = m + ph + ch;

  if (totalMins > 0) {
    totalData.push({ date: p.file.name, value: totalMins });
  }
}

const totalCalendarData = {
  title: "Total Study Activity (Minutes)",
  data: totalData,
  cellStyle: { minWidth: "25px", minHeight: "25px" },
  showAllDayLabel: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 90, color: "#0e4429" },       
    { min: 91, max: 180, color: "#006d32" },      
    { min: 181, max: 270, color: "#26a641" },     
    { min: 271, max: 999999, color: "#39d353" }  
  ],
  onCellClick: (item) => {
    if (item.value) {
      const h = Math.floor(item.value / 60);
      const m = item.value % 60;
      let timeStr = "";
      if (h > 0) timeStr += `${h}h `;
      if (m > 0 || h === 0) timeStr += `${m}min`;
      new Notice(`${timeStr.trim()} studied on ${item.date}`);
    } else {
      new Notice(`0min studied on ${item.date}`);
    }
  }
};
renderContributionGraph(this.container, totalCalendarData);
```

---
```dataviewjs
const btn = this.container.createEl('button', { text: "📝 Open Today's Note", cls: "mod-cta" });
btn.style.width = "100%";
btn.style.padding = "12px";
btn.style.fontSize = "1.2em";
btn.style.fontWeight = "bold";
btn.style.marginBottom = "15px";

btn.onclick = () => {
    app.commands.executeCommandById("daily-notes");
};
```
---

## Topics Covered (Last 7 Days)

```dataviewjs
function extractTopics(fileContent, subjectHeader) {
  const safeHeader = subjectHeader.replace(/\*/g, "\\*");
  const regex = new RegExp(safeHeader + "[\\s\\S]*?```simple-time-tracker\\s*(\\{[\\s\\S]*?\\})\\s*```", "i");
  const match = fileContent.match(regex);
  let topics = new Set();
  if (!match) return [];
  
  try {
    const data = JSON.parse(match[1]);
    if (data.entries) {
      for (let entry of data.entries) {
        const t = entry.name ? entry.name.trim() : "";
        if (t && !t.toLowerCase().startsWith("segment") && !t.toLowerCase().startsWith("part")) topics.add(t);
        if (entry.subEntries) {
          for (let sub of entry.subEntries) {
            const st = sub.name ? sub.name.trim() : "";
            if (st && !st.toLowerCase().startsWith("segment") && !st.toLowerCase().startsWith("part")) topics.add(st);
          }
        }
      }
    }
  } catch (e) {}
  return Array.from(topics);
}

const sevenDaysAgo = window.moment().subtract(7, 'days').format("YYYY-MM-DD");
const weekPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/) && p.file.name >= sevenDaysAgo);

let mathTopics = new Set(), phyTopics = new Set(), chemTopics = new Set();

for (let p of weekPages) {
  const content = await dv.io.load(p.file.path);
  extractTopics(content, "**Math**").forEach(t => mathTopics.add(t));
  extractTopics(content, "**Physics**").forEach(t => phyTopics.add(t));
  extractTopics(content, "**Chemistry**").forEach(t => chemTopics.add(t));
}

dv.table(["Subject", "Concepts & Topics"], [
   ["📐 Math", Array.from(mathTopics).join(", ") || "-"],
   ["🍎 Physics", Array.from(phyTopics).join(", ") || "-"],
   ["🧪 Chemistry", Array.from(chemTopics).join(", ") || "-"]
]);
```

---

## Recent Study Sessions

```dataviewjs
function extractSecsForSubject(fileContent, subjectHeader) {
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
    } 
    return Math.round(totalMs / 1000);
  } catch (e) { return 0; }
}

function formatTime(totalSecs) {
  if (!totalSecs) return "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  
  let str = "";
  if (h > 0) str += `${h}h `;
  if (m > 0 || h > 0) str += `${m}m `;
  str += `${s}s`;
  
  return `<span style='font-size: 0.8em; white-space: nowrap;'>${str.trim()}</span>`;
}

const recentPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'desc')
  .limit(3);

const recentTableData = [];

for (let p of recentPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = extractSecsForSubject(fileContent, "**Math**");
  const ph = extractSecsForSubject(fileContent, "**Physics**");
  const ch = extractSecsForSubject(fileContent, "**Chemistry**");
  const total = m + ph + ch;
  
  const dayNum = p.file.name.slice(-2);
  const dayLink = `<span style="font-size: 0.9em; font-weight: bold;">[[${p.file.path}|${dayNum}]]</span>`;
  
  recentTableData.push([
    dayLink, 
    formatTime(m), 
    formatTime(ph), 
    formatTime(ch), 
    `**${formatTime(total)}**`
  ]);
}

dv.table(["Day", "Math", "Phy", "Chem", "Tot"], recentTableData);
```

---

## All Past Sessions

```dataviewjs
function extractSecsForSubject(fileContent, subjectHeader) {
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
    } 
    return Math.round(totalMs / 1000);
  } catch (e) { return 0; }
}

function formatTime(totalSecs) {
  if (!totalSecs) return "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  
  let str = "";
  if (h > 0) str += `${h}h `;
  if (m > 0 || h > 0) str += `${m}m `;
  str += `${s}s`;
  
  return `<span style='font-size: 0.8em; white-space: nowrap;'>${str.trim()}</span>`;
}

const allPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'desc');

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
        const m = extractSecsForSubject(content, "**Math**");
        const ph = extractSecsForSubject(content, "**Physics**");
        const ch = extractSecsForSubject(content, "**Chemistry**");
        const total = m + ph + ch;
        
        const dayNum = p.file.name.slice(-2);
        const dayLink = `<span style="font-size: 0.9em; font-weight: bold;">[[${p.file.path}|${dayNum}]]</span>`;
        
        tableData.push([
            dayLink, 
            formatTime(m), 
            formatTime(ph), 
            formatTime(ch), 
            `**${formatTime(total)}**`
        ]);
    }
    
    const mdTable = dv.markdownTable(["Day", "Math", "Phy", "Chem", "Tot"], tableData);
    const callout = `> [!info]- 📅 ${month}\n${mdTable.split('\n').map(line => '> ' + line).join('\n')}`;
    dv.paragraph(callout);
}
```

---

## Math Activity

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const mathData = [];
const mathPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of mathPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = extractMinsForSubject(fileContent, "**Math**");
  if (mins > 0) mathData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Math Study (Minutes)",
  data: mathData,
  cellStyle: { minWidth: "14px", minHeight: "14px" },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 60, color: "#7c2d12" },       
    { min: 61, max: 120, color: "#c2410c" },      
    { min: 121, max: 180, color: "#ea580c" },     
    { min: 181, max: 999999, color: "#f97316" }  
  ],
  onCellClick: (item) => {
    if (item.value) {
      const h = Math.floor(item.value / 60);
      const m = item.value % 60;
      let timeStr = "";
      if (h > 0) timeStr += `${h}h `;
      if (m > 0 || h === 0) timeStr += `${m}min`;
      new Notice(`${timeStr.trim()} studied on ${item.date}`);
    } else {
      new Notice(`0min studied on ${item.date}`);
    }
  }
});
```

---

## Physics Activity

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const phyData = [];
const phyPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of phyPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = extractMinsForSubject(fileContent, "**Physics**");
  if (mins > 0) phyData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Physics Study (Minutes)",
  data: phyData,
  cellStyle: { minWidth: "14px", minHeight: "14px" },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 60, color: "#0c4a6e" },       
    { min: 61, max: 120, color: "#0284c7" },      
    { min: 121, max: 180, color: "#0ea5e9" },     
    { min: 181, max: 999999, color: "#38bdf8" }  
  ],
  onCellClick: (item) => {
    if (item.value) {
      const h = Math.floor(item.value / 60);
      const m = item.value % 60;
      let timeStr = "";
      if (h > 0) timeStr += `${h}h `;
      if (m > 0 || h === 0) timeStr += `${m}min`;
      new Notice(`${timeStr.trim()} studied on ${item.date}`);
    } else {
      new Notice(`0min studied on ${item.date}`);
    }
  }
});
```

---

## Chemistry Activity

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const chemData = [];
const chemPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of chemPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = extractMinsForSubject(fileContent, "**Chemistry**");
  if (mins > 0) chemData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Chemistry Study (Minutes)",
  data: chemData,
  cellStyle: { minWidth: "14px", minHeight: "14px" },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 60, color: "#a16207" },       
    { min: 61, max: 120, color: "#ca8a04" },      
    { min: 121, max: 180, color: "#eab308" },     
    { min: 181, max: 999999, color: "#fde047" }  
  ],
  onCellClick: (item) => {
    if (item.value) {
      const h = Math.floor(item.value / 60);
      const m = item.value % 60;
      let timeStr = "";
      if (h > 0) timeStr += `${h}h `;
      if (m > 0 || h === 0) timeStr += `${m}min`;
      new Notice(`${timeStr.trim()} studied on ${item.date}`);
    } else {
      new Notice(`0min studied on ${item.date}`);
    }
  }
});
```

---

## Total Study Trend (All Time)

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } 
    return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const trendPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'asc'); 

const labels = [];
const dataPoints = [];

for (let p of trendPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = extractMinsForSubject(fileContent, "**Math**");
  const ph = extractMinsForSubject(fileContent, "**Physics**");
  const ch = extractMinsForSubject(fileContent, "**Chemistry**");
  
  labels.push(p.file.name.slice(5)); 
  dataPoints.push(m + ph + ch);
}

const chartData = {
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Total Study Time (Mins)',
            data: dataPoints,
            borderColor: '#39d353',
            backgroundColor: 'rgba(57, 211, 83, 0.1)',
            borderWidth: 2,
            pointBackgroundColor: '#26a641',
            fill: true,
            tension: 0.4 
        }]
    },
    options: {
        scales: {
            y: { beginAtZero: true }
        }
    }
};

window.renderChart(chartData, this.container);
```

---

## Subject Trends (All Time)

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } 
    return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const trendPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'asc'); 

const labels = [];
const mathPts = [];
const phyPts = [];
const chemPts = [];

for (let p of trendPages) {
  const fileContent = await dv.io.load(p.file.path);
  labels.push(p.file.name.slice(5)); 
  mathPts.push(extractMinsForSubject(fileContent, "**Math**"));
  phyPts.push(extractMinsForSubject(fileContent, "**Physics**"));
  chemPts.push(extractMinsForSubject(fileContent, "**Chemistry**"));
}

dv.header(3, "Math");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Math (Mins)',
            data: mathPts,
            borderColor: '#f97316',
            backgroundColor: 'rgba(249, 115, 22, 0.1)',
            borderWidth: 2,
            pointBackgroundColor: '#ea580c',
            fill: true,
            tension: 0.4
        }]
    },
    options: { scales: { y: { beginAtZero: true } } }
}, this.container);

dv.header(3, "Physics");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Physics (Mins)',
            data: phyPts,
            borderColor: '#38bdf8',
            backgroundColor: 'rgba(56, 189, 248, 0.1)',
            borderWidth: 2,
            pointBackgroundColor: '#0ea5e9',
            fill: true,
            tension: 0.4
        }]
    },
    options: { scales: { y: { beginAtZero: true } } }
}, this.container);

dv.header(3, "Chemistry");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Chemistry (Mins)',
            data: chemPts,
            borderColor: '#fde047',
            backgroundColor: 'rgba(253, 224, 71, 0.1)',
            borderWidth: 2,
            pointBackgroundColor: '#eab308',
            fill: true,
            tension: 0.4
        }]
    },
    options: { scales: { y: { beginAtZero: true } } }
}, this.container);
```
## Subject Distribution (All Time)

```dataviewjs
function extractMinsForSubject(fileContent, subjectHeader) {
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
    } return Math.round(totalMs / 60000);
  } catch (e) { return 0; }
}

const pages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));
let mTotal = 0, pTotal = 0, cTotal = 0;

for (let p of pages) {
  const fileContent = await dv.io.load(p.file.path);
  mTotal += extractMinsForSubject(fileContent, "**Math**");
  pTotal += extractMinsForSubject(fileContent, "**Physics**");
  cTotal += extractMinsForSubject(fileContent, "**Chemistry**");
}

if (mTotal > 0 || pTotal > 0 || cTotal > 0) {
    const chartData = {
        type: 'doughnut',
        data: {
            labels: ['Math', 'Physics', 'Chemistry'],
            datasets: [{
                data: [mTotal, pTotal, cTotal],
                backgroundColor: ['#ea580c', '#0ea5e9', '#eab308'],
                hoverOffset: 4
            }]
        }
    };
    window.renderChart(chartData, this.container);
} else {
    dv.paragraph("No study data logged yet.");
}
```