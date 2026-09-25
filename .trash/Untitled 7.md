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

// Shrinks font, prevents line breaks, and formats time efficiently
function formatTime(totalSecs) {
  if (!totalSecs) return "<span style='font-size: 0.8em; color: var(--text-muted);'>-</span>";
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  
  let str = "";
  if (h > 0) str += `${h}h `;
  if (m > 0 || h > 0) str += `${m}m `;
  str += `${s}s`;
  
  return `<span style='font-size: 1em; white-space: nowrap;'>${str.trim()}</span>`;
}

const allPages = dv.pages('"Daily Notes"')
  .where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/))
  .sort(p => p.file.name, 'desc');

// Automatically group pages by Month and Year
const groupedByMonth = {};
for (let p of allPages) {
    const monthName = window.moment(p.file.name).format("MMMM YYYY");
    if (!groupedByMonth[monthName]) groupedByMonth[monthName] = [];
    groupedByMonth[monthName].push(p);
}

// Generate a collapsible callout and a table for each month
for (let month in groupedByMonth) {
    const tableData = [];
    
    for (let p of groupedByMonth[month]) {
        const content = await dv.io.load(p.file.path);
        const m = extractSecsForSubject(content, "**Math**");
        const ph = extractSecsForSubject(content, "**Physics**");
        const ch = extractSecsForSubject(content, "**Chemistry**");
        const total = m + ph + ch;
        
        // Slices off the YYYY-MM to leave just the day number, keeping the clickable link intact
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
    
    // Create the markdown table string
    const mdTable = dv.markdownTable(["Day", "Math", "Phy", "Chem", "Tot"], tableData);
    
    // Wrap the table inside a native Obsidian collapsible callout targeting the month
    const callout = `> [!info]- 📅 ${month}\n${mdTable.split('\n').map(line => '> ' + line).join('\n')}`;
    
    dv.paragraph(callout);
}
