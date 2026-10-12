FROM registry.fedoraproject.org/fedora:latest

RUN dnf install -y --setopt=install_weak_deps=False dnf5-plugins \
    && dnf builddep -y python3 \
    && dnf install -y --setopt=install_weak_deps=False git gdb \
    && dnf clean all

COPY . /src
WORKDIR /src

RUN ./configure --with-pydebug && make -s -j"$(nproc)"

# The scan runs offline, so download a copy of the security policy.
RUN curl -fsSL -o /opt/security-policy.rst https://raw.githubusercontent.com/python/devguide/main/security/policy.rst
