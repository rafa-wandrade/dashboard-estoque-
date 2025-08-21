import React, { useEffect, useMemo, useRef, useState } from "react";
import Papa from "papaparse";
import { PieChart, Pie, Tooltip, Cell, Legend, ResponsiveContainer } from "recharts";
import { motion } from "framer-motion";

// ========================================================
// Dashboard de Estoque – Flex por Upload (Categoria Opcional)
// - Exibe/oculta coluna e donut de CATEGORIA por upload, conforme presença da coluna
// - Mantém donut por PRODUTO
// - Tabela sempre mostra TOTAL(ais) por unidade de medida no rodapé
// ========================================================
const LS_KEY = "stockDashboard_v1";

// ------------------------------
// Utils
// ------------------------------
function slugify(s = "") {
  return String(s)
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "_")
    .replace(/^_|_$/g, "");
}

function normalizeHeaderKey(k) {
  const s = slugify(k);
  const map = {
    // tipo de estoque
    tipo: ["tipo", "tipo_estoque", "type", "estoque", "stock_type", "rebanho", "gado", "stock"],

    // produto (nome do item / animal / identificação)
    produto: [
      "produto", "product", "item", "nome_produto", "nome",
      "animal", "descricao", "descr", "identificacao", "brinco",
    ],

    // categoria (classe/sexo/lote/grupo)
    categoria: ["categoria", "category", "grupo", "setor", "classe", "sexo", "lote", "fase"],

    // quantidade (contagem de cabeças / itens)
    quantidade: [
      "quantidade", "qtd", "qtde", "quantity", "quant", "qtdade", "amount",
      "cabecas", "cabeca", "cabecas_", "cabeca_", "cabecas_totais", "cabeças", "cabeça",
      "qtd_cabecas", "qtd_animais", "n_cabecas", "numero_de_cabecas", "headcount", "count", "animals",
      "quant_total"
    ],

    // unidade de medida
    unidade_de_medida: [
      "unidade_de_medida", "unidade", "unidade_medida", "unidade_de_med", "unidade-medida",
      "unid_medida", "unit", "unit_of_measure", "uom", "cabeca", "cabeça", "cabecas", "cabeças",
    ],

    // opcionais
    id: ["id"],
    created_at: ["created_at", "data_criacao", "criado_em", "timestamp"],

    // métricas alternativas
    peso_kg: ["peso", "peso_kg", "kg", "peso_total", "peso_em_kg"],
  };
  for (const [canon, aliases] of Object.entries(map)) if (aliases.includes(s)) return canon;
  return s;
}

function parseLocaleNumber(val) {
  if (typeof val === "number") return Number.isFinite(val) ? val : 0;
  if (val == null) return 0;
  let s = String(val).trim();
  if (s === "") return 0;
  const lastComma = s.lastIndexOf(",");
  const lastDot = s.lastIndexOf(".");
  let decimalSep = null;
  if (lastComma >= 0 && lastDot >= 0) decimalSep = lastComma > lastDot ? "," : ".";
  else if (lastComma >= 0) decimalSep = ",";
  else if (lastDot >= 0) decimalSep = ".";
  if (decimalSep) {
    const groupSep = decimalSep === "," ? "." : ",";
    s = s.replace(new RegExp("\\" + groupSep, "g"), "");
    s = s.replace(new RegExp("\\" + decimalSep), ".");
  }
  const n = parseFloat(s.replace(/[^0-9\.-]/g, ""));
  return Number.isFinite(n) ? n : 0;
}

function buildPalette(n) {
  const colors = [];
  const golden = 0.61803398875;
  let h = 0.1;
  for (let i = 0; i < Math.max(0, n); i++) {
    h = (h + golden) % 1;
    const s = 70;
    const l = 50;
    colors.push(`hsl(${Math.round(h * 360)} ${s}% ${l}%)`);
  }
  return colors.length ? colors : ["#8884d8"]; // fallback
}

function safeConfirm(message) {
  try {
    if (typeof window !== "undefined" && typeof window.confirm === "function") {
      return window.confirm(message);
    }
    return true;
  } catch (_err) {
    return true;
  }
}

// ------------------------------
// UI helpers
// ------------------------------
function NiceButton({ children, variant = "solid", ...props }) {
  const base =
    "inline-flex items-center gap-2 rounded-2xl px-4 py-2 text-sm font-medium transition focus:outline-none focus:ring-2 focus:ring-offset-2";
  const styles = {
    solid: "bg-black text-white hover:bg-zinc-800 focus:ring-black",
    outline: "border border-zinc-300 bg-white text-zinc-800 hover:bg-zinc-50 focus:ring-zinc-400",
    danger: "bg-red-600 text-white hover:bg-red-700 focus:ring-red-600",
    ghost: "text-zinc-700 hover:bg-zinc-100",
  };
  return (
    <button className={`${base} ${styles[variant] || styles.solid}`} {...props}>
      {children}
    </button>
  );
}

