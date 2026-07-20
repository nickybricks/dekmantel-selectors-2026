# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

---

# Projekt: Dekmantel Selectors 2026 — Line-up Explorer

Inoffizielles Fan-Projekt. Eine einzelne statische `index.html` — kein Build, keine Dependencies, kein Backend. Gehostet auf GitHub Pages unter
https://nickybricks.github.io/dekmantel-selectors-2026/

**Sprache:** UI-Texte und Commit-Messages auf Deutsch, Code-Kommentare auf Englisch.

## Datenmodell

`FEATURED` ist ein Array von `[name, genre, blurb, socials]`.

`socials` enthält entweder direkt die Links (Solo-Act) oder eine `members`-Liste (B2B-Set):

```js
// Solo
{instagram:"…", spotify:"…", spotifyId:"…", soundcloud:"…"}

// B2B — ein Eintrag pro DJ
{members:[{name:"Job Jobse", soundcloud:"…"}, {name:"Peach", spotifyId:"…", soundcloud:"…"}]}
```

`membersOf()` normalisiert beides zu einer Liste, damit die Render-Logik nur noch Members kennt. Bei B2B gilt ein Act-Instagram **nicht** für alle — es zeigte sonst bei jedem Partner auf den erstgenannten DJ.

## Was bei Änderungen leicht kaputtgeht

Alles hier ist real passiert, nicht hypothetisch:

- **Player nicht unnötig neu mounten.** Der Herz-Button rief früher `renderDetail()` auf — das riss laufende Musik ab. Zustand in-place umschalten. Testen über Element-Identität: `iframe` vor und nach der Aktion muss dasselbe Objekt sein.
- **Timeouts gegen `Date.now()`, nicht gegen Tick-Zähler.** Ein `waited += 200`-Zähler wurde durch Timer-Throttling auf ein Vielfaches gestreckt: aus 5 Sekunden wurden 25.
- **Der Cover-Flow fängt Wheel-Events ab** (Absicht — „hier oben scrollen"). Ein Scroll über diesem Bereich blättert Artists, statt die Seite zu scrollen. Beim Testen `document.documentElement.scrollTop` setzen.
- **Dekorative Overlays brauchen `pointer-events:none`.** Die Cover-Spiegelung ragte 129 px unter den Viewport und schluckte Klicks auf die Act-Namen — der Klick landete auf dem Nachbar-Cover und sprang einen Act weiter.
- **Merkliste liegt in `localStorage`** (vorher `sessionStorage`, war beim Tab-Schließen weg).

## Externe Player — die nicht offensichtlichen Regeln

- **SoundCloud liefert `MONETIZE`-Tracks im anonymen Embed nicht aus.** Das Widget meldet „spielt", der Abspielkopf bleibt bei 0, kein `PLAY_PROGRESS`. Nur `policy === "ALLOW"` spielt verlässlich. Bei den meisten DJ-Mix-Accounts ist *kein einziger* Track abspielbar — deshalb prüft `warnIfUnplayable()` das per `SC.Widget.getSounds()` und weist es aus.
- **Spotify-Embeds sind deutlich verlässlicher** als SoundCloud. Wo beides existiert, ist Spotify der Default-Tab.
- **Beide Anbieter sind kein Fehlerfall, wenn sie fehlen.** Viele Selectors und Radio-Hosts haben schlicht keine Artist-Seite. `null` ist dort die richtige Antwort, kein Rechercheversäumnis.

## Artist-IDs: niemals ungeprüft übernehmen

Eine falsche ID setzt den **falschen Musiker** in ein veröffentlichtes Line-up. Recherchierte IDs sind Rohdaten, keine Ergebnisse. Jede ID zweifach maschinell prüfen:

1. **Namensabgleich** — `open.spotify.com/oembed?url=…` liefert `title`; muss zum erwarteten Künstler passen.
2. **Inhaltsprüfung** — `open.spotify.com/embed/artist/<id>` auf `trackList` prüfen. Es gibt korrekt benannte, aber **leere** Profile; die ergeben einen toten Player.

Namensgleiche Fallen sind der Normalfall, nicht die Ausnahme: bei „AMORAL" rankt eine finnische Death-Metal-Band zuerst, bei „Introspekt" ein unbeteiligter Künstler von 2003, bei „DJ Koolt" ein albanischer Popsänger. Suchtreffer-Titel reichen als Beleg nicht — das Profil öffnen.

## Verifizieren, nicht annehmen

- **Nicht nur das DOM prüfen, auch den Screenshot ansehen.** Zwei Regressionen waren im DOM unauffällig und erst im Bild sichtbar (Buttons zeigten „(suchen)", Instagram verwies auf den falschen DJ).
- **Für Klick-Ziele `document.elementFromPoint()`** auf die Mitte des Elements — das beweist, wer den Klick tatsächlich bekommt.
- **Kontrollprobe gegen die eigene Hypothese.** Dass Marcel Dettmanns SoundCloud nicht spielte, war erst mit Hunee als funktionierendem Gegenbeispiel bewiesen — vorher wäre die Autoplay-Sperre die genauso plausible Erklärung gewesen.
- **JS-Syntax vor dem Deploy prüfen:** Skriptblöcke extrahieren und `node --check` laufen lassen.

## Deployment

`git push origin main` löst den Pages-Build aus. **Nicht als deployed melden, bevor beides bestätigt ist:**

```bash
gh api repos/nickybricks/dekmantel-selectors-2026/pages/builds/latest --jq '.status, .commit'
curl -s https://nickybricks.github.io/dekmantel-selectors-2026/ | grep -c '<suchbegriff>'
```

Der Build braucht meist 30–60 Sekunden. Lokal testen mit `python3 -m http.server 8765` und danach den Prozess wieder beenden.

## Bewusst offen

Kein Rechercheversäumnis, sondern entschieden:

- **Instagram:** nur 11 von 70 Acts verifiziert verlinkt, der Rest fällt auf einen Suchlink zurück. Instagram-Inhalte lassen sich ohne Facebook-App-Token ohnehin nicht einbetten.
- **Optimo:** bewusst ohne Player. Nach dem Tod von JD Twitch im September 2025 wären die verfügbaren Links eine fast leere Remix-Credit-Seite oder sein privater Account.
- **Kein Timetable:** Set-Zeiten, Stages und Tage fehlen, weil Dekmantel sie noch nicht veröffentlicht hat.
- **Kein Analytics.**
