# PS4 12.52 Sandbox Audit & Exploit Research

**Status:** Archived / Research Phase
**Firmware Target:** PS4 12.52
**Environment:** `remote_lua_loader` (Lua 5.1)
**Authors:** [Your Name/Handle] & Tulip 🌷

## 📌 Overview

This repository contains the complete documentation, scripts, and findings from an intensive research project targeting the PlayStation 4 firmware **12.52**. The goal was to discover a new kernel exploit from scratch using only the `remote_lua_loader` as an entry point, completely avoiding WebKit and BD-J.

While a full jailbreak (Kernel R/W -> GoldHEN) was not achieved due to missing public kernel offsets and primitives for this specific firmware, this repository serves as a comprehensive **map of the sandbox**. It documents exactly what works, what is patched, and what vectors are still theoretically viable for future researchers.

## 🏆 Achievements (What Works)

Despite the lack of a kernel exploit, we successfully achieved and mapped the following on FW 12.52:

1. **Complete Userland R/W:** Using Lua 5.1 primitives (`memory.read_buffer`, `memory.write_buffer`).
2. **Object Manipulation:** `addrof`, `fakeobj`, and `create_fake_cclosure` are fully functional.
3. **KASLR Bypass:** Successfully leaked kernel pointers and mapped fixed bases.
   * Kernel Leak: `0xffffffffb8fd86f0`
   * `libkernel.sprx`: `0x800000000`
   * `libc.prx`: `0x80190000`
4. **Sandbox Mapping:** Fully documented the Event Flag family (Syscalls 538, 539, 544, 548, 638).
5. **Syscall Prober:** Mapped 200+ syscall wrappers and tested their argument validation.

## 💀 Dead Ends (What is Patched)

We spent weeks testing and documenting the following vectors, all of which are dead on 12.52:

* **Event Flag UAF:** Syscalls 544 and 548 do not write to the allocated object. Heap grooming and slab reclamation (UMA) prevent overlapping.
* **IPv6 NetControl:** `setsockopt` with `IPV6_PKTOPTIONS`, `HOPOPTS`, `DSTOPTS`, and `RTHDR` return `EPERM` or `EINVAL`. The kernel filters these structures at the syscall level.
* **Lua 5.3 CVEs:** `string.rep`, `table.sort`, `coroutines`, and `string.format` exploits (CVE-2020-24370, CVE-2021-43519, etc.) are fully patched by Sony's custom Lua implementation.
* **Driver IOCTLs:** `/dev/gc`, `/dev/dce`, and `/dev/dmem` return `-1` for all commands. The filter is in the kernel, not the wrapper.

## 🔬 The Promising Vector: AIO Multi-Cancel (Syscall 601/662)

We discovered a deterministic crash in `aio_submit` (Mode 3). 

* Calling `601(3, ...)` immediately crashes the application (SIGSEGV in userland, **not** a kernel panic).
* This indicates a missing validation check in the kernel's AIO submission queue (likely reading a userland pointer with SMAP enabled, or dereferencing a null pointer).
* **The Goal:** By controlling the structure passed to Mode 3 (an array of `aiocb` pointers), it may be possible to trigger a Use-After-Free (UAF) in the kernel and reclaim the freed slot with a controlled object, achieving Kernel R/W.

*Note: The public `Poops` (NetControl/AIO) exploit by TheFlow covers this exact logic, but the source code and offsets for 12.52 remain private.*

## 🛠️ Repository Contents

* `/scripts/`: Contains all the Lua scripts developed during this research (Leak scripts, Driver fuzzers, AIO probes).
* `/dumps/`: Contains memory dumps and structure analysis.
* `/docs/`: Detailed write-ups on specific sandbox mechanics and failures.

## 🚀 Future Work (How to continue)

If you want to finish this project, you need three things:
1. **The Poops ELF/Source:** Find the compiled native payload (C/ELF) for the AIO race condition.
2. **Kernel Offsets for 12.52:** We need `EVF_OFFSET`, `PRISON0`, and `ROOTVNODE`. These can be found by diffing `kernel_1252.elf` against older firmwares.
3. **WebKit Entry Point:** To load the native ELF, a WebKit userland exploit or a working BD-J image is required.

## 🙏 Credits

* **shahrilnet** for the `remote_lua_loader` and `lapse.lua` framework.
* **TheFlow** & **ChendoChap** for the original AIO/NetControl exploit research.
* **Tulip 🌷** (AI Co-pilot) for assisting with the code generation, vulnerability analysis, and documentation.

---
*“The room is dark again, but it's no longer empty. It has our map.”*
