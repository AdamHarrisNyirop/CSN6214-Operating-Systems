# CSN6214 OPERATING SYSTEMS - TRIMESTER 2530 (CONCURRENT NETWORKED BOARD GAME INTERPRETER)

## Submission Date

8th February 2026, Sunday, 5:00 PM, Week 14

## Team Size

3 to 4 members and must be from the same tutorial or lab section

## Project Overview & Goal

You will design and implement a **multiplayer, text-based board game** for **3 to 5 players** using a **hybrid concurrency model** that **combines multiprocessing and multithreading**. The system must also support the playing of **multiple, successive games** without requiring a full server restart. This assignment demonstrates advanced understanding of:
- **Multiprocessing** via `fork()` for client isolation
- **Multithreading** for internal server tasks (Example: logging, scheduling)
- **Inter-Process Communication (IPC)**
- **Synchronization** across threads **and** processes (using mutexes, semaphores)
- **Round Robin (RR) scheduling** for turn management
- **Concurrent, safe logging** of all game events

You may **choose your own turn-based, text-based game**, provided it meets the constraints in Section 2. The system must support one of the following deployment modes:
- **Single-Machine Mode**: All components on one host; client-server communication via **IPC** (Example: named pipes, message queues)
- **Multi-Machine Mode**: Clients and server on different machines; communication via **TCP sockets (IPv4)**

**Hybrid concurrency is mandatory**: Your server must use **both `fork()` (for clients) AND POSIX threads (`pthreads`) for internal coordination**.

## Game Requirements (Student-Selected)

Select any **turn-based, text-based game** that satisfies:
- Supports **exactly 3 to 5 players**
- **Server-enforced rules** (no client-side validation)
- **Text-only CLI interface**
- **Clear win/loss/draw condition**
- **Moderate complexity** (variants of Tic-Tac-Toe, simplified card games, race games, word games)
- All randomness (dice, cards) must be generated **by the server only**

## Mandatory Concurrency Model: Hybrid (fork + Threads)

Your server **must combine**:

### Multiprocessing (fork())
- For each client that connects or joins, the server **forks a child process** to handle that player’s session
- The parent process **must reap zombies** (Example: using `SIGCHLD` + `waitpid()`)

### Multithreading (pthreads)
- The **main server process (parent) must create at least two internal threads** to handle:
  - **Round Robin Turn Scheduler** (manages whose turn it is and advances turns)
  - **Concurrent Logger** (writes all game events to `game.log`)
- These threads must run **concurrently** with the main accept loop and with each other
- Threads must **coordinate safely** with `fork()`ed children via **shared memory and synchronization primitives**

## Logger Requirements (Thread-Safe & Concurrent)
- Log all events to `game.log`: connections, moves, turn changes, game end
- The **logger must run in its own thread**
- All log messages must be:
  - **Complete** (no interleaving)
  - **Ordered** (chronologically consistent)
  - **Non-blocking** to gameplay
- Use **synchronization** (Example: a mutex or semaphore) to protect the log queue or file access **across threads and processes**

## Round Robin Scheduler (Thread-Based)
- A **dedicated scheduler thread** in the parent process must:
  - Maintain the **cyclic player order**
  - Determine the **current player’s turn**
  - Signal when a player may act
- Turn state must reside in **shared memory** so `fork()`ed children can read it
- All updates to turn state must be **synchronized** (mutex/semaphore) and visible across processes
- The scheduler must **skip disconnected/inactive players**

## Architectural & Synchronization Requirements

### Interprocess Communication (IPC) & Communication
- **Shared game state** (board, positions, turn, etc.) must reside in **POSIX shared memory**
- In **single-machine mode**, client-server communication uses **IPC** (Example: named pipes)
- In **multi-machine mode**, client-server uses **TCP**, but server internals still use shared memory + threads

### Synchronization Across Domains
- You must use **both**:
  - **Process-Shared Mutexes/Semaphores** (for `fork()`ed children ↔ parent threads)
  - **Thread Mutexes** (for internal thread coordination, if needed)
