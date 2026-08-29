---
tags:
  - WIP
  - projects
  - homelab
  - ai
date: "2026-08-19"
title: mentat_provisioning
---

The concrete runbook for `mentat`. The *why* behind these choices lives in [[strix_halo_ai_box|Strix Halo AI box]] — this page is the implementation: every file that goes in git, and the commands that consume them.

Ubuntu Server 26.04 LTS is installed. The box is an **appliance**: it runs the llama.cpp and ComfyUI toolboxes and nothing else.

# Principles

1. **One repo is the source of truth.** A plain git repo holds the `/etc` payloads, the systemd units, the service config and the scripts. Nothing is configured by hand that could live there.
2. **Idempotent by construction, not by checking.** Build to a temp file, `cmp`, `install` only on difference. No `sed -i`, no appends. Every script is safe to run a hundred times.
3. **Derive, don't duplicate.** Any value appearing twice is a future inconsistency. `GPU_RESERVE_GIB` lives in `vars.sh`; both kernel knobs are computed from it.
4. **Expensive side effects are conditional.** `update-grub`, `sysctl --system`, `daemon-reload` and container recreation fire only when their input changed, so a converged `make` is silent and fast.
5. **Nothing on the host that a toolbox could carry.** No Python, no PyTorch, no compilers, no config-management tooling. ComfyUI's entire ROCm/PyTorch stack lives in its container.

> [!note] Earlier drafts of this page used chezmoi + mise. Both were dropped: chezmoi earns its keep managing `$HOME` across several machines with templating, and this is one appliance with no dotfiles; mise was pinning a single Go binary. `git` + `make` + `install(1)` cover the same ground with nothing extra on the host — Principle 5 applied to the automation itself.

# Repo layout

```
mentat/                              # git repo, cloned to ~/mentat
├── Makefile                         # the operational interface
├── vars.sh                          # every tunable, sourced by all scripts
├── etc/
│   └── sysctl.d/99-mentat.conf      # static payload
├── home/
│   ├── .config/llama-swap/config.yaml
│   └── .config/systemd/user/{llama-swap,comfyui}.service
├── bin/
│   ├── lib.sh                       # install_if_changed helper
│   ├── bootstrap                    # packages, groups, model dirs (first run)
│   ├── install-system               # /etc + GRUB drop-in (generated)
│   ├── install-user                 # home files + daemon-reload
│   ├── toolboxes                    # create (idempotent)
│   ├── toolboxes-refresh            # pull + recreate changed
│   ├── llama-swap-install           # pinned binary
│   ├── model-add                    # download a GGUF, check it fits
│   └── verify                       # assert invariants
└── benchmarks/
    ├── baseline.json                # last-known-good tok/s, committed
    └── 2026-08-19-vulkan-radv.txt   # history, committed
```

No templating engine: the GRUB drop-in is the only file needing derived values, and a heredoc in `install-system` handles it. Everything else is static YAML/ini.

What deliberately stays **out** of git: BIOS settings (documented in [[strix_halo_ai_box|the plan]], Phase 0 — unautomatable), model weights (large, re-downloadable), and secrets. For a gated-model HF token, keep it in `~/.hf_token` (mode 0600, gitignored) rather than committed — the appliance has no secret-management story and doesn't need one for public GGUFs.

# vars.sh

The single source of tunables:

```bash
GPU_RESERVE_GIB=120        # both kernel knobs derive from this
SWAPPINESS=10
LLAMA_LISTEN=127.0.0.1:8080
COMFY_PORT=8188
MODELS_DIR=/srv/models
COMFY_MODELS_DIR=/srv/comfy-models
LLAMA_SWAP_VERSION=v151    # pin; check the releases page before bumping
```

# bin/lib.sh

The one abstraction the whole repo leans on. It returns non-zero when nothing changed, so callers can gate their side effects:

