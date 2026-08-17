# Indexbaum

**Warum zwei Zeilen aus einer Million schnell gehen — und eine halbe Million nicht.** Ein B+-Baum
von Hand gebaut, mit gezählten Seitenzugriffen und einem Abfrageplaner, dem man beim Schätzen
zusehen kann.

→ **[Blatt öffnen](https://ssims437.github.io/indexbaum/)**

- **B+-Baum** mit einstellbarer Ordnung (4 bis 128), sichtbar als Ebenen; die Knoten, die die
  laufende Abfrage angefasst hat, sind hervorgehoben
- **Zwei Zugriffswege** auf dieselbe Abfrage: über den Index oder alles durchlesen — mit
  **gezählten** Seiten, nicht geschätzten
- **Kostenkurve** über die Trefferquote, mit gemessenem Kipppunkt: **3,4 %** bei verstreut
  abgelegten Zeilen, **33,4 %** wenn die Tabelle nach dem Schlüssel sortiert liegt
- **Der Planer** schätzt aus einem 64-Klassen-Histogramm und wählt — die Schätzung steht neben dem
  gemessenen Wert
- **Prüflauf** — 1891 Bereichsabfragen erschöpfend gegen den stumpfen Durchlauf, Baumbedingungen
  nach jeder einzelnen Einfügung und jedem Löschen

## Wie gezählt wird

Eine Datenbank liest Seiten, nicht Zeilen. Deshalb zählt das Blatt genau das:

- **Durchlesen**: alle Tabellenseiten, jede genau einmal → `ceil(N / Zeilen je Seite)`
- **Über den Index**: die Knoten auf dem Weg von der Wurzel zum ersten Blatt, dann jedes Blatt, das
  durchlaufen wird, plus für jeden Treffer die Tabellenseite — wobei eine Seite, die zweimal
  gebraucht wird, auch zweimal zählt, solange sie nicht dieselbe wie beim letzten Treffer ist.
  Genau so verhält sich ein Zugriff ohne Bitmap-Zwischenschritt.

Daraus folgt der Kipppunkt: der Index gewinnt, solange die Treffer wenige Seiten berühren. Liegen
die Zeilen verstreut, ist das schon bei ein paar Prozent vorbei.

## Was der Prüflauf zeigt

| Behauptung | Ergebnis |
|---|---|
| Bedingungen nach jeder Einfügung | 1600 Einfügungen über vier Ordnungen, kein Mangel |
| Bedingungen nach jedem Löschen | 1200 Löschungen bis zum leeren Baum, kein Mangel |
| Inhalt stimmt Schritt für Schritt | 358 Einfügungen + 242 Löschungen, nach **jedem** Schritt gegen eine Menge verglichen |
| **jede Bereichsabfrage stimmt** | **1891 Abfragen erschöpfend** über alle Grenzenpaare, 398 837 Zeilen verglichen |
| Punktsuche findet alle gleichen Werte | 90 Werte, 500 Zeilen auf 40 Werte, häufigster Wert 18 mal |
| Höhe bleibt unter der Schranke | m=4: 13 ≤ 15 · m=8: 7 ≤ 8 · m=32: 4 ≤ 4 · m=128: 3 ≤ 3 |
| Punktzugriff kostet die Höhe | 516 Zugriffe, höchstens 5 Seiten bei Höhe 4 — 32 mal musste ins nächste Blatt geschaut werden |
| Planer gegen die Wirklichkeit | kein Fehlgriff in 101 Fällen, Trefferschätzung bis **74 %** daneben, Kostenschätzung bis **91 %** |

Geprüft werden dabei alle Bedingungen einzeln benannt, nicht als Sammelurteil „gültig": Sortierung,
Schranken der Teilbäume, Mindest- und Höchstfüllung, Kinderzahl gegen Trennschlüssel, gleiche Tiefe
aller Blätter, vollständige und sortierte Blattkette.

## Was mich das gekostet hat

**Doppelte Schlüssel machen Trennschlüssel mehrdeutig.** Der erste Entwurf hat den Wert direkt als
Schlüssel eingefügt — bei 20 000 Zeilen auf 1000 verschiedene Werte also 20 Zeilen je Wert. Beim
Durchdenken der Blattteilung fiel auf, dass eine Gruppe gleicher Werte über die Blattgrenze läuft
und derselbe Wert dann **links und rechts** desselben Trennschlüssels steht. Ab da hilft keine
Abstiegsregel mehr: bei „Gleichstand nach links" verliert man die Treffer im rechten Blatt, bei
„nach rechts" die im linken. Die Lösung ist die, die echte Datenbanken benutzen: die Zeilennummer
gehört zum Schlüssel. Hier als eine Zahl, `Wert · 2¹⁸ + Zeile` — eindeutig, ordnungsverträglich und
exakt in einem Double (1000 · 262 144 + 200 000 < 2⁵³).

**Die Abstiegsregel und das Ausleihen widersprachen sich.** Auch mit eindeutigen Schlüsseln blieb
ein Fehler: `stelle()` stieg bei Gleichstand nach **links** ab („erster Schlüssel ≥ s"). Leiht ein
unterfülltes Blatt aber vom linken Nachbarn, wird der verschobene Schlüssel zum neuen
Trennschlüssel — und liegt damit **im rechten Kind**. Wer bei Gleichstand links absteigt, sucht ihn
folglich im falschen Teilbaum. Die Symptome:

| Prüfung | vorher | nachher |
|---|---|---|
| Bedingungen nach jedem Löschen | „m=4: Baum nicht leer" nach 1200 Löschungen | kein Mangel |
| Inhalt Schritt für Schritt | bricht nach 51 Schritten | 600 Schritte grün |

Die Korrektur ist eine Zeile — `oberStelle()` statt `stelle()`, also „erster Schlüssel **>** s".
Gefunden hat sie nicht das Nachdenken, sondern die Prüfung nach *jedem einzelnen* Schritt: bei
einer Prüfung nur am Ende wäre der Baum am Ende zufällig wieder gültig gewesen.

**„Ein Punktzugriff kostet die Höhe" ist falsch.** Das war meine eigene Behauptung, und sie ist um
genau eine Seite zu optimistisch: sitzt der gesuchte Wert am **Ende eines Blattes**, muss die
Bereichssuche ins nächste Blatt schauen, um zu wissen, dass dort nichts mehr kommt. Gemessen bei
Ordnung 32 und 50 000 Zeilen: **32 von 516** Zugriffen brauchen diese zusätzliche Seite,
Obergrenze also Höhe + 1. Die Prüfung heißt jetzt so und zählt die Fälle mit.

**Der Planer wählt richtig und schätzt schlecht.** In 101 Trefferquoten von 0 bis 100 % hat der
Planer **nie** den teureren Weg gewählt — die Entscheidung ist robust. Seine Zahlen sind es nicht:
die geschätzte Trefferzahl liegt bis zu **74 %** neben der gemessenen, die geschätzten Kosten des
Indexzugriffs bis zu **91 %** (verstreut) bzw. 20 % (geclustert). Der Grund ist derselbe wie in
echten Systemen: die Formel `min(Seiten, Treffer)` unterschätzt, wie oft dieselbe Seite mehrfach
geholt wird. Wer Planerkosten als Wahrheit liest, liest die falsche Zahl; wer sie als Rangfolge
liest, liegt meist richtig.

**Die Kostenkurve war unlesbar, bis die Achse logarithmisch wurde.** Der Index kostet bei voller
Trefferquote 21 815 Seiten, der Durchlauf konstant 625. Linear aufgetragen ist der Durchlauf eine
Linie auf dem Nullpunkt, und der interessante Kipppunkt bei 3,4 % liegt im Papierknick. Mit
logarithmischer Achse sind beide Kurven und der Schnittpunkt gleichzeitig zu sehen — dieselben
Daten, ein anderes Bild.

**Was das Blatt nicht kann:** kein Seiten-Cache (jede Abfrage beginnt kalt), keine gemeinsamen
Sperren, keine Nebenläufigkeit, kein Bitmap-Index, kein Index-Only-Scan, keine Kosten für Schreiben
oder Sortieren. Die Zeilen sind eine Zahl je Zeile, keine echten Tupel. Es geht um die eine Frage,
die man an einem Nachmittag wirklich beantworten kann: wie viele Seiten kostet dieser Weg.

## Technik

Eine einzelne HTML-Datei. Kein Build, keine Bibliothek, nichts verlässt den Browser.
Canvas 2D, B+-Baum mit Teilen/Ausleihen/Verschmelzen, typisierte Felder, hell und dunkel.

## Verwandt

- [Plotterblätter](https://github.com/ssims437/plotterblaetter) — Wave Function Collapse, Wellengleichung, Physarum, Lenia
- [Redundanz](https://github.com/ssims437/redundanz) — LZ77 und Huffman, Bitkosten je Zeichen
- [Reparatur](https://github.com/ssims437/reparatur) — Reed-Solomon über GF(256)
- [Würfel](https://github.com/ssims437/wuerfel) — Prüfstand für Zufallsgeneratoren
- [Rechenwerk](https://github.com/ssims437/rechenwerk) — ein Rechner aus NAND-Gattern
- [Nachkomma](https://github.com/ssims437/nachkomma) — IEEE 754, exakt ausgeschrieben
- [Zeitsprung](https://github.com/ssims437/zeitsprung) — Zeitzonen und Sommerzeit
- [Gradtage](https://github.com/ssims437/gradtage) — 41 Jahre Heiz- und Kühlgradtage
- [Stimmführung](https://github.com/ssims437/stimmfuehrung) — Akkorde zu MIDI mit geführten Stimmen
- [Verzerrung](https://github.com/ssims437/verzerrung) — Kartenprojektionen und Tissot-Indikatrizen
- [Handschlag](https://github.com/ssims437/handschlag) — elliptische Kurven und der Schlüsseltausch
- [Wegewahl](https://github.com/ssims437/wegewahl) — Dijkstra, A* und der Preis des Suchens
- [Frequenzgang](https://github.com/ssims437/frequenzgang) — FFT, Fensterfunktionen und der Leckeffekt
- [Auszählung](https://github.com/ssims437/auszaehlung) — Wahlverfahren und Sitzverteilung

## Lizenz

MIT
