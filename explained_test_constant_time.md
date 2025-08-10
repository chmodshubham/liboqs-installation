# Understanding LibOQS Constant-Time Code

LibOQS has a smart way to check if its post-quantum cryptography code leaks secrets through timing. It uses Valgrind (normally for finding memory errors) to see if secret data affects how the program runs.

The trick: it “poisons” secret data — marks it as uninitialized in Valgrind — and then watches if that data changes the program’s control flow.

Code References:

- [tests/test_constant_time.py](https://github.com/open-quantum-safe/liboqs/blob/main/tests/test_constant_time.py)
- [tests/test_kem.c](https://github.com/open-quantum-safe/liboqs/blob/main/tests/test_kem.c)
- [tests/test_sig.c](https://github.com/open-quantum-safe/liboqs/blob/main/tests/test_sig.c)

### Key Components

1. **Valgrind Memcheck**: Memory error detector repurposed for timing analysis
2. **Random Bytes Interception**: Custom callback that marks all random data as "poisoned"
3. **Suppression System**: Documentation of known timing behaviors (acceptable vs. problematic)
4. **Python Test Framework**: Orchestrates testing and manages suppressions

## The Constant-Time Problem

### What is Constant-Time?
Constant-time means writing cryptographic code so that its execution time and resource usage do not depend on secret data (like encryption keys or passwords).

The goal is to stop attackers from learning secrets by measuring how long the code takes to run.

Despite the name, it doesn’t mean the code must take exactly the same time every time — it just means that any small timing differences cannot be linked to secret values.

The golden rule: secret information should only be used in ways that never affect execution time, memory access patterns, or other resource usage.

In short — secret-independent resource usage.

### Examples of Timing Leaks

**BAD - Variable Time:**
```c
// Execution time reveals information about secret_key
if (secret_key[0] == 0) {
    return EARLY;  // Fast path
}
// ... lots of computation ...
return NORMAL;     // Slow path
```

**GOOD - Constant Time:**
```c
// Both branches take same time
int mask = (secret_key[0] == 0) - 1;
result = (EARLY & ~mask) | (NORMAL & mask);
return result;
```

### Visual Representation

```
Secret Key Generation
        ↓
    [Real Random Bytes: 0x4F, 0x2A, 0x91, ...]
        ↓
    MARK AS "UNINITIALIZED" (Poison)
        ↓
    [Same Bytes, but Valgrind Tracking]
        ↓
    Use in Cryptography
        ↓
    if (secret[0] > 128) ← VALGRIND: "WARNING!"
```

## Implementation Architecture

### File Structure

```
liboqs/
├── tests/
│   ├── test_constant_time.py      # Main orchestrator
│   ├── test_kem.c                  # KEM test binary
│   ├── test_sig.c                  # Signature test binary
│   └── constant_time/
│       ├── kem/
│       │   ├── passes.json         # Acceptable timing variations
│       │   ├── issues.json         # Problematic timing leaks
│       │   ├── passes/             # Suppression files (OK)
│       │   │   └── *
│       │   └── issues/             # Suppression files (problems)
│       │       └── *
│       └── sig/
│           └── (same structure)
```

## Key Functions and Their Roles

### 1. `main()` in `test_kem.c` - The Setup

This is where everything begins. The main function sets up the interceptor that will poison all random bytes.

```c
// main function initialization
int main(int argc, char **argv) {
    // ... initialization code ...
    
    #ifdef OQS_ENABLE_TEST_CONSTANT_TIME
    // THIS IS THE KEY: Install our interceptor function
    OQS_randombytes_custom_algorithm(&TEST_KEM_randombytes);
    #endif
    
    // ... rest of main function ...
    // Now all calls to OQS_randombytes will go through TEST_KEM_randombytes
}
```

### 2. `TEST_KEM_randombytes` - The Interceptor

This function intercepts ALL requests for random bytes and marks them as "poisoned" for Valgrind.

```c
#ifdef OQS_ENABLE_TEST_CONSTANT_TIME
static void TEST_KEM_randombytes(uint8_t *random_array, size_t bytes_to_read) {
    // Step 1: Temporarily switch to the real system RNG
    // (to avoid infinite recursion)
    OQS_randombytes_switch_algorithm("system");
    
    // Step 2: Get REAL random bytes (so crypto works correctly)
    OQS_randombytes(random_array, bytes_to_read);
    
    // Step 3: Switch back to ourselves for future calls
    OQS_randombytes_custom_algorithm(&TEST_KEM_randombytes);
    
    // Step 4: THE MAGIC - Tell Valgrind these bytes are "uninitialized"
    // Even though they contain real random data!
    OQS_TEST_CT_CLASSIFY(random_array, bytes_to_read);
}
#endif
```

### 3. `OQS_randombytes_switch_algorithm` - The Algorithm Switcher

This function changes which random number generator is active.

```c
OQS_STATUS OQS_randombytes_switch_algorithm(const char *algorithm) {
    if (strcasecmp("system", algorithm) == 0) {
        // Switch to system RNG
        oqs_randombytes_algorithm = &OQS_randombytes_system;
        return OQS_SUCCESS;
    } 
    #ifdef OQS_USE_OPENSSL
    else if (strcasecmp("openssl", algorithm) == 0) {
        // Switch to OpenSSL RNG
        oqs_randombytes_algorithm = &OQS_randombytes_openssl;
        return OQS_SUCCESS;
    }
    #endif
    return OQS_ERROR;
}
```

### 4. `OQS_randombytes` - The Main RNG Function

This is what all cryptographic code calls to get random bytes.

```c
void OQS_randombytes(uint8_t *random_array, size_t bytes_to_read) {
    // Calls whatever function oqs_randombytes_algorithm points to
    oqs_randombytes_algorithm(random_array, bytes_to_read);
}
```

### 5. `OQS_randombytes_custom_algorithm` - Set Custom RNG

This sets a custom random number generator (like our interceptor).

```c
void OQS_randombytes_custom_algorithm(void (*algorithm_ptr)(uint8_t *, size_t)) {
    // Change the global function pointer
    oqs_randombytes_algorithm = algorithm_ptr;
}
```

### 6. `OQS_TEST_CT_CLASSIFY` - The Poison Macro

This is the actual "poisoning" - it tells Valgrind to track these bytes.

```c
#define OQS_TEST_CT_CLASSIFY(addr, len) \
    VALGRIND_MAKE_MEM_UNDEFINED(addr, len)
```

## Complete Call Flow

### Phase 1: Initialization

```
1. test_constant_time.py starts
        ↓
2. Loads suppression files from JSON
        ↓
3. Builds Valgrind command:
   valgrind --tool=memcheck \
            --suppressions=passes/file1 \
            --suppressions=issues/file2 \
            ./test_kem ML-KEM-512
        ↓
4. Launches test_kem under Valgrind
```

### Phase 2: Setup

```
5. test_kem main() starts
        ↓
6. Installs interceptor:
   OQS_randombytes_custom_algorithm(&TEST_KEM_randombytes)
        ↓
   [Now: oqs_randombytes_algorithm = TEST_KEM_randombytes]
```

### Phase 3: Runtime

```
7. Test function runs:
   kem_test_correctness()
        ↓
8. Crypto needs random bytes:
   OQS_KEM_keypair() calls OQS_randombytes(secret_key, 32)
        ↓
9. OQS_randombytes routes to TEST_KEM_randombytes:
   TEST_KEM_randombytes(secret_key, 32)
        ├── Switch to system RNG
        ├── Get real random bytes [0x4F, 0x2A, ...]
        ├── Switch back to interceptor
        └── POISON the bytes (mark as uninitialized)
        ↓
10. secret_key now contains real data BUT Valgrind thinks it's uninitialized
        ↓
11. Crypto uses the secret key:
    if (secret_key[0] == 0)  // Valgrind: "WARNING! Branch on uninitialized!"
```

### Phase 4: Detection

```
12. Valgrind reports error and exits with code 1
        ↓
13. Python checks exit code:
    - Error + NOT in suppressions = TEST FAILS (new leak!)
    - Error + IN suppressions = TEST PASSES (known issue)
    - No error = TEST PASSES (no leaks detected)
```

## The Function Pointer Magic

The key to understanding this system is the global function pointer:

```c
// Global variable in liboqs (simplified)
void (*oqs_randombytes_algorithm)(uint8_t*, size_t) = &OQS_randombytes_system;

// Timeline of pointer changes:
// 1. Start: points to → OQS_randombytes_system
// 2. main(): changed to → TEST_KEM_randombytes
// 3. Inside TEST_KEM_randombytes:
//    a. Temporarily → OQS_randombytes_system
//    b. Get random bytes
//    c. Change back to → TEST_KEM_randombytes
// 4. Ready for next call
```

## Why The Switching Dance?

The switching prevents infinite recursion:

```c
// WITHOUT switching (would crash!):
TEST_KEM_randombytes() {
    OQS_randombytes();  // Calls TEST_KEM_randombytes
                       // Which calls OQS_randombytes
                       // Which calls TEST_KEM_randombytes
                       // INFINITE LOOP!
}

// WITH switching (works!):
TEST_KEM_randombytes() {
    switch_to_system();    // Now points to system RNG
    OQS_randombytes();     // Calls system RNG (not ourselves!)
    switch_back_to_us();   // Ready for next interception
}
```

## The Suppression System

### Purpose
Document known timing behaviors to focus on new issues.

### Suppression File Format

```
{
   Rejection sampling in Kyber key generation    # Description
   Memcheck:Cond                                 # Error type
   fun:rej_uniform                               # Where it happens
   fun:PQCLEAN_KYBER*_CLEAN_gen_matrix          # Calling function
}
```

### Directory Structure

- **passes/**: Acceptable timing variations (public data, safe operations)
- **issues/**: Real timing leaks that need fixing

### JSON Configuration

example:

```json
// passes.json
{
  "ML-KEM-512": ["kyber_rejection", "public_decode"],
  "ML-KEM-768": ["kyber_rejection"],
  "Dilithium2": ["dilithium_hint"]
}
```