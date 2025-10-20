cat > tools/apply_ui_overlay_v3.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "$0")/.." && pwd)"
mkdir -p "$ROOT/components" "$ROOT/src/utils" "$ROOT/public/_injections"

# 0) CSV 工具（若不存在则补齐）
if [ ! -f "$ROOT/src/utils/csv.ts" ]; then
  cat > "$ROOT/src/utils/csv.ts" <<'TS'
export function downloadCSV(filename: string, rows: (string|number)[][]) {
  const content = rows.map(r => r.map(x=>{
    const s = String(x ?? "");
    if (/[",\n]/.test(s)) return `"${s.replace(/"/g,'""')}"`;
    return s;
  }).join(",")).join("\n");
  const blob = new Blob(["\ufeff"+content], { type: "text/csv;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = filename; a.click();
  setTimeout(()=>URL.revokeObjectURL(url), 0);
}
TS
fi

# 1) 全局自助/第三方脚本（若之前没生成，补一份；已存在则跳过）
if [ ! -f "$ROOT/public/_injections/client_topup.js" ]; then
  cat > "$ROOT/public/_injections/client_topup.js" <<'JS'
(function (){
  if (window.netsClientTopup) return;
  window.netsClientTopup = function (p) {
    const ok = p && p.provider && p.phone && p.amount>=0;
    if (!ok) { console.warn("[netsClientTopup] bad payload", p); return; }
    const item = {
      id: (self.crypto||{}).randomUUID? crypto.randomUUID(): String(Date.now()),
      createdAt: Date.now(),
      provider: String(p.provider),
      phone: String(p.phone),
      amount: Math.round(+p.amount||0),
      bonus: Math.round(+p.bonus||0),
      note: p.note? String(p.note): ""
    };
    try {
      const KEY="netcafe:client_topups";
      const arr = JSON.parse(localStorage.getItem(KEY)||"[]");
      arr.unshift(item);
      localStorage.setItem(KEY, JSON.stringify(arr));
      console.log("[netcafe] recorded client topup:", item);
    } catch(e){ console.error(e); }
  };
  window.netsListClientTopups = function(){
    try { return JSON.parse(localStorage.getItem("netcafe:client_topups")||"[]"); } catch { return []; }
  };
  window.netsClearClientTopups = function(){
    try { localStorage.removeItem("netcafe:client_topups"); console.log("[netcafe] cleared client topups"); } catch(e){}
  };
})();
JS
fi

# 2) ClientTopupLoader：在 client 端把注入脚本挂进来
cat > "$ROOT/components/ClientTopupLoader.tsx" <<'TSX'
"use client";
import React, { useEffect } from "react";
export default function ClientTopupLoader(){
  useEffect(()=>{
    if (typeof window === "undefined") return;
    if ((window as any).netsClientTopup) return;
    const s = document.createElement("script");
    s.src = "/_injections/client_topup.js";
    s.async = true;
    document.head.appendChild(s);
    return ()=>{ try{ document.head.removeChild(s); }catch{} };
  },[]);
  return null;
}
TSX

# 3) QuickDock：右下角悬浮操作条（高频动作 + 快速导航）
cat > "$ROOT/components/QuickDock.tsx" <<'TSX'
"use client";
import React from "react";
import { open } from "@/src/bus";

function Btn(p:{label:string; onClick:()=>void; tone?:'ok'|'warn'|'ghost'}) {
  const base: React.CSSProperties = {
    height: 34, padding: "0 10px", borderRadius: 8, cursor: "pointer",
    border: "1px solid var(--border)", background: "var(--input-bg)", color: "var(--text)",
    whiteSpace: "nowrap"
  };
  const tone = p.tone==='ok' ? { background: "var(--focus)", color: "#fff", borderColor: "var(--focus)"} :
               p.tone==='warn' ? { background: "rgba(255,140,0,.2)" } : {};
  return <button onClick={p.onClick} style={{...base, ...tone}}>{p.label}</button>;
}

export default function QuickDock(){
  const wrap: React.CSSProperties = {
    position:"fixed", right:16, bottom:16, zIndex: 50,
    display:"flex", gap:8, flexWrap: "wrap", maxWidth: "92vw",
    background:"rgba(0,0,0,.15)", border:"1px solid var(--border)", padding:10, borderRadius:12, backdropFilter:"blur(2px)"
  };
  return (
    <div style={wrap}>
      <Btn label="新会员" onClick={()=>open("newMember")} tone="ok"/>
      <Btn label="充值加款" onClick={()=>open("recharge")}/>
      <Btn label="商品售卖" onClick={()=>location.href="/pos"}/>
      <Btn label="交接班" onClick={()=>location.href="/shift"}/>
      <Btn label="统计报表" onClick={()=>location.href="/reports"}/>
      <Btn label="店主模式" onClick={()=>open("owner")} />
      <Btn label="刷新" onClick={()=>location.reload()}/>
      <Btn label="退出" onClick={()=>location.href="/logout"} tone="warn"/>
    </div>
  );
}
TSX

# 4) Patch Providers.tsx：自动引入并渲染 <ClientTopupLoader/> 与 <QuickDock/>
P="$ROOT/app/(console)/Providers.tsx"
if [ ! -f "$P" ]; then
  echo "❌ Providers.tsx 不存在：$P"
  exit 1
fi

cp "$P" "$P.bak.$(date +%Y%m%d-%H%M%S)"

# 4.1 插入 import
if ! grep -q 'ClientTopupLoader' "$P"; then
  awk '
    BEGIN{done=0}
    /^import / && done==0 { print $0; next }
    !/^import / && done==0 {
      print "import ClientTopupLoader from \"@/components/ClientTopupLoader\";"
      print "import QuickDock from \"@/components/QuickDock\";";
      done=1
    }
    { print $0 }
  ' "$P" > "$P.tmp" && mv "$P.tmp" "$P"
fi

# 4.2 在 JSX 中 {children} 后面挂载组件（第一次出现处）
awk '
  BEGIN{inj=0}
  {
    print $0
    if (inj==0 && /\{ *children *\}/) {
      print "        <ClientTopupLoader />"
      print "        <QuickDock />"
      inj=1
    }
  }
' "$P" > "$P.tmp" && mv "$P.tmp" "$P"

echo "✅ QuickDock + ClientTopupLoader 已挂载进 Providers.tsx"
echo "👉 现在运行：npm run dev，然后打开 /dashboard、/shift、/reports 页面自测。"
SH

chmod +x tools/apply_ui_overlay_v3.sh
bash tools/apply_ui_overlay_v3.sh
