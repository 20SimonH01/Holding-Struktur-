import React, { useEffect, useRef, useState } from "react";
import * as THREE from "three";

/* ============================================================
   THERMONEXT · HOLDING-BLUEPRINT
   Inhalt: Steuer-Logik (Nodes + Verbindungen)
   ============================================================ */

const GROUND_Y = 0.18;

const NODE_DEFS = [
  {
    id: "gesellschafter",
    x: -4.9, z: -3.1, top: 1.05, labelY: 2.05,
    color: "#fbbf24",
    label: "BAHA & SIMON",
    sublabel: "je 50 %",
    info: {
      kind: "GESELLSCHAFTER",
      title: "Baha Sinan & Simon Heck",
      sub: "Je 50 % an der Thermonext Holding · GF der Thermonext GmbH",
      paras: [
        "Ihr haltet privat nur noch die Holding. Alles darunter gehört euch mittelbar – steuerlich und haftungsseitig getrennt.",
        "Privat fließt, was ihr zum Leben braucht: GF-Gehalt aus der Thermonext GmbH (dort Betriebsausgabe) plus moderate Ausschüttungen.",
        "Jeder Euro privat kostet 26,375 % Abgeltungsteuer. Jeder Euro in der Holding: effektiv ~1,5 %.",
        "Bei zwei Gesellschaftern entscheidet ihr gemeinsam über Ausschüttungen – regelt das in einer Gesellschaftervereinbarung. Ausbaustufe später: je eine persönliche Holding pro Kopf.",
      ],
      law: ["§ 32d EStG (Abgeltungsteuer)", "§ 32d Abs. 2 Nr. 3 EStG (TEV-Option)"],
      calc:
        "Privat-Ausschüttung 100.000 € (50/50):\n25 % AbgSt + Soli = 26.375 € Steuer\n→ Netto gesamt: 73.625 €\n→ Regel: privat nur so viel wie nötig.",
      warn: "Bei ≥ 25 % Beteiligung kann das Teileinkünfteverfahren günstiger sein (z. B. bei Finanzierungszinsen) – rechnen lassen.",
    },
  },
  {
    id: "holding",
    x: 0.1, z: -2.8, top: 3.0, labelY: 4.3,
    color: "#ff8a3c",
    label: "THERMONEXT HOLDING",
    sublabel: "Spardose · Exit-Vehikel",
    info: {
      kind: "GESELLSCHAFT",
      title: "Thermonext Holding GmbH",
      sub: "Muttergesellschaft · Spardose · Exit-Vehikel",
      paras: [
        "Empfängt Gewinnausschüttungen der Töchter zu 95 % steuerfrei – effektiv ~1,5 % Steuerlast.",
        "Reinvestiert das Kapital: neue Töchter, Zukäufe, Depot, Immobilien-Finanzierung. Im Depot sind Aktien-Kursgewinne ebenfalls 95 % frei; Streubesitz-Dividenden (< 10 %) dagegen voll steuerpflichtig (§ 8b Abs. 4 KStG).",
        "Beim Verkauf einer Tochter (Exit) sind auch Veräußerungsgewinne zu 95 % steuerfrei.",
        "Weg dorthin: Holding gründen, dann bringt ihr beide eure Thermonext-Anteile in einem einheitlichen Vorgang zu Buchwerten ein – löst keine Steuer aus.",
      ],
      law: ["§ 8b Abs. 1, 2, 5 KStG", "§ 9 Nr. 2a GewStG", "§ 21 UmwStG"],
      calc:
        "Ausschüttung 100.000 € Thermonext → Holding:\nsteuerpflichtig: 5 % = 5.000 €\nSteuer ≈ 30 % darauf ≈ 1.500 € (eff. 1,5 %)\n→ Bleiben: 98.500 € zum Investieren\nOhne Holding (privat): nur 73.625 €",
      warn: "Sperrfrist: Verkauft die Holding die eingebrachte Thermonext GmbH innerhalb von 7 Jahren, wird rückwirkend anteilig nachversteuert (Abschmelzung 1/7 pro Jahr, § 22 UmwStG).",
    },
  },
  {
    id: "shk",
    x: -3.8, z: 2.5, top: 1.5, labelY: 2.6,
    color: "#4cc3ff",
    label: "THERMONEXT GmbH",
    sublabel: "operativ · SHK",
    info: {
      kind: "GESELLSCHAFT",
      title: "Thermonext GmbH (operativ)",
      sub: "Euer bestehender Betrieb · Umsatz, Personal, Baustellen",
      paras: [
        "Hier läuft das Geschäft – und hier liegt das Risiko (Gewährleistung, Baustellen, Personal). Deshalb: hier kein Vermögen ansammeln.",
        "Zahlt ~30 % Steuern auf den Gewinn (KSt + Soli + GewSt, abhängig vom Hebesatz eurer Gemeinde).",
        "Stellhebel: eure GF-Gehälter + Tantiemen (Betriebsausgabe), Miete an die Immobilien-GmbH, ggf. Pensionszusagen.",
        "Gewinn, den ihr nicht privat braucht, geht als Ausschüttung an die Holding (95 % frei) – nicht an euch privat.",
      ],
      law: ["§ 23 KStG (15 % KSt)", "§ 16 GewStG (Hebesatz)", "§ 8 Abs. 3 S. 2 KStG (vGA)"],
      calc:
        "Gewinn 200.000 € (Hebesatz 400 %):\nKSt + Soli 15,825 % = 31.650 €\nGewSt 14,0 % = 28.000 €\n→ Gesamt ≈ 59.650 € (~30 %)\n→ Rest an Holding: nur ~1,5 % obendrauf.",
      warn: "Gehälter, Tantiemen und Miete müssen fremdüblich sein – sonst verdeckte Gewinnausschüttung (vGA).",
    },
  },
  {
    id: "immo",
    x: 3.7, z: 2.4, top: 1.45, labelY: 2.55,
    color: "#2dd4bf",
    label: "THERMONEXT IMMO",
    sublabel: "Halle · Büro · Lager",
    info: {
      kind: "GESELLSCHAFT",
      title: "Thermonext Immobilien GmbH",
      sub: "Hält die Betriebsimmobilie · vermietet an die Thermonext GmbH",
      paras: [
        "Kauft und hält Halle, Büro und Lager und vermietet sie fremdüblich an die Thermonext GmbH.",
        "Die Miete ist drüben Betriebsausgabe (~30 % Ersparnis) und wird hier bei reiner Vermietung nur mit 15,825 % besteuert – dank erweiterter Gewerbesteuer-Kürzung.",
        "Zusätzlich: Die Immobilie ist vom Betriebsrisiko getrennt. Geht operativ etwas schief, bleibt die Halle unangetastet.",
      ],
      law: ["§ 9 Nr. 1 S. 2 GewStG (erw. Kürzung)", "§ 8 Nr. 1e GewStG (Freibetrag 200.000 €)"],
      calc:
        "Miete 60.000 €/Jahr:\nThermonext GmbH spart: ~30 % = 18.000 €\nImmo-GmbH zahlt: 15,825 % = 9.495 €\n→ Vorteil: ~8.500 €/Jahr\n→ + Tilgung & Wertzuwachs außerhalb des Risikos",
      warn: "Erweiterte Kürzung ist fragil: keine Betriebsvorrichtungen mitvermieten, Betriebsaufspaltung sauber vermeiden – Gestaltung zwingend mit dem Steuerberater festzurren.",
    },
  },
  {
    id: "zukunft",
    x: 4.9, z: -2.6, top: 1.7, labelY: 2.85,
    color: "#22d3ee",
    label: "WACHSTUM",
    sublabel: "nächste Tochter",
    info: {
      kind: "WACHSTUMS-SLOT",
      title: "Hier docken neue Töchter an",
      sub: "2. Standort · Wärmepumpen/PV-Tochter · Zukäufe",
      paras: [
        "Die Nachfolgewelle im Handwerk ist eure Chance: Viele SHK-Betriebe suchen Käufer. Die Holding kauft sie als eigene Töchter.",
        "Sie zahlt mit Kapital, das nur ~1,5 % Steuer gesehen hat. Privat müsstet ihr erst 26,375 % abgeben, bevor ihr investieren könnt.",
        "Jede Tochter ist eigenständig: Risiko getrennt, einzeln verkaufbar – der Verkauf ist zu 95 % steuerfrei.",
      ],
      law: ["§ 8b Abs. 2 KStG (Exit)", "§ 8b Abs. 1 KStG"],
      calc:
        "Exit-Beispiel – Verkauf einer Tochter für 1.000.000 €:\nPrivat gehalten (TEV): ≈ 270.000 € Steuer\nIn der Holding: 5 % × ~30 % = 15.000 €\n→ Differenz: ~255.000 €",
      warn: null,
    },
  },
];

