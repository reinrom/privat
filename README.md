# Code Review Report: `gbe.bsp::stm32h753` (Communication Adapters)

**Reviewer:** Senior Embedded Software Engineer (SIL3 / Functional Safety)
**Datum:** 2026-03-04
**Geprüfte Dateien:** * `stm32_uart_dma.hpp` (Asynchroner IPC-Adapter)
* `stm32-uart.hpp` (Synchroner Konfigurations-Adapter)
* `tests/test-stm32-uart-dma.cpp`, `test-stm32-uart.cpp`, `test-uart.cpp`, `test-zm-tester-platform.cpp`

---

## 1. Architektur (Design)

Diese Ebene verbindet die abstrakten Verträge (Interfaces) mit der harten STM32-Realität. Die Umsetzung der Papyrus-Architektur in C++ ist hier besonders gut gelungen.

### Architekturbewertung
* **IPK Querkommunikation (`Stm32UartDma`):** Die Klasse implementiert exakt das UML-Modell (`UartWithDMA`). Sie trennt geschickt Sende- und Empfangspfade intern über zwei separate Status-Flags (`m_sending`, `m_receiving`) und kapselt die HAL-Interrupt-Callbacks (`onTxComplete`, `onRxComplete`).
* **SIL3 Cache-Coherency:** Der Cortex-M7 (STM32H7) hat einen L1-Daten-Cache. Wenn DMA genutzt wird, entsteht Dateninkonsistenz. Der Entwickler hat hier hervorragend mitgedacht und `SCB_CleanDCache_by_Addr` (vor dem Senden) sowie `SCB_InvalidateDCache_by_Addr` (nach dem Empfangen) eingebaut. Dies ist sicherheitskritisch und exzellent gelöst!
* **Legacy vs. Modern:** `Stm32UartDma` implementiert aktuell noch vier Interfaces (die alten `IConcurrent*` und die neuen UML-Interfaces). Das ist als Übergangslösung (Adapter-Pattern) völlig legitim.

---

## 2. Befunde & Verstöße (Findings & Violations)

Trotz des starken Designs zwingt die ST-Microelectronics HAL uns hier zu einigen unsauberen C++-Konstrukten, die wir bezüglich MISRA formal absichern müssen.

| ID | Datei | Ort / Zeile | Regel | Beschreibung des Verstoßes | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **V-01** | `stm32-uart.hpp`, `stm32_uart_dma.hpp` | div. | Rule 8.2.3 | **Kritisch:** Die HAL-Funktionen (`HAL_UART_Transmit`) erwarten `uint8_t*`, obwohl wir logisch einen `const uint8_t*` übergeben (Puffer wird nur gelesen). Der Entwickler hat dies mit `const_cast` gelöst. MISRA verbietet `const_cast`. Hierfür ist ein formaler Deviation Request (Abweichungsantrag) zwingend erforderlich, der wie im Code als Kommentar bereits vorbereitet, begründet werden muss. | High |
| **V-02** | `stm32_uart_dma.hpp` | Zeile 353, 354 | Safety / Concurrency | Die Variablen `m_sending` und `m_receiving` sind als `volatile bool` deklariert. In modernem C++ (seit C++11) ist `volatile` **kein** Mechanismus für Thread-Sicherheit oder ISR-Synchronisation. Dies kann zu Undefined Behavior führen. | Medium |
| **V-03** | `stm32-uart.hpp` | Zeile 162 | Safety / Logik | Die Methode `receiveByte()` gibt `0U` zurück, wenn der Handle ein `nullptr` ist oder ein Timeout auftritt. Das ist extrem gefährlich, da `0x00` ein gültiges Datenbyte sein kann. | High |
| **V-04** | `Alle Test-Dateien` | Global | `[ADR-FSM-0017]` | In den Tests wird `uint8_t` oder `uint16_t` verwendet. Gemäß Sprachuntermenge muss es zwingend `std::uint8_t` heißen. | Low |
| **V-05** | `Alle *.hpp Dateien`| Doxygen | `[ADR-FSM-0005]` | Kommentare sind weiterhin auf Deutsch. | Medium |

---

## 3. Verbesserungsvorschläge (Suggestions)

Um die letzten Risiken in der Hardware-Anbindung abzustellen, schlage ich Folgendes vor:

1. **ISR-Synchronisation (V-02 beheben):**
   Ersetze `volatile bool` durch `std::atomic<bool>`. Dies garantiert auf C++-Ebene atomare Zugriffe ohne Memory-Reordering durch den Compiler.
   ```cpp
   #include <atomic>
   // ...
   std::atomic<bool> m_sending{false};
   std::atomic<bool> m_receiving{false};
   ```
