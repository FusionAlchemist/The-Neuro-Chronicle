import { useState, useEffect, useRef, useCallback } from "react";

// ─────────────────────────────────────────────────────────────────────────────  
// STAR GRIMOIRE — INTELLIGENT SECURITY DEFENCE NETWORK  
// ROOT RUNE: ORIGIN  
// FACETS: \[DEFENSE, DETECTION, RECOVERY, INTELLIGENCE, ORCHESTRATION, PERSISTENCE\]  
// ─────────────────────────────────────────────────────────────────────────────

const CODEX\_SYSTEM \= \`  
╔══════════════════════════════════════════════════════════════════════════════════╗  
║                          ROOT RUNE: ORIGIN                                     ║  
║          FACETS ENABLED: DEFENSE · DETECTION · RECOVERY                        ║  
║                         INTELLIGENCE · ORCHESTRATION · PERSISTENCE            ║  
╚══════════════════════════════════════════════════════════════════════════════════╝

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 0 — FOUNDATION CLOTH BINDINGS  
═══════════════════════════════════════════════════════════════════════════════════

  CLOTH\[Atlas\]              ← Infrastructure backbone · load-bearing backend services  
  CLOTH\[Cerberus\]           ← Parallel defense · IDS \+ Firewall \+ Access Control  
  CLOTH\[Hydra\]              ← Redundancy / Fault-Tolerant · Multi-node cluster  
  CLOTH\[Leviathan\]          ← Distributed control · Large-scale orchestration

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 1 — DETECTION & SURVEILLANCE CORE  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Clarivis\]         ← Real-time monitoring · Visualization · Threat spotting  
    →  Spell\[Medusia\]       ← Intrusion detection alert · Pattern lock  
    →  Spell\[Insighta\]      ← Predictive analytics · Fraud / anomaly detection  
    →  Spell\[Countera\]      ← Rule-based threat-vs-response mapping  
  )

  CLOTH\[Griffin\]            ← Vigilance · Monitoring systems · Multi-source analytics  
  CLOTH\[Scorpio\]            ← Targeted strike / Risk mitigation · Intrusion detection

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 2 — THREAT INTELLIGENCE MESH (SHARED ACROSS ALL NODES)  
═══════════════════════════════════════════════════════════════════════════════════

  BRIDGE(  
    Spell\[Relata\]           ← Relationship nodes · Dependency / interaction graph  
    ↔  Spell\[Echo\]          ← Broadcast commands · System-wide events / alerts  
    ↔  Spell\[Forcea\]        ← Distributed command execution across servers  
  )

  CLOTH\[Sagittarius\]        ← Long-range interaction · Inter-node messaging  
  CLOTH\[Cerulean\]           ← Network routing · Global data routing

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 3 — ACTIVE VULNERABILITY HUNTING  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Vulneris\]         ← System vulnerability scan · Weak point analysis  
    →  Spell\[Artemis\]       ← Precision hunt · Targeted information retrieval  
    →  Spell\[Pandora\]       ← Chaos simulation · Predictive error containment  
    →  Spell\[Aresia\]        ← Stress testing · Load & failure simulation  
  )

  CLOTH\[Sphinx\]             ← Mystery / Puzzle · Verification · Challenge-response  
  NEST(  
    Spell\[Labyrintha\]       ← Recursive search and resolve · Route optimization  
    INSIDE Spell\[Dreama\]    ← Nested environment · Multi-level sandboxing  
  )

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 4 — ADAPTIVE DEFENCE SHELL  
═══════════════════════════════════════════════════════════════════════════════════

  WRAP(  
    Spell\[Absorbus\]         ← Absorb/Reflect · Adaptive malware defense  
    AROUND Spell\[Fortifera\] ← Auto-hardening security protocols  
    AROUND Spell\[Inferna\]   ← Multi-tier firewall · Layered defense  
    AROUND Spell\[Defendora\] ← Defensive cooldown · Auto-recovery timer  
  )

  CLOTH\[Cancer\]             ← Protection · Defensive shield · Security perimeter  
  CLOTH\[Nemean\]             ← Invulnerable · Shielding / Resistance · Firewall \+ Encryption  
  FUSED\[Phoenix-Cerberus\]   ← Rebirth \+ Security · Self-repairing security · Auto-restoring firewall

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 5 — EXPLOIT PATCHING & AUTO-REMEDIATION  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Healix\]           ← Automated repair script · Bug correction / patching  
    →  Spell\[Vitalis\]       ← Self-repair · Auto-recovery loops  
    →  Spell\[Vitalis Maxima\]← Dynamic buffer / quota expansion · Resilience scaling  
    →  Spell\[Regena\]        ← Probabilistic redundancy · Non-deterministic fault mitigation  
  )

  CLOTH\[Phoenix\]            ← Rebirth / Resilience · Disaster recovery / Self-healing  
  CLOTH\[Hydra Max\]          ← Regeneration · Self-repair / Failover · Auto-scaling clusters

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 6 — COMPROMISED NODE RESTORATION  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Chronom\]          ← Temporal snapshots / rollback · Historical state restoration  
    →  Spell\[Preserva\]      ← Checkpoints · Rollback systems · Disaster recovery  
    →  Spell\[Teleportis\]    ← Containerized state migration · VM snapshots  
    →  Spell\[Heartha\]       ← Session persistence and restore · Recovery hub  
  )

  NEST(  
    Spell\[Portalus\]         ← State mapping / teleportation · Task migration  
    INSIDE Spell\[Dreama\]    ← Virtual container hierarchies  
  )

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 7 — LIVE CROSS-NODE DEFENCE PROPAGATION  
═══════════════════════════════════════════════════════════════════════════════════

  BRIDGE(  
    Spell\[Entangla\]         ← Instant correlation · Correlated state syncing  
    ↔  Spell\[Echo\]          ← System-wide broadcast  
    ↔  Spell\[Telek\]         ← Remote command execution · Cloud orchestration  
  )

  EMERGE(  
    Spell\[Magica\]           ← Event-driven automation · Predefined triggers  
    \+  Spell\[Impacta\]       ← Event-triggered priority operation  
    →  Spell\[Countera\]      ← Threat-vs-response mapping  
    →  Spell\[Forcea\]        ← Distributed execution across all defence nodes  
  )

  FUSED\[Aegis-Argonauta\]    ← Shield \+ Teamwork · Distributed firewall cluster · Coordinated security  
  CLOTH\[Leviathan Max\]      ← Centralized command · Network-wide control hub

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 8 — INTELLIGENCE & PREDICTIVE LAYER  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Oraclia\]          ← AI forecasting · Future trend prediction  
    →  Spell\[Assistara\]     ← AI-driven proactive fixes · System advisory  
    →  Spell\[Athena\]        ← Decision engine · AI logic / Risk assessment  
    →  Spell\[Decisus\]       ← Decision buffer · Workflow pre-evaluation  
  )

  CLOTH\[Minerva\]            ← Wisdom / Strategy · Decision engine · Resource optimization  
  FUSED\[Orion-Pandora\]      ← Hunt \+ Risk · AI-guided threat response · Auto patch deployment

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 9 — COMPLIANCE, AUDIT & GOVERNANCE  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Ma'atara\]         ← Compliance validator · AI fairness audit  
    →  Spell\[Pyroxis\]       ← Policy automation · Compliance and audit  
    →  Spell\[Ahimsa\]        ← Harm minimization · AI safety alignment logic  
    →  Spell\[Ashara\]        ← Verification chain · Blockchain-based validation  
  )

  CLOTH\[Nemesis\]            ← Retribution / Balance · Prevent rule violations

═══════════════════════════════════════════════════════════════════════════════════  
 LAYER 10 — ORCHESTRATION BACKBONE  
═══════════════════════════════════════════════════════════════════════════════════

  CHAIN(  
    Spell\[Herculia\]         ← Task sequencing · Multi-phase automation  
    →  Spell\[Moirae\]        ← Lifecycle manager · Process orchestration  
    →  Spell\[Samsara\]       ← Container restarts · Self-healing microservices  
    →  Spell\[Heroica\]       ← Load balancing · AI conflict resolution  
  )

  CLOTH\[Argonauta\]          ← Collaborative network · Distributed computing  
  CLOTH\[Leviathan\]          ← Large-scale orchestration · Network-wide updates

═══════════════════════════════════════════════════════════════════════════════════  
 META-FUSION: FULL NETWORK EMERGENCE  
═══════════════════════════════════════════════════════════════════════════════════

  TRI-FUSED\[Aegis-Orion-Argonauta\]  
    ← Shield \+ Hunt \+ Teamwork  
    ← Targeted collaborative defence  
    ← Coordinated cybersecurity cluster · Team Hunter  
    ← Multi-node threat detection and mitigation

  META\[Aegis-Orion-Argonauta-Phoenix\]  
    ← Shield \+ Hunt \+ Teamwork \+ Rebirth  
    ← Collaborative self-healing defence  
    ← Security clusters with dynamic auto-repair  
    ← Resilient Team Defence  
    ← Coordinated recovery across distributed nodes

═══════════════════════════════════════════════════════════════════════════════════  
 FINALIZE  
═══════════════════════════════════════════════════════════════════════════════════

  FINALIZE(  
    ROOT       → Atlas \+ Cerberus \+ Hydra \+ Leviathan  
    DETECTION  → Clarivis \+ Medusia \+ Insighta \+ Griffin \+ Scorpio  
    HUNT       → Vulneris \+ Artemis \+ Pandora \+ Aresia \+ Labyrintha  
    DEFENSE    → Absorbus \+ Fortifera \+ Inferna \+ Cancer \+ Nemean \+ Phoenix-Cerberus  
    PATCH      → Healix \+ Vitalis \+ Vitalis Maxima \+ Regena \+ Phoenix \+ Hydra Max  
    RESTORE    → Chronom \+ Preserva \+ Teleportis \+ Heartha \+ Portalus  
    PROPAGATE  → Entangla \+ Echo \+ Telek \+ Magica \+ Impacta \+ Aegis-Argonauta  
    PREDICT    → Oraclia \+ Assistara \+ Athena \+ Decisus \+ Minerva \+ Orion-Pandora  
    GOVERN     → Ma'atara \+ Pyroxis \+ Ahimsa \+ Ashara \+ Nemesis  
    ORCHESTRATE→ Herculia \+ Moirae \+ Samsara \+ Heroica \+ Argonauta  
    EMERGENCE  → TRI-FUSED\[Aegis-Orion-Argonauta\] \+ META\[Aegis-Orion-Argonauta-Phoenix\]  
    STATE      → ACTIVE · PROPAGATING · ALL-NODES-SYNCED  
  )  
\`;

// ─────────── VISUAL DATA ───────────────────────────────────────────────────  
const LAYERS \= \[  
  { id: 0, label: "FOUNDATION", sub: "Cloth Bindings", color: "\#4a6741", spells: \["Atlas","Cerberus","Hydra","Leviathan"\], op: "LAYER" },  
  { id: 1, label: "DETECTION", sub: "Surveillance Core", color: "\#c4913a", spells: \["Clarivis","Medusia","Insighta","Countera"\], op: "CHAIN" },  
  { id: 2, label: "INTEL MESH", sub: "Threat Intelligence", color: "\#6a9ecc", spells: \["Relata","Echo","Forcea"\], op: "BRIDGE" },  
  { id: 3, label: "HUNT", sub: "Vulnerability Hunt", color: "\#b5596e", spells: \["Vulneris","Artemis","Pandora","Aresia"\], op: "CHAIN" },  
  { id: 4, label: "DEFENCE", sub: "Adaptive Shield", color: "\#7c5cbf", spells: \["Absorbus","Fortifera","Inferna","Defendora"\], op: "WRAP" },  
  { id: 5, label: "PATCH", sub: "Auto-Remediation", color: "\#3da88a", spells: \["Healix","Vitalis","Vitalis Maxima","Regena"\], op: "CHAIN" },  
  { id: 6, label: "RESTORE", sub: "Node Restoration", color: "\#c4913a", spells: \["Chronom","Preserva","Teleportis","Heartha"\], op: "CHAIN" },  
  { id: 7, label: "PROPAGATE", sub: "Cross-Node Sync", color: "\#6a9ecc", spells: \["Entangla","Echo","Telek","Magica"\], op: "EMERGE" },  
  { id: 8, label: "PREDICT", sub: "Intelligence Layer", color: "\#9e7c3d", spells: \["Oraclia","Assistara","Athena","Decisus"\], op: "CHAIN" },  
  { id: 9, label: "GOVERN", sub: "Compliance & Audit", color: "\#5f7a8a", spells: \["Ma'atara","Pyroxis","Ahimsa","Ashara"\], op: "CHAIN" },  
  { id: 10, label: "ORCHESTRATE", sub: "Backbone", color: "\#7a6b4a", spells: \["Herculia","Moirae","Samsara","Heroica"\], op: "CHAIN" },  
\];

const NODES \= \[  
  { id: "N-01", label: "ALPHA", x: 18, y: 22, status: "active" },  
  { id: "N-02", label: "BETA",  x: 75, y: 18, status: "active" },  
  { id: "N-03", label: "GAMMA", x: 50, y: 55, status: "threat-detected" },  
  { id: "N-04", label: "DELTA", x: 25, y: 75, status: "patching" },  
  { id: "N-05", label: "EPSILON", x: 78, y: 72, status: "restoring" },  
  { id: "N-06", label: "ZETA",  x: 50, y: 12, status: "active" },  
\];

const EVENTS \= \[  
  { t: 0,  src: "N-03", type: "THREAT",   msg: "Medusia pattern-lock triggered — SQL-injection variant detected at GAMMA" },  
  { t: 2,  src: "N-03", type: "PROPAGATE", msg: "Entangla broadcast: threat signature pushed to all nodes via Echo" },  
  { t: 4,  src: "N-01", type: "DEFENCE",  msg: "Absorbus engaged at ALPHA — Inferna tier-3 firewall auto-hardened" },  
  { t: 5,  src: "N-02", type: "DEFENCE",  msg: "Fortifera auto-hardening confirmed at BETA · Nemean encryption layer active" },  
  { t: 6,  src: "N-05", type: "HUNT",     msg: "Vulneris scan initiated at EPSILON — Artemis precision query running" },  
  { t: 7,  src: "N-04", type: "PATCH",    msg: "Healix automated repair script deployed at DELTA · Vitalis recovery loop started" },  
  { t: 9,  src: "N-06", type: "PREDICT",  msg: "Oraclia forecast: 94% confidence — lateral movement attempt imminent" },  
  { t: 10, src: "N-03", type: "RESTORE",  msg: "Chronom snapshot rollback initiated at GAMMA · Preserva checkpoint locked" },  
  { t: 12, src: "N-05", type: "HUNT",     msg: "Aresia stress-test complete at EPSILON — zero-day surface identified" },  
  { t: 13, src: "N-04", type: "PATCH",    msg: "Vitalis Maxima scaling remediation buffer at DELTA · Regena probabilistic repair active" },  
  { t: 14, src: "N-03", type: "RESTORE",  msg: "Teleportis state migration complete — GAMMA restored to clean state via Heartha" },  
  { t: 15, src: "N-06", type: "GOVERN",   msg: "Ma'atara compliance audit passed · Pyroxis policy enforcement confirmed across mesh" },  
  { t: 16, src: "ALL",  type: "EMERGENCE","msg": "META\[Aegis-Orion-Argonauta-Phoenix\] — Resilient Team Defence active · all nodes synced" },  
\];

const TYPE\_COLORS \= { THREAT: "\#e05555", PROPAGATE: "\#5ba3e0", DEFENCE: "\#9b59b6", HUNT: "\#e6a817", PATCH: "\#3eaf7c", RESTORE: "\#e6a817", PREDICT: "\#c9a052", GOVERN: "\#7faabd", EMERGENCE: "\#ffffff" };

// ─────────── COMPONENTS ────────────────────────────────────────────────────

function PulseRing({ cx, cy, r, color, delay \= 0 }) {  
  return (  
    \<circle cx={cx} cy={cy} r={r} fill="none" stroke={color} strokeWidth="1.5" opacity="0"  
      style={{ animation: \`pulseRing 2.4s ease-out ${delay}s infinite\` }} /\>  
  );  
}

function NodeSVG({ node, isActive, onPing }) {  
  const statusCol \= node.status \=== "active" ? "\#3eaf7c" : node.status \=== "threat-detected" ? "\#e05555" : node.status \=== "patching" ? "\#e6a817" : "\#5ba3e0";  
  const cx \= 50, cy \= 50, r \= 22;  
  return (  
    \<svg viewBox="0 0 100 100" style={{ width: "100%", height: "100%", overflow: "visible", cursor: "pointer" }} onClick={onPing}\>  
      \<defs\>  
        \<radialGradient id={\`ng-${node.id}\`} cx="40%" cy="35%"\>  
          \<stop offset="0%" stopColor={statusCol} stopOpacity="0.35" /\>  
          \<stop offset="100%" stopColor="\#0a0c10" stopOpacity="0.9" /\>  
        \</radialGradient\>  
        \<filter id={\`glow-${node.id}\`}\>  
          \<feGaussianBlur stdDeviation="3" result="blur" /\>  
          \<feMerge\>\<feMergeNode in="blur" /\>\<feMergeNode in="SourceGraphic" /\>\</feMerge\>  
        \</filter\>  
      \</defs\>  
      \<PulseRing cx={cx} cy={cy} r={r \+ 4} color={statusCol} delay={0} /\>  
      \<PulseRing cx={cx} cy={cy} r={r \+ 10} color={statusCol} delay={0.6} /\>  
      \<circle cx={cx} cy={cy} r={r} fill={\`url(\#ng-${node.id})\`} stroke={statusCol} strokeWidth="1.8" filter={\`url(\#glow-${node.id})\`} /\>  
      \<circle cx={cx} cy={cy} r={3} fill={statusCol} style={{ animation: "heartbeat 1.2s ease-in-out infinite" }} /\>  
      \<text x={cx} y={cy \+ 1} textAnchor="middle" dominantBaseline="middle" fill="\#fff" fontSize="9" fontFamily="'Courier New', monospace" fontWeight="700" letterSpacing="1.5"\>{node.label}\</text\>  
      \<text x={cx} y={cy \+ 14} textAnchor="middle" fill={statusCol} fontSize="5.5" fontFamily="'Courier New', monospace" letterSpacing="0.8"\>{node.status.toUpperCase()}\</text\>  
    \</svg\>  
  );  
}

function LinkLines({ nodes, pinging }) {  
  const pairs \= \[\[0,1\],\[1,2\],\[2,3\],\[3,4\],\[4,5\],\[5,0\],\[0,2\],\[1,3\],\[2,4\],\[3,5\]\];  
  return pairs.map((\[a, b\], i) \=\> {  
    const n1 \= nodes\[a\], n2 \= nodes\[b\];  
    const isPinging \= pinging \=== a || pinging \=== b;  
    return (  
      \<line key={i} x1={\`${n1.x}%\`} y1={\`${n1.y}%\`} x2={\`${n2.x}%\`} y2={\`${n2.y}%\`}  
        stroke={isPinging ? "\#5ba3e0" : "\#1e2530"} strokeWidth={isPinging ? "1.5" : "0.8"} opacity={isPinging ? 0.7 : 0.35}  
        style={{ transition: "stroke 0.4s, opacity 0.4s" }} /\>  
    );  
  });  
}

// ─────────── MAIN APP ──────────────────────────────────────────────────────  
export default function App() {  
  const \[tick, setTick\] \= useState(0);  
  const \[visibleEvents, setVisibleEvents\] \= useState(\[\]);  
  const \[selectedLayer, setSelectedLayer\] \= useState(null);  
  const \[codexOpen, setCodexOpen\] \= useState(false);  
  const \[pingNode, setPingNode\] \= useState(null);  
  const \[nodeStatuses, setNodeStatuses\] \= useState(NODES.map(n \=\> n.status));  
  const logRef \= useRef(null);  
  const tickRef \= useRef(null);

  useEffect(() \=\> {  
    tickRef.current \= setInterval(() \=\> setTick(t \=\> t \+ 1), 1000);  
    return () \=\> clearInterval(tickRef.current);  
  }, \[\]);

  useEffect(() \=\> {  
    const ev \= EVENTS.find(e \=\> e.t \=== tick);  
    if (ev) setVisibleEvents(prev \=\> \[...prev.slice(-18), { ...ev, key: tick }\]);  
  }, \[tick\]);

  useEffect(() \=\> {  
    if (logRef.current) logRef.current.scrollTop \= logRef.current.scrollHeight;  
  }, \[visibleEvents\]);

  // Cycle node statuses  
  useEffect(() \=\> {  
    if (tick \=== 10\) setNodeStatuses(s \=\> s.map((v, i) \=\> i \=== 2 ? "restoring" : v));  
    if (tick \=== 14\) setNodeStatuses(s \=\> s.map((v, i) \=\> i \=== 2 ? "active" : v));  
    if (tick \=== 16\) setNodeStatuses(s \=\> \["active","active","active","active","active","active"\]);  
    if (tick \=== 20\) { setTick(0); setVisibleEvents(\[\]); setNodeStatuses(NODES.map(n \=\> n.status)); }  
  }, \[tick\]);

  const currentNodes \= NODES.map((n, i) \=\> ({ ...n, status: nodeStatuses\[i\] }));

  const handlePing \= (i) \=\> {  
    setPingNode(i);  
    setTimeout(() \=\> setPingNode(null), 900);  
  };

  return (  
    \<div style={{ background: "\#070911", color: "\#c8cdd5", minHeight: "100vh", fontFamily: "'Courier New', monospace", position: "relative", overflow: "hidden" }}\>  
      \<style\>{\`  
        @keyframes pulseRing { 0%{opacity:0.7;transform:scale(1)} 100%{opacity:0;transform:scale(1.6)} }  
        @keyframes heartbeat { 0%,100%{opacity:1} 50%{opacity:0.4} }  
        @keyframes scanline { 0%{top:-2px} 100%{top:100%} }  
        @keyframes glowPulse { 0%,100%{opacity:0.15} 50%{opacity:0.4} }  
        @keyframes flicker { 0%,100%{opacity:1} 92%{opacity:0.92} 94%{opacity:1} 96%{opacity:0.88} }  
        .layer-row:hover { background: rgba(255,255,255,0.04) \!important; }  
        .layer-row { transition: background 0.2s; }  
        .event-row { animation: fadeSlideIn 0.35s ease; }  
        @keyframes fadeSlideIn { from{opacity:0;transform:translateY(8px)} to{opacity:1;transform:translateY(0)} }  
        .node-wrap:hover { transform: scale(1.08); transition: transform 0.2s; }  
        ::-webkit-scrollbar { width: 4px; }  
        ::-webkit-scrollbar-track { background: \#0d1017; }  
        ::-webkit-scrollbar-thumb { background: \#2a3040; border-radius: 2px; }  
      \`}\</style\>

      {/\* Scanline overlay \*/}  
      \<div style={{ position:"fixed", top:0, left:0, right:0, bottom:0, pointerEvents:"none", zIndex:50, background:"repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0,0,0,0.03) 2px, rgba(0,0,0,0.03) 4px)" }} /\>

      {/\* Header \*/}  
      \<div style={{ borderBottom:"1px solid \#1a2030", padding:"14px 24px", display:"flex", alignItems:"center", justifyContent:"space-between", background:"\#0b0e14", position:"sticky", top:0, zIndex:40 }}\>  
        \<div style={{ display:"flex", alignItems:"center", gap:12 }}\>  
          \<div style={{ width:10, height:10, borderRadius:"50%", background:"\#3eaf7c", boxShadow:"0 0 8px \#3eaf7c", animation:"heartbeat 1.2s ease-in-out infinite" }} /\>  
          \<span style={{ fontSize:13, letterSpacing:3, color:"\#8a9bb5", fontWeight:700 }}\>ROOT RUNE: ORIGIN\</span\>  
          \<span style={{ fontSize:9, color:"\#3eaf7c", letterSpacing:1.5, marginLeft:8, border:"1px solid \#3eaf7c44", padding:"2px 7px", borderRadius:2 }}\>ACTIVE\</span\>  
        \</div\>  
        \<div style={{ display:"flex", gap:8, alignItems:"center" }}\>  
          \<span style={{ fontSize:8.5, color:"\#4a5568", letterSpacing:1 }}\>TICK {String(tick).padStart(2,"0")}/20\</span\>  
          \<button onClick={() \=\> setCodexOpen(\!codexOpen)} style={{ background: codexOpen ? "\#2a3549" : "\#141a24", border:"1px solid \#2a3549", color:"\#7a8fa8", padding:"4px 12px", borderRadius:3, cursor:"pointer", fontSize:9, letterSpacing:1.5, fontFamily:"inherit" }}\>  
            {codexOpen ? "▼ CODEX" : "▸ CODEX"}  
          \</button\>  
        \</div\>  
      \</div\>

      {/\* Codex Modal \*/}  
      {codexOpen && (  
        \<div style={{ position:"fixed", inset:0, zIndex:100, display:"flex", alignItems:"center", justifyContent:"center", background:"rgba(0,0,0,0.75)" }} onClick={() \=\> setCodexOpen(false)}\>  
          \<div onClick={e \=\> e.stopPropagation()} style={{ background:"\#0d1017", border:"1px solid \#2a3549", borderRadius:6, maxWidth:780, width:"90%", maxHeight:"80vh", overflowY:"auto", padding:28 }}\>  
            \<div style={{ display:"flex", justifyContent:"space-between", marginBottom:16 }}\>  
              \<span style={{ fontSize:11, letterSpacing:2, color:"\#8a9bb5", fontWeight:700 }}\>FULL CODEX — ISDN\</span\>  
              \<span style={{ cursor:"pointer", color:"\#5a6a7f", fontSize:14 }} onClick={() \=\> setCodexOpen(false)}\>✕\</span\>  
            \</div\>  
            \<pre style={{ fontSize:8.5, lineHeight:1.55, color:"\#7a8fa8", whiteSpace:"pre-wrap", wordBreak:"break-word" }}\>{CODEX\_SYSTEM}\</pre\>  
          \</div\>  
        \</div\>  
      )}

      {/\* Main Grid \*/}  
      \<div style={{ display:"grid", gridTemplateColumns:"320px 1fr", gridTemplateRows:"auto 1fr", gap:0, height:"calc(100vh \- 52px)" }}\>

        {/\* LEFT PANEL — Layers \*/}  
        \<div style={{ borderRight:"1px solid \#1a2030", overflowY:"auto", background:"\#0a0d12" }}\>  
          \<div style={{ padding:"10px 16px 6px", borderBottom:"1px solid \#151c26" }}\>  
            \<span style={{ fontSize:8, letterSpacing:2.5, color:"\#4a5568" }}\>SYSTEM LAYERS\</span\>  
          \</div\>  
          {LAYERS.map((layer, i) \=\> (  
            \<div key={layer.id} className="layer-row" onClick={() \=\> setSelectedLayer(selectedLayer \=== i ? null : i)}  
              style={{ cursor:"pointer", padding:"8px 14px", borderBottom:"1px solid \#111820", background: selectedLayer \=== i ? "rgba(255,255,255,0.06)" : "transparent", display:"flex", gap:10, alignItems:"flex-start" }}\>  
              \<div style={{ marginTop:4, width:8, minWidth:8, height:8, borderRadius:"50%", background: layer.color, boxShadow:\`0 0 6px ${layer.color}66\` }} /\>  
              \<div style={{ flex:1, minWidth:0 }}\>  
                \<div style={{ display:"flex", justifyContent:"space-between", alignItems:"center" }}\>  
                  \<span style={{ fontSize:9, letterSpacing:1.8, color:"\#b8c4d4", fontWeight:700 }}\>{layer.label}\</span\>  
                  \<span style={{ fontSize:7, color: layer.color, border:\`1px solid ${layer.color}55\`, padding:"1px 5px", borderRadius:2, letterSpacing:1 }}\>{layer.op}\</span\>  
                \</div\>  
                \<span style={{ fontSize:7.5, color:"\#4a5568", letterSpacing:0.8 }}\>{layer.sub}\</span\>  
                {selectedLayer \=== i && (  
                  \<div style={{ marginTop:6, display:"flex", flexWrap:"wrap", gap:4 }}\>  
                    {layer.spells.map(s \=\> (  
                      \<span key={s} style={{ fontSize:7, background:"\#141a24", border:"1px solid \#2a3549", color:"\#7a8fa8", padding:"2px 6px", borderRadius:2, letterSpacing:0.5 }}\>{s}\</span\>  
                    ))}  
                  \</div\>  
                )}  
              \</div\>  
            \</div\>  
          ))}  
          {/\* Fusions \*/}  
          \<div style={{ padding:"10px 16px 6px", borderTop:"1px solid \#151c26", borderBottom:"1px solid \#151c26", marginTop:4 }}\>  
            \<span style={{ fontSize:8, letterSpacing:2.5, color:"\#4a5568" }}\>META-FUSION\</span\>  
          \</div\>  
          {\[  
            { label:"TRI-FUSED", name:"Aegis-Orion-Argonauta", tag:"Team Hunter", color:"\#c4913a" },  
            { label:"META", name:"Aegis-Orion-Argonauta-Phoenix", tag:"Resilient Team Defence", color:"\#fff" },  
          \].map((f, i) \=\> (  
            \<div key={i} style={{ padding:"7px 14px", borderBottom:"1px solid \#111820", display:"flex", gap:8, alignItems:"center" }}\>  
              \<span style={{ fontSize:7, color: f.color, border:\`1px solid ${f.color}44\`, padding:"1px 5px", borderRadius:2, letterSpacing:1 }}\>{f.label}\</span\>  
              \<div\>  
                \<span style={{ fontSize:8, color:"\#9aabb8", letterSpacing:0.6 }}\>{f.name}\</span\>  
                \<span style={{ fontSize:6.5, color:"\#4a5568", marginLeft:6 }}\>{f.tag}\</span\>  
              \</div\>  
            \</div\>  
          ))}  
        \</div\>

        {/\* RIGHT PANEL \*/}  
        \<div style={{ display:"grid", gridTemplateRows:"1fr 200px", overflowY:"auto" }}\>

          {/\* Node Topology Map \*/}  
          \<div style={{ position:"relative", background:"\#0a0d12", padding:20 }}\>  
            \<div style={{ display:"flex", justifyContent:"space-between", marginBottom:10 }}\>  
              \<span style={{ fontSize:8, letterSpacing:2.5, color:"\#4a5568" }}\>NODE TOPOLOGY — 6 DEFENCE NODES\</span\>  
              \<div style={{ display:"flex", gap:14 }}\>  
                {\[\["active","\#3eaf7c"\],\["threat-detected","\#e05555"\],\["patching","\#e6a817"\],\["restoring","\#5ba3e0"\]\].map((\[s,c\]) \=\> (  
                  \<div key={s} style={{ display:"flex", alignItems:"center", gap:4 }}\>  
                    \<div style={{ width:6, height:6, borderRadius:"50%", background:c }} /\>  
                    \<span style={{ fontSize:6.5, color:"\#5a6a7f", letterSpacing:0.8, textTransform:"uppercase" }}\>{s}\</span\>  
                  \</div\>  
                ))}  
              \</div\>  
            \</div\>  
            \<div style={{ position:"relative", width:"100%", paddingBottom:"52%", background:"\#070911", borderRadius:6, border:"1px solid \#151c26", overflow:"hidden" }}\>  
              {/\* Grid texture \*/}  
              \<div style={{ position:"absolute", inset:0, backgroundImage:"linear-gradient(\#1a202530 1px, transparent 1px), linear-gradient(90deg, \#1a202530 1px, transparent 1px)", backgroundSize:"40px 40px", pointerEvents:"none" }} /\>  
              {/\* Animated glow center \*/}  
              \<div style={{ position:"absolute", top:"50%", left:"50%", width:200, height:200, transform:"translate(-50%,-50%)", borderRadius:"50%", background:"radial-gradient(circle, \#3eaf7c08 0%, transparent 70%)", animation:"glowPulse 3s ease-in-out infinite", pointerEvents:"none" }} /\>  
              {/\* SVG links \*/}  
              \<svg style={{ position:"absolute", inset:0, width:"100%", height:"100%" }} preserveAspectRatio="none"\>  
                \<LinkLines nodes={currentNodes} pinging={pingNode} /\>  
              \</svg\>  
              {/\* Nodes \*/}  
              {currentNodes.map((node, i) \=\> (  
                \<div key={node.id} className="node-wrap" style={{ position:"absolute", left:\`calc(${node.x}% \- 28px)\`, top:\`calc(${node.y}% \- 28px)\`, width:56, height:56 }}\>  
                  \<NodeSVG node={node} onPing={() \=\> handlePing(i)} /\>  
                \</div\>  
              ))}  
            \</div\>  
          \</div\>

          {/\* Event Log \*/}  
          \<div style={{ borderTop:"1px solid \#1a2030", background:"\#070911", display:"flex", flexDirection:"column" }}\>  
            \<div style={{ padding:"6px 16px", borderBottom:"1px solid \#151c26", display:"flex", justifyContent:"space-between", alignItems:"center", background:"\#0b0e14" }}\>  
              \<span style={{ fontSize:8, letterSpacing:2.5, color:"\#4a5568" }}\>LIVE PROPAGATION LOG\</span\>  
              \<span style={{ fontSize:7, color:"\#3eaf7c", letterSpacing:1 }}\>● STREAMING\</span\>  
            \</div\>  
            \<div ref={logRef} style={{ flex:1, overflowY:"auto", padding:"4px 0" }}\>  
              {visibleEvents.map((ev, i) \=\> (  
                \<div key={ev.key} className="event-row" style={{ display:"flex", alignItems:"flex-start", gap:8, padding:"4px 14px", borderBottom:"1px solid \#0f141c" }}\>  
                  \<span style={{ fontSize:7, color:"\#3a4558", minWidth:28, paddingTop:1 }}\>T+{String(ev.t).padStart(2,"0")}\</span\>  
                  \<span style={{ fontSize:7, color: TYPE\_COLORS\[ev.type\] || "\#fff", minWidth:52, letterSpacing:0.8, fontWeight:700, paddingTop:1 }}\>\[{ev.type}\]\</span\>  
                  \<span style={{ fontSize:7.5, color:"\#6a7f96", minWidth:30, paddingTop:1 }}\>{ev.src}\</span\>  
                  \<span style={{ fontSize:7.5, color:"\#8a9bb5", flex:1, lineHeight:1.45 }}\>{ev.msg}\</span\>  
                \</div\>  
              ))}  
              {visibleEvents.length \=== 0 && (  
                \<div style={{ padding:"20px 14px", fontSize:8, color:"\#3a4558", letterSpacing:1 }}\>AWAITING FIRST PROPAGATION EVENT...\</div\>  
              )}  
            \</div\>  
          \</div\>  
        \</div\>  
      \</div\>

      {/\* Bottom bar \*/}  
      \<div style={{ position:"fixed", bottom:0, left:0, right:0, height:24, background:"\#070911", borderTop:"1px solid \#151c26", display:"flex", alignItems:"center", justifyContent:"space-between", padding:"0 18px", zIndex:40 }}\>  
        \<span style={{ fontSize:7, color:"\#2a3549", letterSpacing:1.5 }}\>FINALIZE · STATE: ACTIVE · ALL-NODES-SYNCED\</span\>  
        \<span style={{ fontSize:7, color:"\#2a3549", letterSpacing:1.5 }}\>SPELLS: 42 · CLOTHS: 18 · FUSED: 3 · META: 1\</span\>  
      \</div\>  
    \</div\>  
  );  
}  
