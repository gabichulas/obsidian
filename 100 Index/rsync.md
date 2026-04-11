# Project Blueprint: The Delta-Transfer Engine (Mini-rsync)

## 1. The Core Objective
Build a bandwidth-optimized file synchronization CLI tool from scratch. Instead of blindly copying an entire file over the network like `scp`, this engine uses a block-level hashing algorithm to detect exactly which parts of a file have changed. It then transmits only those specific modified bytes over an SSH tunnel.

## 2. Why This is an L3/DevOps Level Project
It moves beyond basic configuration and proves fundamental engineering mastery of:
* **Low-Level Data I/O:** Manipulating raw binary data in chunks rather than loading entire files into RAM.
* **Network Optimization:** Solving actual distributed system bandwidth exhaustion.
* **Cryptography:** Utilizing SHA-256 for data integrity and state comparison.
* **Linux IPC (Inter-Process Communication):** Managing `stdin` and `stdout` streams directly through a live SSH subprocess without relying on external network libraries.

## 3. The Architecture & Execution Flow
You will build two components: a local Sender script and a remote Receiver script. They will communicate exclusively through raw SSH pipes.

### Phase 1: The Remote Hash Manifest
* The Sender opens an SSH subprocess from your machine and remotely executes the Receiver script on the target Linux VM.
* The Receiver opens its outdated version of the target file on its disk.
* It splits the file into 4KB (4096-byte) blocks.
* It calculates a SHA-256 hash for each block.
* It streams this list of hashes (the "Manifest") back through `stdout` to the Sender.

### Phase 2: The Differential Analysis (The Diff)
* The Sender reads the Manifest from the SSH pipe.
* The Sender opens its locally updated version of the file and cuts it into matching 4KB blocks.
* It calculates the hash of its local Block 0 and compares it to the remote Block 0 hash.
* If the hashes match, the data is identical. The Sender skips it.
* If the hashes differ (meaning the data was modified) or the block doesn't exist remotely, the Sender flags this block as "Dirty."

### Phase 3: The Delta Stream
* The Sender takes the raw binary data of only the "Dirty" blocks.
* It packs them into a custom binary protocol frame (e.g., attaching the Block Index Number to the front of the Raw Data).
* It streams this patch payload directly into the `stdin` of the SSH subprocess.

### Phase 4: The In-Place Patch
* The Receiver catches the binary patch payload coming from its `stdin`.
* It seeks to the exact block index in its local file.
* It physically overwrites the old 4KB block with the new 4KB data.
* The file is now perfectly synchronized. If a 10GB database had a 4KB row changed, only 4KB of data crossed the network.

## 4. Implementation Milestones
* **Milestone A (Local Logic):** Build the hashing and patching logic locally. Create a dummy file, alter a sentence in the middle, and ensure your logic can detect the changed block, extract it, and successfully patch a backup file without touching the network.
* **Milestone B (The Protocol):** Define how you will format the data being sent over the pipe. The Receiver needs to know where a block index ends and where the raw data begins.
* **Milestone C (The Network):** Wire the scripts together using subprocesses to call SSH and securely manage the bidirectional `stdin`/`stdout` data streams.
* **Milestone D (Resilience):** Add error handling for broken SSH pipes, missing target directories, or corrupted payloads.