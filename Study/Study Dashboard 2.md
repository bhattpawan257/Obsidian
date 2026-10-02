```dataviewjs
// 1. Auto-Sync Master JSON on load
(async () => {
    try {
        const pages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'asc');
        const masterData = [];

        function extractRawEntries(fileContent, subjectHeader) {
            const safeHeader = subjectHeader.replace(/\*/g, "\\*");
            const sectionRegex = new RegExp(safeHeader + "([\\s\\S]*?)(?=\\n#|\\n\\*\\*|$)", "gi");
            let combinedEntries = [];
            let sectionMatch;
            
            while ((sectionMatch = sectionRegex.exec(fileContent)) !== null) {
                const trackerRegex = /```simple-time-tracker\s*(\{[\s\S]*?\})\s*```/g;
                let match;
                while ((match = trackerRegex.exec(sectionMatch[1])) !== null) {
                    try {
                        const data = JSON.parse(match[1]);
                        if (data.entries) combinedEntries.push(...data.entries);
                    } catch (e) {}
                }
            }
            return combinedEntries;
        }

        for (let p of pages) {
            const fileContent = await dv.io.load(p.file.path);
            const mathEntries = extractRawEntries(fileContent, "**Math**");
            const phyEntries = extractRawEntries(fileContent, "**Physics**");
            const chemEntries = extractRawEntries(fileContent, "**Chemistry**");
            
            if (mathEntries.length > 0 || phyEntries.length > 0 || chemEntries.length > 0) {
                masterData.push({
                    date: p.file.name,
                    math: mathEntries,
                    physics: phyEntries,
                    chemistry: chemEntries
                });
            }
        }

        const jsonString = JSON.stringify(masterData, null, 2);
        const filePath = "study-data.json";
        
        const fileExists = await app.vault.adapter.exists(filePath);
        let existingContent = "";
        if (fileExists) {
            existingContent = await app.vault.adapter.read(filePath);
        }

        if (jsonString !== existingContent) {
            await app.vault.adapter.write(filePath, jsonString);
            new Notice("🔄 Master JSON automatically updated!");
        }
    } catch (err) {
        console.error("Error auto-updating JSON:", err);
    }
})();

// 2. Master Helper Functions (Centralized & Robust)
window.studyHelpers = {
    extractMins: function(fileContent, subjectHeader) {
        const safeHeader = subjectHeader.replace(/\*/g, "\\*");
        const sectionRegex = new RegExp(safeHeader + "([\\s\\S]*?)(?=\\n#|\\n\\*\\*|$)", "gi");
        let totalMs = 0;
        let sectionMatch;
        
        while ((sectionMatch = sectionRegex.exec(fileContent)) !== null) {
            const trackerRegex = /```simple-time-tracker\s*(\{[\s\S]*?\})\s*```/g;
            let match;
            while ((match = trackerRegex.exec(sectionMatch[1])) !== null) {
                try {
                    const data = JSON.parse(match[1]);
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
                } catch (e) {}
            }
        }
        return Math.round(totalMs / 60000);
    },
    
    extractSecs: function(fileContent, subjectHeader) {
        const safeHeader = subjectHeader.replace(/\*/g, "\\*");
        const sectionRegex = new RegExp(safeHeader + "([\\s\\S]*?)(?=\\n#|\\n\\*\\*|$)", "gi");
        let totalMs = 0;
        let sectionMatch;
        
        while ((sectionMatch = sectionRegex.exec(fileContent)) !== null) {
            const trackerRegex = /```simple-time-tracker\s*(\{[\s\S]*?\})\s*```/g;
            let match;
            while ((match = trackerRegex.exec(sectionMatch[1])) !== null) {
                try {
                    const data = JSON.parse(match[1]);
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
                } catch (e) {}
            }
        }
        return Math.round(totalMs / 1000);
    },
    
    formatTime: function(totalSecs, isBold = false) {
        const sortKey = String(totalSecs).padStart(7, '0');
        if (!totalSecs) {
            const dash = "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
            return `<!--${sortKey}--> ${isBold ? `<span style="font-weight:bold;">${dash}</span>` : dash}`;
        }
        const h = Math.floor(totalSecs / 3600);
        const m = Math.floor((totalSecs % 3600) / 60);
        const s = totalSecs % 60;
        let str = "";
        if (h > 0) str += `${h}h `;
        if (m > 0 || h > 0) str += `${m}m `;
        str += `${s}s`;
        const span = `<span style='font-size: 0.8em; white-space: nowrap; ${isBold ? "font-weight:bold;" : ""}'>${str.trim()}</span>`;
        return `<!--${sortKey}--> ${span}`;
    },

    extractTopics: function(fileContent, subjectHeader) {
        const safeHeader = subjectHeader.replace(/\*/g, "\\*");
        const sectionRegex = new RegExp(safeHeader + "([\\s\\S]*?)(?=\\n#|\\n\\*\\*|$)", "gi");
        let topics = new Set();
        let sectionMatch;
        
        while ((sectionMatch = sectionRegex.exec(fileContent)) !== null) {
            const trackerRegex = /```simple-time-tracker\s*(\{[\s\S]*?\})\s*```/g;
            let match;
            while ((match = trackerRegex.exec(sectionMatch[1])) !== null) {
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
            }
        }
        return Array.from(topics);
    },

    extractDetailedTopics: function(fileContent, subjectHeader, dateStr) {
        const safeHeader = subjectHeader.replace(/\*/g, "\\*");
        const sectionRegex = new RegExp(safeHeader + "([\\s\\S]*?)(?=\\n#|\\n\\*\\*|$)", "gi");
        let results = [];
        let sectionMatch;
        
        while ((sectionMatch = sectionRegex.exec(fileContent)) !== null) {
            const trackerRegex = /```simple-time-tracker\s*(\{[\s\S]*?\})\s*```/g;
            let match;
            while ((match = trackerRegex.exec(sectionMatch[1])) !== null) {
                try {
                    const data = JSON.parse(match[1]);
                    if (data.entries) {
                        for (let entry of data.entries) {
                            let parentName = entry.name ? entry.name.trim() : "Unnamed Topic";
                            let parentTime = 0;
                            if (entry.startTime && entry.endTime) {
                                parentTime += (new Date(entry.endTime).getTime() - new Date(entry.startTime).getTime());
                            }
                            
                            if (entry.subEntries) {
                                for (let sub of entry.subEntries) {
                                    let subName = sub.name ? sub.name.trim() : "";
                                    let subTime = 0;
                                    if (sub.startTime && sub.endTime) {
                                        subTime += (new Date(sub.endTime).getTime() - new Date(sub.startTime).getTime());
                                    }
                                    
                                    if (subName && !subName.toLowerCase().startsWith("segment") && !subName.toLowerCase().startsWith("part")) {
                                        if (subTime > 0) results.push({ name: subName, duration: subTime, date: dateStr });
                                    } else {
                                        parentTime += subTime; 
                                    }
                                }
                            }
                            
                            if (parentName && !parentName.toLowerCase().startsWith("segment") && !parentName.toLowerCase().startsWith("part") && parentTime > 0) {
                                results.push({ name: parentName, duration: parentTime, date: dateStr });
                            }
                        }
                    }
                } catch (e) {}
            }
        }
        return results;
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
    { min: 1, max: 59, color: "hsl(145, 100%, 12%)" },       
    { min: 60, max: 119, color: "hsl(145, 100%, 18%)" },     
    { min: 120, max: 179, color: "hsl(145, 100%, 25%)" },    
    { min: 180, max: 239, color: "hsl(145, 100%, 31%)" },    
    { min: 240, max: 299, color: "hsl(145, 100%, 38%)" },    
    { min: 300, max: 359, color: "hsl(145, 100%, 44%)" },    
    { min: 360, max: 419, color: "hsl(145, 100%, 50%)" },    
    { min: 420, max: 479, color: "hsl(145, 100%, 56%)" },    
    { min: 480, max: 999999, color: "hsl(145, 100%, 62%)" }  
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

dv.paragraph(`<div style="width: 100%; background-color: var(--background-modifier-border); border-radius: 8px; overflow: hidden; height: 20px; margin-bottom: 25px;">
  <div style="width: ${percentage}%; background-color: #39d353; height: 100%; text-align: center; color: black; font-size: 12px; font-weight: bold; line-height: 20px;">
    ${percentage}% (${Math.round(weekTotalMins / 60)}h / 15h)
  </div>
