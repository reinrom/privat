# Code Review Report: `gbe.utility::Version`

**Reviewer:** Senior Embedded Software Engineer (SIL3 / Functional Safety)
**Datum:** 2026-03-03
**Geprüfte Dateien:** * `lib/elements/gbe.utility/include/gobeyond/utility/version.hpp`
* `tests/version.cpp`

---

## 1. Architektur (Design)

Die `Version`-Komponente ist eine simple Datenstruktur zur Verwaltung von Semantic Versioning (Major, Minor, Patch). 

### Architekturbewertung & Übereinstimmung mit Papyrus Architektur
* **Struct vs. Class (Papyrus-Modell):** Die Umsetzung als `struct` stimmt exakt mit der im Papyrus-Architekturmodell hinterlegten Regelung `[ADR-FSM-0035]` überein. Es handelt sich um eine reine "Data Collection", bei der jeder Zustand zulässig ist. Dementsprechend sind alle Data Member `public` und es gibt keine `non-static Member Functions` (außer Konstruktoren und Typkonvertierungen).
* **Symmetrische Operatoren:** Die Vergleichsoperatoren (`==`, `!=`, `<`, `>`, `<=`, `>=`) sind korrekt als non-member `friend constexpr` ("Hidden Friends") implementiert. Dies entspricht vollumfänglich Rule 16.6.1 und verhindert implizite Konvertierungsfehler bei asymmetrischen Aufrufen.
* **Typkonvertierung:** Die explizite Konvertierung via `explicit operator std::uint32_t()` schützt vor unbeabsichtigten Typumwandlungen und entspricht `[ADR-FSM-0031]`.

### UML-Klassendiagramm
```plantuml
@startuml
namespace gobeyond::utility {
    struct Version {
        + major : std::uint8_t
        + minor : std::uint8_t
        + patch : std::uint8_t
        --
        + <<constexpr>> Version(major: uint8_t, minor: uint8_t, patch: uint8_t)
        + <<constexpr>> operator uint32_t() const
    }
}
note right of gobeyond::utility::Version
  Matches Papyrus Architecture:
  Pure struct, public members,
  hidden friend operators for comparisons.
end note
@enduml
```

---

## 2. Befunde & Verstöße (Findings & Violations)

Der Code ist sehr sauber und erfüllt bereits einen Großteil der Anforderungen. Bei der strikten SIL3- und MISRA-Prüfung sind jedoch folgende Abweichungen aufgefallen, wobei besonders **V-03** sicherheitskritisch (Safety) ist:

| ID | Datei | Ort / Zeile | Regel | Beschreibung des Verstoßes | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | `version.hpp` | Zeile 1 | `[ADR-FSM-0006]`, `[ADR-FSM-0007]` | Es wird `#pragma once` verwendet, aber der geforderte Fallback-Include-Guard (`#ifndef GBE_UTILITY_VERSION_HPP`) fehlt. | High |
| **V-02** | `version.hpp` | Zeile 101 | `[ADR-FSM-0025]` | Dem `operator<` fehlt das `[[nodiscard]]`-Attribut. Alle anderen relationalen Operatoren in dieser Datei besitzen es korrekterweise. | Low |
| **V-03** | `version.hpp` | Zeile 48 | Rule 7.0.5 | Die Berechnung `(major * 100 * 100) + (minor * 100) + patch` vermischt `std::uint8_t` (unsigned) mit dem Literal `100` (signed `int`). Dies führt zu einer impliziten "Integral Promotion" des unsigned-Werts in einen signed-Wert, was Rule 7.0.5 strikt verbietet. Auf 16-Bit Architekturen würde dies bei großen Versionsnummern zudem zu einem Signed Integer Overflow (Undefined Behavior) führen. | High (Safety) |
| **V-04** | `version.hpp` | Zeile 15 | `[ADR-FSM-0036]` | Das `@author` Tag im initialen Doxygen-Block ist leer. Zudem fehlt das zwingend erforderliche `@safety`-Tag, um zu erklären, dass die Operationen konstante Laufzeiten haben. | Medium |

---

## 3. Verbesserungsvorschläge (Suggestions)

Um die identifizierten Verstöße zu beheben und den Code SIL3-konform zu machen, schlage ich folgende Änderungen vor:

