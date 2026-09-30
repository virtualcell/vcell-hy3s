# syntax=docker/dockerfile:1.7
#
# vcell-hy3s: the Hy3S hybrid stochastic solvers, built once and published two
# ways from the same files (SOLVER-RELEASE.md):
#
#   --target archive   the linux64 / linux64arm release directory, for
#                      `--output type=local,dest=...`
#   --target runtime   (default) that same directory on PATH in a slim image
#                      with the standard VCell solver entrypoint; the SIF is
#                      built from this image
#
#   docker build -f docker/Dockerfile --build-arg HY3S_VERSION=1.0.0 -t vcell-hy3s .
#
# The build stage is manylinux_2_28 (AlmaLinux 8, glibc 2.28, GCC from
# gcc-toolset), so the archive runs on any glibc >= 2.28 host. It builds
# natively per architecture -- CI runs amd64 and arm64 on their own runners --
# and needs the vcell-messaging submodule checked out.

ARG MANYLINUX=manylinux_2_28
ARG RUNTIME_BASE=debian:bookworm-slim

FROM quay.io/pypa/${MANYLINUX}_x86_64 AS base-amd64
FROM quay.io/pypa/${MANYLINUX}_aarch64 AS base-arm64

# ---------------------------------------------------------------------------
FROM base-${TARGETARCH} AS build
ENV PATH=/opt/python/cp312-cp312/bin:${PATH} \
    CC=gcc CXX=g++ FC=gfortran
# cmake and ninja from PyPI: Conan's dependency builds (zlib, libcurl) use the
# system's, and the manylinux image has no ninja.
RUN python -m pip install --no-cache-dir 'conan>=2,<3' cmake ninja \
 && gcc --version | head -1 && gfortran --version | head -1 && ninja --version
WORKDIR /src

# Dependencies first, keyed on the recipe alone: libcurl (messaging) and zlib,
# static, from source. SSL is off -- the JMS REST bridge is plain HTTP -- which
# keeps OpenSSL out of the build and the binaries.
COPY conanfile.py ./
RUN conan profile detect --force \
 && conan install . --build=missing \
      -s build_type=Release -s compiler.cppstd=20 \
      -o 'libcurl/*:with_ssl=False' \
      -c tools.cmake.cmaketoolchain:generator=Ninja

COPY . .
ARG HY3S_VERSION=unknown
RUN . build/generators/conanbuild.sh \
 && cmake -B build -S . -G Ninja \
      -DCMAKE_TOOLCHAIN_FILE=/src/build/generators/conan_toolchain.cmake \
      -DCMAKE_BUILD_TYPE=Release \
      -DOPTION_TARGET_MESSAGING=ON \
      -DHY3S_VERSION_STAMP="${HY3S_VERSION}" \
 && cmake --build build \
 && ctest --test-dir build --output-on-failure \
 && packaging/package-linux.sh build/bin /dist "${HY3S_VERSION}"

# ---------------------------------------------------------------------------
FROM scratch AS archive
COPY --from=build /dist/ /

# ---------------------------------------------------------------------------
FROM ${RUNTIME_BASE} AS runtime
LABEL org.opencontainers.image.source="https://github.com/virtualcell/vcell-hy3s" \
      org.opencontainers.image.description="Hy3S hybrid stochastic-deterministic solvers for the Virtual Cell" \
      org.opencontainers.image.licenses="MIT"
COPY --from=build /dist/ /opt/vcell-hy3s/
COPY --chmod=0755 docker/entrypoint.sh /usr/local/bin/vcell-solver-entrypoint
ENV PATH=/opt/vcell-hy3s:${PATH}
ENTRYPOINT ["/usr/local/bin/vcell-solver-entrypoint"]
CMD ["--help"]
