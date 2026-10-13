# Semantic Zotero: Was für Zotero 7–10 aktualisiert werden muss

Stand: 30.09.2026. Grundlage sind der Code in diesem Verzeichnis (upstream `AgiNetz/semantic-zotero`,
letzter Commit `aa7ad52` vom 24.10.2023, Version 0.2) und Testanfragen an die Semantic-Scholar-API vom selben Tag.

## Kurzfassung

- **Das Plugin läuft in aktuellem Zotero nicht.** Es ist ein Zotero-6-Plugin (`install.rdf`, XUL-Overlay,
  `chrome.manifest`). Zotero 7 hat diese Plugin-Art abgeschafft; das Plugin lässt sich nicht einmal installieren.
- **Die fachliche Logik ist klein und größtenteils brauchbar** (~500 Zeilen JavaScript): Eintrag auf Semantic
  Scholar finden, Referenzen holen, Liste anzeigen, fehlende Referenzen mit PDF anlegen.
- **Nötig ist ein Umbau der Hülle** (Bootstrap-Plugin mit `manifest.json`, Menüs per Code, Einstellungen im
  Zotero-Einstellungsfenster) und **robustere API-Nutzung** (Fehler 429, fehlende Daten).
- **API-Key:** Ohne Key funktioniert es grundsätzlich, aber unzuverlässig: Die Anfragen ohne Key teilen sich
  ein globales Kontingent, im Test kam Fehler 429 schon bei der ersten Suchanfrage. Mit Key (kostenlos auf Antrag) ist es stabil. Das
  Plugin muss beides können: mit Key arbeiten und ohne Key sauber warten und wiederholen.
- Aufwand grob: **1–2 Tage** für eine lauffähige, getestete Version; mehr, wenn neue Funktionen dazukommen.

## 1. Was das Plugin heute tut

| Datei | Aufgabe |
|---|---|
| `install.rdf`, `chrome.manifest` | Plugin-Beschreibung, nur Zotero `6.0`–`6.*`; hängt `overlay.xul` in `zoteroPane.xul` ein |
| `chrome/content/overlay.xul` | Menüpunkt „Semantic Zotero Options“ unter Werkzeuge; Kontextmenü „Semantic Zotero → Show References“ |
| `semanticZotero.js` | Semantic-Scholar-ID ermitteln (URL, arXiv, DOI, sonst Titelsuche), Referenzen abrufen, Liste im Fenster `references.html` aufbauen |
| `addPaperDialog.html`, `addPaper.js` | Dialog: Sammlungen und Tags wählen, Referenz als neuen Eintrag anlegen, PDF anhängen, Einträge verknüpfen |
| `options.html`, `options.js` | Eigenes Fenster für API-Key und „Einträge verknüpfen“ |

## 2. Was an der Plugin-Hülle geändert werden muss

Zotero 7 basiert auf Firefox 115 (Zotero 8–10 auf neueren ESR-Versionen). Overlays und `install.rdf` gibt es
dort nicht mehr. Anleitung: <https://www.zotero.org/support/dev/zotero_7_for_developers>.

1. **`install.rdf` und `chrome.manifest` ersetzen** durch `manifest.json` (WebExtension-Stil) mit
   `applications.zotero`: `id`, `update_url`, `strict_min_version: "6.999"`, `strict_max_version: "10.*"`.
   Die ID `tomasdanis26@gmail.com` gehört dem Originalautor; ein eigener Fork braucht eine eigene ID.
2. **`bootstrap.js` hinzufügen** mit `startup`, `shutdown`, `onMainWindowLoad`, `onMainWindowUnload`,
   `install`, `uninstall`. Darin `Zotero.PreferencePanes.register(...)` und das Laden von
   `semanticZotero.js` über `Services.scriptloader.loadSubScript`. Chrome-URLs werden per
   `aomStartup.registerChrome` registriert, falls weiter `chrome://semanticZotero/...` genutzt wird.