function Section({ title, actions, children }) {
  return (
    <section className="mb-8 rounded-2xl border bg-white p-5 shadow-sm">
      <div className="mb-4 flex items-center justify-between gap-4">
        <h2 className="text-lg font-semibold">{title}</h2>
        {actions}
      </div>
      {children}
    </section>
  );
}

function Info({ children }) {
  return (
    <div className="mt-3 rounded-xl border border-blue-2 00 bg-blue-50 p-4 text-sm text-blue-900">
      {children}
    </div>
  );
}

function FileInput({ onFile }) {
  const inputRef = useRef(null);
  return (
    <div className="flex flex-wrap items-center gap-3">
      <input
        ref={inputRef}
        type="file"
        accept=".csv,text/csv"
        className="hidden"
        onChange={(e) => {
          const file = e.target.files?.[0];
          if (file) onFile(file);
          e.target.value = ""; // permitir re-upload do mesmo nome
        }}
      />
      <NiceButton onClick={() => inputRef.current?.click()}>📤 Upload CSV</NiceButton>
    </div>
  );
}

// ------------------------------
// Tabela compacta com TOTAL por unidade de medida
// ------------------------------
function DataTable({ rows, showCategory }) {
  const totals = useMemo(() => {
    const acc = new Map();
    rows.forEach((r) => {
      const u = (r.unidade_de_medida ?? "").trim() || "unid";
      acc.set(u, (acc.get(u) || 0) + (Number(r.quantidade) || 0));
    });
    return Array.from(acc, ([unit, sum]) => ({ unit, sum }));
  }, [rows]);

  const headers = ["tipo", "produto", ...(showCategory ? ["categoria"] : []), "quantidade", "unidade_de_medida"];

  return (
    <div className="max-h-80 overflow-y-auto overflow-x-auto rounded-xl border">
      <table className="min-w-max divide-y divide-zinc-200">
        <thead className="bg-zinc-50">
          <tr>
            {headers.map((h) => (
              <th key={h} className="whitespace-nowrap px-4 py-2 text-left text-xs font-semibold uppercase tracking-wide text-zinc-600">
                {h.replaceAll("_", " ")}
              </th>
            ))}
          </tr>
        </thead>
        <tbody className="divide-y divide-zinc-100 bg-white">
          {rows.map((r, i) => (
            <tr key={i} className="hover:bg-zinc-50">
              <td className="whitespace-nowrap px-4 py-2 text-sm">{r.tipo}</td>
              <td className="whitespace-nowrap px-4 py-2 text-sm">{r.produto}</td>
              {showCategory && <td className="whitespace-nowrap px-4 py-2 text-sm">{r.categoria}</td>}
              <td className="whitespace-nowrap px-4 py-2 text-sm">{r.quantidade}</td>
              <td className="whitespace-nowrap px-4 py-2 text-sm">{r.unidade_de_medida}</td>
            </tr>
          ))}
        </tbody>
        <tfoot className="bg-zinc-50">
          {totals.map(({ unit, sum }) => (
            <tr key={unit}>
              <td className="px-4 py-2 text-xs font-semibold uppercase text-zinc-600">Total</td>
              <td className="px-4 py-2 text-sm text-zinc-700">—</td>
              {showCategory && <td className="px-4 py-2 text-sm text-zinc-700">—</td>}
              <td className="px-4 py-2 text-sm font-semibold">{sum}</td>
              <td className="px-4 py-2 text-sm">{unit}</td>
            </tr>
          ))}
        </tfoot>
      </table>
    </div>
  );
}

