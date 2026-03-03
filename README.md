# Code Review Report: `gbe.utility::BitMask`

**Reviewer:** Senior Embedded Software Engineer (SIL3 / Functional Safety)
**Datum:** 2026-03-03
**Geprüfte Dateien:** * `lib/elements/gbe.utility/include/gobeyond/utility/bitmask.hpp`
* `tests/bitmask.cpp`

---

## 1. Architektur (Design)

Die `BitMask`-Komponente ist ein grundlegendes Utility, das typsichere bitweise Operationen für `enum class`-Typen ermöglicht. Das Design trennt sauber den internen Zustand (`m_value`) von den öffentlichen Zugriffsstrukturen (`Enable`, `Disable`, etc.).

### Architekturbewertung
* **Datenkapselung:** Sehr gut. Die Trennung zwischen `struct` (für Wrapper-Typen wie `Enable`, die laut Vorgaben nur öffentliche Member haben dürfen) und `class` (für die eigentliche `BitMask` mit privatem `m_value`) ist exakt nach Vorschrift implementiert.
* **Symmetrische Operatoren:** Die Implementierung der binären Operatoren (`|`, `&`, `==`, `!=`) als `friend inline` ("Hidden Friends") ist architektonisch hervorragend und entspricht Rule 16.6.1.

### UML-Klassendiagramm
```plantuml
@startuml
namespace gobeyond::utility {
    class BitMask<TEnum> <<template>> {
        - m_value : underlying_type
        + BitMask()
        + BitMask(value: enum_type)
        + enable(value: enum_type) : void
        + disable(value: enum_type) : void
        + isEnabled(value: enum_type) : bool
        + isDisabled(value: enum_type) : bool
    }

    struct Enable {
        + value : enum_type
    }
    struct Disable {
        + value : enum_type
    }
    struct Enabled {
        + value : enum_type
    }
    struct Disabled {
        + value : enum_type
    }
    
    BitMask +-- Enable
    BitMask +-- Disable
    BitMask +-- Enabled
    BitMask +-- Disabled
}

note bottom of gobeyond::utility
  Global SFINAE operators operator| and operator&
  must be moved inside this namespace.
end note
@enduml
```

---

## 2. Befunde & Verstöße (Findings & Violations)

Bei der statischen Code-Analyse anhand der SIL3-Regularien sind folgende Abweichungen aufgefallen:

| ID | Datei | Ort / Zeile | Regel | Beschreibung des Verstoßes | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | `bitmask.hpp` | Zeile 1 | `[ADR-FSM-0006]`, `[ADR-FSM-0007]` | Es wird `#pragma once` verwendet, aber der geforderte Fallback-Include-Guard (`#ifndef GBE_UTILITY_BITMASK_HPP`) fehlt. | High |
| **V-02** | `bitmask.hpp` | Zeile 151-160 | Rule 6.0.3 | Die SFINAE-Templates für `operator\|` und `operator&` befinden sich im globalen Namensraum. Dies "verschmutzt" den globalen Namensraum für alle Enums im Projekt. | High |
| **V-03** | `bitmask.hpp` | Zeile 68, 73, 107-142 | `[ADR-FSM-0025]` | Alle `isEnabled`, `isDisabled` Methoden sowie die überladenen Operatoren (`==`, `!=`, `\|`, `&`) liefern ein Ergebnis zurück, das der Hauptzweck der Funktion ist. Das Attribut `[[nodiscard]]` fehlt. | Medium |
| **V-04** | `bitmask.hpp` | Zeile 17-42 | `[ADR-FSM-0024]` | Die Konstruktoren der structs (`Enable`, `Disable`, etc.) sowie der Basis-Konstruktor von `BitMask` haben keine `noexcept`-Spezifizierung. | Medium |
| **V-05** | `bitmask.hpp` | Zeile 57-65 | Rule 7.0.4 | Bitweise Operatoren (`&=`, `\|=`) dürfen nur auf Typen der Kategorie "unsigned" angewendet werden. Es fehlt ein `static_assert`, das sicherstellt, dass `TEnum` keinen *signed* underlying type hat. | High (Safety) |
| **V-06** | `bitmask.hpp` | Zeile 78-99 | Rule 15.0.2 | Zuweisungsoperatoren (`operator=`) sollten ref-qualifiziert sein (mit `&`), damit keine Zuweisungen an temporäre R-Values stattfinden können (Vermeidung von Dangling References). | Low |
| **V-07** | `bitmask.hpp` | Global | `[ADR-FSM-0036]` | Fehlende Doxygen-Dokumentation (`@brief`, `@param`, `@return`) sowie das für die Safety-Analyse zwingend notwendige `@safety` Tag. | Medium |

---

## 3. Verbesserungsvorschläge (Suggestions)

Um die identifizierten Verstöße zu beheben und den Code SIL3-konform zu machen, müssen folgende Änderungen vorgenommen werden:

