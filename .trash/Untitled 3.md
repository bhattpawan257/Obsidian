// Function to find the block based on the bold Markdown header above it
function extractMinsForSubject(fileContent, subjectHeader) {
  // Create a search pattern looking for the header (like **Math**) and the block right below it
  const safeHeader = subjectHeader.replace(/\*/g, "\\*");
  const regex = new RegExp(safeHeader + "[\\s\\S]*?```simple-time-tracker\\s*(\\{[\\s\\S]*?\\})\\s*```", "i");
  const match = fileContent.match(regex);
  
  if (!match) return 0;
  
  try {
    const data = JSON.parse(match[1]);
    let totalMs = 0;
    
    // Calculate time using the "entries" format (startTime and endTime)
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
// Target files that look like YYYY-MM-DD
const totalPages = dv.pages('"Daily Notes"').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/));

for (let p of totalPages) {
  const fileContent = await dv.io.load(p.file.path);
  
  // Search for the trackers based on your exact Markdown headers
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
