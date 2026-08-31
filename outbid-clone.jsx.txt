import { useState, useEffect, useRef } from "react";
import { Crown, Zap, X, ArrowUp, Flame, Skull } from "lucide-react";

const FONT_IMPORT = `@import url('https://fonts.googleapis.com/css2?family=Bebas+Neue&family=JetBrains+Mono:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap');`;

const STARTER_ENTRIES = [
  { id: "seed-1", name: "Nebula Notes", url: "nebulanotes.app", total: 340, boosts: 6, createdAt: Date.now() - 900000 },
  { id: "seed-2", name: "Forkwise", url: "forkwise.dev", total: 210, boosts: 3, createdAt: Date.now() - 700000 },
  { id: "seed-3", name: "Loopcraft", url: "loopcraft.co", total: 95, boosts: 2, createdAt: Date.now() - 500000 },
  { id: "seed-4", name: "Driftline", url: "driftline.io", total: 40, boosts: 1, createdAt: Date.now() - 300000 },
  { id: "seed-5", name: "Hushbox", url: "hushbox.xyz", total: 12, boosts: 1, createdAt: Date.now() - 100000 },
];

function money(n) {
  return "$" + Number(n).toLocaleString("en-US");
}

function sortEntries(list) {
  return [...list].sort((a, b) => b.total - a.total || a.createdAt - b.createdAt);
}

