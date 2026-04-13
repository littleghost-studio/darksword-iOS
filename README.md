DOCUMENTATION AND EDUCATIONAL PURPOSES ONLY


DARKSWORD iOS 18 ANALYSIS

Dump of everything I could find for darkword iOS 18 exploit. 
Content creds go to respective creators.

My analysis of DarkSword:

DARKSWORD iOS MALWARE COMPREHENSIVE TECHNICAL ANALYSIS

EXECUTIVE SUMMARY
DarkSword represents a highly sophisticated, multi-stage iOS exploit chain that achieves zero-click remote code execution through WebKit vulnerabilities. This analysis provides an in-depth examination of the malware's internal architecture, exploitation techniques, file interconnections, and the underlying iOS security mechanisms it bypasses.

PART I: MALWARE ARCHITECTURE AND FILE INTERCONNECTIONS

1.1 DELIVERY VECTOR AND INITIAL INFRASTRUCTURE

The infection chain begins with two HTML files that establish the initial attack surface:

index.html serves as the primary delivery mechanism:
- Creates a session UID via sessionStorage.setItem('uid', '1') for tracking
- Dynamically generates a hidden iframe with anti-detection properties:
  * 1px x 1px dimensions
  * 0.01 opacity for near-invisibility
  * Positioned at -9999px left coordinate
  * No border styling
- Appends iframe to document.body, triggering the secondary payload

frame.html acts as the secondary loader:
- Contains a single script tag that dynamically loads rce_loader.js from static.cdncounter.net
- Uses document.write with timestamp-based cache busting
- Serves as the bridge between the initial page and the exploit infrastructure

The CDN infrastructure (static.cdncounter.net) serves multiple purposes:
- Primary payload delivery mechanism
- Evades static analysis through dynamic loading
- Provides infrastructure resilience and load balancing
- Enables rapid payload updates without modifying the initial HTML

1.2 CORE EXPLOITATION FRAMEWORK

rce_loader.js functions as the central orchestrator:
- Implements comprehensive iOS version detection through user agent parsing
- Maintains logging infrastructure for exploit success tracking
- Coordinates the loading and execution of version-specific exploit modules
- Manages Web Worker creation and communication channels

Version detection mechanism:
```javascript
const ios_version = (function() {
    let version = /iPhone OS ([0-9_]+)/g.exec(navigator.userAgent)?.[1];
    if (version) {
        return version.split('_').map(part => parseInt(part));
    }
})();
```

This regex-based parsing extracts major, minor, and patch versions, enabling precise payload selection.

Dynamic payload loading strategy:
- iOS 18.6/18.6.1/18.6.2: Loads rce_worker_18.6.js and rce_module_18.6.js
- Other versions (primarily 18.4): Loads rce_worker_18.4.js and rce_module.js
- Timestamp-based cache busting prevents CDN caching issues

1.3 WEB WORKER ARCHITECTURE

The exploit leverages Web Workers for isolation and parallel execution:

Primary exploit worker (rce_worker.js/rce_worker_18.6.js):
- Executes in isolated thread context, evading some security monitoring
- Implements sophisticated type confusion primitives
- Contains the core memory corruption exploitation logic
- Communicates with main thread via postMessage API

DLOpen workers (auxiliary):
- Created in pairs for synchronization primitives
- Used to trigger specific memory conditions
- Assist in heap grooming and memory layout manipulation
- Controlled through precise message passing coordination

Worker communication protocol:
- Structured message types: 'init', 'dlopen', 'stage1_rce', 'setup_fcall'
- Synchronization through message acknowledgments
- Error handling and fallback mechanisms

1.4 EXPLOIT MODULE INTERCONNECTIONS

rce_module.js contains the exploitation primitives and offset database:
- Extensive offset tables for multiple iOS versions and device models
- Kernel function addresses and gadget locations
- Memory layout information for precise exploitation
- Version-specific exploitation parameters

Offset database structure:
```javascript
rce_offsets = {
   "iPhone11,2_4_6_22F76": {
      AVFAudio__AVLoadSpeechSynthesisImplementation_onceToken: 0x1edadec98n,
      JavaScriptCore__globalFuncParseFloat: 0x19a8715ecn,
      WebCore__DedicatedWorkerGlobalScope_vtable: 0x1f137cf70n,
      // ... hundreds more offsets
   }
   // ... more device/version combinations
}
```