const CONN_DEFS = [
  {
    id: "c-ges-hold", from: "gesellschafter", to: "holding", kind: "structure",
    color: "#8fa3c2", lift: 0.9, side: 0, label: "2 × 50 %",
    info: {
      kind: "BETEILIGUNG",
      title: "Baha & Simon → Holding: je 50 %",
      sub: "Der entscheidende Einbringungsschritt",
      paras: [
        "Ihr gründet die Holding gemeinsam (25.000 € Stammkapital) und bringt beide eure Thermonext-Anteile in einem einheitlichen Vorgang ein.",
        "Nur so klappt es steuerneutral: Einzeln wären eure je 50 % nicht mehrheitsvermittelnd. Zusammen erhält die Holding 100 % – qualifizierter Anteilstausch zu Buchwerten.",
      ],
      law: ["§ 21 Abs. 1 S. 2 UmwStG", "UmwStE Rn. 21.09", "§ 5 GmbHG"],
      calc: null,
      warn: "7 Jahre Sperrfrist nach Einbringung (§ 22 UmwStG) – Verkauf in der Zeit löst rückwirkende Besteuerung aus (abschmelzend 1/7 pro Jahr).",
    },
  },
  {
    id: "c-hold-shk", from: "holding", to: "shk", kind: "structure",
    color: "#8fa3c2", lift: 0.7, side: -0.55, label: "100 %",
    info: {
      kind: "BETEILIGUNG",
      title: "Holding → Thermonext GmbH: 100 %",
      sub: "Nach dem Anteilstausch",
      paras: [
        "Die Holding hält 100 % am operativen Betrieb. Ihr kontrolliert mittelbar über eure Holding-Anteile.",
        "Gewinne wandern ab jetzt fast steuerfrei nach oben statt teuer ins Privatvermögen.",
      ],
      law: ["§ 21 UmwStG", "§ 8b KStG"],
      calc: null,
      warn: null,
    },
  },
  {
    id: "c-hold-immo", from: "holding", to: "immo", kind: "structure",
    color: "#8fa3c2", lift: 0.7, side: 0.55, label: "100 %",
    info: {
      kind: "BETEILIGUNG",
      title: "Holding → Immobilien-GmbH: 100 %",
      sub: "Schwester der Thermonext GmbH – bewusst",
      paras: [
        "Die Immo-GmbH hängt als Schwester neben dem Betrieb, nicht darunter. Nur so ist die Immobilie wirklich vom operativen Risiko getrennt.",
        "Hinge sie unter der Thermonext GmbH, würde sie bei einer Insolvenz des Betriebs mit in die Masse fallen.",
      ],
      law: ["§ 13 Abs. 2 GmbHG (Haftungstrennung)"],
      calc: null,
      warn: null,
    },
  },
  {
    id: "f-div", from: "shk", to: "holding", kind: "flow",
    color: "#34d399", lift: 2.3, side: 0.75, label: "Dividende · 95 % frei",
    info: {
      kind: "GELDFLUSS",
      title: "Gewinnausschüttung Thermonext → Holding",
      sub: "Das Herzstück der Struktur",
      paras: [
        "Die Thermonext GmbH schüttet ihren versteuerten Gewinn nach oben aus. Bei der Holding bleiben die Bezüge zu 95 % außer Ansatz; nur 5 % gelten als nicht abziehbare Betriebsausgaben.",
        "Gewerbesteuerlich greift das Schachtelprivileg (Beteiligung ≥ 15 %).",
      ],
      law: ["§ 8b Abs. 1 + 5 KStG", "§ 9 Nr. 2a GewStG"],
      calc:
        "100.000 € Ausschüttung:\nsteuerpflichtig 5.000 € × ~30 % ≈ 1.500 €\n→ Effektivbelastung: 1,5 %\nZum Vergleich privat: 26.375 €",
      warn: "Praxis: Die Thermonext GmbH behält 25 % Kapitalertragsteuer ein; die Holding bekommt sie angerechnet/erstattet. Liquiditätsthema – Dauerüberzahler-Bescheinigung (§ 44a Abs. 5 EStG) prüfen.",
    },
  },
  {
    id: "f-miete", from: "shk", to: "immo", kind: "flow",
    color: "#60a5fa", lift: 1.5, side: 0, label: "Miete (fremdüblich)",
    info: {
      kind: "GELDFLUSS",
      title: "Miete Thermonext → Immobilien-GmbH",
      sub: "Steuersatz-Gefälle nutzen",
      paras: [
        "Die Miete verlässt die 30-%-Welt (Betriebsausgabe beim Betrieb) und landet in der 15,8-%-Welt der Immo-GmbH.",
        "Beim Mieter droht zwar GewSt-Hinzurechnung von Mieten – aber erst oberhalb des Freibetrags von 200.000 €. Bei eurer Größe: irrelevant.",
      ],
      law: ["§ 9 Nr. 1 S. 2 GewStG", "§ 8 Nr. 1e GewStG"],
      calc:
        "60.000 € Miete/Jahr:\nErsparnis Thermonext GmbH: 18.000 €\nSteuer Immo-GmbH: 9.495 €\n→ Netto-Vorteil: ~8.500 €/Jahr – jedes Jahr",
      warn: "Miethöhe muss dem Fremdvergleich standhalten, sonst vGA.",
    },
  },
  {
    id: "f-privat", from: "holding", to: "gesellschafter", kind: "flow",
    color: "#f59e0b", lift: 2.0, side: 0.7, label: "Ausschüttung privat",
    info: {
      kind: "GELDFLUSS",
      title: "Ausschüttung Holding → privat",
      sub: "Die teuerste Leitung im System",
      paras: [
        "Erst hier entsteht die volle private Steuer. Deshalb fließt hier nur, was ihr wirklich privat braucht – der Rest arbeitet in der Holding weiter (Steuerstundungseffekt).",
        "Als 50/50-Gesellschafter beschließt ihr Ausschüttungen gemeinsam.",
      ],
      law: ["§ 32d EStG", "§ 43 EStG (KapESt)"],
      calc:
        "100.000 € privat:\nAbgeltungsteuer + Soli 26,375 % = 26.375 €\n→ Netto: 73.625 €",
      warn: null,
    },
  },
  {
    id: "f-kapital", from: "holding", to: "zukunft", kind: "flow",
    color: "#22d3ee", lift: 1.6, side: 0.4, label: "Kapital für Zukäufe",
    info: {
      kind: "GELDFLUSS",
      title: "Kapital Holding → neue Töchter",
      sub: "Wachstum mit Brutto-Kapital",
      paras: [
        "Die Holding finanziert Gründungen und Zukäufe direkt aus ihren fast unversteuerten Rücklagen – als Eigenkapital oder Gesellschafterdarlehen.",
      ],
      law: ["§ 8b Abs. 1 + 2 KStG"],
      calc:
        "Zukauf für 500.000 €:\nAus der Holding: direkt verfügbar\nPrivat nötig dafür: ~679.000 € ausschütten\n→ Differenz: 179.000 €",
      warn: null,
    },
  },
];

