```dataviewjs

// Independent Helper Functions (No external scripts required)
function extractSecs(fileContent, subjectHeader) {
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
}

function formatTime(totalSecs, isBold = false) {
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
}

// Data Fetching and Table Generation
const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc');
const masterTableData = [];

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    const m = extractSecs(content, "**Math**");
    const ph = extractSecs(content, "**Physics**");
    const ch = extractSecs(content, "**Chemistry**");
    const total = m + ph + ch;
    
    // Hidden HTML comments force Dataview to sort dates and empty fields correctly
    const fullDateLink = `<!--${p.file.name}--> [[${p.file.path}|${p.file.name}]]`;
    
    const mStr = formatTime(m);
    const phStr = formatTime(ph);
    const chStr = formatTime(ch);
    const totStr = formatTime(total, true);

    masterTableData.push([fullDateLink, mStr, phStr, chStr, totStr]);
}

// Render the table directly without the callout dropdown
dv.table(["Date", "Math", "Phy", "Chem", "Tot"], masterTableData);

```
---
```dataviewjs 

// Independent Helper Functions (No external scripts required)
function extractSecs(fileContent, subjectHeader) {
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
}

function formatTime(totalSecs, isBold = false) {
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
}

// Data Fetching and Table Generation
const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc');
const masterTableData = [];

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    const m = extractSecs(content, "**Math**");
    const ph = extractSecs(content, "**Physics**");
    const ch = extractSecs(content, "**Chemistry**");
    const total = m + ph + ch;
    
    // Hidden HTML comments force Dataview to sort dates and empty fields correctly
    const fullDateLink = `<!--${p.file.name}--> [[${p.file.path}|${p.file.name}]]`;
    
    const mStr = formatTime(m);
    const phStr = formatTime(ph);
    const chStr = formatTime(ch);
    const totStr = formatTime(total, true);

    masterTableData.push([fullDateLink, mStr, phStr, chStr, totStr]);
}

// Render only the All-Time Master Table
const mdMasterTable = dv.markdownTable(["Date", "Math", "Phy", "Chem", "Tot"], masterTableData);
const masterCallout = `> [!info]- 📚 All-Time Master Log\n${mdMasterTable.split('\n').map(line => '> ' + line).join('\n')}`;
dv.paragraph(masterCallout);

```
---
```dataviewjs 

// Independent Helper Functions (No external scripts required)
function extractSecs(fileContent, subjectHeader) {
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
}

function formatTime(totalSecs, isBold = false) {
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
}

// Data Fetching and Table Generation
const allPages = dv.pages('#study-log').where(p => p.file.name.match(/\d{4}-\d{2}-\d{2}/)).sort(p => p.file.name, 'desc');
const masterTableData = [];

for (let p of allPages) {
    const content = await dv.io.load(p.file.path);
    const m = extractSecs(content, "**Math**");
    const ph = extractSecs(content, "**Physics**");
    const ch = extractSecs(content, "**Chemistry**");
    const total = m + ph + ch;
    
    // Use Dataview's native fileLink function for clean rendering and sorting
    const fullDateLink = dv.fileLink(p.file.path, false, p.file.name);
    
    const mStr = formatTime(m);
    const phStr = formatTime(ph);
    const chStr = formatTime(ch);
    const totStr = formatTime(total, true);

    masterTableData.push([fullDateLink, mStr, phStr, chStr, totStr]);
}

// Render the table directly without the callout dropdown
dv.table(["Date", "Math", "Phy", "Chem", "Tot"], masterTableData);

```