// ------------------------------
// Donut por CATEGORIA (opcional por upload)
// ------------------------------
function CategoryDonut({ rows }) {
  const data = useMemo(() => {
    const counts = new Map();
    rows.forEach((r) => {
      const k = (r.categoria ?? "").toString().trim();
      if (!k) return;
      counts.set(k, (counts.get(k) || 0) + 1);
    });
    return Array.from(counts, ([name, value]) => ({ name, value })).sort((a,b)=>b.value-a.value);
  }, [rows]);
  const palette = useMemo(() => buildPalette(data.length), [data.length]);
  if (!data.length) return null;
  return (
    <div className="h-[380px] w-full">
      <ResponsiveContainer width="100%" height="100%">
        <PieChart margin={{ top: 10, right: 10, bottom: 60, left: 10 }}>
          <Pie data={data} dataKey="value" nameKey="name" innerRadius={70} outerRadius={110} paddingAngle={1}>
            {data.map((_, i) => (
              <Cell key={i} fill={palette[i % palette.length]} />
            ))}
          </Pie>
          <Tooltip formatter={(value, name) => {
            const total = data.reduce((sum, d) => sum + d.value, 0);
            const pct = total ? ((value / total) * 100).toFixed(1) + "%" : "0%";
            return [pct, name];
          }} />
          <Legend layout="horizontal" verticalAlign="bottom" align="center" />
        </PieChart>
      </ResponsiveContainer>
      <div className="mt-2 text-center text-xs text-zinc-600">Percentual por número de <strong>registros</strong> em cada categoria.</div>
    </div>
  );
}

// ------------------------------
// Donut por PRODUTO (sempre disponível)
// ------------------------------
function ProductsDonutPerUnit({ rows }) {
  const units = useMemo(() => {
    const set = new Set(rows.map((r) => (r.unidade_de_medida ?? "").trim()));
    return Array.from(set).filter(Boolean);
  }, [rows]);

  const [unit, setUnit] = useState(() => units[0] || "");
  useEffect(() => { if (!units.includes(unit)) setUnit(units[0] || ""); }, [units, unit]);

  const data = useMemo(() => {
    const sums = new Map();
    rows
      .filter((r) => (r.unidade_de_medida ?? "").trim() === unit)
      .forEach((r) => {
        const k = (r.produto ?? "(sem produto)").trim() || "(sem produto)";
        sums.set(k, (sums.get(k) || 0) + (Number(r.quantidade) || 0));
      });
    return Array.from(sums, ([name, value]) => ({ name, value })).sort((a,b)=>b.value-a.value);
  }, [rows, unit]);

  const palette = useMemo(() => buildPalette(data.length), [data.length]);

  return (
    <div className="w-full">
      <div className="mb-2 flex flex-wrap items-center justify-between gap-2">
        <div className="text-sm text-zinc-700">Produtos por unidade: <span className="font-medium">{unit || "–"}</span></div>
        <div className="flex flex-wrap gap-2">
          {units.map((u) => (
            <button key={u} onClick={() => setUnit(u)} className={`rounded-full border px-3 py-1 text-xs transition ${unit === u ? "border-black bg-black text-white" : "border-zinc-300 bg-white text-zinc-700 hover:bg-zinc-50"}`}>{u}</button>
          ))}
        </div>
      </div>
      <div className="h-[380px] w-full">
        <ResponsiveContainer width="100%" height="100%">
          <PieChart margin={{ top: 10, right: 10, bottom: 60, left: 10 }}>
            <Pie data={data} dataKey="value" nameKey="name" innerRadius={70} outerRadius={110} paddingAngle={1}>
              {data.map((_, i) => (
                <Cell key={i} fill={palette[i % palette.length]} />
              ))}
            </Pie>
            <Tooltip formatter={(value, name) => {
              const total = data.reduce((sum, d) => sum + d.value, 0);
              const pct = total ? ((value / total) * 100).toFixed(1) + "%" : "0%";
              return [`${pct} | ${value} ${unit}`, name];
            }} />
            <Legend layout="horizontal" verticalAlign="bottom" align="center" />
          </PieChart>
        </ResponsiveContainer>
      </div>
    </div>
  );
}

