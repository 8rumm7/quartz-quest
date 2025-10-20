```dataviewjs
// -------- Einstellungen ----------
const TIME_RANGE   = '-0001-12-30|0000-01-06';
const EVENTS_FOLDER   = '"Welt/Ereignisse"';
const SESSIONS_FOLDER = '"Log/Sessions"';
const DATE_FIELD      = 'Datum';            // Text, z.B. "0005-01-01"
const RANGE_FIELD     = 'Zeitraum';         // Text, z.B. "0005-03-10~0005-05-22"
// ---------------------------------

// --- Utilities ---
function padYear4(s) {
  // auch negative Jahre auf 4-stellig auffüllen: -5 -> -0005, 5 -> 0005
  if (!s) return s;
  const m = String(s).match(/^(-?)(\d{1,3})(.*)$/);
  if (!m) return s;
  return m[1] + m[2].padStart(4,'0') + m[3];
}
function sanitizeChronosDate(s) {
  if (!s) return '';
  let x = String(s).trim();
  x = x.replace(/\.\d+/, '');                  // Millisekunden weg
  x = x.replace(/(Z|[+\-]\d{2}:\d{2})$/, '');  // Zeitzone weg
  x = x.replace(/\s+/g, '');
  // Jahr 1–3-stellig (inkl. negativ) -> auf 4-stellig
  x = padYear4(x);
  return x;
}
const DATE_RX = /^-?\d{1,6}(?:-\d{2}(?:-\d{2}(?:T\d{2}(?::\d{2}(?::\d{2})?)?)?)?)?$/;
const isValidChronosDate = s => DATE_RX.test(s);

// Parser in Komponenten (inkl. Defaults)
function parseParts(s) {
  const m = s.match(/^(-?)(\d{1,6})(?:-(\d{2})(?:-(\d{2})(?:T(\d{2})(?::(\d{2})(?::(\d{2}))?)?)?)?)?$/);
  if (!m) return null;
  const sign = m[1] === '-' ? -1 : 1;
  const year = Number(m[2]);
  const mon  = m[3] ? Number(m[3]) : 1;
  const day  = m[4] ? Number(m[4]) : 1;
  const hh   = m[5] ? Number(m[5]) : 0;
  const mm   = m[6] ? Number(m[6]) : 0;
  const ss   = m[7] ? Number(m[7]) : 0;
  return { sign, year, mon, day, hh, mm, ss };
}
// Vergleich zweier Chronos-Dates (true wenn a < b)
function lessThan(a, b) {
  const A = parseParts(a), B = parseParts(b);
  if (!A || !B) return String(a) < String(b);
  // zuerst nach Era: negative (v. Chr.) ist "früher" als positiv
  if (A.sign !== B.sign) return A.sign < B.sign;
  // innerhalb gleicher Era: bei negativen Jahren ist größeres year "früher" (z.B. -0012 < -0005)
  const dir = (A.sign === -1) ? -1 : 1;
  const seqA = [A.year*dir, A.mon, A.day, A.hh, A.mm, A.ss];
  const seqB = [B.year*dir, B.mon, B.day, B.hh, B.mm, B.ss];
  for (let i=0; i<seqA.length; i++) {
    if (seqA[i] !== seqB[i]) return seqA[i] < seqB[i];
  }
  return false;
}
function minDate(a, b) { return lessThan(a,b) ? a : b; }
function maxDate(a, b) { return lessThan(a,b) ? b : a; }

function sanitizeRange(s) {
  if (!s) return null;
  const parts = String(s).split(/\s*~\s*/).map(x => sanitizeChronosDate(x));
  if (parts.length !== 2) return null;
  const [a, b] = parts;
  if (!isValidChronosDate(a) || !isValidChronosDate(b)) return null;
  const start = lessThan(a,b) ? a : b;
  const end   = lessThan(a,b) ? b : a;
  return { start, end };
}

// #red für "Schlacht"
function isSchlacht(p) {
  const v = p["Ereignistyp"];
  if (v == null) return false;
  if (Array.isArray(v)) return v.some(x => String(x).trim().toLowerCase() === "schlacht");
  return String(v).trim().toLowerCase() === "schlacht";
}

// Paritätsfarbe für Sessions nach Muster "Session NN"
function sessionColorToken(fileName) {
  const m = String(fileName).match(/session\s+(\d+)/i);
  if (!m) return "";
  const n = parseInt(m[1], 10);
  return (n % 2 === 0) ? "#cyan " : "#blue ";
}

// ===== 1) Sessions einsammeln und zu Perioden verdichten =====
const sessionPages = dv.pages(SESSIONS_FOLDER);
const sessionItems = [];

for (const s of sessionPages) {
  let minStart = null, maxEnd = null;

  // a) eigene Felder der Session
  const sRange = sanitizeRange(s[RANGE_FIELD]);
  if (sRange) { minStart = sRange.start; maxEnd = sRange.end; }
  const sDate = s[DATE_FIELD] ? sanitizeChronosDate(s[DATE_FIELD]) : null;
  if (sDate && isValidChronosDate(sDate)) {
    minStart = minStart ? minDate(minStart, sDate) : sDate;
    maxEnd   = maxEnd   ? maxDate(maxEnd, sDate)   : sDate;
  }

  // b) Outlinks zu Ereignissen (Datum + Zeitraum)
  const evLinks = (s.file.outlinks ?? [])
    .filter(l => l.path && l.path.startsWith("Welt/Ereignisse/"));

  for (const l of evLinks) {
    const ev = dv.page(l.path);
    if (!ev) continue;

    const zr = sanitizeRange(ev[RANGE_FIELD]);
    if (zr) {
      minStart = minStart ? minDate(minStart, zr.start) : zr.start;
      maxEnd   = maxEnd   ? maxDate(maxEnd, zr.end)     : zr.end;
    }

    const d = ev[DATE_FIELD] ? sanitizeChronosDate(ev[DATE_FIELD]) : null;
    if (d && isValidChronosDate(d)) {
      minStart = minStart ? minDate(minStart, d) : d;
      maxEnd   = maxEnd   ? maxDate(maxEnd, d)   : d;
    }
  }

  if (minStart && maxEnd) {
    const color = sessionColorToken(s.file.name);
    const link  = `[[${s.file.name}]]`;  // nur Dateiname
    sessionItems.push({
      date: minStart,
      line: `@ [${minStart}~${maxEnd}] ${color}{Sessions} ${link}`
    });
  }
}

// ===== 2) Ereignisse (Punkte/Perioden), Gruppe {Ereignisse} =====
const eventPages = dv.pages(EVENTS_FOLDER).where(p => p[DATE_FIELD] || p[RANGE_FIELD]);
const eventItems = [];

for (const p of eventPages) {
  const link = `[[${p.file.name}]]`;    // nur Dateiname
  const color = isSchlacht(p) ? "#red " : "";

  const range = sanitizeRange(p[RANGE_FIELD]);
  if (range) {
    eventItems.push({
      date: range.start,
      line: `@ [${range.start}~${range.end}] ${color}{Ereignisse} ${link}`
    });
    continue;
  }

  const d = p[DATE_FIELD] ? sanitizeChronosDate(p[DATE_FIELD]) : null;
  if (d && isValidChronosDate(d)) {
    eventItems.push({
      date: d,
      line: `- [${d}] ${color}{Ereignisse} ${link}`
    });
  }
}

// ===== 3) Ausgeben: erst Sessions (oben), dann Ereignisse =====
sessionItems.sort((a,b) => lessThan(a.date,b.date) ? -1 : 1);
eventItems.sort((a,b) => lessThan(a.date,b.date) ? -1 : 1);

const allLines = [...sessionItems, ...eventItems].map(x => x.line).join("\n");

dv.paragraph(
  allLines.length
    ? `\`\`\`chronos
> DEFAULTVIEW ${TIME_RANGE}
> ORDERBY start
# Welt – Sitzungen & Ereignisse
${allLines}
\`\`\``
    : "Keine gültigen Daten für Sessions/Ereignisse gefunden."
);

