FROM registry.fedoraproject.org/fedora:latest
ENV CC=clang \
    CXX=clang++ \
    ASAN_OPTIONS=detect_leaks=0:allocator_may_return_null=1:handle_segv=0:abort_on_error=1 \
    UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1:suppressions=/src/Tools/ubsan/suppressions.txt \
    PYTHONMALLOC=malloc

RUN dnf install -y --setopt=install_weak_deps=False dnf5-plugins \
    && dnf builddep -y python3 \
    && dnf install -y --setopt=install_weak_deps=False git clang llvm compiler-rt gdb \
    && dnf clean all

COPY . /src
WORKDIR /src

# Special OPT to avoid Clang bug (python/cpython#157958).
RUN ./configure OPT="-O2 -g" --with-pydebug --with-address-sanitizer --with-undefined-behavior-sanitizer \
      --with-strict-overflow --without-pymalloc --without-ensurepip \
    && make -s -j"$(nproc)"

# The scan runs offline, so pull in useful files.
RUN curl -fsSL -o /opt/security-policy.rst https://raw.githubusercontent.com/python/devguide/main/security/policy.rst \
    && git clone --quiet --depth=1 https://github.com/python/library-fuzzers.git /opt/library-fuzzers

# Smoke tests. Run the full suite inside the image with:
#     ./python -m test -j0
RUN ./python -m test -j"$(nproc)" \
      test_xxtestfuzz test_json test_re test_struct test_csv test_email test_tarfile \
      test_zipfile test_plistlib test_urlparse test_ipaddress test_binascii \
    || echo "WARNING: test suite sample reported failures"