```bash
# install_if_changed <src> <dst> <mode> [--sudo]
# Returns 0 if the file was installed (changed), 1 if already current.
install_if_changed() {
  local src=$1 dst=$2 mode=$3 sudo_cmd=""
  [ "${4:-}" = "--sudo" ] && sudo_cmd="sudo"
  if $sudo_cmd cmp -s "$src" "$dst" 2>/dev/null; then
    echo "  unchanged: $dst"
    return 1
  fi
  $sudo_cmd install -D -m "$mode" "$src" "$dst"
  echo "  installed: $dst"
  return 0
}
```

# Makefile

```make
.PHONY: all bootstrap system user toolboxes refresh llama-swap verify bench

all: system user toolboxes          ## converge the machine

bootstrap:  ; @bin/bootstrap
system:     ; @bin/install-system
user:       ; @bin/install-user
toolboxes:  ; @bin/toolboxes
refresh:    ; @bin/toolboxes-refresh
llama-swap: ; @bin/llama-swap-install
verify:     ; @bin/verify
```

# First run

```bash
git clone git@github.com:<me>/mentat.git ~/mentat && cd ~/mentat
make bootstrap       # packages, groups, model dirs
make system          # /etc + GRUB — prints REBOOT REQUIRED
sudo reboot

cd ~/mentat
make llama-swap toolboxes
bin/model-add <hf-repo> <filename>   # at least one model, see below
make user            # units + linger + enable
make verify
```

The reboot is mandatory: the GTT reservation is a boot parameter, and it also activates the new `render`/`video` group membership.

# bin/bootstrap

```bash
#!/usr/bin/env bash
set -euo pipefail
. "$(dirname "$0")/../vars.sh"

# apt-get install is idempotent, so re-running is a no-op.
# Note what is absent: no python, no rocm SDK, no compilers. Both workloads
# bring their own stack inside their toolbox.
REQUIRED=(podman crun distrobox mesa-vulkan-drivers vulkan-tools git curl jq make)
OPTIONAL=(rocm-smi radeontop)   # host monitoring is a convenience, not a dependency

sudo DEBIAN_FRONTEND=noninteractive apt-get update -qq
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends "${REQUIRED[@]}"
for pkg in "${OPTIONAL[@]}"; do
  sudo DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends "$pkg" \
    || echo "note: optional package '$pkg' unavailable, skipping" >&2
done

# zram would compress model weights into RAM-backed swap: catastrophic here.
# Report rather than purge — silently removing a swap provider is not something
# a provisioning script should do unasked.
if dpkg-query -W -f='${Status}' zram-tools 2>/dev/null | grep -q 'ok installed'; then
  echo "WARNING: zram-tools installed. Remove it: sudo apt-get purge zram-tools" >&2
fi

# Additive and idempotent; takes effect at next login, which the reboot covers.
sudo usermod -aG render,video "$USER"

# ComfyUI's toolbox expects ~/comfy-models: symlink rather than duplicate.
sudo install -d -o "$USER" -g "$USER" -m 0755 "$MODELS_DIR" "$MODELS_DIR/hf" "$COMFY_MODELS_DIR"
ln -sfn "$COMFY_MODELS_DIR" "$HOME/comfy-models"
```

# bin/install-system

The highest-stakes script; the drop-in mechanism is what keeps it safe.

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
. ./vars.sh
. ./bin/lib.sh

TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT

# Ubuntu's grub-mkconfig sources /etc/default/grub.d/*.cfg after
# /etc/default/grub (Ubuntu ships 50-cloudimg-settings.cfg the same way), so we
# *append* via a drop-in rather than rewriting the main file. Idempotent by
# whole-file replacement, no sed, and a release upgrade that rewrites
# /etc/default/grub cannot silently drop our parameters.
#
# The two GTT knobs use different units and published guides routinely quote
# pairs that disagree. Both are computed here from one value:
#   ttm.pages_limit -> 4 KiB pages -> gib * 262144
#   amdgpu.gttsize  -> MiB         -> gib * 1024
cat >"$TMP/grub.cfg" <<EOF
# Managed by ~/mentat — edits here will be overwritten.
GRUB_CMDLINE_LINUX_DEFAULT="\$GRUB_CMDLINE_LINUX_DEFAULT amd_iommu=off amdgpu.gttsize=$((GPU_RESERVE_GIB*1024)) ttm.pages_limit=$((GPU_RESERVE_GIB*262144))"
EOF

