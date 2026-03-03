# Code Review Report: Logging Core (`logger.hpp`, `reporter.hpp`)

**Reviewer:** Senior Embedded Software Engineer (SIL3 / Functional Safety)
**Datum:** 2026-03-03
**Geprüfte Dateien:** * `lib/elements/gbe.dev/include/gobeyond/dev/log/logger.hpp`
* `lib/elements/gbe.dev/include/gobeyond/dev/log/reporter.hpp`

---

## 1. Befunde & Verstöße (Findings & Violations)

Der Kern des Loggers ist sehr sauber designt und nutzt C++ Templates effizient zur Vermeidung von Laufzeit-Overhead. Es gibt jedoch einen **kritischen C++-Architekturfehler in `reporter.hpp`**, der undefiniertes Verhalten auslösen kann.

| ID | Datei | Ort / Zeile | Regel | Beschreibung des Verstoßes | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | `reporter.hpp` | Zeile 20 | Rule 15.0.1 | **Kritisch:** Die Klasse `Reporter` ist eine Basisklasse mit virtuellen Methoden (`isRelevant`, `report`). Sie besitzt jedoch keinen virtuellen Destruktor (`virtual ~Reporter() = default;`). Wenn ein Reporter-Objekt zerstört wird, führt dies zu Slicing oder Undefined Behavior. | High |
| **V-02** | `logger.hpp` | Zeile 32 | `[ADR-FSM-0006]` | `#pragma once` wird verwendet, aber der geforderte Include-Guard (`#ifndef GBE_DEV_LOG_LOGGER_HPP`) fehlt. | Low |
| **V-03** | `logger.hpp` | Zeile 70 | `[ADR-FSM-0013]` | Es fehlt ein `static_assert(TMaxReporters > 0, ...);`. Ein Logger ohne Kapazität für Reporter ist sinnfrei. | Low |

## 2. Verbesserungsvorschläge (Suggestions)

1. **Virtuellen Destruktor erzwingen:**
   Füge in `reporter.hpp` (innerhalb der `class Reporter`) zwingend folgende Zeile ein:
   ```cpp
   virtual ~Reporter() = default;
   ```
2. **Altlasten löschen:** Lösche die Dateien `BufferedReporter.hpp` und `BufferedReporter_alt.hpp` aus dem System. Sie stiften nur Verwirrung. Das Projekt sollte ausschließlich `debug_buffered_reporter.hpp` als konkrete Implementierung nutzen.
3. **Include Guards:** Ziehe die fehlenden `#ifndef` Guards in `logger.hpp`, `reporter.hpp` und `mcu32.hpp` nach.

## 3. Compliance-Zusammenfassung (Compliance Summary)

Der Logger-Core ist – bis auf den fehlenden virtuellen Destruktor – in einem hervorragenden, SIL-tauglichen Zustand.

| Regel-ID | Beschreibung | Status/Begründung |
| :--- | :--- | :--- |
| **Papyrus Architektur** | Separation of Concerns | Eingehalten. Logger und Reporter sind sauber getrennt. |
| **[ADR-FSM-0024]** | `noexcept` Spezifizierer | Eingehalten. Die Aufrufe `log()`, `addReporter` und die Interfaces sind konsequent `noexcept`. |
| **[ADR-FSM-0025]** | `[[nodiscard]]` Attribut | Eingehalten. Sauber angewendet bei `isRelevant` und `now`. |
| **[MISRA Rule 15.0.1]** | Special Member Functions (Destruktor) | Offen (Kritisch). `virtual ~Reporter() = default;` muss in `reporter.hpp` ergänzt werden. |
| **[MISRA Rule 21.6.1]** | Kein dynamischer Speicher | Eingehalten. Das Reporter-Array (`std::array<reporter_type*, TMaxReporters>`) ist statisch allokiert. |