</div>`);

// Button positioned beneath the goal bar
const btn = this.container.createEl('button', { text: "📝 Open Today's Note", cls: "mod-cta" });
btn.style.width = "100%";
btn.style.padding = "12px";
btn.style.fontSize = "1.2em";
btn.style.fontWeight = "bold";
btn.onclick = () => { app.commands.executeCommandById("daily-notes"); };
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
    window.studyHelpers.formatTime(total, true)
  ]);
}

dv.table(["Day", "Math", "Phy", "Chem", "Tot"], recentTableData);
```

---

## All Past Sessions

```dataviewjs
const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc');
const masterTableData = [];
const groupedByMonth = {};

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    const m = window.studyHelpers.extractSecs(content, "**Math**");
    const ph = window.studyHelpers.extractSecs(content, "**Physics**");
    const ch = window.studyHelpers.extractSecs(content, "**Chemistry**");
    const total = m + ph + ch;
    
    const fullDateLink = `<!--${p.file.name}--> [[${p.file.path}|${p.file.name}]]`;
    const dayNum = p.file.name.slice(-2);
    const dayLink = `<!--${dayNum}--> [[${p.file.path}|${dayNum}]]`;
    
    const mStr = window.studyHelpers.formatTime(m);
    const phStr = window.studyHelpers.formatTime(ph);
    const chStr = window.studyHelpers.formatTime(ch);
    const totStr = window.studyHelpers.formatTime(total, true);

    masterTableData.push([fullDateLink, mStr, phStr, chStr, totStr]);

    const monthName = window.moment(p.file.name).format("MMMM YYYY");
    if (!groupedByMonth[monthName]) groupedByMonth[monthName] = [];
    groupedByMonth[monthName].push([dayLink, mStr, phStr, chStr, totStr]);
}

const mdMasterTable = dv.markdownTable(["Date", "Math", "Phy", "Chem", "Tot"], masterTableData);
const masterCallout = `> [!info]- 📚 All-Time Master Log\n${mdMasterTable.split('\n').map(line => '> ' + line).join('\n')}`;
dv.paragraph(masterCallout);

for (let month in groupedByMonth) {
    const mdTable = dv.markdownTable(["Day", "Math", "Phy", "Chem", "Tot"], groupedByMonth[month]);
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
    { min: 1, max: 29, color: "hsl(48, 100%, 12%)" },       
    { min: 30, max: 59, color: "hsl(48, 100%, 18%)" },      
    { min: 60, max: 89, color: "hsl(48, 100%, 25%)" },      
    { min: 90, max: 119, color: "hsl(48, 100%, 31%)" },     
    { min: 120, max: 149, color: "hsl(48, 100%, 38%)" },    
    { min: 150, max: 179, color: "hsl(48, 100%, 44%)" },    
    { min: 180, max: 999999, color: "hsl(48, 100%, 50%)" }  
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

const chartData = {
    type: 'line',
    data: {
        labels: labels,
        datasets: [
            {
                label: 'Math (Mins)', 
                data: mathPts, 
                borderColor: '#f97316',
                backgroundColor: 'rgba(249, 115, 22, 0.1)', 
                borderWidth: 2, 
                pointBackgroundColor: '#ea580c', 
                fill: true, 
                tension: 0.4
            },
            {
                label: 'Physics (Mins)', 
                data: phyPts, 
                borderColor: '#38bdf8',
                backgroundColor: 'rgba(56, 189, 248, 0.1)', 
                borderWidth: 2, 
                pointBackgroundColor: '#0ea5e9', 
                fill: true, 
                tension: 0.4
            },
            {
                label: 'Chemistry (Mins)', 
                data: chemPts, 
                borderColor: '#fde047',
                backgroundColor: 'rgba(253, 224, 71, 0.1)', 
                borderWidth: 2, 
                pointBackgroundColor: '#eab308', 
                fill: true, 
                tension: 0.4
            }
        ]
    },
    options: { 
        scales: { y: { beginAtZero: true } },
        interaction: { mode: 'index', intersect: false }
    }
};

dv.header(3, "Combined Subject Trends");
window.renderChart(chartData, this.container);
```


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

