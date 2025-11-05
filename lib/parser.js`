// File: `lib/parser.js`
/* Simple stack trace analyzer with heuristics for Java, Node and Python */
function firstUserFrame(frames) {
  const ignore = [
    'java.lang.', 'org.springframework', 'sun.reflect', 'jdk.internal',
    'node:internal', 'node_modules', '(internal', 'com.intellij'
  ];
  return frames.find(f => !ignore.some(i => f.frameString.includes(i))) || frames[0];
}

function parseJavaFrame(line) {
  // e.g. at com.example.MyClass.myMethod(MyClass.java:123)
  const m = line.match(/at\s+([^\s]+)\(([^:]+):(\d+)\)/);
  if (m) {
    return { function: m[1], file: m[2], line: Number(m[3]), frameString: line.trim() };
  }
  return null;
}

function parseNodeFrame(line) {
  // e.g. at Object.foo (/path/to/file.js:10:15) or at /path/to/file.js:10:15
  // try file:line:col
  let m = line.match(/at\s+(?:.+?\s+\()?(.+):(\d+):(\d+)\)?/);
  if (m) {
    return { function: null, file: m[1], line: Number(m[2]), frameString: line.trim() };
  }
  // try file:line (no column)
  m = line.match(/at\s+(?:.+?\s+\()?(.+):(\d+)\)?/);
  if (m) {
    return { function: null, file: m[1], line: Number(m[2]), frameString: line.trim() };
  }
  return null;
}

function parsePythonFrame(line) {
  // e.g.   File "/path/to/file.py", line 12, in <module>
  const m = line.match(/File\s+"([^"]+)",\s+line\s+(\d+),\s+in\s+(.*)/);
  if (m) {
    return { function: m[3], file: m[1], line: Number(m[2]), frameString: line.trim() };
  }
  return null;
}

function extractTraces(text) {
  const lines = text.split(/\r?\n/);
  const traces = [];
  for (let i = 0; i < lines.length; i++) {
    const l = lines[i];
    // relaxed heuristics for start of a trace / exception
    if (/(Exception|Error|Traceback|Throwable)/i.test(l) || /^\s*Caused by:/i.test(l)) {
      const block = [l];
      let j = i + 1;
      while (j < lines.length && (lines[j].match(/^\s+at\s+/) || lines[j].match(/^\s+File\s+/) || lines[j].trim() === '' || lines[j].match(/^\s+\.{3}/))) {
        block.push(lines[j]);
        j++;
      }
      // Look ahead for a following exception line. Some languages (Python) place the final
      // exception name/message after the indented file/code lines; there may be intervening
      // non-exception indented lines. Scan forward past indented/code lines to find a
      // non-indented exception header and include it.
      let k = j;
      while (k < lines.length && (lines[k].trim() === '' || /^\s+/.test(lines[k]))) {
        k++;
      }
      if (k < lines.length) {
        const next = lines[k].trim();
        // If the following non-indented line is an exception header or an explicit
        // "Caused by:" line, include it and any following indented frame lines
        // into the current block (this captures nested "Caused by" traces).
        if (next && (/^Caused by:/i.test(next) || /^(?:[A-Za-z0-9_.<>$]+(?:Exception|Error|Throwable|Warning))(?:\:)?\s*/.test(next))) {
          // include the header
          block.push(lines[k]);
          // absorb any indented frames after that header
          let m = k + 1;
          while (m < lines.length && (lines[m].match(/^\s+at\s+/) || lines[m].match(/^\s+File\s+/) || lines[m].trim() === '' || lines[m].match(/^\s+\.{3}/))) {
            block.push(lines[m]);
            m++;
          }
          i = Math.max(i, m - 1);
          j = m;
        }
      }
      traces.push({ header: l.trim(), lines: block });
      i = Math.max(i, j - 1);
    }
  }
  return traces;
}

function extractExceptionInfo(lines, header) {
  // Prefer explicit "Caused by:" lines
  for (const raw of lines) {
    const l = raw.trim();
    const caused = l.match(/^Caused by:\s*(.+)$/i);
    if (caused) {
      const txt = caused[1];
      const parts = txt.split(/:\s*(.+)/);
      const name = parts[0].trim();
      const msg = (parts[1] || '').trim();
      return { exception: name, message: msg };
    }
  }

  // Search from bottom for a typical "Name: message" line (Python/Java/Node)
  for (let i = lines.length - 1; i >= 0; i--) {
    const l = lines[i].trim();
    if (!l) continue;
    // match patterns like "java.lang.NullPointerException: something" or "ValueError: msg" or "Error: msg"
    const m = l.match(/^(.+?(?:Exception|Error|Throwable|Warning|Exit))(?:\:)?\s*(.*)$/);
    if (m) {
      return { exception: m[1].trim(), message: (m[2] || '').trim() };
    }
  }

  // Fall back to header -> try to split header into name and message
  const h = header || '';
  const hm = h.match(/^(.+?(?:Exception|Error|Throwable|Warning))(?:\:)?\s*(.*)$/);
  if (hm) {
    return { exception: hm[1].trim(), message: (hm[2] || '').trim() };
  }

  // Nothing found
  return { exception: h.trim(), message: '' };
}

// New: find the deepest/root cause exception by traversing Caused by: lines or taking the last exception header
function findRootCause(lines, header) {
  let lastCaused = null;
  const excRegex = /^(.+?(?:Exception|Error|Throwable|Warning|Exit))(?:\:)?\s*(.*)$/;

  // First, prefer explicit "Caused by:" lines (take the last one)
  for (const raw of lines) {
    const l = raw.trim();
    const caused = l.match(/^Caused by:\s*(.+)$/i);
    if (caused) {
      lastCaused = caused[1];
    }
  }
  if (lastCaused) {
    const parts = lastCaused.split(/:\s*(.+)/);
    const name = parts[0].trim();
    const msg = (parts[1] || '').trim();
    return { exception: name, message: msg };
  }

  // If no explicit Caused by, find all exception-like lines and take the deepest (last) occurrence
  let lastMatch = null;
  for (let i = 0; i < lines.length; i++) {
    const l = lines[i].trim();
    if (!l) continue;
    const m = l.match(excRegex);
    if (m) lastMatch = { name: m[1].trim(), msg: (m[2] || '').trim() };
  }
  if (lastMatch) {
    return { exception: lastMatch.name, message: lastMatch.msg };
  }

  // Fall back to header
  const h = header || '';
  const hm = h.match(excRegex);
  if (hm) return { exception: hm[1].trim(), message: (hm[2] || '').trim() };

  return { exception: h.trim(), message: '' };
}

function analyze(text) {
  if (!text || !text.trim()) return { found: false };
  const traces = extractTraces(text);
  if (!traces.length) return { found: false };

  // For each trace, parse frames
  const candidates = traces.map(t => {
    const parsedFrames = [];
    for (const line of t.lines) {
      const p = parseJavaFrame(line) || parseNodeFrame(line) || parsePythonFrame(line);
      if (p) parsedFrames.push(p);
    }
    return { header: t.header, lines: t.lines, frames: parsedFrames };
  }).filter(t => t.frames.length);

  if (!candidates.length) return { found: false };

  // Pick the trace with the most frames (heuristic)
  candidates.sort((a, b) => b.frames.length - a.frames.length);
  const chosen = candidates[0];
  const frame = firstUserFrame(chosen.frames);

  // Extract exception name and message
  const info = extractExceptionInfo(chosen.lines, chosen.header);
  const exceptionText = info.exception + (info.message ? ': ' + info.message : '');

  // Determine the deepest/root cause (prefer explicit "Caused by:" or the last exception header)
  const root = findRootCause(chosen.lines, chosen.header);
  const rootText = root.exception + (root.message ? ': ' + root.message : '');

  // Build a StackOverflow search link using the root exception (more likely to be the real cause)
  const query = encodeURIComponent((root.exception + ' ' + (frame.function || frame.file)).trim());
  const soLink = `https://stackoverflow.com/search?q=${query}`;

  return {
    found: true,
    exception: exceptionText,
    exceptionMessage: info.message || '',
    rootException: rootText,
    rootExceptionMessage: root.message || '',
    frameString: frame.frameString,
    file: frame.file,
    line: frame.line,
    function: frame.function,
    link: soLink
  };
}

module.exports = { analyze };