Each device/version combination contains:
- Symbol addresses for dynamic resolution
- Gadget locations for ROP chain construction
- Memory structure offsets for precise manipulation
- Function pointers for privilege escalation

PART II: WEBKIT AND JAVASCRIPTCORE EXPLOITATION

2.1 TYPE CONFUSION FUNDAMENTALS

The exploit leverages sophisticated type confusion techniques in JavaScriptCore:

Array type manipulation:
```javascript
const no_cow = 1.1;
const unboxed_arr = [no_cow];
const boxed_arr = [{}];
```

This creates two arrays with different internal representations:
- unboxed_arr: Contains double values, stored as raw binary data
- boxed_arr: Contains object references, stored as pointers

The exploit abuses JavaScriptCore's internal array representations:
- JSArray::IndexingType determines storage format
- Butterfly structure manages array elements
- Structure transitions enable type confusion

2.2 MEMORY CORRUPTION PRIMITIVES

The exploit establishes arbitrary read/write capabilities through:

ArrayBuffer manipulation:
- Uses ArrayBuffer as controlled memory regions
- Leverages TypedArray views for different data interpretations
- Exploits JavaScriptCore's garbage collection mechanisms

BigInt type confusion:
```javascript
const ab = new ArrayBuffer(8);
const u64 = new BigUint64Array(ab);
const f64 = new Float64Array(ab);

BigInt.fromDouble = function(v) { f64[0] = v; return u64[0]; };
BigInt.prototype.asDouble = function() { u64[0] = this; return f64[0]; };
```

This enables:
- Arbitrary pointer manipulation
- Data type conversion for exploitation
- Bypass of JavaScript type restrictions

2.3 PAC (POINTER AUTHENTICATION) BYPASS

iOS implements Pointer Authentication to prevent code reuse attacks. DarkSword bypasses this through:

PAC stripping techniques:
```javascript
BigInt.prototype.noPAC = function() { 
    return this & 0x7fffffffffn; 
};
```

This removes the PAC signature from pointers, enabling:
- Direct pointer manipulation
- Gadget chain construction
- Function pointer corruption

PAC gadget exploitation:
- Uses legitimate PAC-signed gadgets
- Leverages kernel PAC validation mechanisms
- Constructs ROP chains with valid PAC signatures

2.4 ARBITRARY READ/WRITE ESTABLISHMENT

The exploit builds sophisticated memory manipulation primitives:

Read primitive:
```javascript
function uread64(address) {
    // Type confusion to read arbitrary memory
    // Returns 64-bit value from specified address
}
```

Write primitive:
```javascript
function uwrite64(address, value) {
    // Type confusion to write arbitrary memory
    // Writes 64-bit value to specified address
}
```

These primitives enable:
- Kernel memory modification
- Function pointer overwrites
- Security feature bypasses

PART III: KERNEL EXPLOITATION AND SANDBOX ESCAPE

3.1 SHARED CACHE SLIDE CALCULATION

iOS uses Address Space Layout Randomization (ASLR) with shared cache sliding. DarkSword bypasses this through:

Slide calculation mechanism:
```javascript
function get_shared_cache_slide() {
    let start_address = gpu_new_uint64_t();
    gpu_fcall(func_resolve("syscall"), 294n, 0n, 0n, 0n, 0n, 0n, 0n, 0n, start_address);
    let DYLD_SHARED_CACHE_LOAD_ADDR = 0x0000000180000000n;
    let dyld_shared_cache_slide = uread64(start_address) - DYLD_SHARED_CACHE_LOAD_ADDR;
    return dyld_shared_cache_slide;
}
```

This uses syscall 294 (likely mach_vm_region_recurse_64) to:
- Locate the dyld shared cache in memory
- Calculate the slide offset from the base address
- Enable precise address calculation for all symbols

3.2 MACH PORT MANIPULATION

The exploit leverages Mach IPC for privilege escalation:

Port allocation and manipulation:
```javascript
offsets.mach_port_allocate = dlsym(libsystem_kernel, 'mach_port_allocate').noPAC();
offsets.mach_port_insert_right = dlsym(libsystem_kernel, 'mach_port_insert_right').noPAC();
offsets.mach_msg_fn = dlsym(libsystem_kernel, 'mach_msg').noPAC();
```

This enables:
- Creation of malicious Mach ports
- Port rights manipulation for privilege escalation
- Inter-process communication exploitation

3.3 GPU PROCESS EXPLOITATION