3. **Menüs per Code statt Overlay.**
   - Kontextmenü: `Zotero.MenuManager.registerMenu` (Zotero 8+) oder, für Zotero 7, per DOM in `zotero-itemmenu`
     (`document.createXULElement('menu')`) in `onMainWindowLoad`, und in `onMainWindowUnload` wieder entfernen.
   - Der Menüpunkt unter Werkzeuge entfällt, weil die Einstellungen ins Einstellungsfenster wandern (Punkt 4).
   - Das Overlay nutzt `<popup id="zotero-itemmenu">`. `<popup>` ist in neuerem Firefox kein gültiges Element mehr.
   - Die ID des Untermenüs `zotero-itemmenu-citationcounts-menupopup` ist aus einem anderen Plugin kopiert und
     kann kollidieren. Sie muss eindeutig werden.
4. **Einstellungen ins Zotero-Einstellungsfenster verlegen** (`prefs.xhtml` und `Zotero.PreferencePanes.register`)
   und Standardwerte in `prefs.js` festlegen.
   - Vorschlag für die Präferenz-Namen: `extensions.semanticzotero.apiKey` und `extensions.semanticzotero.relateItems`.
   - Heute speichert das Plugin unter `extensions.zotero.SemanticZotero.*`. Das funktioniert noch, gehört aber nicht in den Namensraum von Zotero.
   - Der Key steht im Klartext in `prefs.js`. Das ist üblich, gehört aber als Hinweis in die Einstellungen.
5. **Fenster.** `window.open("chrome://…/references.html", …, "chrome,…")` und `window.openDialog(...)` funktionieren
   im Prinzip weiter, wenn die Chrome-URLs registriert sind. Dazu drei Anpassungen:
   - Die Seiten als `.xhtml` mit XHTML-Namensraum führen; das ist in Zotero 7 der übliche Weg.
   - `ZoteroPane`, `window.opener.SemanticZotero` und `Zotero.getActiveZoteroPane()` sind im neuen Aufbau nicht
     automatisch im Dialogfenster verfügbar. Objekte werden über `window.arguments` bzw.
     `Zotero.getMainWindow()` übergeben.
   - Inline-Handler wie `onclick="SemanticZoteroOptions.saveOptions()"` durch `addEventListener` ersetzen.
6. **Globaler Zustand.** `SemanticZotero` hängt heute als `var` am Hauptfenster. Im Bootstrap-Modell lebt das
   Objekt im Plugin-Scope und wird beim `shutdown` aufgeräumt (Menüs entfernen, Fenster schließen), sonst
   bleiben nach dem Deaktivieren Reste.
7. **Oberfläche.**
   - Texte sind heute hart kodiertes Englisch; mit Fluent (`.ftl`, `locale/de-DE`, `locale/en-US`) lassen sie sich übersetzen.
   - Das Referenzfenster braucht ein Stylesheet für den Dunkelmodus: Zotero 7 hat ihn, und fest gesetzte Farben sind dann schlecht lesbar.
8. **Build und Release.**
   - Heute gibt es kein Build: Die `.xpi` ist nur ein Zip der Dateien. Das bleibt möglich.
   - Alternativ als Grundgerüst das Zotero-Plugin-Template (`windingwind/zotero-plugin-template`, TypeScript, esbuild, Hot Reload).
   - Für automatische Updates eine `updates.json` bereitstellen, zum Beispiel über das Portal unter `/downloads/semantic-zotero/`, wie bei SeekChat, ZotSeek und SeekBook.

## 3. Fehler und Schwächen in der Logik, die beim Umbau behoben werden sollten