1. **Include Guards einbauen:** Ergänze `#ifndef GBE_UTILITY_BITMASK_HPP` direkt unter `#pragma once` und schließe die Datei mit `#endif // GBE_UTILITY_BITMASK_HPP`.
2. **Namensraum-Kapselung:** Verschiebe die beiden Templates `operator|` und `operator&` am Ende der Datei *in* den Block von `namespace gobeyond::utility`.
3. **Compile-Time Safety (`static_assert`):** Füge am Anfang der Klasse `BitMask` folgende Checks ein, um Laufzeitfehler zu verhindern:
   * `static_assert(std::is_enum_v<TEnum>, "...");`
   * `static_assert(std::is_unsigned_v<std::underlying_type_t<TEnum>>, "...");`
4. **Attribute hinzufügen:** Versehe alle Member-Funktionen der Klasse, Konstruktoren und Operatoren, sofern zutreffend, konsequent mit `constexpr inline ... noexcept`. Bei reinen Gettern/Operatoren zusätzlich `[[nodiscard]]` voranstellen.
5. **Ref-Qualifier für Assignments:** Ändere die Signaturen der Zuweisungen zu `inline BitMask& operator=(...) & noexcept`.

---

## 4. Verifikation (Verification - `bitmask.cpp`)

Die Unit-Tests in `bitmask.cpp` nutzen das Google Test (GTest) Framework.

### Positive Befunde
* Die öffentliche Schnittstelle der Klasse wird gut abgedeckt, was der Regel `[ADR-FSM-0038]` entspricht. 
* Das Test-Enum `LogLocation` verwendet richtigerweise explizite Zweierpotenzen und besitzt mit `std::uint32_t` einen expliziten, *unsigned* underlying type.

### Lücken & Vorschläge für die Tests
* **Compile-Time Checks:** Es fehlt ein Test, der verifiziert, dass die Klasse das Kompilieren mit einem *signed* Enum blockiert (dies kann im CMake-Setup über spezielle GTest Compile-Failure-Tests sichergestellt werden).
* **Grenzwerte (Boundary Values):** Ein Test-Szenario, das überprüft, was passiert, wenn alle Bits gesetzt sind (z.B. `0xFFFFFFFF`) oder `NONE`, fehlt zum Teil. Zwar ist `ALL` definiert, es wird aber nicht extensiv auf Overflows oder Clipping getestet.
* **Doxygen:** Auch die Testdatei sollte einen kurzen Datei-Header enthalten, der dokumentiert, welche Funktionalitäten hier verifiziert werden.

---

## 5. Compliance-Zusammenfassung (Compliance Summary)

Die Datei befindet sich prinzipiell in einem sehr guten, modernen C++-Zustand. Die strikte Typsicherheit durch SFINAE (`std::enable_if_t`) und die strikte Unterscheidung von `struct` vs `class` zeigen ein tiefes Verständnis für die projektinternen Regeln.

Die primären Lücken betreffen die formelle Absicherung (fehlende Doxygen-Kommentare, fehlendes `[[nodiscard]]` und `noexcept` auf einigen Pfaden) sowie die fehlende `unsigned`-Validierung, die für die Hardware-Ebene bei Bit-Operatoren sicherheitskritisch ist (MISRA Rule 7.0.4).

| Regel-ID | Beschreibung | Status/Begründung |
| :--- | :--- | :--- |
| **[ADR-FSM-0005]** | Englisch für Bezeichner und Kommentare | Eingehalten. Doxygen-Kommentare müssen noch ergänzt werden. |
| **[ADR-FSM-0006/0007]** | Include Guards + `#pragma once` | Offen. `#ifndef GBE_UTILITY_BITMASK_HPP` muss um das File gelegt werden. |
| **[ADR-FSM-0013]** | Nutzung von `static_assert` | Offen. Muss ergänzt werden, um sicherzustellen, dass das Enum existiert und **unsigned** ist. |
| **[ADR-FSM-0024]** | `noexcept` Spezifizierer in structs/class | Offen. Muss für Konstruktoren und Getter nachgezogen werden. |
| **[ADR-FSM-0025]** | `[[nodiscard]]` Attribut | Offen. Jede Abfrage (z.B. `isEnabled` und `operator\|`) erfordert eine Reaktion des Callers. |
| **[ADR-FSM-0027]** | `constexpr` Funktionen | Offen. Die Klasse muss maximal zur Compile-Zeit auswertbar gemacht werden. |
| **[ADR-FSM-0035]** | `struct` vs. `class` Trennung | Eingehalten. `Enable`, `Disable` etc. sind `structs` mit öffentlichen Membern, `BitMask` als Klasse kapselt den Wert `m_value`. |
| **[MISRA Rule 6.0.3]** | Keine Deklarationen im globalen Namensraum | Offen. SFINAE-Operatoren müssen in `gobeyond::utility` gezogen werden. |
| **[MISRA Rule 7.0.4]** | Bitweise Operatoren nur auf `unsigned` | Offen. `static_assert(std::is_unsigned_v<...>)` muss ein signed-underlying-type strikt ablehnen. |
| **[MISRA Rule 15.0.2]** | Signatur der Copy/Move Assignment Ops | Offen. `operator=` muss mit `&` lvalue-qualifiziert werden. |
