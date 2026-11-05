/* ============================================================
   tts.js — ระบบเสียงช่วยอ่าน (Web Speech API + เสียงไทยออนไลน์)
   แบบเดียวกับเว็บ la2569:
   - ปุ่ม 🔊 อ่าน หน้าแต่ละ section (h2), หัวข้อย่อย (h3) และแต่ละข้อ quiz
   - แถบควบคุมลอย: เลือกเสียงไทย/อังกฤษ, ความเร็ว, พัก/หยุด, อ่านทั้งหน้า
   - ไฮไลต์หัวข้อที่กำลังอ่าน + เลื่อนจอตาม
   เสียงไทยเมื่อเครื่องไม่มี voice ไทย (เช่น Chrome บน Linux):
   - เอนจิน "อัตโนมัติ" จะส่งท่อนภาษาไทยไปให้ Google TTS (translate_tts)
     อ่านแทนโดยอัตโนมัติ — ใช้ได้ทุกเครื่อง แค่ต้องต่ออินเทอร์เน็ต
   - เลือกเอนจินเองได้ในแถบควบคุม: อัตโนมัติ / ในเครื่อง / ออนไลน์
   ปรับให้เข้ากับเว็บ OS:
   - ตาราง → อ่านเป็นประโยค · .steps → "ขั้นที่ N" · ยกกำลัง → "ยกกำลัง"
   - บล็อกโค้ด/แผนภาพ → บอกสั้น ๆ ไม่อ่านโค้ดดิบ
   - quiz → อ่านโจทย์แล้วให้ลองทำเอง (ไม่อ่านตัวเลือก/เฉลย)
   ============================================================ */