- All access to shared memory (by threads **or** child processes) must be **mutually exclusive**
- Initialize shared mutexes with `PTHREAD PROCESS SHARED`

## Persistent Scoring & History

The server must implement a persistent scoring mechanism to track player statistics across sessions.

### Scores File Requirements
- **Persistent Storage**: The server must maintain player win statistics in a file named `scores.txt`.
- **Loading**: The server must **load the `scores.txt` file into shared memory** upon server startup. If the file does not exist, it should be created
- **Updating**: At the conclusion of every game, the winning player’s score must be **atomically updated** in the shared memory structure.
- **Saving**: Theservermustwrite the updated scores from memory back to `scores.txt` upon server shutdown (using signal handling, example: `SIGINT`) and potentially after every completed game

### Concurrency & Synchronization
- The in-memory score structure is a critical shared resource
- All read and write operations on the score structure must be protected using the **Process-Shared Mutexes/Semaphores** to prevent race conditions, especially when game-end events are triggered by child processes

## Deliverables

### Source Code (C/C++)
- `server.c`, `client.c`, `Makefile`
- Must use `fork()` **and** `pthread_create()`
- Compile on Linux with `gcc -pthread`

### Design Report (PDF, 10-15 pages)
- **Game description** and rules
- **Deployment Mode** (IPC or TCP)
- **Hybrid Architecture**: Diagram showing processes, threads, and data flow
- **IPC mechanism** and shared memory layout
- **Synchronization Strategy**: How mutexes/semaphores coordinate threads and processes
- **Logger Design**: thread structure, queue, safety
- **Round Robin Scheduler**: How the thread manages turns across processes
- **Persistence Strategy**: Describe the file format, the loading/saving mechanism for `scores.txt`, and the synchronization used to protect the in-memory score data
- **Multi-Game Handling**: Explain how the server resets and re-initializes for the start of a new game
- **Testing Evidence**: gameplay screenshots and sample `game.log`
- **Screenshots**: screenshots of each client view, part of the logger and persistent storage content

### README.txt
- How to compile (`make`) and run
- Example commands
- Game rules summary
- Mode supported

### Video Demonstration
- A video recording (maximum **5 minutes**) demonstrating the following:
  - Compilation using make
  - Running the server and connecting the minimum number of clients (3 players)
  - Demonstrating a few full rounds of gameplay
  - Showing the concurrent logging (`game.log`) occurring in real-time
  - Showing the updated `scores.txt` file after a game ends
  - Each student must present his part of the project as provided in the table of responsibilities

### Team Responsibilities
The team must consist of 3 to 4 members from the same tutorial/lab section. Include the following table detailing the division of labor. Ensure all critical components are assigned. If the team has 4 members, add an extra row to the table.

| **Name/ID** | **Primary Role** | **Key Components Developed/Implemented** |
| :-----: | :-----: | :-----: |
| [Member 1 Name/ID] | [Example: Server Core, IPC] | [List specific components: `fork()`, shared memory setup, logger thread] |
| [Member 2 Name/ID] | [Example: Client/Game Logic] | [List specific components: Client handler, game state rules, scheduler thread] |
| [Member 3 Name/ID] | [Example: Persistence/Networking] | [List specific components: `scores.txt` management, TCP/IPC connection logic] |

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
| **Multithreading (`pthreads`)** | Drake | the type | to type | in this table | **10** |
| **IPC & Shared Memory** | Hawk | Tuah | Spit on | that thang | **10** |
| **Cross-Domain Synchronization** | Boiiiiiiiiii | what | the | chungus 😂 | **15** |
| **Round Robin Scheduler (Thread)** | Those | who | know | 💀 | **10** |
| **Concurrent Logger (Thread)** | 6 | 7 | 6 | 1 | **10** |
| **Persistent Scoring & Multi-Game** | Tung | Tung | Tung | Sahur | **10** |
| **Student-Chosen Game** | Gween | Bean | Whatchu | Meaaaaan | **5** |

### Individual/Deliverable Mark (25 Points)

