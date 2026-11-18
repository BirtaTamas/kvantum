# BB84-szimulátor – Megvalósítási terv

Tárgy: Kvantuminformatikai alkalmazások, 2026 ősz – 4. feladat
Forrás: `Hazi_leiras_Kvantuminformatikaialkalmazasok_2026osz_20261015.pdf`

> ⚠️ **Határidő:** a rendes határidő 2026. november 15. volt, ez már lejárt. Késedelmes beadás a pótlási hét 4. napjáig lehetséges, különeljárási díj ellenében, a Moodle külön késedelmes felületén. **A pontos dátumot a Neptunban / Moodle-ben ellenőrizni kell.**

---

## 1. Követelmények (a PDF alapján)

### 1.1 Beadandók – mindhármat be kell adni
| # | Beadandó | Hol lesz nálunk |
|---|----------|-----------------|
| B1 | A szimulátor kódja | `leadando/forraskod/` |
| B2 | Futtatható szimulátor, standalone, böngészővel rendelkező Windows-os gépen, speciális fejlesztőkörnyezet telepítése nélkül | `leadando/futtathato/bb84_szimulator.html` |
| B3 | Legalább 3 oldalas dokumentáció | `leadando/dokumentacio/BB84_dokumentacio.pdf` |

Beadás módja: Moodle. Egyénileg vagy kettesben is beadható.

### 1.2 Értékelés
Három szempontot pontoznak 1–5-ig. **A végső jegy a három jegy középső értéke (medián), és minden szempontra legalább 2-es kell.** Vagyis mindhárom szempontra 5-öst célzunk; egyik sem lehet gyenge pont.

**Sz1 – Dokumentáció minősége**
- 5: megfelelő hosszúságú (≥ 3 oldal), és tartalmazza a használathoz szükséges információkat, a felhasznált könyvtárakat stb.

**Sz2 – Az elkészült program** (hány szempontot teljesít az 5-ből)
- P1: Helyes a működés?
- P2: A bit- és bázisválasztásokat lehetséges megadni?
- P3: A támadó megfelelően jelenik meg a programban?
- P4: A támadó hatása érzékelhető?
- P5: Léptethető a program a protokoll egyes pontjait követve? (Nem elég, hogy a végén kiad egy megegyező bitsorozatot.)

**Sz3 – Felhasználói élmény**
- 5: a felület kreatív, frappáns, látványos, és segíti a program használatát és bemutatását (pl. egy hallgatótársaknak szóló bemutatóban).
- 4: igényes és könnyen értelmezhető.

---

## 2. Technológiai döntés – „programozási környezet”

A feladat szövege szerint a programozási környezet *tetszőlegesen választható*. Ez a nyelvet és az eszközkészletet jelenti. A döntő kényszer a B2: a programnak **egy sima Windows-os gépen, böngészővel, telepítés nélkül** kell futnia.

| Lehetőség | Telepítés nélkül fut Windowson? | Megjegyzés |
|-----------|-------------------|-----------|
| **HTML + CSS + JavaScript (vanilla)** | ✅ dupla kattintás a `.html`-re | Nincs build, nincs függőség, és a böngésző maga a futtatókörnyezet |
| Python (Tkinter/PyQt) | ❌ Python vagy PyInstaller-exe kell | Az exe-t Macen nem lehet Windowsra buildelni, a vírusirtó is gyanakodhat |
| Java / C# | ❌ JRE / .NET kell | Telepítési kockázat |
| React/Vue + bundler | ✅ a build után | Fölösleges komplexitás, és build-lánc kell hozzá |

**Döntés: HTML5 + CSS3 + vanilla JavaScript (ES2020), külső könyvtár és CDN nélkül.**
- **Nincs semmilyen külső függőség, még CDN sem.** Így offline, egyetemi hálózat nélkül is fut, és a dokumentáció „felhasznált könyvtárak” része egyszerű: csak a böngésző beépített API-jait használjuk (DOM, Canvas/SVG, `crypto.getRandomValues`).
- **Célböngészők:** Chrome, Edge és Firefox aktuális verziói (a Windows-os gépeken alapból Edge van).
- A kódot olvashatóan, modulokra bontva írjuk (B1), a futtatható változat pedig egyetlen önálló HTML fájl, amelybe a CSS és a JS be van ágyazva (B2). Így a fájl `file://`-ból is biztosan működik, nem ütközik a `file://` alatti ES-modul tiltásba.

