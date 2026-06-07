# CONTEXT.md — Vektkontroll

Kontekst og beslutninger for videre utvikling. Les denne før du endrer koden.

## Hva appen er

En **privat, mobil-først vektlogg i én enkelt fil** (`index.html`). Alt — HTML, CSS,
JavaScript, matvaredatabase, ikoner, manifest — ligger inni den ene fila. Ingen
byggesteg, ingen avhengigheter, ingen backend. Data lagres kun lokalt i nettleseren
(`localStorage`), så ingenting forlater enheten.

Appen implementerer en bestemt slankemetode (se under), ikke kun en vektlogg.

Tre faner: **Vekt** (logg + plan + graf), **Mat** (matvekt-kalkulator per måltid),
**Metode** (forklaring, visualisering, FAQ, pausemodus).

## Metoden (closed-loop vektstyring) — KJERNEN

I stedet for å telle kalorier styres vekten direkte mot en synkende mållinje.

- **Startvekt** = veies om **kvelden** etter siste måltid (viktig: en kveldsvekt, høyere
  enn morgenvekten). Sammen med **målvekt** og **periode** (antall dager).
- **Daglig reduksjon:** `tap_per_dag = (startvekt − målvekt) / periode`.
- **Mållinje for en dag:** `mållinje(dag) = startvekt − tap_per_dag × dag`, der
  `dag = 1` er **startdato** (dag 1 = første reduksjonsdag, så dagens andel av
  reduksjonen er allerede trukket fra). Målvekt nås på dag = `periode`.
- **Matbudsjett i dag (gram):** `(mållinje_i_dag − morgenvekt) × 1000`. Dette er hvor
  mye (i vekt) man kan spise og likevel treffe målet til kvelden.
- Budsjettet **fordeles på måltidene** etter prosentene (frokost/lunsj/middag/kvelds,
  standard 30/10/50/10).
- **Kvelds-justering:** veiing etter middag (= før kvelds) gir nøyaktig kveldsrasjon:
  `mållinje − vekt_før_kvelds`.
- Budsjettet blir **0** hvis morgenvekt ≥ mållinje (man er ikke under linja). Det er
  korrekt per metoden, ikke en feil — appen forklarer dette i plankortet.
- **Pausemodus:** setter `tap_per_dag = 0` (flat mållinje på startvekt) → større daglige
  måltider mens vekten holdes.

Verifisert mot metodens eget eksempel: 80→76 kg på 40 dager = 100 g/dag; morgen 78,8 mot
kveldsmål 79,9 → 1,1 kg mat den dagen.

Kilde til metode/FAQ: opphavspersonens beskrivelse (gjengitt i Metode-fanen).

## Viktige beslutninger (og hvorfor)

1. **Én selvstendig fil.** Brukeren laster opp `index.html` til et GitHub-repo og hoster
   via GitHub Pages. Derfor er manifest + ikoner + matdata **innebygd** (manifest som
   `data:`-URI, ikoner som base64), og service worker er **fjernet** i den hostede
   versjonen (kan ikke kjøre fra `file://`/inline pålitelig).
2. **Personvern:** den offentlige hostede versjonen inneholder **ingen persondata**.
   Brukerens historikk importeres lokalt fra en backup-fil. (Repoet `Clauderepository`
   som ble brukt under utviklingen er privat; appen hostes fra et eget repo `vektkontroll`.)
3. **Startvekt = kveldsvekt.** Avgjørende for at metoden gir et positivt budsjett dag 1.
   Ble besluttet til 114,2 kg for brukeren ut fra historikk (morgen 113,6 + typisk
   morgen→kveld-differanse +0,6 kg).
4. **Dag 1 = første reduksjonsdag** (`mållinje = startvekt − tap×dag`, ikke `tap×(dag−1)`),
   og `tap = (s−m)/periode` (ikke `/(periode−1)`), slik at dagens andel av reduksjonen
   regnes med fra startdato.
5. **Matvekt i gram, ikke kalorier.** Brukeren ville ha gram fordelt etter prosenter, slik
   originalmetoden fungerer.

## Arkitektur og datamodell

Alt i `<script>` nederst i `index.html`. Sentralt `state`-objekt lagres i `localStorage`
under nøkkelen `vektkontroll_v1`:

```js
state = {
  entries: [ { date:"YYYY-MM-DD", w:Number, note:"", wm:{frokost,lunsj,middag,kvelds} } ],
  goal: Number|null,            // målvekt-linje i grafen
  height: Number|null,          // for BMI
  plan: { startvekt, maalvekt, startdato, periode, frokost, lunsj, middag, kvelds, pause },
  meal: { date:"YYYY-MM-DD", items:[ {name, g, qty, meal} ] },  // dagens matlogg
  customFoods: [ {name, g} ]    // egne matvarer lagt til i valglisten
}
```

