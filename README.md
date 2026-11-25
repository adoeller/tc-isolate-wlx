# WLX Isolate

[English documentation](readme_en.md)

Generischer Lister-Wrapper für Total Commander. Das eigentliche WLX läuft in
einem eigenen Prozess pro Viewer, nicht in Total Commander. Ein Absturz des
Zielplugins kann dadurch als Fehler im Viewer angezeigt werden; über **Plugin
neu starten** lässt sich derselbe Viewer mit einem neuen Hilfsprozess öffnen.

Erster realer Testfall ist CudaLister 1.8.6.3. Der Wrapper lädt und entlädt das
Zielplugin auf demselben langlebigen UI-Thread des Helpers. Insbesondere wird
die Original-DLL auch für die Erkennungsabfrage nie in den TC-Prozess geladen.

## Für die vorhandene CudaLister-Installation

Die ausdrücklich angeforderte `wlx_isolate.ini` ist bereits im Projektordner
angelegt. Sie verweist für Win64 auf:

`C:\Program Files\totalcmd\plugins\wlx\CudaLister\cudalister.wlx64`

Als bisherige Einstellungsdatei ist
`C:\Program Files\totalcmd\lsplugin.ini` hinterlegt. Der vorhandene Endungsfilter
für txt, ini, yaml, readme, list, md und pas ist übernommen.

**Manuell in Total Commander den Wrapper anstelle des bisherigen CudaLister-
Eintrags auswählen.** Für Win64 gehören diese Dateien zusammen in einen Ordner:

- `wlx_isolate.wlx64`
- `wlx_isolate_host64.exe`
- `wlx_isolate.ini`

Alternativ kann der Wrapper direkt aus diesem Projektordner verwendet werden.
TC wählt beim normalen `.wlx`-Eintrag die danebenliegende `.wlx64`-Variante.
Die 32-Bit-Variante verwendet `wlx_isolate.wlx` und `wlx_isolate_host.exe` und
benötigt eine tatsächliche 32-Bit-DLL des Zielplugins. Im angegebenen installierten
CudaLister-Ordner war bei der Einrichtung nur die 64-Bit-DLL vorhanden.

Es wurden keine installierten Plugins, TC-Einstellungen oder Archive verändert.
Keine Original-DLL in den Wrapper umbenennen oder durch ihn überschreiben.

## Originalordner und Original-Einstellungen

`Plugin`, `Plugin32` und `Plugin64` zeigen auf die Originaldatei. Der Helper
setzt sein Arbeitsverzeichnis vor dem Laden auf deren Ordner und verwendet
`LoadLibraryExW(..., LOAD_WITH_ALTERED_SEARCH_PATH)`. Der Modulpfad des Zielplugins
bleibt dessen tatsächlicher Pfad. Benachbarte Lexer, Themes, DLLs und sonstige
Ressourcen bleiben dort liegen und müssen nicht zum Wrapper kopiert werden.

Eine zum Zielplugin gehörende `<Zielplugin>.ini` neben der Original-DLL hat
Vorrang. Andernfalls wird `BackendIni`, oder bei leerem Wert der von TC an
`ListSetDefaultParams` übergebene INI-Pfad, weitergereicht. Der Wrapper schreibt
diese INIs nicht; das Zielplugin kann seine normalen Einstellungen dort speichern.
Die WLX-ABI beschränkt den übergebenen INI-Pfad auf ein ANSI-Feld mit MAX_PATH.
Nicht verlustfrei darstellbare oder zu lange Pfade werden abgelehnt.

## Konfiguration

Für andere Plugins `wlx_isolate_default.ini` manuell als `wlx_isolate.ini` kopieren
und den Zielpfad setzen. Die Default-Datei ist nur eine Vorlage und wird nicht
zur Laufzeit gelesen. Bei umbenannter Wrapper-DLL heißt ihre INI entsprechend
dem neuen Basisnamen; die Hilfsprogramme behalten ihre Dateinamen.

