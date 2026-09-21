

## Total Study Activity

```dataviewjs
function parseTotalMins(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) {
    return Math.round(parts[0] * 60 + parts[1] + parts[2] / 60);
  } else if (parts.length === 2) {
    return Math.round(parts[0] + parts[1] / 60);
  }
  return Number(val) || 0;
}

const totalData = [];
const totalPages = dv.pages("").where(p => p.math_time || p.phy_time || p.chem_time);

for (let p of totalPages) {
  const dateStr = p.file.name;
  const m = parseTotalMins(p.math_time);
  const ph = parseTotalMins(p.phy_time);
  const ch = parseTotalMins(p.chem_time);
  const totalMins = m + ph + ch;

  if (totalMins > 0) {
    totalData.push({
      date: dateStr,
      value: totalMins
    });
  }
}

const totalCalendarData = {
  title: "Total Study Activity (Minutes)",
  data: totalData,
  cellStyle: {
    minWidth: "25px",
    minHeight: "25px"
  },
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
function getRecentSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return parts[0] * 3600 + parts[1] * 60 + parts[2];
  if (parts.length === 2) return parts[0] * 60 + parts[1];
  return Number(val) || 0;
}

function formatRecentTime(totalSecs) {
  if (!totalSecs) return "00:00:00";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
}

const recentPages = dv.pages("")
  .where(p => p.file.name !== "Daily note template" && (p.math_time || p.phy_time || p.chem_time))
  .sort(p => p.file.name, 'desc')
  .limit(3);

const recentTableData = recentPages.map(p => {
  const m = getRecentSecs(p.math_time);
  const ph = getRecentSecs(p.phy_time);
  const ch = getRecentSecs(p.chem_time);
  const total = m + ph + ch;
  return [
    p.file.link,
    formatRecentTime(m),
    formatRecentTime(ph),
    formatRecentTime(ch),
    formatRecentTime(total)
  ];
});

dv.table(["Date", "Math", "Physics", "Chemistry", "Total"], recentTableData);
```

---

## Math Activity

```dataviewjs
function parseMathMins(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 60 + parts[1] + parts[2] / 60);
  if (parts.length === 2) return Math.round(parts[0] + parts[1] / 60);
  return Number(val) || 0;
}

const mathData = [];
const mathPages = dv.pages("").where(p => p.math_time);

for (let p of mathPages) {
  const mins = parseMathMins(p.math_time);
  if (mins > 0) mathData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Math Study (Minutes)",
  data: mathData,
  cellStyle: {
    minWidth: "14px",
    minHeight: "14px"
  },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
cellStyleRules: [
    { min: 1, max: 30, color: "#7c2d12" },       // Deep burnt orange (up to 30 mins)
    { min: 31, max: 60, color: "#c2410c" },      // Dark orange (30–60 mins)
    { min: 61, max: 120, color: "#ea580c" },     // Bright standard orange (1–2 hours)
    { min: 121, max: 999999, color: "#f97316" }  // Vibrant light orange (2+ hours)
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
function parsePhyMins(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 60 + parts[1] + parts[2] / 60);
  if (parts.length === 2) return Math.round(parts[0] + parts[1] / 60);
  return Number(val) || 0;
}

const phyData = [];
const phyPages = dv.pages("").where(p => p.phy_time);

for (let p of phyPages) {
  const mins = parsePhyMins(p.phy_time);
  if (mins > 0) phyData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Physics Study (Minutes)",
  data: phyData,
  cellStyle: {
    minWidth: "14px",
    minHeight: "14px"
  },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 30, color: "#0c4a6e" },       // Deep navy blue (up to 30 mins)
    { min: 31, max: 60, color: "#0284c7" },      // Dark blue (30–60 mins)
    { min: 61, max: 120, color: "#0ea5e9" },     // Bright azure blue (1–2 hours)
    { min: 121, max: 999999, color: "#38bdf8" }  // Vibrant light blue (2+ hours)
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
function parseChemMins(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 60 + parts[1] + parts[2] / 60);
  if (parts.length === 2) return Math.round(parts[0] + parts[1] / 60);
  return Number(val) || 0;
}

const chemData = [];
const chemPages = dv.pages("").where(p => p.chem_time);

for (let p of chemPages) {
  const mins = parseChemMins(p.chem_time);
  if (mins > 0) chemData.push({ date: p.file.name, value: mins });
}

renderContributionGraph(this.container, {
  title: "Chemistry Study (Minutes)",
  data: chemData,
  cellStyle: {
    minWidth: "14px",
    minHeight: "14px"
  },
  showAllDays: true,
  fromDate: "2026-09-01",
  toDate: window.moment().format("YYYY-MM-DD"),
  cellStyleRules: [
    { min: 1, max: 30, color: "#a16207" },       // Deep golden yellow (up to 30 mins)
    { min: 31, max: 60, color: "#ca8a04" },      // Dark yellow (30–60 mins)
    { min: 61, max: 120, color: "#eab308" },     // Bright standard yellow (1–2 hours)
    { min: 121, max: 999999, color: "#fde047" }  // Vibrant light yellow (2+ hours)
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

## DAILY SESSIONS
```dataviewjs
function getRecentSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return parts[0] * 3600 + parts[1] * 60 + parts[2];
  if (parts.length === 2) return parts[0] * 60 + parts[1];
  return Number(val) || 0;
}

function formatRecentTime(totalSecs) {
  if (!totalSecs) return "00:00:00";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
}

const recentPages = dv.pages("")
  .where(p => p.file.name !== "Daily note template" && (p.math_time || p.phy_time || p.chem_time))
  .sort(p => p.file.name, 'desc');

const recentTableData = recentPages.map(p => {
  const m = getRecentSecs(p.math_time);
  const ph = getRecentSecs(p.phy_time);
  const ch = getRecentSecs(p.chem_time);
  const total = m + ph + ch;
  return [
    p.file.link,
    formatRecentTime(m),
    formatRecentTime(ph),
    formatRecentTime(ch),
    formatRecentTime(total)
  ];
});

dv.table(["Date", "Math", "Physics", "Chemistry", "Total"], recentTableData);
```