/* ============================================================
   HELFER
   ============================================================ */

const clamp = (v, a, b) => Math.min(b, Math.max(a, v));

function roundedRectShape(w, d, r) {
  const s = new THREE.Shape();
  const hw = w / 2, hd = d / 2;
  s.moveTo(-hw + r, -hd);
  s.lineTo(hw - r, -hd);
  s.quadraticCurveTo(hw, -hd, hw, -hd + r);
  s.lineTo(hw, hd - r);
  s.quadraticCurveTo(hw, hd, hw - r, hd);
  s.lineTo(-hw + r, hd);
  s.quadraticCurveTo(-hw, hd, -hw, hd - r);
  s.lineTo(-hw, -hd + r);
  s.quadraticCurveTo(-hw, -hd, -hw + r, -hd);
  return s;
}

/* Scharfe Glas-Labels, 2x gerendert, optional zweizeilig */
function makeLabel(main, sub, accent, mainPx, worldH) {
  const R = 2;
  const cv = document.createElement("canvas");
  const mctx = cv.getContext("2d");
  const mainFont = "700 " + mainPx * R + "px system-ui, -apple-system, sans-serif";
  const subPx = Math.round(mainPx * 0.6);
  const subFont = "600 " + subPx * R + "px system-ui, -apple-system, sans-serif";
  mctx.font = mainFont;
  let tw = mctx.measureText(main).width;
  if (sub) {
    mctx.font = subFont;
    tw = Math.max(tw, mctx.measureText(sub).width);
  }
  const padX = 18 * R;
  const topPad = 9 * R;
  const gap = 4 * R;
  const botPad = 9 * R;
  cv.width = Math.ceil(tw + padX * 2);
  cv.height = Math.ceil(topPad + mainPx * R + (sub ? gap + subPx * R : 0) + botPad);
  const c = cv.getContext("2d");
  const w = cv.width, h = cv.height, rad = Math.min(14 * R, h / 2);

  c.beginPath();
  c.moveTo(rad, 0); c.lineTo(w - rad, 0); c.quadraticCurveTo(w, 0, w, rad);
  c.lineTo(w, h - rad); c.quadraticCurveTo(w, h, w - rad, h);
  c.lineTo(rad, h); c.quadraticCurveTo(0, h, 0, h - rad);
  c.lineTo(0, rad); c.quadraticCurveTo(0, 0, rad, 0);
  c.closePath();
  c.fillStyle = "rgba(6,10,20,0.84)";
  c.fill();
  c.lineWidth = 1.6 * R;
  c.strokeStyle = accent;
  c.globalAlpha = 0.95;
  c.stroke();
  c.globalAlpha = 1;

  c.textAlign = "center";
  c.textBaseline = "alphabetic";
  c.font = mainFont;
  c.fillStyle = "#f5f8ff";
  c.fillText(main, w / 2, topPad + mainPx * R * 0.86);
  if (sub) {
    c.font = subFont;
    c.fillStyle = "#93a5c4";
    c.fillText(sub, w / 2, h - botPad - subPx * R * 0.1);
  }

  const tex = new THREE.CanvasTexture(cv);
  tex.minFilter = THREE.LinearFilter;
  tex.anisotropy = 4;
  const sp = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex, transparent: true, depthTest: false }));
  const s = worldH / h;
  sp.scale.set(w * s, h * s, 1);
  sp.renderOrder = 20;
  return sp;
}

function makeGlowTexture() {
  const c = document.createElement("canvas");
  c.width = 128; c.height = 128;
  const ctx = c.getContext("2d");
  const g = ctx.createRadialGradient(64, 64, 2, 64, 64, 62);
  g.addColorStop(0, "rgba(255,255,255,1)");
  g.addColorStop(0.3, "rgba(255,255,255,0.4)");
  g.addColorStop(1, "rgba(255,255,255,0)");
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, 128, 128);
  return new THREE.CanvasTexture(c);
}

function prismRoofGeometry(w, h, depth) {
  const s = new THREE.Shape();
  s.moveTo(-w / 2, 0);
  s.lineTo(w / 2, 0);
  s.lineTo(0, h);
  s.closePath();
  const g = new THREE.ExtrudeGeometry(s, { depth, bevelEnabled: false });
  g.translate(0, 0, -depth / 2);
  return g;
}

function box(w, h, d, color, opts) {
  const m = new THREE.Mesh(
    new THREE.BoxGeometry(w, h, d),
    new THREE.MeshStandardMaterial(Object.assign({ color, roughness: 0.82, metalness: 0.08 }, opts || {}))
  );
  m.castShadow = true;
  m.receiveShadow = true;
  return m;
}

function addEdges(mesh, color, opacity) {
  const e = new THREE.LineSegments(
    new THREE.EdgesGeometry(mesh.geometry),
    new THREE.LineBasicMaterial({ color: color || 0x060b16, transparent: true, opacity: opacity || 0.4 })
  );
  mesh.add(e);
}

function addWindows(group, cols, rows, faceW, baseY, gapY, z, litColor) {
  for (let r = 0; r < rows; r++) {
    for (let cIdx = 0; cIdx < cols; cIdx++) {
      const lit = Math.random() > 0.28;
      const w = new THREE.Mesh(
        new THREE.PlaneGeometry(0.2, 0.26),
        new THREE.MeshBasicMaterial({ color: lit ? litColor : "#182238" })
      );
      const x = -faceW / 2 + (faceW / (cols + 1)) * (cIdx + 1);
      w.position.set(x, baseY + r * gapY, z);
      group.add(w);
    }
  }
}