| Stelle | Problem | Folge | Lösung |
|---|---|---|---|
| `fetchReferences` | Bei Verlagen, die Referenzen sperren (z. B. Elsevier), liefert die API `"data": null` und im `disclaimer` den Hinweis „fields have been elided by the publisher: {'references'}“ | `data["data"].map` wirft TypeError, die Nutzerin sieht „Unknown internal error“ | `data` auf `null` prüfen, verständliche Meldung („Der Verlag gibt die Referenzliste nicht frei“) |
| `fetchReferences` | Nur Seite 1 (`limit=1000`), ohne `offset`/`next` | Bei sehr langen Literaturlisten fehlt der Rest (selten) | `next` auswerten und weitere Seiten holen |
| Fehlerbehandlung | Nur 404 und 403 werden unterschieden; 429 (zu viele Anfragen) und Netzwerkfehler landen bei „Unknown internal error“ | Ohne Key häufig; wirkt wie ein Absturz | Bei 429 mit Wartezeit wiederholen (exponentiell, 1 s → 2 s → 4 s …, `Retry-After` beachten), danach klare Meldung mit Hinweis auf den API-Key |
| Fehler 403 | Wird nur bei der ersten Anfrage behandelt; bei der Titelsuche im Fallback nicht | Ungültiger Key erscheint als „Unknown internal error“ | Einheitliche Fehlerbehandlung für alle Anfragen |
| `searchSemanticScholarByTitle` | Nimmt blind den ersten Treffer von `/paper/search` | Bei ähnlichen Titeln wird das falsche Paper genommen | Treffer mit Titel und Jahr vergleichen; optional erst `/paper/search/match` (exakt), dann `/paper/search` (unscharf). Im Test fand `match` den Titel wegen „Nanometre“ statt „Nanometer“ nicht, deshalb ist der Rückfall auf die unscharfe Suche nötig |
| `determineItemId` | URL hat Vorrang vor DOI, arXiv-ID aus `archiveID` wird ungeprüft übernommen | `archiveID` ist oft `arXiv:2101.00001`; die API erwartet `ARXIV:2101.00001` | DOI zuerst, dann arXiv (Präfix normalisieren), dann URL. In Zotero 7 steht die DOI bei Preprints auch in `extra`; dort mitlesen |
| `determineItemId` | Kein Fall für Einträge ohne Titel oder für Anhänge | Fehler bei ausgewähltem PDF-Anhang oder Notiz | Nur reguläre Einträge zulassen, sonst Menüpunkt deaktivieren |
| `showReferences` | Öffnet das Fenster vor der Abfrage und prüft `ZoteroPane.canEdit()`; bei nicht editierbarer Bibliothek bleibt ein leeres Fenster offen | Leeres Fenster | Erst prüfen, dann öffnen; Ladeanzeige im Fenster |
| `buildLibraryMap` | Lädt **alle** Einträge der Bibliothek und vergleicht nur Titel (klein geschrieben) | Langsam bei großen Bibliotheken; falsche Treffer bei gleichen Titeln, fehlende bei kleinen Abweichungen | Abgleich über DOI/arXiv-ID (`Zotero.Items` per Suche), Titel nur als Rückfall, normalisiert |
| `addToCollection` | Legt jeden Treffer als `preprint` an und übernimmt keine DOI, Zeitschrift oder Seiten | Schlechte Metadaten; Zitierstil falsch | Besser: vorhandene DOI/arXiv-ID an Zoteros eigene Übersetzer geben (`Zotero.Translate.Search`, wie „Hinzufügen über Identifier“). Nur ohne ID auf die heutigen Felder zurückfallen |
| `addToCollection` | `reference.isOpenAccess` ohne `openAccessPdf` → `openAccessPdf.url` auf `null` | TypeError beim Anlegen (in `semanticZotero.js` schon behoben, in `addPaper.js` nicht) | Gleiche Prüfung wie in `populateReferences` |
| `addToCollection` | Autorennamen werden am letzten Leerzeichen geteilt | „van der Berg“, „Ng Wei Ming“ usw. falsch | `Zotero.Utilities.cleanAuthor(name, 'author')` nutzen |
| `addToCollection` | Zwei Speichervorgänge je Eintrag für die Verknüpfung, ohne Transaktion | Halbfertige Einträge bei Fehlern | `Zotero.DB.executeTransaction` oder `addRelatedItem` vor `saveTx` beider Einträge |
| `addPaper.js` | `label.htmlFor = checkbox.key` (Tippfehler, `key` existiert nicht) | Klick auf den Sammlungsnamen setzt kein Häkchen | `checkbox.id` |
| `addPaper.js` | `Zotero.Tags.getAll()` ohne Bibliothek | Tags aus allen Bibliotheken, auch Gruppen | `Zotero.Tags.getAll(libraryID)` |
| `addPaper.js` | Sammlungen der **ausgewählten** Bibliothek, aber der Eintrag wird ohne `libraryID` angelegt | Beim Arbeiten in einer Gruppenbibliothek landet der neue Eintrag in „Meine Bibliothek“ und die Sammlungen passen nicht | `newItem.libraryID = item.libraryID` setzen |
| `populateReferences` | Referenzen ohne Titel (`reference.title` ist `null`) | `toLowerCase` auf `null` → Abbruch der ganzen Liste | Auf `null` prüfen |