if install_if_changed "$TMP/grub.cfg" /etc/default/grub.d/99-mentat-amdgpu.cfg 0644 --sudo; then
  sudo update-grub
  sudo touch /var/run/reboot-required
  echo ">>> ${GPU_RESERVE_GIB} GiB GTT — REBOOT REQUIRED"
fi

if install_if_changed etc/sysctl.d/99-mentat.conf /etc/sysctl.d/99-mentat.conf 0644 --sudo; then
  sudo sysctl --system >/dev/null
fi
```

The escaped `\$GRUB_CMDLINE_LINUX_DEFAULT` is deliberate: it must land in the file **literally** so GRUB expands it at `update-grub` time, while the `$(( ))` arithmetic is resolved when the script runs.

`etc/sysctl.d/99-mentat.conf`:

```
# Managed by ~/mentat — edits here will be overwritten.
# With 128 GB and a hard GTT ceiling, healthy operation should never swap. If it
# does, the GTT reservation is too aggressive.
vm.swappiness = 10
```

# bin/install-user

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
. ./bin/lib.sh

CHANGED=0
install_if_changed home/.config/llama-swap/config.yaml \
  "$HOME/.config/llama-swap/config.yaml" 0644 && CHANGED=1
for unit in llama-swap comfyui; do
  install_if_changed "home/.config/systemd/user/$unit.service" \
    "$HOME/.config/systemd/user/$unit.service" 0644 && CHANGED=1
done

# Headless box: without lingering, user units don't start until someone logs in.
# Presents as "the service didn't come back after reboot" and is easy to
# misdiagnose. Idempotent.
loginctl enable-linger "$USER"

if [ "$CHANGED" -eq 1 ]; then
  systemctl --user daemon-reload
  systemctl --user enable --now llama-swap comfyui
  systemctl --user restart llama-swap comfyui
else
  systemctl --user enable --now llama-swap comfyui
fi
```

# bin/llama-swap-install

```bash
#!/usr/bin/env bash
set -euo pipefail
. "$(dirname "$0")/../vars.sh"

# Pinned, not "latest": this is the reproducibility story without a lockfile.
if command -v llama-swap >/dev/null && llama-swap --version 2>&1 | grep -q "${LLAMA_SWAP_VERSION#v}"; then
  echo "llama-swap ${LLAMA_SWAP_VERSION} already installed"; exit 0
fi

URL=$(curl -fsSL "https://api.github.com/repos/mostlygeek/llama-swap/releases/tags/${LLAMA_SWAP_VERSION}" \
      | jq -r '.assets[].browser_download_url' | grep -iE 'linux.*(amd64|x86_64)' | head -1)
[ -n "$URL" ] || { echo "could not find a linux amd64 asset for ${LLAMA_SWAP_VERSION}" >&2; exit 1; }

curl -fsSL "$URL" -o /tmp/llama-swap.tar.gz
sudo tar -xzf /tmp/llama-swap.tar.gz -C /usr/local/bin llama-swap
sudo chmod +x /usr/local/bin/llama-swap
llama-swap --version
```

# bin/toolboxes

