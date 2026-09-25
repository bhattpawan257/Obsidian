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
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
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
    totalData.push({
      date: p.file.name,
      value: totalMins
    });
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
        if (entry.startTime && entry.endTime) {
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
        }
      }
    } 
    return Math.round(totalMs / 1000);
  } catch (e) {
    return 0;
  }
}

function formatRecentTime(totalSecs) {
  if (!totalSecs) return "00:00:00";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
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
  
  if (total > 0) {
    recentTableData.push([
      p.file.link,
      formatRecentTime(m),
      formatRecentTime(ph),
      formatRecentTime(ch),
      formatRecentTime(total)
    ]);
  }
}

dv.table(["Date", "Math", "Physics", "Chemistry", "Total"], recentTableData);
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
        if (entry.startTime && entry.endTime) {
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
        }
      }
    } 
    return Math.round(totalMs / 60000);
  } catch (e) {
    return 0;
  }
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
        if (entry.startTime && entry.endTime) {
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
        }
      }
    } 
    return Math.round(totalMs / 60000);
  } catch (e) {
    return 0;
  }
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
        if (entry.startTime && entry.endTime) {
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
        }
      }
    } 
    return Math.round(totalMs / 60000);
  } catch (e) {
    return 0;
  }
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

## DAILY SESSIONS

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
        if (entry.startTime && entry.endTime) {
          const start = new Date(entry.startTime).getTime();
          const end = new Date(entry.endTime).getTime();
          totalMs += (end - start);
        }
      }
    } 
    return Math.round(totalMs / 1000);
  } catch (e) {
    return 0;
  }
}

function formatRecentTime(totalSecs) {
  if (!totalSecs) return "00:00:00";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
}

const allRecentPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'desc');

const allTableData = [];

for (let p of allRecentPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = extractSecsForSubject(fileContent, "**Math**");
  const ph = extractSecsForSubject(fileContent, "**Physics**");
  const ch = extractSecsForSubject(fileContent, "**Chemistry**");
  const total = m + ph + ch;
  
  if (total > 0) {
    allTableData.push([
      p.file.link,
      formatRecentTime(m),
      formatRecentTime(ph),
      formatRecentTime(ch),
      formatRecentTime(total)
    ]);
  }
}

dv.table(["Date", "Math", "Physics", "Chemistry", "Total"], allTableData);
```
