# Study Dashboard

## Total Study Activity

```dataviewjs
function parseDurationToSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) {
    return Math.round(parts[0] * 3600 + parts[1] * 60 + parts[2]);
  } else if (parts.length === 2) {
    return Math.round(parts[0] * 60 + parts[1]);
  }
  return Number(val) || 0;
}

const data = [];
const pages = dv.pages("").where(p => p.math_time || p.phy_time || p.chem_time);

for (let p of pages) {
  const dateStr = p.file.name;
  const m = parseDurationToSecs(p.math_time);
  const ph = parseDurationToSecs(p.phy_time);
  const ch = parseDurationToSecs(p.chem_time);
  const totalSecs = m + ph + ch;

  if (totalSecs > 0) {
    data.push({
      date: dateStr,
      value: totalSecs
    });
  }
}

const calendarData = {
  title: "Total Study Activity (Seconds)",
  data: data,
  fromDate: "2026-09-01",
  toDate: "2027-05-31",
  cellStyleRules: [
    { min: 1, max: 1800, color: "#0e4429" },       // Up to 30 mins (1,800s)
    { min: 1801, max: 3600, color: "#006d32" },    // 30–60 mins (3,600s)
    { min: 3601, max: 7200, color: "#26a641" },    // 1–2 hours (7,200s)
    { min: 7201, max: 999999, color: "#39d353" }   // 2+ hours
  ],
  onCellClick: (item) => {
    if (item.value) {
      const h = Math.floor(item.value / 3600);
      const m = Math.floor((item.value % 3600) / 60);
      
      let timeStr = "";
      if (h > 0) timeStr += `${h}h `;
      if (m > 0 || h === 0) timeStr += `${m}min`;
      
      new Notice(`${timeStr.trim()} studied on ${item.date}`);
    } else {
      new Notice(`0min studied on ${item.date}`);
    }
  }
};

renderContributionGraph(this.container, calendarData);
```

---

## Recent Study Sessions

```dataview
TABLE 
  round(number(split(default(math_time, "0:0:0"), ":")[0]) * 60 + number(split(default(math_time, "0:0:0"), ":")[1]) + number(split(default(math_time, "0:0:0"), ":")[2]) / 60) AS "Math (m)",
  round(number(split(default(phy_time, "0:0:0"), ":")[0]) * 60 + number(split(default(phy_time, "0:0:0"), ":")[1]) + number(split(default(phy_time, "0:0:0"), ":")[2]) / 60) AS "Physics (m)",
  round(number(split(default(chem_time, "0:0:0"), ":")[0]) * 60 + number(split(default(chem_time, "0:0:0"), ":")[1]) + number(split(default(chem_time, "0:0:0"), ":")[2]) / 60) AS "Chemistry (m)",
  round(
    (number(split(default(math_time, "0:0:0"), ":")[0]) * 60 + number(split(default(math_time, "0:0:0"), ":")[1]) + number(split(default(math_time, "0:0:0"), ":")[2]) / 60) +
    (number(split(default(phy_time, "0:0:0"), ":")[0]) * 60 + number(split(default(phy_time, "0:0:0"), ":")[1]) + number(split(default(phy_time, "0:0:0"), ":")[2]) / 60) +
    (number(split(default(chem_time, "0:0:0"), ":")[0]) * 60 + number(split(default(chem_time, "0:0:0"), ":")[1]) + number(split(default(chem_time, "0:0:0"), ":")[2]) / 60)
  ) AS "Total (m)"
FROM ""
WHERE math_time OR phy_time OR chem_time
SORT file.name DESC
LIMIT 14
```

---

## Math Activity

```dataviewjs
function parseDurationToSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 3600 + parts[1] * 60 + parts[2]);
  if (parts.length === 2) return Math.round(parts[0] * 60 + parts[1]);
  return Number(val) || 0;
}

const data = [];
const pages = dv.pages("").where(p => p.math_time);

for (let p of pages) {
  const secs = parseDurationToSecs(p.math_time);
  if (secs > 0) data.push({ date: p.file.name, value: secs });
}

renderContributionGraph(this.container, {
  title: "Math Study (Seconds)",
  data: data,
  fromDate: "2026-09-01",
  toDate: "2027-05-31",
  cellStyleRules: [
    { min: 1, max: 1800, color: "#0e4429" },
    { min: 1801, max: 3600, color: "#006d32" },
    { min: 3601, max: 7200, color: "#26a641" },
    { min: 7201, max: 999999, color: "#39d353" }
  ]
});
```

---

## Physics Activity

```dataviewjs
function parseDurationToSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 3600 + parts[1] * 60 + parts[2]);
  if (parts.length === 2) return Math.round(parts[0] * 60 + parts[1]);
  return Number(val) || 0;
}

const data = [];
const pages = dv.pages("").where(p => p.phy_time);

for (let p of pages) {
  const secs = parseDurationToSecs(p.phy_time);
  if (secs > 0) data.push({ date: p.file.name, value: secs });
}

renderContributionGraph(this.container, {
  title: "Physics Study (Seconds)",
  data: data,
  fromDate: "2026-09-01",
  toDate: "2027-05-31",
  cellStyleRules: [
    { min: 1, max: 1800, color: "#0e4429" },
    { min: 1801, max: 3600, color: "#006d32" },
    { min: 3601, max: 7200, color: "#26a641" },
    { min: 7201, max: 999999, color: "#39d353" }
  ]
});
```

---

## Chemistry Activity

```dataviewjs
function parseDurationToSecs(val) {
  if (!val) return 0;
  if (typeof val === "number") return val;
  const parts = String(val).replace(/["']/g, "").trim().split(":").map(Number);
  if (parts.length === 3) return Math.round(parts[0] * 3600 + parts[1] * 60 + parts[2]);
  if (parts.length === 2) return Math.round(parts[0] * 60 + parts[1]);
  return Number(val) || 0;
}

const data = [];
const pages = dv.pages("").where(p => p.chem_time);

for (let p of pages) {
  const secs = parseDurationToSecs(p.chem_time);
  if (secs > 0) data.push({ date: p.file.name, value: secs });
}

renderContributionGraph(this.container, {
  title: "Chemistry Study (Seconds)",
  data: data,
  fromDate: "2026-09-01",
  toDate: "2027-05-31",
  cellStyleRules: [
    { min: 1, max: 1800, color: "#0e4429" },
    { min: 1801, max: 3600, color: "#006d32" },
    { min: 3601, max: 7200, color: "#26a641" },
    { min: 7201, max: 999999, color: "#39d353" }
  ]
});
```
