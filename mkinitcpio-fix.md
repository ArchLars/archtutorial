# OPTIONAL: How to fix those annoying 'missing firmware' warnings in mkinitcpio

#### 0) Find the module names that warn

```bash
# So everytime you run mkinicpio -P it warns you about a bunch of "missing firmware" 
# but none of these are important, they are ancient. You probably won't need them.
#
# IF you are unsure that you might need them or if the ones you are seeing are the ones
# I am talking about or actual problems, then you can find a list of them here :
#
# https://wiki.archlinux.org/title/Mkinitcpio#Possibly_missing_firmware_for_module_XXXX
#
# To see them, run this, and you'll see multiple lines ala:
# "Possibly missing firmware for module: qla2xxx"
# Write down all you see and confirm that you won't need them.
#
sudo mkinitcpio -P
```

#### OPTION A) Install all the firmware from the AUR (which will take up space)
```bash
yay -S --needed --noconfirm mkinitcpio-firmware
```

#### OPTION B) Copy a script that silences them by making dummy firmware
```bash
# The Arch Wiki recommends instead writing dummy files manually for them
#
# I wrote a script that creates harmless dummy firmware files for you
# It automatically captures the ones on a single run and then writes dummies for all of them
# then you run mkinitcpio again to build with the dummies to never see them ever again
#
# First open nano like so, which will create a new file:
#
sudo nano /usr/local/sbin/mkinitcpio-silence-missing-fw
```

#### 1.5) Then paste the script with CTRL + SHIFT + V
```bash
# ----- /usr/local/sbin/mkinitcpio-silence-missing-fw -----
##!/usr/bin/env bash
set -euo pipefail

FWROOT="/usr/lib/firmware"
LIST="/etc/mkinitcpio.local-firmware-ignore"
MARKER="### mkinitcpio-dummy-fw created, remove if you add matching hardware ###"
LOG="/var/tmp/mkinitcpio-warnings.$$.log"
HOST="$(uname -n 2>/dev/null || echo unknown-host)"

usage() {
  cat <<'USAGE'
Usage:
  mkinitcpio-silence-missing-fw         Run mkinitcpio, show output live, detect modules that warn, create dummy firmware only for those modules not in use.
  mkinitcpio-silence-missing-fw --undo  Remove only the dummy firmware files previously created by this tool.
Notes:
  - Future new warnings remain visible, this only touches modules found in this run.
  - Modules currently loaded are skipped.
USAGE
}

undo() {
  if [[ ! -f "$LIST" ]]; then
    echo "Nothing to undo, $LIST not found."
    exit 0
  fi
  while IFS= read -r f; do
    [[ -z "$f" || ! -e "$f" ]] && continue
    if grep -q "$MARKER" "$f" 2>/dev/null; then
      rm -v -- "$f"
    else
      echo "Skip non-dummy file: $f"
    fi
  done < "$LIST"
  : > "$LIST"
  echo "Removed recorded dummy firmware files."
}

# Try modinfo with common dash/underscore variants
get_fw_list() {
  local m="$1"
  # print unique non-empty firmware paths to stdout
  for name in "$m" "${m//-/_}" "${m//_/-}"; do
    if mapfile -t _x < <(modinfo -F firmware "$name" 2>/dev/null); then
      printf '%s\n' "${_x[@]}" | sed '/^$/d' | sort -u
      return 0
    fi
  done
  return 1
}

create_from_warnings() {
  echo "Running mkinitcpio -P (output will be shown and logged to $LOG)..."
  set +e
  mkinitcpio -P 2>&1 | tee "$LOG"
  status=${PIPESTATUS[0]}
  set -e
  if [[ $status -ne 0 ]]; then
    echo "mkinitcpio exited with status $status, continuing (warnings were captured)."
  fi

  # Extract module names from warning lines
  declare -A seen=()
  modules=()
  while IFS= read -r line; do
    case "$line" in
      *"Possibly missing firmware for module"*)
        m="${line##*: }"
        m="${m#"${m%%[![:space:]]*}"}"; m="${m%"${m##*[![:space:]]}"}"
        [[ "$m" == \"*\" && "$m" == *\" ]] && m="${m:1:${#m}-2}"
        [[ "$m" == \'*\' && "$m" == *\' ]] && m="${m:1:${#m}-2}"
        if [[ -n "$m" && -z "${seen[$m]+x}" ]]; then
          seen[$m]=1
          modules+=("$m")
        fi
      ;;
    esac
  done < "$LOG"

  if [[ ${#modules[@]} -eq 0 ]]; then
    echo "No 'Possibly missing firmware' warnings found in this run."
    echo "Log: $LOG"
    return 0
  fi

  echo "Modules with warnings: ${modules[*]}"
  touch "$LIST"

  for module in "${modules[@]}"; do
    # Skip if the module is actually in use
    if lsmod | grep -qw "$module"; then
      echo "SKIP $module (module is loaded, will not create dummies for hardware in use)"
      continue
    fi

    mapfile -t fws < <(get_fw_list "$module" || true)
    if [[ ${#fws[@]} -eq 0 ]]; then
      echo "No firmware filenames reported by $module, nothing to create."
      continue
    fi

    for fw in "${fws[@]}"; do
      target="$FWROOT/$fw"
      mkdir -p "$(dirname "$target")"
      if [[ -e "$target" ]]; then
        echo "Exists: $target, leaving as is."
        continue
      fi
      {
        echo "$MARKER"
        echo "Dummy firmware for module $module on $HOST."
        echo "If you later add matching hardware, delete this file and install the proper firmware package."
      } > "$target"
      chmod 0644 "$target"
      echo "$target" >> "$LIST"
      echo "Created dummy: $target"
    done
  done

  echo "Recorded created files in $LIST"
  echo "Done. You can now rebuild: sudo mkinitcpio -P"
  echo "Full mkinitcpio output is in: $LOG"
}

case "${1:-}" in
  --undo) undo ;;
  -h|--help) usage ;;
  *) create_from_warnings ;;
esac
```
#### 2) Make it executable
```bash
sudo chmod +x /usr/local/sbin/mkinitcpio-silence-missing-fw
```
#### 3) Run & Undo

```bash
## 1) FIRST run this to write the dummies for the ancient modules warned on your machine
sudo /usr/local/sbin/mkinitcpio-silence-missing-fw

## 2) THEN run this to confirm, and you are done
sudo mkinitcpio -P

# Disclaimer: there is no guarantee that what its writing a dummy for is one of these ancient modules
# You should confirm with the wiki first before running this script.
---

## 1) If you ever want to undo
sudo /usr/local/sbin/mkinitcpio-silence-missing-fw --undo

## 2) And if so, run to confirm.
sudo mkinitcpio -P
```
---