---

## 3. Mappastruktúra

```
kvantum/
├── Hazi_leiras_...pdf              # feladatleírás
├── TERV.md                         # ez a terv
├── leadando/                       # CSAK ami a Moodle-re megy
│   ├── forraskod/                  # B1 – olvasható forrás
│   │   ├── index.html
│   │   ├── style.css
│   │   ├── bb84.js                 # tiszta protokoll-logika (UI nélkül, tesztelhető)
│   │   └── ui.js                   # felület, léptetés, megjelenítés
│   ├── futtathato/
│   │   └── bb84_szimulator.html    # B2 – egyetlen fájl, minden beágyazva
│   └── dokumentacio/
│       └── BB84_dokumentacio.pdf   # B3
└── munka/                          # minden más (nem adjuk le)
    ├── eszkozok/build.py           # forraskod → futtathato (inline-olás)
    ├── tesztek/                    # logikai tesztek (headless Chrome-ban futnak)
    └── dokumentacio_forras/        # a dokumentáció forrása (md/docx) + képernyőképek
```
A leadáskor a `leadando/` mappa tartalmát zipeljük: `BB84_<Neptunkód>.zip`.

---

## 4. Funkcionális specifikáció

### 4.1 Protokoll-modell (`bb84.js`, UI-tól független)
- **Bázisok:** `+` (rektilineáris: 0 → |0⟩ ↕, 1 → |1⟩ ↔) és `×` (diagonális: 0 → |+⟩ ⤢, 1 → |−⟩ ⤡).
- **Mérés szabálya:** ha a bázis egyezik, a mért bit megegyezik a küldött bittel. Ha nem egyezik, a mért bit 50–50% eséllyel 0 vagy 1, és az állapot a mérési bázis szerinti állapotba omlik össze.
- **Eve (intercept–resend):** minden fotonnál (vagy beállítható arányban, 0–100%) saját véletlen bázisban mér, és a mért eredményt a saját bázisában küldi tovább Bobnak.
- **Csatornazaj:** opcionális, 0–20% bitátfordítási valószínűség. Ezzel az is megmutatható, hogy a zajt és a támadást a QBER alapján kell megkülönböztetni.
- **Véletlen:** `crypto.getRandomValues`. Ezen felül **seed-elhető PRNG** (pl. mulberry32), hogy egy bemutató megismételhető legyen, és a tesztek determinisztikusak legyenek.

### 4.2 A protokoll lépései – a léptetés egységei (P5)
1. **Alice bitjei:** véletlenszerűen generálja vagy kézzel adja meg a biteket.
2. **Alice bázisai:** véletlenszerűen generálja vagy kézzel adja meg őket.
3. **Kódolás és küldés:** a fotonok polarizációja megjelenik, és animálva mennek át a kvantumcsatornán.
4. **(Eve) lehallgatás:** ha be van kapcsolva, Eve bázisai és mérési eredményei, valamint az újraküldött fotonok láthatók.
5. **Bob bázisai:** véletlenszerűen generálja vagy kézzel adja meg őket.
6. **Bob mérése:** a mért bitek; a véletlen kimenetű mérések kiemelve.
7. **Bázisegyeztetés (nyilvános csatorna):** a bázisokat összehasonlítják, a bitek ebben a lépésben nem nyilvánosak.
8. **Szitálás (sifting):** csak az egyező bázisú pozíciók maradnak meg, ez a nyers (sifted) kulcs.
9. **Hibabecslés:** a sifted kulcs egy választható hányadát (pl. 50%) nyilvánosan összehasonlítják, ebből számolják a QBER-t.
10. **Döntés:** ha a QBER egy küszöb fölött van (alapból 11%), a kulcsot eldobják, mert lehallgatást gyanítanak. Különben a maradék bitek adják a végső kulcsot.

Vezérlés: **⏮ Elejére / ◀ Vissza / ▶ Következő / ⏭ Végig / ⏯ Automatikus lejátszás (sebességgel)**. Minden lépésnél egy magyarázó panel írja le, mi történik és miért.

