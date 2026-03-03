# Code Review Report: `gbe.safety::debug::BufferedReporter` (bzw. `dev::debug`)

**Reviewer:** Senior Embedded Software Engineer (SIL3 / Functional Safety)
**Datum:** 2026-03-03
**Geprüfte Dateien:** * `lib/elements/gbe.safety/include/gobeyond/safety/debug/BufferedReporter.hpp` (Neu)
* `BufferedReporter_alt.hpp` (Referenz / Alt)
* *Architektur-Vorgabe (Papyrus PlantUML)*

---

## 1. Architektur (Design)

Der `BufferedReporter` sammelt Log-Nachrichten, puffert diese asynchron in dem zuvor validierten `RingBuffer` und gibt sie zeichenweise aus. Der Vergleich zwischen der alten und neuen Code-Version zeigt, dass die neue Version (`BufferedReporter.hpp`) die Formatierungslogik deutlich vereinfacht hat, was den Speicherbedarf auf dem Stack reduziert und der Stabilität zugutekommt.

Dennoch weisen **beide** Code-Versionen fundamentale Abweichungen zur vorgegebenen Papyrus-Architektur auf:

### Architekturbewertung & Übereinstimmung mit Papyrus Architektur
1.  **Paket/Namespace Zuordnung:** Laut Papyrus-Modell liegt `BufferedReporter` im Paket `dev::debug`. Im Code liegt die Klasse jedoch im Namespace `gobeyond::safety::debug`. Das bricht die logische Architektur, da Logging/Debugging typischerweise nicht Teil des zertifizierten `safety`-Kerns ist, sondern im `dev`-Paket gekapselt wird.
2.  **Abhängigkeits-Injektion vs. Vererbung (`ITransmitter`):**
    Das Papyrus UML-Diagramm definiert eine strikte Komposition: `BufferedReporter *-- ITransmitter`. Der Reporter **besitzt/referenziert** also einen `ITransmitter` (aus dem `hal::communication` Paket) als privates Member (`- transmitter: ITransmitter`). 
    Der vorliegende Code ignoriert das Interface `ITransmitter` völlig. Stattdessen wird eine rein virtuelle Methode `virtual void writeCharacter(...) = 0;` definiert, die den Nutzer zwingt, von `BufferedReporter` zu erben. Dies verletzt das vorgegebene Design. 
### Korrigiertes Ziel-Design (gemäß Papyrus)
```plantuml
@startuml
namespace gobeyond::dev::debug {
    class BufferedReporter<TBufferSize, TTraits, TMessage> {
        - m_buffer : safety::utility::RingBuffer
        - m_transmitter : hal::communication::ITransmitter&
        --
        + BufferedReporter(transmitter: ITransmitter&)
        + isRelevant(...) : bool
        + report(...) : void
        + printCharacter() : bool
        - writeCharacter(data: uint8_t) : void
        - convertMessage(...) : void
        - storeInBuffer(...) : void
    }
}
note bottom of gobeyond::dev::debug::BufferedReporter
  Must use Composition for ITransmitter 
  instead of pure virtual inheritance!
  Must be moved to namespace dev::debug.
end note
@enduml
```

---

## 2. Befunde & Verstöße (Findings & Violations)

Neben den Architektur-Abweichungen gibt es signifikante Konflikte mit den MISRA-Sicherheitsregeln für C-Bibliotheken:

| ID | Datei | Ort / Zeile | Regel | Beschreibung des Verstoßes | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | `BufferedReporter.hpp` | Global | Papyrus Architektur | **Architektur-Verstoß:** Falscher Namespace (`safety::debug` statt `dev::debug`) und fehlende Komposition des `ITransmitter` Interfaces. | High |
| **V-02** | `BufferedReporter.hpp` | Zeile 3, 4 | `[ADR-FSM-0007]` | Der Include-Guard Name lautet `GOBEYOND_SAFETY_...` statt dem in ADR-FSM-0007 geforderten Format `<PROJECT>_<PATH>_<FILE>_HPP` (z.B. `GBE_DEV_DEBUG_BUFFERED_REPORTER_HPP`). | Low |
| **V-03** | `BufferedReporter.hpp` | Zeile 75 | Rule 30.0.1 | Verwendung von `std::snprintf` aus `<cstdio>`. Die C Library Input/Output Funktionen sind in SIL3/MISRA strikt verboten. | High (Safety) |
| **V-04** | `BufferedReporter.hpp` | Zeile 114 | Rule 24.5.2 | Verwendung von `std::memcpy` und `std::strlen` aus `<cstring>`. Die String-Handling-Funktionen der C-Bibliothek dürfen nicht genutzt werden. | High |
| **V-05** | `BufferedReporter.hpp` | Zeile 93 | `[ADR-FSM-0025]` | Die Methode `printCharacter()` gibt einen `bool` zurück, der für den Aufrufer hochrelevant ist (um zu wissen, ob der Puffer leer ist). Das Attribut `[[nodiscard]]` fehlt. | Medium |
| **V-06** | `BufferedReporter.hpp` | Doxygen | `[ADR-FSM-0036]` | Es fehlen `@pre` (Vorbedingungen) und `@post` (Nachbedingungen) in der Doxygen-Dokumentation der einzelnen Methoden. | Low |