## 4. Semantic Scholar API: mit und ohne Key

Getestet am 30.09.2026 gegen `https://api.semanticscholar.org/graph/v1`:

| Anfrage | Ergebnis |
|---|---|
| `GET /paper/DOI:10.1038/nature12373` ohne Key, 6× hintereinander | 6× 200 |
| `GET /paper/DOI:10.1038/nature12373/references?limit=1000&fields=title,contexts,openAccessPdf,isOpenAccess` ohne Key | 200; Antwort enthält zusätzlich `citingPaperInfo` (neu, stört nicht) |
| `GET /paper/DOI:10.1016/j.cell.2011.02.013/references` (Elsevier) | 200, aber `"data": null`, Hinweis auf gesperrte Referenzen |
| `GET /paper/search?query=…` und `/paper/search/match` ohne Key | zunächst 429 „Too Many Requests … apply for a key“, nach Pausen teils 200/404, teils wieder 429 |
| beliebige Anfrage mit ungültigem `x-api-key` | 403 |
| Key aus `.semanticscholar.apikey.secret` (40 Zeichen, Format unauffällig): Suche, `search/match`, `citations`, 5× Suche | jedes Mal 403 „Forbidden“, der Key ist also ungültig, abgelaufen oder noch nicht freigeschaltet |

Die verwendeten Endpunkte und Felder (`title`, `publicationDate`, `year`, `abstract`, `url`, `externalIds`,
`authors`, `isOpenAccess`, `openAccessPdf`, `citationCount`, `contexts`) gibt es weiterhin.

**Ohne Key**
- Alle Nutzer ohne Key teilen sich ein gemeinsames Kontingent. Wie viel frei ist, hängt von der Gesamtlast ab.
- Die Suchendpunkte sind besonders knapp.
- Ein einzelner Abruf per DOI klappt meistens; die Titelsuche als Rückfall scheitert oft.
- Das Plugin muss deshalb:
  - bei Fehler 429 mit Wartezeit wiederholen (siehe Abschnitt 3),
  - Anfragen hintereinander statt parallel stellen,
  - Ergebnisse kurz zwischenspeichern (Referenzen je `paperId`, z. B. 24 h im Speicher), damit ein erneutes Öffnen keine neue Anfrage auslöst,
  - bei anhaltendem 429 erklären, dass ein Key hilft, und auf die Einstellungen verweisen.

