---
layout: post
title: "Porting Local Privilege Escalation Research"
date: 2026-07-22
category: android
tags: [android, kernel, lpe, mali, selinux, offsets]
description: "A deep dive into the challenges of porting Android kernel LPE research from a Google Pixel to a Realme 6, highlighting OEM symbol changes and static versus dynamic analysis."
---

# Porting Local Privilege Escalation Research

If you spend enough time in the Android security community, you’ll notice a recurring theme: most local privilege escalation (LPE) proof-of-concepts (PoCs) are written for Google Pixels. They are the standard testbeds. Their kernels are close to AOSP, the offsets are predictable, and the symbol names are consistent. 

But what happens when you take a well-documented Pixel PoC and point it at a mid-range device like a Realme 6? I learned firsthand while analyzing [*Man Yue Mo’s excellent Mali GPU research*](https://github.blog/author/mymo/) that the transition is rarely straightforward. 

What started as a task of simply updating a few memory offsets turned into a much steeper learning curve than I anticipated. Here is a breakdown of how I traced the necessary kernel symbols across three different approaches, and how a missing letters—`avc_deny` vs `avc_denied`—taught me a lot about OEM kernel modifications.

## The Destination vs. The Journey

Early on, I spent a lot of time focused purely on the *primitive*—the Mali GPU bug itself. I was trying to wrap my head around the exact mechanism of the memory corruption. 

Then, I read the GitHub Security Lab’s great article, [*Corrupting Memory Without Memory Corruption*](https://github.blog/security/vulnerability-research/corrupting-memory-without-memory-corruption/). It really changed how I looked at the problem. The article highlights that many modern vulnerabilities don't require executing complex ROP chains or injecting arbitrary code; they simply rely on manipulating critical kernel state variables. 

It started to click for me that the Mali GPU bug was just the *primitive*. The *destination* was always the same. To demonstrate a full LPE on Android, regardless of the specific vulnerability, research eventually converges on a core set of kernel symbols:

1.  **`init_cred`**: An extremely useful target. This is the `struct cred` belonging to PID 1 (init). It holds `uid=0`, `gid=0`, and the SELinux context `u:r:su:s0`. If a memory corruption primitive can make the current task's credential pointer point to `init_cred`, it achieves root access and mitigates SELinux restrictions in a single write.
2.  **`commit_cred`**: The traditional path. If you can't overwrite a pointer, you might have to construct forged credentials in a controlled buffer and pass them to this function. 
3.  **`selinux_enforcing`** (sometimes `selinux_enforce`): The global toggle. Even if you elevate to `uid=0` via `commit_cred`, SELinux will still restrict access to sensitive application data. Flipping this integer from `1` to `0` puts the phone into Permissive mode. 
4.  **`avc_deny` / `avc_denied`**: Another common research target. If you can't locate the enforcing toggle, you can overwrite the SELinux Access Vector Cache (AVC) denial hook so it simply returns `0` (success) every time the kernel checks permissions.

My goal was clear: I was working with the Mali write primitive. I just needed to figure out how to find the runtime addresses of these symbols on the Realme 6. 

## Angle 1: The Source Code Illusion (GitHub)

Like many people would, I started with the source code. I tracked down the Realme 6 kernel source tree on GitHub (a heavily modified MediaTek 4.14 fork). 

I searched for `init_cred`, `commit_cred`, and `selinux_enforcing`. I found them, calculated the offsets based on the compiled vmlinux, applied what I thought was the KASLR slide, and executed the PoC.

*Kernel Panic. Not syncing: Fatal exception.*

I quickly realized that OEM kernel sources on GitHub can be misleading. To comply with GPL, manufacturers release "close enough" code. The source tree I was looking at didn't perfectly match the exact binary running on my specific Realme 6 firmware patch level. MediaTek had applied out-of-tree patches, and different compiler optimizations had shifted the layout of the kernel. The offsets derived purely from the GitHub source were invalid.

## Angle 2: The Static Reality (The Boot Image & `nm`)

Since the source code wasn't an exact match, I had to look at the truth: the actual `boot.img` running on the device. 

Instead of loading a heavy GUI disassembler just to check a symbol name, I opted for a simpler command-line approach. I extracted the raw uncompressed Image from the Realme 6 `boot.img` and converted it into a standard ELF format. Once it was an ELF, I had access to the standard Unix binutils toolkit. I ran `nm` to dump the symbol table and piped it to `grep`:

```bash
nm vmlinux.elf | grep avc
```

This is where I hit the most frustrating roadblock of the entire process. I was looking for `avc_deny`—the exact symbol name used in the Pixel PoC. But `nm` returned nothing. I knew the function had to exist; the kernel wouldn't boot without SELinux. 

I broadened my grep and finally saw it:

`ffffffff81234560 T avc_denied`

A missing 'd' at the end of the word. 

This was a crucial realization for me. Realme/BBK or MediaTek had modified the SELinux subsystem, likely backporting a specific patch or just shifting to a slightly different naming convention for their out-of-tree hooks. In C, `avc_deny` and `avc_denied` are entirely different entities. If the original PoC used a hardcoded string like `kallsyms_lookup_name("avc_deny")`, it would return `0x0` on the Realme 6. The code would blindly attempt to write to address `0x0`, causing an instant null-pointer dereference kernel panic. 

A quick second check confirmed another OEM quirk: `selinux_enforcing` was actually exported as `selinux_enforce`. The GitHub source code had misled me, but the raw binary and `nm` showed me what was actually there.

## Angle 3: The Dynamic Truth (The Rooted Device)

I now knew the exact symbol names present in the Realme 6 binary. But I still needed their runtime addresses, which are randomized by KASLR on every boot.

To get these, I relied on a rooted Realme 6. By gaining a temporary root shell, I could access `/proc/kallsyms`—the kernel's runtime symbol table.

I wrote a quick script to grep the exact symbols I found via `nm`:
*   `ffffffffa8c00120 T init_cred`
*   `ffffffffa8b456a0 T commit_cred`
*   `ffffffffa8d00200 D selinux_enforce`
*   `ffffffffa8c98b40 T avc_denied`

By calculating the difference from a known base address, I could figure out the KASLR slide for that specific boot cycle. 

## Bridging the Gap: The Final Implementation

With all the pieces in place—the Mali GPU arbitrary read/write primitive and the dynamic addresses of the LPE symbols—I could finally integrate these findings. 

Keeping the GitHub Security Lab article in mind, I bypassed `commit_cred` entirely. Constructing a fake `cred` struct and routing execution through a ROP chain can be fragile. Instead, I used the Mali primitive to:

1. Find my current task struct.
2. Locate the `cred` pointer within it.
3. Overwrite that pointer with the dynamic address of `init_cred`.

With that, the execution context was elevated to `uid=0` with the `u:r:su:s0` SELinux context. No ROP chain required, just direct state manipulation.

However, to make the PoC robust for future boots (where the KASLR slide changes and `kallsyms` might be restricted), I couldn't hardcode the symbol names. I had to write a fuzzy resolver that accounted for both the standard AOSP names and the modified OEM names I found using `nm`.

Here is a simplified version of the resolver I implemented:

```c
unsigned long find_symbol(const char* name_a, const char* name_b, unsigned long fallback_offset, unsigned long base_addr) {
    unsigned long addr = 0;

    // 1. Try the standard AOSP/Pixel name
    addr = kallsyms_lookup_name(name_a);
    if (addr > 0) return addr;

    // 2. Try the OEM modified name (e.g., avc_deny vs avc_denied)
    if (name_b != NULL) {
        addr = kallsyms_lookup_name(name_b);
        if (addr > 0) return addr;
    }

    // 3. Fallback: If kallsyms is restricted, use a stable neighbor 
    // symbol and apply the offset we found via 'nm' in static analysis
    if (fallback_offset > 0 && base_addr > 0) {
        return base_addr + fallback_offset;
    }

    return 0; // Fail gracefully
}

// Usage in the PoC setup:
unsigned long init_cred_addr = find_symbol("init_cred", NULL, 0, 0);
unsigned long avc_denied_addr = find_symbol("avc_deny", "avc_denied", AVC_DENIED_OFFSET_FROM_BASE, kaslr_base);
```

## Lessons Learned

My main takeaway from this whole process is that porting vulnerability research from a Pixel to an OEM device isn't just a 1:1 translation of hex values. It requires understanding the intent of the target kernel. 

I realized that relying solely on released source code can be misleading. I also learned that SoC vendors like MediaTek will silently rename critical security functions—changing a simple `avc_deny` to `avc_denied`. But most importantly, I learned that if you understand the core mechanics of Linux privilege escalation—targets like `init_cred` and `selinux_enforce`—you can adapt your research methodology to work around whatever obscure modifications the OEM decided to implement.