/* Sockelplatte mit Leuchtring in Node-Farbe */
function buildPlinth(radius, accent) {
  const g = new THREE.Group();
  const plate = new THREE.Mesh(
    new THREE.CylinderGeometry(radius, radius + 0.07, 0.1, 44),
    new THREE.MeshStandardMaterial({ color: "#0f1930", roughness: 0.9 })
  );
  plate.position.y = 0.05;
  plate.receiveShadow = true;
  g.add(plate);
  const ring = new THREE.Mesh(
    new THREE.RingGeometry(radius - 0.05, radius, 56),
    new THREE.MeshBasicMaterial({ color: accent, transparent: true, opacity: 0.38, side: THREE.DoubleSide })
  );
  ring.rotation.x = -Math.PI / 2;
  ring.position.y = 0.105;
  g.add(ring);
  return g;
}

/* ============================================================
   GEBÄUDE
   ============================================================ */

function buildHolding(accent) {
  const g = new THREE.Group();
  g.add(buildPlinth(1.6, accent));
  const base = box(1.9, 2.3, 1.9, "#27324e");
  base.position.y = 1.25;
  addEdges(base);
  g.add(base);
  addWindows(base, 3, 4, 1.9, 0.55, 0.5, 0.96, "#ffd9a8");
  const upper = box(1.45, 1.05, 1.45, "#303c5e");
  upper.position.y = 2.92;
  addEdges(upper);
  g.add(upper);
  addWindows(upper, 2, 2, 1.45, 2.7, 0.46, 0.74, "#ffd9a8");
  const roof = box(1.62, 0.12, 1.62, accent, { emissive: accent, emissiveIntensity: 0.55, roughness: 0.35 });
  roof.position.y = 3.5;
  g.add(roof);
  const logoRing = new THREE.Mesh(
    new THREE.TorusGeometry(0.3, 0.045, 10, 36),
    new THREE.MeshBasicMaterial({ color: accent })
  );
  logoRing.position.set(0, 1.7, 0.97);
  g.add(logoRing);
  const mast = box(0.05, 0.7, 0.05, "#93a5c4");
  mast.position.y = 3.9;
  g.add(mast);
  const tip = new THREE.Mesh(new THREE.SphereGeometry(0.09, 12, 12), new THREE.MeshBasicMaterial({ color: accent }));
  tip.position.y = 4.28;
  g.add(tip);
  return g;
}

function buildSHK(accent, spinners) {
  const g = new THREE.Group();
  g.add(buildPlinth(1.8, accent));
  const body = box(2.5, 1.05, 1.9, "#2a3655");
  body.position.y = 0.63;
  addEdges(body);
  g.add(body);
  const roof = new THREE.Mesh(
    prismRoofGeometry(2.7, 0.7, 2.05),
    new THREE.MeshStandardMaterial({ color: accent, roughness: 0.55, flatShading: true })
  );
  roof.castShadow = true;
  roof.position.y = 1.16;
  addEdges(roof, 0x0a2a44, 0.5);
  g.add(roof);
  const gate = new THREE.Mesh(new THREE.PlaneGeometry(1.0, 0.7), new THREE.MeshBasicMaterial({ color: "#0d1526" }));
  gate.position.set(-0.45, 0.5, 0.96);
  g.add(gate);
  const win = new THREE.Mesh(new THREE.PlaneGeometry(0.34, 0.3), new THREE.MeshBasicMaterial({ color: "#ffd9a8" }));
  win.position.set(0.62, 0.68, 0.96);
  g.add(win);

  /* Wärmepumpe mit rotierendem Lüfter – Thermonext-Detail */
  const hp = box(0.52, 0.58, 0.3, "#c7d2e4");
  hp.position.set(1.62, 0.4, 0.5);
  g.add(hp);
  const fanRim = new THREE.Mesh(
    new THREE.TorusGeometry(0.15, 0.02, 8, 24),
    new THREE.MeshBasicMaterial({ color: "#0f172a" })
  );
  fanRim.position.set(1.62, 0.43, 0.66);
  g.add(fanRim);
  const fan = new THREE.Group();
  for (let i = 0; i < 3; i++) {
    const blade = new THREE.Mesh(
      new THREE.BoxGeometry(0.035, 0.12, 0.02),
      new THREE.MeshBasicMaterial({ color: "#31415f" })
    );
    blade.position.y = 0.065;
    const holder = new THREE.Group();
    holder.rotation.z = (i / 3) * Math.PI * 2;
    holder.add(blade);
    fan.add(holder);
  }
  const hub = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 8), new THREE.MeshBasicMaterial({ color: "#0f172a" }));
  fan.add(hub);
  fan.position.set(1.62, 0.43, 0.665);
  g.add(fan);
  spinners.push(fan);

  /* Firmenwagen */
  const van = new THREE.Group();
  const vb = box(0.85, 0.4, 0.44, "#ff8a3c");
  vb.position.set(0, 0.33, 0);
  van.add(vb);
  const cab = box(0.3, 0.3, 0.44, "#ffd7b0");
  cab.position.set(0.55, 0.28, 0);
  van.add(cab);
  const wheelGeo = new THREE.CylinderGeometry(0.1, 0.1, 0.07, 14);
  const wheelMat = new THREE.MeshStandardMaterial({ color: "#0a0f1c" });
  [[-0.25, 0.25], [0.45, 0.25], [-0.25, -0.25], [0.45, -0.25]].forEach((p) => {
    const wh = new THREE.Mesh(wheelGeo, wheelMat);
    wh.rotation.x = Math.PI / 2;
    wh.position.set(p[0], 0.1, p[1]);
    van.add(wh);
  });
  van.position.set(-0.4, 0, 1.9);
  van.rotation.y = -0.32;
  g.add(van);
  return g;
}

function buildImmo(accent) {
  const g = new THREE.Group();
  g.add(buildPlinth(1.85, accent));
  const hall = box(2.9, 1.25, 2.0, "#2a3655");
  hall.position.y = 0.73;
  addEdges(hall);
  g.add(hall);
  const roof = box(3.15, 0.13, 2.25, accent, { emissive: accent, emissiveIntensity: 0.35, roughness: 0.45 });
  roof.position.y = 1.42;
  g.add(roof);
  const gate = new THREE.Mesh(new THREE.PlaneGeometry(1.35, 0.88), new THREE.MeshBasicMaterial({ color: "#0d1526" }));
  gate.position.set(0, 0.64, 1.01);
  g.add(gate);
  for (let i = 0; i < 3; i++) {
    const sky = new THREE.Mesh(new THREE.PlaneGeometry(0.3, 0.18), new THREE.MeshBasicMaterial({ color: "#ffd9a8" }));
    sky.position.set(-1.0 + i * 1.0, 1.14, 1.01);
    g.add(sky);
  }
  const crate1 = box(0.3, 0.3, 0.3, "#7a5f40");
  crate1.position.set(1.9, 0.25, 1.5);
  g.add(crate1);
  const crate2 = box(0.24, 0.24, 0.24, "#93744e");
  crate2.position.set(1.56, 0.22, 1.72);
  g.add(crate2);
  return g;
}