### 4.3 Bemenetek (P2)
- Fotonok száma: 8–64; a táblázatos nézetben ajánlott 16–24.
- Alice bitjei és bázisai, valamint Bob bázisai kétféleképpen adhatók meg: **szövegmezőben** (pl. `0110…`, `++×+…`, a bázisokra `x` is elfogadott), vagy **cellánként kattintva** a táblázatban. Mindkettőhöz van „🎲 Véletlen” gomb.
- Eve bázisai is megadhatók kézzel vagy véletlenszerűen.
- A bemenet validálva van: hossz és megengedett karakterek. Hibás bemenetnél magyar nyelvű hibaüzenet jelenik meg.

### 4.4 Támadó megjelenítése és hatása (P3, P4)
- Eve külön sávként jelenik meg a csatorna közepén, saját színnel. Bekapcsolt állapotban a fotonok nála „megállnak”, és látható a mérése.
- Az Eve által elrontott pozíciók (ahol Alice és Bob bázisa egyezik, a bitjük mégis eltér) **piros kiemelést** kapnak.
- **Statisztika panel:** sifted kulcs hossza, a hibás bitek száma, QBER (%), Eve információja a kulcsról (hány bitet ismer helyesen), és az elméleti várható érték (25% teljes intercept-resend esetén).
- **„Összehasonlítás” mód:** ugyanaz a seed Eve-vel és Eve nélkül, egymás mellett. Ez a „frappáns” bemutató eleme.
- **Opcionális QBER-grafikon:** a QBER Eve lehallgatási arányának függvényében (0–100%), sok futásból, az elméleti `p/4` egyenessel összevetve.

### 4.5 Felület (Sz3)
- Teljesen **magyar nyelvű**.
- Felépítés: fent a szereplők (Alice → kvantumcsatorna (Eve) → Bob) animált fotonokkal, középen a protokolltáblázat (soronként egy szereplő, oszloponként egy foton), lent a lépésmagyarázat és a statisztika.
- Jelölések: polarizációs nyilak (↕ ↔ ⤢ ⤡), színkódok (egyező bázis = zöld, eldobott = szürke, hiba = piros).
- Világos és sötét téma; reszponzív kialakítás, de az elsődleges cél a laptop és a projektoros bemutató.
- „Súgó / Mi az a BB84?” felugró ablak röviden az elméletről.

---

## 5. Megvalósítási pipeline

| Fázis | Feladat | Kimenet | Kész, ha… |
|------|---------|---------|-----------|
| F0 | Struktúra és terv | mappák, `TERV.md` | ✅ |
| F1 | Protokoll-logika (`bb84.js`): PRNG, kódolás, mérés, Eve, zaj, sifting, QBER, döntés, lépésállapotok | `forraskod/bb84.js` | Az F2 tesztjei zöldek |
| F2 | Logikai tesztek | `munka/tesztek/teszt.html` + futtató | Minden T-teszt átmegy (lásd 6.1) |
| F3 | UI váz: elrendezés, bemenetek, táblázat, léptetés | `index.html`, `style.css`, `ui.js` | Mind a 10 lépés végigléptethető, előre és vissza is |
| F4 | Vizualizáció: fotonanimáció, Eve-sáv, kiemelések, statisztika, összehasonlító mód, QBER-grafikon | `ui.js`, `style.css` | A 6.2 ellenőrzőlista UI-pontjai teljesülnek |
| F5 | Build: inline-olás egy fájlba | `munka/eszkozok/build.py` → `futtathato/bb84_szimulator.html` | A fájl `file://`-ból, hálózat nélkül fut Chrome-ban (headless ellenőrzés + kézi próba) |
| F6 | Dokumentáció: tartalom, képernyőképek, PDF | `dokumentacio/BB84_dokumentacio.pdf` | ≥ 3 teljes oldal, minden kötelező fejezet benne van (lásd 6.3) |
| F7 | Végső QA és csomagolás | `BB84_<Neptunkód>.zip` | A 7. fejezet minden pontja kipipálva |

---

## 6. Ellenőrzési feltételek

