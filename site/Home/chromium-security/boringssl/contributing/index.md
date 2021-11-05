---
breadcrumbs:
- - /Home
  - Chromium
- - /Home/chromium-security
  - Chromium Security
- - /Home/chromium-security/boringssl
  - BoringSSL
page_name: contributing
title: Contributing to BoringSSL
---

## Location of the code

The [BoringSSL](/Home/chromium-security/boringssl) code lives at
<https://boringssl.googlesource.com/boringssl.git>.

It is mapped into the Chromium tree via
[src/DEPS](https://chromium.googlesource.com/chromium/src/+/HEAD/DEPS) to
[src/third_party/boringssl](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/boringssl/&sq=package:chromium)

## Filing bugs

Bugs are filed under the
[label:Cr-Internals-Network-SSL](https://code.google.com/p/chromium/issues/list)
using the [Chromium issue
tracker](https://code.google.com/p/chromium/issues/list).

## Building

Refer to
[BUILDING.md](https://boringssl.googlesource.com/boringssl/+/HEAD/BUILDING.md)
for the canonical build instructions. Assuming those haven't changed, set things
up to build through ninja by executing:

```none
cd src/third_party/boringssl/src
mkdir build
cd build
cmake -GNinja ..
ninja
```

Once the ninja files are generated you can re-build from other directories
using:

```none
ninja -C <path-to-boringssl-src-build>
```

## Running the tests

See
[instructions](https://boringssl.googlesource.com/boringssl/+/HEAD/BUILDING.md#Running-tests)
in BUILDING.md to run the tests. From a Chromium checkout, this incantation may
be used:

```none
cd src/third_party/boringssl/src
ninja -C build && go run util/all_tests.go && (cd ssl/test/runner/ && go fmt && go test)
```

Or, if CMake is 3.2 or higher:

```none
cd src/third_party/boringssl/src
ninja -C build run_tests
```

## Uploading changes for review

See
[CONTRIBUTING.md](https://boringssl.googlesource.com/boringssl/+/HEAD/CONTRIBUTING.md)
in the BoringSSL repository.

## Rolling DEPS into Chromium

Because BoringSSL lives in a separate repository, it must be "rolled" into
Chromium to get the updates.

To roll BoringSSL create a changelist in the Chromium repository that modifies
[src/DEPS](https://chromium.googlesource.com/chromium/src/+/HEAD/DEPS) and
re-generates the gypi and asm files:

*   Simple example: <https://codereview.chromium.org/866213002> (Just
            modifies DEPS):
*   More complicated example:
            <https://codereview.chromium.org/693893006> (ASM changed and test
            added)

There is a script to automate all these steps:

```none
python third_party/boringssl/roll_boringssl.py
```

If the roll has added tests (the gypi modifications will list a new test), then
you must add a new entry in
[boringssl_unittest.cc](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/boringssl/boringssl_unittest.cc&sq=package:chromium&type=cs).
See util/all_tests.go for how the test is run. For a test which takes no
arguments, the new entry should look like this:

```none
TEST(BoringSSL, PQueue) {
  TestSimple("pqueue_test");
}
```