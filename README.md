# Branch: feature/basic_labormuster

## Management Summary

Dieser Branch dokumentiert die erfolgreiche architektonische Zusammenführung der hardwarenahen Initialisierung (ursprüngliche CubeMX-Firmware) mit der modularen, SIL3-orientierten C++17-Architektur (LaborMuster). 

Ziel dieser Integration ("Variante 3") war es, die exakten physikalischen Hardware-Gegebenheiten der Master- und Slave-Platinen beizubehalten, diese jedoch in eine vollständig entkoppelte, testbare und schnittstellenbasierte C++-Struktur zu überführen. Das Ergebnis ist ein lauffähiges System, das eine asynchrone Inter-Prozessor-Kommunikation (IPK) via DMA und ein non-blocking Logging-System über Ringpuffer auf zwei STM32H7-Controllern demonstriert.

---

## Architektonische Meilensteine

### 1. Entkopplung von Hardware-Initialisierung und Applikationslogik
Die direkte Abhängigkeit der Applikation von STM32-HAL-Handlen wurde aufgelöst. 
* Die von STM32CubeMX generierten Initialisierungsfunktionen (inkl. Pin-Routings und MSP-Konfigurationen) wurden strikt in dedizierte Dateien ausgelagert (`board.master.cpp` und `board.slave.cpp`).
* Die zentrale `stm32h7xx_hal_msp.c` wurde aufgelöst, da Master und Slave unterschiedliche Pin-Belegungen für dieselben logischen Schnittstellen (z.B. SPI, UART) nutzen. Die MSP-Initialisierung erfolgt nun zielspezifisch in den jeweiligen Board-Dateien.

### 2. CMake-basiertes Multi-Target Build-System
Das Build-System wurde so konfiguriert, dass aus einer gemeinsamen Code-Basis auf Knopfdruck zwei getrennte Binaries (`app_master.elf` und `app_slave.elf`) generiert werden.
* Über die CMake-Funktion `add_safety_firmware_target` wird dem Compiler exakt die Hardware-Datei übergeben, die für das jeweilige Target benötigt wird.
* Globale Präprozessor-Makros (`-DIPK_MASTER`, `-DIPK_SLAVE`) werden automatisch durch CMake injiziert.

### 3. Ablösung des Legacy-Loggers (`debug.cpp`)
Die veraltete, synchron blockierende Logging-Implementierung (`debug.cpp`), welche C-Standardfunktionen wie `std::snprintf` und direkte HAL-Aufrufe (`HAL_UART_Transmit`) nutzte, wurde vollständig entfernt.
* **Neues Konzept:** Es wird der Papyrus-Architektur-konforme `BufferedReporter` eingesetzt.
* Dieser nutzt einen asynchronen Ringpuffer und kommuniziert ausschließlich über das abstrakte `ITransmitter`-Interface. Dies verhindert CPU-Blockaden durch langsame serielle Schnittstellen und erhöht die Safety-Konformität.

### 4. Asynchrone IPK-Querkommunikation (UART-DMA)
Die Kommunikation zwischen Master und Slave wurde über die Schnittstellen `IConcurrentTransmitter` und `IConcurrentReceiver` implementiert.
* Die Klasse `Stm32UartDma` kapselt die Hardware-Details und die Interrupt-Callbacks.
* **Memory Protection (MPU):** Um Datenkorruption durch den L1-Daten-Cache des Cortex-M7 zu verhindern, wurde eine dedizierte MPU-Region für das D2-SRAM (0x30000000) konfiguriert (`MPU_ACCESS_NOT_CACHEABLE`). Die DMA-Puffer liegen in diesem geschützten Bereich, zusätzlich abgesichert durch explizite Cache-Maintenance-Operationen (`SCB_CleanDCache`, `SCB_InvalidateDCache`).

---

## Umgang mit CubeMX (`.ioc` Dateien)

Die STM32CubeMX-Projektdateien (`master.ioc`, `slave.ioc`) verbleiben als Konfigurations- und Dokumentationswerkzeuge im Projekt. CubeMX wird jedoch **nicht** mehr als Build-Master oder zentraler Code-Generator für die gesamte Projektstruktur verwendet. 
Bei zukünftigen Hardware-Änderungen (z.B. neuen Pins) wird der Code temporär via CubeMX generiert und die relevanten Abschnitte manuell in die Dateien `board.master.cpp` bzw. `board.slave.cpp` überführt. Dies schützt die C++-Architektur vor destruktiven Überschreibungen durch den Generator.

---

## Ausstehende Optimierungen (Next Steps)

Das aktuelle System dient als voll funktionsfähige Baseline. Für die finale Produktionsreife stehen folgende Refactorings an:

1. **Auflösung der Präprozessor-Branches in der Applikationslogik:**
   Gemäß Architekturrichtlinie *[ADR-FSM-0009]* dürfen `#ifdef`-Verzweigungen nicht zur Steuerung der Applikationslogik verwendet werden. Die in der `main.cpp` verbliebenen `#ifdef IPK_MASTER` Blöcke für die Ping-Pong-Logik werden im nächsten Schritt in separate Compilation-Units (`app_logic_master.cpp`, `app_logic_slave.cpp`) ausgelagert und über CMake verlinkt.
   
2. **Vorbereitung der "Basic"-Schicht:**
   Die `main.cpp` wird weiter verschlankt, um als reine Start-Routine zu fungieren. Die eigentliche Applikationsschleife (Main-Loop) wird in eine abstrakte Hardware-unabhängige Ebene verschoben, um zukünftig nahtlos zwischen echter Hardware und Software-in-the-Loop (SIL) Simulation wechseln zu können.

3. **Speicherallokation der DMA-Puffer:**
   Die aktuell über `reinterpret_cast` definierten Speicheradressen für die DMA-Puffer werden durch eine saubere Linker-Script-Sektion (z.B. `.d2_sram`) und die Verwendung von `__attribute__((section("...")))` ersetzt.