- `entries`: én per dato. `w` = morgenvekt. `wm` = valgfrie veiinger gjennom dagen
  (`{frokost,lunsj,middag,kvelds}`, etter hvert måltid). Sorteres på dato. Både morgenvekt
  og de fire veiingene logges i **«Ny registrering»**-seksjonen og lagres samlet via
  «Lagre registrering»; skjemaet fylles fra valgt dato av `loadEntryForm(date)`.
- `meal`: dagens måltidslogg; nullstilles automatisk ved nytt døgn (`meal.date !== i dag`).
- `MATVARER` (innebygd, fra Excel-arket «Matvekt») + `customFoods` = valglisten.
  `KCAL100` = referansetabell «gram per 100 kcal».

**Nøkkelfunksjoner:**
- `tapPerDag(p)`, `maallinje(p, isoDato)`, `planEnd(p)` — planmatematikken.
- `render()` → `renderStats / renderPlan / renderChart / renderList / renderPauseBtn`.
- `renderPlan()` — plankort: mållinje, morgenvekt, måltidsrasjoner (gram), «spist i dag»,
  prognose, status (inkl. pausemodus). Kvelds-rasjonen justeres av `wm.middag`
  (veiing etter middag = før kvelds).
- `renderChart()` — **dato-basert** SVG: vektkurve + mållinje (synkende) + flat målvekt.
  Tidsfilter (Alt/1år/90d/30d/7d) + egendefinert startdato (`chartFrom`).
- `renderDay()` — Mat-fanen: dagstotal + per-måltid-grupper, budsjettsammenligning.
- `addEntry / editEntry` — vektregistreringer (morgenvekt + `wm`-veiinger). Hjelpere:
  `loadEntryForm(date)`, `readWmBoxes()`, `fillWmBoxes(wm)`.
- `addFoodToMeal` — matlogg. Valglisten har «✏️ Egen matvare (fritekst)»: navn + gram +
  antall, med avkryssing for å lagre i `customFoods`. `populateFoodSelect / toggleCustom`.
- `renderPauseBtn` + pause-knapp i Metode-fanen (`state.plan.pause`).
- `showTab(name)` — bytter fane (`vekt|mat|metode`).
- Backup: `exportData / importData` (hele `state` som JSON).

**Konvensjoner:** vanilla JS, ingen rammeverk. Komma som desimalskilletegn håndteres med
`.replace(",",".")`. All UI-tekst på norsk. Hold appen som **én fil**.

## Bygg, host og data

- **Kjøre lokalt:** åpne `index.html` i nettleser, eller `python3 -m http.server`.
- **Host:** last opp `index.html` til repoet `vektkontroll` → Settings → Pages →
  «Deploy from a branch» → main / root. URL: `https://codelamoro.github.io/vektkontroll/`.
  Pages krever offentlig repo (gratis). Den hostede koden er persondata-fri.
- **localStorage** er per origin (samme URL) → data består når man laster opp ny
  `index.html` til samme adresse. «Legg til på Hjem-skjerm» gir app-ikon (krever https).
- **Datakilder (ikke nødvendig for å kjøre):**
  - `WeightCtrl.xlsx` — opprinnelig regneark. Ark «Ny periode 2026» = closed-loop-modellen;
    ark «Matvekt» = matvaredatabasen (innebygd i appen).
  - `vektkontrollbackup.json` — brukerens data (310 veiinger 2017–2026 + plan), importeres
    i appen. Garmin-eksport ble flettet inn; for dager med flere veiinger ble den tidligste
    (nærmest morgen) valgt.

## Mulige neste steg (nevnt, ikke gjort)

- Arkivere gårsdagens dagstotal (mathistorikk).
- Live-oppdatering av «spist i dag» på Vekt-fanen uten å bytte fane.
- Automatisk Garmin-henting (krever en liten backend pga. pålogging/CORS — bryter
  én-fils-prinsippet).

## Når du endrer noe

- **Test planmatematikken** mot metodens eksempel (80→76/40d → 1100 g dag 1) hvis du rører
  `tapPerDag`/`maallinje`.
- Behold **én fil** og **ingen persondata i koden** i den hostede versjonen.
- Etter endring: bytt ut `index.html` i `vektkontroll`-repoet (Pages bygger på nytt).
