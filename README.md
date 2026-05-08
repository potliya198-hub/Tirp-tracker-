import { useState, useEffect } from "react";

const COLORS = ["#6366f1","#ec4899","#f97316","#22c55e","#06b6d4","#a855f7","#f59e0b","#ef4444"];

function getInitials(name) {
  return name.split(" ").map(w => w[0]).join("").toUpperCase().slice(0,2);
}

function calculateBalances(members, expenses) {
  const paid = {}, share = {};
  members.forEach(m => { paid[m] = 0; share[m] = 0; });
  expenses.forEach(exp => {
    paid[exp.paidBy] = (paid[exp.paidBy] || 0) + exp.amount;
    const forMembers = exp.forMembers.length > 0 ? exp.forMembers : members;
    const perPerson = exp.amount / forMembers.length;
    forMembers.forEach(m => { share[m] = (share[m] || 0) + perPerson; });
  });
  const balance = {};
  members.forEach(m => { balance[m] = Math.round((paid[m] - share[m]) * 100) / 100; });
  return balance;
}

function calculateSettlements(members, expenses) {
  const bal = calculateBalances(members, expenses);
  const creditors = [], debtors = [];
  Object.entries(bal).forEach(([m, b]) => {
    if (b > 0.01) creditors.push({ name: m, amt: b });
    else if (b < -0.01) debtors.push({ name: m, amt: -b });
  });
  creditors.sort((a,b) => b.amt - a.amt);
  debtors.sort((a,b) => b.amt - a.amt);
  const settlements = [];
  let i = 0, j = 0;
  while (i < creditors.length && j < debtors.length) {
    const amt = Math.min(creditors[i].amt, debtors[j].amt);
    if (amt > 0.01) settlements.push({ from: debtors[j].name, to: creditors[i].name, amt: Math.round(amt) });
    creditors[i].amt -= amt; debtors[j].amt -= amt;
    if (creditors[i].amt < 0.01) i++;
    if (debtors[j].amt < 0.01) j++;
  }
  return settlements;
}