/* Zwei Gesellschafter auf gemeinsamem Deck */
function buildGesellschafter(accent) {
  const g = new THREE.Group();
  g.add(buildPlinth(1.25, accent));

  function figure(helmetColor, x) {
    const f = new THREE.Group();
    const bodyM = new THREE.Mesh(
      new THREE.CylinderGeometry(0.22, 0.26, 0.6, 16),
      new THREE.MeshStandardMaterial({ color: "#9a5a14", roughness: 0.8 })
    );
    bodyM.position.y = 0.4;
    bodyM.castShadow = true;
    f.add(bodyM);
    const head = new THREE.Mesh(
      new THREE.SphereGeometry(0.2, 18, 18),
      new THREE.MeshStandardMaterial({ color: "#f1c08a", roughness: 0.7 })
    );
    head.position.y = 0.9;
    head.castShadow = true;
    f.add(head);
    const helmet = new THREE.Mesh(
      new THREE.SphereGeometry(0.235, 18, 12, 0, Math.PI * 2, 0, Math.PI / 2),
      new THREE.MeshStandardMaterial({ color: helmetColor, roughness: 0.45 })
    );
    helmet.position.y = 0.92;
    f.add(helmet);
    f.position.x = x;
    return f;
  }
  g.add(figure("#ff8a3c", -0.42));
  g.add(figure("#4cc3ff", 0.42));
  return g;
}

function buildZukunft(accent) {
  const g = new THREE.Group();
  const ring = new THREE.Mesh(
    new THREE.RingGeometry(1.0, 1.08, 44),
    new THREE.MeshBasicMaterial({ color: accent, transparent: true, opacity: 0.42, side: THREE.DoubleSide })
  );
  ring.rotation.x = -Math.PI / 2;
  ring.position.y = 0.03;
  g.add(ring);
  const ghost = new THREE.Mesh(
    new THREE.BoxGeometry(1.55, 1.55, 1.55),
    new THREE.MeshStandardMaterial({
      color: accent, transparent: true, opacity: 0.12,
      emissive: accent, emissiveIntensity: 0.35, roughness: 0.3,
    })
  );
  ghost.position.y = 0.85;
  g.add(ghost);
  const edges = new THREE.LineSegments(
    new THREE.EdgesGeometry(ghost.geometry),
    new THREE.LineBasicMaterial({ color: "#67e8f9", transparent: true, opacity: 0.85 })
  );
  edges.position.copy(ghost.position);
  g.add(edges);
  return g;
}

/* ============================================================
   KOMPONENTE
   ============================================================ */