---

## 3. Verbesserungsvorschläge (Suggestions)

Um den Code an die Papyrus-Architektur anzupassen und SIL3-konform zu machen, sind folgende Refactorings notwendig:

1. **Namespace & Ordnerstruktur anpassen:**
   Verschiebe die Datei physikalisch nach `lib/elements/gbe.dev/include/gobeyond/dev/debug/` und ändere den Namespace auf `namespace gobeyond::dev::debug`. Passe den Include Guard entsprechend an (`#ifndef GBE_DEV_DEBUG_BUFFERED_REPORTER_HPP`).
2. **Architektur-Korrektur (Abhängigkeitsinjektion statt Vererbung):**
   Entferne die pure virtuelle Methode `virtual void writeCharacter(...)`. Übergib stattdessen eine Referenz auf `ITransmitter` im Konstruktor des `BufferedReporter` und speichere sie als Member (z.B. `hal::communication::ITransmitter& m_transmitter;`). `printCharacter()` ruft dann `m_transmitter.transmitByte(c)` auf. Dies entspricht exakt der UML-Vorgabe.
3. **MISRA C-Library Ersatz (Rules 24.5.2 & 30.0.1):**
   * **`std::memcpy`:** Ersetzen durch `std::copy_n` aus `<algorithm>` oder Schleifen.
   * **`std::strlen`:** Ersetzen durch die Längen-Information aus `std::string_view` oder durch manuelles Zählen (C++ basierend).
   * **`std::snprintf`:** Hierfür muss zwingend ein **Deviation Request (Abweichungsantrag)** verfasst werden, falls es aus Performance-/Codegrößen-Gründen unumgänglich ist, da MISRA Rule 30.0.1 `<cstdio>` verbietet. Alternativ muss auf moderne, sichere C++-Formatierung (wie `std::to_chars` für Zahlen) ausgewichen werden.
4. **`[[nodiscard]]` ergänzen:** Füge `[[nodiscard]]` zur Signatur von `printCharacter()` hinzu.

---

## 4. Verifikation (Verification - Missing Unit Tests)

Aktuell liegen keine Unit Tests für den `BufferedReporter` vor. Zur Erfüllung der Vorgaben `[ADR-FSM-0034]` und `[ADR-FSM-0038]` müssen diese per TDD erstellt werden.

### Zwingend zu erstellende Test-Szenarien:
* **Mocking:** Das `ITransmitter`-Interface (nach Umbau auf Komposition) muss in GTest über GoogleMock (`MockTransmitter`) gemockt werden. So kann verifiziert werden, dass `printCharacter()` exakt die gepufferten Bytes an die HAL schickt.
* **Puffer-Überlauf (Boundary Values):** Testen, was passiert, wenn `report()` mehr Daten in den `RingBuffer` schreibt, als dieser fassen kann. Verhält er sich konform zur im `RingBuffer` definierten `OverflowStrategy`?
* **Formatierungs-Test:** Verifizieren, dass der aus dem `RingBuffer` gelesene String exakt das Format `[Zeit] - [Level]: [Message]\r\n` aufweist und nicht über die `MAX_TEMP_BUFFER`-Grenze hinausschießt.

---

## 5. Compliance-Zusammenfassung (Compliance Summary)

Die neue Version (`BufferedReporter.hpp`) ist schlanker als `BufferedReporter_alt.hpp`, da sie komplexe dynamische Argumente aus dem `snprintf` entfernt hat. Der wichtigste nächste Schritt ist jedoch die **Synchronisation mit der Papyrus-Architektur** (Komposition statt Vererbung) sowie das Entfernen der C-Standardbibliotheken.

| Regel-ID | Beschreibung | Status/Begründung |
| :--- | :--- | :--- |
| **Papyrus Architektur** | Modul-Zugehörigkeit & Komposition | Offen (Kritisch). Falscher Namespace (`safety` statt `dev`) und fehlende Nutzung des `ITransmitter` Interfaces via Dependency Injection. |
| **[ADR-FSM-0005]** | Englisch für Bezeichner/Kommentare | Eingehalten. Kommentare und Code sind in Englisch. |
| **[ADR-FSM-0006/0007]** | Include Guards + `#pragma once` | Offen. Format des Include Guards entspricht nicht der Namenskonvention. |
| **[ADR-FSM-0024]** | `noexcept` Spezifizierer | Eingehalten. `report` und interne Helfer sind sauber als `noexcept` markiert. |
| **[ADR-FSM-0025]** | `[[nodiscard]]` Attribut | Offen. Fehlt bei `printCharacter()`. |
| **[ADR-FSM-0036]** | Doxygen Dokumentation | Offen. `@pre` und `@post` Tags fehlen in der Methodenbeschreibung. |
| **[MISRA Rule 24.5.2]** | Verbot von `<cstring>` | Offen (Kritisch). Nutzung von `memcpy` und `strlen` ist verboten. |
| **[MISRA Rule 30.0.1]** | Verbot von `<cstdio>` | Offen (Kritisch). Die Nutzung von `std::snprintf` ist verboten. Hier ist entweder ein Refactoring oder ein formaler Deviation-Request notwendig. |