| Schlüssel unter `[Wrapper]` | Standard | Bedeutung |
|---|---|---|
| `Plugin` | leer | Original-WLX; relative Pfade beziehen sich auf den Wrapperordner. Umgebungsvariablen werden aufgelöst. Bei Win64 wird zu `.wlx` eine vorhandene `.wlx64` bevorzugt. |
| `Plugin32` | leer | Explizites Ziel für Win32, sonst `Plugin`. |
| `Plugin64` | leer | Explizites Ziel für Win64, sonst `Plugin`. |
| `BackendIni` | leer | INI-Fallback des Zielplugins, siehe oben. |
| `DetectString` | leer | Expliziter TC-Erkennungsstring. Leer fragt das Original in einem kurzlebigen isolierten Helper ab. |
| `FallbackOnReject` | 1 | Bei regulärer Ablehnung der ersten Datei in TCs internen Textmodus (Taste `1`) wechseln; `0` behält die Fehlerfläche für Diagnose. |
| `StartupTimeoutMs` | 15000 | Zeitlimit für Initialisierung und asynchrones Datei-Nachladen. |
| `CommandTimeoutMs` | 3000 | Zeitlimit für Befehle/Suche. Sichtbare Dialoge des Zielplugins dürfen auf die Nutzereingabe warten. |
| `ShutdownTimeoutMs` | 3000 | Warteintervall beim Schließen. Bei ausbleibender Antwort ohne sichtbaren Originaldialog kann der Nutzer weiter warten oder den Helper beenden. |
| `Language` | auto | `auto`, `de` oder `en` für die Wrapperoberfläche. |

Zeitlimits akzeptieren 100 bis 300000 ms; ungültige Werte verwenden den Standard.
Inline-Kommentare nach Leerraum mit `;` oder `#` werden außerhalb von Quotes
entfernt. Gequotete Werte und URL-Fragmente bleiben erhalten.

## Unterstützte WLX-Funktionen

- ANSI- und Unicode-Laden, Nachladen, Schließen und Erkennungsstring.
- `ListSetDefaultParams`, `ListSendCommand`, Textsuche, Suchdialog und Drucken.
- ANSI-Fallback, wenn das Original nur ANSI-Exporte anbietet; nicht in der
  aktuellen Windows-Codepage darstellbare Texte werden nicht still ersetzt.
- Native Einbettung, Größenänderung, QuickView-Fokus, weitergeleitete Tasten
  und die üblichen Lister-Statusmeldungen.
- Ein aus dem Vordergrund geöffneter F3-Lister wird nach dem asynchronen Laden
  aktiviert und erhält den Tastaturfokus im Originaleditor. QuickView lässt
  den Fokus im TC-Panel und übernimmt ihn erst bei einem expliziten Wechsel
  in die Vorschau. Ein Fensterwechsel während des Ladens wird respektiert.
- Der Wrapper reagiert bei `lc_newparams` auf TC-Dark-Mode und reicht die
  Parameter weiter. Die Darstellung des Originalplugins bestimmt dieses selbst.

Dateiöffnung und Nachladen sind asynchron: Der Wrapper gibt früh ein Childfenster
zurück. Lehnt das Original die erste Datei später regulär ab, wird der Helper
geschlossen und genau eine Modus-Taste `1` an den betreffenden Lister-Parent
gepostet. Damit verlässt TC die Pluginanzeige und verwendet seinen internen
Textmodus, ohne das Zielplugin erneut zu öffnen. QuickView behält den Panelfokus.
Gedrückte Strg/Alt/Umschalt-Tasten verzögern den Wechsel bis zum Loslassen.
Abstürze, Timeouts und Startfehler bleiben zur Diagnose sichtbar; ebenso eine
Ablehnung mit `FallbackOnReject=0`. Bei
abgelehntem Nachladen bleibt der vorherige Editor erhalten und eine Meldung
erscheint darüber. Ein fehlender
Nachlade-Export wird sofort mit dem normalen WLX-Fehlerwert beantwortet.

