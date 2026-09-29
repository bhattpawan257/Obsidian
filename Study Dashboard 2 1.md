```dataviewjs
// 1. Daily Note Quick-Action Button
const btn = this.container.createEl('button', { text: "📝 Open Today's Note", cls: "mod-cta" });
btn.style.width = "100%";
btn.style.padding = "12px";
btn.style.fontSize = "1.2em";
btn.style.fontWeight = "bold";
btn.style.marginBottom = "15px";
btn.onclick = () => { app.commands.executeCommandById("daily-notes"); };

// 2. Master Helper Functions (Centralized)
// These functions load once here and power the entire rest of the dashboard
window.studyHelpers = {
    extractMins: function(fileContent, subjectHeader) {
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
    },
    
    extractSecs: function(fileContent, subjectHeader) {
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
    },
    
    formatTime: function(totalSecs) {
        if (!totalSecs) return "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
        const h = Math.floor(totalSecs / 3600);
        const m = Math.floor((totalSecs % 3600) / 60);
        const s = totalSecs % 60;
        let str = "";
        if (h > 0) str += `${h}h `;
        if (m > 0 || h > 0) str += `${m}m `;
        str += `${s}s`;
        return `<span style='font-size: 0.8em; white-space: nowrap;'>${str.trim()}</span>`;
    },

    extractTopics: function(fileContent, subjectHeader) {
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
};
```

```dataviewjs
const pages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));
let mTotal = 0, pTotal = 0, cTotal = 0, days = 0;
let activeDates = new Set();

for (let p of pages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = window.studyHelpers.extractMins(fileContent, "**Math**");
  const ph = window.studyHelpers.extractMins(fileContent, "**Physics**");
  const c = window.studyHelpers.extractMins(fileContent, "**Chemistry**");
  
  mTotal += m; pTotal += ph; cTotal += c;
  
  if (m + ph + c > 0) {
      activeDates.add(p.file.name);
      days++;
  }
}

const totalMins = mTotal + pTotal + cTotal;
const totalHours = (totalMins / 60).toFixed(1);
const avgMins = days > 0 ? Math.round(totalMins / days) : 0;

let topSubject = "None";
if (mTotal >= pTotal && mTotal >= cTotal && mTotal > 0) topSubject = "📐 Math";
else if (pTotal >= mTotal && pTotal >= cTotal && pTotal > 0) topSubject = "🍎 Physics";
else if (cTotal >= mTotal && cTotal >= pTotal && cTotal > 0) topSubject = "🧪 Chemistry";

// Automated Streak Counter
let streak = 0;
const today = window.moment().format("YYYY-MM-DD");
const yesterday = window.moment().subtract(1, 'days').format("YYYY-MM-DD");
let checkDate = today;

if (!activeDates.has(today)) checkDate = yesterday;

while(activeDates.has(checkDate)) {
    streak++;
    checkDate = window.moment(checkDate).subtract(1, 'days').format("YYYY-MM-DD");
}

dv.paragraph(`> [!abstract] 📊 Quick Stats\n> **Total Time:** ${totalHours} hours\n> **Daily Average:** ${avgMins} mins/day\n> **Top Subject:** ${topSubject}\n> 🔥 **Current Streak:** ${streak} Days`);
```

---

## Total Study Activity

```dataviewjs
const totalData = [];
const totalPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of totalPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = window.studyHelpers.extractMins(fileContent, "**Math**");
  const ph = window.studyHelpers.extractMins(fileContent, "**Physics**");
  const ch = window.studyHelpers.extractMins(fileContent, "**Chemistry**");
  const totalMins = m + ph + ch;

  if (totalMins > 0) {
    totalData.push({ date: p.file.name, value: totalMins });
  }
}

renderContributionGraph(this.container, {
  title: "Total Study Activity (Minutes)",
  data: totalData,
  cellStyle: { minWidth: "25px", minHeight: "25px" },
  showAllDayLabel: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
    cellStyleRules: [
    { min: 1, max: 60, color: "#022c22" },       
    { min: 60, max: 120, color: "#064e3b" },     
    { min: 120, max: 180, color: "#047857" },    
    { min: 180, max: 240, color: "#059669" },    
    { min: 240, max: 300, color: "#10b981" },    
    { min: 300, max: 360, color: "#34d399" },    
    { min: 360, max: 420, color: "#00e676" },    
    { min: 420, max: 480, color: "#14ff86" },    
    { min: 480, max: 999999, color: "#42ff9f" }  
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

## Weekly Goal: 15 Hours

```dataviewjs
const sevenDaysAgo = window.moment().subtract(7, 'days').format("YYYY-MM-DD");
const weekPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/) && p.file.name >= sevenDaysAgo);

