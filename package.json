import { useState, useEffect, useCallback, useRef } from "react";

const API_BASE = "https://api.todoist.com/rest/v2";

const C = {
  bg: "#0d0d12", surface: "#16161f", surfaceHover: "#1e1e2e", border: "#252538",
  accent: "#e44332", accentSoft: "#ff6b5b", text: "#eeeef5", textMuted: "#777799",
  green: "#3dba78", yellow: "#f5c842", orange: "#f5823a", blue: "#4a8ef5",
};

const priorityConfig = {
  4: { label: "P1", color: "#e44332", bg: "#2e1212" },
  3: { label: "P2", color: "#f5a623", bg: "#2e2012" },
  2: { label: "P3", color: "#4a8ef5", bg: "#12202e" },
  1: { label: "P4", color: "#777799", bg: "#16161f" },
};

async function todoistFetch(token, path, options = {}) {
  const res = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json", ...(options.headers || {}) },
  });
  if (!res.ok) throw new Error(`${res.status}`);
  if (res.status === 204) return null;
  return res.json();
}

function getDueDate(task) {
  if (!task.due) return null;
  if (task.due.datetime) return new Date(task.due.datetime);
  if (task.due.date) return new Date(task.due.date + "T23:59:59");
  return null;
}

function getUrgencyLevel(task) {
  const due = getDueDate(task);
  if (!due) return "none";
  const diff = due - new Date();
  if (diff < 0) return "overdue";
  if (diff < 3600000) return "critical";
  if (diff < 86400000) return "urgent";
  if (diff < 86400000 * 2) return "soon";
  return "normal";
}

const urgencyStyle = {
  overdue: { border: "1.5px solid #e44332", glow: "0 0 12px #e4433244", badge: { bg: "#e44332", text: "#fff", label: "⏰ GECİKTİ" } },
  critical: { border: "1.5px solid #f5823a", glow: "0 0 10px #f5823a33", badge: { bg: "#f5823a", text: "#fff", label: "🔥 Kritik" } },
  urgent: { border: "1.5px solid #f5c842", glow: "0 0 8px #f5c84222", badge: { bg: "#f5c842", text: "#111", label: "⚡ Bugün" } },
  soon: { border: "1.5px solid #4a8ef5", glow: "none", badge: { bg: "#4a8ef5", text: "#fff", label: "📅 Yakında" } },
  normal: { border: `1px solid #252538`, glow: "none", badge: null },
  none: { border: `1px solid #252538`, glow: "none", badge: null },
};

function playAlertSound() {
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    [0, 150, 300].forEach((delay, i) => {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain); gain.connect(ctx.destination);
      osc.frequency.value = 660 - i * 80;
      osc.type = "sine";
      gain.gain.setValueAtTime(0.3, ctx.currentTime + delay / 1000);
      gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + delay / 1000 + 0.3);
      osc.start(ctx.currentTime + delay / 1000);
      osc.stop(ctx.currentTime + delay / 1000 + 0.4);
    });
  } catch (_) {}
}