### 6.1 Automatikus logikai tesztek (F2) – a P1 bizonyítéka
- **T1:** Eve és zaj nélkül, egyező bázisoknál Bob bitje mindig megegyezik Alice-ével, és a QBER = 0%.
- **T2:** Eltérő bázisnál Bob bitje statisztikailag 50/50 (N = 100 000; 0,49–0,51 közé esik).
- **T3:** Teljes intercept-resend esetén a QBER ≈ 25% (N = 100 000; 0,24–0,26).
- **T4:** p arányú lehallgatásnál a QBER ≈ p/4 (p = 0,2 / 0,5 / 0,8).
- **T5:** A sifted kulcs hossza ≈ N/2.
- **T6:** Azonos seed esetén a kimenet bitre pontosan megegyezik (determinizmus).
- **T7:** Kézi bemenet: a megadott bitek és bázisok pontosan ezek a beállított értékek lesznek; hibás bemenetre validációs hiba jön.
- **T8:** Léptetés: `next()`/`prev()` után az állapot konzisztens, és ugyanarra a lépésre visszalépve ugyanazt mutatja.
- **T9:** Zaj nélkül, Eve-vel a küszöb fölötti QBER esetén a döntés „elvetve”; Eve nélkül a döntés „elfogadva”.

Futtatás: a tesztek egy `teszt.html` oldalon futnak, headless Chrome-mal (`--dump-dom`). Node nincs telepítve, ezért nem azzal futtatjuk.

### 6.2 Programszempontok ellenőrzőlistája (Sz2 → 5)
- [ ] **P1:** a T1–T9 tesztek zöldek; kézi ellenőrzéssel egy 8 fotonos példa papíron is kijön.
- [ ] **P2:** Alice bitjei, Alice bázisai, Bob bázisai (és Eve bázisai) szövegesen is és kattintással is megadhatók.
- [ ] **P3:** Eve be- és kikapcsolható, külön vizuális elemként jelenik meg, és mérései láthatók.
- [ ] **P4:** Eve bekapcsolásakor a QBER látványosan megnő, a hibák pirossal ki vannak emelve, és a protokoll elveti a kulcsot.
- [ ] **P5:** mind a 10 lépés egyenként léptethető előre és vissza, mindegyikhez magyarázat tartozik.

### 6.3 Dokumentáció ellenőrzőlistája (Sz1 → 5)
Célhossz 4–6 oldal, hogy a „≥ 3 oldal” biztosan teljesüljön (A4, 11 pt).
- [ ] Bevezetés és a BB84 protokoll rövid elmélete (bázisok, mérés, intercept-resend, QBER, 11%-os küszöb)
- [ ] **Rendszerkövetelmények és indítás:** Windows + Chrome/Edge/Firefox, dupla kattintás, nincs telepítés
- [ ] **Felhasznált technológiák és könyvtárak:** HTML5, CSS3, JavaScript ES2020, Web Crypto API, SVG/Canvas; *külső könyvtár nincs*; esetleges AI-használat megjelölése a kari irányelv szerint
- [ ] **Kezelési útmutató képernyőképekkel:** bemenetek, léptetés, Eve, összehasonlító mód
- [ ] Mintafuttatás: egy példa Eve nélkül és egy Eve-vel, táblázattal és értelmezéssel
- [ ] A program felépítése: fájlok, modulok, fő függvények
- [ ] Tesztelés: a T1–T9 tesztek és eredményeik
- [ ] Korlátok és továbbfejlesztési lehetőségek
- [ ] Címlap: név, Neptun-kód, tárgy, félév, feladat száma

Eszköz: Markdown → pandoc → DOCX/PDF, vagy HTML → PDF Chrome headless-szel. Utána ellenőrizzük az oldalszámot.

### 6.4 UX ellenőrzőlista (Sz3 → 5)
- [ ] Bemutatható egy hallgatótársnak 3 perc alatt, magyarázat nélkül is érthető (lépésszövegek)
- [ ] Animáció, színkódolás, összehasonlító mód (a „frappáns” elem)
- [ ] Nincs angol vagy félbehagyott felirat, nincs konzolhiba
- [ ] 1366×768-as felbontáson (tipikus laptop) is áttekinthető
- [ ] Edge-ben is ellenőrizve (a Windows alapértelmezett böngészője)

---

## 7. Végső leadási ellenőrzés (F7)
- [ ] `futtathato/bb84_szimulator.html` megnyitva egy **friss mappából, internet nélkül**: hibátlanul fut
- [ ] A forráskód és a futtatható verzió ugyanabból a buildből készült
- [ ] A dokumentáció PDF ≥ 3 oldal, megnyílik, és a képek olvashatók
- [ ] A zip-ben **csak** a `leadando/` tartalma van (nincs `munka/`, nincs `.DS_Store`)
- [ ] Feltöltés a Moodle **késedelmes** beadási felületére, a határidő előtt
- [ ] A Neptun-üzenetek figyelése az utólagos adminisztratív teendők miatt