```bash
#!/usr/bin/env bash
set -euo pipefail

TOOLBOXES=(
  "llama-vulkan-radv|docker.io/kyuz0/amd-strix-halo-toolboxes:vulkan-radv"
  "llama-rocm|docker.io/kyuz0/amd-strix-halo-toolboxes:rocm-7.14"
  "comfyui|docker.io/kyuz0/amd-strix-halo-comfyui:latest"
)

# keep-groups is not a style choice. Under rootless podman a named --group-add
# (video/render) resolves against the *container's* /etc/group; the GID lands in
# the user namespace and never maps to the host's, granting no access to
# /dev/kfd. It fails as "the GPU isn't there", not as a permissions error.
#
# Every toolbox gets /dev/kfd: ComfyUI is ROCm-only, and it's harmless on the
# Vulkan box. One flag set, one less thing to get wrong.
FLAGS="--device /dev/dri --device /dev/kfd --group-add keep-groups --security-opt seccomp=unconfined"

for entry in "${TOOLBOXES[@]}"; do
  IFS='|' read -r name image <<<"$entry"
  # podman container exists is exact and scriptable; parsing `distrobox list` is not.
  if podman container exists "$name"; then
    echo "  present: $name"
  else
    echo "  creating: $name"
    distrobox create --yes --name "$name" --image "$image" --additional-flags "$FLAGS"
  fi
done
```

`bin/toolboxes-refresh` pulls each image, compares the digest, and only `distrobox rm --force`s containers whose image actually moved — then re-runs `bin/toolboxes` and restarts the services. Kept separate from `toolboxes` precisely because it destroys containers. Safe to do: toolboxes are stateless, `$HOME` is shared from the host, so nothing of value lives inside one.

# Models

Nothing here works without at least one GGUF on disk. `bin/model-add` wraps discovery, fit-checking and download:

```bash
#!/usr/bin/env bash
#   bin/model-add <hf-repo> [filename-filter]
# With no filter, lists the GGUFs in the repo and exits.
set -euo pipefail
. "$(dirname "$0")/../vars.sh"

REPO=${1:?usage: model-add <hf-repo> [filename-filter]}
FILTER=${2:-}

FILES=$(curl -fsSL "https://huggingface.co/api/models/${REPO}" \
        | jq -r '.siblings[].rfilename' | grep -i '\.gguf$' || true)
[ -n "$FILES" ] || { echo "no GGUF files in ${REPO}" >&2; exit 1; }

if [ -z "$FILTER" ]; then
  echo "$FILES"; echo; echo "re-run with a filter to download"; exit 0
fi

# Split models (…-00001-of-00003.gguf) need every part; llama.cpp is pointed at
# the first and finds the rest.
MATCHES=$(echo "$FILES" | grep -i -- "$FILTER")
echo "will download:"; echo "$MATCHES"

for f in $MATCHES; do
  dest="${MODELS_DIR}/$(basename "$f")"
  [ -f "$dest" ] && { echo "  have: $(basename "$f")"; continue; }
  echo "  fetching: $f"
  curl -fL --progress-bar -o "$dest" "https://huggingface.co/${REPO}/resolve/main/${f}"
done

FIRST=$(echo "$MATCHES" | head -1 | xargs basename)
echo
echo "checking fit:"
distrobox enter -n llama-vulkan-radv -- \
  python /opt/gguf-vram-estimator.py "${MODELS_DIR}/${FIRST}" --context 32768 || true
echo
echo "add to home/.config/llama-swap/config.yaml, then: make user"
```

## Choosing models for this hardware

Strix Halo's ceiling is **memory bandwidth** (~256 GB/s), not compute — far below a discrete GPU. The practical consequence:

**Prefer MoE over dense.** A mixture-of-experts model with few active parameters per token reads far less memory per token than a dense model of the same total size, so it runs dramatically faster here. A dense 70B is bandwidth-starved on this box; a much larger MoE with ~10B active can outrun it. This is the single most important model-selection rule for gfx1151.

Sizing against the 120 GiB pool, leaving room for ComfyUI:

| Tier | Target footprint | Role |
|---|---|---|
| Small, always resident | ~6–10 GiB | quick edits, autocomplete, cheap agent steps |
| Large, TTL-evicted | ~40–50 GiB | the model that does real work |
| Both + ComfyUI | ~58 GiB + ~60 GiB free | comfortable, incl. video workflows |