```

```dataviewjs
// ---- Einstellungen ----
const SESSIONS_FOLDER = '"Log/Sessions"';
const EVENTS_FOLDER   = '"Welt/Ereignisse"';
// -----------------------

const sessions   = dv.pages(SESSIONS_FOLDER);
const eventPages = dv.pages(EVENTS_FOLDER);

// Hilfsfunktion: aus einem Outlink den reinen Dateinamen gewinnen
function linkToBaseName(l) {
  if (!l) return null;
  const raw = String(l.path ?? l).trim();        // .path, falls vorhanden
  if (!raw) return null;
  const noExt = raw.replace(/\.md$/i, "");
  const parts = noExt.split("/");
  return parts[parts.length - 1];                // letzter Segment = Dateiname
}

// 1) Alle verlinkten Namen aus Sessions einsammeln (nur Dateinamen, kleingeschrieben)
const linkedNames = new Set();
for (const s of sessions) {
  const outs = s.file.outlinks ?? [];
  for (const l of outs) {
    const name = linkToBaseName(l);
    if (name) linkedNames.add(name.toLowerCase());
  }
}

// 2) Ereignisse, die nicht in irgendeiner Session verlinkt sind
const unlinked = [];
for (const ev of eventPages) {
  const name = ev.file.name.toLowerCase();
  if (!linkedNames.has(name)) unlinked.push(ev);
}

// 3) Ausgabe
if (unlinked.length == 1) {
  dv.paragraph(`⚠️ **${unlinked.length} Ereignis nicht in Sessions verlinkt:**`);
  dv.list(unlinked.map(p => `[[${p.file.name}]]`));  // nur Dateiname als Link
} else {
if (unlinked.length > 0) {
  dv.paragraph(`⚠️ **${unlinked.length} Ereignisse nicht in Sessions verlinkt:**`);
  dv.list(unlinked.map(p => `[[${p.file.name}]]`));  // nur Dateiname als Link
  }
}
```