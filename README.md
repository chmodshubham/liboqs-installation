# LibOQS Installation with Constant-Time Testing Enabled

## Prerequisites

- **OS**: Ubuntu 22.04
- **Memory**: 4-8GB RAM minimum
- **Disk Space**: >5GB

```bash
sudo apt update
sudo apt install astyle cmake gcc ninja-build libssl-dev python3-pytest python3-pytest-xdist unzip xsltproc doxygen graphviz python3-yaml valgrind
```

## Quick Installation

```bash
git clone https://github.com/open-quantum-safe/liboqs.git
cd liboqs

mkdir build && cd build
cmake -GNinja .. \
    -DCMAKE_BUILD_TYPE=Debug \
    -DOQS_ENABLE_TEST_CONSTANT_TIME=ON \
    -DOQS_DIST_BUILD=ON

# Additional Optional Flags
#    -DCMAKE_INSTALL_PREFIX=/usr/local \     # Installation directory
#    -DOQS_USE_OPENSSL=ON \                  # Use OpenSSL for primitives
#    -DOQS_ENABLE_KEM_CLASSIC_MCELIECE=OFF \ # Disable slow algorithms for testing
#    -DBUILD_SHARED_LIBS=ON                  # Build shared libraries

# Build the library
ninja
```

## Running Constant-Time Tests

### Test All Algorithms
```bash
# From the build/ directory
python3 ../tests/test_constant_time.py
```

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