Q4_K_M is the value sweet spot; Q6_K if it fits. Always run `gguf-vram-estimator.py` with your intended `--context` before committing to a download — KV cache grows fast and 128k context can cost more than the weights.

# Serving config

`home/.config/llama-swap/config.yaml` — macros kill the boilerplate, groups express the concurrency shape:

```yaml
healthCheckTimeout: 300

macros:
  vulkan: >
    distrobox enter --name llama-vulkan-radv --
    llama-server --host 127.0.0.1 --port ${PORT}
    -fa 1 --no-mmap -ngl 999

groups:
  "always-on":
    persistent: true      # other groups can't unload this one
    swap: false           # members run concurrently
    exclusive: false      # loading it doesn't unload anything else
    members: ["small"]
  "heavy":
    swap: true            # one big model at a time
    exclusive: false      # critical: true would evict always-on
    members: ["large"]

models:
  "small":
    # no ttl — this one is meant to stay resident
    cmd: "${vulkan} -m /srv/models/<small>.gguf -c 16384"
  "large":
    ttl: 600              # release the pool for ComfyUI when idle
    cmd: "${vulkan} -m /srv/models/<large>.gguf -c 32768"
```

`-fa 1` and `--no-mmap` are mandatory on gfx1151 or llama.cpp crashes; putting them in the macro means they can't be forgotten. Confirm the exact macro interpolation syntax against the llama-swap README.

`home/.config/systemd/user/llama-swap.service`:

```ini
[Unit]
Description=llama-swap (gfx1151 LLM router)
After=network-online.target

[Service]
ExecStart=/usr/local/bin/llama-swap --config %h/.config/llama-swap/config.yaml --listen 127.0.0.1:8080
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

`home/.config/systemd/user/comfyui.service`:

```ini
[Unit]
Description=ComfyUI (gfx1151, ROCm)
After=network-online.target

[Service]
# start_comfy_ui is a shell ALIAS inside the toolbox, and bash does not expand
# aliases non-interactively — so neither `distrobox enter -- start_comfy_ui` nor
# `bash -lc '…'` works here. Resolve it once:
#     distrobox enter -n comfyui -- bash -ic 'type start_comfy_ui'
# and use the real script path. If the alias already carries the flags below,
# don't repeat them.
ExecStart=/usr/bin/distrobox enter --name comfyui -- <resolved /opt script> \
    --listen 127.0.0.1 --port 8188 --bf16-vae --disable-mmap --cache-none
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

`--disable-mmap` is the same unified-memory lesson as llama.cpp's `--no-mmap`. `--cache-none` keeps ComfyUI from holding models between runs, which is half the coexistence story — the other half is `ttl` above.

## Exposure

```bash
tailscale serve --bg --https=443  http://127.0.0.1:8080     # LLM endpoint
tailscale serve --bg --https=8188 http://127.0.0.1:8188     # ComfyUI
```

TLS, no firewall rules, no tailnet IP to drift, and `0.0.0.0` never enters the picture. Config persists across reboots.

# bin/verify

The script that earns its keep. On an LTS with unattended upgrades enabled, post-reboot is exactly when the machine will have changed underneath me.

