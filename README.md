# CSN6214 OPERATING SYSTEMS - TRIMESTER 2530 (CONCURRENT NETWORKED BOARD GAME INTERPRETER)

## Submission Date

8th February 2026, Sunday, 5:00 PM, Week 14

## Team Size

3 to 4 members and must be from the same tutorial or lab section

## Project Overview & Goal

You will design and implement a **multiplayer, text-based board game** for **3 to 5 players** using a **hybrid concurrency model** that **combines multiprocessing and multithreading**. The system must also support the playing of **multiple, successive games** without requiring a full server restart. This assignment demonstrates advanced understanding of:
- **Multiprocessing** via `fork()` for client isolation
- **Multithreading** for internal server tasks[^1]
- **Inter-Process Communication (IPC)**
- **Synchronization** across threads **and** processes[^2]
- **Round Robin (RR) scheduling** for turn management
- **Concurrent, safe logging** of all game events

You may **choose your own turn-based, text-based game**, provided it meets the constraints in the next section. The system must support one of the following deployment modes:
- **Single-Machine Mode**: All components on one host; client-server communication via **IPC**[^3]
- **Multi-Machine Mode**: Clients and server on different machines; communication via **TCP sockets (IPv4)**

**Hybrid concurrency is mandatory**: Your server must use **both `fork()`[^4] AND POSIX threads[^5] for internal coordination**.

## Game Requirements (Student-Selected)

Select any **turn-based, text-based game** that satisfies:
- Supports **exactly 3 to 5 players**
- **Server-enforced rules**[^6]
- **Text-only CLI interface**
- **Clear win/loss/draw condition**
- **Moderate complexity**[^7]

> [!IMPORTANT]
> All randomness[^8] must be generated **by the server only**.

## Mandatory Concurrency Model: Hybrid (fork + Threads)

Your server **must combine**:

### Multiprocessing (fork())
- For each client that connects or joins, the server **forks a child process** to handle that player’s session
- The parent process **must reap zombies**[^9]

### Multithreading (pthreads)
- The **main server process[^10] must create at least two internal threads** to handle:
  - **Round Robin Turn Scheduler**[^11]
  - **Concurrent Logger**[^12]
- These threads must run **concurrently** with the main accept loop and with each other
- Threads must **coordinate safely** with `fork()`ed children via **shared memory and synchronization primitives**

## Logger Requirements (Thread-Safe & Concurrent)
- Log all events to `game.log`: connections, moves, turn changes, game end
- The **logger must run in its own thread**
- All log messages must be:
  - **Complete**[^13]
  - **Ordered**[^14]
  - **Non-blocking** to gameplay
- Use **synchronization**[^15] to protect the log queue or file access **across threads and processes**

## Round Robin Scheduler (Thread-Based)
- A **dedicated scheduler thread** in the parent process must:
  - Maintain the **cyclic player order**
  - Determine the **current player’s turn**
  - Signal when a player may act
- Turn state must reside in **shared memory** so `fork()`ed children can read it
- All updates to turn state must be **synchronized**[^16] and visible across processes
- The scheduler must **skip disconnected/inactive players**

## Architectural & Synchronization Requirements

### Inter-Process Communication (IPC) & Communication
- **Shared game state**[^17] must reside in **POSIX shared memory**
- In **single-machine mode**, client-server communication uses **IPC**[^18]
- In **multi-machine mode**, client-server uses **TCP**, but server internals still use shared memory and threads

### Synchronization Across Domains
- You must use **both**:
  - **Process-Shared Mutexes/Semaphores**[^19]
  - **Thread Mutexes**[^20]
- All access to shared memory[^21] must be **mutually exclusive**
- Initialize shared mutexes with `PTHREAD PROCESS SHARED`

## Persistent Scoring & History

The server must implement a persistent scoring mechanism to track player statistics across sessions.

### Scores File Requirements
- **Persistent Storage**: The server must maintain player win statistics in a file named `scores.txt`.
- **Loading**: The server must **load the `scores.txt` file into shared memory** upon server startup. If the file does not exist, it should be created
- **Updating**: At the conclusion of every game, the winning player’s score must be **atomically updated** in the shared memory structure.
- **Saving**: The server must write the updated scores from memory back to `scores.txt` upon server shutdown using signal handling [^22] and potentially after every completed game