| **Component** | **Fail (Zero Points)** | **Poor (Partial Points)** | **Good (Intermediate Points)** | **Excellent (Maximum Points)** | **Maximum Points** |
| :-----: | :-----: | :-----: | :-----: | :-----: | :-----: |
| **Code Quality & Build** | Student did not contribute to both the coding and the report (must contribute equally to both) | **3 Points**: Student’s part of the code is present but does not perform all the expected tasks | **6 Points**: Student’s code performs the majority of the tasks as expected | **8-10 Points**: Excellent, professional-quality code with consistent style, comprehensive comments and a fully functional `Makefile` | **10** |
| **Deliverables Quality** | Critical deliverables (Report or Video) are missing | **5 Points**: Deliverables present but lack detail (Report ≤ 2 pages, video ≤ 3 required points shown) | **10 Points**: All deliverables present; Report is 10 to 15 pages, video shows most required points, roles defined | **12-15 Points**: All deliverables are professional and complete; Report is 4-5 pages, video clearly demonstrates all functional points and roles are clearly defined and balanced | **15** |

## Why This Design?

Real-world systems (Example: web servers, databases) often combine processes (for fault isolation) and threads (for efficiency). This assignment gives you hands-on experience with **complex concurrency orchestration** - a hallmark of robust OS-level programming.

# Policies Of Artificial Intelligence (AI) Tool Usage

## Preamble

These policies govern the responsible, ethical, and academically honest use of organizational internet resources and Artificial Intelligence (AI) tools, specifically for the CSN6214 Operating Systems Assignment. These rules are mandatory and failure to comply will result in disciplinary action.

## Artificial Intelligence (AI) Tools Usage Policy

### Academic Integrity and Attribution (Students)

#### Disclosure and Citation (Permitted Use):
- AI tools (e.g., ChatGPT, Bard, Copilot) may be used **only** for general concept understanding (Example: ”Explain synchronization primitives”) or to generate draft documentation text
- Any text, diagrams, or utility code generated by an AI tool and included in the **Design Report** or submitted code **must be clearly and accurately cited**
- Failure to disclose is plagiarism

#### Prohibition on Core Logic Generation (Strictly Forbidden):
- AI tools **must not** be used to generate the core, architectural, and graded components of this assignment
- Prohibited Core Components include: Hybrid concurrency logic (`fork()` and `pthreads`), shared memory setup, cross-domain synchronization logic, Round Robin Scheduler logic, and Concurrent Logger logic

### Data Privacy and Confidentiality

#### No Sensitive Data Input
- Users must **never input, upload, or paste confidential, proprietary, or personally identifiable information (PII)** into any third-party AI tool
- Data submitted is often used to train the model and may not remain private

#### Verification of Output
- Users must **independently verify** all facts, data, and sources generated by AI tools
- AI output may contain errors or ”hallucinations” (false information)

## Academic Honesty & Code Originality Policy

### Prohibition on Plagiarism and Code Copying
- **Code Originality**: The submitted source code (`server.c`, `client.c`, `Makefile`) **must be original work** created solely by the members of the registered group
- **Unauthorized Duplication**: Copying code or solutions, even partially, from another student, another team, or external online sources (Example: GitHub, Stack Overflow) that address the core assignment components is strictly forbidden

### Strict Secrecy and Zero Inter-Group Communication
- **Total Secrecy**: Teams must maintain **absolute secrecy** regarding their project design, implementation, and code
- **No External Discussion**: You must **not** discuss the assignment problem, potential solutions, architectural decisions, synchronization strategies, or implementation details with any student outside of your registered group
- **Internal Collaboration Only**: All collaboration must be strictly confined to the 3 to 4 registered members of your team

## Enforcement & Consequences
**Violations**: Failure to comply with any part of these policies (including undisclosed AI use, plagiarism, or inter-group communication) will result in disciplinary action, including but not limited to:
- A grade of **zero (0)** for the entire assignment for all involved students
- Further disciplinary action in line with academic integrity policies

> [!NOTE]
> Students are responsible for reading and understanding all sections of this policy prior to beginning work on the assignment.