export default function OutrankArena() {
  const [entries, setEntries] = useState([]);
  const [loaded, setLoaded] = useState(false);
  const [mode, setMode] = useState(null); // null | "claim" | { boostId }
  const [name, setName] = useState("");
  const [url, setUrl] = useState("");
  const [amount, setAmount] = useState("");
  const [err, setErr] = useState("");
  const [flash, setFlash] = useState(null); // id of entry to flash gold
  const [dethroned, setDethroned] = useState(null); // name of previous #1
  const prevTopRef = useRef(null);

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get("arena-entries", true);
        if (res && res.value) {
          const parsed = JSON.parse(res.value);
          setEntries(sortEntries(parsed));
          prevTopRef.current = sortEntries(parsed)[0]?.id || null;
        } else {
          setEntries(sortEntries(STARTER_ENTRIES));
          prevTopRef.current = sortEntries(STARTER_ENTRIES)[0]?.id;
        }
      } catch (e) {
        setEntries(sortEntries(STARTER_ENTRIES));
      } finally {
        setLoaded(true);
      }
    })();
  }, []);

  async function persist(next) {
    const sorted = sortEntries(next);
    setEntries(sorted);
    try {
      await window.storage.set("arena-entries", JSON.stringify(sorted), true);
    } catch (e) {
      // best effort; local state already updated
    }
    const newTop = sorted[0];
    if (prevTopRef.current && newTop && newTop.id !== prevTopRef.current) {
      const prevEntry = next.find((e) => e.id === prevTopRef.current);
      setDethroned(prevEntry ? prevEntry.name : null);
      setTimeout(() => setDethroned(null), 2600);
    }
    prevTopRef.current = newTop?.id || null;
    setFlash(newTop?.id || null);
    setTimeout(() => setFlash(null), 900);
  }

  function openClaim() {
    setName("");
    setUrl("");
    setAmount("");
    setErr("");
    setMode("claim");
  }

  function openBoost(id) {
    setAmount("");
    setErr("");
    setMode({ boostId: id });
  }

  function closeModal() {
    setMode(null);
  }

  async function submitClaim(e) {
    e.preventDefault();
    const amt = Math.floor(Number(amount));
    if (!name.trim() || !url.trim()) {
      setErr("Enter a name and a URL.");
      return;
    }
    if (!amt || amt < 5) {
      setErr("Minimum claim is $5.");
      return;
    }
    const entry = {
      id: "e-" + Date.now() + "-" + Math.random().toString(36).slice(2, 7),
      name: name.trim(),
      url: url.trim().replace(/^https?:\/\//, ""),
      total: amt,
      boosts: 1,
      createdAt: Date.now(),
    };
    await persist([...entries, entry]);
    closeModal();
  }

  async function submitBoost(e, id) {
    e.preventDefault();
    const amt = Math.floor(Number(amount));
    if (!amt || amt < 1) {
      setErr("Enter at least $1.");
      return;
    }
    const next = entries.map((en) =>
      en.id === id ? { ...en, total: en.total + amt, boosts: en.boosts + 1 } : en
    );
    await persist(next);
    closeModal();
  }

  const pot = entries.reduce((sum, e) => sum + e.total, 0);
  const top = entries[0];

  return (
    <div style={styles.page}>
      <style>{`
        ${FONT_IMPORT}
        * { box-sizing: border-box; }
        .oa-root { font-family: 'Inter', sans-serif; }
        .oa-display { font-family: 'Bebas Neue', sans-serif; letter-spacing: 0.02em; }
        .oa-mono { font-family: 'JetBrains Mono', monospace; }
        .oa-row { transition: transform 220ms ease, box-shadow 220ms ease, border-color 220ms ease; }
        .oa-row:hover { transform: translateX(2px); }
        .oa-btn { transition: transform 120ms ease, background 120ms ease, box-shadow 120ms ease; cursor: pointer; }
        .oa-btn:active { transform: scale(0.96); }
        .oa-flash { animation: oaflash 900ms ease; }
        @keyframes oaflash {
          0% { box-shadow: 0 0 0 0 rgba(242,183,5,0.9); }
          40% { box-shadow: 0 0 0 10px rgba(242,183,5,0.15); }
          100% { box-shadow: 0 0 0 0 rgba(242,183,5,0); }
        }
        .oa-toast { animation: oaToastIn 300ms ease, oaToastOut 300ms ease 2200ms; }
        @keyframes oaToastIn { from { opacity: 0; transform: translateY(-10px); } to { opacity: 1; transform: translateY(0); } }
        @keyframes oaToastOut { from { opacity: 1; } to { opacity: 0; } }
        .oa-pulse-dot { animation: oaPulse 1.6s ease-in-out infinite; }
        @keyframes oaPulse { 0%,100% { opacity: 1; } 50% { opacity: 0.35; } }
        .oa-modal-backdrop { animation: oaFadeIn 180ms ease; }
        @keyframes oaFadeIn { from { opacity: 0; } to { opacity: 1; } }
        .oa-modal-card { animation: oaSlideUp 220ms cubic-bezier(.2,.8,.2,1); }
        @keyframes oaSlideUp { from { opacity: 0; transform: translateY(16px) scale(0.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
        .oa-input:focus { outline: none; border-color: #F2B705 !important; box-shadow: 0 0 0 3px rgba(242,183,5,0.18); }
        .oa-btn:focus-visible { outline: 2px solid #F2B705; outline-offset: 2px; }
        @media (prefers-reduced-motion: reduce) {
          .oa-row, .oa-btn, .oa-flash, .oa-toast, .oa-pulse-dot, .oa-modal-backdrop, .oa-modal-card { animation: none !important; transition: none !important; }
        }
      `}</style>

      <div className="oa-root" style={{ maxWidth: 480, margin: "0 auto", padding: "20px 14px 40px" }}>
        {/* Header */}
        <div style={{ textAlign: "center", marginBottom: 22 }}>
          <div style={{ display: "inline-flex", alignItems: "center", gap: 6, background: "#151922", border: "1px solid #262B36", borderRadius: 999, padding: "5px 12px", marginBottom: 14 }}>
            <span className="oa-pulse-dot" style={{ width: 6, height: 6, borderRadius: 999, background: "#F2B705", display: "inline-block" }} />
            <span className="oa-mono" style={{ fontSize: 11, color: "#9CA3AF", letterSpacing: "0.05em" }}>LIVE — RANK IS THE BID</span>
          </div>
          <h1 className="oa-display" style={{ fontSize: 56, lineHeight: 0.95, color: "#F5F3EE", margin: 0 }}>
            THE ARENA
          </h1>
          <p style={{ color: "#8B90A0", fontSize: 14.5, marginTop: 8, lineHeight: 1.5 }}>
            No merit. No algorithm. Whoever pays the most owns the top of this page.
          </p>

          <div style={{ marginTop: 18, background: "linear-gradient(180deg, #151922 0%, #10131A 100%)", border: "1px solid #262B36", borderRadius: 14, padding: "16px 18px" }}>
            <div className="oa-mono" style={{ fontSize: 11, color: "#6B7280", letterSpacing: "0.08em", marginBottom: 4 }}>TOTAL PAID SO FAR</div>
            <div className="oa-display" style={{ fontSize: 40, color: "#F2B705", lineHeight: 1 }}>{money(pot)}</div>
          </div>
        </div>

        {/* Dethrone toast */}
        {dethroned && (
          <div className="oa-toast" style={{ display: "flex", alignItems: "center", gap: 8, background: "#2A1414", border: "1px solid #5C2323", borderRadius: 10, padding: "10px 12px", marginBottom: 14 }}>
            <Skull size={16} color="#FF6B6B" />
            <span style={{ fontSize: 13, color: "#FFB4B4" }}>
              <strong style={{ color: "#FF6B6B" }}>{dethroned}</strong> just got dethroned from #1.
            </span>
          </div>
        )}

        {/* Claim button */}
        <button
          className="oa-btn"
          onClick={openClaim}
          style={{
            width: "100%",
            background: "#F2B705",
            color: "#151107",
            border: "none",
            borderRadius: 12,
            padding: "14px 16px",
            fontSize: 15,
            fontWeight: 700,
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            gap: 8,
            marginBottom: 22,
            boxShadow: "0 6px 20px rgba(242,183,5,0.18)",
          }}
        >
          <Zap size={17} strokeWidth={2.5} />
          Claim a spot — from $5
        </button>

        {/* Leaderboard */}
        {!loaded ? (
          <div style={{ textAlign: "center", color: "#6B7280", padding: "40px 0", fontSize: 13 }} className="oa-mono">
            loading leaderboard…
          </div>
        ) : (
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            {entries.map((entry, i) => {
              const rank = i + 1;
              const isTop = rank === 1;
              return (
                <div
                  key={entry.id}
                  className={"oa-row" + (flash === entry.id ? " oa-flash" : "")}
                  style={{
                    display: "flex",
                    alignItems: "center",
                    gap: 12,
                    background: isTop ? "linear-gradient(90deg, rgba(242,183,5,0.10), rgba(21,25,34,0.6))" : "#151922",
                    border: isTop ? "1px solid #4A3D0F" : "1px solid #21252F",
                    borderRadius: 12,
                    padding: "12px 14px",
                  }}
                >
                  <div style={{ width: 38, textAlign: "center", flexShrink: 0 }}>
                    {isTop ? (
                      <Crown size={22} color="#F2B705" fill="#F2B705" />
                    ) : (
                      <span className="oa-display" style={{ fontSize: 26, color: rank <= 3 ? "#D9DBE3" : "#565B68" }}>
                        {rank}
                      </span>
                    )}
                  </div>

                  <div style={{ flex: 1, minWidth: 0 }}>
                    <div style={{ fontSize: 14.5, fontWeight: 600, color: "#F0EEE8", whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>
                      {entry.name}
                    </div>
                    <div className="oa-mono" style={{ fontSize: 11.5, color: "#6B7280", whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>
                      {entry.url}
                    </div>
                  </div>

                  <div style={{ textAlign: "right", flexShrink: 0 }}>
                    <div className="oa-mono" style={{ fontSize: 15, fontWeight: 700, color: isTop ? "#F2B705" : "#D9DBE3" }}>
                      {money(entry.total)}
                    </div>
                    <button
                      className="oa-btn"
                      onClick={() => openBoost(entry.id)}
                      style={{
                        marginTop: 4,
                        background: "transparent",
                        border: "1px solid #363B47",
                        borderRadius: 7,
                        padding: "3px 8px",
                        fontSize: 11,
                        color: "#9CA3AF",
                        display: "inline-flex",
                        alignItems: "center",
                        gap: 3,
                      }}
                    >
                      <ArrowUp size={11} /> boost
                    </button>
                  </div>
                </div>
              );
            })}
          </div>
        )}

        <p style={{ textAlign: "center", color: "#4B5063", fontSize: 11.5, marginTop: 24, lineHeight: 1.6 }}>
          Demo prototype — inspired by outbid.lol. No real payments are processed;<br />amounts are simulated and stored for this arena only.
        </p>
      </div>

      {/* Modal */}
      {mode && (
        <div
          className="oa-modal-backdrop"
          onClick={closeModal}
          style={{ position: "fixed", inset: 0, background: "rgba(6,7,10,0.75)", display: "flex", alignItems: "flex-end", justifyContent: "center", zIndex: 50 }}
        >
          <div
            className="oa-modal-card"
            onClick={(e) => e.stopPropagation()}
            style={{ width: "100%", maxWidth: 480, background: "#12151C", border: "1px solid #262B36", borderTopLeftRadius: 18, borderTopRightRadius: 18, padding: "20px 18px 28px" }}
          >
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14 }}>
              <h2 className="oa-display" style={{ fontSize: 24, color: "#F5F3EE", margin: 0, display: "flex", alignItems: "center", gap: 8 }}>
                {mode === "claim" ? (<><Flame size={18} color="#F2B705" /> Claim a spot</>) : (<><ArrowUp size={18} color="#F2B705" /> Boost your rank</>)}
              </h2>
              <button className="oa-btn" onClick={closeModal} style={{ background: "transparent", border: "none", color: "#6B7280" }}>
                <X size={20} />
              </button>
            </div>

            <form onSubmit={mode === "claim" ? submitClaim : (e) => submitBoost(e, mode.boostId)}>
              {mode === "claim" && (
                <>
                  <label style={fieldLabel}>Name</label>
                  <input className="oa-input" value={name} onChange={(e) => setName(e.target.value)} placeholder="Your website or product" style={inputStyle} />
                  <label style={fieldLabel}>URL</label>
                  <input className="oa-input" value={url} onChange={(e) => setUrl(e.target.value)} placeholder="yoursite.com" style={inputStyle} />
                </>
              )}
              <label style={fieldLabel}>{mode === "claim" ? "Opening bid (min $5)" : "Add to your total ($)"}</label>
              <input
                className="oa-input"
                type="number"
                min={mode === "claim" ? 5 : 1}
                value={amount}
                onChange={(e) => setAmount(e.target.value)}
                placeholder={mode === "claim" ? "5" : "1"}
                style={inputStyle}
              />
              {err && <div style={{ color: "#FF6B6B", fontSize: 12.5, marginTop: 6 }}>{err}</div>}
              <button
                className="oa-btn"
                type="submit"
                style={{ width: "100%", marginTop: 16, background: "#F2B705", color: "#151107", border: "none", borderRadius: 10, padding: "13px", fontSize: 14.5, fontWeight: 700 }}
              >
                {mode === "claim" ? "Pay & claim spot" : "Pay & boost"}
              </button>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}

const fieldLabel = { display: "block", fontSize: 11.5, color: "#8B90A0", marginTop: 12, marginBottom: 5, letterSpacing: "0.03em" };
const inputStyle = {
  width: "100%",
  background: "#0D0F14",
  border: "1px solid #2A2F3A",
  borderRadius: 9,
  padding: "11px 12px",
  fontSize: 14.5,
  color: "#F0EEE8",
};

const styles = {
  page: { background: "#0B0E14", minHeight: "100vh" },
};