iOS separates rendering into GPU processes for security. DarkSword compromises this through:

GPU process targeting:
- Extensive GPU process offset tables in sbx0_main_18.4.js
- Remote rendering backend exploitation
- Graphics context manipulation

Critical GPU targets:
```javascript
RemoteGraphicsContextGL_AttachShader: 0x40c,
RemoteGraphicsContextGL_BindBuffer: 0x411,
RemoteGraphicsContextGL_CreateTexture: 0x447,
RemoteRenderingBackend_CreateImageBuffer: 0x5ac,
```

This enables:
- Sandbox escape through GPU process compromise
- Code execution in higher-privilege context
- System-wide compromise

3.4 SANDBOX ESCAPE TECHNIQUES

The malware implements multiple sandbox escape strategies:

sbx0_main_18.4.js provides the initial sandbox escape:
- Targets GPU process vulnerabilities
- Leverages WebKit's multiprocess architecture
- Escapes Safari sandbox into GPU process

sbx1_main.js extends the escape:
- Targets system-level privileges
- Implements kernel exploitation
- Achieves root privileges

PART IV: POST-EXPLOITATION AND PERSISTENCE

4.1 PRIVILEGE ESCALATION (pe_main.js)

pe_main.js implements comprehensive post-exploitation capabilities:

System function resolution:
```javascript
let CALLOC = func_resolve("calloc");
let MALLOC = func_resolve("malloc");
let MACH_VM_ALLOCATE = func_resolve("mach_vm_allocate");
let OBJC_MSGSEND = func_resolve("objc_msgSend");
```

This provides access to:
- Memory management functions
- Mach kernel APIs
- Objective-C runtime functions

Objective-C runtime manipulation:
```javascript
function objc_getClass(class_name) {
    return fcall(OBJC_GETCLASS, get_cstring(class_name));
}

function objc_msgSend(...args) {
    return fcall(OBJC_MSGSEND, ...args);
}
```

Enables:
- Dynamic class loading
- Method invocation
- Runtime manipulation

4.2 DYLD SHARED CACHE PATCHING

The malware achieves persistence through dyld manipulation:

Runtime patching:
- Modifies dyld shared cache in memory
- Patches system functions for stealth
- Maintains persistence across reboots

Function hooking:
- Intercepts system calls
- Hides malicious activity
- Maintains covert access

4.3 STEALTH AND ANTI-ANALYSIS

DarkSword implements multiple stealth techniques:

Hidden delivery:
- Invisible iframe prevents user detection
- Minimal network footprint
- Automatic redirect to 404 page

Anti-analysis:
- Dynamic payload loading prevents static analysis
- Version-specific targeting complicates analysis
- Extensive obfuscation in JavaScript code

Logging and tracking:
- Comprehensive logging infrastructure
- Success/failure tracking
- Remote telemetry capabilities

PART V: iOS SECURITY BYPASS ANALYSIS

5.1 WEBKIT SECURITY MECHANISMS BYPASSED

DarkSword bypasses multiple WebKit security features:

Same-Origin Policy:
- Uses iframe isolation to exploit cross-origin vulnerabilities
- Leverages postMessage API for controlled communication
- Bypasses CORS restrictions through careful message crafting

Content Security Policy:
- Avoids inline script restrictions through dynamic loading
- Bypasses script-src limitations using CDN delivery
- Evades CSP through iframe isolation

JavaScript Engine Protections:
- Bypasses JIT hardening through type confusion
- Circumvents array bounds checking
- Defeats pointer authentication through gadget chains

5.2 iOS KERNEL SECURITY BYPASSED

The malware defeats multiple kernel security mechanisms:

Kernel Address Space Layout Randomization (KASLR):
- Shared cache slide calculation reveals kernel layout
- Symbol resolution enables precise targeting
- Offset database provides comprehensive coverage

Pointer Authentication (PAC):
- PAC stripping removes authentication signatures
- Gadget chains use legitimate PAC-signed code
- Bypasses kernel code signing requirements

System Integrity Protection (SIP):
- dyld manipulation bypasses SIP protections
- Runtime patching modifies system behavior
- Kernel exploitation defeats SIP boundaries

5.3 SANDBOX BYPASS TECHNIQUES

DarkSword defeats iOS sandboxing through:

Process Isolation Bypass:
- GPU process exploitation breaks isolation boundaries
- Mach port manipulation enables inter-process compromise
- Shared memory regions facilitate data exfiltration

