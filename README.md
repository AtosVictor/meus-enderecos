import React, { useState } from "react";
import { Plus, Trash2, MapPin, Download, Upload } from "lucide-react";

// --- Geocoding via Nominatim (OpenStreetMap) ------------------------------
async function geocodeAddress(address) {
  const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(address)}`;
  const res = await fetch(url, { headers: { "Accept-Language": "pt-BR" } });
  if (!res.ok) throw new Error("Falha ao geocodificar");
  const data = await res.json();
  if (!data.length) throw new Error("Endereço não encontrado");
  const { lat, lon, display_name } = data[0];
  return { lat: parseFloat(lat), lng: parseFloat(lon), label: display_name };
}

function StopRow({ stop, onRemove }) {
  return (
    <div className="flex items-center gap-3 p-3 rounded-xl bg-white border shadow-sm">
      <MapPin className="w-4 h-4 text-emerald-600" />
      <div className="flex-1">
        <div className="font-medium">{stop.label}</div>
        <div className="text-xs text-gray-500">{stop.lat.toFixed(5)}, {stop.lng.toFixed(5)}</div>
      </div>
      <button className="p-2 rounded-lg hover:bg-red-50" onClick={() => onRemove(stop.id)} title="Remover">
        <Trash2 className="w-4 h-4" />
      </button>
    </div>
  );
}

export default function EnderecosApp() {
  const [stops, setStops] = useState([]);
  const [addr, setAddr] = useState("");
  const [manual, setManual] = useState(false);
  const [lat, setLat] = useState("");
  const [lng, setLng] = useState("");
  const [error, setError] = useState("");

  async function addStop() {
    setError("");
    try {
      let payload;
      if (manual) {
        if (!lat || !lng) throw new Error("Preencha lat e lng");
        payload = { lat: parseFloat(lat), lng: parseFloat(lng), label: addr || `Parada ${stops.length + 1}` };
      } else {
        payload = await geocodeAddress(addr);
      }
      const item = { id: Date.now(), ...payload };
      setStops((s) => [...s, item]);
      setAddr(""); setLat(""); setLng("");
    } catch (e) {
      setError(e.message || "Erro ao adicionar endereço");
    }
  }

  function removeStop(id) {
    setStops((s) => s.filter((x) => x.id !== id));
  }

  function downloadJSON() {
    const blob = new Blob([JSON.stringify({ enderecos: stops }, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url; a.download = "enderecos.json"; a.click();
    URL.revokeObjectURL(url);
  }

  function importJSON(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result);
        if (Array.isArray(data.enderecos)) setStops(data.enderecos);
      } catch {}
    };
    reader.readAsText(file);
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-gray-50 to-white">
      {/* Header */}
      <header className="sticky top-0 z-10 backdrop-blur bg-white/70 border-b">
        <div className="max-w-3xl mx-auto px-4 py-3 flex items-center justify-between">
          <span className="font-semibold flex items-center gap-2"><MapPin className="w-5 h-5"/> Meus Endereços</span>
          <div className="flex items-center gap-2">
            <label className="text-xs px-3 py-1 rounded-full bg-gray-100 border cursor-pointer">
              <input type="file" accept="application/json" className="hidden" onChange={importJSON} />
              <span className="inline-flex items-center gap-2"><Upload className="w-3 h-3"/>Importar</span>
            </label>
            <button className="text-xs px-3 py-1 rounded-full bg-gray-100 border inline-flex items-center gap-2" onClick={downloadJSON}><Download className="w-3 h-3"/>Exportar</button>
          </div>
        </div>
      </header>

      {/* Main */}
      <main className="max-w-3xl mx-auto px-4 py-6 space-y-6">
        <div className="p-4 rounded-2xl bg-white border shadow-sm">
          <div className="flex items-center justify-between mb-3">
            <h2 className="font-semibold text-lg">Adicionar endereço</h2>
            <div className="flex items-center gap-2 text-xs">
              <input id="manual" type="checkbox" checked={manual} onChange={(e) => setManual(e.target.checked)} />
              <label htmlFor="manual">Inserir lat/lng manual</label>
            </div>
          </div>

          {!manual && (
            <div className="space-y-3">
              <input className="w-full border rounded-xl p-2" placeholder="Endereço (ex: Av. Paulista 1000, São Paulo, SP)" value={addr} onChange={(e) => setAddr(e.target.value)} />
              <button onClick={addStop} className="w-full inline-flex items-center justify-center gap-2 rounded-xl px-3 py-2 bg-emerald-600 text-white shadow hover:bg-emerald-700">
                <Plus className="w-4 h-4"/> Adicionar
              </button>
            </div>
          )}

          {manual && (
            <div className="grid grid-cols-3 gap-2">
              <input className="col-span-3 border rounded-xl p-2" placeholder="Nome/rotulo (opcional)" value={addr} onChange={(e) => setAddr(e.target.value)} />
              <input className="border rounded-xl p-2" placeholder="lat" value={lat} onChange={(e) => setLat(e.target.value)} />
              <input className="border rounded-xl p-2" placeholder="lng" value={lng} onChange={(e) => setLng(e.target.value)} />
              <button onClick={addStop} className="col-span-3 inline-flex items-center justify-center gap-2 rounded-xl px-3 py-2 bg-emerald-600 text-white shadow hover:bg-emerald-700">
                <Plus className="w-4 h-4"/> Adicionar
              </button>
            </div>
          )}

          {error && <p className="text-sm text-red-600 mt-2">{error}</p>}
        </div>

        <div className="space-y-2">
          {stops.map((s) => (
            <StopRow key={s.id} stop={s} onRemove={removeStop} />
          ))}
        </div>
      </main>

      <footer className="max-w-3xl mx-auto px-4 pb-10 text-xs text-gray-500">
        MVP simples: apenas cadastro e exportação de endereços.
      </footer>
    </div>
  );
}
