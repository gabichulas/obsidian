# The Delta-Transfer Engine: Block-Level Synchronization over SSH

## 📌 Overview
The Delta-Transfer Engine is a custom-built, bandwidth-optimized file synchronization utility. Designed as a foundational study in Inter-Process Communication (IPC) and distributed data replication, this project replaces standard `scp` with an intelligent algorithm that only transmits the modified bytes of a file. It demonstrates a deep understanding of cryptography, standard I/O streams, block-level filesystem operations, and network bandwidth conservation.

## 🏗️ Architecture
The system operates across a secure shell tunnel, utilizing standard input/output streams to communicate between a local Sender and a remote Receiver without requiring external APIs.

    [ SOURCE SERVER (Pop!_OS) ]                            [ TARGET SERVER (Ubuntu VM) ]
           File: db_dump.sql                                      File: db_dump.sql
                  |                                                      |
        1. Cut into 4KB Blocks                                 1. Cut into 4KB Blocks
        2. Generate SHA-256 Hashes                             2. Generate SHA-256 Hashes
                  |                                                      |
                  |<----------- [ SSH PIPE: Send Target Hashes ] --------|
                  |
        3. Compare Source Hashes
           against Target Hashes
                  |
        4. Extract only the changed
           4KB binary blocks
                  |
                  |------------- [ SSH PIPE: Send Delta Patch ] -------->|
                                                                         |
                                                               3. Apply binary patch 
                                                                  to existing file blocks

## 🛠️ Technology Stack
* **Networking:** SSH Subprocesses, TCP/IP, Linux IPC Pipes (`stdin`/`stdout`)
* **Logic/Scripting:** Python (Pure Standard Library)
* **Cryptography:** SHA-256 Hashing Algorithms
* **OS Level:** Binary File I/O, POSIX File Descriptors

## 💻 The Core Engine (Python Implementation)
Below is the foundational hashing and patching logic that powers the engine. This code handles the block-level differential analysis before data is streamed over the SSH tunnel.

```python
import hashlib
import os

# 4KB chunk size is standard for filesystem block optimization
CHUNK_SIZE = 4096  

def generate_block_hashes(file_path):
    """
    Reads a file and returns a dictionary mapping the block index 
    to its SHA-256 hash string.
    """
    hashes = {}
    if not os.path.exists(file_path):
        return hashes

    with open(file_path, 'rb') as f:
        block_index = 0
        while True:
            chunk = f.read(CHUNK_SIZE)
            if not chunk:
                break
            
            # Calculate SHA-256 for this specific binary chunk
            block_hash = hashlib.sha256(chunk).hexdigest()
            hashes[block_index] = block_hash
            block_index += 1

    return hashes

def get_delta_payload(source_path, target_hashes):
    """
    Compares the source file against the target's hashes.
    Returns a dictionary of {block_index: raw_binary_data} ONLY for 
    the blocks that are missing or modified.
    """
    delta_payload = {}
    with open(source_path, 'rb') as f:
        block_index = 0
        while True:
            chunk = f.read(CHUNK_SIZE)
            if not chunk:
                break

            current_hash = hashlib.sha256(chunk).hexdigest()
            
            # If the target doesn't have this block index, or the hash 
            # is different (meaning the data changed), we pack the raw bytes.
            if target_hashes.get(block_index) != current_hash:
                delta_payload[block_index] = chunk
                
            block_index += 1

    return delta_payload

def apply_patch(target_path, delta_payload):
    """
    Takes the raw binary patches and applies them to the target file.
    """
    blocks = {}
    
    # 1. Read the existing target file into memory blocks
    if os.path.exists(target_path):
        with open(target_path, 'rb') as f:
            block_index = 0
            while True:
                chunk = f.read(CHUNK_SIZE)
                if not chunk:
                    break
                blocks[block_index] = chunk
                block_index += 1
                
    # 2. Overwrite the old blocks with the new patched data
    for index, raw_data in delta_payload.items():
        blocks[index] = raw_data
        
    # 3. Write the fully reconstructed file back to disk
    with open(target_path, 'wb') as f:
        max_index = max(blocks.keys()) if blocks else -1
        for i in range(max_index + 1):
            if i in blocks:
                f.write(blocks[i])
```

## ⚙️ The Execution Flow
1. **The Remote Hash Manifest:** The local script opens an SSH tunnel and executes the receiver script on the target server. The receiver reads its local version of the file, splits it into 4096-byte chunks, calculates a SHA-256 hash for each chunk, and streams this manifest back over `stdout`.
2. **The Local Diff Calculation:** The local script intercepts the hash manifest. It chunks its own file and compares the hashes. If Block 500 on the local machine has a different hash than Block 500 on the remote machine, Block 500 is flagged as "dirty."
3. **The Delta Stream:** The local script packs the raw binary bytes of the "dirty" blocks into a custom frame (Index Number + Data) and streams it directly into the SSH `stdin` pipe. 
4. **The In-Place Patch:** The remote server catches the binary stream and physically overwrites the specific 4KB blocks in its local file, avoiding a complete rewrite and preserving disk IOPS.

## 📄 Sample Post-Mortem Incident Report

**Incident ID:** INC-20260410-02
**Date:** April 10, 2026
**Severity:** SEV-2 (Intermittent API Latency & Network Saturation)

**1. Executive Summary:**
At 02:00 UTC, the primary production database triggered its scheduled nightly backup transfer to the off-site disaster recovery environment. The backup file (50GB PostgreSQL dump) saturated the cross-region WAN link for 45 minutes. This bandwidth exhaustion caused intermittent timeouts for edge API services sharing the same network interface. The issue was permanently resolved by replacing `scp` with the custom Delta-Transfer Engine.

**2. Timeline (UTC):**
* **02:00** - Scheduled `cron` job initiated `scp` transfer of `billing_prod.sql`.
* **02:05** - PagerDuty alerted L3 Support to a spike in 504 Gateway Timeouts on the main billing API.
* **02:10** - L3 Engineer investigated edge routers and noted 99% bandwidth utilization on the outgoing network interface (`eth0`).
* **02:15** - Identified the `scp` process as the culprit. The process was throttled using `tc` (Traffic Control) to restore API stability, but this extended the backup window to 3 hours.
* **05:00** - Backup completed. Link saturation subsided.

**3. Root Cause:**
The standard `scp` protocol is naive; it copies the entire file indiscriminately. Although only 15MB of new transactional data was added to the 50GB PostgreSQL database that day, `scp` forced the transmission of all 50,000,000,000 bytes over the network, choking the outbound bandwidth.

**4. Resolution & Action Items:**
* **Immediate Fix:** Implemented bandwidth throttling via `tc qdisc` to prevent future SEV-2 alerts during the backup window.
* **Permanent Fix:** Replaced the legacy `scp` cron job with the custom Delta-Transfer Engine. 
* **Validation:** On the next nightly run, the Delta Engine successfully hashed the 50GB file locally, identified that only 15MB of 4KB blocks had changed, and transmitted the delta patch. 
* **Result:** Total network transfer size was reduced from 50GB to 15MB. Transfer time was reduced from 45 minutes to 4.2 seconds. API latency remained stable at <50ms during the transfer.