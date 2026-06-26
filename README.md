# PCB Studio

Een lichte, **single-file** PCB-designtool (schematic + PCB-layout + BOM) die touch-first
op tablet/mobiel werkt en in de browser draait — geen build step, geen server, geen account.
Alles wordt lokaal opgeslagen (localStorage). Klaar om later via Capacitor naar Android te
packagen.

> **Status: MVP.** Bruikbaar om een echt, simpel-tot-middelmatig bordje te ontwerpen
> (microcontroller-breakouts, sensorboards, adapter-PCB's). Géén vervanging voor KiCad/EasyEDA —
> zie [Gaps t.o.v. een echte tool](#gaps-tov-een-echte-tool-kicadeasyeda).

## Gebruik

Open `index.html` in een browser (desktop of tablet). Dat is alles.

```
# lokaal serveren (optioneel, niet nodig — file:// werkt ook):
python3 -m http.server 8000   # → http://localhost:8000/index.html
```

Het werk wordt automatisch lokaal bewaard. Gebruik **Exporteer project (JSON)** om een
back-up te maken of tussen apparaten te wisselen.

### Bediening
- **Tabs** bovenaan: Schema / PCB / BOM.
- **Plaatsen uit onderdelen-DB:** in het Schema-tab → *Uit onderdelen-DB → Kies partnummer & plaats*.
  Tik een LCSC-partnummer in (of kies een bestaand), tik dan op het canvas. Het juiste symbool,
  de waarde én de LCSC-koppeling worden automatisch ingevuld → komt zo ook meteen in de BOM.
- **Plaatsen (kaal symbool):** kies links een component/footprint, tik op het canvas.
- **Pan:** sleep op lege ruimte (1 vinger) of sleep met de muis. **Zoom:** pinch (touch) of muiswiel.
- **Draad/Trace:** kies de tool, tik pin→pin. Dubbeltik, **Enter**, of de **✓ Klaar**-knop sluit af.
- **Sneltoetsen:** `R` draaien · `Del`/`Backspace` verwijderen · `F` passend zoomen ·
  `Ctrl/Cmd+Z` undo · `Ctrl/Cmd+Shift+Z` redo · `Esc` annuleren.
- Op smalle schermen: **☰** opent het gereedschap, **ℹ** de eigenschappen.

---

## Wat werkt per fase

### Fase 1 — Schematic editor
- Componenten: weerstand, condensator, **gepolariseerde condensator / elco (+/−)**, LED, diode,
  IC (instelbaar aantal pins), connector/header (N pins), generiek 2-pin en generiek N-pin.
- **Polariteit:** bij polaire onderdelen (elco, LED, diode) tonen we duidelijke **+ (rood)** en
  **− (blauw)** markeringen bij de pinnen — in het schema én op de PCB (silkscreen).
- Pins verbinden met orthogonale draden (tik-tik), met automatische elleboog en junction-dots.
- **Net labels** geven verbindingen een naam; gelijke namen worden hetzelfde net (ook zonder
  fysieke draad).
- Referentie (R1, C1, U1…) automatisch toegekend + waarde-veld (bv. "10k", "100nF").
- Pan/zoom met touch én muis, undo/redo, opslaan/laden als JSON (localStorage + bestand-export/import).
- **Net-engine:** berekent netten uit pins + draden + labels (union-find, inclusief
  T-junctions waar een pin midden op een draad ligt).

### Fase 2 — PCB layout editor
- **Footprints:** automatisch genereren uit de schematic (knop *Genereer footprints uit schema*)
  óf handmatig kiezen uit een bibliotheek: 0805, 0603, THT-weerstand, SOT-23-3, 3 mm LED,
  SOIC-8/14/16, DIP-8/14/16, 2.54 mm headers 1×2…1×8. De set is eenvoudig uitbreidbaar in
  `makeFootprint()`.
- Plaatsen en draaien op een instelbaar mm-grid (0.5–2.54 mm).
- Traces tekenen op **2 lagen** (top/bottom copper), instelbare trace-breedte.
- **Ratsnest:** stippellijnen tonen welke pads volgens de schematic verbonden moeten worden maar
  nog geen trace hebben (MST per net, houdt rekening met bestaande traces).
- **Basic DRC:** waarschuwt bij pads/traces dichter dan de instelbare clearance (rode markers).
- **Board outline:** rechthoek tekenen.
- **Fabricage-export (JLCPCB-klaar):** echte **Gerber (RS-274X)** — top/bottom koper,
  top/bottom soldermask, top silkscreen (met courtyards + polariteit), board outline (edge cuts) —
  plus een **Excellon-boorfile**, een **CPL / pick&place** (JLCPCB-formaat) en de **assemblage-BOM**,
  allemaal verpakt in één **ZIP** (zonder externe libraries; eigen ZIP-writer met CRC32).
  Daarnaast top/bottom-copper als **SVG** + volledige **JSON** ter referentie.
  > Best-effort uit een MVP-tool: **controleer altijd in een gratis Gerber-viewer** (JLCPCB online
  > viewer / KiCad GerbView) vóór je bestelt — vooral pad-vormen, drill-maten en de CPL-rotaties.

### Fase 3 — LCSC + BOM
**Gekozen aanpak: a + c (hybride).** LCSC heeft geen publieke API en hun site is JS-rendered,
dus live opzoeken vanuit een WebView is onbetrouwbaar. Daarom:
- Een **lokale, handmatig samengestelde dataset** van veelgebruikte onderdelen (passieven in
  standaardwaarden, een paar populaire IC's, connectors) met LCSC-partnummer, prijsindicatie en
  footprint-match.
- Een **formulier om zelf onderdelen toe te voegen** (naam, partnummer, prijs, footprint), bewaard
  in je eigen lokale database.

Partnummers en prijzen zijn **schattingen** (laatst bijgewerkt jun 2026) en in de UI overal als
zodanig gelabeld — **controleer altijd op lcsc.com**. (Optie b, live fetch, is bewust niet
ingebouwd: CORS/JS-rendering blokkeert dit vrijwel zeker en het zou een schijnzekerheid geven.)

Verder:
- **Partnummer-gedreven plaatsen:** tik een LCSC-partnummer in (of kies uit de DB) en plaats het
  onderdeel direct in het schema. Categorie (R/C/D/U/J) bepaalt het symbool, de footprint bepaalt
  het aantal pins (bv. SOIC-8 → 8-pins IC); waarde + LCSC-koppeling worden meteen ingevuld.
- Onderdeel koppelen aan een bestaand schematic-symbool (knop *Koppel onderdeel*).
- **BOM** automatisch gegenereerd uit de schematic: groepeert op waarde/footprint/partnummer,
  met aantal, designators, partnummer en geschatte totaalprijs.
- **CSV-export:** JLCPCB SMT-formaat (`Comment, Designator, Footprint, LCSC Part #`) + een
  gedetailleerde CSV met prijzen.

---

## Hoe het is getest
De app is in headless Chromium (via het Chrome DevTools Protocol) end-to-end gedreven: 27 checks
op netberekening, footprint-geometrie (pad-aantallen per type), DRC, pointer-gestuurd plaatsen/
slepen/draden, LCSC→BOM-koppeling, save/load round-trip en export — plus visuele controle van
schema, PCB, BOM en modals. Geen console-/runtime-fouten.

---

## Gaps t.o.v. een echte tool (KiCad/EasyEDA)
Dit is een MVP. De belangrijkste beperkingen, zodat je weet wat je kunt verwachten:

1. **Gerber/drill/CPL zijn best-effort, niet gevalideerd door een fab.** De export is geldig
   RS-274X/Excellon en opent in gerber-viewers, maar er zit geen volledige conformiteitstest of
   CAM-controle op. Verifieer altijd in een viewer; CPL-rotaties kloppen mogelijk niet voor elk
   onderdeel (een bekend lastig punt, zelfs in pro-tools). Geen X2-attributen, geen paste-laag.
2. **Geen design-symbool/footprint-bibliotheken** zoals KiCad. Symbolen en footprints zijn een
   kleine, hardcoded set; geen import van standaardlibs of 3D-modellen.
3. **DRC is basaal.** Alleen clearance pad-pad en trace-trace (per laag). Geen controle op
   trace-naar-pad over alle gevallen, annular ring, via's, korte-/open-netten, courtyard-overlap,
   of of álle ratsnest-verbindingen daadwerkelijk gerouteerd zijn.
4. **Geen via's / >2 lagen.** Alleen top en bottom copper; geen doorverbindingen tussen lagen,
   geen power planes/zones/copper pours, geen thermal reliefs.
5. **Geen auto-router en geen push-&-shove.** Traces zijn handmatig en orthogonaal/recht; geen
   45°-only mode, geen lengte-matching, geen differentiële paren.
6. **Board outline = alleen rechthoek.** Geen polygon/uitsparingen/mounting holes als feature.
7. **Schematic capture is licht.** Geen hiërarchische sheets, bussen, power-symbolen, ERC,
   of multi-unit componenten; netten worden puur geometrisch afgeleid.
8. **Geen echte voorraad/prijs-API.** LCSC-data is een handmatige momentopname; geen live
   beschikbaarheid of staffelprijzen.
9. **Schaal.** Net-engine en DRC zijn O(n²)-achtig; prima voor kleine/middelgrote bordjes,
   niet voor honderden componenten.
10. **Mobiel.** Werkt op telefoon maar is krap; **tablet is het realistische doel** voor
    PCB-werk.

## Architectuur (1 bestand)
Alles zit in `index.html` (HTML + CSS + vanilla JS, geen dependencies). De JS is opgedeeld in
genummerde secties: constants/state · utilities · symbol-library · footprint-library ·
net-engine · viewport/canvas · rendering (schema/PCB) · input/tools · properties · LCSC/BOM ·
export/UI. Een test-hook `window.PCB` stelt de kernfuncties bloot voor geautomatiseerde tests.
