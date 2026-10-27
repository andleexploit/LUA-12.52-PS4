# PS4 12.52 Sandbox Audit & Exploit Research

**Status:** Archived / Research Phase
**Firmware Target:** PS4 12.52
**Environment:** `remote_lua_loader` (Lua 5.1)
**Authors:** [andre

> 📄 **[READ THE FULL INTERACTIVE REPORT HERE (index.html)](https://TU_USUARIO.github.io/TU_REPO/)** 
> *(Replace this link with your actual GitHub Pages URL)*

---

## 📌 Overview

This repository contains the complete documentation, scripts, and findings from an intensive research project targeting the PlayStation 4 firmware **12.52**. The goal was to discover a new kernel exploit from scratch using only the `remote_lua_loader` as an entry point, completely avoiding WebKit and BD-J.

While a full jailbreak (Kernel R/W -> GoldHEN) was not achieved due to missing public kernel offsets and primitives for this specific firmware, this repository serves as a comprehensive **map of the sandbox**. It documents exactly what works, what is patched, and what vectors are still theoretically viable for future researchers.

---

## 🏆 Complete List of Achievements & Features (What Works)

We successfully achieved and mapped the following capabilities on FW 12.52:

1. **Complete Userland R/W:** Using Lua 5.1 primitives (`memory.read_buffer`, `memory.write_buffer`, `memory.read_qword`, `memory.write_qword`).
2. **Object Manipulation:** `lua.addrof` and `lua.fakeobj` are fully functional. We successfully mapped the `create_fake_cclosure` structure (confirming the controllable pointer at offset `0x20`).
3. **KASLR Bypass:** Successfully leaked kernel pointers and mapped fixed bases.
   * **Kernel Leak:** `0xffffffffb8fd86f0` (via Event Flag Syscall 638).
   * **Userland Ptr:** `0x0000ffffaba0` (fixed pointer in the Event Flag object).
   * **Fixed Bases:** `eboot.bin` (0x400000), `libkernel.sprx` (0x800000000), `libc.prx` (0x80190000).
4. **Syscall Prober:** Mapped 200+ syscall wrappers. Identified the return behaviors and argument validation for the proprietary Syscall families (500-600).
5. **Event Flag Family Mapped:** Fully documented Syscalls `538` (Create), `539` (Delete), `544` (Write), `548` (Write offsets), and `638` (Leak).
6. **AIO Subsystem Mapped:** Documented syscalls `600-610` (`aio_submit_cmd`, `aio_multi_delete`, `aio_multi_poll`, etc.).

---

## 💀 Complete List of Dead Ends (What is Patched)

We spent weeks testing and documenting the following vectors, all of which are dead on 12.52:

* **Event Flag UAF (Syscalls 544/548):** We proved that writing to offsets `0x0`, `0x8`, `0x10`, `0x80`, `0x90`, and `0x400` does not reflect back in the Syscall 638 read. Heap grooming and slab reclamation (UMA allocator) prevent overlapping.
* **IPv6 NetControl:** `setsockopt` with `IPV6_PKTOPTIONS`, `HOPOPTS`, `DSTOPTS`, `RTHDR`, and `NEXTHOP` return `EPERM` or `EINVAL`. The kernel filters these structures at the syscall level. `IPV6_TCLASS` and `IPV6_PKTINFO` are accepted but are no-ops for escalation.
* **Lua 5.3 CVEs:** Fully patched. We tested CVE-2020-24370 (`string.rep`), CVE-2020-24371 (coroutines), CVE-2021-43519 (`table.sort`), CVE-2022-28805 (`string.format`), etc. All survived execution.
* **Driver IOCTLs:** `/dev/gc`, `/dev/dce`, `/dev/dmem0-2`, and `/dev/dbggc`. All return `-1` for generic ioctls (`FIONBIO`, `FIONREAD`) and custom commands (`0xC118850F`). The filter is in the kernel, not the Lua wrapper.
* **Syscall 251 (rfork):** Crashes the console with a SIGSEGV at `RIP: 0x301`. The registers `r14` and `r15` are controllable (`0x400042`), but the jump address (`0x301`) is hardcoded and not controllable via userland memory.

---

## 🔬 The Promising Vector: AIO Multi-Cancel (Syscall 601)

Despite the dead ends, we discovered a deterministic crash in `aio_submit` (Syscall 601, Mode 3).

* Calling `601(3, ...)` immediately crashes the application (SIGSEGV in userland, **not** a kernel panic).
* This happens regardless of the arguments passed (valid `aiocb` pointers, arrays of pointers, NULL, or kernel addresses).
* This indicates a missing validation check in the kernel's AIO submission queue (likely reading a userland pointer with SMAP enabled, or dereferencing a null pointer).
* **The Goal:** By controlling the structure passed to Mode 3 (an array of `aiocb` pointers), it may be possible to trigger a Use-After-Free (UAF) in the kernel and reclaim the freed slot with a controlled object (using the `IPV6_RTHDR`/`DSTOPTS` spray), achieving Kernel R/W.

*Note: The public `Poops` (NetControl/AIO) exploit by TheFlow covers this exact logic, but the source code and offsets for 12.52 remain private.*

---

## 🛠️ Repository Contents

* **`index.html`**: Interactive, dark-mode HTML report with all the code and findings.
* **`/scripts`**: Contains the raw Lua scripts developed during this research:
  * `01_recon.lua`: Environment verification and base addresses.
  * `02_kaslr_leak.lua`: Kernel ASLR bypass.
  * `03_syscall_prober.lua`: Bruteforce of Syscalls 500-620.
  * `04_aio_investigate.lua`: Forensics and crash logging for AIO Mode 3.
  * `05_driver_fuzzer.lua`: IOCTL probing on custom `/dev/` nodes.
  * `06_heap_grooming_tests.lua`: UAF and slab reclamation attempts.

---

## 🚀 Future Work (How to continue)

If you want to finish this project, you need three things:
1. **The Poops ELF/Source:** Find the compiled native payload (C/ELF) for the AIO race condition.
2. **Kernel Offsets for 12.52:** We need `EVF_OFFSET`, `PRISON0`, and `ROOTVNODE`. These can be found by diffing `kernel_1252.elf` against older firmwares.
3. **WebKit Entry Point:** To load the native ELF, a WebKit userland exploit or a working BD-J image is required.

---

## 🙏 Credits

* **shahrilnet** for the `remote_lua_loader` and `lapse.lua` framework.
* **TheFlow** & **ChendoChap** for the original AIO/NetControl exploit research.
* **Tulip 🌷** (AI Co-pilot) for assisting with the code generation, vulnerability analysis, and documentation.

---
*"The room is dark again, but it's no longer empty. It has our map."*