### Concurrency & Synchronization
- The in-memory score structure is a critical shared resource
- All read and write operations on the score structure must be protected using the **Process-Shared Mutexes/Semaphores** to prevent race conditions, especially when game-end events are triggered by child processes

## Deliverables

### Source Code (C/C++)
- `server.c`, `client.c`, `Makefile`
- Must use `fork()` **and** `pthread_create()`
- Compile on Linux with `gcc -pthread`

### Design Report (PDF, 10-15 Pages)
1. **Game description** and rules
2. **Deployment Mode**[^23]
3. **Hybrid Architecture**: Diagram showing processes, threads, and data flow
4. **IPC mechanism** and shared memory layout
5. **Synchronization Strategy**: How mutexes/semaphores coordinate threads and processes
6. **Logger Design**: thread structure, queue, safety
7. **Round Robin Scheduler**: How the thread manages turns across processes
8. **Persistence Strategy**: Describe the file format, the loading/saving mechanism for `scores.txt`, and the synchronization used to protect the in-memory score data
9. **Multi-Game Handling**: Explain how the server resets and re-initializes for the start of a new game
10. **Testing Evidence**: gameplay screenshots and sample `game.log`
11. **Screenshots**: screenshots of each client view, part of the logger and persistent storage content

### README.txt
- How to compile[^24] and run
- Example commands
- Game rules summary
- Mode supported

### Video Demonstration
- A video recording[^25] demonstrating the following:
  1. Compilation using make
  2. Running the server and connecting the minimum number of clients[^26]
  3. Demonstrating a few full rounds of gameplay
  4. Showing the concurrent logging[^27] occurring in real-time
  5. Showing the updated `scores.txt` file after a game ends
  6. Each student must present his part of the project as provided in the table of responsibilities

### Team Responsibilities
The team must consist of 3 to 4 members from the same tutorial/lab section. Include the following table detailing the division of labor. Ensure all critical components are assigned. If the team has 4 members, add an extra row to the table.

| **Name/ID** | **Primary Role** | **Key Components Developed/Implemented** |
| :-----: | :-----: | :-----: |
| [Member 1 Name/ID] | [**EXAMPLE**: Server Core, IPC] | [List specific components: `fork()`, shared memory setup, logger thread] |
| [Member 2 Name/ID] | [**EXAMPLE**: Client/Game Logic] | [List specific components: Client handler, game state rules, scheduler thread] |
| [Member 3 Name/ID] | [**EXAMPLE**: Persistence/Networking] | [List specific components: `scores.txt` management, TCP/IPC connection logic] |

## Critical Constraints
- **Both `fork()` AND `pthreads` are required** because omitting either results in significant point loss
- **No pure-thread or pure-process solutions**
- **Shared mutexes must be initialized with** `PTHREAD PROCESS SHARED` to work across `fork()`
- **The logger and scheduler must be threads in the parent process** - not separate processes
- **All players must be handled in child processes** - not threads

## Evaluation Criteria (Total: 100 Points) - Detailed Rubric

The assignment will be graded based on the technical implementation of concurrency and synchronization, adherence to the hybrid model, and the quality of the deliverables.

### Group Mark (Technical Core - 75 Points)

