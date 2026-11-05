/* ============================================================
   OS Final Exam Prep — shared app logic
   - sidebar / theme / toTop
   - quiz engine (รองรับรูปแบบข้อสอบจริง: ถูก/ผิด +2/−1, เติมตัวเลข, ปรนัย)
   - progress บันทึกลง localStorage
   ============================================================ */
(function () {
  "use strict";

  /* ---------- Chapter config (sidebar) ---------- */
  const CHAPTERS = [
    { group: "ภาพรวม", items: [
      { id: "index", num: "🏠", name: "หน้าแรก & แนวข้อสอบ", file: "index.html" },
      { id: "exam", num: "📝", name: "ห้องซ้อมข้อสอบ Final", file: "exam.html" },
      { id: "cheatsheet", num: "📋", name: "Cheatsheet ก่อนเข้าห้องสอบ", file: "cheatsheet.html" },
    ]},
    { group: "ช่วงต้นเทอม (ทบทวนเร็ว)", items: [
      { id: "ch01", num: "1", name: "รู้จัก OS, Boot, Dual Mode, VM/Container", file: "01-intro.html" },
      { id: "ch02", num: "2", name: "OS Structure & System Call", file: "02-structure.html" },
      { id: "ch03", num: "3", name: "Process & fork/exec/wait", file: "03-process.html" },
      { id: "ch04", num: "4", name: "Thread & Multithreading", file: "04-thread.html" },
      { id: "ch05", num: "5", name: "CPU Scheduling (คำนวณ!)", file: "05-scheduling.html" },
      { id: "ch06", num: "6", name: "IPC: Signal, Pipe, Shared Memory", file: "06-ipc.html" },
    ]},
    { group: "ช่วง Final (เน้นสุด ๆ)", final: true, items: [
      { id: "ch07", num: "7", name: "Race Condition & Critical Section", file: "07-sync.html" },
      { id: "ch08", num: "8", name: "Semaphore & Monitor", file: "08-semaphore.html" },
      { id: "ch09", num: "9", name: "Deadlock & Banker's Algorithm", file: "09-deadlock.html" },
      { id: "ch10", num: "10", name: "Real Memory Management", file: "10-realmem.html" },
      { id: "ch11", num: "11", name: "Paging & Address Translation (คำนวณ!)", file: "11-paging.html" },
      { id: "ch12", num: "12", name: "Virtual Memory & Page Replacement (คำนวณ!)", file: "12-virtualmem.html" },
      { id: "ch13", num: "13", name: "File System", file: "13-filesystem.html" },
    ]},
  ];

  /* ---------- Sidebar ---------- */
  const sbRoot = document.getElementById("sidebar-root");
  if (sbRoot) {
    const page = document.body.dataset.page || "index";
    let html = `
      <div class="sidebar-head">
        <span class="title">📚 OS Final 2569</span>
        <button class="icon-btn" id="sbClose" aria-label="ปิดเมนู">✕</button>
      </div>
      <input class="searchbox" id="sbSearch" type="text" placeholder="ค้นหาบทเรียน..." aria-label="ค้นหาบทเรียน">`;
    for (const g of CHAPTERS) {
      html += `<div class="side-group ${g.final ? "final" : ""}">${g.group}</div>`;
      for (const it of g.items) {
        const active = it.id === page ? " active" : "";
        const done = localStorage.getItem("os-done-" + it.id) === "1" ? " done" : "";
        html += `<a class="side-link${active}${done}" data-sbitem data-name="${it.name}" href="${it.file}">
          <span class="num">${it.num}</span><span>${it.name}</span></a>`;
      }
    }
    sbRoot.innerHTML = '<aside class="sidebar">' + html + "</aside>";
  }

  const sidebar = document.querySelector(".sidebar");
  const backdrop = document.querySelector(".sidebar-backdrop");
  const openSb = (v) => {
    if (!sidebar) return;
    sidebar.classList.toggle("open", v);
    backdrop && backdrop.classList.toggle("show", v);
    document.body.style.overflow = v && window.innerWidth < 1080 ? "hidden" : "";
  };
  document.getElementById("menuBtn")?.addEventListener("click", () => openSb(true));
  document.getElementById("sbClose")?.addEventListener("click", () => openSb(false));
  backdrop?.addEventListener("click", () => openSb(false));
  document.addEventListener("keydown", (e) => { if (e.key === "Escape") openSb(false); });

  // search filter
  document.getElementById("sbSearch")?.addEventListener("input", (e) => {
    const q = e.target.value.trim().toLowerCase();
    document.querySelectorAll("[data-sbitem]").forEach((a) => {
      a.style.display = !q || a.dataset.name.toLowerCase().includes(q) ? "" : "none";
    });
  });

  /* ---------- Theme ---------- */
  const themeBtn = document.getElementById("themeBtn");
  const applyTheme = (t) => {
    document.documentElement.dataset.theme = t;
    if (themeBtn) themeBtn.textContent = t === "dark" ? "☀️" : "🌙";
    localStorage.setItem("os-theme", t);
  };
  applyTheme(localStorage.getItem("os-theme") || (window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"));
  themeBtn?.addEventListener("click", () =>
    applyTheme(document.documentElement.dataset.theme === "dark" ? "light" : "dark"));

  /* ---------- To top ---------- */
  const toTop = document.getElementById("toTop");
  window.addEventListener("scroll", () => {
    if (toTop) toTop.classList.toggle("show", window.scrollY > 600);
  }, { passive: true });
  toTop?.addEventListener("click", () => window.scrollTo({ top: 0, behavior: "smooth" }));

  /* ---------- Quiz engine ---------- */
  const store = JSON.parse(localStorage.getItem("os-quiz") || "{}");

  function saveStore() { localStorage.setItem("os-quiz", JSON.stringify(store)); }
  function answeredCount() { return Object.keys(store).length; }
  function totalQuizzes() { return document.querySelectorAll("[data-q]").length; }
  function updatePill() {
    const pill = document.getElementById("quizPill");
    if (!pill) return;
    const total = totalQuizzes();
    let done = 0;
    document.querySelectorAll("[data-q]").forEach((q) => { if (store[q.id]) done++; });
    pill.textContent = total ? `ซ้อมแล้ว ${done}/${total} ข้อ (หน้านี้) · รวม ${answeredCount()} ข้อ` : "";
  }

  function getAnswer(q) {
    const type = q.dataset.type;
    if (type === "num") return q.querySelector("input")?.value.trim().replace(/\s+/g, "");
    if (type === "text") return q.querySelector("input")?.value.trim().toLowerCase();
    const sel = q.querySelector(".choice.sel");
    return sel ? sel.dataset.val : null;
  }

  function normalizeNum(s) {
    if (s === null || s === undefined) return null;
    s = s.replace(/,/g, "").replace(/(μs|us|kb|mb|bytes|byte|เพจ|page|บิต|bit|ครั้ง|frames|frame)/gi, "").trim();
    if (s === "") return null;
    const n = Number(s);
    return isNaN(n) ? s : n;
  }

  function checkQuiz(q) {
    const type = q.dataset.type;
    const id = q.id;
    const given = getAnswer(q);
    const verdict = q.querySelector(".q-verdict");
    const explain = q.querySelector(".q-explain");
    let correct = false, skipped = given === null || given === "";

    if (!skipped) {
      if (type === "num") {
        const want = q.dataset.answer.split(",").map((x) => normalizeNum(x));
        const g = normalizeNum(given);
        const tol = parseFloat(q.dataset.tolerance || "0");
        correct = want.some((w) => typeof w === "number" && typeof g === "number" && Math.abs(w - g) <= tol);
        if (!correct && typeof g === "string") correct = want.some((w) => String(w) === g);
      } else if (type === "text") {
        const want = q.dataset.answer.toLowerCase().split("|").map((s) => s.trim());
        correct = want.includes(String(given).toLowerCase());
      } else {
        correct = given === q.dataset.answer;
      }
    }

    // mark choices
    if (type !== "num" && type !== "text") {
      q.querySelectorAll(".choice").forEach((c) => {
        c.classList.remove("correct", "wrong");
        if (c.dataset.val === q.dataset.answer) c.classList.add("correct");
        else if (c.classList.contains("sel")) c.classList.add("wrong");
      });
    } else {
      const inp = q.querySelector("input");
      if (inp) {
        inp.style.borderColor = skipped ? "var(--amber)" : correct ? "var(--green)" : "var(--red)";
        inp.style.background = skipped ? "var(--amber-soft)" : correct ? "var(--green-soft)" : "var(--red-soft)";
      }
    }

    const isTF = type === "tf";
    verdict.style.display = "block";
    if (skipped) {
      verdict.className = "q-verdict skip";
      verdict.textContent = "⏭️ ไม่ตอบ → ได้ 0 คะแนน" + (isTF ? " (ข้อสอบจริง: ไม่ตอบได้ 0)" : "");
    } else if (correct) {
      verdict.className = "q-verdict ok";
      verdict.textContent = "✅ ถูกต้อง!" + (isTF ? " ได้ +2 คะแนน" : "");
    } else {
      verdict.className = "q-verdict no";
      verdict.textContent = "❌ ยังไม่ถูก" + (isTF ? " — ถ้าเป็นห้องสอบจริงโดน −1! (ไม่แน่ใจควรข้าม ได้ 0 ปลอดภัยกว่า)" : "");
    }
    if (explain) explain.style.display = "block";
    q.querySelector(".act-check").disabled = true;
    const resetBtn = q.querySelector(".act-reset");
    if (resetBtn) resetBtn.style.display = "";

    store[id] = { ok: correct, skip: skipped };
    saveStore();
    updatePill();
    document.dispatchEvent(new CustomEvent("quizchecked", { detail: { id, correct, skipped } }));
  }

  function resetQuiz(q) {
    delete store[q.id];
    saveStore();
    q.querySelectorAll(".choice").forEach((c) => c.classList.remove("sel", "correct", "wrong"));
    const inp = q.querySelector("input");
    if (inp) { inp.value = ""; inp.style.background = ""; inp.style.borderColor = ""; }
    q.querySelector(".q-verdict").style.display = "none";
    const ex = q.querySelector(".q-explain");
    if (ex) ex.style.display = "none";
    q.querySelector(".act-check").disabled = false;
    const rb = q.querySelector(".act-reset");
    if (rb) rb.style.display = "none";
    updatePill();
  }

  function initQuiz(q) {
    if (!q.id) {
      let seq = parseInt(localStorage.getItem("os-quiz-seq") || "0", 10) + 1;
      localStorage.setItem("os-quiz-seq", String(seq));
      q.id = "q-" + (document.body.dataset.page || "p") + "-" + seq;
    }
    const type = q.dataset.type;
    q.querySelectorAll(".choice").forEach((c) => {
      c.addEventListener("click", () => {
        if (q.querySelector(".act-check").disabled) return;
        q.querySelectorAll(".choice").forEach((x) => x.classList.remove("sel"));
        c.classList.add("sel");
      });
    });
    q.querySelector(".act-check")?.addEventListener("click", () => checkQuiz(q));
    q.querySelector(".act-reset")?.addEventListener("click", () => resetQuiz(q));
    q.querySelector("input")?.addEventListener("keydown", (e) => {
      if (e.key === "Enter") { e.preventDefault(); if (!q.querySelector(".act-check").disabled) checkQuiz(q); }
    });
    // restore answered state: just show that it was done (do not auto-reveal answers)
    if (store[q.id]) {
      const rb = q.querySelector(".act-reset");
      if (rb) rb.style.display = "";
      rb && (rb.textContent = "ลองอีกครั้ง");
    }
  }

  document.querySelectorAll("[data-q]").forEach(initQuiz);
  updatePill();

  /* ---------- Mark chapter done ---------- */
  const doneBtn = document.getElementById("doneBtn");
  if (doneBtn) {
    const pid = document.body.dataset.page;
    const refresh = () => {
      const done = localStorage.getItem("os-done-" + pid) === "1";
      doneBtn.textContent = done ? "✅ ทำบทนี้แล้ว (กดเพื่อยกเลิก)" : "☑️ ทำเนื้อหาบทนี้แล้ว — ทำเครื่องหมาย";
    };
    refresh();
    doneBtn.addEventListener("click", () => {
      const pid2 = document.body.dataset.page;
      const done = localStorage.getItem("os-done-" + pid2) === "1";
      localStorage.setItem("os-done-" + pid2, done ? "0" : "1");
      refresh();
      document.querySelectorAll(".side-link.active").forEach((a) => a.classList.toggle("done", !done));
    });
  }
})();