```bash
#!/usr/bin/env bash
set -uo pipefail   # deliberately not -e: collect every failure, don't stop at the first
cd "$(dirname "$0")/.."
. ./vars.sh

FAIL=0
ok()  { printf '  \033[32mok  \033[0m %s\n' "$1"; }
bad() { printf '  \033[31mFAIL\033[0m %s\n' "$1"; FAIL=1; }
ver_ge() { [ "$(printf '%s\n%s\n' "$2" "$1" | sort -V | head -n1)" = "$2" ]; }

echo "kernel & firmware"
KVER=$(uname -r); KVER=${KVER%%-*}
ver_ge "$KVER" 6.18.9 && ok "kernel $KVER >= 6.18.9" || bad "kernel $KVER below the gfx1151 floor"
dpkg-query -W -f='${Version}' linux-firmware 2>/dev/null | grep -q '20251125' \
  && bad "linux-firmware 20251125 — known to break ROCm on Strix Halo" \
  || ok "linux-firmware not the known-bad build"

echo "kernel cmdline"
grep -q 'amd_iommu=off' /proc/cmdline && ok "iommu off" || bad "amd_iommu=off missing"
GTT_MIB=$(grep -oP 'amdgpu\.gttsize=\K[0-9]+' /proc/cmdline || true)
PAGES=$(grep -oP 'ttm\.pages_limit=\K[0-9]+' /proc/cmdline || true)
if [ -n "$GTT_MIB" ] && [ -n "$PAGES" ]; then
  # 1 MiB = 256 x 4 KiB pages. Catches the copy-paste-from-a-blog-post bug.
  [ "$((GTT_MIB * 256))" -eq "$PAGES" ] \
    && ok "gttsize and pages_limit agree ($((GTT_MIB / 1024)) GiB)" \
    || bad "gttsize=${GTT_MIB}MiB disagrees with pages_limit=${PAGES} pages"
  [ "$((GTT_MIB / 1024))" -eq "$GPU_RESERVE_GIB" ] && ok "reservation is ${GPU_RESERVE_GIB} GiB" \
    || bad "reservation is $((GTT_MIB / 1024)) GiB, vars.sh says ${GPU_RESERVE_GIB}"
else
  bad "GTT parameters absent from /proc/cmdline"
fi

echo "memory behaviour"
[ "$(cat /proc/sys/vm/swappiness)" -eq "$SWAPPINESS" ] && ok "swappiness $SWAPPINESS" || bad "swappiness $(cat /proc/sys/vm/swappiness)"
dpkg-query -W -f='${Status}' zram-tools 2>/dev/null | grep -q 'ok installed' \
  && bad "zram-tools installed" || ok "no zram"
SWAP_USED=$(awk '/^SwapTotal/{t=$2} /^SwapFree/{f=$2} END{print t-f}' /proc/meminfo)
[ "$SWAP_USED" -lt 65536 ] && ok "swap essentially unused" \
  || bad "swap in use (${SWAP_USED} kB) — GTT reservation may be too aggressive"

echo "appliance hygiene"
# Principle 5: if these appear, something was installed by hand and the repo is
# no longer the source of truth.
for unwanted in python3-torch rocm-hip-sdk build-essential; do
  dpkg-query -W -f='${Status}' "$unwanted" 2>/dev/null | grep -q 'ok installed' \
    && bad "$unwanted on the host — belongs in a toolbox" || ok "$unwanted absent"
done

echo "gpu access"
command -v crun >/dev/null && ok "crun present (keep-groups works)" || bad "crun missing — rootless GPU access will silently fail"
for tb in llama-vulkan-radv llama-rocm comfyui; do
  podman container exists "$tb" && ok "$tb exists" || bad "$tb missing"
done
distrobox enter -n llama-vulkan-radv -- llama-cli --list-devices 2>/dev/null | grep -qiE 'rocm|vulkan' \
  && ok "llama toolbox sees the iGPU" || bad "llama toolbox reports no GPU — CPU fallback"
distrobox enter -n comfyui -- python -c 'import torch; assert torch.cuda.is_available()' 2>/dev/null \
  && ok "comfyui torch sees the GPU" || bad "comfyui torch has no GPU"

echo "models"
ls "${MODELS_DIR}"/*.gguf >/dev/null 2>&1 && ok "GGUFs present" || bad "no models in ${MODELS_DIR}"

echo "services"
loginctl show-user "$USER" -p Linger --value | grep -q yes && ok "linger enabled" || bad "linger disabled — units won't start on boot"
for svc in llama-swap comfyui; do
  systemctl --user is-active --quiet "$svc" && ok "$svc active" || bad "$svc not active"
done
curl -fsS --max-time 5 "http://${LLAMA_LISTEN}/v1/models" >/dev/null && ok "LLM endpoint answers" || bad "LLM endpoint not responding"
curl -fsS --max-time 10 "http://127.0.0.1:${COMFY_PORT}/" >/dev/null && ok "ComfyUI answers" || bad "ComfyUI not responding"

echo
[ "$FAIL" -eq 0 ] && echo "mentat is converged." || echo "drift detected."
exit "$FAIL"
```