let weekTotalMins = 0;
for (let p of weekPages) {
  const content = await dv.io.load(p.file.path);
  weekTotalMins += window.studyHelpers.extractMins(content, "**Math**");
  weekTotalMins += window.studyHelpers.extractMins(content, "**Physics**");
  weekTotalMins += window.studyHelpers.extractMins(content, "**Chemistry**");
}

const goalMins = 900; 
const percentage = Math.min(Math.round((weekTotalMins / goalMins) * 100), 100);

dv.paragraph(`<div style="width: 100%; background-color: var(--background-modifier-border); border-radius: 8px; overflow: hidden; height: 20px;">
  <div style="width: ${percentage}%; background-color: #39d353; height: 100%; text-align: center; color: black; font-size: 12px; font-weight: bold; line-height: 20px;">
    ${percentage}% (${Math.round(weekTotalMins / 60)}h / 15h)
  </div>
</div>`);
```

---

## Topics Covered (Last 7 Days)

```dataviewjs
const sevenDaysAgo = window.moment().subtract(7, 'days').format("YYYY-MM-DD");
const weekPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/) && p.file.name >= sevenDaysAgo);

let mathTopics = new Set(), phyTopics = new Set(), chemTopics = new Set();

for (let p of weekPages) {
  const content = await dv.io.load(p.file.path);
  window.studyHelpers.extractTopics(content, "**Math**").forEach(t => mathTopics.add(t));
  window.studyHelpers.extractTopics(content, "**Physics**").forEach(t => phyTopics.add(t));
  window.studyHelpers.extractTopics(content, "**Chemistry**").forEach(t => chemTopics.add(t));
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
const recentPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc').limit(3);
const recentTableData = [];

for (let p of recentPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = window.studyHelpers.extractSecs(fileContent, "**Math**");
  const ph = window.studyHelpers.extractSecs(fileContent, "**Physics**");
  const ch = window.studyHelpers.extractSecs(fileContent, "**Chemistry**");
  const total = m + ph + ch;
  
  const dayNum = p.file.name.slice(-2);
  const dayLink = `<span style="font-size: 0.9em; font-weight: bold;">[[${p.file.path}|${dayNum}]]</span>`;
  
  recentTableData.push([
    dayLink, 
    window.studyHelpers.formatTime(m), 
    window.studyHelpers.formatTime(ph), 
    window.studyHelpers.formatTime(ch), 
    `**${window.studyHelpers.formatTime(total)}**`
  ]);
}

dv.table(["Day", "Math", "Phy", "Chem", "Tot"], recentTableData);
```

---

## All Past Sessions

```dataviewjs
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
        const m = window.studyHelpers.extractSecs(content, "**Math**");
        const ph = window.studyHelpers.extractSecs(content, "**Physics**");
        const ch = window.studyHelpers.extractSecs(content, "**Chemistry**");
        const total = m + ph + ch;
        
        const dayNum = p.file.name.slice(-2);
        const dayLink = `<span style="font-size: 0.9em; font-weight: bold;">[[${p.file.path}|${dayNum}]]</span>`;
        
        tableData.push([
            dayLink, 
            window.studyHelpers.formatTime(m), 
            window.studyHelpers.formatTime(ph), 
            window.studyHelpers.formatTime(ch), 
            `**${window.studyHelpers.formatTime(total)}**`
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
const mathData = [];
const mathPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of mathPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = window.studyHelpers.extractMins(fileContent, "**Math**");
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
    { min: 1, max: 29, color: "hsl(18, 100%, 12%)" },    
    { min: 30, max: 59, color: "hsl(18, 100%, 18%)" },   
    { min: 60, max: 89, color: "hsl(18, 100%, 25%)" },   
    { min: 90, max: 119, color: "hsl(18, 100%, 31%)" },  
    { min: 120, max: 149, color: "hsl(18, 100%, 38%)" }, 
    { min: 150, max: 179, color: "hsl(18, 100%, 44%)" }, 
    { min: 180, max: 999999, color: "hsl(18, 100%, 50%)" } 
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
const phyData = [];
const phyPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of phyPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = window.studyHelpers.extractMins(fileContent, "**Physics**");
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
    { min: 1, max: 29, color: "hsl(203, 100%, 12%)" },    
    { min: 30, max: 59, color: "hsl(203, 100%, 18%)" },   
    { min: 60, max: 89, color: "hsl(203, 100%, 25%)" },   
    { min: 90, max: 119, color: "hsl(203, 100%, 31%)" },  
    { min: 120, max: 149, color: "hsl(203, 100%, 38%)" }, 
    { min: 150, max: 179, color: "hsl(203, 100%, 44%)" }, 
    { min: 180, max: 999999, color: "hsl(203, 100%, 50%)" } 
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
const chemData = [];
const chemPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of chemPages) {
  const fileContent = await dv.io.load(p.file.path);
  const mins = window.studyHelpers.extractMins(fileContent, "**Chemistry**");
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
    { min: 1, max: 30, color: "#291502" },       
    { min: 30, max: 60, color: "#593003" },      
    { min: 60, max: 90, color: "#8a4d04" },      
    { min: 90, max: 120, color: "#ca8a04" },     
    { min: 120, max: 150, color: "#facc15" },    
    { min: 150, max: 180, color: "#ffe100" },    
    { min: 180, max: 999999, color: "#f7ff00" }  
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
const trendPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'asc'); 
const labels = [];
const dataPoints = [];

for (let p of trendPages) {
  const fileContent = await dv.io.load(p.file.path);
  const m = window.studyHelpers.extractMins(fileContent, "**Math**");
  const ph = window.studyHelpers.extractMins(fileContent, "**Physics**");
  const ch = window.studyHelpers.extractMins(fileContent, "**Chemistry**");
  
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
    options: { scales: { y: { beginAtZero: true } } }
};

window.renderChart(chartData, this.container);
```

---

## Subject Trends (All Time)

```dataviewjs
const trendPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'asc'); 
const labels = [];
const mathPts = [], phyPts = [], chemPts = [];

for (let p of trendPages) {
  const fileContent = await dv.io.load(p.file.path);
  labels.push(p.file.name.slice(5)); 
  mathPts.push(window.studyHelpers.extractMins(fileContent, "**Math**"));
  phyPts.push(window.studyHelpers.extractMins(fileContent, "**Physics**"));
  chemPts.push(window.studyHelpers.extractMins(fileContent, "**Chemistry**"));
}

dv.header(3, "Math");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Math (Mins)', data: mathPts, borderColor: '#f97316',
            backgroundColor: 'rgba(249, 115, 22, 0.1)', borderWidth: 2, pointBackgroundColor: '#ea580c', fill: true, tension: 0.4
        }]
    }, options: { scales: { y: { beginAtZero: true } } }
}, this.container);

dv.header(3, "Physics");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Physics (Mins)', data: phyPts, borderColor: '#38bdf8',
            backgroundColor: 'rgba(56, 189, 248, 0.1)', borderWidth: 2, pointBackgroundColor: '#0ea5e9', fill: true, tension: 0.4
        }]
    }, options: { scales: { y: { beginAtZero: true } } }
}, this.container);

dv.header(3, "Chemistry");
window.renderChart({
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Chemistry (Mins)', data: chemPts, borderColor: '#fde047',
            backgroundColor: 'rgba(253, 224, 71, 0.1)', borderWidth: 2, pointBackgroundColor: '#eab308', fill: true, tension: 0.4
        }]
    }, options: { scales: { y: { beginAtZero: true } } }
}, this.container);
```

---

## Subject Distribution (All Time)

```dataviewjs
const pages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));
let mTotal = 0, pTotal = 0, cTotal = 0;

for (let p of pages) {
  const fileContent = await dv.io.load(p.file.path);
  mTotal += window.studyHelpers.extractMins(fileContent, "**Math**");
  pTotal += window.studyHelpers.extractMins(fileContent, "**Physics**");
  cTotal += window.studyHelpers.extractMins(fileContent, "**Chemistry**");
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
