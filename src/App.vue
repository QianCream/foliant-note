<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick } from "vue";
import { marked } from "marked";
import { markedHighlight } from "marked-highlight";
import hljs from "highlight.js/lib/common";
import "highlight.js/styles/github-dark.css";
import { invoke } from "@tauri-apps/api/core";
import { open, save } from "@tauri-apps/plugin-dialog";
import katex from "katex";
import "katex/dist/katex.min.css";

type Mode = "diary" | "article" | "note";
type ViewMode = "edit" | "preview" | "split";
type Theme = "light" | "dark";

type Note = {
  id: string;
  title: string;
  content: string;
  tags: string[];
  pinned?: boolean;
  createdAt: number;
  updatedAt: number;
};

type DiaryEntry = {
  id: string;
  date: string;
  mood: string;
  content: string;
  createdAt: number;
  updatedAt: number;
};

type Chapter = {
  id: string;
  title: string;
  content: string;
};

type Article = {
  id: string;
  title: string;
  chapters: Chapter[];
  createdAt: number;
  updatedAt: number;
  pinned?: boolean;
};

type TrashItem = {
  id: string;
  type: "diary" | "article" | "chapter" | "note";
  data: DiaryEntry | Article | Chapter | Note;
  parentId?: string;
  deletedAt: number;
};

// Legacy localStorage keys — used only for one-time migration
const DIARY_KEY = "foliant.diary.v2";
const ARTICLES_KEY = "foliant.articles.v2";
const MODE_KEY = "foliant.mode.v2";
const VIEW_KEY = "foliant.view.v2";
const THEME_KEY = "foliant.theme.v1";
const FOCUS_KEY = "foliant.focus.v1";
const TOC_KEY = "foliant.toc.v1";
const GOAL_KEY = "foliant.goal.v1";
const TRASH_KEY = "foliant.trash.v1";

type AppData = {
  diaries: DiaryEntry[];
  articles: Article[];
  notes: Note[];
  trash: TrashItem[];
  mode: Mode;
  view: ViewMode;
  theme: Theme;
  focusMode: boolean;
  tocOpen: boolean;
  goal: number;
};

let saveTimer: ReturnType<typeof setTimeout> | null = null;
function clearPendingSave() {
  if (!saveTimer) return;
  clearTimeout(saveTimer);
  saveTimer = null;
}

function buildPayload(): AppData {
  return {
    diaries: diaries.value,
    articles: articles.value,
    notes: notes.value,
    trash: trash.value,
    mode: mode.value,
    view: view.value,
    theme: theme.value,
    focusMode: focusMode.value,
    tocOpen: tocOpen.value,
    goal: goal.value,
  };
}

function scheduleSave() {
  clearPendingSave();
  saveState.value = "saving";
  saveTimer = setTimeout(persistData, 1200);
}

async function persistData() {
  saveTimer = null;
  const payload = buildPayload();
  try {
    await invoke("save_data", { data: JSON.stringify(payload) });
    saveState.value = "saved";
    if (savedFlashTimer) clearTimeout(savedFlashTimer);
    savedFlashTimer = setTimeout(() => { saveState.value = "idle"; }, 1600);
  } catch (e) {
    console.error(e);
    saveState.value = "idle";
  }
}

marked.use(
  markedHighlight({
    langPrefix: "hljs language-",
    highlight(code, lang) {
      const language = hljs.getLanguage(lang) ? lang : "plaintext";
      try {
        return hljs.highlight(code, { language }).value;
      } catch {
        return code;
      }
    },
  }),
);
marked.setOptions({ breaks: true, gfm: true });

// LaTeX math: extract first (so marked won't break the contents with <br>),
// then re-inject rendered HTML after marked is done.
const MATH_TOKEN = "\u0000MATH\u0001";
function renderMarkdown(src: string): string {
  const slots: string[] = [];
  // Block math $$ ... $$  (kept on its own line so marked treats it as a paragraph)
  let s = src.replace(/\$\$([\s\S]+?)\$\$/g, (_m, expr) => {
    try {
      const html = katex.renderToString(expr.trim(), { displayMode: true, throwOnError: false });
      slots.push(`<div class="math-block">${html}</div>`);
    } catch {
      slots.push(_m);
    }
    return `\n\n${MATH_TOKEN}${slots.length - 1}${MATH_TOKEN}\n\n`;
  });
  // Inline math $ ... $ — exclude $5 (digit-adjacent) and \$ (escaped)
  s = s.replace(/(^|[^\\$\d])\$([^\n$]+?)\$(?!\d)/g, (_m, pre, expr) => {
    try {
      const html = katex.renderToString(expr.trim(), { displayMode: false, throwOnError: false });
      slots.push(html);
    } catch {
      slots.push(`$${expr}$`);
    }
    return `${pre}${MATH_TOKEN}${slots.length - 1}${MATH_TOKEN}`;
  });

  let html = marked.parse(s) as string;

  // Re-inject KaTeX HTML at every token site (handles tokens that survived intact
  // and tokens that got entity-escaped by marked, e.g. \u0000 → \uFFFD or numeric refs)
  html = html.replace(new RegExp(`${MATH_TOKEN}(\\d+)${MATH_TOKEN}`, "g"), (_m, i) => slots[+i] ?? "");
  return html;
}

const isMac = /Mac|iPhone|iPod|iPad/i.test(navigator.platform);
const logoUrl = "/logo.png";

const mode = ref<Mode>("diary");
const view = ref<ViewMode>("split");
const search = ref("");
const theme = ref<Theme>("light");
const focusMode = ref(false);
const tocOpen = ref(false);
const trashOpen = ref(false);
const goal = ref(200);
const paletteOpen = ref(false);
const paletteQuery = ref("");
const paletteIndex = ref(0);
const saveState = ref<"idle" | "saving" | "saved">("idle");
let savedFlashTimer: ReturnType<typeof setTimeout> | null = null;

// In-app confirm + toast (Tauri 2 webviews don't support window.confirm/alert)
const confirmState = ref<{ title: string; body?: string; danger?: boolean; resolve?: (ok: boolean) => void } | null>(null);
function askConfirm(title: string, body?: string, danger = false): Promise<boolean> {
  return new Promise((resolve) => {
    confirmState.value = { title, body, danger, resolve };
  });
}
function answerConfirm(ok: boolean) {
  confirmState.value?.resolve?.(ok);
  confirmState.value = null;
}

const toast = ref<string>("");
let toastTimer: ReturnType<typeof setTimeout> | null = null;
function showToast(msg: string) {
  toast.value = msg;
  if (toastTimer) clearTimeout(toastTimer);
  toastTimer = setTimeout(() => { toast.value = ""; }, 2200);
}

const diaries = ref<DiaryEntry[]>([]);
const currentDiaryId = ref<string | null>(null);

const articles = ref<Article[]>([]);
const currentArticleId = ref<string | null>(null);
const currentChapterId = ref<string | null>(null);
const expanded = ref<Record<string, boolean>>({});

const notes = ref<Note[]>([]);
const currentNoteId = ref<string | null>(null);
const noteTagFilter = ref<string>("");
const noteTagInput = ref("");

const trash = ref<TrashItem[]>([]);

const diaryEditor = ref<HTMLTextAreaElement | null>(null);
const chapterEditor = ref<HTMLTextAreaElement | null>(null);
const noteEditor = ref<HTMLTextAreaElement | null>(null);
const fileInput = ref<HTMLInputElement | null>(null);

const moods = ["☀️", "🌧️", "🌙", "🔥", "🌱", "☕️", "💭", "🫧"];

function isoDate(d: Date): string {
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${y}-${m}-${day}`;
}

const todayStr = computed(() => isoDate(new Date()));

function resolveDialogPath(path: string | string[] | null): string | null {
  if (!path) return null;
  return Array.isArray(path) ? path[0] ?? null : path;
}

function sanitizeFilenamePart(value: string, fallback: string): string {
  const cleaned = value
    .replace(/[<>:"/\\|?*\u0000-\u001f]/g, " ")
    .replace(/\s+/g, " ")
    .trim()
    .replace(/[. ]+$/g, "")
    .slice(0, 80);
  const reserved = /^(con|prn|aux|nul|com[1-9]|lpt[1-9])$/i;
  const safe = cleaned || fallback;
  return reserved.test(safe) ? `${fallback}-${safe}` : safe;
}

function ensureExtension(path: string, ext: string): string {
  return path.toLowerCase().endsWith(ext.toLowerCase()) ? path : `${path}${ext}`;
}

const diariesSorted = computed(() =>
  [...diaries.value].sort((a, b) => b.date.localeCompare(a.date)),
);

const filteredDiaries = computed(() => {
  const q = search.value.trim().toLowerCase();
  if (!q) return diariesSorted.value;
  return diariesSorted.value.filter(
    (d) => d.date.includes(q) || d.content.toLowerCase().includes(q),
  );
});

const diariesByMonth = computed(() => {
  const groups = new Map<string, DiaryEntry[]>();
  for (const d of filteredDiaries.value) {
    const m = d.date.slice(0, 7);
    if (!groups.has(m)) groups.set(m, []);
    groups.get(m)!.push(d);
  }
  return Array.from(groups.entries());
});

function sortArticles(items: Article[]): Article[] {
  return [...items].sort((a, b) => {
    if (!!b.pinned !== !!a.pinned) return (b.pinned ? 1 : 0) - (a.pinned ? 1 : 0);
    return b.updatedAt - a.updatedAt;
  });
}

const filteredArticles = computed(() => {
  const q = search.value.trim().toLowerCase();
  const arr = sortArticles(articles.value);
  if (!q) return arr;
  return arr.filter(
    (a) =>
      a.title.toLowerCase().includes(q) ||
      a.chapters.some(
        (c) =>
          c.title.toLowerCase().includes(q) ||
          c.content.toLowerCase().includes(q),
      ),
  );
});

function togglePinArticle(id: string) {
  const a = articles.value.find((x) => x.id === id);
  if (a) a.pinned = !a.pinned;
}

// 随笔 — flat note list, lightweight, optimized for quick capture
const allNoteTags = computed(() => {
  const set = new Set<string>();
  for (const n of notes.value) for (const t of n.tags) set.add(t);
  return Array.from(set).sort();
});

function sortNotes(items: Note[]): Note[] {
  return [...items].sort((a, b) => {
    if (!!b.pinned !== !!a.pinned) return (b.pinned ? 1 : 0) - (a.pinned ? 1 : 0);
    return b.updatedAt - a.updatedAt;
  });
}

const filteredNotes = computed(() => {
  const q = search.value.trim().toLowerCase();
  const tag = noteTagFilter.value;
  let arr = sortNotes(notes.value);
  if (tag) arr = arr.filter((n) => n.tags.includes(tag));
  if (q) arr = arr.filter((n) => n.title.toLowerCase().includes(q) || n.content.toLowerCase().includes(q));
  return arr;
});

const currentNote = computed(() =>
  notes.value.find((n) => n.id === currentNoteId.value) ?? null,
);
const noteSuggestedTags = computed(() =>
  currentNote.value
    ? allNoteTags.value.filter((tag) => !currentNote.value!.tags.includes(tag)).slice(0, 6)
    : [],
);

const noteHtml = ref("");

function noteSnippet(n: Note): string {
  const body = n.content.replace(/^#+\s.*$/gm, "").replace(/[#*`>_~\[\]()]/g, "").trim();
  return body.split("\n").filter(Boolean).slice(0, 2).join(" · ").slice(0, 80);
}