'use strict';
(function () {
  const LS_KEY = 'os-tts';

  const THAI_RE = /[\u0E00-\u0E7F]/;
  const EMOJI_RE = /[\u{1F000}-\u{1FAFF}\u{1F1E6}-\u{1F1FF}\u{2600}-\u{27BF}\u{2B00}-\u{2BFF}\u{2190}-\u{21FF}\u{2300}-\u{23FF}\u{FE00}-\u{FE0F}\u{200D}\u{2764}]/gu;

  const state = {
    engine: 'auto',          // auto | local | google
    voiceTh: null, voiceEn: null, rate: 1,
    chunks: [], idx: 0, playing: false, paused: false, stopped: true,
    target: null, label: '', epoch: 0,
    audio: null,             // เสียงจาก Google TTS ที่กำลังเล่นอยู่
  };

  /* ---------- ตั้งค่าที่บันทึกไว้ ---------- */
  function loadPrefs() {
    try {
      const p = JSON.parse(localStorage.getItem(LS_KEY) || '{}');
      if (p.rate) state.rate = p.rate;
      if (p.engine) state.engine = p.engine;
      state._savedTh = p.th || '';
      state._savedEn = p.en || '';
    } catch (e) { /* ไม่มีค่าเดิมก็ข้าม */ }
  }
  function savePrefs() {
    try {
      localStorage.setItem(LS_KEY, JSON.stringify({
        th: state.voiceTh ? state.voiceTh.name : '',
        en: state.voiceEn ? state.voiceEn.name : '',
        rate: state.rate,
        engine: state.engine,
      }));
    } catch (e) { /* localStorage ใช้ไม่ได้ก็ไม่เป็นไร */ }
  }

  /* ---------- รายชื่อเสียงในเครื่อง ---------- */
  function hasSpeech() { return 'speechSynthesis' in window; }
  function allVoices() {
    if (!hasSpeech()) return [];
    const vs = speechSynthesis.getVoices();
    const uniq = new Map();
    vs.forEach((v) => { if (!uniq.has(v.name)) uniq.set(v.name, v); });
    return [...uniq.values()];
  }
  function localTh() { return allVoices().filter((v) => /^th/i.test(v.lang)); }
  function localEn() { return allVoices().filter((v) => /^en/i.test(v.lang)); }
  function fillSelect(sel, voices, savedName, placeholder) {
    sel.innerHTML = '';
    if (!voices.length) {
      const o = document.createElement('option');
      o.value = '';
      o.textContent = placeholder;
      sel.appendChild(o);
      return;
    }
    voices.forEach((v) => {
      const o = document.createElement('option');
      o.value = v.name;
      const isTh = /^th/i.test(v.lang);
      o.textContent = (isTh ? '🇹🇭 ' : '') + v.name + ' (' + v.lang + ')';
      sel.appendChild(o);
    });
    if (savedName && voices.some((v) => v.name === savedName)) sel.value = savedName;
  }
  function refreshVoices() {
    const list = allVoices();
    if (!list.length && tries > 10) return;
    const th = list.filter((v) => /^th/i.test(v.lang));
    const en = list.filter((v) => /^en/i.test(v.lang));
    const rest = list.filter((v) => !/^th/i.test(v.lang) && !/^en/i.test(v.lang));
    fillSelect(el.selTh, th.concat(en, rest), state._savedTh, '🇹🇭 ออนไลน์ (Google) — ต้องต่อเน็ต');
    fillSelect(el.selEn, en.concat(th), state._savedEn, '🌐 ออนไลน์ (Google) — ต้องต่อเน็ต');
    state.voiceTh = byName(el.selTh) || th[0] || null;
    state.voiceEn = byName(el.selEn) || en[0] || null;
    updateHint(th.length > 0);
    if (state.voiceTh) el.selTh.value = state.voiceTh.name;
    if (state.voiceEn) el.selEn.value = state.voiceEn.name;
    if (el.selEngine) el.selEngine.value = state.engine;
  }
  function byName(sel) { return allVoices().find((v) => v.name === sel.value) || null; }
  function updateHint(hasThai) {
    if (hasThai) {
      el.hint.textContent = 'เครื่องนี้มีเสียงไทยในตัว ✓';
    } else {
      el.hint.textContent = 'เครื่องนี้ไม่มีเสียงไทยในตัว — โหมด "อัตโนมัติ" จะใช้ 🇹🇭 เสียงไทยออนไลน์ (Google) ให้เอง ต้องต่ออินเทอร์เน็ต (ท่อนแรกอาจหน่วง 1–2 วินาที ท่อนถัด ๆ ไปจะต่อเนื่อง)';
    }
  }

  /* ---------- เลือกเอนจินของแต่ละท่อน ---------- */
  function engineForChunk(thai) {
    if (state.engine === 'google') return 'google';
    if (state.engine === 'local') {
      if (hasSpeech()) return 'local';
      return 'google';
    }
    /* auto: มีเสียงตรงภาษาในเครื่อง → ใช้เครื่อง, ไม่มี → ออนไลน์ */
    if (thai) return (hasSpeech() && localTh().length) ? 'local' : 'google';
    return (hasSpeech() && localEn().length) ? 'local' : 'google';
  }
  const ENGINE_NAME = { auto: 'อัตโนมัติ', local: 'ในเครื่อง', google: 'ออนไลน์' };

  /* ---------- ดึงข้อความจาก DOM (ปรับให้อ่านลื่น) ---------- */
  function cleanText(root) {
    const clone = root.cloneNode(true);
    /* ตัดสิ่งที่ไม่ควรอ่าน: ปุ่ม/ฟอร์ม/เมนู/ตัวเลือก quiz/เฉลย/แถบเครื่องมือ */
    clone.querySelectorAll(
      'script, style, svg, input, button, select, label, textarea, .tts-btn, .tts-bar,' +
      '.choices, .q-verdict, .q-explain, .q-actions, .toc, .pager, .chip-row, .gantt, .gantt-scale,' +
      '.sidebar, .topbar, #toTop, .codetag, .uc-tags, .badge-pts'
    ).forEach((n) => n.remove());
    /* โค้ดดิบ: อ่านแค่บอกว่ามีโค้ด ไม่อ่านทีละบรรทัด */
    clone.querySelectorAll('pre.code').forEach((p) => {
      const tag = p.closest('.codewrap')?.querySelector('.codetag');
      const name = tag ? 'โค้ดตัวอย่าง ' + tag.textContent.trim() : 'ตัวอย่างโค้ด';
      p.replaceWith(document.createTextNode(' (' + name + ' — ดูรายละเอียดบนหน้าจอ) '));
    });
    /* quiz: อ่านเฉพาะโจทย์ + บอกให้ลองทำเอง (ตัดตัวเลือก/เฉลยไปแล้วข้างบน) */
    clone.querySelectorAll('.quiz').forEach((q) => {
      q.appendChild(document.createTextNode(' จบข้อนี้ — ลองทำเองก่อน แล้วกดปุ่มตรวจคำตอบบนหน้าจอ เพื่อดูเฉลยทีละขั้น '));
    });
    /* ตาราง: อ่านเป็นประโยคแต่ละแถว */
    clone.querySelectorAll('table.tbl').forEach((t) => {
      const lines = [];
      t.querySelectorAll('tr').forEach((tr) => {
        const cells = [...tr.querySelectorAll('th,td')].map((c) => c.textContent.replace(/\s+/g, ' ').trim()).filter(Boolean);
        if (cells.length) lines.push(cells.join(' คือ '));
      });
      t.replaceWith(document.createTextNode(' (ตาราง) ' + lines.join('. ') + '. (จบตาราง) '));
    });
    /* ขั้นตอน .steps: เติมเลขขั้นให้ TTS อ่าน */
    clone.querySelectorAll('.steps').forEach((ol) => {
      [...ol.querySelectorAll(':scope > li')].forEach((li, i) => {
        li.prepend(document.createTextNode('ขั้นที่ ' + (i + 1) + ': '));
      });
    });
    /* แถวการ์ด .memgrid: อ่านรวมเป็นรายการ */
    clone.querySelectorAll('.memgrid').forEach((g) => {
      const items = [...g.querySelectorAll('.memcell')].map((m) => m.textContent.replace(/\s+/g, ' ').trim()).filter(Boolean);
      g.replaceWith(document.createTextNode(' (รายการ) ' + items.join(', ') + ' '));
    });
    /* อักษรยก 2ⁿ → "2 ยกกำลัง n" */
    clone.querySelectorAll('sup').forEach((s) => {
      s.replaceWith(document.createTextNode(' ยกกำลัง ' + s.textContent.trim() + ' '));
    });
    /* ลบอีโมจิ/กวาดช่องว่าง */
    let text = clone.textContent || '';
    text = text.replace(EMOJI_RE, ' ');
    text = text.replace(/[\t\r]+/g, ' ').replace(/[ \u00A0]+/g, ' ');
    text = text.replace(/\n{2,}/g, '\n').replace(/\n /g, '\n').trim();
    text = text.replace(/ +([,.;:!?])/g, '$1');
    return text;
  }

  /* ---------- หั่นข้อความเป็นท่อน ๆ สำหรับอ่าน ---------- */
  function chunkText(text) {
    const parts = text.split(/\n+/).flatMap((para) => {
      const sents = para.split(/(?<=[\.!?;:ๆ])\s+/);
      const out = [];
      for (let s of sents) {
        s = s.trim();
        if (!s) continue;
        if (s.length <= 190) { out.push(s); continue; }
        /* ท่อนยาวเกิน: หั่นตรงช่องว่างใกล้ ๆ 175 ตัวอักษร */
        let rest = s;
        while (rest.length > 190) {
          let cut = rest.lastIndexOf(' ', 175);
          if (cut < 60) cut = 175;
          out.push(rest.slice(0, cut).trim());
          rest = rest.slice(cut).trim();
        }
        if (rest) out.push(rest);
      }
      return out;
    });
    /* รวมท่อนสั้น ๆ ที่ติดกันให้อ่านต่อเนื่อง (ไม่เกิน 190 เพราะ Google TTS จำกัด ~200 ตัวอักษร) */
    const merged = [];
    for (const p of parts) {
      if (merged.length && (merged[merged.length - 1] + ' ' + p).length <= 190) {
        merged[merged.length - 1] += ' ' + p;
      } else merged.push(p);
    }
    return merged.filter(Boolean).map((t) => ({ text: t, thai: THAI_RE.test(t) }));
  }

  /* ---------- เครื่องยนต์อ่าน ---------- */
  function startRead(target, label) {
    const text = cleanText(target);
    if (!text) { setStatus('ไม่พบข้อความให้อ่าน'); return; }
    stopAudio();
    if (hasSpeech()) speechSynthesis.cancel();
    state.epoch += 1;
    state.stopped = false; state.playing = true; state.paused = false;
    state.target = target; state.label = label;
    state.chunks = chunkText(text); state.idx = 0;
    const anchor = target.__anchor || target;
    markReading(anchor);
    if (typeof anchor.scrollIntoView === 'function') {
      try { anchor.scrollIntoView({ behavior: 'smooth', block: 'start' }); } catch (e) { /* เลื่อนไม่ได้ก็อ่านต่อ */ }
    }
    el.pauseBtn.textContent = '⏸ พัก';
    speakChunk(0, state.epoch);
  }

  function speakChunk(i, epoch) {
    if (state.stopped || epoch !== state.epoch) return;
    if (i >= state.chunks.length) { finish(); return; }
    state.idx = i;
    const c = state.chunks[i];
    setStatus('กำลังอ่าน (' + (i + 1) + '/' + state.chunks.length + '): ' + state.label);
    if (engineForChunk(c.thai) === 'google') playOnlineChunk(c, i, epoch);
    else speakLocalChunk(c, i, epoch);
  }

  /* เสียงในเครื่อง (Web Speech API) */
  function speakLocalChunk(c, i, epoch) {
    if (!hasSpeech()) { setTimeout(() => speakChunk(i + 1, epoch), 10); return; }
    const u = new SpeechSynthesisUtterance(c.text);
    const v = c.thai ? state.voiceTh : state.voiceEn;
    if (v) u.voice = v;
    u.lang = c.thai ? 'th-TH' : 'en-US';
    u.rate = state.rate;
    u.onend = () => {
      if (epoch !== state.epoch || state.stopped || state.paused) return;
      setTimeout(() => speakChunk(i + 1, epoch), 60);
    };
    u.onerror = (e) => {
      if (e.error !== 'interrupted' && e.error !== 'canceled' && epoch === state.epoch) finish();
    };
    speechSynthesis.speak(u);
  }

  /* เสียงออนไลน์ (Sound of Text — ใช้เสียง Google, มีภาษาไทย, เรียกจากเบราว์เซอร์ได้ตรง ๆ) */
  const soCache = new Map(); /* text → mp3 url */
  function sleep(ms) { return new Promise((r) => setTimeout(r, ms)); }
  async function onlineLocation(text, thai) {
    const key = (thai ? 'th:' : 'en:') + text;
    if (soCache.has(key)) return soCache.get(key);
    const cr = await fetch('https://api.soundoftext.com/sounds', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ engine: 'Google', data: { text, voice: thai ? 'th-TH' : 'en-US' } }),
    }).then((r) => r.json());
    if (!cr.id) throw new Error('สร้างเสียงไม่สำเร็จ');
    let loc = null;
    for (let t = 0; t < 25 && !loc; t++) {
      await sleep(600);
      const s = await fetch('https://api.soundoftext.com/sounds/' + cr.id).then((r) => r.json());
      if (s.status === 'Done') loc = s.location;
      else if (s.status === 'Error') throw new Error('งานเสียงล้มเหลว');
    }
    if (!loc) throw new Error('หมดเวลารอเสียง');
    soCache.set(key, loc);
    return loc;
  }
  async function playOnlineChunk(c, i, epoch) {
    stopAudio();
    let loc;
    try {
      loc = await onlineLocation(c.text, c.thai);
    } catch (e) {
      if (epoch !== state.epoch || state.stopped) return;
      setStatus('⚠️ เสียงออนไลน์ใช้ไม่ได้ช่วงนี้ (' + e.message + ') — ลองเสียงในเครื่องแทน');
      if (hasSpeech() && allVoices().length) speakLocalChunk(c, i, epoch);
      else setTimeout(() => speakChunk(i + 1, epoch), 10);
      return;
    }
    if (epoch !== state.epoch || state.stopped) return;
    const a = new Audio(loc);
    state.audio = a;
    a.playbackRate = Math.min(2, Math.max(0.5, state.rate));
    a.onended = () => {
      if (epoch !== state.epoch || state.stopped || state.paused) return;
      setTimeout(() => speakChunk(i + 1, epoch), 150);
    };
    a.onerror = () => {
      if (epoch !== state.epoch || state.stopped) return;
      setStatus('⚠️ เล่นไฟล์เสียงไม่สำเร็จ — ข้ามไปท่อนถัดไป');
      setTimeout(() => speakChunk(i + 1, epoch), 10);
    };
    const p = a.play();
    if (p && p.catch) p.catch(() => { /* autoplay ถูกบล็อก — รอผู้ใช้กดใหม่ */ });
    /* ดึงเสียงท่อนถัดไปเตรียมไว้ล่วงหน้า ให้อ่านต่อเนื่องไม่มีช่วงเงียบ */
    const nxt = state.chunks[i + 1];
    if (nxt && engineForChunk(nxt.thai) === 'google') {
      onlineLocation(nxt.text, nxt.thai).catch(() => { /* prefetch พลาดก็ปล่อย */ });
    }
  }
  function stopAudio() {
    if (state.audio) {
      try { state.audio.pause(); } catch (e) { /* ไม่เป็นไร */ }
      state.audio.onended = null;
      state.audio.onerror = null;
      state.audio = null;
    }
  }

  function stopAll() {
    state.epoch += 1;
    state.stopped = true; state.playing = false; state.paused = false;
    stopAudio();
    if (hasSpeech()) speechSynthesis.cancel();
    clearReading();
    setStatus('หยุดแล้ว');
    el.pauseBtn.textContent = '⏸ พัก';
  }
  function togglePause() {
    if (!state.playing) return;
    if (state.paused) {
      if (state.audio) state.audio.play().catch(() => { });
      else if (hasSpeech()) speechSynthesis.resume();
      state.paused = false;
      el.pauseBtn.textContent = '⏸ พัก';
      setStatus('กำลังอ่าน: ' + state.label);
    } else {
      if (state.audio) state.audio.pause();
      else if (hasSpeech()) speechSynthesis.pause();
      state.paused = true;
      el.pauseBtn.textContent = '▶ เล่นต่อ';
      setStatus('พักการอ่านอยู่');
    }
  }
  function finish() {
    state.playing = false; state.paused = false; state.stopped = true;
    clearReading();
    setStatus('อ่านจบแล้ว ✓');
    el.pauseBtn.textContent = '⏸ พัก';
  }
  function setStatus(t) { el.status.textContent = t; }
  function markReading(t) { clearReading(); t.classList.add('tts-reading'); }
  function clearReading() { document.querySelectorAll('.tts-reading').forEach((n) => n.classList.remove('tts-reading')); }

  /* ---------- ปุ่ม 🔊 หน้า section / หัวข้อย่อย / ข้อ quiz ---------- */
  function makeBtn(target, label, cls) {
    const b = document.createElement('button');
    b.type = 'button';
    b.className = 'tts-btn' + (cls ? ' ' + cls : '');
    b.textContent = '🔊 อ่าน';
    b.setAttribute('aria-label', 'อ่านออกเสียง: ' + label);
    b.addEventListener('click', (e) => {
      e.stopPropagation();
      if (state.playing && state.target === target) stopAll();
      else startRead(target, label);
    });
    return b;
  }
  /* เนื้อหาของหัวข้อย่อย h3 = ทุก node ถัดจาก h3 จนถึง h1/h2/h3 ถัดไป */
  function subsectionContent(h3) {
    const box = document.createElement('div');
    let node = h3.nextElementSibling;
    while (node && !/^H[123]$/.test(node.tagName)) {
      box.appendChild(node.cloneNode(true));
      node = node.nextElementSibling;
    }
    const head = document.createElement('h3');
    head.textContent = h3.textContent;
    box.prepend(head);
    return box;
  }
  /* เนื้อหาของ section h2 = ทุก node ถัดจาก h2 จนถึง h1/h2 ถัดไป */
  function sectionContent(h2) {
    const box = document.createElement('div');
    let node = h2.nextElementSibling;
    while (node && !/^H[12]$/.test(node.tagName)) {
      box.appendChild(node.cloneNode(true));
      node = node.nextElementSibling;
    }
    const head = document.createElement('h2');
    head.textContent = h2.textContent;
    box.prepend(head);
    return box;
  }
  function cleanLabel(s) { return s.replace(EMOJI_RE, '').replace(/\s+/g, ' ').trim().slice(0, 60); }
  function injectButtons() {
    /* ระดับ section (h2) — .content คือ <main> เอง */
    document.querySelectorAll('.content > h2').forEach((h2) => {
      const box = sectionContent(h2);
      box.__anchor = h2;
      h2.appendChild(makeBtn(box, cleanLabel(h2.textContent), 'tts-sec'));
      /* ระดับหัวข้อย่อย (h3) — อ่านเฉพาะท่อนของหัวข้อนั้น */
      let node = h2.nextElementSibling;
      while (node && !/^H[12]$/.test(node.tagName)) {
        if (node.tagName === 'H3') {
          const sub = subsectionContent(node);
          sub.__anchor = node;
          node.appendChild(makeBtn(sub, cleanLabel(node.textContent), 'tts-sub'));
        }
        node = node.nextElementSibling;
      }
    });
    /* ระดับข้อ quiz — อ่านโจทย์ (ไม่อ่านเฉลย) */
    document.querySelectorAll('.content .quiz[data-q]').forEach((q) => {
      const head = q.querySelector('.quiz-head');
      if (!head) return;
      const box = document.createElement('div');
      box.appendChild(head.cloneNode(true));
      const ctx = q.querySelector('.q-ctx'); if (ctx) box.appendChild(ctx.cloneNode(true));
      const qt = q.querySelector('.q-text'); if (qt) box.appendChild(qt.cloneNode(true));
      box.__anchor = q;
      head.appendChild(makeBtn(box, cleanLabel(head.textContent)));
    });
  }

  /* ---------- แถบควบคุมลอย ---------- */
  const el = {};
  function buildBar() {
    const bar = document.createElement('div');
    bar.className = 'tts-bar';
    bar.innerHTML = `
      <div class="tts-bar-head">
        <span class="tts-bar-title">🔊 อ่านออกเสียง</span>
        <button type="button" class="tts-fold" aria-label="ย่อ/กางแถบควบคุม">▾</button>
      </div>
      <div class="tts-bar-body">
        <div class="tts-status" role="status">พร้อม — กดปุ่ม 🔊 อ่าน ที่หัวข้อหรือข้อฝึกหัดที่ต้องการฟัง</div>
        <div class="tts-row">
          <label>เอนจินเสียง
            <select class="tts-engine">
              <option value="auto" selected>อัตโนมัติ (แนะนำ)</option>
              <option value="local">ในเครื่อง (Web Speech)</option>
              <option value="google">ออนไลน์ (Google เสียงไทย — ต้องต่อเน็ต)</option>
            </select>
          </label>
          <label>ความเร็ว
            <select class="tts-rate">
              <option value="0.75">ช้า 0.75×</option>
              <option value="0.9">ช้านิด 0.9×</option>
              <option value="1" selected>ปกติ 1×</option>
              <option value="1.15">เร็วนิด 1.15×</option>
              <option value="1.3">เร็ว 1.3×</option>
              <option value="1.5">เร็วมาก 1.5×</option>
            </select>
          </label>
        </div>
        <div class="tts-row">
          <label>เสียงไทย (ในเครื่อง) <select class="tts-vth"></select></label>
          <label>เสียงอังกฤษ (ในเครื่อง) <select class="tts-ven"></select></label>
        </div>
        <div class="tts-row">
          <button type="button" class="tts-pause">⏸ พัก</button>
          <button type="button" class="tts-stop">⏹ หยุด</button>
          <button type="button" class="tts-all">📖 อ่านทั้งหน้า</button>
        </div>
        <div class="tts-hint"></div>
      </div>`;
    document.body.appendChild(bar);
    document.body.classList.add('has-tts');
    el.bar = bar;
    el.status = bar.querySelector('.tts-status');
    el.hint = bar.querySelector('.tts-hint');
    el.selEngine = bar.querySelector('.tts-engine');
    el.selTh = bar.querySelector('.tts-vth');
    el.selEn = bar.querySelector('.tts-ven');
    el.selRate = bar.querySelector('.tts-rate');
    el.pauseBtn = bar.querySelector('.tts-pause');
    bar.querySelector('.tts-fold').addEventListener('click', () => {
      bar.classList.toggle('min');
      document.body.classList.toggle('tts-min', bar.classList.contains('min'));
    });
    el.pauseBtn.addEventListener('click', togglePause);
    bar.querySelector('.tts-stop').addEventListener('click', stopAll);
    bar.querySelector('.tts-all').addEventListener('click', () => {
      const inner = document.querySelector('.content');
      if (state.playing && state.target === inner) stopAll();
      else startRead(inner, 'ทั้งหน้า');
    });
    el.selRate.value = String(state.rate);
    el.selRate.addEventListener('change', () => {
      state.rate = parseFloat(el.selRate.value) || 1;
      savePrefs();
      if (state.audio) {
        /* เสียงออนไลน์ปรับความเร็วสด ๆ ได้เลย */
        state.audio.playbackRate = Math.min(2, Math.max(0.5, state.rate));
      } else if (state.playing && !state.paused) {
        /* เสียงในเครื่องรีสตาร์ตท่อนปัจจุบัน */
        const t2 = state.target, l2 = state.label;
        startRead(t2, l2);
      }
    });
    el.selEngine.value = state.engine;
    el.selEngine.addEventListener('change', () => {
      state.engine = el.selEngine.value;
      savePrefs();
      /* ถ้ากำลังอ่านอยู่ เริ่มท่อนปัจจุบันใหม่ด้วยเอนจินที่เลือก */
      if (state.playing && !state.paused) {
        const t2 = state.target, l2 = state.label;
        startRead(t2, l2);
      }
    });
    el.selTh.addEventListener('change', () => { state.voiceTh = byName(el.selTh); savePrefs(); });
    el.selEn.addEventListener('change', () => { state.voiceEn = byName(el.selEn); savePrefs(); });
  }

  /* ---------- เริ่มทำงาน ---------- */
  loadPrefs();
  buildBar();
  injectButtons();
  let tries = 0;
  let timer = null;
  function initVoices() {
    tries += 1;
    refreshVoices();
    if (allVoices().length || tries > 10) { clearInterval(timer); timer = null; }
  }
  if (hasSpeech()) {
    if (typeof speechSynthesis.addEventListener === 'function') {
      speechSynthesis.addEventListener('voiceschanged', refreshVoices);
    } else {
      speechSynthesis.onvoiceschanged = refreshVoices;
    }
    /* เสียงบางเครื่องมาช้า — ลองโหลดซ้ำช่วงแรก */
    timer = setInterval(initVoices, 400);
    initVoices();
  } else {
    refreshVoices();
  }
  window.addEventListener('beforeunload', () => { stopAudio(); if (hasSpeech()) speechSynthesis.cancel(); });
})();