**Mit Key**
- Den Key gibt es kostenlos auf Antrag: <https://www.semanticscholar.org/product/api#api-key-form>.
- Er wird als Header `x-api-key` mitgeschickt; das macht das Plugin schon richtig.
- Der vorhandene Key in `.semanticscholar.apikey.secret` wird derzeit abgelehnt (403). Bei Semantic Scholar
  prüfen oder einen neuen beantragen. Ein ungültiger Key ist schlechter als keiner: Mit einem ungültigen Key scheitert
  jede Anfrage, ohne Key klappen wenigstens manche. Das Plugin sollte bei 403 deutlich sagen, dass der Key falsch ist.
  Die Datei ist über `.git/info/exclude` von Git ausgeschlossen.
- Das Kontingent gehört dann dem Key allein; die Einführungsrate liegt bei etwa 1 Anfrage pro Sekunde. Das
  Plugin muss seine Anfragen also auch mit Key drosseln (Warteschlange mit ≥ 1 s Abstand).
- Für das ILS bietet sich **ein gemeinsamer Institutsschlüssel** an. Den teilen sich dann alle Nutzer, und die Drosselung wird wichtiger.
  - Der Key sollte nicht fest im Plugin stehen, weil es öffentlich zum Download liegt.
  - Besser ist ein kleiner Proxy im Portal (wie `/translate/`): Er hängt den Key serverseitig an und ist nur für angemeldete Nutzer erreichbar.
  - Das Plugin bekäme dafür eine einstellbare Basis-URL: Standard `https://api.semanticscholar.org/graph/v1`, am ILS `https://zotero.ils.local/s2/…`.
  - Nebeneffekt: Die Suchanfragen laufen dann gebündelt über den Server statt direkt vom Rechner der Nutzer.

**Datenschutz:** Titel, DOIs und URLs der ausgewählten Einträge gehen an Semantic Scholar (Allen Institute for AI,
USA). Das weicht von der Aussage „Die Daten bleiben im Haus“ auf der Downloads-Seite ab und sollte dort und in den
Plugin-Einstellungen deutlich stehen.

## 5. Sinnvolle Erweiterungen (optional)

- **„Zitiert von“** (`/paper/{id}/citations`) zusätzlich zu den Referenzen. Das ist dieselbe Logik mit einem anderen Endpunkt.
- **Empfehlungen** (`https://api.semanticscholar.org/recommendations/v1/papers/forpaper/{id}`) für „ähnliche Paper“.
- **Mehrere Einträge auf einmal** (Batch-Endpunkt `POST /paper/batch`, bis 500 IDs). Spart Anfragen und damit Kontingent.
- Filter und Sortierung in der Liste (Jahr, Zitationen, „nur mit PDF“, „nur nicht in der Bibliothek“).
- Anzeige im Eintragsbereich (Item Pane Section, `Zotero.ItemPaneManager.registerSection`) statt eines eigenen Fensters.
- Abgleich mit dem ZotSeek-Index: Referenzen markieren, die inhaltlich schon durch eigene PDFs abgedeckt sind.

## 6. Vorgeschlagene Reihenfolge

1. Fork anlegen (eigene ID, Lizenz beachten: siehe `LICENSE`), Grundgerüst Zotero 7 (`manifest.json`, `bootstrap.js`,
   Kontextmenü, Einstellungsseite). Ziel: Plugin installiert sich in Zotero 7 und 10 und öffnet die Referenzliste.
2. API-Schicht neu: gemeinsame Funktion für alle Anfragen mit Key-Header, Drosselung, Wiederholung bei 429,
   klaren Fehlermeldungen, Umgang mit `data: null`, Seitenweise Abruf, Zwischenspeicher.
3. Fehlerliste aus Abschnitt 3 abarbeiten, vor allem Gruppenbibliotheken (`libraryID`) und Metadaten über Zoteros Übersetzer.
4. Tests: Einträge mit DOI, mit arXiv-ID, nur mit Titel, Elsevier-Paper, Gruppenbibliothek, ohne Key mit
   künstlich provoziertem 429, mit ungültigem Key (403), mit gültigem Key.
5. Optional: Portal-Proxy mit Institutsschlüssel, Veröffentlichung auf der Downloads-Seite mit `updates.json`.