Bei CudaLister kann die Meldung „Das Plugin konnte diese Datei nicht anzeigen“
eine reguläre Ablehnung binärer Inhalte bedeuten. Eine kompilierte `.o`-Datei
ist beispielsweise keine Textdatei; auch erzwungene Pluginanzeige ändert diese
Prüfung im Original nicht. Der Wrapper wechselt dann automatisch zur Textanzeige;
für Binärdaten kann anschließend TCs Hexmodus gewählt werden.
Für den Quelltext die zugehörige `.pas`-, `.pp`- oder `.lpr`-Datei öffnen. Ein Neustart
des Helpers behebt diese inhaltliche Ablehnung nicht.

**Grenzen:** Kein universelles Kompatibilitätsversprechen. Prozessübergreifende
Fenster, modale Dialoge, Fokus und besondere Host-Annahmen müssen pro Plugin
praktisch geprüft werden. Thumbnails (`ListGetPreviewBitmap*`) und ein direkter
TC-Export für beliebige `WM_NOTIFY`-Strukturen sind nicht implementiert. Native
Notifications am Helper-Parent können innerhalb des Helperprozesses an das
Original weitergereicht werden. Plugins mit solchen Spezialanforderungen brauchen
eine gesonderte Erweiterung. Windows kann bei eingebetteten fremden Fenstern
Eingabequeues koppeln; begrenzte IPC-Wartezeiten garantieren daher keine völlige
Unabhängigkeit von jedem möglichen GUI-/Treiber-Hänger.

Absturzisolierung ist keine Sicherheits-Sandbox für bösartige Plugins. Ein
Absturz oder erzwungenes Beenden des Helpers verliert dessen ungespeicherte
Änderungen. Bearbeiten und Speichern zum Testen ausschließlich mit Testkopien.

## Lebensdauer

Pro Viewer eigene Mapping-/Event-Handles, die ausschließlich über eine explizite
Handle-Liste vererbt werden. Der Helper erbt nur einen SYNCHRONIZE-Handle auf den
Hostprozess. Ein Job-Object mit `KILL_ON_JOB_CLOSE` verhindert verwaiste Helper
bei Hostende. Normales Schließen ruft den Original-Schließexport und FreeLibrary
auf dem Helper-UI-Thread auf. Absturz-/Timeout-Pfade beenden nur den jeweiligen
Helper. Die Wrapper-DLL besitzt keine Worker oder nach dem Schließen lebenden
Fensterprozeduren. Ihre Fensterklasse wird nach dem letzten Viewer abgemeldet;
DLL-Detach enthält nur den kurzen Klassen-Fallback.

## Bauen und prüfen

Toolchain: vorhandenes Zig 0.16.0 (`zig cc`), Windows-API, keine LCL und keine
Free-Pascal-RTL in der Wrapper-DLL. Deshalb wird dieses C-Projekt mit dem
Buildskript und nicht mit lazbuild gebaut. Alle Caches und Ergebnisse bleiben
im Projektordner.

Wrapper und Hilfsprozess immer zusammen aktualisieren. Ihre interne
Protokollversion muss übereinstimmen (aktuell 2).

```powershell
.\build.ps1 -Target all -Tests
python test\run_tests.py --bits all
python test\run_tests.py --bits all --real
python test\run_tests.py --bits all --variant ansi
python test\run_tests.py --bits all --variant no-next
python test\check_orphans.py
```

Die realen Tests verwenden ausschließlich DLL-Kopien aus dem bereits vorhandenen
Referenzarchiv unter `analysis_cudalister`, kopierte Ressourcen und Test-INIs.
Es werden keine echten Nutzerdokumente bearbeitet. Ergebnisse stehen in
`test/run32`, `test/run64`, `test/real32` und `test/real64`, jeweils `results.log`.
SHA-256-Zuordnungen stehen in `test/results.json` und `test/real-results.json`.
Aktueller Prüfstand und verbleibende Grenzen: `STATUS.md`.