---

## Detailed Topic Breakdown (All Time)

```dataviewjs
const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));
let subjectTopics = { "📐 Math": {}, "🍎 Physics": {}, "🧪 Chemistry": {} };

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    const mathT = window.studyHelpers.extractDetailedTopics(content, "**Math**", p.file.name);
    const phyT = window.studyHelpers.extractDetailedTopics(content, "**Physics**", p.file.name);
    const chemT = window.studyHelpers.extractDetailedTopics(content, "**Chemistry**", p.file.name);

    const addTopics = (topicList, subKey) => {
        for (let t of topicList) {
            if (!subjectTopics[subKey][t.name]) subjectTopics[subKey][t.name] = [];
            subjectTopics[subKey][t.name].push(t);
        }
    };
    
    addTopics(mathT, "📐 Math");
    addTopics(phyT, "🍎 Physics");
    addTopics(chemT, "🧪 Chemistry");
}

for (let sub in subjectTopics) {
    let tableData = [];
    for (let topicName in subjectTopics[sub]) {
        let sessions = subjectTopics[sub][topicName];
        
        // Group sessions that happen on the same exact day
        let dateMap = {};
        sessions.forEach(s => {
            if (!dateMap[s.date]) dateMap[s.date] = 0;
            dateMap[s.date] += s.duration;
        });

        let sortedDates = Object.keys(dateMap).sort((a,b) => b.localeCompare(a));
        let totalMs = 0;
        let historyHtml = `<details><summary style="cursor:pointer; color:var(--text-accent); font-weight:bold;">View ${sortedDates.length} Day(s)</summary><div style="margin-top:5px; padding-left:10px; border-left:2px solid var(--background-modifier-border);">`;
        
        for (let d of sortedDates) {
            let dTime = dateMap[d];
            totalMs += dTime;
            historyHtml += `<span style="font-size:0.85em;">${d}: ${window.studyHelpers.formatTime(Math.round(dTime/1000))}</span><br>`;
        }
        historyHtml += `</div></details>`;
        
        let totalSecs = Math.round(totalMs / 1000);
        let timeStr = window.studyHelpers.formatTime(totalSecs, true);

        tableData.push([`<span style="font-weight:bold;">${topicName}</span>`, timeStr, historyHtml]);
    }
    
    if (tableData.length > 0) {
        dv.header(3, sub);
        dv.table(["Topic", "Total Time", "Session History"], tableData);
    }
}
```
---
```dataviewjs

const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

let mathData = {}, phyData = {}, chemData = {};

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    
    window.studyHelpers.extractDetailedTopics(content, "**Math**", p.file.name).forEach(t => {
        if (!mathData[t.name]) mathData[t.name] = 0;
        mathData[t.name] += t.duration;
    });
    
    window.studyHelpers.extractDetailedTopics(content, "**Physics**", p.file.name).forEach(t => {
        if (!phyData[t.name]) phyData[t.name] = 0;
        phyData[t.name] += t.duration;
    });
    
    window.studyHelpers.extractDetailedTopics(content, "**Chemistry**", p.file.name).forEach(t => {
        if (!chemData[t.name]) chemData[t.name] = 0;
        chemData[t.name] += t.duration;
    });
}

// Added 'container' parameter to safely pass the rendering context
function createPieChart(title, dataObj, palette, container) {
    const sortedTopics = Object.keys(dataObj).sort((a, b) => dataObj[b] - dataObj[a]);
    if (sortedTopics.length === 0) return;
    
    const labels = sortedTopics;
    const dataVals = sortedTopics.map(t => Math.round(dataObj[t] / 60000)); 
    const bgColors = labels.map((_, i) => palette[i % palette.length]);
    
    const chartData = {
        type: 'pie',
        data: {
            labels: labels,
            datasets: [{
                data: dataVals,
                backgroundColor: bgColors,
                borderWidth: 2,
                borderColor: 'var(--background-primary)'
            }]
        },
        options: {
            plugins: {
                legend: { 
                    position: 'right', 
                    labels: { color: 'var(--text-normal)' } 
                },
                tooltip: {
                    callbacks: {
                        label: function(context) {
                            let mins = context.raw;
                            let h = Math.floor(mins / 60);
                            let m = mins % 60;
                            let timeStr = (h > 0 ? h + "h " : "") + m + "m";
                            return " " + context.label + ": " + timeStr;
                        }
                    }
                }
            }
        }
    };
    
    dv.header(3, title);
    window.renderChart(chartData, container);
}

const mathColors = ['#ffb700', '#ff9500', '#f97316', '#d44304', '#9a2b04', '#5c1904', '#2a0a02'];
const phyColors = ['#00eaff', '#00ccff', '#00a6ff', '#0284c7', '#065996', '#05375c', '#021a2e'];
const chemColors = ['#f7ff00', '#ffe100', '#facc15', '#ca8a04', '#8a4d04', '#593003', '#291502'];

// Passed 'this.container' into the function calls
createPieChart("📐 Math Topics", mathData, mathColors, this.container);
createPieChart("🍎 Physics Topics", phyData, phyColors, this.container);
createPieChart("🧪 Chemistry Topics", chemData, chemColors, this.container);

```