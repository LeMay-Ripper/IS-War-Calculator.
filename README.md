
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Imagine Sports WAR-ish Calculator (Standard)</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; margin: 16px; line-height: 1.25; }
    h1 { font-size: 18px; margin: 0 0 8px; }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 10px; }
    label { display: block; font-size: 12px; opacity: 0.85; margin-bottom: 4px; }
    input, select { width: 100%; padding: 8px; font-size: 14px; }
    .row { display: flex; gap: 10px; align-items: end; flex-wrap: wrap; margin-top: 10px; }
    button { padding: 10px 12px; font-size: 14px; cursor: pointer; }
    .out { margin-top: 14px; padding: 12px; border: 1px solid #ddd; border-radius: 10px; }
    .out h2 { font-size: 15px; margin: 0 0 8px; }
    .mono { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; white-space: pre-wrap; }
    .note { font-size: 12px; opacity: 0.85; margin-top: 8px; }
    .warn { color: #8a2b2b; font-size: 12px; margin-top: 8px; }
  </style>
</head>
<body>
  <h1>Imagine Sports WAR-ish Calculator (Standard baselines)</h1>

  <div class="grid">
    <div>
      <label>Player name (optional)</label>
      <input id="name" placeholder="e.g., Pop Lloyd" />
    </div>

    <div>
      <label>Baseline</label>
      <select id="baseline">
        <option value="rep">Replacement (position-adjusted)</option>
        <option value="avg">Average starter (position-adjusted)</option>
      </select>
    </div>

    <div>
      <label>Position (for baselines + error baselines)</label>
      <select id="pos">
        <option>C</option>
        <option>1B</option>
        <option>2B</option>
        <option>3B</option>
        <option>SS</option>
        <option>LF</option>
        <option>CF</option>
        <option>RF</option>
      </select>
    </div>

    <div>
      <label>Playing Time Factor (PT)</label>
      <input id="pt" type="number" step="0.01" value="1.00" />
    </div>

    <div>
      <label>AB</label>
      <input id="ab" type="number" step="1" value="550" />
    </div>
    <div>
      <label>AVG</label>
      <input id="avg" type="number" step="0.001" value="0.260" />
    </div>
    <div>
      <label>OBP</label>
      <input id="obp" type="number" step="0.001" value="0.330" />
    </div>
    <div>
      <label>SLG</label>
      <input id="slg" type="number" step="0.001" value="0.410" />
    </div>

    <div>
      <label>Range grade (Pr/Fr/Av/Vg/Ex) — ignored for C</label>
      <select id="range">
        <option>Pr</option><option>Fr</option><option selected>Av</option><option>Vg</option><option>Ex</option>
      </select>
    </div>

    <div>
      <label>Error % number (the “/X”, e.g. Av/89 → 89)</label>
      <input id="errpct" type="number" step="1" value="100" />
    </div>

    <div>
      <label>Arm grade (Pr/Fr/Av/Vg/Ex) — used for OF + C</label>
      <select id="arm">
        <option>Pr</option><option>Fr</option><option selected>Av</option><option>Vg</option><option>Ex</option>
      </select>
    </div>

    <div>
      <label>Run grade (Pr/Fr/Av/Vg/Ex)</label>
      <select id="run">
        <option>Pr</option><option>Fr</option><option selected>Av</option><option>Vg</option><option>Ex</option>
      </select>
    </div>

    <div>
      <label>PB (catchers only; leave blank = average)</label>
      <input id="pb" type="number" step="1" placeholder="e.g., 3" />
    </div>
  </div>

  <div class="row">
    <button onclick="calc()">Calculate</button>
    <button onclick="resetForm()">Reset</button>
  </div>

  <div class="out">
    <h2>Results</h2>
    <div id="results" class="mono">—</div>
    <div class="note">
      Model rules: Range = 10 runs/band (non-C), Arm = 3 runs/band (OF + C), Run = 3 runs/band,
      Errors = 0.5 runs per error saved vs avg, PB (C only) = 0.25 runs per PB saved vs PBavg=8.
      WAR ≈ TotalRuns / 10.
    </div>
    <div id="warn" class="warn"></div>
  </div>

<script>
  // --- Baselines (Standard) ---
  const Eavg = { "C":14, "1B":14, "2B":22, "3B":24, "SS":29, "LF":8, "CF":8, "RF":8 };
  const RCavg = { "C":65, "SS":68, "2B":72, "CF":75, "3B":78, "LF":82, "RF":84, "1B":88 };
  const RCrep = { "C":45, "SS":48, "2B":52, "CF":55, "3B":58, "LF":62, "RF":64, "1B":68 };

  const bands = { "Pr":-2, "Fr":-1, "Av":0, "Vg":1, "Ex":2 };

  // Constants (your rules)
  const RANGE_RUNS_PER_BAND = 10;
  const ARM_RUNS_PER_BAND = 3;
  const RUN_RUNS_PER_BAND = 3;
  const RUNS_PER_ERROR = 0.5;
  const PB_AVG = 8;
  const RUNS_PER_PB = 0.25;

  function n(id){ return document.getElementById(id); }
  function clamp01(x){ return Math.max(0, Math.min(1, x)); }

  function resetForm(){
    n("name").value = "";
    n("baseline").value = "rep";
    n("pos").value = "C";
    n("pt").value = "1.00";
    n("ab").value = "550";
    n("avg").value = "0.260";
    n("obp").value = "0.330";
    n("slg").value = "0.410";
    n("range").value = "Av";
    n("errpct").value = "100";
    n("arm").value = "Av";
    n("run").value = "Av";
    n("pb").value = "";
    n("results").textContent = "—";
    n("warn").textContent = "";
  }

  function calc(){
    const name = n("name").value.trim() || "(player)";
    const baselineType = n("baseline").value; // "rep" or "avg"
    const pos = n("pos").value;
    const PT = Number(n("pt").value);
    const AB = Number(n("ab").value);
    const AVG = Number(n("avg").value);
    const OBP = Number(n("obp").value);
    const SLG = Number(n("slg").value);

    const rangeG = n("range").value;
    const errPct = Number(n("errpct").value);
    const armG = n("arm").value;
    const runG = n("run").value;
    const pbRaw = n("pb").value.trim();

    let warnings = [];

    if (!(pos in Eavg)) warnings.push("Unknown position for error baselines.");
    if (!(pos in RCavg) || !(pos in RCrep)) warnings.push("Unknown position for RC baselines.");
    if (!(rangeG in bands) || !(armG in bands) || !(runG in bands)) warnings.push("Bad grade input (Pr/Fr/Av/Vg/Ex).");
    if (!isFinite(PT) || PT <= 0) warnings.push("PT should be > 0 (e.g., 1.00 full time, 0.50 half).");
    if (!isFinite(AB) || AB <= 0) warnings.push("AB must be > 0.");
    if (!isFinite(AVG) || AVG < 0 || AVG > 1) warnings.push("AVG must be between 0 and 1.");
    if (!isFinite(OBP) || OBP <= 0 || OBP >= 1) warnings.push("OBP must be between 0 and 1 (not 0 or 1).");
    if (!isFinite(SLG) || SLG < 0 || SLG > 4) warnings.push("SLG must be reasonable (0–4).");
    if (!isFinite(errPct) || errPct <= 0 || errPct > 400) warnings.push("Error % should look like the /X number (e.g., 75, 100, 134).");

    // --- Offense: derive H, TB, estimate BB, compute RC ---
    const H = AVG * AB;
    const TB = SLG * AB;

    // BB estimate from OBP using approximation (ignores HBP/SF)
    const denom = (1 - OBP);
    let BB = 0;
    if (denom > 0){
      BB = ((OBP * AB) - H) / denom;
    }
    if (!isFinite(BB)) BB = 0;
    // Keep BB non-negative (OBP can include HBP etc)
    BB = Math.max(0, BB);

    const RC = ((H + BB) * TB) / (AB + BB);
    const RCbase = baselineType === "rep" ? RCrep[pos] : RCavg[pos];
    const OffRuns = (RC - RCbase) * PT;

    // --- Defense: Range (non-C) ---
    const rangeBands = bands[rangeG];
    const RangeRuns = (pos === "C") ? 0 : (RANGE_RUNS_PER_BAND * rangeBands * PT);

    // --- Defense: Errors (all positions) ---
    const eAvg = Eavg[pos];
    const errorsSaved = eAvg * (1 - (errPct / 100)) * PT; // vs avg (Av/100)
    const ErrorRuns = RUNS_PER_ERROR * errorsSaved;

    // --- Defense: Arm (OF + C) ---
    const armBands = bands[armG];
    const armEligible = (pos === "C" || pos === "RF" || pos === "CF" || pos === "LF");
    const ArmRuns = armEligible ? (ARM_RUNS_PER_BAND * armBands * PT) : 0;

    // --- Running ---
    const runBands = bands[runG];
    const RunRuns = RUN_RUNS_PER_BAND * runBands * PT;

    // --- Catcher PB (ONLY if pos C) ---
    let PB = null;
    if (pbRaw !== "") PB = Number(pbRaw);
    let PBRuns = 0;
    if (pos === "C"){
      if (PB === null || !isFinite(PB)){
        // Treat blank as average -> 0 runs
        PBRuns = 0;
      } else {
        const pbSaved = (PB_AVG - PB) * PT;
        PBRuns = RUNS_PER_PB * pbSaved;
      }
    }

    // --- Totals ---
    const DefRuns = RangeRuns + ErrorRuns + ArmRuns + PBRuns;
    const TotalRuns = OffRuns + DefRuns + RunRuns;
    const WAR = TotalRuns / 10;

    // Output
    const lines = [];
    lines.push(`${name}  (${pos})  | Baseline: ${baselineType === "rep" ? "Replacement" : "Average starter"}`);
    lines.push("");
    lines.push(`OFFENSE`);
    lines.push(`  H = ${H.toFixed(1)}   TB = ${TB.toFixed(1)}   BB(est) = ${BB.toFixed(1)}`);
    lines.push(`  RC = ${RC.toFixed(1)}   RC baseline(${pos}) = ${RCbase.toFixed(1)}`);
    lines.push(`  OffRuns = ${OffRuns.toFixed(1)}`);
    lines.push("");
    lines.push(`DEFENSE + RUNNING (PT=${PT.toFixed(2)})`);
    lines.push(`  RangeRuns = ${RangeRuns.toFixed(1)}  (range ${rangeG}, bands ${rangeBands})`);
    lines.push(`  ErrorRuns = ${ErrorRuns.toFixed(1)}  (Err /${errPct}, avgE ${eAvg})`);
    lines.push(`  ArmRuns   = ${ArmRuns.toFixed(1)}  (${armEligible ? "eligible" : "not used"}, arm ${armG}, bands ${armBands})`);
    lines.push(`  PBRuns    = ${PBRuns.toFixed(1)}  (${pos === "C" ? (PB===null ? "blank=>avg" : "PB="+PB) : "not catcher"})`);
    lines.push(`  RunRuns   = ${RunRuns.toFixed(1)}  (run ${runG}, bands ${runBands})`);
    lines.push("");
    lines.push(`TOTAL`);
    lines.push(`  DefRuns = ${DefRuns.toFixed(1)}`);
    lines.push(`  TotalRuns = ${TotalRuns.toFixed(1)}`);
    lines.push(`  WAR ≈ ${WAR.toFixed(2)}`);

    n("results").textContent = lines.join("\n");
    n("warn").textContent = warnings.length ? ("Notes: " + warnings.join(" ")) : "";
  }
</script>
</body>
</html>