// ------------------------------
// App principal
// ------------------------------
export default function DashboardEstoqueFlexCategoriaOpcional() {
  const [uploads, setUploads] = useState(() => {
    try {
      const raw = localStorage.getItem(LS_KEY);
      const parsed = raw ? JSON.parse(raw) : [];
      return Array.isArray(parsed) ? parsed : [];
    } catch (_err) {
      return [];
    }
  });

  useEffect(() => {
    try { localStorage.setItem(LS_KEY, JSON.stringify(uploads)); } catch (_err) {}
  }, [uploads]);

  function normalizeRow(row) {
    const normalized = {};
    for (const [k, v] of Object.entries(row)) normalized[normalizeHeaderKey(k)] = v;

    // base
    let tipo = String(normalized.tipo ?? "").trim();
    let produto = String(normalized.produto ?? "").trim();
    let categoria = String(normalized.categoria ?? "").trim();
    let unidade_de_medida = String(normalized.unidade_de_medida ?? normalized.unidade ?? "").trim();

    // quantidade preferencial: quantidade -> peso_kg
    let quantidade = parseLocaleNumber(normalized.quantidade);
    if (!Number.isFinite(quantidade) || quantidade === 0) {
      const q2 = parseLocaleNumber(normalized.peso_kg);
      if (q2) {
        quantidade = q2;
        if (!unidade_de_medida) unidade_de_medida = "kg";
      }
    }

    // Defaults/inteligência para gado
    if (!tipo) tipo = "gado"; // inferimos gado por padrão quando ausente

    // se não veio unidade, e quantidade parece contagem inteira, usamos "cabeça"
    if (!unidade_de_medida) {
      const isIntLike = Number.isFinite(quantidade) && Math.abs(quantidade - Math.round(quantidade)) < 1e-9;
      unidade_de_medida = isIntLike ? "cabeça" : "unid";
    }

    // heranças leves
    if (!categoria && produto) categoria = ""; // não impor categoria onde não existe

    const created_at = normalized.created_at ?? "";
    const id = normalized.id ?? "";

    return { id, created_at, tipo, produto, categoria, quantidade: Number.isFinite(quantidade) ? quantidade : 0, unidade_de_medida };
  }

  async function handleFile(file) {
    const text = await file.text();
    return new Promise((resolve) => {
      Papa.parse(text, {
        header: true,
        skipEmptyLines: true,
        encoding: "UTF-8",
        complete: (results) => {
          let rows = (results.data || []).map(normalizeRow)
            // filtra linhas vazias (aceita se tiver produto OU categoria)
            .filter((r) => (r.produto || r.categoria))
            // ignora linhas de total/soma vindo do arquivo
            .filter((r) => !String(r.produto || r.categoria).trim().toLowerCase().match(/^total(es)?$/));

          if (!rows.length) {
            alert("Arquivo vazio ou cabeçalhos não reconhecidos.");
            return resolve();
          }

          // Inferir/definir tipo
          let tipo = (rows.find((r) => r.tipo)?.tipo) || (slugify(file.name).includes("gado") ? "gado" : "estoque");

          // Evitar duplicar uploads pelo mesmo tipo
          if (uploads.some((u) => (u.tipo || "").toLowerCase() === String(tipo).toLowerCase())) {
            alert(`Já existe um upload para o tipo "${tipo}". Remova o existente (\"Remover último upload\" ou \"Limpar tudo\").`);
            return resolve();
          }

          // normalizar unidade "cabeça"/"cabeças"
          rows = rows.map((r) => ({
            ...r,
            unidade_de_medida: r.unidade_de_medida?.toLowerCase() === "cabeças" ? "cabeça" : r.unidade_de_medida,
          }));

          const upload = { tipo, fileName: file.name, uploadedAt: new Date().toISOString(), rows };
          setUploads((prev) => [...prev, upload]);
          resolve();
        },
        error: () => { alert("Falha ao ler o CSV. Verifique o arquivo."); resolve(); },
      });
    });
  }

  function clearAll() {
    if (!uploads.length) return;
    const ok = safeConfirm("Tem certeza que deseja limpar TODOS os uploads?");
    if (!ok) return;
    setUploads(() => {
      const next = [];
      try { localStorage.setItem(LS_KEY, JSON.stringify(next)); } catch (_err) {}
      return next;
    });
  }

  function removeLastUpload() {
    if (!uploads.length) return;
    const last = uploads[uploads.length - 1];
    const ok = safeConfirm(`Remover o último upload? (tipo: ${last.tipo}, arquivo: ${last.fileName || "(desconhecido)"})`);
    if (!ok) return;
    setUploads((prev) => {
      const next = prev.slice(0, -1);
      try { localStorage.setItem(LS_KEY, JSON.stringify(next)); } catch (_err) {}
      return next;
    });
  }

  function hardReset() {
    try { localStorage.removeItem(LS_KEY); } catch (_err) {}
    setUploads([]);
    try {
      if (typeof location !== "undefined" && typeof location.reload === "function") {
        setTimeout(() => location.reload(), 50);
      }
    } catch (_err) {}
  }

  return (
    <div className="mx-auto max-w-7xl p-6">
      <header className="mb-8">
        <h1 className="text-2xl font-bold tracking-tight">Dashboard de Estoque (Flex por Upload)</h1>
        <p className="mt-2 max-w-3xl text-sm text-zinc-600">
          Cada upload decide dinamicamente se mostra <strong>categoria</strong>. Tabela com <strong>totais</strong> no rodapé e donuts por <strong>produto</strong> e, quando houver, por <strong>categoria</strong>.
        </p>
      </header>

      <Section
        title="Envio de CSV e controles"
        actions={
          <div className="flex flex-wrap gap-2">
            <NiceButton variant="outline" onClick={removeLastUpload} disabled={!uploads.length}>↩️ Remover último upload</NiceButton>
            <NiceButton variant="danger" onClick={clearAll} disabled={!uploads.length}>🗑️ Limpar tudo</NiceButton>
            <NiceButton variant="ghost" onClick={hardReset}>🧹 Recuperar (reset armazenamento)</NiceButton>
          </div>
        }
      >
        <div className="flex flex-col gap-4 lg:flex-row lg:items-start">
          <FileInput onFile={handleFile} />
          <div className="grow" />
        </div>

        <Info>
          <div className="mb-1 font-medium">Formato do CSV aceito (flexível)</div>
          <ul className="ml-4 list-disc space-y-1 text-sm">
            <li>Aliases: <code>animal</code> → <code>produto</code>; <code>cabecas/cabeças</code> → <code>quantidade</code>; <code>peso/peso_kg</code> para kg.</li>
            <li>Se faltar <code>tipo</code>, inferimos <code>gado</code>. Se faltar <code>unidade_de_medida</code>, usamos <code>cabeça</code> para contagens inteiras.</li>
            <li>Colunas entendidas: <code>tipo</code>, <code>produto</code>, <code>categoria</code>, <code>quantidade</code>, <code>unidade_de_medida</code>, <code>id</code>, <code>created_at</code>, <code>peso_kg</code>.</li>
            <li>Linhas de <em>total</em> do arquivo são ignoradas; o total exibido no rodapé é calculado aqui.</li>
          </ul>
        </Info>
      </Section>

      {!uploads.length ? (
        <div className="rounded-2xl border border-dashed p-10 text-center text-zinc-600">Nenhum upload ainda. Use o botão <strong>Upload CSV</strong> acima.</div>
      ) : (
        <div className="space-y-8">
          {uploads.map((u, idx) => {
            const hasCategory = u.rows.some((r) => (r.categoria ?? "").toString().trim());
            return (
              <motion.div key={`${u.tipo}-${idx}`} initial={{ opacity: 0, y: 12 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.35, delay: idx * 0.05 }}>
                <section className="rounded-2xl border bg-white p-5 shadow-sm">
                  <div className="mb-4 flex flex-wrap items-center justify-between gap-2">
                    <div>
                      <h3 className="text-xl font-semibold capitalize">{u.tipo}</h3>
                      <div className="mt-1 text-xs text-zinc-500">{u.rows.length} registros • arquivo: {u.fileName || "(desconhecido)"}</div>
                    </div>
                    <div className="flex items-center gap-2 text-xs text-zinc-500">
                      <div>Enviado em {(() => { try { return new Date(u.uploadedAt).toLocaleString(); } catch (_err) { return ""; } })()}</div>
                    </div>
                  </div>

                  <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
                    <div>
                      <h4 className="mb-2 text-sm font-medium text-zinc-700">Tabela (somente leitura)</h4>
                      <DataTable rows={u.rows} showCategory={hasCategory} />
                    </div>

                    <div className="space-y-6">
                      {hasCategory && (
                        <div className="rounded-xl border p-3">
                          <h4 className="mb-2 text-sm font-medium text-zinc-700">Categorias (%)</h4>
                          <CategoryDonut rows={u.rows} />
                        </div>
                      )}

                      <div className="rounded-xl border p-3">
                        <h4 className="mb-2 text-sm font-medium text-zinc-700">Produtos (% / quantidade por unidade)</h4>
                        <ProductsDonutPerUnit rows={u.rows} />
                      </div>
                    </div>
                  </div>
                </section>
              </motion.div>
            );
          })}
        </div>
      )}

      <footer className="mt-10 text-center text-xs text-zinc-500">Dica: os dados ficam salvos no navegador (localStorage). Use “Limpar tudo” para começar do zero ou “Remover último upload” para desfazer o último envio.</footer>
    </div>
  );
}

// ------------------------------
// DEV: pequenos testes dos utilitários (não afetam UI)
// ------------------------------
(function runDevTests() {
  try {
    console.assert(parseLocaleNumber("1.234,56") === 1234.56, "parseLocaleNumber pt-BR falhou");
    console.assert(parseLocaleNumber("1,234.56") === 1234.56, "parseLocaleNumber en-US falhou");
  } catch (_err) {
    // silencioso
  }
})();
