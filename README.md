# ⚡ Tomasulo Algorithm Simulator
### A Cycle-Accurate MIPS Processor Simulator in Java

[![Java](https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk&logoColor=white)](https://www.oracle.com/java/)
[![Architecture](https://img.shields.io/badge/CS-Computer_Architecture-blue?logo=intel)]()
[![Status](https://img.shields.io/badge/Status-Completed-green)]()

## 📖 Overview
This project is a sophisticated **Instruction Level Parallelism (ILP)** simulator that implements the **Tomasulo Algorithm**. It allows users to visualize how a processor dynamically schedules instructions to execute out-of-order while preserving data integrity.

The simulator mimics a superscalar MIPS processor pipeline, handling **RAW (Read-After-Write)**, **WAR (Write-After-Read)**, and **WAW (Write-After-Write)** hazards through register renaming and a **Common Data Bus (CDB)**.

## ⚙️ Key Features
* **🔄 Dynamic Scheduling:** Implements full out-of-order execution using Reservation Stations.
* **🏗️ Pipeline Stages:** Accurately simulates **Issue**, **Execute**, and **Write Result** stages.
* **💾 Memory Hierarchy:** Simulates a **Cache + Main Memory** system with configurable hit/miss latencies (`Data.java`).
* **🚧 Hazard Resolution:** Automatically resolves data hazards using register tags and renaming.
* **🔀 Branch Handling:** Supports branch instructions (`BNE`, `BEQ`) with execution logic.
* **📊 Cycle-by-Cycle State:** Tracks the exact state of every component (Registers, Buffers, Queue) at every clock cycle.

---

## 🏗️ System Architecture

### Core Components
| Component | Description |
| :--- | :--- |
| **Processor** | The central unit that manages the clock cycle and orchestrates the pipeline stages (`Issue`, `Execute`, `Write`). |
| **Reservation Stations** | buffers holding arithmetic instructions waiting for operands (e.g., `ADD`, `MUL`). |
| **Load/Store Buffers** | Queues for memory access operations to handle cache/memory latency. |
| **Register File** | Simulates 32 Floating Point registers (`F0` - `F31`) with "Qi" fields for dependency tracking. |
| **Common Data Bus (CDB)** | Broadcasts results to all waiting Reservation Stations and Registers immediately upon completion. |

### Supported Instruction Set (MIPS)
The simulator supports a wide range of operations defined in `InstructionType.java`:
* **Arithmetic:** `ADD_D`, `SUB_D`, `MUL_D`, `DIV_D`, `ADD_S`, `SUB_S`...
* **Immediate:** `DADDI`, `DSUBI`
* **Memory:** `L_D` (Load Double), `S_D` (Store Double), `LW`, `SW`...
* **Control Flow:** `BNE` (Branch Not Equal), `BEQ` (Branch Equal)

---

## 🛠️ Configuration & Usage

### 1. Setup
Clone the repository and open it in any Java IDE (IntelliJ IDEA, Eclipse, or VS Code).
```bash
git clone [https://github.com/NourSaberO/Tomasulo-Simulator.git](https://github.com/NourSaberO/Tomasulo-Simulator.git)
```
## 2. Configure the Simulation
All hardware parameters can be customized in the main method of `Processor.java`. You can define the number of functional units, operation latencies, and memory specifications.

```java
// 1. Define Reservation Stations (Hardware Resources)
List<ReservationStationGroup> stations = new ArrayList<>();
stations.add(new ReservationStationGroup(3, InstructionType.ADD_D)); // 3 Adders
stations.add(new ReservationStationGroup(2, InstructionType.MUL_D)); // 2 Multipliers

// 2. Define Latencies (Clock Cycles)
InstructionType.setLatency(InstructionType.ADD_D, 2);
InstructionType.setLatency(InstructionType.MUL_D, 10);
InstructionType.setLatency(InstructionType.DIV_D, 40);

// 3. Initialize Memory (Cache Size, Block Size, Hit Latency, Miss Penalty, Memory Size)
Data memory = new Data(5, 2, 2, 10, 100);
```
## 3. Define the Assembly Program
You can hardcode the MIPS instructions directly into the program list. The simulator supports `L_D`, `S_D`, `ADD_D`, `SUB_D`, `MUL_D`, `DIV_D`, and branching.

```java
ArrayList<Instruction> program = new ArrayList<>();

// Format: new Instruction(OpCode, Destination, Source1, Source2)
program.add(new Instruction("L_D", "F6", "21", ""));     // Load F6 from address 21
program.add(new Instruction("L_D", "F2", "20", ""));     // Load F2 from address 20
program.add(new Instruction("MUL_D", "F0", "F2", "F4")); // F0 = F2 * F4
program.add(new Instruction("SUB_D", "F8", "F6", "F2")); // F8 = F6 - F2
program.add(new Instruction("DIV_D", "F10", "F0", "F6")); // F10 = F0 / F6
program.add(new Instruction("ADD_D", "F6", "F8", "F2")); // F6 = F8 + F2 (WAR Hazard Example)

// Run the simulation
Processor cpu = new Processor(stations, buffers, memory, program, 32);
cpu.simulateAll();
```
# 📊 Logic Flow & Pipeline Stages

The simulator implements the classic three-stage Tomasulo pipeline to handle out-of-order execution.

## 1. Issue Stage
*Fetches the next instruction and allocates resources.*

- **Fetch:** Retrieves the next instruction from the Instruction Queue.
- **Check Resources:** Verifies if a Reservation Station (RS) corresponding to the instruction type is free.
- **Rename Registers (Hazard Resolution):**
  - **Available Operands:** If a source operand is ready in the Register File, the value is read into $V_j$ or $V_k$.
  - **Pending Operands:** If the operand is currently being computed by another instruction, the tag (ID of the producing RS) is recorded in $Q_j$ or $Q_k$ to wait for the result.
  - **Busy Bit:** The allocated RS is marked as `Busy`.

## 2. Execute Stage
*Waits for data dependencies and performs the operation.*

- **Monitor CDB:** The RS monitors the Common Data Bus (CDB) for incoming tags.
- **Resolve RAW Hazards:** When a tag matches a waiting $Q_j$ or $Q_k$, the value is latched immediately.
- **Execution:** - Once all source operands are ready (both $Q_j$ and $Q_k$ are empty), the instruction begins execution.
  - **Load/Store:** Effective addresses are calculated during this stage.

## 3. Write Result Stage
*Broadcasts the result to all waiting entities.*

- **Broadcast:** The computed result and the RS Tag are broadcast on the Common Data Bus (CDB).
- **Update Dependents:**
  - **Reservation Stations:** Any RS waiting for this specific tag captures the value.
  - **Register File:** If the Register File is waiting for this tag (and hasn't been reassigned to a newer instruction), it updates the register value.
- **Cleanup:** The Reservation Station is cleared and marked as available for new instructions.