export default function TripTracker() {
  const [trips, setTrips] = useState(() => JSON.parse(localStorage.getItem("trips-data") || "[]"));
  const [page, setPage] = useState("home"); // home, newTrip, trip, newExp
  const [curTrip, setCurTrip] = useState(null);
  const [tripName, setTripName] = useState("");
  const [members, setMembers] = useState([""]);
  const [expDesc, setExpDesc] = useState("");
  const [expAmt, setExpAmt] = useState("");
  const [expPaidBy, setExpPaidBy] = useState("");
  const [expFor, setExpFor] = useState([]);
  const [activeTab, setActiveTab] = useState("expenses");

  useEffect(() => { localStorage.setItem("trips-data", JSON.stringify(trips)); }, [trips]);

  const saveTrips = (t) => { setTrips(t); localStorage.setItem("trips-data", JSON.stringify(t)); };

  const createTrip = () => {
    const mems = members.filter(m => m.trim());
    if (!tripName.trim() || mems.length < 2) return;
    const trip = { id: Date.now(), name: tripName.trim(), members: mems, expenses: [], createdAt: new Date().toLocaleDateString("en-IN") };
    saveTrips([trip, ...trips]);
    setTripName(""); setMembers([""]);
    setPage("home");
  };

  const openTrip = (trip) => { setCurTrip(trip); setExpPaidBy(trip.members[0]); setExpFor(trip.members); setPage("trip"); };

  const addExpense = () => {
    if (!expDesc.trim() || !expAmt || parseFloat(expAmt) <= 0) return;
    const exp = { id: Date.now(), desc: expDesc.trim(), amount: parseFloat(expAmt), paidBy: expPaidBy, forMembers: expFor, date: new Date().toLocaleDateString("en-IN") };
    const updated = trips.map(t => t.id === curTrip.id ? { ...t, expenses: [exp, ...t.expenses] } : t);
    saveTrips(updated);
    setCurTrip(updated.find(t => t.id === curTrip.id));
    setExpDesc(""); setExpAmt(""); setPage("trip");
  };

  const deleteExp = (expId) => {
    const updated = trips.map(t => t.id === curTrip.id ? { ...t, expenses: t.expenses.filter(e => e.id !== expId) } : t);
    saveTrips(updated); setCurTrip(updated.find(t => t.id === curTrip.id));
  };

  const deleteTrip = (tripId) => { saveTrips(trips.filter(t => t.id !== tripId)); setPage("home"); };

  const totalExp = curTrip ? curTrip.expenses.reduce((s,e) => s + e.amount, 0) : 0;
  const balances = curTrip ? calculateBalances(curTrip.members, curTrip.expenses) : {};
  const settlements = curTrip ? calculateSettlements(curTrip.members, curTrip.expenses) : [];

  const inp = { width:"100%", padding:"13px 16px", borderRadius:14, border:"2px solid #e5e7eb", background:"#f9fafb", fontSize:14, outline:"none", marginBottom:10, display:"block", color:"#222" };
  const btn = (bg, color="#fff") => ({ width:"100%", padding:14, borderRadius:14, border:"none", background:bg, color, fontSize:14, fontWeight:800, cursor:"pointer", marginBottom:8 });

  // HOME
  if (page === "home") return (
    <div style={{ minHeight:"100vh", background:"linear-gradient(160deg,#f0f9ff,#e0f2fe,#f0fdf4)", fontFamily:"'Segoe UI',sans-serif", paddingBottom:40 }}>
      <div style={{ background:"linear-gradient(90deg,#06b6d4,#0284c7)", padding:"20px 16px", textAlign:"center", boxShadow:"0 4px 20px rgba(6,182,212,0.3)" }}>
        <div style={{ fontSize:32 }}>✈️</div>
        <h1 style={{ color:"#fff", fontSize:22, fontWeight:900, margin:"6px 0 2px" }}>Trip Expense Tracker</h1>
        <p style={{ color:"rgba(255,255,255,0.8)", fontSize:12 }}>No calculations needed — app does it all!</p>
      </div>
      <div style={{ padding:"20px 16px", maxWidth:480, margin:"0 auto" }}>
        <button style={btn("linear-gradient(90deg,#06b6d4,#0284c7)")} onClick={() => setPage("newTrip")}>✈️ New Trip</button>
        {trips.length === 0 ? (
          <div style={{ textAlign:"center", padding:"50px 20px", color:"#aaa" }}>
            <div style={{ fontSize:56, marginBottom:12 }}>🏕️</div>
            <p style={{ fontWeight:600 }}>No trips yet! Create your first trip.</p>
          </div>
        ) : trips.map(t => (
          <div key={t.id} onClick={() => openTrip(t)} style={{ background:"#fff", borderRadius:20, padding:18, marginBottom:12, boxShadow:"0 4px 16px rgba(0,0,0,0.08)", cursor:"pointer", borderLeft:"4px solid #06b6d4" }}>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center" }}>
              <div>
                <h3 style={{ fontSize:16, fontWeight:800, marginBottom:4 }}>{t.name}</h3>
                <p style={{ fontSize:12, color:"#888" }}>{t.members.length} members • {t.expenses.length} expenses • {t.createdAt}</p>
                <p style={{ fontSize:13, color:"#0284c7", fontWeight:700, marginTop:4 }}>
                  Total: ₹{t.expenses.reduce((s,e)=>s+e.amount,0).toLocaleString()}
                </p>
              </div>
              <div style={{ display:"flex", gap:4 }}>
                {t.members.slice(0,3).map((m,i) => (
                  <div key={i} style={{ width:34, height:34, borderRadius:"50%", background:COLORS[i%COLORS.length], display:"flex", alignItems:"center", justifyContent:"center", color:"#fff", fontSize:12, fontWeight:800 }}>
                    {getInitials(m)}
                  </div>
                ))}
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  // NEW TRIP
  if (page === "newTrip") return (
    <div style={{ minHeight:"100vh", background:"linear-gradient(160deg,#f0f9ff,#e0f2fe)", fontFamily:"'Segoe UI',sans-serif", paddingBottom:40 }}>
      <div style={{ background:"linear-gradient(90deg,#06b6d4,#0284c7)", padding:"16px 16px", display:"flex", alignItems:"center", gap:12 }}>
        <button onClick={() => setPage("home")} style={{ background:"rgba(255,255,255,0.2)", border:"none", color:"#fff", borderRadius:10, padding:"8px 14px", cursor:"pointer", fontWeight:700 }}>← Back</button>
        <h2 style={{ color:"#fff", fontSize:18, fontWeight:900 }}>New Trip</h2>
      </div>
      <div style={{ padding:"20px 16px", maxWidth:480, margin:"0 auto" }}>
        <div style={{ background:"#fff", borderRadius:20, padding:20, marginBottom:16, boxShadow:"0 4px 16px rgba(0,0,0,0.08)" }}>
          <h3 style={{ fontSize:15, color:"#0284c7", marginBottom:14 }}>✈️ Trip Details</h3>
          <input style={inp} placeholder="Trip name (e.g. Jaipur Trip 🏰)" value={tripName} onChange={e=>setTripName(e.target.value)} />
          <p style={{ fontSize:11, color:"#999", textTransform:"uppercase", fontWeight:700, marginBottom:8 }}>Members</p>
          {members.map((m,i) => (
            <div key={i} style={{ display:"flex", gap:8, marginBottom:8 }}>
              <input style={{ ...inp, flex:1, marginBottom:0 }} placeholder={i===0?"Your name":"Friend's name"} value={m} onChange={e=>{const nm=[...members];nm[i]=e.target.value;setMembers(nm);}} />
              {members.length > 1 && <button onClick={()=>setMembers(members.filter((_,j)=>j!==i))} style={{ background:"#fee2e2", border:"none", borderRadius:10, padding:"8px 12px", cursor:"pointer", fontSize:16 }}>✕</button>}
            </div>
          ))}
          <button onClick={()=>setMembers([...members,""])} style={{ width:"100%", padding:11, borderRadius:12, border:"2px dashed #d1d5db", background:"#f9fafb", cursor:"pointer", fontSize:13, fontWeight:700, color:"#888", marginBottom:16 }}>+ Add Member</button>
          <button style={btn("linear-gradient(90deg,#06b6d4,#0284c7)")} onClick={createTrip}>✅ Create Trip</button>
        </div>
      </div>
    </div>
  );

  // ADD EXPENSE
  if (page === "newExp" && curTrip) return (
    <div style={{ minHeight:"100vh", background:"linear-gradient(160deg,#f0f9ff,#e0f2fe)", fontFamily:"'Segoe UI',sans-serif", paddingBottom:40 }}>
      <div style={{ background:"linear-gradient(90deg,#06b6d4,#0284c7)", padding:"16px", display:"flex", alignItems:"center", gap:12 }}>
        <button onClick={() => setPage("trip")} style={{ background:"rgba(255,255,255,0.2)", border:"none", color:"#fff", borderRadius:10, padding:"8px 14px", cursor:"pointer", fontWeight:700 }}>← Back</button>
        <h2 style={{ color:"#fff", fontSize:18, fontWeight:900 }}>Add Expense</h2>
      </div>
      <div style={{ padding:"20px 16px", maxWidth:480, margin:"0 auto" }}>
        <div style={{ background:"#fff", borderRadius:20, padding:20, boxShadow:"0 4px 16px rgba(0,0,0,0.08)" }}>
          <input style={inp} placeholder="What was it for? (e.g. Petrol, Hotel...)" value={expDesc} onChange={e=>setExpDesc(e.target.value)} />
          <input style={inp} type="tel" inputMode="numeric" placeholder="₹ Amount" value={expAmt} onChange={e=>setExpAmt(e.target.value)} />
          <p style={{ fontSize:11, color:"#999", textTransform:"uppercase", fontWeight:700, marginBottom:8 }}>Who Paid?</p>
          <div style={{ display:"flex", gap:8, flexWrap:"wrap", marginBottom:16 }}>
            {curTrip.members.map((m,i) => (
              <button key={i} onClick={()=>setExpPaidBy(m)} style={{ padding:"8px 16px", borderRadius:20, border:`2px solid ${expPaidBy===m?COLORS[i%COLORS.length]:"#e5e7eb"}`, background:expPaidBy===m?COLORS[i%COLORS.length]+"22":"#f9fafb", color:expPaidBy===m?COLORS[i%COLORS.length]:"#888", fontWeight:700, fontSize:13, cursor:"pointer" }}>
                {m}
              </button>
            ))}
          </div>
          <p style={{ fontSize:11, color:"#999", textTransform:"uppercase", fontWeight:700, marginBottom:8 }}>For Whom? (All selected = everyone)</p>
          <div style={{ display:"flex", gap:8, flexWrap:"wrap", marginBottom:16 }}>
            {curTrip.members.map((m,i) => {
              const sel = expFor.includes(m);
              return <button key={i} onClick={()=>setExpFor(sel?expFor.filter(x=>x!==m):[...expFor,m])} style={{ padding:"8px 16px", borderRadius:20, border:`2px solid ${sel?COLORS[i%COLORS.length]:"#e5e7eb"}`, background:sel?COLORS[i%COLORS.length]+"22":"#f9fafb", color:sel?COLORS[i%COLORS.length]:"#888", fontWeight:700, fontSize:13, cursor:"pointer" }}>
                {sel?"✓ ":""}{m}
              </button>;
            })}
          </div>
          <button style={btn("linear-gradient(90deg,#06b6d4,#0284c7)")} onClick={addExpense}>💾 Save Expense</button>
        </div>
      </div>
    </div>
  );

  // TRIP DETAIL
  if (page === "trip" && curTrip) return (
    <div style={{ minHeight:"100vh", background:"linear-gradient(160deg,#f0f9ff,#e0f2fe)", fontFamily:"'Segoe UI',sans-serif", paddingBottom:40 }}>
      <div style={{ background:"linear-gradient(90deg,#06b6d4,#0284c7)", padding:"16px" }}>
        <div style={{ display:"flex", alignItems:"center", gap:12, marginBottom:12 }}>
          <button onClick={() => setPage("home")} style={{ background:"rgba(255,255,255,0.2)", border:"none", color:"#fff", borderRadius:10, padding:"8px 14px", cursor:"pointer", fontWeight:700 }}>← Back</button>
          <h2 style={{ color:"#fff", fontSize:18, fontWeight:900, flex:1 }}>{curTrip.name}</h2>
          <button onClick={()=>deleteTrip(curTrip.id)} style={{ background:"rgba(239,68,68,0.3)", border:"none", color:"#fff", borderRadius:10, padding:"8px 12px", cursor:"pointer", fontSize:16 }}>🗑️</button>
        </div>
        <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr 1fr", gap:8, maxWidth:480, margin:"0 auto" }}>
          <div style={{ background:"rgba(255,255,255,0.15)", borderRadius:14, padding:12, textAlign:"center" }}>
            <p style={{ fontSize:11, color:"rgba(255,255,255,0.8)", marginBottom:4 }}>Total</p>
            <p style={{ fontSize:18, fontWeight:900, color:"#fff" }}>₹{totalExp.toLocaleString()}</p>
          </div>
          <div style={{ background:"rgba(255,255,255,0.15)", borderRadius:14, padding:12, textAlign:"center" }}>
            <p style={{ fontSize:11, color:"rgba(255,255,255,0.8)", marginBottom:4 }}>Members</p>
            <p style={{ fontSize:18, fontWeight:900, color:"#fff" }}>{curTrip.members.length}</p>
          </div>
          <div style={{ background:"rgba(255,255,255,0.15)", borderRadius:14, padding:12, textAlign:"center" }}>
            <p style={{ fontSize:11, color:"rgba(255,255,255,0.8)", marginBottom:4 }}>Expenses</p>
            <p style={{ fontSize:18, fontWeight:900, color:"#fff" }}>{curTrip.expenses.length}</p>
          </div>
        </div>
      </div>

      <div style={{ padding:"16px", maxWidth:480, margin:"0 auto" }}>
        <button style={btn("linear-gradient(90deg,#06b6d4,#0284c7)")} onClick={()=>{setExpPaidBy(curTrip.members[0]);setExpFor(curTrip.members);setPage("newExp");}}>+ Add Expense</button>

        {/* Tabs */}
        <div style={{ display:"flex", background:"#fff", borderRadius:14, padding:4, marginBottom:16, boxShadow:"0 2px 10px rgba(0,0,0,0.08)" }}>
          {["expenses","balances","settle"].map(t => (
            <button key={t} onClick={()=>setActiveTab(t)} style={{ flex:1, padding:10, borderRadius:10, border:"none", background:activeTab===t?"linear-gradient(90deg,#06b6d4,#0284c7)":"transparent", color:activeTab===t?"#fff":"#999", fontWeight:800, fontSize:12, cursor:"pointer" }}>
              {t==="expenses"?"💸 Expenses":t==="balances"?"⚖️ Balances":"✅ Settle Up"}
            </button>
          ))}
        </div>

        {/* Expenses Tab */}
        {activeTab === "expenses" && (
          curTrip.expenses.length === 0 ? (
            <div style={{ textAlign:"center", padding:"40px 20px", color:"#aaa" }}>
              <div style={{ fontSize:48, marginBottom:12 }}>💸</div>
              <p style={{ fontWeight:600 }}>No expenses yet! Add your first expense.</p>
            </div>
          ) : curTrip.expenses.map(exp => {
            const idx = curTrip.members.indexOf(exp.paidBy);
            return (
              <div key={exp.id} style={{ background:"#fff", borderRadius:16, padding:14, marginBottom:10, boxShadow:"0 2px 12px rgba(0,0,0,0.07)", display:"flex", gap:12, alignItems:"center", borderLeft:`4px solid ${COLORS[idx%COLORS.length]}` }}>
                <div style={{ width:44, height:44, borderRadius:12, background:COLORS[idx%COLORS.length]+"22", display:"flex", alignItems:"center", justifyContent:"center", fontSize:11, fontWeight:800, color:COLORS[idx%COLORS.length], flexShrink:0 }}>
                  {getInitials(exp.paidBy)}
                </div>
                <div style={{ flex:1 }}>
                  <p style={{ fontWeight:700, fontSize:14, marginBottom:3 }}>{exp.desc}</p>
                  <p style={{ fontSize:11, color:"#888" }}>Paid by <b style={{ color:COLORS[idx%COLORS.length] }}>{exp.paidBy}</b> • {exp.date}</p>
                  <p style={{ fontSize:11, color:"#aaa" }}>For: {exp.forMembers.length===curTrip.members.length?"Everyone":exp.forMembers.join(", ")}</p>
                </div>
                <div style={{ textAlign:"right", flexShrink:0 }}>
                  <p style={{ fontWeight:900, fontSize:16, color:COLORS[idx%COLORS.length], marginBottom:4 }}>₹{exp.amount.toLocaleString()}</p>
                  <button onClick={()=>deleteExp(exp.id)} style={{ background:"#fee2e2", border:"none", cursor:"pointer", fontSize:14, padding:"4px 8px", borderRadius:8 }}>🗑️</button>
                </div>
              </div>
            );
          })
        )}

        {/* Balances Tab */}
        {activeTab === "balances" && (
          <div style={{ background:"#fff", borderRadius:20, padding:16, boxShadow:"0 4px 16px rgba(0,0,0,0.08)" }}>
            <h4 style={{ fontSize:14, marginBottom:14, color:"#0284c7" }}>Who spent how much?</h4>
            {curTrip.members.map((m,i) => {
              const bal = balances[m] || 0;
              const positive = bal >= 0;
              return (
                <div key={i} style={{ display:"flex", alignItems:"center", gap:12, padding:"12px 0", borderBottom:"1px solid #f3f4f6" }}>
                  <div style={{ width:40, height:40, borderRadius:"50%", background:COLORS[i%COLORS.length], display:"flex", alignItems:"center", justifyContent:"center", color:"#fff", fontSize:13, fontWeight:800, flexShrink:0 }}>
                    {getInitials(m)}
                  </div>
                  <div style={{ flex:1 }}>
                    <p style={{ fontWeight:700, fontSize:14 }}>{m}</p>
                    <p style={{ fontSize:11, color:"#888" }}>
                      Paid: ₹{(curTrip.expenses.filter(e=>e.paidBy===m).reduce((s,e)=>s+e.amount,0)).toLocaleString()}
                    </p>
                  </div>
                  <div style={{ textAlign:"right" }}>
                    <p style={{ fontWeight:900, fontSize:15, color:positive?"#16a34a":"#ef4444" }}>
                      {positive?"+ ":"- "}₹{Math.abs(bal).toLocaleString()}
                    </p>
                    <p style={{ fontSize:10, color:positive?"#16a34a":"#ef4444", fontWeight:700 }}>
                      {positive?"Gets back":"Owes"}
                    </p>
                  </div>
                </div>
              );
            })}
          </div>
        )}

        {/* Settle Up Tab */}
        {activeTab === "settle" && (
          <div>
            {settlements.length === 0 ? (
              <div style={{ background:"#fff", borderRadius:20, padding:24, textAlign:"center", boxShadow:"0 4px 16px rgba(0,0,0,0.08)" }}>
                <div style={{ fontSize:48, marginBottom:12 }}>🎉</div>
                <p style={{ fontWeight:800, fontSize:16, color:"#16a34a" }}>All settled!</p>
                <p style={{ fontSize:13, color:"#888", marginTop:4 }}>No one owes anything</p>
              </div>
            ) : (
              <>
                <div style={{ background:"#f0fdf4", borderRadius:14, padding:12, marginBottom:14, border:"1px solid #bbf7d0" }}>
                  <p style={{ fontSize:12, color:"#16a34a", fontWeight:700 }}>✅ Follow these steps to settle all expenses:</p>
                </div>
                {settlements.map((s,i) => {
                  const fromIdx = curTrip.members.indexOf(s.from);
                  const toIdx = curTrip.members.indexOf(s.to);
                  return (
                    <div key={i} style={{ background:"#fff", borderRadius:16, padding:16, marginBottom:10, boxShadow:"0 2px 12px rgba(0,0,0,0.07)", display:"flex", alignItems:"center", gap:12 }}>
                      <div style={{ width:40, height:40, borderRadius:"50%", background:COLORS[fromIdx%COLORS.length], display:"flex", alignItems:"center", justifyContent:"center", color:"#fff", fontSize:12, fontWeight:800, f
