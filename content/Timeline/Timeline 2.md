
```chronos
> DEFAULTVIEW -0001-12-31|0000-01-05
- [-0012-01-01~0000-01-01] #cyan {Sessions} Session-1 10.10.2025
> ORDERBY -color|start

#Session-1
- [-0012-01-01~0000-01-01] #red [[Krieg zwischen Howard und Shal’Khazir]]
- [0000-01-01]  [[Schlacht von Drakkon]]
- [ -0010-01-01T00:00:00] [[Vorgeschichte B pt1]]
- [ -0002-10-01T00:00:00] [[Vorgeschichte B pt2]]
- [ -0002-03-23] [[Vorgeschichte Torgael Gorrn pt1]]
- [ -0001-10-23] [[Vorgeschichte Torgael Gorrn pt2]]
- [ -0005-08-15] [[Vorgeschichte Shirvan pt1]]
- [ -0001-06-25] [[Vorgeschichte Shirvan pt2]]

#Session-2
- [ 0000-01-01~0000-01-03] {Sessions} Session-2 19.10.2025
- [0000-01-02T12:00:00] [[Flucht aus dem Heerlager]]
- [0000-01-02T18:00:000] [[Die Durchquerung des Canyons]]
- [0000-01-03T12:00:00] [[Ankunft in Gibra]]
- [0000-01-03T16:00:00] [[Der Knacker-Turm]]
    
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
else {
	dv.paragraph(`⚠️ **Alle Ereignisse verlinkt**`);
}
}
```