function noteAutoTitle(n: Note): string {
  if (n.title) return n.title;
  const firstLine = n.content.split("\n").find((l) => l.trim());
  if (!firstLine) return "未命名";
  return firstLine.replace(/^#+\s*/, "").slice(0, 40) || "未命名";
}

function createNote() {
  const id = crypto.randomUUID();
  notes.value.unshift({
    id,
    title: "",
    content: "",
    tags: [],
    createdAt: Date.now(),
    updatedAt: Date.now(),
  });
  currentNoteId.value = id;
  nextTick(() => noteEditor.value?.focus());
}

function updateNoteContent(v: string) {
  if (!currentNote.value) return;
  currentNote.value.content = v;
  currentNote.value.updatedAt = Date.now();
}

function updateNoteTitle(v: string) {
  if (!currentNote.value) return;
  currentNote.value.title = v;
  currentNote.value.updatedAt = Date.now();
}

function deleteNote(id: string) {
  const i = notes.value.findIndex((n) => n.id === id);
  if (i === -1) return;
  trash.value.unshift({
    id: crypto.randomUUID(),
    type: "note",
    data: { ...notes.value[i] },
    deletedAt: Date.now(),
  });
  notes.value.splice(i, 1);
  if (currentNoteId.value === id) currentNoteId.value = filteredNotes.value[0]?.id ?? null;
}

function togglePinNote(id: string) {
  const n = notes.value.find((x) => x.id === id);
  if (n) n.pinned = !n.pinned;
}

function addNoteTag(tag: string) {
  if (!currentNote.value) return;
  const t = tag.trim().replace(/^#/, "").trim();
  if (!t) return;
  if (!currentNote.value.tags.includes(t)) {
    currentNote.value.tags.push(t);
    currentNote.value.updatedAt = Date.now();
  }
}

function removeNoteTag(tag: string) {
  if (!currentNote.value) return;
  currentNote.value.tags = currentNote.value.tags.filter((t) => t !== tag);
  currentNote.value.updatedAt = Date.now();
}

function submitNoteTag() {
  if (!noteTagInput.value.trim()) return;
  addNoteTag(noteTagInput.value);
  noteTagInput.value = "";
}

type PaletteItem = {
  id: string;
  kind: "diary" | "article" | "note" | "action";
  label: string;
  hint?: string;
  run: () => void;
};

const paletteItems = computed<PaletteItem[]>(() => {
  const items: PaletteItem[] = [];
  // actions
  items.push(
    { id: "act-diary", kind: "action", label: "切换到日记", hint: "Mode", run: () => { mode.value = "diary"; } },
    { id: "act-article", kind: "action", label: "切换到长文", hint: "Mode", run: () => { mode.value = "article"; } },
    { id: "act-note", kind: "action", label: "切换到随笔", hint: "Mode", run: () => { mode.value = "note"; } },
    { id: "act-theme", kind: "action", label: theme.value === "dark" ? "切换到浅色主题" : "切换到深色主题", hint: "Theme", run: () => { theme.value = theme.value === "dark" ? "light" : "dark"; } },
    { id: "act-focus", kind: "action", label: focusMode.value ? "退出专注模式" : "进入专注模式", hint: "View", run: () => { focusMode.value = !focusMode.value; } },
    { id: "act-toc", kind: "action", label: tocOpen.value ? "关闭大纲" : "显示大纲", hint: "View", run: () => { trashOpen.value = false; tocOpen.value = !tocOpen.value; } },
    { id: "act-checkin", kind: "action", label: "今日打卡", hint: "Diary", run: () => { mode.value = "diary"; checkIn(); } },
    { id: "act-new-article", kind: "action", label: "新建文章", hint: "Article", run: () => { mode.value = "article"; createArticle(); } },
    { id: "act-new-note", kind: "action", label: "新建随笔", hint: "Note · ⌘N", run: () => { mode.value = "note"; createNote(); } },
    { id: "act-export-current", kind: "action", label: "导出当前 Markdown", hint: "Export", run: () => { void exportCurrent(); } },
    { id: "act-export-backup", kind: "action", label: "备份全部数据", hint: "Backup", run: () => { void exportBackup(); } },
    { id: "act-import-backup", kind: "action", label: "恢复数据备份", hint: "Backup", run: () => { void importBackup(); } },
  );
  for (const d of diariesSorted.value) {
    const preview = d.content.replace(/\s+/g, " ").slice(0, 36);
    items.push({
      id: `d-${d.id}`,
      kind: "diary",
      label: `${d.date} ${d.mood}`,
      hint: preview || "（空）",
      run: () => { mode.value = "diary"; currentDiaryId.value = d.id; },
    });
  }
  for (const a of articles.value) {
    items.push({
      id: `a-${a.id}`,
      kind: "article",
      label: a.title || "未命名文章",
      hint: `${a.chapters.length} 章`,
      run: () => { mode.value = "article"; currentArticleId.value = a.id; currentChapterId.value = a.chapters[0]?.id ?? null; expanded.value[a.id] = true; },
    });
  }
  for (const n of notes.value) {
    const preview = n.content.replace(/\s+/g, " ").slice(0, 36);
    items.push({
      id: `n-${n.id}`,
      kind: "note",
      label: noteAutoTitle(n),
      hint: n.tags.length ? n.tags.map((t) => "#" + t).join(" ") : (preview || "（空）"),
      run: () => { mode.value = "note"; currentNoteId.value = n.id; },
    });
  }
  return items;
});

const paletteResults = computed(() => {
  const q = paletteQuery.value.trim().toLowerCase();
  if (!q) return paletteItems.value.slice(0, 30);
  return paletteItems.value
    .filter((it) => it.label.toLowerCase().includes(q) || (it.hint?.toLowerCase().includes(q)))
    .slice(0, 30);
});

function openPalette() {
  paletteQuery.value = "";
  paletteIndex.value = 0;
  paletteOpen.value = true;
  nextTick(() => {
    const el = document.getElementById("palette-input") as HTMLInputElement | null;
    el?.focus();
  });
}

function runPaletteItem(it: PaletteItem) {
  it.run();
  paletteOpen.value = false;
}

function onPaletteKey(e: KeyboardEvent) {
  if (e.key === "ArrowDown") { e.preventDefault(); paletteIndex.value = Math.min(paletteResults.value.length - 1, paletteIndex.value + 1); }
  else if (e.key === "ArrowUp") { e.preventDefault(); paletteIndex.value = Math.max(0, paletteIndex.value - 1); }
  else if (e.key === "Enter") { e.preventDefault(); const it = paletteResults.value[paletteIndex.value]; if (it) runPaletteItem(it); }
  else if (e.key === "Escape") { paletteOpen.value = false; }
}

watch(paletteQuery, () => { paletteIndex.value = 0; });

const checkedInToday = computed(() =>
  diaries.value.some((d) => d.date === todayStr.value),
);

const streak = computed(() => {
  const dates = new Set(diaries.value.map((d) => d.date));
  const cur = new Date();
  if (!dates.has(isoDate(cur))) cur.setDate(cur.getDate() - 1);
  let count = 0;
  while (dates.has(isoDate(cur))) {
    count++;
    cur.setDate(cur.getDate() - 1);
  }
  return count;
});

const totalDays = computed(() => new Set(diaries.value.map((d) => d.date)).size);

const currentDiary = computed(
  () => diaries.value.find((d) => d.id === currentDiaryId.value) ?? null,
);

const currentArticle = computed(
  () => articles.value.find((a) => a.id === currentArticleId.value) ?? null,
);

const currentChapter = computed(() => {
  if (!currentArticle.value) return null;
  return (
    currentArticle.value.chapters.find((c) => c.id === currentChapterId.value) ??
    null
  );
});

function wordCount(s: string): number {
  const cn = (s.match(/[\u4e00-\u9fa5]/g) || []).length;
  const en = (s.match(/[a-zA-Z]+/g) || []).length;
  return cn + en;
}

const chapterWords = computed(() =>
  currentChapter.value ? wordCount(currentChapter.value.content) : 0,
);
const articleWords = computed(() =>
  currentArticle.value
    ? currentArticle.value.chapters.reduce(
        (s, c) => s + wordCount(c.content),
        0,
      )
    : 0,
);

// Debounced preview — rendering marked + KaTeX + hljs on every keystroke is heavy,
// so we recompute 180ms after the last edit. Switching notes/mode renders immediately.
const diaryHtml = ref("");
const chapterHtml = ref("");
let renderTimer: ReturnType<typeof setTimeout> | null = null;
function refreshPreview(immediate = false) {
  if (renderTimer) { clearTimeout(renderTimer); renderTimer = null; }
  const run = () => {
    if (view.value === "edit") return; // skip work when preview isn't visible
    diaryHtml.value = currentDiary.value ? renderMarkdown(currentDiary.value.content) : "";
    chapterHtml.value = currentChapter.value ? renderMarkdown(currentChapter.value.content) : "";
    noteHtml.value = currentNote.value ? renderMarkdown(currentNote.value.content) : "";
  };
  if (immediate) run();
  else renderTimer = setTimeout(run, 180);
}
watch(
  () => [currentDiary.value?.content, currentChapter.value?.content, currentNote.value?.content],
  () => refreshPreview(false),
);
watch(
  [currentDiaryId, currentChapterId, currentNoteId, mode, view],
  () => refreshPreview(true),
);

const todayWords = computed(() => {
  const t = diaries.value.find((d) => d.date === todayStr.value);
  return t ? wordCount(t.content) : 0;
});

const todayProgress = computed(() =>
  Math.min(100, Math.round((todayWords.value / Math.max(1, goal.value)) * 100)),
);

const tocSource = computed(() => {
  if (mode.value === "diary") return currentDiary.value?.content ?? "";
  if (mode.value === "note") return currentNote.value?.content ?? "";
  return currentChapter.value?.content ?? "";
});

const toc = computed(() => {
  const lines = tocSource.value.split("\n");
  const items: { level: number; text: string; line: number }[] = [];
  let inCode = false;
  for (let i = 0; i < lines.length; i++) {
    if (lines[i].startsWith("```")) inCode = !inCode;
    if (inCode) continue;
    const m = lines[i].match(/^(#{1,4})\s+(.+)/);
    if (m) items.push({ level: m[1].length, text: m[2].trim(), line: i });
  }
  return items;
});

const heatmap = computed(() => {
  const byDate = new Map<string, DiaryEntry>();
  for (const d of diaries.value) byDate.set(d.date, d);
  const cells: { date: string; level: number; today: boolean }[] = [];
  const today = new Date();
  for (let i = 90; i >= 0; i--) {
    const d = new Date(today);
    d.setDate(today.getDate() - i);
    const key = isoDate(d);
    const entry = byDate.get(key);
    const level = entry ? Math.min(4, Math.ceil(wordCount(entry.content) / 80) || 1) : 0;
    cells.push({ date: key, level, today: key === todayStr.value });
  }
  return cells;
});

function applyLoadedData(saved: Partial<AppData>) {
  clearPendingSave();
  const cutoff = Date.now() - 30 * 24 * 60 * 60 * 1000;

  diaries.value = Array.isArray(saved.diaries) ? saved.diaries : [];
  articles.value = Array.isArray(saved.articles) ? saved.articles : [];
  notes.value = Array.isArray(saved.notes) ? saved.notes : [];
  trash.value = Array.isArray(saved.trash)
    ? saved.trash.filter((x) => x.deletedAt > cutoff)
    : [];

  mode.value = saved.mode === "diary" || saved.mode === "article" || saved.mode === "note"
    ? saved.mode
    : "diary";
  view.value = saved.view === "edit" || saved.view === "preview" || saved.view === "split"
    ? saved.view
    : "split";
  theme.value = saved.theme === "dark" ? "dark" : "light";
  focusMode.value = saved.focusMode ?? false;
  tocOpen.value = saved.tocOpen ?? false;
  goal.value = typeof saved.goal === "number" ? Math.max(1, saved.goal) : 200;

  search.value = "";
  noteTagFilter.value = "";
  noteTagInput.value = "";
  trashOpen.value = false;

  expanded.value = {};
  currentDiaryId.value = diariesSorted.value[0]?.id ?? null;

  const firstArticle = sortArticles(articles.value)[0];
  currentArticleId.value = firstArticle?.id ?? null;
  currentChapterId.value = firstArticle?.chapters[0]?.id ?? null;
  if (firstArticle) expanded.value[firstArticle.id] = true;

  currentNoteId.value = sortNotes(notes.value)[0]?.id ?? null;
  applyTheme();
  refreshPreview(true);
}

async function load() {
  // Try loading from Rust/filesystem first
  let loaded = false;
  try {
    const raw = await invoke<string>("load_data");
    if (raw) {
      applyLoadedData(JSON.parse(raw) as Partial<AppData>);
      loaded = true;
    }
  } catch {
    // Running in browser/dev without Tauri — fall through to localStorage migration
  }

  if (!loaded) {
    // One-time migration from localStorage (first launch after update)
    try {
      const migrated: Partial<AppData> = {};
      const d = localStorage.getItem(DIARY_KEY);
      if (d) migrated.diaries = JSON.parse(d);
      const a = localStorage.getItem(ARTICLES_KEY);
      if (a) migrated.articles = JSON.parse(a);
      const m = localStorage.getItem(MODE_KEY);
      if (m === "diary" || m === "article") migrated.mode = m;
      const v = localStorage.getItem(VIEW_KEY);
      if (v === "edit" || v === "preview" || v === "split") migrated.view = v;
      const t = localStorage.getItem(THEME_KEY);
      if (t === "light" || t === "dark") migrated.theme = t;
      const f = localStorage.getItem(FOCUS_KEY);
      if (f === "1") migrated.focusMode = true;
      const tk = localStorage.getItem(TOC_KEY);
      if (tk === "1") migrated.tocOpen = true;
      const g = localStorage.getItem(GOAL_KEY);
      if (g) migrated.goal = Math.max(1, parseInt(g, 10) || 200);
      const tr = localStorage.getItem(TRASH_KEY);
      if (tr) migrated.trash = JSON.parse(tr) as TrashItem[];
      applyLoadedData(migrated);
      loaded = true;
      // Persist migrated data to disk and clear localStorage
      persistData();
      [DIARY_KEY, ARTICLES_KEY, MODE_KEY, VIEW_KEY, THEME_KEY, FOCUS_KEY, TOC_KEY, GOAL_KEY, TRASH_KEY]
        .forEach((k) => localStorage.removeItem(k));
    } catch {}
  }

  if (!loaded) {
    applyTheme();
    refreshPreview(true);
  }
}

function applyTheme() {
  document.documentElement.setAttribute("data-theme", theme.value);
}

document.documentElement.setAttribute("data-platform", isMac ? "mac" : "other");

watch(diaries, scheduleSave, { deep: true });
watch(articles, scheduleSave, { deep: true });
watch(notes, scheduleSave, { deep: true });
watch(trash, scheduleSave, { deep: true });
watch(mode, scheduleSave);
watch(view, scheduleSave);
watch(theme, () => { applyTheme(); scheduleSave(); });
watch(focusMode, scheduleSave);
watch(tocOpen, scheduleSave);
watch(goal, scheduleSave);

function checkIn() {
  const existing = diaries.value.find((d) => d.date === todayStr.value);
  if (existing) {
    currentDiaryId.value = existing.id;
    nextTick(() => diaryEditor.value?.focus());
    return;
  }
  const id = crypto.randomUUID();
  diaries.value.push({
    id,
    date: todayStr.value,
    mood: "",
    content: "",
    createdAt: Date.now(),
    updatedAt: Date.now(),
  });
  currentDiaryId.value = id;
  nextTick(() => diaryEditor.value?.focus());
}

function deleteDiary(id: string) {
  const i = diaries.value.findIndex((d) => d.id === id);
  if (i === -1) return;
  trash.value.unshift({
    id: crypto.randomUUID(),
    type: "diary",
    data: { ...diaries.value[i] },
    deletedAt: Date.now(),
  });
  diaries.value.splice(i, 1);
  if (currentDiaryId.value === id) currentDiaryId.value = diariesSorted.value[0]?.id ?? null;
}

function updateDiary(value: string) {
  if (!currentDiary.value) return;
  currentDiary.value.content = value;
  currentDiary.value.updatedAt = Date.now();
}

function setMood(m: string) {
  if (!currentDiary.value) return;
  currentDiary.value.mood = currentDiary.value.mood === m ? "" : m;
  currentDiary.value.updatedAt = Date.now();
}

function createArticle() {
  const id = crypto.randomUUID();
  const cid = crypto.randomUUID();
  articles.value.unshift({
    id,
    title: "未命名文章",
    chapters: [{ id: cid, title: "第 1 章", content: "" }],
    createdAt: Date.now(),
    updatedAt: Date.now(),
  });
  currentArticleId.value = id;
  currentChapterId.value = cid;
  expanded.value[id] = true;
  nextTick(() => chapterEditor.value?.focus());
}

function addChapter(articleId: string) {
  const a = articles.value.find((x) => x.id === articleId);
  if (!a) return;
  const id = crypto.randomUUID();
  a.chapters.push({ id, title: `第 ${a.chapters.length + 1} 章`, content: "" });
  a.updatedAt = Date.now();
  currentArticleId.value = articleId;
  currentChapterId.value = id;
  expanded.value[articleId] = true;
  nextTick(() => chapterEditor.value?.focus());
}

function deleteArticle(id: string) {
  const i = articles.value.findIndex((a) => a.id === id);
  if (i === -1) return;
  trash.value.unshift({
    id: crypto.randomUUID(),
    type: "article",
    data: JSON.parse(JSON.stringify(articles.value[i])),
    deletedAt: Date.now(),
  });
  articles.value.splice(i, 1);
  if (currentArticleId.value === id) {
    const next = filteredArticles.value[0];
    currentArticleId.value = next?.id ?? null;
    currentChapterId.value = next?.chapters[0]?.id ?? null;
  }
}

function deleteChapter(articleId: string, chapterId: string) {
  const a = articles.value.find((x) => x.id === articleId);
  if (!a) return;
  if (a.chapters.length === 1) {
    showToast("至少保留一个章节");
    return;
  }
  const i = a.chapters.findIndex((c) => c.id === chapterId);
  if (i === -1) return;
  trash.value.unshift({
    id: crypto.randomUUID(),
    type: "chapter",
    data: { ...a.chapters[i] },
    parentId: articleId,
    deletedAt: Date.now(),
  });
  a.chapters.splice(i, 1);
  a.updatedAt = Date.now();
  if (currentChapterId.value === chapterId) {
    currentChapterId.value = a.chapters[Math.max(0, i - 1)].id;
  }
}

function restoreTrash(itemId: string) {
  const idx = trash.value.findIndex((t) => t.id === itemId);
  if (idx === -1) return;
  const item = trash.value[idx];
  if (item.type === "diary") {
    diaries.value.push(item.data as DiaryEntry);
    currentDiaryId.value = (item.data as DiaryEntry).id;
  } else if (item.type === "article") {
    articles.value.unshift(item.data as Article);
    currentArticleId.value = (item.data as Article).id;
    currentChapterId.value = (item.data as Article).chapters[0]?.id ?? null;
  } else if (item.type === "note") {
    notes.value.unshift(item.data as Note);
    currentNoteId.value = (item.data as Note).id;
  } else if (item.type === "chapter") {
    const parent = articles.value.find((a) => a.id === item.parentId);
    if (!parent) {
      showToast("原文章已不存在，无法恢复章节");
      return;
    }
    parent.chapters.push(item.data as Chapter);
    parent.updatedAt = Date.now();
  }
  trash.value.splice(idx, 1);
}

function purgeTrash(itemId: string) {
  const i = trash.value.findIndex((t) => t.id === itemId);
  if (i !== -1) trash.value.splice(i, 1);
}

async function clearTrash() {
  if (!trash.value.length) return;
  const ok = await askConfirm("永久清空回收站？", "此操作不可撤销，所有项目将被彻底删除。", true);
  if (!ok) return;
  trash.value = [];
  showToast("回收站已清空");
}

function trashTitle(t: TrashItem): string {
  if (t.type === "diary") return (t.data as DiaryEntry).date;
  if (t.type === "note") return noteAutoTitle(t.data as Note);
  return (t.data as Article | Chapter).title;
}

function importMd() {
  fileInput.value?.click();
}

function onImportFile(e: Event) {
  const input = e.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = () => {
    const text = String(reader.result || "");
    const baseName = file.name.replace(/\.(md|markdown|txt)$/i, "");
    if (mode.value === "diary") {
      const id = crypto.randomUUID();
      diaries.value.push({
        id,
        date: todayStr.value,
        mood: "",
        content: text,
        createdAt: Date.now(),
        updatedAt: Date.now(),
      });
      currentDiaryId.value = id;
    } else if (mode.value === "note") {
      const id = crypto.randomUUID();
      const titleMatch = text.match(/^#\s+(.+)$/m);
      const content = titleMatch ? text.replace(/^#\s+.+\n?/, "").trim() : text;
      notes.value.unshift({
        id,
        title: titleMatch?.[1]?.trim() || baseName,
        content,
        tags: [],
        createdAt: Date.now(),
        updatedAt: Date.now(),
      });
      currentNoteId.value = id;
    } else {
      const id = crypto.randomUUID();
      const titleMatch = text.match(/^#\s+(.+)$/m);
      const title = titleMatch?.[1]?.trim() || baseName;
      const sections = text.split(/^##\s+/m);
      const chapters: Chapter[] = [];
      if (sections.length > 1) {
        const intro = sections.shift()!.replace(/^#\s+.+\n?/, "").trim();
        if (intro) {
          chapters.push({ id: crypto.randomUUID(), title: "前言", content: intro });
        }
        for (const sec of sections) {
          const nl = sec.indexOf("\n");
          const head = nl === -1 ? sec : sec.slice(0, nl);
          const body = nl === -1 ? "" : sec.slice(nl + 1).trim();
          chapters.push({
            id: crypto.randomUUID(),
            title: head.trim() || "章节",
            content: body,
          });
        }
      } else {
        chapters.push({ id: crypto.randomUUID(), title: "第 1 章", content: text });
      }
      articles.value.unshift({
        id,
        title,
        chapters,
        createdAt: Date.now(),
        updatedAt: Date.now(),
      });
      currentArticleId.value = id;
      currentChapterId.value = chapters[0].id;
      expanded.value[id] = true;
    }
  };
  reader.readAsText(file);
  input.value = "";
}

function scrollToHeading(line: number) {
  const ta = getActiveEditor();
  if (!ta) return;
  const lines = ta.value.split("\n");
  let pos = 0;
  for (let i = 0; i < line && i < lines.length; i++) pos += lines[i].length + 1;
  ta.focus();
  ta.setSelectionRange(pos, pos);
  const lineHeight = parseFloat(getComputedStyle(ta).lineHeight) || 28;
  ta.scrollTop = Math.max(0, line * lineHeight - 80);
}

function setGoal(v: number) {
  goal.value = Math.max(10, Math.min(5000, Math.floor(v) || 200));
}

function moveChapter(articleId: string, chapterId: string, dir: -1 | 1) {
  const a = articles.value.find((x) => x.id === articleId);
  if (!a) return;
  const i = a.chapters.findIndex((c) => c.id === chapterId);
  const t = i + dir;
  if (t < 0 || t >= a.chapters.length) return;
  [a.chapters[i], a.chapters[t]] = [a.chapters[t], a.chapters[i]];
  a.updatedAt = Date.now();
}

function selectChapter(articleId: string, chapterId: string) {
  currentArticleId.value = articleId;
  currentChapterId.value = chapterId;
  expanded.value[articleId] = true;
}

function toggleExpand(id: string) { expanded.value[id] = !expanded.value[id]; }

function updateChapterContent(v: string) {
  if (!currentChapter.value || !currentArticle.value) return;
  currentChapter.value.content = v;
  currentArticle.value.updatedAt = Date.now();
}

function updateChapterTitle(v: string) {
  if (!currentChapter.value || !currentArticle.value) return;
  currentChapter.value.title = v || "未命名章节";
  currentArticle.value.updatedAt = Date.now();
}

function updateArticleTitle(v: string) {
  if (!currentArticle.value) return;
  currentArticle.value.title = v || "未命名文章";
  currentArticle.value.updatedAt = Date.now();
}

function formatMonth(s: string): string {
  const [y, m] = s.split("-");
  return `${y} 年 ${parseInt(m, 10)} 月`;
}

function formatDay(s: string) {
  const d = new Date(s + "T00:00:00");
  return {
    day: String(d.getDate()).padStart(2, "0"),
    weekday: "日一二三四五六"[d.getDay()],
  };
}

function preview(s: string): string {
  const stripped = s.replace(/[#*`_>~\-]/g, "").trim();
  return stripped.split("\n").map((l) => l.trim()).filter(Boolean).join(" ") || "暂无内容";
}

function relativeDate(s: string): string {
  if (s === todayStr.value) return "今天";
  const t = new Date();
  t.setDate(t.getDate() - 1);
  if (s === isoDate(t)) return "昨天";
  return s;
}

// Markdown toolbar
function getActiveEditor(): HTMLTextAreaElement | null {
  if (mode.value === "diary") return diaryEditor.value;
  if (mode.value === "note") return noteEditor.value;
  return chapterEditor.value;
}

function applyFormat(action: string) {
  const ta = getActiveEditor();
  if (!ta) return;
  const start = ta.selectionStart;
  const end = ta.selectionEnd;
  const value = ta.value;
  const sel = value.slice(start, end);
  let before = "";
  let after = "";
  let placeholder = sel || "";
  let newline = false;

  switch (action) {
    case "bold": before = "**"; after = "**"; placeholder = sel || "粗体"; break;
    case "italic": before = "*"; after = "*"; placeholder = sel || "斜体"; break;
    case "h1": before = "# "; placeholder = sel || "一级标题"; newline = true; break;
    case "h2": before = "## "; placeholder = sel || "二级标题"; newline = true; break;
    case "h3": before = "### "; placeholder = sel || "三级标题"; newline = true; break;
    case "quote": before = "> "; placeholder = sel || "引用"; newline = true; break;
    case "ul": before = "- "; placeholder = sel || "列表项"; newline = true; break;
    case "ol": before = "1. "; placeholder = sel || "列表项"; newline = true; break;
    case "code": before = "`"; after = "`"; placeholder = sel || "code"; break;
    case "codeblock": before = "```js\n"; after = "\n```"; placeholder = sel || "code"; newline = true; break;
    case "link": before = "["; after = "](https://)"; placeholder = sel || "链接文字"; break;
    case "hr": before = "\n---\n"; placeholder = ""; break;
    case "task": before = "- [ ] "; placeholder = sel || "待办"; newline = true; break;
    case "math": before = "$"; after = "$"; placeholder = sel || "E = mc^2"; break;
    case "mathblock": before = "$$\n"; after = "\n$$"; placeholder = sel || "\\int_a^b f(x)\\,dx"; newline = true; break;
  }

  let prefix = "";
  if (newline && start > 0 && value[start - 1] !== "\n") prefix = "\n";
  const inserted = prefix + before + placeholder + after;
  const next = value.slice(0, start) + inserted + value.slice(end);

  if (mode.value === "diary") updateDiary(next);
  else if (mode.value === "note") updateNoteContent(next);
  else updateChapterContent(next);

  nextTick(() => {
    ta.focus();
    const cursor = start + prefix.length + before.length + placeholder.length;
    ta.setSelectionRange(cursor, cursor);
  });
}

function onEditorKey(e: KeyboardEvent) {
  const ta = e.target as HTMLTextAreaElement;

  if (e.metaKey || e.ctrlKey) {
    const k = e.key.toLowerCase();
    const map: Record<string, string> = { b: "bold", i: "italic", k: "link" };
    if (map[k]) {
      e.preventDefault();
      applyFormat(map[k]);
      return;
    }
  }

  if (e.key === "Tab") {
    e.preventDefault();
    const start = ta.selectionStart;
    const end = ta.selectionEnd;
    const value = ta.value;
    if (e.shiftKey) {
      const before = value.slice(0, start);
      const lineStart = before.lastIndexOf("\n") + 1;
      const head = value.slice(lineStart, start);
      const trim = head.startsWith("  ") ? 2 : head.startsWith("\t") ? 1 : 0;
      if (!trim) return;
      const next = value.slice(0, lineStart) + head.slice(trim) + value.slice(start);
      writeContent(next);
      nextTick(() => ta.setSelectionRange(start - trim, end - trim));
    } else {
      const next = value.slice(0, start) + "  " + value.slice(end);
      writeContent(next);
      nextTick(() => ta.setSelectionRange(start + 2, start + 2));
    }
    return;
  }

  if (e.key === "Enter" && !e.shiftKey && !e.metaKey && !e.ctrlKey) {
    const start = ta.selectionStart;
    if (start !== ta.selectionEnd) return;
    const value = ta.value;
    const lineStart = value.lastIndexOf("\n", start - 1) + 1;
    const lineText = value.slice(lineStart, start);

    const task = lineText.match(/^(\s*)- \[[ xX]\] (.*)$/);
    const ul = !task && lineText.match(/^(\s*)([-*+]) (.*)$/);
    const ol = lineText.match(/^(\s*)(\d+)\. (.*)$/);
    const quote = lineText.match(/^(\s*)>\s?(.*)$/);

    const continueList = (indent: string, marker: string, body: string) => {
      e.preventDefault();
      if (!body.trim()) {
        const next = value.slice(0, lineStart) + indent + value.slice(start);
        writeContent(next);
        const cursor = lineStart + indent.length;
        nextTick(() => ta.setSelectionRange(cursor, cursor));
      } else {
        const insertion = "\n" + indent + marker;
        const next = value.slice(0, start) + insertion + value.slice(start);
        writeContent(next);
        const cursor = start + insertion.length;
        nextTick(() => ta.setSelectionRange(cursor, cursor));
      }
    };

    if (task) continueList(task[1], "- [ ] ", task[2]);
    else if (ul) continueList(ul[1], `${ul[2]} `, ul[3]);
    else if (ol) {
      const n = parseInt(ol[2], 10) + 1;
      continueList(ol[1], `${n}. `, ol[3]);
    } else if (quote) continueList(quote[1], "> ", quote[2]);
  }
}

function writeContent(value: string) {
  if (mode.value === "diary") updateDiary(value);
  else if (mode.value === "note") updateNoteContent(value);
  else updateChapterContent(value);
}

function insertAtCursor(text: string) {
  const ta = getActiveEditor();
  if (!ta) return;
  const start = ta.selectionStart;
  const end = ta.selectionEnd;
  const next = ta.value.slice(0, start) + text + ta.value.slice(end);
  writeContent(next);
  nextTick(() => {
    ta.focus();
    const cursor = start + text.length;
    ta.setSelectionRange(cursor, cursor);
  });
}

async function compressImage(file: File, maxWidth = 1600, quality = 0.86): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => {
        const scale = Math.min(1, maxWidth / img.width);
        const canvas = document.createElement("canvas");
        canvas.width = Math.round(img.width * scale);
        canvas.height = Math.round(img.height * scale);
        const ctx = canvas.getContext("2d");
        if (!ctx) return reject(new Error("canvas"));
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        const out = file.type === "image/png" && img.width <= maxWidth
          ? canvas.toDataURL("image/png")
          : canvas.toDataURL("image/jpeg", quality);
        resolve(out);
      };
      img.onerror = reject;
      img.src = String(reader.result);
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

async function handleImageFile(file?: File | null) {
  if (!file || !file.type.startsWith("image/")) return;
  try {
    const dataUrl = await compressImage(file);
    const alt = file.name.replace(/\.[^.]+$/, "") || "image";
    insertAtCursor(`\n![${alt}](${dataUrl})\n`);
  } catch {
    showToast("图片处理失败");
  }
}

function pickImage() {
  // Make sure the active editor doesn't lose focus before insertAtCursor runs
  const ta = getActiveEditor();
  ta?.focus();
  const input = document.createElement("input");
  input.type = "file";
  input.accept = "image/*";
  input.style.position = "fixed";
  input.style.left = "-9999px";
  input.style.opacity = "0";
  document.body.appendChild(input);
  input.onchange = async () => {
    const f = input.files?.[0];
    document.body.removeChild(input);
    if (f) await handleImageFile(f);
  };
  input.click();
}

function onEditorPaste(e: ClipboardEvent) {
  const items = e.clipboardData?.items;
  if (!items) return;
  for (const item of Array.from(items)) {
    if (item.type.startsWith("image/")) {
      e.preventDefault();
      handleImageFile(item.getAsFile());
      return;
    }
  }
}

function onEditorDrop(e: DragEvent) {
  const file = e.dataTransfer?.files?.[0];
  if (file && file.type.startsWith("image/")) {
    e.preventDefault();
    handleImageFile(file);
  }
}

function currentMarkdownExport(): { filename: string; body: string } | null {
  if (mode.value === "diary" && currentDiary.value) {
    return {
      filename: `${sanitizeFilenamePart(currentDiary.value.date, "diary")}.md`,
      body: `# ${currentDiary.value.date} ${currentDiary.value.mood}\n\n${currentDiary.value.content}`,
    };
  } else if (mode.value === "note" && currentNote.value) {
    return {
      filename: `${sanitizeFilenamePart(noteAutoTitle(currentNote.value), "note")}.md`,
      body: currentNote.value.title
        ? `# ${currentNote.value.title}\n\n${currentNote.value.content}`
        : currentNote.value.content,
    };
  } else if (mode.value === "article" && currentArticle.value) {
    return {
      filename: `${sanitizeFilenamePart(currentArticle.value.title, "article")}.md`,
      body: `# ${currentArticle.value.title}\n\n` +
        currentArticle.value.chapters.map((c) => `## ${c.title}\n\n${c.content}`).join("\n\n"),
    };
  }
  return null;
}

async function exportCurrent() {
  const current = currentMarkdownExport();
  if (!current) {
    showToast("当前没有可导出的内容");
    return;
  }

  try {
    const path = await save({
      defaultPath: current.filename,
      filters: [{ name: "Markdown", extensions: ["md"] }],
    });
    if (!path) return;
    await invoke("write_text_file", { path: ensureExtension(path, ".md"), data: current.body });
    showToast("Markdown 已导出");
  } catch (e) {
    console.error(e);
    showToast("导出失败");
  }
}

async function exportBackup() {
  try {
    const path = await save({
      defaultPath: `foliant-backup-${todayStr.value}.json`,
      filters: [{ name: "Foliant Backup", extensions: ["json"] }],
    });
    if (!path) return;
    await invoke("write_text_file", {
      path: ensureExtension(path, ".json"),
      data: JSON.stringify(buildPayload(), null, 2),
    });
    showToast("备份已保存");
  } catch (e) {
    console.error(e);
    showToast("备份失败");
  }
}

async function importBackup() {
  const ok = await askConfirm(
    "恢复备份？",
    "会用备份内容覆盖当前本地数据。建议先执行一次备份。",
    true,
  );
  if (!ok) return;

  try {
    const selected = await open({
      multiple: false,
      directory: false,
      filters: [{ name: "Foliant Backup", extensions: ["json"] }],
    });
    const path = resolveDialogPath(selected);
    if (!path) return;

    const raw = await invoke<string>("read_text_file", { path });
    const saved = JSON.parse(raw) as Partial<AppData>;
    applyLoadedData(saved);
    await persistData();
    showToast("备份已恢复");
  } catch (e) {
    console.error(e);
    showToast("恢复失败，备份文件可能已损坏");
  }
}

onMounted(() => {
  load();
  window.addEventListener("keydown", (e) => {
    if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === "k" && !e.shiftKey) {
      // Don't intercept ⌘K inside the editor — that's the link shortcut
      const tag = (e.target as HTMLElement | null)?.tagName;
      if (tag === "TEXTAREA") return;
      e.preventDefault();
      openPalette();
    }
  });
});
</script>

<template>
  <main class="app" :class="{ focus: focusMode, 'has-toc': tocOpen }">
    <header class="topbar" data-tauri-drag-region>
      <div class="topbar-left">
        <div class="brand">
          <img :src="logoUrl" alt="Foliant" class="brand-mark" draggable="false" />
          <span class="brand-name">Foliant</span>
        </div>
      </div>

      <nav class="mode-switch">
        <button :class="{ on: mode === 'diary' }" @click="mode = 'diary'" title="日记">
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
            <rect x="3" y="4" width="18" height="18" rx="2"/>
            <path d="M16 2v4M8 2v4M3 10h18"/>
          </svg>
        </button>
        <button :class="{ on: mode === 'note' }" @click="mode = 'note'" title="随笔">
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
            <path d="M7 4h10a2 2 0 0 1 2 2v12a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2Z"/>
            <path d="M9 8h6M9 12h6M9 16h4"/>
          </svg>
        </button>
        <button :class="{ on: mode === 'article' }" @click="mode = 'article'" title="长文">
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
            <path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/>
            <path d="M14 3v6h6M9 13h6M9 17h4"/>
          </svg>
        </button>
      </nav>

      <div class="topbar-right">
        <div class="save-state" :data-state="saveState" :title="saveState === 'saving' ? '保存中…' : saveState === 'saved' ? '已保存' : ''">
          <span class="save-dot" />
          <span class="save-text">{{ saveState === "saving" ? "保存中" : saveState === "saved" ? "已保存" : "" }}</span>
        </div>
        <div class="stats">
          <template v-if="mode === 'diary'">
            <span class="stat"><span class="stat-num">{{ streak }}</span><span class="stat-label">连续</span></span>
            <span class="divider" />
            <span class="stat"><span class="stat-num">{{ totalDays }}</span><span class="stat-label">累计</span></span>
          </template>
          <template v-else-if="mode === 'article'">
            <span class="stat"><span class="stat-num">{{ articles.length }}</span><span class="stat-label">文章</span></span>
            <span class="divider" />
            <span class="stat"><span class="stat-num">{{ articleWords }}</span><span class="stat-label">字</span></span>
          </template>
          <template v-else>
            <span class="stat"><span class="stat-num">{{ notes.length }}</span><span class="stat-label">随笔</span></span>
            <span class="divider" />
            <span class="stat"><span class="stat-num">{{ currentNote ? wordCount(currentNote.content) : 0 }}</span><span class="stat-label">字</span></span>
          </template>
        </div>
        <div class="icon-group">
          <button class="icon-btn palette-btn" title="命令面板 ⌘K" @click="openPalette">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="11" cy="11" r="7"/>
              <path d="m21 21-4.3-4.3"/>
            </svg>
            <span class="kbd">⌘K</span>
          </button>
          <button class="icon-btn" :class="{ on: tocOpen }" title="大纲" @click="trashOpen = false; tocOpen = !tocOpen">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M8 6h13M8 12h13M8 18h13M3 6h.01M3 12h.01M3 18h.01"/>
            </svg>
          </button>
          <button class="icon-btn" :class="{ on: focusMode }" title="专注模式" @click="focusMode = !focusMode">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M8 3H5a2 2 0 0 0-2 2v3M21 8V5a2 2 0 0 0-2-2h-3M3 16v3a2 2 0 0 0 2 2h3M16 21h3a2 2 0 0 0 2-2v-3"/>
            </svg>
          </button>
          <button class="icon-btn" :class="{ on: trashOpen }" title="回收站" @click="tocOpen = false; trashOpen = !trashOpen">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
            </svg>
            <span v-if="trash.length" class="badge">{{ trash.length }}</span>
          </button>
          <button class="icon-btn" title="导入 Markdown" @click="importMd">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4M17 8l-5-5-5 5M12 3v12"/>
            </svg>
          </button>
          <button class="icon-btn" title="导出 Markdown" @click="exportCurrent">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4M7 10l5 5 5-5M12 15V3"/>
            </svg>
          </button>
          <button class="icon-btn" title="备份全部数据" @click="exportBackup">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M4 7a2 2 0 0 1 2-2h9l5 5v7a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V7z"/>
              <path d="M9 9h6M9 13h6M9 17h4"/>
            </svg>
          </button>
          <button class="icon-btn" title="恢复备份" @click="importBackup">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M3 12a9 9 0 1 0 3-6.7L3 8"/>
              <path d="M3 3v5h5"/>
              <path d="M12 8v4l3 2"/>
            </svg>
          </button>
          <button class="icon-btn" :title="theme === 'light' ? '切换暗色' : '切换亮色'" @click="theme = theme === 'light' ? 'dark' : 'light'">
            <svg v-if="theme === 'light'" viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
            </svg>
            <svg v-else viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="4"/>
              <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41"/>
            </svg>
          </button>
        </div>
        <input ref="fileInput" type="file" accept=".md,.markdown,text/markdown,.txt" style="display:none" @change="onImportFile" />
      </div>
    </header>

    <aside class="sidebar">
      <div class="search">
        <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="11" cy="11" r="7"/><path d="m20 20-3.5-3.5"/>
        </svg>
        <input v-model="search" type="text" :placeholder="mode === 'diary' ? '搜索日记…' : mode === 'note' ? '搜索随笔…' : '搜索文章…'" />
      </div>

      <!-- Diary sidebar -->
      <template v-if="mode === 'diary'">
        <div class="checkin-card" :class="{ done: checkedInToday }">
          <div class="checkin-top">
            <div>
              <div class="checkin-label">今日打卡</div>
              <div class="checkin-date">{{ todayStr }}</div>
            </div>
            <div class="checkin-status">
              <svg v-if="checkedInToday" viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">
                <path d="m20 6-11 11-5-5"/>
              </svg>
              <svg v-else viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <circle cx="12" cy="12" r="10"/><path d="M12 6v6l4 2"/>
              </svg>
            </div>
          </div>
          <div class="heatmap" :title="`最近 90 天 · 累计 ${totalDays} 天`">
            <span
              v-for="cell in heatmap"
              :key="cell.date"
              class="heat-cell"
              :class="['lvl-' + cell.level, { today: cell.today }]"
              :title="cell.date"
            />
          </div>
          <button class="checkin-btn" @click="checkIn">
            <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">
              <path d="M12 5v14M5 12h14"/>
            </svg>
            {{ checkedInToday ? "继续书写" : "立即打卡" }}
          </button>
        </div>

        <div class="scroll timeline">
          <p v-if="!diariesByMonth.length" class="empty">还没有日记</p>
          <div v-for="[month, items] in diariesByMonth" :key="month" class="month-block">
            <div class="month-label">{{ formatMonth(month) }}</div>
            <div class="month-items">
              <button
                v-for="entry in items"
                :key="entry.id"
                class="day-item"
                :class="{ active: entry.id === currentDiaryId }"
                @click="currentDiaryId = entry.id"
              >
                <div class="day-dot" />
                <div class="day-date">
                  <div class="day-num">{{ formatDay(entry.date).day }}</div>
                  <div class="day-week">周{{ formatDay(entry.date).weekday }}</div>
                </div>
                <div class="day-body">
                  <div class="day-title">
                    <span v-if="entry.mood" class="day-mood">{{ entry.mood }}</span>
                    {{ relativeDate(entry.date) }}
                  </div>
                  <div class="day-preview">{{ preview(entry.content) }}</div>
                </div>
              </button>
            </div>
          </div>
        </div>
      </template>

      <!-- Article sidebar -->
      <template v-else-if="mode === 'article'">
        <button class="primary-btn" @click="createArticle">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">
            <path d="M12 5v14M5 12h14"/>
          </svg>
          新建文章
        </button>
        <div class="scroll article-tree">
          <p v-if="!filteredArticles.length" class="empty">还没有文章</p>
          <div v-for="article in filteredArticles" :key="article.id" class="article-block">
            <div
              class="article-row"
              :class="{ active: article.id === currentArticleId }"
              @click="selectChapter(article.id, article.chapters[0]?.id ?? '')"
            >
              <button class="caret" @click.stop="toggleExpand(article.id)">
                <svg viewBox="0 0 24 24" width="12" height="12" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round" :style="{ transform: expanded[article.id] ? 'rotate(90deg)' : 'rotate(0deg)' }">
                  <path d="m9 18 6-6-6-6"/>
                </svg>
              </button>
              <svg class="art-ico" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                <path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/><path d="M14 3v6h6"/>
              </svg>
              <span class="article-title">{{ article.title }}</span>
              <svg v-if="article.pinned" class="pin-flag" viewBox="0 0 24 24" width="11" height="11" fill="currentColor">
                <path d="M14 4l6 6-4 1-3 7-3-3-5 5v-5l5-5-3-3 7-3z"/>
              </svg>
              <span class="article-count">{{ article.chapters.length }}</span>
              <button class="row-act" :class="{ on: article.pinned }" @click.stop="togglePinArticle(article.id)" :title="article.pinned ? '取消置顶' : '置顶'">
                <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M12 2v6M9 8h6l1 5-4 3-4-3 1-5zM12 16v6"/>
                </svg>
              </button>
              <button class="row-act" @click.stop="deleteArticle(article.id)" title="删除文章">
                <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                </svg>
              </button>
            </div>
            <div v-if="expanded[article.id]" class="chapter-list">
              <button
                v-for="(chap, idx) in article.chapters"
                :key="chap.id"
                class="chapter-row"
                :class="{ active: chap.id === currentChapterId }"
                @click="selectChapter(article.id, chap.id)"
              >
                <span class="chapter-index">{{ String(idx + 1).padStart(2, '0') }}</span>
                <span class="chapter-title">{{ chap.title }}</span>
                <span class="chapter-actions">
                  <button class="row-act" @click.stop="moveChapter(article.id, chap.id, -1)" title="上移">
                    <svg viewBox="0 0 24 24" width="11" height="11" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="m18 15-6-6-6 6"/></svg>
                  </button>
                  <button class="row-act" @click.stop="moveChapter(article.id, chap.id, 1)" title="下移">
                    <svg viewBox="0 0 24 24" width="11" height="11" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="m6 9 6 6 6-6"/></svg>
                  </button>
                  <button class="row-act" @click.stop="deleteChapter(article.id, chap.id)" title="删除章节">
                    <svg viewBox="0 0 24 24" width="11" height="11" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6 6 18M6 6l12 12"/></svg>
                  </button>
                </span>
              </button>
              <button class="add-chapter" @click="addChapter(article.id)">
                <svg viewBox="0 0 24 24" width="12" height="12" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M12 5v14M5 12h14"/></svg>
                新增章节
              </button>
            </div>
          </div>
        </div>
      </template>

      <!-- Note sidebar -->
      <template v-else>
        <button class="primary-btn" @click="createNote">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">
            <path d="M12 5v14M5 12h14"/>
          </svg>
          新建随笔
        </button>
        <div v-if="allNoteTags.length" class="note-filters">
          <button class="tag-chip" :class="{ on: !noteTagFilter }" @click="noteTagFilter = ''">
            全部
          </button>
          <button
            v-for="tag in allNoteTags"
            :key="tag"
            class="tag-chip"
            :class="{ on: noteTagFilter === tag }"
            @click="noteTagFilter = noteTagFilter === tag ? '' : tag"
          >
            #{{ tag }}
          </button>
        </div>
        <div class="scroll note-list">
          <p v-if="!filteredNotes.length" class="empty">还没有随笔</p>
          <div
            v-for="note in filteredNotes"
            :key="note.id"
            class="note-card"
            :class="{ active: note.id === currentNoteId }"
            @click="currentNoteId = note.id"
          >
            <div class="note-card-head">
              <div class="note-card-title">
                <span class="note-name">{{ noteAutoTitle(note) }}</span>
                <svg v-if="note.pinned" class="pin-flag" viewBox="0 0 24 24" width="11" height="11" fill="currentColor">
                  <path d="M14 4l6 6-4 1-3 7-3-3-5 5v-5l5-5-3-3 7-3z"/>
                </svg>
              </div>
              <div class="note-row-actions">
                <button class="row-act" :class="{ on: note.pinned }" @click.stop="togglePinNote(note.id)" :title="note.pinned ? '取消置顶' : '置顶'">
                  <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M12 2v6M9 8h6l1 5-4 3-4-3 1-5zM12 16v6"/>
                  </svg>
                </button>
                <button class="row-act" @click.stop="deleteNote(note.id)" title="删除随笔">
                  <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                  </svg>
                </button>
              </div>
            </div>
            <div class="note-snippet">{{ noteSnippet(note) || "暂无内容" }}</div>
            <div class="note-meta">
              <span>{{ new Date(note.updatedAt).toLocaleDateString('zh-CN', { month: 'numeric', day: 'numeric' }) }}</span>
              <span v-if="note.tags.length" class="note-meta-tags">{{ note.tags.map((tag) => `#${tag}`).join(" ") }}</span>
            </div>
          </div>
        </div>
      </template>
    </aside>

    <section class="canvas">
      <!-- Diary canvas -->
      <template v-if="mode === 'diary'">
        <template v-if="currentDiary">
          <div class="canvas-head">
            <div class="title-block">
              <div class="canvas-eyebrow">{{ relativeDate(currentDiary.date) }} · 周{{ formatDay(currentDiary.date).weekday }}</div>
              <h2 class="canvas-title">{{ currentDiary.date }}</h2>
              <div v-if="currentDiary.date === todayStr" class="goal-bar">
                <div class="goal-track">
                  <div class="goal-fill" :class="{ done: todayProgress >= 100 }" :style="{ width: todayProgress + '%' }" />
                </div>
                <div class="goal-text">
                  <span>{{ todayWords }}</span>
                  <span class="goal-sep">/</span>
                  <input
                    class="goal-input"
                    type="number"
                    min="10"
                    step="50"
                    :value="goal"
                    @input="setGoal(parseInt(($event.target as HTMLInputElement).value, 10))"
                  /> 字
                  <span v-if="todayProgress >= 100" class="goal-done">✓ 达成</span>
                </div>
              </div>
              <div v-else class="canvas-sub">{{ wordCount(currentDiary.content) }} 字</div>
            </div>
            <div class="head-tools">
              <div class="mood-row">
                <button v-for="m in moods" :key="m" class="mood-btn" :class="{ on: currentDiary.mood === m }" @click="setMood(m)">{{ m }}</button>
              </div>
              <button class="icon-btn danger" title="删除" @click="deleteDiary(currentDiary.id)">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                </svg>
              </button>
            </div>
          </div>

          <div class="toolbar">
            <button v-for="t in [
              { a: 'h1', t: 'H₁' },
              { a: 'h2', t: 'H₂' },
              { a: 'h3', t: 'H₃' },
            ]" :key="t.a" class="tb-btn" @click="applyFormat(t.a)" :title="t.a">{{ t.t }}</button>
            <span class="tb-sep" />
            <button class="tb-btn bold" @click="applyFormat('bold')" title="加粗 Ctrl/⌘+B"><b>B</b></button>
            <button class="tb-btn italic" @click="applyFormat('italic')" title="斜体 Ctrl/⌘+I"><i>I</i></button>
            <button class="tb-btn" @click="applyFormat('code')" title="行内代码">‹/›</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('quote')" title="引用">❝</button>
            <button class="tb-btn" @click="applyFormat('ul')" title="无序列表">•</button>
            <button class="tb-btn" @click="applyFormat('ol')" title="有序列表">1.</button>
            <button class="tb-btn" @click="applyFormat('task')" title="待办">☑</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('link')" title="链接 Ctrl/⌘+K">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.72-1.71"/></svg>
            </button>
            <button class="tb-btn" @click="pickImage" title="插入图片">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="9" cy="9" r="2"/><path d="m21 15-5-5L5 21"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('codeblock')" title="代码块">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m16 18 6-6-6-6M8 6l-6 6 6 6"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('math')" title="行内公式 $...$"><span class="tb-math">𝑥</span></button>
            <button class="tb-btn" @click="applyFormat('mathblock')" title="块级公式 $$...$$"><span class="tb-math">∑</span></button>
            <button class="tb-btn" @click="applyFormat('hr')" title="分割线">—</button>
            <span class="tb-spacer" />
            <div class="view-switch">
              <button :class="{ on: view === 'edit' }" @click="view = 'edit'" title="编辑">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4z"/></svg>
              </button>
              <button :class="{ on: view === 'split' }" @click="view = 'split'" title="分屏">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 3v18"/></svg>
              </button>
              <button :class="{ on: view === 'preview' }" @click="view = 'preview'" title="预览">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></svg>
              </button>
            </div>
          </div>

          <div class="editor-pane" :class="['view-' + view]">
            <textarea
              v-if="view !== 'preview'"
              ref="diaryEditor"
              class="md-editor"
              :value="currentDiary.content"
              placeholder="今天发生了什么？支持 Markdown，可粘贴/拖拽图片…"
              spellcheck="false"
              @input="updateDiary(($event.target as HTMLTextAreaElement).value)"
              @keydown="onEditorKey"
              @paste="onEditorPaste"
              @drop="onEditorDrop"
              @dragover.prevent
            />
            <article v-if="view !== 'edit'" class="md-preview" v-html="diaryHtml || '<p class=&quot;ph&quot;>预览区</p>'" />
          </div>
        </template>
        <div v-else class="placeholder">
          <div class="ph-card">
            <h3>还没有今天的日记</h3>
            <p>点击下方按钮开始今天的记录</p>
            <button class="primary-btn" @click="checkIn">立即打卡</button>
          </div>
        </div>
      </template>

      <!-- Article canvas -->
      <template v-else-if="mode === 'article'">
        <template v-if="currentArticle && currentChapter">
          <div class="canvas-head">
            <div class="title-block">
              <input class="canvas-eyebrow editable" :value="currentArticle.title" @input="updateArticleTitle(($event.target as HTMLInputElement).value)" placeholder="文章标题" />
              <input class="canvas-title editable" :value="currentChapter.title" @input="updateChapterTitle(($event.target as HTMLInputElement).value)" placeholder="章节标题" />
              <div class="canvas-sub">本章 {{ chapterWords }} 字 · 全文 {{ articleWords }} 字 · 共 {{ currentArticle.chapters.length }} 章</div>
            </div>
          </div>
          <div class="toolbar">
            <button v-for="t in [
              { a: 'h1', t: 'H₁' },
              { a: 'h2', t: 'H₂' },
              { a: 'h3', t: 'H₃' },
            ]" :key="t.a" class="tb-btn" @click="applyFormat(t.a)">{{ t.t }}</button>
            <span class="tb-sep" />
            <button class="tb-btn bold" @click="applyFormat('bold')"><b>B</b></button>
            <button class="tb-btn italic" @click="applyFormat('italic')"><i>I</i></button>
            <button class="tb-btn" @click="applyFormat('code')">‹/›</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('quote')">❝</button>
            <button class="tb-btn" @click="applyFormat('ul')">•</button>
            <button class="tb-btn" @click="applyFormat('ol')">1.</button>
            <button class="tb-btn" @click="applyFormat('task')">☑</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('link')" title="链接 Ctrl/⌘+K">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/></svg>
            </button>
            <button class="tb-btn" @click="pickImage" title="插入图片">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="9" cy="9" r="2"/><path d="m21 15-5-5L5 21"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('codeblock')" title="代码块">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m16 18 6-6-6-6M8 6l-6 6 6 6"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('math')" title="行内公式"><span class="tb-math">𝑥</span></button>
            <button class="tb-btn" @click="applyFormat('mathblock')" title="块级公式"><span class="tb-math">∑</span></button>
            <button class="tb-btn" @click="applyFormat('hr')">—</button>
            <span class="tb-spacer" />
            <div class="view-switch">
              <button :class="{ on: view === 'edit' }" @click="view = 'edit'" title="编辑">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4z"/></svg>
              </button>
              <button :class="{ on: view === 'split' }" @click="view = 'split'" title="分屏">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 3v18"/></svg>
              </button>
              <button :class="{ on: view === 'preview' }" @click="view = 'preview'" title="预览">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></svg>
              </button>
            </div>
          </div>
          <div class="editor-pane" :class="['view-' + view]">
            <textarea
              v-if="view !== 'preview'"
              ref="chapterEditor"
              class="md-editor"
              :value="currentChapter.content"
              placeholder="开始书写本章正文，支持 Markdown，可粘贴/拖拽图片…"
              spellcheck="false"
              @input="updateChapterContent(($event.target as HTMLTextAreaElement).value)"
              @keydown="onEditorKey"
              @paste="onEditorPaste"
              @drop="onEditorDrop"
              @dragover.prevent
            />
            <article v-if="view !== 'edit'" class="md-preview" v-html="chapterHtml || '<p class=&quot;ph&quot;>预览区</p>'" />
          </div>
        </template>
        <div v-else class="placeholder">
          <div class="ph-card">
            <h3>开始一篇长文</h3>
            <p>新建文章后可分章节书写并支持 Markdown 排版</p>
            <button class="primary-btn" @click="createArticle">+ 新建文章</button>
          </div>
        </div>
      </template>

      <!-- Note canvas -->
      <template v-else>
        <template v-if="currentNote">
          <div class="canvas-head">
            <div class="title-block">
              <div class="canvas-eyebrow">随笔 · {{ new Date(currentNote.updatedAt).toLocaleString('zh-CN', { month: 'numeric', day: 'numeric', hour: '2-digit', minute: '2-digit' }) }}</div>
              <input class="canvas-title editable" :value="currentNote.title" @input="updateNoteTitle(($event.target as HTMLInputElement).value)" placeholder="随笔标题（可留空）" />
              <div class="canvas-sub">{{ wordCount(currentNote.content) }} 字 · {{ currentNote.tags.length }} 个标签</div>
              <div class="note-tags-wrap">
                <button
                  v-for="tag in currentNote.tags"
                  :key="tag"
                  class="tag-chip on removable"
                  @click="removeNoteTag(tag)"
                  :title="`移除 #${tag}`"
                >
                  #{{ tag }}
                  <span>×</span>
                </button>
                <input
                  v-model="noteTagInput"
                  class="tag-input"
                  type="text"
                  maxlength="24"
                  placeholder="+ 添加标签"
                  @keydown.enter.prevent="submitNoteTag"
                />
              </div>
              <div v-if="noteSuggestedTags.length" class="note-suggest">
                <span class="note-suggest-label">常用标签</span>
                <button
                  v-for="tag in noteSuggestedTags"
                  :key="tag"
                  class="tag-chip ghost-chip"
                  @click="addNoteTag(tag)"
                >
                  #{{ tag }}
                </button>
              </div>
            </div>
            <div class="head-tools">
              <button class="icon-btn danger" title="删除" @click="deleteNote(currentNote.id)">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                </svg>
              </button>
            </div>
          </div>

          <div class="toolbar">
            <button v-for="t in [
              { a: 'h1', t: 'H₁' },
              { a: 'h2', t: 'H₂' },
              { a: 'h3', t: 'H₃' },
            ]" :key="t.a" class="tb-btn" @click="applyFormat(t.a)">{{ t.t }}</button>
            <span class="tb-sep" />
            <button class="tb-btn bold" @click="applyFormat('bold')"><b>B</b></button>
            <button class="tb-btn italic" @click="applyFormat('italic')"><i>I</i></button>
            <button class="tb-btn" @click="applyFormat('code')">‹/›</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('quote')">❝</button>
            <button class="tb-btn" @click="applyFormat('ul')">•</button>
            <button class="tb-btn" @click="applyFormat('ol')">1.</button>
            <button class="tb-btn" @click="applyFormat('task')">☑</button>
            <span class="tb-sep" />
            <button class="tb-btn" @click="applyFormat('link')" title="链接 Ctrl/⌘+K">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/></svg>
            </button>
            <button class="tb-btn" @click="pickImage" title="插入图片">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="9" cy="9" r="2"/><path d="m21 15-5-5L5 21"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('codeblock')" title="代码块">
              <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m16 18 6-6-6-6M8 6l-6 6 6 6"/></svg>
            </button>
            <button class="tb-btn" @click="applyFormat('math')" title="行内公式"><span class="tb-math">𝑥</span></button>
            <button class="tb-btn" @click="applyFormat('mathblock')" title="块级公式"><span class="tb-math">∑</span></button>
            <button class="tb-btn" @click="applyFormat('hr')">—</button>
            <span class="tb-spacer" />
            <div class="view-switch">
              <button :class="{ on: view === 'edit' }" @click="view = 'edit'" title="编辑">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4z"/></svg>
              </button>
              <button :class="{ on: view === 'split' }" @click="view = 'split'" title="分屏">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 3v18"/></svg>
              </button>
              <button :class="{ on: view === 'preview' }" @click="view = 'preview'" title="预览">
                <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></svg>
              </button>
            </div>
          </div>

          <div class="editor-pane" :class="['view-' + view]">
            <textarea
              v-if="view !== 'preview'"
              ref="noteEditor"
              class="md-editor"
              :value="currentNote.content"
              placeholder="记下一段想法、一个片段，或临时灵感…"
              spellcheck="false"
              @input="updateNoteContent(($event.target as HTMLTextAreaElement).value)"
              @keydown="onEditorKey"
              @paste="onEditorPaste"
              @drop="onEditorDrop"
              @dragover.prevent
            />
            <article v-if="view !== 'edit'" class="md-preview" v-html="noteHtml || '<p class=&quot;ph&quot;>预览区</p>'" />
          </div>
        </template>
        <div v-else class="placeholder">
          <div class="ph-card">
            <h3>开始一则随笔</h3>
            <p>适合记录想法、摘抄、灵感和临时草稿</p>
            <button class="primary-btn" @click="createNote">+ 新建随笔</button>
          </div>
        </div>
      </template>
    </section>

    <aside class="drawer toc-drawer" :class="{ open: tocOpen }">
      <div class="drawer-head">
        <div class="drawer-title">
          <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M8 6h13M8 12h13M8 18h13M3 6h.01M3 12h.01M3 18h.01"/>
          </svg>
          大纲
          <span class="drawer-count">{{ toc.length }}</span>
        </div>
        <button class="row-act" @click="tocOpen = false" title="关闭">
          <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6 6 18M6 6l12 12"/></svg>
        </button>
      </div>
      <div class="drawer-body">
        <p v-if="!toc.length" class="empty">暂无标题<br/><span class="dim">在正文中使用 # ## ### 创建标题</span></p>
        <button
          v-for="h in toc"
          :key="h.line + h.text"
          class="toc-item"
          :class="['lvl-' + h.level]"
          @click="scrollToHeading(h.line)"
        >
          <span class="toc-bullet" />
          <span class="toc-text">{{ h.text }}</span>
        </button>
      </div>
    </aside>

    <aside class="drawer trash-drawer" :class="{ open: trashOpen }">
      <div class="drawer-head">
        <div class="drawer-title">
          <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
          </svg>
          回收站
          <span class="drawer-count">{{ trash.length }}</span>
        </div>
        <div class="drawer-actions">
          <button v-if="trash.length" class="ghost" @click="clearTrash">清空</button>
          <button class="row-act" @click="trashOpen = false" title="关闭">
            <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6 6 18M6 6l12 12"/></svg>
          </button>
        </div>
      </div>
      <div class="drawer-body">
        <p v-if="!trash.length" class="empty">回收站为空<br/><span class="dim">删除项保留 30 天后自动清除</span></p>
        <div v-for="t in trash" :key="t.id" class="trash-item">
          <div class="trash-info">
            <div class="trash-meta">
              <span class="trash-tag" :class="t.type">
                {{ t.type === 'diary' ? '日记' : t.type === 'article' ? '文章' : t.type === 'note' ? '随笔' : '章节' }}
              </span>
              <span class="trash-date">{{ new Date(t.deletedAt).toLocaleString('zh-CN', { month: 'numeric', day: 'numeric', hour: '2-digit', minute: '2-digit' }) }}</span>
            </div>
            <div class="trash-title">{{ trashTitle(t) }}</div>
          </div>
          <div class="trash-acts">
            <button class="ghost" @click="restoreTrash(t.id)" title="还原">
              <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M3 12a9 9 0 1 0 3-6.7L3 8"/><path d="M3 3v5h5"/>
              </svg>
            </button>
            <button class="row-act danger" @click="purgeTrash(t.id)" title="永久删除">
              <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6 6 18M6 6l12 12"/></svg>
            </button>
          </div>
        </div>
      </div>
    </aside>

    <div v-if="tocOpen || trashOpen" class="drawer-mask" @click="tocOpen = false; trashOpen = false" />

    <transition name="palette">
      <div v-if="confirmState" class="confirm-overlay" @click.self="answerConfirm(false)">
        <div class="confirm-card">
          <h3 class="confirm-title">{{ confirmState.title }}</h3>
          <p v-if="confirmState.body" class="confirm-body">{{ confirmState.body }}</p>
          <div class="confirm-actions">
            <button class="confirm-cancel" @click="answerConfirm(false)">取消</button>
            <button class="confirm-ok" :class="{ danger: confirmState.danger }" @click="answerConfirm(true)">确定</button>
          </div>
        </div>
      </div>
    </transition>

    <transition name="toast">
      <div v-if="toast" class="toast">{{ toast }}</div>
    </transition>

    <transition name="palette">
      <div v-if="paletteOpen" class="palette-overlay" @click.self="paletteOpen = false">
        <div class="palette" @keydown="onPaletteKey">
          <div class="palette-input-wrap">
            <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="11" cy="11" r="7"/><path d="m21 21-4.3-4.3"/>
            </svg>
            <input
              id="palette-input"
              v-model="paletteQuery"
              class="palette-input"
              placeholder="搜索笔记 / 章节 / 命令…"
              autocomplete="off"
              spellcheck="false"
            />
            <span class="palette-esc">esc</span>
          </div>
          <div class="palette-list">
            <p v-if="!paletteResults.length" class="palette-empty">没有结果</p>
            <button
              v-for="(it, i) in paletteResults"
              :key="it.id"
              class="palette-item"
              :class="{ active: i === paletteIndex }"
              @mousemove="paletteIndex = i"
              @click="runPaletteItem(it)"
            >
              <span class="palette-kind" :data-kind="it.kind">
                <svg v-if="it.kind === 'diary'" viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="18" rx="2"/><path d="M16 2v4M8 2v4M3 10h18"/></svg>
                <svg v-else-if="it.kind === 'article'" viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/><path d="M14 3v6h6"/></svg>
                <svg v-else viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 2"/></svg>
              </span>
              <span class="palette-label">{{ it.label }}</span>
              <span v-if="it.hint" class="palette-hint">{{ it.hint }}</span>
            </button>
          </div>
          <div class="palette-foot">
            <span><kbd>↑</kbd><kbd>↓</kbd> 移动</span>
            <span><kbd>↵</kbd> 选择</span>
            <span><kbd>esc</kbd> 关闭</span>
          </div>
        </div>
      </div>
    </transition>
  </main>
</template>

<style>
:root,
:root[data-theme="light"] {
  --bg: #f7f8fa;
  --surface: #ffffff;
  --surface-2: #f4f6f9;
  --side: #fafbfc;
  --line: #e6e9ee;
  --line-soft: #eef0f4;
  --ink: #0a0d14;
  --ink-soft: #3f4756;
  --ink-mute: #8b94a3;
  --primary: #2563eb;
  --primary-deep: #1d4ed8;
  --primary-soft: rgba(37, 99, 235, 0.08);
  --primary-tint: rgba(37, 99, 235, 0.04);
  --success: #10b981;
  --danger: #ef4444;
  --cta: #0a0d14;
  --cta-fg: #ffffff;
  --cta-hover: #1c2030;
  --code-bg: #0f1a30;
  --code-fg: #e2e8f0;
  --shadow-sm: 0 1px 2px rgba(10, 13, 20, 0.04);
  --shadow-md: 0 2px 12px rgba(10, 13, 20, 0.06);
  --shadow-lg: 0 8px 28px rgba(10, 13, 20, 0.08);
}

:root[data-theme="dark"] {
  --bg: #0b0e15;
  --surface: #11151f;
  --surface-2: #161b27;
  --side: #0f131c;
  --line: #232a39;
  --line-soft: #1a2030;
  --ink: #e8ebf2;
  --ink-soft: #aeb6c5;
  --ink-mute: #6a7388;
  --primary: #3b82f6;
  --primary-deep: #60a5fa;
  --primary-soft: rgba(59, 130, 246, 0.18);
  --primary-tint: rgba(59, 130, 246, 0.08);
  --success: #34d399;
  --danger: #f87171;
  --cta: #ffffff;
  --cta-fg: #0a0d14;
  --cta-hover: #e6e9ee;
  --code-bg: #060912;
  --code-fg: #cbd5e1;
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.4);
  --shadow-md: 0 4px 16px rgba(0, 0, 0, 0.5);
  --shadow-lg: 0 12px 36px rgba(0, 0, 0, 0.6);
}

html, body, #app { height: 100%; margin: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "PingFang SC",
    "Helvetica Neue", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: var(--ink);
  background: var(--bg);
  font-size: 15px;
}
* { box-sizing: border-box; }
button { font-family: inherit; }
input { font-family: inherit; }
</style>

<style scoped>
.app {
  display: grid;
  grid-template-columns: 340px 1fr;
  grid-template-rows: 64px 1fr;
  grid-template-areas:
    "top top"
    "side main";
  height: 100vh;
}

/* topbar — 3 column grid keeps mode-switch perfectly centered */
.topbar {
  grid-area: top;
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  padding: 0 22px;
  border-bottom: 1px solid var(--line);
  background: var(--surface);
  -webkit-app-region: drag;
}
[data-platform="mac"] .topbar { padding-left: 96px; padding-right: 36px; }
[data-platform="other"] .topbar { padding-left: 22px; padding-right: 156px; }
.topbar button,
.topbar input,
.topbar .mode-switch,
.topbar .icon-group,
.topbar .stats,
.topbar .save-state { -webkit-app-region: no-drag; }
.topbar-left { display: flex; align-items: center; min-width: 0; }
.topbar-right { display: flex; align-items: center; gap: 10px; justify-self: end; min-width: 0; }
@media (max-width: 1180px) {
  .topbar .save-state .save-text { display: none; }
  .topbar .stats { display: none; }
}

/* brand inside topbar — padding-left on .topbar already clears mac traffic lights */
.brand {
  display: flex;
  align-items: center;
  gap: 10px;
}
.brand-mark {
  width: 32px;
  height: 32px;
  border-radius: 9px;
  object-fit: cover;
  filter: drop-shadow(0 1px 3px rgba(8, 14, 32, 0.18));
  user-select: none;
}
.brand-name {
  font-size: 15px;
  font-weight: 600;
  letter-spacing: -0.01em;
  color: var(--ink);
}
[data-theme="dark"] .brand-mark { filter: drop-shadow(0 2px 6px rgba(0, 0, 0, 0.5)); }

.mode-switch {
  display: inline-flex;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 12px;
  padding: 4px;
  gap: 2px;
  box-shadow: var(--shadow-sm);
}
.mode-switch button {
  width: 44px;
  height: 36px;
  border: none;
  background: transparent;
  border-radius: 8px;
  color: var(--ink-mute);
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: 0.18s;
}
.mode-switch button:hover { color: var(--ink); }
.mode-switch button.on {
  background: var(--cta);
  color: var(--cta-fg);
}

.stats {
  display: inline-flex;
  align-items: center;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 12px;
  padding: 6px 14px;
  gap: 12px;
  box-shadow: var(--shadow-sm);
}
.stat { display: inline-flex; align-items: baseline; gap: 6px; }
.stat-num {
  font-weight: 700;
  font-size: 16px;
  color: var(--primary);
  font-variant-numeric: tabular-nums;
}
.stat-label { font-size: 11px; color: var(--ink-mute); }
.divider { width: 1px; height: 14px; background: var(--line); }

.icon-btn {
  width: 38px;
  height: 38px;
  border: 1px solid var(--line);
  background: var(--surface);
  border-radius: 10px;
  color: var(--ink-soft);
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: 0.18s;
  box-shadow: var(--shadow-sm);
}
.icon-btn:hover {
  color: var(--primary);
  border-color: var(--primary);
  transform: translateY(-1px);
}
.icon-btn.danger:hover { color: var(--danger); border-color: var(--danger); }

/* sidebar */
.sidebar {
  grid-area: side;
  border-right: 1px solid var(--line);
  background: var(--side);
  display: flex;
  flex-direction: column;
  min-height: 0;
  padding: 16px 14px;
  gap: 14px;
}

.search {
  display: flex;
  align-items: center;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 10px;
  padding: 9px 12px;
  color: var(--ink-mute);
  transition: 0.18s;
  box-shadow: var(--shadow-sm);
}
.search:focus-within { border-color: var(--primary); box-shadow: 0 0 0 3px var(--primary-soft); }
.search input {
  border: none;
  background: transparent;
  outline: none;
  flex: 1;
  font-size: 13.5px;
  margin-left: 8px;
  color: var(--ink);
}

/* check-in */
.checkin-card {
  background: var(--cta);
  color: var(--cta-fg);
  border-radius: 14px;
  padding: 16px;
  position: relative;
  overflow: hidden;
}
.checkin-card.done {
  background: var(--primary);
  color: #fff;
}
.checkin-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  position: relative;
}
.checkin-label {
  font-size: 12px;
  letter-spacing: 0.08em;
  opacity: 0.85;
  text-transform: uppercase;
  font-weight: 600;
}
.checkin-date {
  font-size: 22px;
  font-weight: 700;
  letter-spacing: 0.02em;
  margin-top: 4px;
  font-variant-numeric: tabular-nums;
}
.checkin-status { opacity: 0.9; }

.heatmap {
  position: relative;
  display: grid;
  grid-template-columns: repeat(13, 1fr);
  gap: 3px;
  margin: 14px 0 14px;
}
.heat-cell {
  aspect-ratio: 1;
  border-radius: 3px;
  background: color-mix(in oklab, currentColor 18%, transparent);
  transition: 0.15s;
}
.heat-cell.lvl-1 { background: color-mix(in oklab, currentColor 40%, transparent); }
.heat-cell.lvl-2 { background: color-mix(in oklab, currentColor 60%, transparent); }
.heat-cell.lvl-3 { background: color-mix(in oklab, currentColor 80%, transparent); }
.heat-cell.lvl-4 { background: currentColor; }
.heat-cell.today { box-shadow: 0 0 0 1.5px currentColor; }

.checkin-btn {
  width: 100%;
  border: none;
  background: color-mix(in oklab, currentColor 18%, transparent);
  color: inherit;
  height: 38px;
  border-radius: 10px;
  font-weight: 600;
  font-size: 13.5px;
  cursor: pointer;
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  transition: 0.18s;
}
.checkin-btn:hover { background: color-mix(in oklab, currentColor 28%, transparent); transform: translateY(-1px); }

/* timeline */
.scroll { flex: 1; overflow-y: auto; padding-right: 4px; min-height: 0; }
.empty {
  text-align: center;
  font-size: 13px;
  color: var(--ink-mute);
  padding: 32px 8px;
  font-style: italic;
}
.month-block { margin-bottom: 14px; }
.month-label {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: var(--ink-mute);
  padding: 6px 8px 8px;
}
.month-items {
  position: relative;
  padding-left: 24px;
}
.month-items::before {
  content: "";
  position: absolute;
  left: 12px;
  top: 8px;
  bottom: 8px;
  width: 1px;
  background: var(--line);
}
.day-item {
  position: relative;
  width: 100%;
  display: grid;
  grid-template-columns: 48px 1fr;
  gap: 12px;
  text-align: left;
  padding: 10px 10px 10px 4px;
  border: none;
  background: transparent;
  border-radius: 10px;
  cursor: pointer;
  transition: 0.18s;
}
.day-item:hover { background: var(--primary-tint); }
.day-item.active { background: var(--primary-soft); }
.day-dot {
  position: absolute;
  left: -18px;
  top: 16px;
  width: 11px;
  height: 11px;
  border-radius: 50%;
  background: var(--surface);
  border: 2px solid var(--ink-mute);
  transition: 0.18s;
}
.day-item:hover .day-dot { border-color: var(--primary); }
.day-item.active .day-dot {
  background: var(--primary);
  border-color: var(--primary);
  box-shadow: 0 0 0 4px var(--primary-soft);
}
.day-date { text-align: center; }
.day-num {
  font-size: 22px;
  font-weight: 700;
  line-height: 1;
  color: var(--ink);
  font-variant-numeric: tabular-nums;
}
.day-week {
  font-size: 11px;
  color: var(--ink-mute);
  margin-top: 4px;
}
.day-body { min-width: 0; }
.day-title {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--ink);
  display: flex;
  align-items: center;
  gap: 6px;
}
.day-mood { font-size: 14px; }
.day-preview {
  font-size: 12px;
  color: var(--ink-mute);
  margin-top: 4px;
  line-height: 1.5;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* article tree */
.primary-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  width: 100%;
  height: 38px;
  border: none;
  border-radius: 10px;
  background: var(--cta);
  color: var(--cta-fg);
  font-weight: 600;
  font-size: 13.5px;
  cursor: pointer;
  transition: 0.18s;
}
.primary-btn:hover { background: var(--cta-hover); }

.article-tree { display: flex; flex-direction: column; }
.article-block { margin-bottom: 4px; }
.article-row {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 9px 8px;
  border-radius: 10px;
  cursor: pointer;
  transition: background 0.18s;
  font-size: 14px;
  font-weight: 600;
  color: var(--ink);
}
.article-row:hover { background: var(--primary-tint); }
.article-row.active { background: var(--primary-soft); color: var(--primary-deep); }
.caret {
  width: 22px;
  height: 22px;
  border: none;
  background: transparent;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: var(--ink-mute);
  border-radius: 5px;
}
.caret:hover { color: var(--primary); background: rgba(255, 255, 255, 0.6); }
.caret svg { transition: transform 0.18s; }
.art-ico { color: var(--primary); flex-shrink: 0; }
.article-title {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.article-count {
  font-size: 11px;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 999px;
  padding: 2px 8px;
  color: var(--ink-soft);
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}
.row-act {
  width: 22px;
  height: 22px;
  border: none;
  background: transparent;
  border-radius: 5px;
  cursor: pointer;
  color: var(--ink-mute);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: 0.15s;
  padding: 0;
}
.row-act:hover { background: var(--surface); color: var(--primary); }

.chapter-list {
  margin-left: 26px;
  border-left: 2px solid var(--line);
  padding: 4px 0 4px 8px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.chapter-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 7px 10px;
  border-radius: 8px;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 13px;
  color: var(--ink-soft);
  transition: 0.18s;
  text-align: left;
  width: 100%;
}
.chapter-row:hover { background: var(--primary-tint); }
.chapter-row.active {
  background: var(--primary-soft);
  color: var(--primary-deep);
  font-weight: 600;
}
.chapter-index {
  font-size: 11px;
  font-weight: 700;
  width: 22px;
  text-align: center;
  color: var(--ink-mute);
  font-variant-numeric: tabular-nums;
  letter-spacing: 0.05em;
}
.chapter-row.active .chapter-index { color: var(--primary); }
.chapter-title {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.chapter-actions { display: none; gap: 0; }
.chapter-row:hover .chapter-actions { display: inline-flex; }
.add-chapter {
  margin-top: 4px;
  font-size: 12.5px;
  border: 1.5px dashed var(--line);
  background: transparent;
  color: var(--ink-mute);
  border-radius: 8px;
  padding: 8px 10px;
  cursor: pointer;
  transition: 0.18s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}
.add-chapter:hover {
  border-color: var(--primary);
  color: var(--primary);
  background: var(--primary-tint);
}

/* note list */
.note-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.tag-chip {
  height: 28px;
  border: 1px solid var(--line);
  background: var(--surface);
  color: var(--ink-soft);
  border-radius: 999px;
  padding: 0 10px;
  font-size: 12px;
  cursor: pointer;
  transition: 0.15s;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}
.tag-chip:hover {
  border-color: var(--primary);
  color: var(--primary);
  background: var(--primary-tint);
}
.tag-chip.on {
  border-color: transparent;
  background: var(--primary);
  color: #fff;
}
.tag-chip.removable span {
  opacity: 0.8;
  font-size: 11px;
}
.tag-chip.ghost-chip {
  background: transparent;
}
.note-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.note-card {
  border: 1px solid var(--line-soft);
  background: var(--surface);
  border-radius: 12px;
  padding: 12px;
  cursor: pointer;
  transition: 0.18s;
}
.note-card:hover {
  border-color: var(--line);
  box-shadow: var(--shadow-sm);
  transform: translateY(-1px);
}
.note-card.active {
  border-color: color-mix(in oklab, var(--primary) 36%, var(--line));
  background: var(--primary-tint);
  box-shadow: inset 0 0 0 1px var(--primary-soft);
}
.note-card-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 8px;
}
.note-card-title {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}
.note-name {
  font-size: 14px;
  font-weight: 700;
  color: var(--ink);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.note-row-actions {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  flex-shrink: 0;
  opacity: 0;
  transition: opacity 0.15s;
}
.note-card:hover .note-row-actions,
.note-card.active .note-row-actions {
  opacity: 1;
}
.note-snippet {
  margin-top: 8px;
  font-size: 12.5px;
  line-height: 1.65;
  color: var(--ink-soft);
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
  min-height: 3.9em;
}
.note-meta {
  margin-top: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  font-size: 11.5px;
  color: var(--ink-mute);
}
.note-meta-tags {
  flex: 1;
  text-align: right;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.note-tags-wrap {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 12px;
}
.tag-input {
  min-width: 120px;
  height: 30px;
  border: 1px dashed var(--line);
  background: transparent;
  color: var(--ink);
  border-radius: 999px;
  padding: 0 12px;
  font-size: 12.5px;
  outline: none;
  font-family: inherit;
}
.tag-input:focus {
  border-color: var(--primary);
  background: var(--surface);
}
.note-suggest {
  margin-top: 10px;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
}
.note-suggest-label {
  font-size: 11.5px;
  color: var(--ink-mute);
}

/* canvas */
.canvas {
  grid-area: main;
  background: var(--surface);
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.canvas-head {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 28px 56px 20px;
  gap: 16px;
  border-bottom: 1px solid var(--line-soft);
}
.title-block { flex: 1; min-width: 0; }
.canvas-eyebrow {
  font-size: 12px;
  font-weight: 600;
  color: var(--primary);
  letter-spacing: 0.1em;
  text-transform: uppercase;
}
.canvas-title {
  margin: 6px 0 6px;
  font-size: 32px;
  font-weight: 700;
  letter-spacing: -0.02em;
  color: var(--ink);
}
.canvas-sub { font-size: 12.5px; color: var(--ink-mute); font-variant-numeric: tabular-nums; }
.canvas-eyebrow.editable, .canvas-title.editable {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  padding: 0;
}
.canvas-title.editable { font-size: 32px; }

.head-tools {
  display: flex;
  align-items: center;
  gap: 10px;
}
.mood-row { display: inline-flex; gap: 2px; }
.mood-btn {
  width: 34px;
  height: 34px;
  border: 1px solid transparent;
  background: transparent;
  border-radius: 8px;
  font-size: 17px;
  cursor: pointer;
  opacity: 0.5;
  transition: 0.18s;
}
.mood-btn:hover { opacity: 1; background: var(--primary-tint); }
.mood-btn.on {
  opacity: 1;
  background: var(--primary-soft);
  border-color: var(--primary);
}

/* toolbar */
.toolbar {
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 10px 24px;
  border-bottom: 1px solid var(--line-soft);
  background: var(--surface-2);
  flex-wrap: wrap;
}
.tb-btn {
  min-width: 32px;
  height: 32px;
  padding: 0 8px;
  border: none;
  background: transparent;
  border-radius: 7px;
  font-size: 13px;
  color: var(--ink-soft);
  cursor: pointer;
  transition: 0.15s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-weight: 500;
}
.tb-btn:hover { background: var(--primary-soft); color: var(--primary); }
.tb-btn.bold b { font-weight: 800; font-family: Georgia, serif; }
.tb-btn.italic i { font-family: Georgia, serif; font-size: 15px; }
.tb-btn .tb-math { font-family: "Latin Modern Math", "Times New Roman", serif; font-size: 16px; font-style: italic; line-height: 1; }
.tb-sep { width: 1px; height: 18px; background: var(--line); margin: 0 6px; }
.tb-spacer { flex: 1; }

/* KaTeX theming */
.md-preview :deep(.katex) { font-size: 1.05em; color: var(--ink); }
.md-preview :deep(.katex-display) {
  margin: 1em 0;
  padding: 12px 16px;
  background: var(--surface-2);
  border-radius: 10px;
  border: 1px solid var(--line);
  overflow-x: auto;
}
[data-theme="dark"] .md-preview :deep(.katex) { color: var(--ink); }

.view-switch {
  display: inline-flex;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 8px;
  padding: 2px;
  gap: 1px;
}
.view-switch button {
  width: 32px;
  height: 26px;
  border: none;
  background: transparent;
  border-radius: 5px;
  color: var(--ink-mute);
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: 0.15s;
}
.view-switch button:hover { color: var(--ink); }
.view-switch button.on {
  background: var(--primary);
  color: #fff;
}

/* editor pane */
.editor-pane {
  flex: 1;
  display: grid;
  min-height: 0;
}
.editor-pane.view-edit { grid-template-columns: 1fr; }
.editor-pane.view-preview { grid-template-columns: 1fr; }
.editor-pane.view-split { grid-template-columns: 1fr 1fr; }

.md-editor {
  width: 100%;
  height: 100%;
  border: none;
  outline: none;
  resize: none;
  padding: 24px 56px 64px;
  font-size: 15px;
  line-height: 1.85;
  background: transparent;
  color: var(--ink);
  font-family: "SF Mono", "JetBrains Mono", ui-monospace, Menlo, Consolas, monospace;
  letter-spacing: 0.005em;
  overflow-y: auto;
}
.md-editor::placeholder { color: var(--ink-mute); }

.view-split .md-editor { border-right: 1px solid var(--line-soft); }

.md-preview {
  padding: 24px 56px 64px;
  overflow-y: auto;
  font-size: 15.5px;
  line-height: 1.8;
  color: var(--ink);
  background: var(--surface-2);
}
.view-preview .md-preview { background: var(--surface); }

.md-preview :deep(h1),
.md-preview :deep(h2),
.md-preview :deep(h3),
.md-preview :deep(h4) {
  color: var(--ink);
  letter-spacing: -0.01em;
  margin: 1.4em 0 0.6em;
  line-height: 1.3;
}
.md-preview :deep(h1) {
  font-size: 28px;
  border-bottom: 2px solid var(--line);
  padding-bottom: 0.3em;
}
.md-preview :deep(h2) { font-size: 22px; }
.md-preview :deep(h3) { font-size: 18px; }
.md-preview :deep(p) { margin: 0.8em 0; }
.md-preview :deep(a) { color: var(--primary); text-decoration: none; border-bottom: 1px solid var(--primary-soft); }
.md-preview :deep(a:hover) { border-bottom-color: var(--primary); }
.md-preview :deep(strong) { color: var(--ink); font-weight: 700; }
.md-preview :deep(em) { color: var(--ink-soft); }
.md-preview :deep(code) {
  background: var(--primary-tint);
  color: var(--primary-deep);
  padding: 2px 6px;
  border-radius: 4px;
  font-family: "SF Mono", ui-monospace, monospace;
  font-size: 0.9em;
}
.md-preview :deep(pre) {
  background: var(--code-bg);
  color: var(--code-fg);
  padding: 16px 18px;
  border-radius: 10px;
  overflow-x: auto;
  margin: 1em 0;
  font-size: 13.5px;
  line-height: 1.65;
}
.md-preview :deep(pre code) {
  background: transparent;
  color: inherit;
  padding: 0;
}
.md-preview :deep(blockquote) {
  margin: 1em 0;
  padding: 6px 16px;
  border-left: 3px solid var(--primary);
  background: var(--primary-tint);
  color: var(--ink-soft);
  border-radius: 0 8px 8px 0;
}
.md-preview :deep(ul),
.md-preview :deep(ol) { padding-left: 1.4em; margin: 0.8em 0; }
.md-preview :deep(li) { margin: 0.3em 0; }
.md-preview :deep(li input[type="checkbox"]) { margin-right: 6px; }
.md-preview :deep(hr) {
  border: none;
  height: 1px;
  background: var(--line);
  margin: 2em 0;
}
.md-preview :deep(table) {
  border-collapse: collapse;
  width: 100%;
  margin: 1em 0;
}
.md-preview :deep(th),
.md-preview :deep(td) {
  border: 1px solid var(--line);
  padding: 8px 12px;
  text-align: left;
}
.md-preview :deep(th) { background: var(--surface-2); font-weight: 600; }
.md-preview :deep(.ph) { color: var(--ink-mute); font-style: italic; }
.md-preview :deep(img) {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 1em auto;
  border-radius: 10px;
  border: 1px solid var(--line);
  box-shadow: var(--shadow-sm);
}
.md-preview :deep(pre code.hljs) {
  background: transparent;
  padding: 0;
  font-family: "SF Mono", ui-monospace, "JetBrains Mono", monospace;
}
.md-preview :deep(pre) {
  position: relative;
  border: 1px solid color-mix(in oklab, var(--code-bg) 60%, var(--line));
}

.placeholder {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.ph-card {
  text-align: center;
  background: var(--surface-2);
  border: 1px solid var(--line);
  border-radius: 18px;
  padding: 36px 48px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  box-shadow: var(--shadow-md);
}
.ph-card h3 { margin: 0; font-size: 19px; font-weight: 700; }
.ph-card p { margin: 0 0 8px; color: var(--ink-mute); font-size: 13.5px; }
.ph-card .primary-btn { width: auto; padding: 0 22px; }

/* scrollbars */
.scroll::-webkit-scrollbar,
.md-editor::-webkit-scrollbar,
.md-preview::-webkit-scrollbar { width: 8px; }
.scroll::-webkit-scrollbar-thumb,
.md-editor::-webkit-scrollbar-thumb,
.md-preview::-webkit-scrollbar-thumb {
  background: var(--line);
  border-radius: 4px;
  border: 2px solid transparent;
  background-clip: padding-box;
}
.scroll::-webkit-scrollbar-thumb:hover,
.md-editor::-webkit-scrollbar-thumb:hover,
.md-preview::-webkit-scrollbar-thumb:hover {
  background: var(--ink-mute);
  background-clip: padding-box;
  border: 2px solid transparent;
}

/* icon group + badge */
.icon-group {
  display: inline-flex;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 12px;
  padding: 3px;
  gap: 1px;
  box-shadow: var(--shadow-sm);
}
.icon-group .icon-btn {
  width: 34px;
  height: 32px;
  border: none;
  border-radius: 8px;
  background: transparent;
  box-shadow: none;
  color: var(--ink-soft);
  position: relative;
}
.icon-group .icon-btn:hover {
  transform: none;
  background: var(--primary-tint);
  border-color: transparent;
  color: var(--primary);
}
.icon-group .icon-btn.on {
  background: var(--primary-soft);
  color: var(--primary-deep);
}
.badge {
  position: absolute;
  top: 2px;
  right: 2px;
  min-width: 14px;
  height: 14px;
  padding: 0 3px;
  background: var(--danger);
  color: #fff;
  border-radius: 999px;
  font-size: 9.5px;
  font-weight: 700;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

/* save state indicator */
.save-state {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 0 10px;
  height: 26px;
  border-radius: 999px;
  font-size: 11.5px;
  color: var(--ink-mute);
  background: transparent;
  transition: opacity 0.2s, background 0.2s;
  -webkit-app-region: no-drag;
  min-width: 22px;
}
.save-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--ink-mute);
  transition: background 0.2s, transform 0.2s;
}
.save-state[data-state="saving"] .save-dot {
  background: var(--primary);
  animation: pulse 1.1s ease-in-out infinite;
}
.save-state[data-state="saved"] .save-dot { background: #22c55e; }
.save-state[data-state="idle"] .save-text { display: none; }
.save-state[data-state="idle"] { opacity: 0.4; }
@keyframes pulse {
  0%, 100% { opacity: 0.4; transform: scale(0.85); }
  50% { opacity: 1; transform: scale(1.15); }
}

/* palette button kbd */
.palette-btn { gap: 6px; padding: 0 10px !important; width: auto !important; }
.palette-btn .kbd {
  font-size: 10px;
  padding: 1px 5px;
  border-radius: 4px;
  background: var(--surface-2);
  border: 1px solid var(--line);
  color: var(--ink-mute);
  font-family: "SF Mono", ui-monospace, monospace;
  letter-spacing: 0.02em;
}

/* article pinned flag */
.pin-flag {
  color: var(--primary);
  flex-shrink: 0;
  margin-right: 2px;
}
.row-act.on { color: var(--primary); }

/* command palette */
.palette-overlay {
  position: fixed;
  inset: 0;
  background: color-mix(in oklab, var(--ink) 36%, transparent);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  z-index: 100;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 14vh;
}
.palette {
  width: min(640px, calc(100vw - 48px));
  max-height: 60vh;
  display: flex;
  flex-direction: column;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 14px;
  box-shadow: 0 20px 60px -10px rgba(8, 14, 32, 0.35), 0 0 0 1px var(--line) inset;
  overflow: hidden;
}
.palette-input-wrap {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid var(--line);
  color: var(--ink-mute);
}
.palette-input {
  flex: 1;
  border: none;
  background: transparent;
  outline: none;
  font-size: 15px;
  color: var(--ink);
  font-family: inherit;
}
.palette-input::placeholder { color: var(--ink-mute); }
.palette-esc {
  font-size: 10.5px;
  padding: 2px 6px;
  border-radius: 4px;
  background: var(--surface-2);
  border: 1px solid var(--line);
  color: var(--ink-mute);
  font-family: "SF Mono", ui-monospace, monospace;
}
.palette-list {
  overflow-y: auto;
  padding: 6px;
  flex: 1;
  min-height: 0;
}
.palette-empty {
  text-align: center;
  color: var(--ink-mute);
  font-size: 13px;
  padding: 32px 0;
  margin: 0;
}
.palette-item {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 9px 10px;
  border: none;
  background: transparent;
  color: var(--ink);
  border-radius: 8px;
  cursor: pointer;
  font-size: 13.5px;
  text-align: left;
  font-family: inherit;
}
.palette-item.active { background: var(--primary-soft); color: var(--primary-deep); }
.palette-kind {
  width: 22px;
  height: 22px;
  border-radius: 6px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: var(--surface-2);
  color: var(--ink-mute);
  flex-shrink: 0;
}
.palette-item.active .palette-kind { background: var(--primary-tint); color: var(--primary); }
.palette-kind[data-kind="action"] { color: var(--primary); }
.palette-label { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.palette-hint {
  font-size: 11.5px;
  color: var(--ink-mute);
  max-width: 220px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex-shrink: 0;
}
.palette-item.active .palette-hint { color: var(--primary-deep); opacity: 0.7; }
.palette-foot {
  display: flex;
  gap: 14px;
  padding: 8px 14px;
  border-top: 1px solid var(--line);
  font-size: 11px;
  color: var(--ink-mute);
  background: var(--surface-2);
}
.palette-foot kbd {
  font-family: "SF Mono", ui-monospace, monospace;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 3px;
  padding: 0 4px;
  margin-right: 4px;
  font-size: 10px;
}
.palette-enter-from, .palette-leave-to { opacity: 0; }
.palette-enter-from .palette, .palette-leave-to .palette { transform: translateY(-8px) scale(0.98); }
.palette-enter-active, .palette-leave-active { transition: opacity 0.16s; }
.palette-enter-active .palette, .palette-leave-active .palette { transition: transform 0.18s ease-out; }

/* in-app confirm modal */
.confirm-overlay {
  position: fixed;
  inset: 0;
  background: color-mix(in oklab, var(--ink) 36%, transparent);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  z-index: 110;
  display: flex;
  align-items: center;
  justify-content: center;
}
.confirm-card {
  width: min(420px, calc(100vw - 48px));
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 14px;
  padding: 22px 22px 18px;
  box-shadow: 0 20px 60px -10px rgba(8, 14, 32, 0.35);
}
.confirm-title { margin: 0 0 8px; font-size: 16px; font-weight: 700; color: var(--ink); }
.confirm-body { margin: 0 0 18px; font-size: 13.5px; color: var(--ink-soft); line-height: 1.6; }
.confirm-actions { display: flex; gap: 10px; justify-content: flex-end; }
.confirm-cancel,
.confirm-ok {
  border: 1px solid var(--line);
  background: var(--surface-2);
  color: var(--ink);
  border-radius: 8px;
  padding: 7px 16px;
  font-size: 13px;
  cursor: pointer;
  font-family: inherit;
}
.confirm-cancel:hover { background: var(--surface); }
.confirm-ok { background: var(--cta); color: var(--cta-fg); border-color: transparent; }
.confirm-ok:hover { background: var(--cta-hover); }
.confirm-ok.danger { background: var(--danger); color: #fff; }
.confirm-ok.danger:hover { filter: brightness(0.92); }

/* toast */
.toast {
  position: fixed;
  bottom: 36px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--cta);
  color: var(--cta-fg);
  padding: 10px 20px;
  border-radius: 999px;
  font-size: 13px;
  box-shadow: 0 8px 24px rgba(8, 14, 32, 0.25);
  z-index: 200;
}
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translate(-50%, 8px); }
.toast-enter-active, .toast-leave-active { transition: opacity 0.18s, transform 0.18s; }

/* math block container styling already from KaTeX; ensure spacing */
.md-preview :deep(.math-block) { margin: 1em 0; }

/* goal bar */
.goal-bar {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-top: 8px;
  max-width: 360px;
}
.goal-track {
  height: 6px;
  border-radius: 999px;
  background: var(--line);
  overflow: hidden;
}
.goal-fill {
  height: 100%;
  background: var(--primary);
  transition: width 0.3s ease;
  border-radius: 999px;
}
.goal-fill.done { background: var(--success); }
.goal-text {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--ink-mute);
  font-variant-numeric: tabular-nums;
}
.goal-text > span:first-child { color: var(--ink); font-weight: 600; }
.goal-sep { color: var(--ink-mute); }
.goal-input {
  width: 56px;
  border: 1px solid var(--line);
  background: var(--surface);
  color: var(--ink);
  border-radius: 6px;
  padding: 1px 6px;
  font-size: 12px;
  outline: none;
  font-variant-numeric: tabular-nums;
  font-family: inherit;
}
.goal-input:focus { border-color: var(--primary); }
.goal-done {
  margin-left: 6px;
  color: var(--success);
  font-weight: 600;
}

/* focus mode */
.app.focus {
  grid-template-columns: 1fr;
  grid-template-areas:
    "top"
    "main";
}
.app.focus .sidebar { display: none; }
.app.focus .topbar .stats,
.app.focus .topbar .icon-group { opacity: 0.55; transition: 0.18s; }
.app.focus .topbar:hover .stats,
.app.focus .topbar:hover .icon-group { opacity: 1; }
.app.focus .canvas-head { padding: 32px 80px 20px; }
.app.focus .toolbar { padding: 10px 80px; }
.app.focus .md-editor,
.app.focus .md-preview { padding: 24px 80px 80px; max-width: 920px; margin: 0 auto; width: 100%; }

/* drawers */
.drawer {
  position: fixed;
  top: 64px;
  right: 0;
  bottom: 0;
  width: 320px;
  background: var(--surface);
  border-left: 1px solid var(--line);
  box-shadow: var(--shadow-lg);
  display: flex;
  flex-direction: column;
  transform: translateX(100%);
  transition: transform 0.22s cubic-bezier(.32, .72, 0, 1);
  z-index: 50;
}
.drawer.open { transform: translateX(0); }
.drawer-mask {
  position: fixed;
  top: 64px;
  left: 0;
  right: 320px;
  bottom: 0;
  background: rgba(0, 0, 0, 0.04);
  z-index: 40;
}
.drawer-head {
  height: 52px;
  padding: 0 14px 0 18px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid var(--line-soft);
  flex-shrink: 0;
}
.drawer-title {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  font-weight: 700;
  letter-spacing: 0.02em;
  color: var(--ink);
}
.drawer-count {
  font-size: 11px;
  font-weight: 600;
  background: var(--surface-2);
  border: 1px solid var(--line);
  border-radius: 999px;
  padding: 1px 8px;
  color: var(--ink-soft);
  font-variant-numeric: tabular-nums;
}
.drawer-actions { display: inline-flex; align-items: center; gap: 6px; }
.drawer-body {
  flex: 1;
  overflow-y: auto;
  padding: 8px 12px 16px;
}
.drawer-body .empty {
  text-align: center;
  font-size: 13px;
  color: var(--ink-mute);
  padding: 32px 12px;
  line-height: 1.7;
}
.drawer-body .dim { font-size: 11.5px; color: var(--ink-mute); }

/* TOC items */
.toc-item {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  text-align: left;
  border: none;
  background: transparent;
  border-radius: 7px;
  padding: 7px 10px;
  cursor: pointer;
  font-size: 13px;
  color: var(--ink-soft);
  transition: 0.15s;
  line-height: 1.5;
}
.toc-item:hover { background: var(--primary-tint); color: var(--primary); }
.toc-item.lvl-1 { font-weight: 700; color: var(--ink); font-size: 13.5px; }
.toc-item.lvl-2 { padding-left: 22px; }
.toc-item.lvl-3 { padding-left: 36px; font-size: 12.5px; }
.toc-item.lvl-4 { padding-left: 50px; font-size: 12px; color: var(--ink-mute); }
.toc-bullet {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: currentColor;
  opacity: 0.5;
  flex-shrink: 0;
}
.toc-item.lvl-1 .toc-bullet { width: 6px; height: 6px; opacity: 1; background: var(--primary); }
.toc-text {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* trash items */
.trash-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid var(--line-soft);
  background: var(--surface);
  margin-bottom: 6px;
  transition: 0.15s;
}
.trash-item:hover { border-color: var(--line); box-shadow: var(--shadow-sm); }
.trash-info { min-width: 0; flex: 1; }
.trash-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}
.trash-tag {
  font-size: 10.5px;
  font-weight: 700;
  letter-spacing: 0.04em;
  border-radius: 4px;
  padding: 1px 6px;
  background: var(--surface-2);
  color: var(--ink-soft);
}
.trash-tag.diary { background: rgba(37, 99, 235, 0.1); color: var(--primary); }
.trash-tag.article { background: rgba(16, 185, 129, 0.12); color: var(--success); }
.trash-tag.note { background: rgba(245, 158, 11, 0.14); color: #b45309; }
.trash-tag.chapter { background: rgba(139, 148, 163, 0.16); color: var(--ink-soft); }
.trash-date {
  font-size: 11px;
  color: var(--ink-mute);
  font-variant-numeric: tabular-nums;
}
.trash-title {
  font-size: 13px;
  font-weight: 600;
  color: var(--ink);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.trash-acts { display: inline-flex; align-items: center; gap: 4px; flex-shrink: 0; }
.row-act.danger:hover { background: rgba(239, 68, 68, 0.1); color: var(--danger); }

.ghost {
  border: 1px solid var(--line);
  background: transparent;
  color: var(--ink-soft);
  border-radius: 7px;
  padding: 5px 10px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: 0.15s;
  font-family: inherit;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
}
.ghost:hover {
  border-color: var(--primary);
  color: var(--primary);
  background: var(--primary-tint);
}
</style>
