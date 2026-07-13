# Projekt: Lokales Push-to-Talk-STT zur Agentensteuerung (Windows, CPU-only)

## Ziel

Baue eine Python-Anwendung für Windows, die per globalem Hotkey (Push-to-Talk)
Sprache aufnimmt, lokal mit faster-whisper transkribiert und das Transkript an
einen konfigurierbaren Ausgabekanal weitergibt. Sie dient der Sprachsteuerung
von KI-Agenten auf einem Entwicklerrechner.

## Zielumgebung (WICHTIG — weicht von deiner Build-Umgebung ab)

- Zielsystem: Windows 11, Python 3.11+
- Hardware: AMD Ryzen 7 PRO 5850U (8C/16T), KEINE dedizierte GPU — alles muss
  CPU-only laufen (CTranslate2 mit compute_type="int8")
- Du selbst läufst in einem abgeschotteten Linux-Container OHNE Audiogeräte
  und OHNE GPU. Du kannst Mikrofonaufnahme und Hotkeys NICHT interaktiv
  testen. Konsequenzen:
  - Architektur so entwerfen, dass Audioquelle und Hotkey-Layer hinter
    Interfaces liegen und durch Fakes/Fixtures ersetzbar sind
  - Tests ausschließlich mit WAV-Fixtures und gemockter Aufnahme
  - Windows-spezifische Teile (Hotkey, Audiogerät) sauber isolieren und im
    README als "auf Zielsystem manuell zu verifizieren" markieren

## Funktionale Anforderungen

1. **Push-to-Talk per globalem Hotkey**
   - Hotkey gedrückt halten = Aufnahme läuft, loslassen = Aufnahme endet und
     Transkription startet
   - Zwei konfigurierbare Hotkeys:
     - Hotkey 1 → Transkription mit language="de"
     - Hotkey 2 → Transkription mit language="en"
   - Default-Belegung vorschlagen (z. B. F9 / F10), in Config änderbar
   - Bibliothek: `keyboard` oder `pynput` — wähle die robustere Option für
     globale Hotkeys unter Windows und begründe die Wahl kurz im README
     (inkl. Hinweis, falls Adminrechte nötig sind)

2. **Audioaufnahme**
   - 16 kHz mono, via `sounddevice`
   - Aufnahmegerät in Config wählbar (Default: System-Default)
   - Maximale Aufnahmedauer als Sicherheitsnetz (Default 30 s)

3. **Transkription (lokal, CPU-only)**
   - `faster-whisper`, Modell per Config (Default: "small"), compute_type
     "int8", Sprache fest gemäß gedrücktem Hotkey (KEIN Auto-Detect)
   - `initial_prompt` aus einer Vokabular-Datei (`vocabulary.txt`) laden und
     bei jeder Transkription mitgeben. Lege eine Beispieldatei mit
     Entwickler-Fachvokabular an (git commit, push, pull request, merge,
     branch, rebase, Docker, Container, pipeline, npm install, pytest,
     Refactoring, Repository, staging, deployment, ...)
   - VAD-Filter von faster-whisper aktivieren, um Stille zu trimmen
   - Modell-Download beim ersten Start automatisch (Hugging Face Cache),
     mit klarer Konsolenmeldung

4. **Ausgabekanäle (Plugin-Prinzip, per Config wählbar, mehrere gleichzeitig
   erlaubt)**
   - `stdout`: Transkript auf Konsole
   - `clipboard`: Transkript in Zwischenablage (pyperclip)
   - `http`: POST als JSON `{"text": ..., "language": ..., "duration_s": ...}`
     an konfigurierbare URL (für die spätere Agenten-Anbindung)
   - `keystrokes`: Transkript als simulierte Tastatureingabe ins aktive
     Fenster tippen (optional aktivierbar, Default aus)

5. **Feedback für den Nutzer**
   - Dezente akustische oder Konsolen-Signale für: Aufnahme gestartet,
     Aufnahme beendet, Transkript fertig (inkl. Anzeige des Texts)
   - Latenzmessung: Zeit von "Taste losgelassen" bis "Transkript fertig"
     loggen, damit der Nutzer small vs. medium vergleichen kann

## Nicht-funktionale Anforderungen

- Geringer Idle-Overhead: Modell einmal beim Start laden und im RAM halten;
  im Leerlauf keine nennenswerte CPU-Last (kein Polling-Loop mit hoher
  Frequenz)
- Konfiguration über eine einzige `config.yaml` mit kommentierten Defaults
- Sauberes Logging (Level per Config), keine Debug-Flut im Normalbetrieb
- Klare Projektstruktur, z. B.:
  ```
  stt_ptt/
    __main__.py        # Entry Point: python -m stt_ptt
    config.py
    audio.py           # Aufnahme (Interface + sounddevice-Implementierung)
    hotkeys.py         # Hotkey-Layer (Interface + Windows-Implementierung)
    transcriber.py     # faster-whisper-Wrapper
    outputs/           # Ausgabekanäle als Plugins
  tests/
    fixtures/          # kurze WAV-Dateien (siehe Tests)
  config.yaml
  vocabulary.txt
  requirements.txt
  README.md
  ```

## Tests (müssen in deinem Container ohne Audio-Hardware laufen)

- Unit-Tests für Config-Parsing, Vokabular-Laden, Output-Plugins (http via
  Mock-Server oder responses), Transkriptions-Pipeline mit gemockter
  Whisper-Klasse
- Ein Integrationstest, der eine echte kurze WAV-Fixture durch faster-whisper
  (Modell "tiny", nur für den Test) schickt, um die Pipeline Ende-zu-Ende zu
  prüfen. Erzeuge die Fixture synthetisch (z. B. Sinuston + Stille) oder
  nutze eine frei lizenzierte kurze Sprachdatei; wenn im Container kein
  Modell-Download möglich ist, markiere den Test als skippable mit klarer
  Begründung
- Keine Tests, die echte Hotkeys oder Mikrofone erfordern

## README muss enthalten

- Setup auf Windows: venv anlegen, `pip install -r requirements.txt`,
  erster Start und Modell-Download
- Hinweis auf Adminrechte, falls die Hotkey-Bibliothek sie braucht
- Anleitung zum Qualitäts-Tuning: Modell small vs. medium wechseln,
  vocabulary.txt erweitern, typische Latenzwerte auf einem 5850U als
  Orientierung
- Kurze Checkliste "Manuell auf dem Zielsystem verifizieren" (Hotkeys,
  Mikrofonwahl, Keystroke-Ausgabe)

## Vorgehen

Arbeite iterativ: erst Projektgerüst + Config + Tests, dann Transkription,
dann Aufnahme/Hotkeys, dann Output-Plugins. Committe in sinnvollen Schritten
mit aussagekräftigen Messages, damit das Repo sauber per GitHub übertragen
werden kann. Wenn du Annahmen treffen musst (z. B. Bibliothekswahl),
dokumentiere sie im README statt nachzufragen.