export default function App() {
  const [token, setToken] = useState(() => localStorage.getItem("todoist_token") || "");
  const [tokenInput, setTokenInput] = useState("");
  const [tasks, setTasks] = useState([]);
  const [projects, setProjects] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const [activeTab, setActiveTab] = useState("tasks");
  const [selectedProject, setSelectedProject] = useState("all");
  const [showAddTask, setShowAddTask] = useState(false);
  const [newTask, setNewTask] = useState({ content: "", priority: 1, project_id: "", due_string: "" });
  const [editTask, setEditTask] = useState(null);
  const [completingId, setCompletingId] = useState(null);
  const [aiLoading, setAiLoading] = useState(false);
  const [aiSuggestions, setAiSuggestions] = useState([]);
  const [aiPrompt, setAiPrompt] = useState("");
  const [notifPermission, setNotifPermission] = useState(Notification?.permission || "default");
  const [reminders, setReminders] = useState(() => {
    try { return JSON.parse(localStorage.getItem("todoist_reminders") || "{}"); } catch { return {}; }
  });
  const [popups, setPopups] = useState([]);
  const [reminderModal, setReminderModal] = useState(null);
  const [customMinutes, setCustomMinutes] = useState("");
  const firedRef = useRef({});

  const loadData = useCallback(async (t) => {
    setLoading(true); setError("");
    try {
      const [tasksData, projectsData] = await Promise.all([
        todoistFetch(t, "/tasks"), todoistFetch(t, "/projects"),
      ]);
      setTasks(tasksData); setProjects(projectsData);
    } catch { setError("API token geçersiz veya bağlantı hatası."); }
    finally { setLoading(false); }
  }, []);

  useEffect(() => { if (token) loadData(token); }, [token, loadData]);

  useEffect(() => {
    localStorage.setItem("todoist_reminders", JSON.stringify(reminders));
  }, [reminders]);

  const handleConnect = async () => {
    if (!tokenInput.trim()) return;
    localStorage.setItem("todoist_token", tokenInput.trim());
    setToken(tokenInput.trim());
  };

  const handleLogout = () => {
    localStorage.removeItem("todoist_token");
    setToken(""); setTasks([]); setProjects([]);
  };

  const requestNotifPermission = async () => {
    if (!("Notification" in window)) return;
    const perm = await Notification.requestPermission();
    setNotifPermission(perm);
  };

  useEffect(() => {
    if (!token || tasks.length === 0) return;
    const check = () => {
      const now = new Date();
      tasks.forEach((task) => {
        const due = getDueDate(task);
        if (!due) return;
        const autoLevels = [
          { key: "overdue", condition: due < now },
          { key: "1day", condition: due - now < 86400000 && due - now > 0 },
        ];
        autoLevels.forEach(({ key, condition }) => {
          const fk = `${task.id}-${key}`;
          if (condition && !firedRef.current[fk]) { firedRef.current[fk] = true; triggerAlert(task, key); }
        });
        (reminders[task.id] || []).forEach((mins) => {
          const targetTime = new Date(due.getTime() - mins * 60000);
          const fk = `${task.id}-custom-${mins}`;
          if (now >= targetTime && !firedRef.current[fk]) { firedRef.current[fk] = true; triggerAlert(task, `${mins}dk önce`); }
        });
      });
    };
    check();
    const interval = setInterval(check, 30000);
    return () => clearInterval(interval);
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [token, tasks, reminders]);

  const triggerAlert = (task, reason) => {
    const urgency = getUrgencyLevel(task);
    const popupId = Date.now() + Math.random();
    setPopups((prev) => [...prev, { id: popupId, title: task.content, reason, urgency, taskId: task.id }]);
    setTimeout(() => setPopups((prev) => prev.filter((p) => p.id !== popupId)), 8000);
    playAlertSound();
    if (Notification?.permission === "granted") {
      const reasonText = reason === "overdue" ? "Süresi geçti!" : reason === "1day" ? "1 gün kaldı" : `${reason} hatırlatıcı`;
      new Notification(`⏰ ${task.content}`, { body: reasonText, tag: `todoist-${task.id}-${reason}` });
    }
  };

  const addReminder = (taskId, minutes) => setReminders((prev) => ({ ...prev, [taskId]: [...new Set([...(prev[taskId] || []), minutes])] }));
  const removeReminder = (taskId, minutes) => setReminders((prev) => ({ ...prev, [taskId]: (prev[taskId] || []).filter((m) => m !== minutes) }));

  const handleComplete = async (taskId) => {
    setCompletingId(taskId);
    try {
      await todoistFetch(token, `/tasks/${taskId}/close`, { method: "POST" });
      setTasks((prev) => prev.filter((t) => t.id !== taskId));
      setPopups((prev) => prev.filter((p) => p.taskId !== taskId));
    } catch { setError("Görev tamamlanamadı."); }
    finally { setCompletingId(null); }
  };

  const handleAddTask = async () => {
    if (!newTask.content.trim()) return;
    setLoading(true);
    try {
      const body = { content: newTask.content, priority: Number(newTask.priority) };
      if (newTask.project_id) body.project_id = newTask.project_id;
      if (newTask.due_string) body.due_string = newTask.due_string;
      const created = await todoistFetch(token, "/tasks", { method: "POST", body: JSON.stringify(body) });
      setTasks((prev) => [created, ...prev]);
      setNewTask({ content: "", priority: 1, project_id: "", due_string: "" });
      setShowAddTask(false);
    } catch { setError("Görev eklenemedi."); }
    finally { setLoading(false); }
  };

  const handleUpdateTask = async () => {
    if (!editTask) return;
    setLoading(true);
    try {
      const body = { content: editTask.content, priority: Number(editTask.priority) };
      if (editTask.due_string) body.due_string = editTask.due_string;
      await todoistFetch(token, `/tasks/${editTask.id}`, { method: "POST", body: JSON.stringify(body) });
      setTasks((prev) => prev.map((t) => (t.id === editTask.id ? { ...t, ...body } : t)));
      setEditTask(null);
    } catch { setError("Görev güncellenemedi."); }
    finally { setLoading(false); }
  };

  const handleDeleteTask = async (taskId) => {
    try {
      await todoistFetch(token, `/tasks/${taskId}`, { method: "DELETE" });
      setTasks((prev) => prev.filter((t) => t.id !== taskId));
    } catch { setError("Görev silinemedi."); }
  };

  const handleAISuggest = async () => {
    if (!aiPrompt.trim()) return;
    setAiLoading(true); setAiSuggestions([]);
    try {
      const taskList = tasks.slice(0, 20).map((t) => `- ${t.content} (öncelik: ${t.priority})`).join("\n");
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514", max_tokens: 1000,
          messages: [{ role: "user", content: `Sen bir üretkenlik koçusun. Mevcut görevler:\n${taskList}\n\nİstek: "${aiPrompt}"\n\n3-5 görev öner. SADECE JSON array: [{"content":"...","priority":1-4,"due_string":"...veya boş"}]` }],
        }),
      });
      const data = await res.json();
      const text = data.content?.map((c) => c.text || "").join("") || "";
      setAiSuggestions(JSON.parse(text.replace(/```json|```/g, "").trim()));
    } catch { setError("AI önerileri alınamadı."); }
    finally { setAiLoading(false); }
  };

  const handleAddSuggestion = async (s) => {
    setLoading(true);
    try {
      const body = { content: s.content, priority: Number(s.priority) };
      if (s.due_string) body.due_string = s.due_string;
      const created = await todoistFetch(token, "/tasks", { method: "POST", body: JSON.stringify(body) });
      setTasks((prev) => [created, ...prev]);
      setAiSuggestions((prev) => prev.filter((x) => x.content !== s.content));
    } catch { setError("Görev eklenemedi."); }
    finally { setLoading(false); }
  };

  const filteredTasks = selectedProject === "all" ? tasks : tasks.filter((t) => t.project_id === selectedProject);
  const getProjectName = (id) => projects.find((p) => p.id === id)?.name || "Gelen Kutusu";
  const urgencyCounts = { overdue: 0, critical: 0, urgent: 0 };
  tasks.forEach((t) => { const u = getUrgencyLevel(t); if (urgencyCounts[u] !== undefined) urgencyCounts[u]++; });

  if (!token) {
    return (
      <div style={{ minHeight: "100vh", background: C.bg, display: "flex", alignItems: "center", justifyContent: "center", fontFamily: "Georgia, serif" }}>
        <style>{`@keyframes fadeIn{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:translateY(0)}}`}</style>
        <div style={{ width: 440, padding: "52px 44px", background: C.surface, border: `1px solid ${C.border}`, borderRadius: 20, animation: "fadeIn .4s ease" }}>
          <div style={{ display: "flex", alignItems: "center", gap: 12, marginBottom: 6 }}>
            <div style={{ width: 38, height: 38, background: C.accent, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 22 }}>✓</div>
            <h1 style={{ color: C.text, fontSize: 26, fontWeight: 700, margin: 0 }}>Todoist AI</h1>
          </div>
          <p style={{ color: C.textMuted, fontSize: 13, marginBottom: 36, marginTop: 4 }}>Akıllı görev yönetimi · Hatırlatmalar · AI önerileri</p>
          <label style={{ display: "block", color: C.textMuted, fontSize: 11, fontFamily: "monospace", letterSpacing: 1.5, marginBottom: 8 }}>API TOKEN</label>
          <input type="password" value={tokenInput} onChange={(e) => setTokenInput(e.target.value)} onKeyDown={(e) => e.key === "Enter" && handleConnect()} placeholder="Todoist API token..." style={{ width: "100%", padding: "13px 15px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 10, color: C.text, fontSize: 14, outline: "none", boxSizing: "border-box", marginBottom: 14 }} />
          <button onClick={handleConnect} style={{ width: "100%", padding: 13, background: C.accent, border: "none", borderRadius: 10, color: "#fff", fontSize: 15, fontWeight: 700, cursor: "pointer" }}>Bağlan →</button>
          {error && <p style={{ color: C.accentSoft, fontSize: 13, marginTop: 12, textAlign: "center" }}>{error}</p>}
          <p style={{ color: C.textMuted, fontSize: 12, marginTop: 24, lineHeight: 1.7 }}>Token: <strong style={{ color: C.text }}>Todoist → Ayarlar → Entegrasyonlar → Geliştirici</strong></p>
        </div>
      </div>
    );
  }

  return (
    <div style={{ minHeight: "100vh", background: C.bg, fontFamily: "Georgia, serif", color: C.text }}>
      <style>{`
        @keyframes slideIn{from{opacity:0;transform:translateX(110%)}to{opacity:1;transform:translateX(0)}}
        @keyframes fadeIn{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:translateY(0)}}
        @keyframes blink{0%,100%{box-shadow:0 0 12px #e4433266}50%{box-shadow:0 0 22px #e44332aa}}
        .task-row:hover{background:#1e1e2e!important;}
        .btn-ghost:hover{background:#252538!important;color:#eeeef5!important;}
      `}</style>

      {/* POPUPS */}
      <div style={{ position: "fixed", top: 20, right: 20, zIndex: 9999, display: "flex", flexDirection: "column", gap: 10, maxWidth: 340 }}>
        {popups.map((popup) => {
          const bgMap = { overdue: "#2e1212", critical: "#2e1a0a", urgent: "#2a2200", soon: C.surface };
          const borderMap = { overdue: C.accent, critical: C.orange, urgent: C.yellow, soon: C.blue };
          return (
            <div key={popup.id} style={{ background: bgMap[popup.urgency] || C.surface, border: `1.5px solid ${borderMap[popup.urgency] || C.border}`, borderRadius: 12, padding: "14px 16px", animation: "slideIn .35s cubic-bezier(.22,1,.36,1)", boxShadow: "0 8px 32px #00000088" }}>
              <div style={{ display: "flex", justifyContent: "space-between", gap: 10 }}>
                <div>
                  <div style={{ fontSize: 12, color: borderMap[popup.urgency], fontWeight: 700, marginBottom: 4, fontFamily: "monospace" }}>
                    {popup.urgency === "overdue" ? "⏰ SÜRESİ GEÇTİ" : popup.urgency === "critical" ? "🔥 ACİL" : "⚡ HATIRLATMA"}
                  </div>
                  <div style={{ fontSize: 14, color: C.text }}>{popup.title}</div>
                  <div style={{ fontSize: 11, color: C.textMuted, marginTop: 3 }}>{popup.reason}</div>
                </div>
                <button onClick={() => setPopups((prev) => prev.filter((p) => p.id !== popup.id))} style={{ background: "none", border: "none", color: C.textMuted, cursor: "pointer", fontSize: 16 }}>✕</button>
              </div>
            </div>
          );
        })}
      </div>

      {/* REMINDER MODAL */}
      {reminderModal && (
        <div style={{ position: "fixed", inset: 0, background: "#00000088", zIndex: 8888, display: "flex", alignItems: "center", justifyContent: "center" }} onClick={() => setReminderModal(null)}>
          <div style={{ background: C.surface, border: `1px solid ${C.border}`, borderRadius: 16, padding: 28, width: 360, animation: "fadeIn .25s ease" }} onClick={(e) => e.stopPropagation()}>
            <h3 style={{ margin: "0 0 6px", fontSize: 17 }}>⏰ Hatırlatıcı Ayarla</h3>
            <p style={{ color: C.textMuted, fontSize: 12, marginBottom: 20 }}>{tasks.find((t) => t.id === reminderModal)?.content}</p>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 8, marginBottom: 16 }}>
              {[{ label: "1 gün önce", mins: 1440 }, { label: "3 saat önce", mins: 180 }, { label: "1 saat önce", mins: 60 }, { label: "30 dk önce", mins: 30 }, { label: "10 dk önce", mins: 10 }].map(({ label, mins }) => {
                const active = (reminders[reminderModal] || []).includes(mins);
                return (
                  <button key={mins} onClick={() => active ? removeReminder(reminderModal, mins) : addReminder(reminderModal, mins)}
                    style={{ padding: "7px 14px", borderRadius: 8, border: `1px solid ${active ? C.green : C.border}`, background: active ? "#122e20" : "transparent", color: active ? C.green : C.textMuted, fontSize: 12, cursor: "pointer", fontWeight: active ? 700 : 400 }}>
                    {active ? "✓ " : ""}{label}
                  </button>
                );
              })}
            </div>
            <div style={{ display: "flex", gap: 8, marginBottom: 20 }}>
              <input value={customMinutes} onChange={(e) => setCustomMinutes(e.target.value)} placeholder="Özel (dakika önce)..." type="number" min="1"
                style={{ flex: 1, padding: "9px 12px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 13, outline: "none" }} />
              <button onClick={() => { if (customMinutes > 0) { addReminder(reminderModal, Number(customMinutes)); setCustomMinutes(""); } }}
                style={{ padding: "9px 16px", background: C.accent, border: "none", borderRadius: 8, color: "#fff", fontSize: 13, cursor: "pointer" }}>Ekle</button>
            </div>
            {(reminders[reminderModal] || []).length > 0 && (
              <div style={{ marginBottom: 16 }}>
                <div style={{ fontSize: 11, color: C.textMuted, fontFamily: "monospace", letterSpacing: 1, marginBottom: 8 }}>AKTİF HATIRLATICILAR</div>
                <div style={{ display: "flex", flexWrap: "wrap", gap: 6 }}>
                  {(reminders[reminderModal] || []).sort((a, b) => b - a).map((m) => (
                    <span key={m} style={{ padding: "4px 10px", background: "#122e20", border: `1px solid ${C.green}`, borderRadius: 6, fontSize: 12, color: C.green, display: "flex", alignItems: "center", gap: 6 }}>
                      {m >= 1440 ? `${m / 1440} gün` : m >= 60 ? `${Math.floor(m / 60)} saat` : `${m} dk`} önce
                      <button onClick={() => removeReminder(reminderModal, m)} style={{ background: "none", border: "none", color: C.textMuted, cursor: "pointer", fontSize: 14, padding: 0 }}>×</button>
                    </span>
                  ))}
                </div>
              </div>
            )}
            <button onClick={() => setReminderModal(null)} style={{ width: "100%", padding: 10, background: C.accent, border: "none", borderRadius: 8, color: "#fff", fontSize: 14, fontWeight: 600, cursor: "pointer" }}>Tamam</button>
          </div>
        </div>
      )}

      {/* HEADER */}
      <div style={{ borderBottom: `1px solid ${C.border}`, padding: "15px 28px", display: "flex", alignItems: "center", justifyContent: "space-between", gap: 12, flexWrap: "wrap" }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 32, height: 32, background: C.accent, borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 17 }}>✓</div>
          <span style={{ fontSize: 20, fontWeight: 700 }}>Todoist AI</span>
          {(urgencyCounts.overdue + urgencyCounts.critical) > 0 && (
            <span style={{ padding: "3px 9px", background: C.accent, borderRadius: 10, fontSize: 11, fontWeight: 700, animation: "blink 1.5s infinite" }}>
              {urgencyCounts.overdue + urgencyCounts.critical} ACİL
            </span>
          )}
        </div>
        <div style={{ display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" }}>
          {notifPermission !== "granted" && (
            <button onClick={requestNotifPermission} style={{ padding: "7px 14px", background: "#1a2a3a", border: `1px solid ${C.blue}`, borderRadius: 8, color: C.blue, fontSize: 12, cursor: "pointer", fontWeight: 600 }}>🔔 Bildirimlere İzin Ver</button>
          )}
          {notifPermission === "granted" && <span style={{ fontSize: 12, color: C.green }}>🔔 Bildirimler Açık</span>}
          {["tasks", "ai"].map((tab) => (
            <button key={tab} onClick={() => setActiveTab(tab)} className="btn-ghost"
              style={{ padding: "7px 16px", borderRadius: 8, border: "none", cursor: "pointer", fontSize: 13, fontWeight: 600, background: activeTab === tab ? C.accent : "transparent", color: activeTab === tab ? "#fff" : C.textMuted, transition: "all .2s" }}>
              {tab === "tasks" ? "📋 Görevler" : "✨ AI"}
            </button>
          ))}
          <button onClick={handleLogout} className="btn-ghost" style={{ padding: "7px 12px", borderRadius: 8, border: `1px solid ${C.border}`, cursor: "pointer", fontSize: 12, background: "transparent", color: C.textMuted }}>Çıkış</button>
        </div>
      </div>

      {/* URGENCY BAR */}
      {(urgencyCounts.overdue > 0 || urgencyCounts.critical > 0 || urgencyCounts.urgent > 0) && (
        <div style={{ padding: "10px 28px", borderBottom: `1px solid ${C.border}`, display: "flex", gap: 12, flexWrap: "wrap" }}>
          {urgencyCounts.overdue > 0 && <span style={{ fontSize: 12, padding: "4px 12px", background: "#2e1212", border: `1px solid ${C.accent}`, borderRadius: 8, color: C.accent }}>⏰ {urgencyCounts.overdue} gecikmiş</span>}
          {urgencyCounts.critical > 0 && <span style={{ fontSize: 12, padding: "4px 12px", background: "#2e1a0a", border: `1px solid ${C.orange}`, borderRadius: 8, color: C.orange }}>🔥 {urgencyCounts.critical} kritik</span>}
          {urgencyCounts.urgent > 0 && <span style={{ fontSize: 12, padding: "4px 12px", background: "#2a2200", border: `1px solid ${C.yellow}`, borderRadius: 8, color: C.yellow }}>⚡ {urgencyCounts.urgent} bugün</span>}
        </div>
      )}

      <div style={{ maxWidth: 920, margin: "0 auto", padding: "24px 20px" }}>
        {error && (
          <div style={{ background: "#2e1212", border: `1px solid ${C.accent}`, borderRadius: 8, padding: "10px 16px", marginBottom: 18, color: C.accentSoft, fontSize: 13 }}>
            ⚠️ {error} <button onClick={() => setError("")} style={{ float: "right", background: "none", border: "none", color: C.accentSoft, cursor: "pointer" }}>✕</button>
          </div>
        )}

        {activeTab === "tasks" && (
          <>
            <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 18, flexWrap: "wrap", gap: 10 }}>
              <div style={{ display: "flex", gap: 6, flexWrap: "wrap" }}>
                <button onClick={() => setSelectedProject("all")} style={{ padding: "5px 13px", borderRadius: 20, border: "none", cursor: "pointer", fontSize: 12, background: selectedProject === "all" ? C.accent : C.surface, color: selectedProject === "all" ? "#fff" : C.textMuted }}>
                  Tümü ({tasks.length})
                </button>
                {projects.map((p) => (
                  <button key={p.id} onClick={() => setSelectedProject(p.id)} style={{ padding: "5px 13px", borderRadius: 20, border: "none", cursor: "pointer", fontSize: 12, background: selectedProject === p.id ? C.accent : C.surface, color: selectedProject === p.id ? "#fff" : C.textMuted }}>
                    {p.name} ({tasks.filter((t) => t.project_id === p.id).length})
                  </button>
                ))}
              </div>
              <button onClick={() => setShowAddTask(true)} style={{ padding: "8px 18px", background: C.accent, border: "none", borderRadius: 8, color: "#fff", fontSize: 13, fontWeight: 700, cursor: "pointer" }}>+ Görev Ekle</button>
            </div>

            {showAddTask && (
              <div style={{ background: C.surface, border: `1px solid ${C.border}`, borderRadius: 12, padding: 22, marginBottom: 18, animation: "fadeIn .2s ease" }}>
                <h3 style={{ margin: "0 0 14px", fontSize: 15 }}>Yeni Görev</h3>
                <input value={newTask.content} onChange={(e) => setNewTask((p) => ({ ...p, content: e.target.value }))} placeholder="Görev adı..." style={{ width: "100%", padding: "10px 12px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 14, marginBottom: 10, boxSizing: "border-box", outline: "none" }} />
                <div style={{ display: "flex", gap: 8, marginBottom: 10, flexWrap: "wrap" }}>
                  <select value={newTask.priority} onChange={(e) => setNewTask((p) => ({ ...p, priority: e.target.value }))} style={{ flex: 1, minWidth: 120, padding: "8px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 13, outline: "none" }}>
                    <option value={4}>🔴 P1 - Acil</option><option value={3}>🟠 P2 - Yüksek</option><option value={2}>🔵 P3 - Orta</option><option value={1}>⚪ P4 - Düşük</option>
                  </select>
                  <select value={newTask.project_id} onChange={(e) => setNewTask((p) => ({ ...p, project_id: e.target.value }))} style={{ flex: 1, minWidth: 120, padding: "8px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 13, outline: "none" }}>
                    <option value="">Proje seç...</option>
                    {projects.map((p) => <option key={p.id} value={p.id}>{p.name}</option>)}
                  </select>
                  <input value={newTask.due_string} onChange={(e) => setNewTask((p) => ({ ...p, due_string: e.target.value }))} placeholder="Tarih (yarın, Pazartesi...)" style={{ flex: 1, minWidth: 140, padding: "8px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 13, outline: "none" }} />
                </div>
                <div style={{ display: "flex", gap: 8 }}>
                  <button onClick={handleAddTask} style={{ padding: "8px 20px", background: C.accent, border: "none", borderRadius: 8, color: "#fff", fontSize: 13, fontWeight: 600, cursor: "pointer" }}>Ekle</button>
                  <button onClick={() => setShowAddTask(false)} style={{ padding: "8px 14px", background: "transparent", border: `1px solid ${C.border}`, borderRadius: 8, color: C.textMuted, fontSize: 13, cursor: "pointer" }}>İptal</button>
                </div>
              </div>
            )}

            {loading ? (
              <div style={{ textAlign: "center", padding: 60, color: C.textMuted }}>Yükleniyor...</div>
            ) : filteredTasks.length === 0 ? (
              <div style={{ textAlign: "center", padding: 60, color: C.textMuted }}><div style={{ fontSize: 40, marginBottom: 10 }}>🎉</div>Tüm görevler tamamlandı!</div>
            ) : (
              <div style={{ display: "flex", flexDirection: "column", gap: 7 }}>
                {[...filteredTasks].sort((a, b) => {
                  const order = { overdue: 0, critical: 1, urgent: 2, soon: 3, normal: 4, none: 5 };
                  return (order[getUrgencyLevel(a)] - order[getUrgencyLevel(b)]) || (b.priority - a.priority);
                }).map((task) => {
                  const pc = priorityConfig[task.priority] || priorityConfig[1];
                  const urg = getUrgencyLevel(task);
                  const us = urgencyStyle[urg];
                  const hasReminders = (reminders[task.id] || []).length > 0;

                  return editTask?.id === task.id ? (
                    <div key={task.id} style={{ background: C.surface, border: `1.5px solid ${C.accent}`, borderRadius: 10, padding: 16 }}>
                      <input value={editTask.content} onChange={(e) => setEditTask((p) => ({ ...p, content: e.target.value }))} style={{ width: "100%", padding: "8px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 6, color: C.text, fontSize: 14, marginBottom: 8, boxSizing: "border-box", outline: "none" }} />
                      <div style={{ display: "flex", gap: 8, marginBottom: 10 }}>
                        <select value={editTask.priority} onChange={(e) => setEditTask((p) => ({ ...p, priority: e.target.value }))} style={{ flex: 1, padding: "7px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 6, color: C.text, fontSize: 12, outline: "none" }}>
                          <option value={4}>🔴 P1</option><option value={3}>🟠 P2</option><option value={2}>🔵 P3</option><option value={1}>⚪ P4</option>
                        </select>
                        <input value={editTask.due_string || ""} onChange={(e) => setEditTask((p) => ({ ...p, due_string: e.target.value }))} placeholder="Tarih..." style={{ flex: 1, padding: "7px 10px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 6, color: C.text, fontSize: 12, outline: "none" }} />
                      </div>
                      <div style={{ display: "flex", gap: 8 }}>
                        <button onClick={handleUpdateTask} style={{ padding: "6px 16px", background: C.green, border: "none", borderRadius: 6, color: "#fff", fontSize: 12, cursor: "pointer" }}>Kaydet</button>
                        <button onClick={() => setEditTask(null)} style={{ padding: "6px 12px", background: "transparent", border: `1px solid ${C.border}`, borderRadius: 6, color: C.textMuted, fontSize: 12, cursor: "pointer" }}>İptal</button>
                      </div>
                    </div>
                  ) : (
                    <div key={task.id} className="task-row" style={{ background: C.surface, border: us.border, borderRadius: 10, padding: "13px 15px", display: "flex", alignItems: "flex-start", gap: 12, boxShadow: us.glow, transition: "all .2s" }}>
                      <button onClick={() => handleComplete(task.id)} disabled={completingId === task.id} style={{ width: 21, height: 21, borderRadius: "50%", border: `2px solid ${pc.color}`, background: "transparent", cursor: "pointer", flexShrink: 0, marginTop: 2 }} />
                      <div style={{ flex: 1, minWidth: 0 }}>
                        <div style={{ fontSize: 14, color: C.text, marginBottom: 5 }}>{task.content}</div>
                        <div style={{ display: "flex", gap: 7, flexWrap: "wrap", alignItems: "center" }}>
                          <span style={{ fontSize: 10, padding: "2px 7px", borderRadius: 4, background: pc.bg, color: pc.color, fontWeight: 700 }}>{pc.label}</span>
                          <span style={{ fontSize: 11, color: C.textMuted }}>{getProjectName(task.project_id)}</span>
                          {task.due?.string && <span style={{ fontSize: 11, color: urg === "overdue" ? C.accent : urg === "critical" ? C.orange : urg === "urgent" ? C.yellow : C.textMuted }}>📅 {task.due.string}</span>}
                          {us.badge && <span style={{ fontSize: 10, padding: "2px 8px", borderRadius: 4, background: us.badge.bg, color: us.badge.text, fontWeight: 700 }}>{us.badge.label}</span>}
                          {hasReminders && <span style={{ fontSize: 10, color: C.green }}>🔔 {reminders[task.id].length} hatırlatıcı</span>}
                        </div>
                      </div>
                      <div style={{ display: "flex", gap: 5, flexShrink: 0 }}>
                        <button onClick={() => setReminderModal(task.id)} style={{ padding: "4px 9px", background: hasReminders ? "#122e20" : "transparent", border: `1px solid ${hasReminders ? C.green : C.border}`, borderRadius: 6, color: hasReminders ? C.green : C.textMuted, fontSize: 12, cursor: "pointer" }}>🔔</button>
                        <button onClick={() => setEditTask(task)} style={{ padding: "4px 9px", background: "transparent", border: `1px solid ${C.border}`, borderRadius: 6, color: C.textMuted, fontSize: 12, cursor: "pointer" }}>✏️</button>
                        <button onClick={() => handleDeleteTask(task.id)} style={{ padding: "4px 9px", background: "transparent", border: `1px solid ${C.border}`, borderRadius: 6, color: C.textMuted, fontSize: 12, cursor: "pointer" }}>🗑️</button>
                      </div>
                    </div>
                  );
                })}
              </div>
            )}
          </>
        )}

        {activeTab === "ai" && (
          <div>
            <div style={{ background: C.surface, border: `1px solid ${C.border}`, borderRadius: 12, padding: 24, marginBottom: 22 }}>
              <h3 style={{ margin: "0 0 6px", fontSize: 18 }}>✨ AI Görev Önerileri</h3>
              <p style={{ color: C.textMuted, fontSize: 13, margin: "0 0 18px" }}>Görevlerinizi analiz ederek akıllı öneriler sunuyorum.</p>
              <div style={{ display: "flex", gap: 10 }}>
                <input value={aiPrompt} onChange={(e) => setAiPrompt(e.target.value)} onKeyDown={(e) => e.key === "Enter" && handleAISuggest()} placeholder="Örn: 'Sağlık hedeflerim için görevler öner'..." style={{ flex: 1, padding: "11px 14px", background: C.bg, border: `1px solid ${C.border}`, borderRadius: 8, color: C.text, fontSize: 14, outline: "none" }} />
                <button onClick={handleAISuggest} disabled={aiLoading} style={{ padding: "11px 22px", background: aiLoading ? C.border : C.accent, border: "none", borderRadius: 8, color: "#fff", fontSize: 14, fontWeight: 700, cursor: aiLoading ? "default" : "pointer" }}>
                  {aiLoading ? "⏳..." : "Öner"}
                </button>
              </div>
            </div>
            {aiSuggestions.length > 0 && (
              <div style={{ display: "flex", flexDirection: "column", gap: 9 }}>
                {aiSuggestions.map((s, i) => {
                  const pc = priorityConfig[s.priority] || priorityConfig[1];
                  return (
                    <div key={i} style={{ background: C.surface, border: `1px solid ${C.border}`, borderRadius: 10, padding: "13px 15px", display: "flex", alignItems: "center", gap: 12 }}>
                      <div style={{ flex: 1 }}>
                        <div style={{ fontSize: 14, marginBottom: 5 }}>{s.content}</div>
                        <div style={{ display: "flex", gap: 8 }}>
                          <span style={{ fontSize: 10, padding: "2px 7px", borderRadius: 4, background: pc.bg, color: pc.color, fontWeight: 700 }}>{pc.label}</span>
                          {s.due_string && <span style={{ fontSize: 11, color: C.yellow }}>📅 {s.due_string}</span>}
                        </div>
                      </div>
                      <button onClick={() => handleAddSuggestion(s)} style={{ padding: "7px 16px", background: C.green, border: "none", borderRadius: 8, color: "#fff", fontSize: 13, fontWeight: 600, cursor: "pointer" }}>+ Ekle</button>
                    </div>
                  );
                })}
              </div>
            )}
            {!aiLoading && aiSuggestions.length === 0 && (
              <div style={{ textAlign: "center", padding: 60, color: C.textMuted }}>
                <div style={{ fontSize: 44, marginBottom: 10 }}>🤖</div>
                <p>Yukarıya bir istek yazın, AI görevlerinizi analiz etsin!</p>
              </div>
            )}
          </div>
        )}
      </div>
    </div>
  );
}