Resource Restrictions Bypass:
- File system access through kernel exploitation
- Network access through compromised system processes
- Hardware access through GPU process compromise

Permission Escalation:
- Safari sandbox escape to user context
- User to root privilege escalation
- System-wide compromise achieved

PART VI: TECHNICAL IMPLEMENTATION DEEP DIVE

6.1 EXPLOIT TIMING AND SYNCHRONIZATION

The malware implements precise timing control:

Worker synchronization:
```javascript
async function prepare_dlopen_workers() {
    for (let i = 1; i <= 2; ++i) {
        const worker = new Worker(dlopen_worker_url);
        dlopen_workers.push(worker);
        await new Promise(r => {
            worker.postMessage({
                type: 'init',
                data: 0x11111111 * i
            });
            worker.onmessage = r;
        });
    }
}
```

This ensures:
- Proper initialization order
- Synchronization between exploit stages
- Reliable exploitation conditions

6.2 MEMORY LAYOUT MANIPULATION

The exploit carefully controls memory layout:

Heap grooming:
- Array allocation patterns control heap structure
- Object placement influences exploitation success
- Garbage collection timing is precisely controlled

Structure exploitation:
```javascript
const no_cow = 1.1;
const unboxed_arr = [no_cow];
const boxed_arr = [{}];

self[0] = unboxed_arr;
self[1] = boxed_arr;
```

This creates predictable memory layouts for exploitation.

6.3 EXPLOIT RELIABILITY MECHANISMS

DarkSword includes multiple reliability features:

Fallback mechanisms:
- Multiple exploitation paths for different scenarios
- Error handling and recovery procedures
- Adaptive exploitation based on system state

Version compatibility:
- Comprehensive offset database
- Dynamic payload selection
- Graceful degradation for unsupported versions

Exploit success verification:
- Extensive logging and telemetry
- Success/failure tracking
- Remote monitoring capabilities

PART VII: DETECTION AND MITIGATION ANALYSIS

7.1 DETECTION CHALLENGES

DarkSword presents significant detection challenges:

Stealth techniques:
- Hidden iframe prevents visual detection
- Minimal network footprint evades network monitoring
- Dynamic loading prevents static analysis

Anti-forensics:
- Automatic redirect to 404 page
- Comprehensive logging removal
- Memory-only execution

Evasion capabilities:
- Version-specific targeting complicates signature creation
- Obfuscated JavaScript code defeats pattern matching
- CDN-based delivery enables rapid infrastructure changes

7.2 MITIGATION LIMITATIONS

Current iOS protections have limitations against this threat:

WebKit protections:
- Type confusion vulnerabilities remain exploitable
- PAC bypasses defeat pointer authentication
- Sandbox escape mechanisms remain effective

Kernel protections:
- KASLR bypass through shared cache calculation
- dyld manipulation defeats runtime protections
- GPU process exploitation breaks isolation boundaries

Network protections:
- CDN-based delivery evades blocklists
- HTTPS encryption prevents content inspection
- Dynamic URLs prevent static filtering

7.3 DEFENSE RECOMMENDATIONS

Enhanced protection strategies:

Runtime monitoring:
- Monitor for hidden iframe creation
- Detect unusual Web Worker activity
- Track abnormal memory allocation patterns

Network security:
- Implement dynamic URL filtering
- Monitor CDN traffic patterns
- Deploy TLS inspection where possible

System hardening:
- Regular iOS updates to patch vulnerabilities
- JavaScript disabled in high-risk environments
- Mobile threat detection solutions

CONCLUSION

DarkSword represents a sophisticated, well-engineered iOS exploit chain that demonstrates:
- Deep understanding of iOS internals and security mechanisms
- Advanced exploitation techniques across multiple layers
- Comprehensive bypass of modern security protections
- Professional-grade malware development practices

The malware's modular design, version-specific targeting, and extensive stealth features indicate a well-resourced development effort with ongoing maintenance. Its ability to achieve zero-click remote code execution and full system compromise makes it a significant threat to iOS security.

Security researchers and defenders should focus on:
- Developing detection techniques for hidden WebKit exploitation
- Implementing runtime monitoring for abnormal JavaScript behavior
- Creating comprehensive threat intelligence for iOS malware
- Developing mitigation strategies for multi-stage exploit chains

This analysis provides the technical foundation for understanding, detecting, and defending against this sophisticated iOS malware threat.

