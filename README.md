# WhatsApp OpenBox (OBI) Native Memory-Safety Research

Reverse-engineering analysis of a native memory-safety issue identified in WhatsApp for Android's OpenBox (OBI) decompression path.

## Overview

This research analyzes `libsuperpack.so`, the native library responsible for parts of WhatsApp's Superpack/OpenBox handling.

The analysis identified a potential native out-of-bounds heap write caused by unchecked size propagation during OBI decompression.

A length value derived from OBI metadata reaches a `memcpy()` operation without an apparent validation against the destination buffer capacity.

The relevant native path was reconstructed through static reverse engineering and data-flow analysis.

## Affected Version

- **Application:** WhatsApp for Android
- **Version analyzed:** `2.26.10.72`
- **Native library:** `libsuperpack.so`
- **Component:** OpenBox / OBI decompression

## High-Level Data Flow

```text
WhatsAppObiInputStream.openBytes()
            │
            ▼
openBytesNative(byte[], int, int)
            │
            ▼
      fcn.00009234
            │
            ▼
      fcn.0000aef0
            │
            ▼
          funopen()
            │
            ▼
      fcn.0000af94
            │
            ▼
          memcpy()

The analysis focused on the propagation of the OBI-related length value through this native call chain.
Java → Native Interface
WhatsAppObiInputStream exposes several native methods, including:
public static native long openBytesNative(
    byte[] bArr,
    int i,
    int i2
);

public static native long openObArchiveBytesNative(
    long j,
    long j2,
    int i
);

The openBytes() method validates the Java byte-array bounds before passing the selected region to the native implementation.
The security-relevant question is therefore not the Java array-bound check itself, but how the supplied OBI data and its metadata are subsequently handled inside the native implementation.
See:
WhatsAppObiInputStream.java
OpenBox Archive Interface
WhatsAppOpenboxArchive provides the native archive interface used to access OpenBox archive data.
Relevant native methods include:
public static native long openNative(int i, long j, long j2);

public static native long getFilePtrNative(long j, int i);

public static native long getFileSizeNative(long j, int i);

The archive reader is initialized using the file descriptor, start offset, and length obtained from the Android AssetFileDescriptor.
See:
WhatsAppOpenboxArchive.java
Native Analysis
The relevant native function was identified as:
fcn.0000af94

Within this function, the analysis identified the following ARM64 instructions:
mov x0, x20
sxtw x2, w21
add x1, x9, x8
bl sym.imp.memcpy

This corresponds to a memcpy() invocation where the third argument is derived from w21.
The analysis traced the relevant value back through the OBI decompression path and identified it as originating from OBI metadata.
The destination capacity is not apparently validated against this propagated length before the copy.
Root Cause
The suspected root cause is unchecked size propagation into a native memory-copy operation.
Conceptually:
OBI metadata
     │
     ▼
 length value
     │
     ▼
 native decompression path
     │
     ▼
 memcpy(destination, source, length)

The analysis indicates that the length can reach memcpy() without an apparent bounds check against the destination capacity.
This creates a potential native out-of-bounds heap write.
JNI Registration
The analysis also reconstructed the JNI registration path.
JNI_OnLoad_Weak at 0x8f30 branches to:
fcn.00008fa0

The registration code references:
com/whatsapp/superpack/WhatsAppObiInputStream

and registers its native methods.
Other relevant strings identified during analysis included:
failed to create obi handler
failed to open obi file handler
Openbox cannot decompress, models (%d, %d) required...
Could not read openbox archive
com/whatsapp/superpack/WhatsAppOpenboxArchive
Not an OB file

These references were used to correlate the Java interfaces with the native OpenBox implementation.
Relevant Native Call Chain
The reconstructed path includes:
fcn.00009234
    │
    └── fcn.0000aef0
             │
             └── funopen()
                    │
                    └── fcn.0000af94
                             │
                             └── memcpy()

fcn.0000aef0 installs fcn.0000af94 as the funopen() read callback.
Relevant instructions:
adr x1, fcn.0000af94
adr x4, fcn.0000ae94
mov x2, xzr
mov x3, xzr
mov x20, x0
bl sym.imp.funopen

Impact
The identified primitive is a native heap out-of-bounds write.
Potential impact discussed in the research includes:
- Memory corruption
- Reliable denial-of-service
- Potential code execution under favorable heap conditions
The research did not demonstrate a working standalone RCE exploit.
Attack / Delivery Considerations
The analysis identified a path involving OBI/OpenBox data, but delivering attacker-controlled OBI content through the relevant production WhatsApp asset/update mechanisms is non-trivial.
The report therefore distinguishes the existence of the native memory-safety issue from the practical requirements for delivering malicious data to the vulnerable code path.
No abnormal invocation or developer-only interface was required for the analyzed code path.
Methodology
The analysis was performed through static reverse engineering of the official WhatsApp Android APK and its extracted:
libsuperpack.so

Techniques included:
- ARM64 disassembly
- Native call-chain reconstruction
- JNI registration analysis
- String and cross-reference analysis
- Data-flow tracing
- Java-to-native interface analysis
- Native memory-operation analysis
The analysis was performed using reverse-engineering tooling including Ghidra/radare2.
Limitations
This research is primarily based on static reverse engineering and data-flow analysis.
A standalone malicious OBI proof-of-concept was not demonstrated because reproducing the relevant production asset-delivery conditions is non-trivial.
No working remote code execution exploit is included in this repository.
Therefore, this repository should not be interpreted as a complete weaponized exploit.
Repository Contents
.
├── README.md
├── WhatsAppObiInputStream.java
└── WhatsAppOpenboxArchive.java

The Java files are decompiled representations of the relevant WhatsApp classes and are included to document the Java/native boundary analyzed during the research.
Disclosure
This research was submitted to Meta Security as a vulnerability report.
The report was reviewed by Meta Security and the issue was confirmed as a valid security issue. The report was determined to be non-payable under Meta's bug bounty criteria.
This repository is intended for security research, reverse-engineering, and educational purposes.
Disclaimer
This repository documents security research and analysis of third-party software.
It does not contain proprietary WhatsApp native binaries, a working exploit, or instructions intended to compromise WhatsApp users.
Use this material only for lawful security research and analysis.