| **Component** | **Fail (Zero Points)** | **Poor (Partial Points)** | **Good (Intermediate Points)** | **Excellent (Maximum Points)** | **Maximum Points** |
| :-----: | :-----: | :-----: | :-----: | :-----: | :-----: |
| **Multiprocessing (`fork`)** | Sequential processing or no `waitpid` used | **2 Points**: Forking exists but zombie reaping is unstable/blocking | **3 Points**: Stable forking per client; functional, non-blocking zombie reaping | **4-5 Points**: Stable forking; `SIGCHLD`-based zombie reaping; proper error handling | **5** |
| **Multithreading (`pthreads`)** | Server uses only processes or only one internal thread | **3 Points**: Two threads exist but are unstable, frequently blocking or fail often | **6 Points**: Two threads exist, run concurrently, but exhibit minor coordination issues | **8-10 Points**: Scheduler and Logger threads correctly created, stable, concurrent and fulfill their roles perfectly | **10** |
| **IPC & Shared Memory** | No shared memory or critical data stored locally per process | **3 Points**: Shared memory set up, but only partial critical data stored or IPC is unreliable | **6 Points**: Shared memory holds all critical state; communication mechanism works | **8-10 Points**: Shared memory correctly initialized and efficiently used for all critical state; IPC/TCP is robust and reliable | **10** |
| **Cross-Domain Synchronization** | Shared memory used without any synchronization[^28] | **5 Points**: Synchronization used, but incorrect type[^29] or not protecting all critical sections | **10 Points**: Synchronization used correctly on all critical sections | **12-15 Points**: All accesses to shared memory are fully protected using the correct primitives; system is provably safe and deadlock-free | **15** |
| **Round Robin Scheduler (Thread)** | Turn logic is handled sequentially by client processes or fails to cycle | **3 Points**: Scheduler thread exists but updates turn state without synchronization or fails to cycle | **6 Points**: Scheduler correctly manages turn order and uses synchronization, but player skipping is buggy | **8-10 Points**: Dedicated thread fully implements Round Robin (RR) scheduling, safely updates shared state and reliably skips inactive players | **10** |
| **Concurrent Logger (Thread)** | Logging is performed inline by client processes or file synchronization is absent | **3 Points**: Logger thread exists but exhibits file corruption[^30] or blocking behavior | **6 Points**: Logger thread is safe and functional, but may occasionally cause brief delays or logs are not fully ordered | **8-10 Points**: Dedicated thread ensures complete, ordered, non-blocking logging to `game.log` using a highly efficient synchronized queue/mechanism | **10** |
| **Persistent Scoring & Multi-Game** | Scoring is volatile or server crashes after one game | **3 Points**: Scores load/save correctly, but updates are not protected[^31], or game reset is buggy | **6 Points**: Scores are saved/loaded; updates are protected; system supports multiple games | **8-10 Points**: `scores.txt` correctly handled; score updates are atomically protected; server reliably resets and handles successive games flawlessly | **10** |
| **Student-Chosen Game** | Game is unplayable or violates mandatory player constraints | **2 Points**: Game is playable but basic[^32] or rules are unclear | **3 Points**: Game is functional, meets all constraints and rules are clear | **4-5 Points**: Game is well-designed, functional, meets all constraints and demonstrates moderate complexity | **5** |

### Individual/Deliverable Mark (25 Points)

| **Component** | **Fail (Zero Points)** | **Poor (Partial Points)** | **Good (Intermediate Points)** | **Excellent (Maximum Points)** | **Maximum Points** |
| :-----: | :-----: | :-----: | :-----: | :-----: | :-----: |
| **Code Quality & Build** | Student did not contribute to both the coding and the report[^33] | **3 Points**: Student’s part of the code is present but does not perform all the expected tasks | **6 Points**: Student’s code performs the majority of the tasks as expected | **8-10 Points**: Excellent, professional-quality code with consistent style, comprehensive comments and a fully functional `Makefile` | **10** |
| **Deliverables Quality** | Critical deliverables[^34] are missing | **5 Points**: Deliverables present but lack detail[^35] | **10 Points**: All deliverables present; report is 10 to 15 pages, video shows most required points, roles defined | **12-15 Points**: All deliverables are professional and complete; report is 4-5 pages, video clearly demonstrates all functional points and roles are clearly defined and balanced | **15** |

## Why This Design?

Real-world systems[^36] often combine processes[^37] and threads[^38]. This assignment gives you hands-on experience with **complex concurrency orchestration** - a hallmark of robust OS-level programming.

# Policies Of Artificial Intelligence (AI) Tool Usage

## Preamble

These policies govern the responsible, ethical, and academically honest use of organizational internet resources and Artificial Intelligence (AI) tools, specifically for the CSN6214 Operating Systems Assignment. These rules are mandatory and failure to comply will result in disciplinary action.

## Artificial Intelligence (AI) Tools Usage Policy

### Academic Integrity and Attribution (Students)