2. **Sicheres `receiveByte` (V-03 beheben):**
   Da das Interface `ITransceiver` einen fixen Return-Value vorgibt, gibt es keine Fehler-Rückgabe. Da dieses Interface synchron blockiert, sollte die Methode aus dem Interface idealerweise in `[[nodiscard]] virtual std::optional<std::uint8_t> receiveByte() = 0;` geändert werden. Falls das Interface nicht geändert werden darf, MUSS der Aufrufer den Status des UARTs vorab abfragen können (was aktuell nicht implementiert ist).
3. **MISRA Deviation für `const_cast` (V-01):**
   Erstellt ein offizielles Dokument im Projekt: "Deviation von MISRA 8.2.3: ST HAL APIs für Transmit-Funktionen sind historisch bedingt nicht const-correct. Da die Hardware-Register in diesem Modus nur Lesezugriffe auf den RAM ausführen, ist der `const_cast` an dieser Grenze sicherheitstechnisch unbedenklich."
4. **Namespace in Tests (V-04 beheben):**
   Führe ein Suchen & Ersetzen in den Test-Dateien durch: `uint8_t` -> `std::uint8_t`.

---

## 4. Verifikation (Test Coverage)

Die Unit-Tests sind ein absolutes Highlight dieses Päckchens. Der Entwickler hat C-Funktionen der HAL (z.B. `HAL_UART_Transmit`) in den Tests überschrieben (gemockt), um den Zustand des Adapters zu spyen. 

### Positive Befunde
* **Contract-Testing:** `test-zm-tester-platform.cpp` verifiziert erfolgreich, dass das Architektur-Konzept der Rollenverteilung (`ITransmitter` und `IReceiver` zeigen auf dieselbe Instanz) funktioniert.
* **Vollständige Pfad-Abdeckung:** `test-stm32-uart-dma.cpp` deckt gezielt Fehlerfälle (z.B. Nullpointer-Übergaben, leere Puffer) ab.
* **Robustheit:** Die Tests verifizieren, dass Aufrufe von `startSending` bei bereits laufendem DMA korrekt mit `AsyncStatus::Busy` abgewiesen werden (`[ADR-FSM-0034]`).

### Lücken & Vorschläge für die Tests
* Der Test-Coverage Report im Code markiert, dass der Zweig für den DMA-Counter in `getRemainingBytesToSend()` schwer testbar ist. Eine Möglichkeit wäre, die Dummy-C-Funktion `__HAL_DMA_GET_COUNTER` in der Test-Umgebung so umzuschreiben, dass sie einen globalen Mock-Wert zurückgibt, den der Test vorher setzt.

---

## 5. Compliance-Zusammenfassung (Compliance Summary)

Die Hardware-Adapter sind hochprofessionell geschrieben. Die Cache-Verwaltung für den DMA zeigt tiefes Verständnis der Cortex-M7 Architektur. Mit dem Umstieg auf `std::atomic` und der Dokumentation der `const_cast` Deviation ist der Code uneingeschränkt SIL3-tauglich.

| Regel-ID | Beschreibung | Status/Begründung |
| :--- | :--- | :--- |
| **Papyrus Architektur** | UML Abdeckung | Exzellent. Die `ZmTesterPlatform` verknüpft die Instanzen exakt nach Vorgabe. |
| **[ADR-FSM-0005]** | Englisch für Bezeichner/Kommentare | Offen. Doxygen ist aktuell auf Deutsch. |
| **[ADR-FSM-0017]** | Triviale Datentypen / Fixed Width | Fast eingehalten. In den Tests fehlt vereinzelt der `std::` Namespace. |
| **[ADR-FSM-0022]** | Early Exit | Eingehalten. Nullpointer und leere Arrays werden sofort abgewiesen. |
| **[ADR-FSM-0024]** | `noexcept` Spezifizierer | Eingehalten. Alle kritischen HAL-Methoden sind konsequent `noexcept`. |
| **[MISRA Rule 8.2.3]** | Entfernen von `const` Qualifizierern | Abweichung (Deviation) zwingend nötig. Die ST-HAL-API erfordert einen `const_cast`. |
| **Safety / Concurrency** | Race Conditions | Offen (Kritisch). `volatile bool` für ISR-Status-Flags muss durch `std::atomic<bool>` ersetzt werden. |