> [!tip] Run `shellcheck bin/*` before trusting these. `verify` failing for the wrong reason is worse than not having it, and this is exactly the kind of quoting-heavy bash where a typo hides for months. Worth a pre-commit hook.

## Throughput baseline

Boolean checks can all pass while inference has silently fallen back to CPU, so the real regression detector is a number. `bin/bench` runs `llama-bench` per backend and writes a dated file into `benchmarks/`; both the results and `baseline.json` get committed, giving a git-tracked performance history across kernel and image upgrades — a regression becomes a diff rather than a vague feeling.

```bash
#!/usr/bin/env bash
set -euo pipefail
. "$(dirname "$0")/../vars.sh"
MODEL="${1:?usage: bin/bench <model.gguf>}"
STAMP=$(date +%F)
mkdir -p benchmarks
for tb in llama-vulkan-radv llama-rocm; do
  podman container exists "$tb" || continue
  echo "==> $tb"
  distrobox enter -n "$tb" -- llama-bench -m "$MODEL" -fa 1 --no-mmap -ngl 999 \
    | tee "benchmarks/${STAMP}-${tb}.txt"
done
```

Wiring a tolerance check into `verify` is worth doing once there are a few runs to calibrate from — guessing a threshold beforehand just produces false alarms. ComfyUI needs an equivalent (wall-clock on a fixed workflow and seed), which is a separate script to write.

# Operational cheatsheet

| Intent | Command |
|---|---|
| Converge the machine | `make` |
| Change the GTT reservation | edit `vars.sh`, `make system`, reboot |
| Restart both services | `systemctl --user restart llama-swap comfyui` |
| Tail logs | `journalctl --user -u llama-swap -u comfyui -f` |
| See what's resident | `rocm-smi --showmeminfo all` + `curl -s localhost:8080/running` |
| List GGUFs in an HF repo | `bin/model-add <repo>` |
| Add a model | `bin/model-add <repo> <filter>`, edit config, `make user` |
| Adopt newer toolbox images | `make refresh` |
| Bump `llama-swap` | edit `vars.sh`, `make llama-swap` |
| Record performance | `bin/bench <model>` |
| After any reboot | `make verify` |

# Open items

- Resolve the real `start_comfy_ui` target so `comfyui.service` can be finalised — currently a placeholder.
- Write `bin/toolboxes-refresh` (described above, not yet written out here).
- Wire the `bench` → `baseline.json` tolerance check into `verify` once there's history, and write the ComfyUI equivalent.
- `shellcheck` pre-commit hook.
- Decide whether `verify` should run post-boot via a systemd unit that reports failures, rather than relying on me remembering.
- Confirm llama-swap's macro interpolation syntax and whether it hot-reloads config or needs a restart.

# References

- [llama-swap configuration](https://github.com/mostlygeek/llama-swap/blob/main/config.example.yaml) and [groups/swapping policies](https://deepwiki.com/mostlygeek/llama-swap/3.4-groups-and-swapping-policies)
- [Debian `grub-mkconfig`](https://sources.debian.org/src/grub2/sid/util/grub-mkconfig.in/) — confirms `/etc/default/grub.d/*.cfg` is sourced after the main file
- [How to Run AMD GPU Containers with Podman](https://oneuptime.com/blog/post/2026-03-18-run-amd-gpu-containers-podman/view) — the rootless `keep-groups` trap
- [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) and [ComfyUI toolboxes](https://github.com/kyuz0/amd-strix-halo-comfyui-toolboxes) — images, tags, upstream helper scripts
- [Hugging Face model API](https://huggingface.co/docs/hub/api) — the `/api/models/<repo>` endpoint `bin/model-add` uses to list files