export default function ThermonextHoldingMap() {
  const mountRef = useRef(null);
  const apiRef = useRef(null);
  const onSelectRef = useRef(null);
  const [selected, setSelected] = useState(null);
  const [showHint, setShowHint] = useState(true);
  const [webglError, setWebglError] = useState(false);

  onSelectRef.current = setSelected;

  useEffect(() => {
    const mount = mountRef.current;
    if (!mount) return;

    let renderer;
    try {
      renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    } catch (e) {
      setWebglError(true);
      return;
    }
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
    renderer.setSize(mount.clientWidth, mount.clientHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.12;
    renderer.domElement.style.touchAction = "none";
    renderer.domElement.style.display = "block";
    mount.appendChild(renderer.domElement);

    const scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2("#05070f", 0.011);

    const camera = new THREE.PerspectiveCamera(50, mount.clientWidth / mount.clientHeight, 0.1, 240);
    const camTarget = new THREE.Vector3(0, 1.15, 0);
    const INIT = { r: 15.8, theta: 0.55, phi: 1.02 };
    const sph = { r: 27, theta: INIT.theta - 0.5, phi: INIT.phi };
    const sphT = { r: INIT.r, theta: INIT.theta, phi: INIT.phi };

    /* Licht */
    scene.add(new THREE.AmbientLight("#8ea2c9", 0.5));
    scene.add(new THREE.HemisphereLight("#b9c8ef", "#10182b", 0.55));
    const dir = new THREE.DirectionalLight("#ffe3bd", 1.15);
    dir.position.set(7, 12, 5);
    dir.castShadow = true;
    dir.shadow.mapSize.set(2048, 2048);
    dir.shadow.camera.left = -11;
    dir.shadow.camera.right = 11;
    dir.shadow.camera.top = 11;
    dir.shadow.camera.bottom = -11;
    dir.shadow.camera.far = 40;
    dir.shadow.bias = -0.0004;
    scene.add(dir);
    const rim = new THREE.DirectionalLight("#5b8cff", 0.35);
    rim.position.set(-8, 6, -7);
    scene.add(rim);

    /* Sterne */
    const starGeo = new THREE.BufferGeometry();
    const starPos = new Float32Array(300 * 3);
    for (let i = 0; i < 300; i++) {
      const rr = 40 + Math.random() * 32;
      const th = Math.random() * Math.PI * 2;
      const ph = Math.acos(2 * Math.random() - 1);
      starPos[i * 3] = rr * Math.sin(ph) * Math.cos(th);
      starPos[i * 3 + 1] = rr * Math.cos(ph) * 0.6 + 7;
      starPos[i * 3 + 2] = rr * Math.sin(ph) * Math.sin(th);
    }
    starGeo.setAttribute("position", new THREE.BufferAttribute(starPos, 3));
    const stars = new THREE.Points(
      starGeo,
      new THREE.PointsMaterial({ color: "#8aa0c8", size: 0.13, transparent: true, opacity: 0.55, fog: false })
    );
    scene.add(stars);

    /* Insel */
    const island = new THREE.Group();
    scene.add(island);

    const baseGeo = new THREE.ExtrudeGeometry(roundedRectShape(15.6, 11.2, 2.6), { depth: 1.1, bevelEnabled: false });
    baseGeo.rotateX(-Math.PI / 2);
    baseGeo.translate(0, -1.1, 0);
    const baseMesh = new THREE.Mesh(baseGeo, new THREE.MeshStandardMaterial({ color: "#0b1322", roughness: 1 }));
    baseMesh.receiveShadow = true;
    island.add(baseMesh);

    const grassGeo = new THREE.ExtrudeGeometry(roundedRectShape(14.9, 10.5, 2.4), { depth: GROUND_Y, bevelEnabled: false });
    grassGeo.rotateX(-Math.PI / 2);
    const grassMesh = new THREE.Mesh(grassGeo, new THREE.MeshStandardMaterial({ color: "#152e29", roughness: 0.95 }));
    grassMesh.receiveShadow = true;
    island.add(grassMesh);

    const glowTex = makeGlowTexture();
    const underGlow = new THREE.Mesh(
      new THREE.PlaneGeometry(24, 16),
      new THREE.MeshBasicMaterial({
        map: glowTex, color: "#4f6cff", transparent: true, opacity: 0.32,
        blending: THREE.AdditiveBlending, depthWrite: false,
      })
    );
    underGlow.rotation.x = -Math.PI / 2;
    underGlow.position.y = -1.7;
    island.add(underGlow);

    /* Bäume */
    const treePos = [[-6.4, 0.3], [-6.0, 3.6], [1.0, 4.4], [6.4, 1.0], [6.1, 3.6], [-1.6, -4.5], [2.2, -4.6]];
    treePos.forEach((p, i) => {
      const t = new THREE.Group();
      const trunk = new THREE.Mesh(
        new THREE.CylinderGeometry(0.06, 0.09, 0.3, 8),
        new THREE.MeshStandardMaterial({ color: "#5f4329", roughness: 1 })
      );
      trunk.position.y = 0.15;
      trunk.castShadow = true;
      t.add(trunk);
      const c1 = new THREE.Mesh(
        new THREE.ConeGeometry(0.38, 0.66, 8),
        new THREE.MeshStandardMaterial({ color: i % 2 ? "#123c28" : "#155036", flatShading: true })
      );
      c1.position.y = 0.58;
      c1.castShadow = true;
      t.add(c1);
      const c2 = new THREE.Mesh(
        new THREE.ConeGeometry(0.26, 0.46, 8),
        new THREE.MeshStandardMaterial({ color: "#186a45", flatShading: true })
      );
      c2.position.y = 0.94;
      c2.castShadow = true;
      t.add(c2);
      t.position.set(p[0], GROUND_Y, p[1]);
      island.add(t);
    });

    /* Schwebende Lichtpartikel */
    const dustGeo = new THREE.BufferGeometry();
    const DUST = 26;
    const dustPos = new Float32Array(DUST * 3);
    for (let i = 0; i < DUST; i++) {
      dustPos[i * 3] = (Math.random() - 0.5) * 17;
      dustPos[i * 3 + 1] = Math.random() * 6 + 0.5;
      dustPos[i * 3 + 2] = (Math.random() - 0.5) * 12;
    }
    dustGeo.setAttribute("position", new THREE.BufferAttribute(dustPos, 3));
    const dust = new THREE.Points(
      dustGeo,
      new THREE.PointsMaterial({
        color: "#7fa2ff", size: 0.07, transparent: true, opacity: 0.5,
        blending: THREE.AdditiveBlending, depthWrite: false,
      })
    );
    island.add(dust);

    /* Nodes */
    const selectables = [];
    const registry = new Map();
    const nodeGroups = new Map();
    const spinners = [];
    const builders = {
      gesellschafter: (c) => buildGesellschafter(c),
      holding: (c) => buildHolding(c),
      shk: (c) => buildSHK(c, spinners),
      immo: (c) => buildImmo(c),
      zukunft: (c) => buildZukunft(c),
    };

    NODE_DEFS.forEach((n) => {
      const g = builders[n.id](n.color);
      g.position.set(n.x, GROUND_Y, n.z);
      g.traverse((o) => {
        if (o.isMesh) {
          o.userData.pick = n.id;
          selectables.push(o);
        }
      });
      const lbl = makeLabel(n.label, n.sublabel, n.color, 30, n.sublabel ? 0.78 : 0.52);
      lbl.position.set(0, n.labelY, 0);
      g.add(lbl);
      island.add(g);
      nodeGroups.set(n.id, g);
      registry.set(n.id, { type: "node", def: n });
    });

    const plus = makeLabel("+", null, "#22d3ee", 40, 0.6);
    const zg = nodeGroups.get("zukunft");
    plus.position.set(0, 2.2, 0);
    zg.add(plus);

    /* Verbindungen */
    const flowAnims = [];
    const anchors = {};
    NODE_DEFS.forEach((n) => {
      anchors[n.id] = new THREE.Vector3(n.x, GROUND_Y + n.top, n.z);
    });

    function makeArc(a, b, lift, side) {
      const mid = a.clone().add(b).multiplyScalar(0.5);
      mid.y = Math.max(a.y, b.y) + lift;
      const dirv = b.clone().sub(a);
      const perp = new THREE.Vector3(-dirv.z, 0, dirv.x).normalize().multiplyScalar(side);
      mid.add(perp);
      return new THREE.QuadraticBezierCurve3(a.clone(), mid, b.clone());
    }

    CONN_DEFS.forEach((cd) => {
      const curve = makeArc(anchors[cd.from], anchors[cd.to], cd.lift, cd.side);
      const isFlow = cd.kind === "flow";

      const tube = new THREE.Mesh(
        new THREE.TubeGeometry(curve, 64, isFlow ? 0.045 : 0.024, 8, false),
        new THREE.MeshBasicMaterial({
          color: cd.color, transparent: true,
          opacity: isFlow ? 0.9 : 0.35, depthWrite: false,
        })
      );
      island.add(tube);

      if (isFlow) {
        const halo = new THREE.Mesh(
          new THREE.TubeGeometry(curve, 48, 0.13, 8, false),
          new THREE.MeshBasicMaterial({
            color: cd.color, transparent: true, opacity: 0.13,
            blending: THREE.AdditiveBlending, depthWrite: false,
          })
        );
        island.add(halo);
      }

      const hit = new THREE.Mesh(
        new THREE.TubeGeometry(curve, 24, 0.32, 6, false),
        new THREE.MeshBasicMaterial({ transparent: true, opacity: 0, depthWrite: false })
      );
      hit.userData.pick = cd.id;
      tube.userData.pick = cd.id;
      selectables.push(hit, tube);
      island.add(hit);

      const arrow = new THREE.Mesh(
        new THREE.ConeGeometry(isFlow ? 0.1 : 0.075, isFlow ? 0.26 : 0.19, 10),
        new THREE.MeshBasicMaterial({ color: cd.color })
      );
      const ap = curve.getPoint(0.88);
      const at = curve.getTangent(0.88).normalize();
      arrow.position.copy(ap);
      arrow.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), at);
      island.add(arrow);

      const mid = curve.getPoint(0.5).clone();
      const lbl = makeLabel(cd.label, null, cd.color, 21, 0.42);
      lbl.position.copy(mid).add(new THREE.Vector3(0, 0.3, 0));
      island.add(lbl);

      const glow = new THREE.Sprite(
        new THREE.SpriteMaterial({
          map: glowTex, color: cd.color, transparent: true, opacity: 0.9,
          blending: THREE.AdditiveBlending, depthWrite: false,
        })
      );
      glow.scale.set(1.5, 1.5, 1);
      glow.position.copy(mid);
      glow.visible = false;
      island.add(glow);

      if (isFlow) {
        const dots = [];
        for (let i = 0; i < 4; i++) {
          const d = new THREE.Sprite(
            new THREE.SpriteMaterial({
              map: glowTex, color: cd.color, transparent: true, opacity: 0.95,
              blending: THREE.AdditiveBlending, depthWrite: false,
            })
          );
          d.scale.set(0.4, 0.4, 1);
          island.add(d);
          dots.push({ m: d, off: i / 4 });
        }
        flowAnims.push({ curve, dots, speed: 0.1 });
      }
      registry.set(cd.id, { type: "conn", def: cd, glow });
    });

    /* Auswahl-Ring */
    const ring = new THREE.Mesh(
      new THREE.RingGeometry(1.35, 1.6, 56),
      new THREE.MeshBasicMaterial({ color: "#ffffff", transparent: true, opacity: 0.8, side: THREE.DoubleSide })
    );
    ring.rotation.x = -Math.PI / 2;
    ring.visible = false;
    island.add(ring);

    let currentSel = null;
    let pulseGroup = null;

    function clearSel() {
      if (pulseGroup) pulseGroup.scale.setScalar(1);
      pulseGroup = null;
      ring.visible = false;
      registry.forEach((e) => { if (e.glow) e.glow.visible = false; });
      currentSel = null;
      if (onSelectRef.current) onSelectRef.current(null);
    }

    function select(id) {
      const entry = registry.get(id);
      if (!entry || currentSel === id) return;
      if (pulseGroup) pulseGroup.scale.setScalar(1);
      pulseGroup = null;
      ring.visible = false;
      registry.forEach((e) => { if (e.glow) e.glow.visible = false; });
      currentSel = id;
      if (entry.type === "node") {
        const d = entry.def;
        ring.position.set(d.x, GROUND_Y + 0.045, d.z);
        ring.material.color.set(d.color);
        ring.visible = true;
        pulseGroup = nodeGroups.get(id);
        if (onSelectRef.current) onSelectRef.current(Object.assign({ color: d.color }, d.info));
      } else {
        entry.glow.visible = true;
        if (onSelectRef.current) onSelectRef.current(Object.assign({ color: entry.def.color }, entry.def.info));
      }
    }

    apiRef.current = {
      clear: clearSel,
      reset: () => { sphT.r = INIT.r; sphT.theta = INIT.theta; sphT.phi = INIT.phi; },
    };

    /* Steuerung */
    const pointers = new Map();
    let pinchDist = null;
    let interacted = false;
    let tapOk = false;
    let downT = 0, moved = 0;
    const raycaster = new THREE.Raycaster();
    const ndc = new THREE.Vector2();

    function onDown(e) {
      interacted = true;
      setShowHint(false);
      pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });
      renderer.domElement.setPointerCapture(e.pointerId);
      if (pointers.size === 1) {
        tapOk = true;
        moved = 0;
        downT = performance.now();
      } else if (pointers.size === 2) {
        tapOk = false;
        const pts = Array.from(pointers.values());
        pinchDist = Math.hypot(pts[0].x - pts[1].x, pts[0].y - pts[1].y);
      }
    }
    function onMove(e) {
      if (!pointers.has(e.pointerId)) return;
      const prev = pointers.get(e.pointerId);
      const dx = e.clientX - prev.x;
      const dy = e.clientY - prev.y;
      pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });
      if (pointers.size === 2 && pinchDist !== null) {
        const pts = Array.from(pointers.values());
        const nd = Math.hypot(pts[0].x - pts[1].x, pts[0].y - pts[1].y);
        if (nd > 0) {
          sphT.r = clamp(sphT.r * (pinchDist / nd), 6.5, 30);
          pinchDist = nd;
        }
      } else if (pointers.size === 1) {
        moved += Math.abs(dx) + Math.abs(dy);
        if (moved > 8) tapOk = false;
        sphT.theta -= dx * 0.0055;
        sphT.phi = clamp(sphT.phi - dy * 0.0045, 0.3, 1.4);
      }
    }
    function onUp(e) {
      pointers.delete(e.pointerId);
      if (pointers.size < 2) pinchDist = null;
      if (tapOk && performance.now() - downT < 550 && moved <= 8) {
        const rect = renderer.domElement.getBoundingClientRect();
        ndc.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
        ndc.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
        raycaster.setFromCamera(ndc, camera);
        const hits = raycaster.intersectObjects(selectables, false);
        if (hits.length > 0) select(hits[0].object.userData.pick);
        else clearSel();
      }
      tapOk = false;
    }
    function onWheel(e) {
      e.preventDefault();
      interacted = true;
      setShowHint(false);
      sphT.r = clamp(sphT.r * (1 + e.deltaY * 0.0012), 6.5, 30);
    }
    const el = renderer.domElement;
    el.addEventListener("pointerdown", onDown);
    el.addEventListener("pointermove", onMove);
    el.addEventListener("pointerup", onUp);
    el.addEventListener("pointercancel", onUp);
    el.addEventListener("wheel", onWheel, { passive: false });
    const noCtx = (e) => e.preventDefault();
    el.addEventListener("contextmenu", noCtx);

    function onResize() {
      const w = mount.clientWidth, h = mount.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    }
    window.addEventListener("resize", onResize);

    /* Loop */
    const clock = new THREE.Clock();
    let raf = 0;
    function animate() {
      raf = requestAnimationFrame(animate);
      const dt = Math.min(clock.getDelta(), 0.05);
      const t = clock.elapsedTime;

      /* Intro: Insel steigt, Kamera fährt heran */
      const intro = clamp(t / 1.6, 0, 1);
      const ease = 1 - Math.pow(1 - intro, 3);

      if (!interacted) sphT.theta += 0.001;
      sph.r += (sphT.r - sph.r) * 0.1;
      sph.theta += (sphT.theta - sph.theta) * 0.1;
      sph.phi += (sphT.phi - sph.phi) * 0.1;
      camera.position.set(
        camTarget.x + sph.r * Math.sin(sph.phi) * Math.sin(sph.theta),
        camTarget.y + sph.r * Math.cos(sph.phi),
        camTarget.z + sph.r * Math.sin(sph.phi) * Math.cos(sph.theta)
      );
      camera.lookAt(camTarget);

      island.position.y = (1 - ease) * -2.2 + Math.sin(t * 0.5) * 0.06 * ease;

      flowAnims.forEach((f) => {
        f.dots.forEach((d) => {
          const tt = (t * f.speed + d.off) % 1;
          d.m.position.copy(f.curve.getPoint(tt));
        });
      });

      const dp = dust.geometry.attributes.position;
      for (let i = 0; i < DUST; i++) {
        let y = dp.getY(i) + dt * 0.14;
        if (y > 6.5) y = 0.4;
        dp.setY(i, y);
      }
      dp.needsUpdate = true;

      spinners.forEach((s) => { s.rotation.z += dt * 5.5; });

      plus.position.y = 2.2 + Math.sin(t * 1.6) * 0.12;

      if (ring.visible) ring.material.opacity = 0.5 + 0.3 * Math.sin(t * 4.2);
      if (pulseGroup) pulseGroup.scale.setScalar(1 + 0.018 * Math.sin(t * 4.2));

      renderer.render(scene, camera);
    }
    animate();

    return () => {
      cancelAnimationFrame(raf);
      window.removeEventListener("resize", onResize);
      el.removeEventListener("pointerdown", onDown);
      el.removeEventListener("pointermove", onMove);
      el.removeEventListener("pointerup", onUp);
      el.removeEventListener("pointercancel", onUp);
      el.removeEventListener("wheel", onWheel);
      el.removeEventListener("contextmenu", noCtx);
      renderer.dispose();
      if (renderer.domElement.parentNode === mount) mount.removeChild(renderer.domElement);
      apiRef.current = null;
    };
  }, []);

  const LEGEND = [
    { c: "#34d399", t: "Dividende" },
    { c: "#60a5fa", t: "Miete" },
    { c: "#f59e0b", t: "Privat" },
    { c: "#8fa3c2", t: "Beteiligung" },
  ];

  /* Rechnungszeilen: Ergebniszeilen (→) hervorheben */
  function CalcBlock({ calc }) {
    const lines = calc.split("\n");
    return (
      <div
        className="rounded-xl font-mono"
        style={{ marginTop: 12, padding: "11px 13px", background: "rgba(2,6,14,0.65)", border: "1px solid #1c2942" }}
      >
        <div style={{ fontSize: 9, letterSpacing: "0.2em", color: "#5f7196", fontWeight: 700, marginBottom: 6 }}>
          BEISPIELRECHNUNG
        </div>
        {lines.map((l, i) => {
          const strong = l.startsWith("→");
          return (
            <div
              key={i}
              style={{
                fontSize: 11.5, lineHeight: 1.6,
                color: strong ? "#f5f8ff" : "#86efac",
                fontWeight: strong ? 700 : 400,
              }}
            >
              {l}
            </div>
          );
        })}
      </div>
    );
  }

  return (
    <div
      className="relative w-full overflow-hidden"
      style={{
        height: "100vh",
        background: "radial-gradient(1100px 750px at 50% 28%, #0c1530 0%, #05070f 58%, #020409 100%)",
        color: "#e9eef8",
        fontFamily: "system-ui, -apple-system, sans-serif",
      }}
    >
      <div ref={mountRef} className="absolute inset-0" />

      {/* Vignette */}
      <div
        className="absolute inset-0 pointer-events-none"
        style={{ background: "radial-gradient(ellipse at 50% 42%, rgba(0,0,0,0) 52%, rgba(0,0,0,0.55) 100%)", zIndex: 5 }}
      />

      {webglError && (
        <div className="absolute inset-0 flex items-center justify-center p-6 text-center text-sm" style={{ color: "#93a5c4" }}>
          3D konnte nicht geladen werden (WebGL nicht verfügbar).
        </div>
      )}

      {/* Kopf */}
      <div className="absolute top-0 left-0 right-0 flex items-start justify-between p-4 pointer-events-none" style={{ zIndex: 10 }}>
        <div>
          <div className="font-bold" style={{ fontSize: 17, letterSpacing: "0.26em" }}>
            THERMONEXT
          </div>
          <div style={{ fontSize: 10.5, color: "#8496b6", marginTop: 3, letterSpacing: "0.06em" }}>
            HOLDING-BLUEPRINT · GEBÄUDE &amp; LEITUNGEN ANTIPPEN
          </div>
          <div className="flex items-center" style={{ gap: 11, marginTop: 8 }}>
            {LEGEND.map((l) => (
              <span key={l.t} className="flex items-center" style={{ gap: 5, fontSize: 10, color: "#9db0d0" }}>
                <span style={{ width: 7, height: 7, borderRadius: 99, background: l.c, display: "inline-block", boxShadow: "0 0 6px " + l.c }} />
                {l.t}
              </span>
            ))}
          </div>
        </div>
        <button
          onClick={() => apiRef.current && apiRef.current.reset()}
          className="pointer-events-auto rounded-full font-semibold"
          style={{
            fontSize: 12, padding: "7px 13px",
            background: "rgba(10,16,32,0.75)", border: "1px solid #2a3a5c", color: "#c9d5ea",
            backdropFilter: "blur(8px)",
          }}
        >
          ↺ Ansicht
        </button>
      </div>

      {/* Hinweis */}
      {showHint && !selected && (
        <div className="absolute left-0 right-0 flex flex-col items-center" style={{ bottom: 24, gap: 7, zIndex: 10 }}>
          <div
            className="rounded-full"
            style={{
              fontSize: 11.5, padding: "9px 15px",
              background: "rgba(8,13,26,0.85)", border: "1px solid #253656", color: "#c9d5ea",
              backdropFilter: "blur(8px)",
            }}
          >
            Ziehen = drehen · 2 Finger / Scroll = Zoom · Antippen = Steuer-Info
          </div>
          <div style={{ fontSize: 9.5, color: "#57678a" }}>
            Veranschaulichung – keine Steuerberatung
          </div>
        </div>
      )}

      {/* Infofenster */}
      {selected && (
        <div className="absolute inset-x-0 bottom-0" style={{ zIndex: 20 }}>
          <div
            className="mx-auto w-full rounded-t-2xl shadow-2xl"
            style={{
              maxWidth: 470,
              background: "rgba(6,10,20,0.92)",
              border: "1px solid #223250",
              borderBottom: "none",
              backdropFilter: "blur(12px)",
            }}
          >
            <div style={{ height: 3, background: selected.color, borderTopLeftRadius: 16, borderTopRightRadius: 16, boxShadow: "0 0 14px " + selected.color }} />
            <div className="overflow-y-auto" style={{ maxHeight: "54vh", padding: "13px 17px 15px" }}>
              <div className="flex items-start justify-between" style={{ gap: 10 }}>
                <div>
                  <div style={{ fontSize: 9.5, letterSpacing: "0.24em", color: selected.color, fontWeight: 700 }}>
                    {selected.kind}
                  </div>
                  <div className="font-bold" style={{ fontSize: 16.5, marginTop: 3, letterSpacing: "0.01em" }}>{selected.title}</div>
                  {selected.sub && <div style={{ fontSize: 11.5, color: "#8496b6", marginTop: 3 }}>{selected.sub}</div>}
                </div>
                <button
                  onClick={() => apiRef.current && apiRef.current.clear()}
                  className="rounded-full font-bold flex items-center justify-center"
                  style={{ width: 28, height: 28, minWidth: 28, background: "#141f38", color: "#9db0d0", fontSize: 13, border: "1px solid #26375a" }}
                  aria-label="Schließen"
                >
                  ✕
                </button>
              </div>

              <div style={{ height: 1, background: "#1a2742", margin: "11px 0" }} />

              <div>
                {selected.paras.map((p, i) => (
                  <p key={i} style={{ fontSize: 13, lineHeight: 1.5, color: "#d7e0f0", marginTop: i === 0 ? 0 : 8 }}>
                    {p}
                  </p>
                ))}
              </div>

              {selected.law && selected.law.length > 0 && (
                <div className="flex flex-wrap" style={{ gap: 6, marginTop: 12 }}>
                  {selected.law.map((l) => (
                    <span
                      key={l}
                      className="rounded-full"
                      style={{ fontSize: 10, padding: "3.5px 10px", background: "#10192e", border: "1px solid #2a3a5c", color: "#aebdd8" }}
                    >
                      {l}
                    </span>
                  ))}
                </div>
              )}

              {selected.calc && <CalcBlock calc={selected.calc} />}

              {selected.warn && (
                <div style={{ fontSize: 11, color: "#f5c169", marginTop: 11, lineHeight: 1.5 }}>
                  ⚠ {selected.warn}
                </div>
              )}

              <div style={{ fontSize: 9.5, color: "#57678a", marginTop: 12 }}>
                Vereinfachte Modellrechnung (Hebesatz 400 %) – keine Steuerberatung. Umsetzung nur mit Steuerberater.
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
