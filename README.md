# ollama-tests

This script installs Ollama on Linux without requiring sudo

```bash
#!/bin/sh

# This script installs Ollama on Linux without requiring sudo.
set -eu
status() { echo ">>> $*" >&2; }
error() { echo "ERROR $*"; exit 1; }
warning() { echo "WARNING: $*"; }

TEMP_DIR=$(mktemp -d)
cleanup() { rm -rf "$TEMP_DIR"; }
trap cleanup EXIT

available() { command -v $1 >/dev/null; }
require() {
    local MISSING=''
    for TOOL in $*; do
        if ! available $TOOL; then
            MISSING="$MISSING $TOOL"
        fi
    done
    echo "$MISSING"
}

[ "$(uname -s)" = "Linux" ] || error 'This script is intended to run on Linux only.'
ARCH=$(uname -m)
case "$ARCH" in
    x86_64) ARCH="amd64" ;;
    aarch64|arm64) ARCH="arm64" ;;
    *) error "Unsupported architecture: $ARCH" ;;
esac

IS_WSL2=false
KERN=$(uname -r)
case "$KERN" in
    *icrosoft*WSL2 | *icrosoft*wsl2) IS_WSL2=true;;
    *icrosoft) error "Microsoft WSL1 is not currently supported. Please use WSL2 with 'wsl --set-version <distro> 2'" ;;
    *) ;;
esac

VER_PARAM="${OLLAMA_VERSION:+?version=$OLLAMA_VERSION}"
NEEDS=$(require curl awk grep sed tee xargs)
if [ -n "$NEEDS" ]; then
    status "ERROR: The following tools are required but missing:"
    for NEED in $NEEDS; do
        echo "  - $NEED"
    done
    exit 1
fi

# Install to a non-root directory
OLLAMA_INSTALL_DIR="$HOME/.local/share/ollama"
BINDIR="$HOME/.local/bin"

status "Installing ollama to $OLLAMA_INSTALL_DIR"
mkdir -p "$BINDIR"
mkdir -p "$OLLAMA_INSTALL_DIR"

if curl -I --silent --fail --location "https://ollama.com/download/ollama-linux-${ARCH}.tgz${VER_PARAM}" >/dev/null ; then
    status "Downloading Linux ${ARCH} bundle"
    curl --fail --show-error --location --progress-bar \
        "https://ollama.com/download/ollama-linux-${ARCH}.tgz${VER_PARAM}" | \
        tar -xzf - -C "$OLLAMA_INSTALL_DIR" --strip-components=1
else
    status "Downloading Linux ${ARCH} CLI"
    curl --fail --show-error --location --progress-bar -o "$TEMP_DIR/ollama"\
    "https://ollama.com/download/ollama-linux-${ARCH}${VER_PARAM}"
    mv "$TEMP_DIR/ollama" "$OLLAMA_INSTALL_DIR/ollama"
fi

# Create a symlink in $BINDIR
if [ ! -f "$BINDIR/ollama" ] ; then
    ln -sf "$OLLAMA_INSTALL_DIR/ollama" "$BINDIR/ollama"
fi

install_success() {
    status 'The Ollama API is now available at 127.0.0.1:11434.'
    status 'Install complete. Run "ollama" from the command line.'
}
trap install_success EXIT
```
