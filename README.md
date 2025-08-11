# LibOQS Installation with Constant-Time Testing Enabled

## Prerequisites

- **OS**: Ubuntu 22.04
- **Memory**: At least 4-8 GB
- **Disk Space**: More than 5 GB

Install required tools and libraries:

```bash
sudo apt update
sudo apt install astyle cmake gcc ninja-build libssl-dev \
                 python3-pytest python3-pytest-xdist unzip xsltproc \
                 doxygen graphviz python3-yaml valgrind
```

## Quick Installation

```bash
# Get the source code
git clone https://github.com/open-quantum-safe/liboqs.git
cd liboqs

# Create build directory
mkdir build && cd build

# Configure build with constant-time testing enabled
cmake -GNinja .. \
    -DCMAKE_BUILD_TYPE=Debug \
    -DOQS_ENABLE_TEST_CONSTANT_TIME=ON \
    -DOQS_DIST_BUILD=ON

# Optional extra flags you can add:
# -DCMAKE_INSTALL_PREFIX=/usr/local      # Install location
# -DOQS_USE_OPENSSL=ON                   # Use OpenSSL primitives
# -DOQS_ENABLE_KEM_CLASSIC_MCELIECE=OFF  # Skip slow algorithms
# -DBUILD_SHARED_LIBS=ON                  # Build shared libs

# Build the library
ninja
```

## Running Constant-Time Tests

### Test All Algorithms
```bash
# From the build/ directory
python3 ../tests/test_constant_time.py
```

Ignore the extra line breaks — I pressed Enter multiple times to ensure the program hadn’t frozen.

<img width="1920" height="887" alt="image" src="https://github.com/user-attachments/assets/d37daf2e-9b8c-4eaf-a5a4-724213c08462" />

### Test Specific Algorithms
```bash
# Test only Kyber
python3 ../tests/test_constant_time.py -k Kyber

# Test specific variant
python3 ../tests/test_constant_time.py -k ML-KEM-512

# Test with verbose output
python3 ../tests/test_constant_time.py -v -k Dilithium
```

### Skip Slow Algorithms
```bash
# Set environment variable to skip certain algorithms
export SKIP_ALGS="Classic-McEliece,HQC"
python3 ../tests/test_constant_time.py
```

### Direct Valgrind Testing
```bash
# Run Valgrind directly on a test binary
valgrind --tool=memcheck \
         --error-exitcode=1 \
         ./tests/test_kem ML-KEM-512
```