1. **Include Guards einbauen:** Ergänze `#ifndef GBE_UTILITY_VERSION_HPP` und `#define GBE_UTILITY_VERSION_HPP` direkt unter `#pragma once` und schließe die Datei mit `#endif // GBE_UTILITY_VERSION_HPP`.
2. **Korrektur der Typ-Promotion (MISRA 7.0.5):**
   Damit keine implizite Konvertierung von `unsigned` zu `signed` stattfindet und kein Überlauf entsteht, müssen die Variablen vor der Multiplikation auf den Zieldatentyp `std::uint32_t` gecastet werden und `unsigned` Literale (`100U`) genutzt werden:
   ```cpp
   return (static_cast<std::uint32_t>(major) * 10000U) + 
          (static_cast<std::uint32_t>(minor) * 100U) + 
          static_cast<std::uint32_t>(patch);
   ```
3. **Ergänzung von `[[nodiscard]]`:**
   Füge in Zeile 101 das Attribut vor dem Rückgabetyp ein: `[[nodiscard]] friend constexpr bool operator<...`
4. **Doxygen-Kommentare vervollständigen:**
   Fülle das `@author` Tag aus (z.B. `@author t.schwarzinger@dina.de`) und ergänze ein `@safety`-Tag im Klassenkommentar (z.B. `@safety All methods are constexpr and noexcept, ensuring deterministic execution time and memory safety.`).

---

## 4. Verifikation (Verification - `version.cpp`)

Die Unit-Tests nutzen das Google Test (GTest) Framework und testen die Komponente gemäß `[ADR-FSM-0038]`.

### Positive Befunde
* Alle geforderten public-Schnittstellen (Konstruktor, Konvertierung, Operatoren) werden durch entsprechende Test Cases aufgerufen und verifiziert.
* Die Tests sind klar strukturiert.

### Lücken & Vorschläge für die Tests
* **Grenzwerte (Boundary Values):** Ein Test-Szenario, das die maximal zulässigen Werte für `std::uint8_t` prüft (z.B. `Version{255, 255, 255}`), fehlt komplett. Dieser Test ist essenziell, um zu beweisen, dass die MISRA-konforme Berechnung (V-03) korrekt funktioniert und keinen Overflow in einen negativen Bereich erzeugt.
* **Typ-Sicherheit der Konvertierung:** Ein Test, der verifiziert, dass die Umwandlung `static_cast<std::uint32_t>(v)` exakt funktioniert (insbesondere für Werte wie `minor = 0`, was in `VersionTest.Conversion` bereits angeschnitten, aber nicht tiefgreifend geprüft ist).

---

## 5. Compliance-Zusammenfassung (Compliance Summary)

Die Datei passt sich hervorragend in das vorgesehene Architekturmodell (Papyrus) ein und nutzt moderne C++17 Konzepte sicherheitsbewusst. Mit der Behebung des MISRA Integral-Promotion-Fehlers (V-03) erreicht die Klasse SIL3-Niveau.

| Regel-ID | Beschreibung | Status/Begründung |
| :--- | :--- | :--- |
| **[ADR-FSM-0005]** | Englisch für Bezeichner/Kommentare | Eingehalten. |
| **[ADR-FSM-0006/0007]** | Include Guards + `#pragma once` | Offen. `#ifndef` Fallback muss nachgerüstet werden. |
| **[ADR-FSM-0017]** | Triviale Datentypen / Fixed Width | Eingehalten. `std::uint8_t` und `std::uint32_t` werden konsequent genutzt. |
| **[ADR-FSM-0024]** | `noexcept` Spezifizierer | Eingehalten. Operatoren werfen keine Exceptions. |
| **[ADR-FSM-0025]** | `[[nodiscard]]` Attribut | Offen. Fehlt nur bei `operator<`. |
| **[ADR-FSM-0027]** | `constexpr` Funktionen | Eingehalten. Die Klasse ist voll compile-time fähig. |
| **[ADR-FSM-0031]** | Implizite Konvertierungen | Eingehalten. Konvertierung nach `uint32_t` ist `explicit`. |
| **[ADR-FSM-0035]** | `struct` vs. `class` | Eingehalten. Papyrus Architektur-Vorgaben für Structs werden exakt erfüllt. |
| **[MISRA Rule 7.0.5]** | Keine sign/unsigned Wechsel bei Promotion | Offen (Kritisch). Die Berechnung der Version führt zu Typ-Promotion. Muss durch explizite Casts und unsigned-Literale behoben werden. |
| **[MISRA Rule 16.6.1]** | Symmetrische Operatoren non-member | Eingehalten. Operatoren sind als `friend` deklariert. |