#### Disclosure and Citation (Permitted Use):
- AI tools[^39] may be used **only** for general concept understanding[^40] or to generate draft documentation text
- Any text, diagrams, or utility code generated by an AI tool and included in the **Design Report** or submitted code **must be clearly and accurately cited**
- Failure to disclose is plagiarism

#### Prohibition on Core Logic Generation (Strictly Forbidden):
- AI tools **must not** be used to generate the core, architectural, and graded components of this assignment
- Prohibited Core Components include: Hybrid concurrency logic[^41], shared memory setup, cross-domain synchronization logic, Round Robin Scheduler logic, and Concurrent Logger logic

### Data Privacy and Confidentiality

#### No Sensitive Data Input
- Users must **never input, upload, or paste confidential, proprietary, or personally identifiable information (PII)** into any third-party AI tool
- Data submitted is often used to train the model and may not remain private

#### Verification of Output
- Users must **independently verify** all facts, data, and sources generated by AI tools
- AI output may contain errors or ”hallucinations”[^42]

## Academic Honesty & Code Originality Policy

### Prohibition on Plagiarism and Code Copying
- **Code Originality**: The submitted source code[^43] **must be original work** created solely by the members of the registered group
- **Unauthorized Duplication**: Copying code or solutions, even partially, from another student, another team, or external online sources[^44] that address the core assignment components is strictly forbidden

### Strict Secrecy and Zero Inter-Group Communication
- **Total Secrecy**: Teams must maintain **absolute secrecy** regarding their project design, implementation, and code
- **No External Discussion**: You must **not** discuss the assignment problem, potential solutions, architectural decisions, synchronization strategies, or implementation details with any student outside of your registered group
- **Internal Collaboration Only**: All collaboration must be strictly confined to the 3 to 4 registered members of your team

## Enforcement & Consequences
**Violations**: Failure to comply with any part of these policies will result in disciplinary action, including but not limited to:
- A grade of **zero (0)** for the entire assignment for all involved students
- Further disciplinary action in line with academic integrity policies

> [!NOTE]
> Students are responsible for reading and understanding all sections of this policy prior to beginning work on the assignment.

[^1]: **EXAMPLE**: logging, scheduling
[^2]: using mutexes, semaphores
[^3]: **EXAMPLE**: named pipes, message queues
[^4]: for clients
[^5]: `pthreads`
[^6]: no client-side validation
[^7]: variants of Tic-Tac-Toe, simplified card games, race games, word games
[^8]: dice, cards
[^9]: **EXAMPLE**: using `SIGCHLD` + `waitpid()`
[^10]: parent
[^11]: manages whose turn it is and advances turns
[^12]: writes all game events to `game.log`
[^13]: no interleaving
[^14]: chronologically consistent
[^15]: **EXAMPLE**: a mutex or semaphore
[^16]: mutex/semaphore
[^17]: board, positions, turn, etc.
[^18]: **EXAMPLE**: named pipes
[^19]: for `fork()`ed children ↔ parent threads
[^20]: for internal thread coordination, if needed
[^21]: by threads **or** child processes
[^22]: **EXAMPLE**: `SIGINT`
[^23]: Inter-Process Communication (IPC) or Transmission Control Protocol (TCP)
[^24]: `make`
[^25]: maximum **5 minutes**
[^26]: 3 players
[^27]: `game.log`
[^28]: guaranteed race conditions
[^29]: **EXAMPLE**: non-shared mutex
[^30]: interleaved logs
[^31]: race condition risk
[^32]: **EXAMPLE**: simple Tic-Tac-Toe
[^33]: must contribute equally to both
[^34]: Report or Video
[^35]: report ≤ 2 pages, video ≤ 3 required points shown
[^36]: **EXAMPLE**: web servers, databases
[^37]: for fault isolation
[^38]: for efficiency
[^39]: **EXAMPLE**: ChatGPT, Bard, Copilot
[^40]: **EXAMPLE**: ”Explain synchronization primitives”
[^41]: `fork()` and `pthreads`
[^42]: false information
[^43]: `server.c`, `client.c`, `Makefile`
[^44]: **EXAMPLE**: GitHub, Stack Overflow
[^45]: including undisclosed AI use, plagiarism, or inter-group communication
