# cell_monitor

**cell_monitor is the core RF measurement component of [OpenCellSurvey (OCS)](https://github.com/lmesserStep/OpenCellSurvey).** It handles all physical layer work. Scanning LTE bands, decoding the MIB, and streaming RSRP/RSRQ/SNR measurements as newline delimited JSON, so the OCS pipeline can ingest, store, and visualize signal data without coupling to SDR hardware directly.

Written against [srsRAN](https://github.com/srsran/srsRAN_4G) and targets Ettus USRP B200/B210 hardware.

---

## Hardware Requirements

- **SDR**: Ettus Research USRP B200 or B210 (hardcoded `type=b200` RF args), though this is because that's the only hardware that I have. 
- **USB 3.0** port (required for B200/B210 throughput)
- **Host OS**: Linux — Ubuntu 20.04 or 22.04 LTS recommended

---

## Software Dependencies

| Dependency | Purpose | Install / Build |
|---|---|---|
| **srsRAN 4G** | LTE PHY, cell search, UE sync | Build from source (see below) |
| **UHD (USRP Hardware Driver)** | B200/B210 RF driver | `apt install libuhd-dev uhd-host` |
| **libfftw3** | FFT (required by srsRAN) | `apt install libfftw3-dev` |
| **CMake ≥ 3.10** | Build system | `apt install cmake` |
| **GCC or Clang** | C/C++ compiler | `apt install build-essential` |

### Install all apt dependencies at once

```bash
sudo apt update
sudo apt install -y build-essential cmake libfftw3-dev libuhd-dev uhd-host \
    libboost-all-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev
```

### Build and install srsRAN 4G

```bash
git clone https://github.com/srsran/srsRAN_4G.git
cd srsRAN_4G
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
sudo ldconfig
```

This installs srsRAN headers to `/usr/local/include/srsran/` and libraries to `/usr/local/lib/`.

---

## Compiling cell_monitor

### Option A — Standalone gcc (quickest)

```bash
gcc -O2 -std=c11 -o cell_monitor cell_monitor.c \
    -I/usr/local/include \
    -L/usr/local/lib \
    -lsrsran_phy \
    -lsrsran_rf \
    -lsrsran_common \
    -luhd \
    -lm \
    -lpthread \
    -lfftw3f
```

If srsRAN was installed to a custom prefix (e.g. `/opt/srsran`), replace `/usr/local` with that path and set:

```bash
export LD_LIBRARY_PATH=/opt/srsran/lib:$LD_LIBRARY_PATH
```

### Option B — CMake (if integrating into a larger project)

Create a minimal `CMakeLists.txt` alongside `cell_monitor.c`:

```cmake
cmake_minimum_required(VERSION 3.10)
project(cell_monitor C)

find_package(srsran REQUIRED)
find_package(Threads REQUIRED)

add_executable(cell_monitor cell_monitor.c)
target_link_libraries(cell_monitor
    srsran_phy srsran_rf srsran_common
    Threads::Threads m fftw3f)
```

Then:

```bash
mkdir build && cd build
cmake ..
make
```

---

## Usage

```
Usage: cell_monitor -b <band> [options]

Required:
  -b <band>     LTE band number

Search options:
  -p <pci>      Target specific PCI (faster)
  -f            Fast mode
  -n <frames>   PSS frames per channel
  -t <psr>      Min PSR threshold (default: 2.0)
  -k            Skip channels with low RSSI

TDD options:
  -T            Force TDD mode
  -F <freq>     Manual frequency in MHz
  -C <config>   TDD UL/DL config 0-6 (default: 2)
  -S <config>   TDD special SF config 0-9 (default: 7)

Bandwidth:
  -w <prb>      Force bandwidth: 6, 15, 25, 50, 75, 100

RF options:
  -a <args>     Additional RF args passed to UHD
  -d <device>   RF device name
  -g <gain>     RX gain in dB (default: 60)

Range:
  -s <earfcn>   Start EARFCN
  -e <earfcn>   End EARFCN

Debug:
  -V            Verbose error output
  -v            Increase srsRAN verbose level
```

### Examples

```bash
# Scan FDD Band 2 for a specific cell
sudo ./cell_monitor -b 2 -p 67

# Scan TDD Band 48 (CBRS), force 100 PRB (20 MHz) bandwidth
sudo ./cell_monitor -b 48 -p 1 -w 100

# Band 48 with explicit TDD UL/DL config 2, special SF config 7
sudo ./cell_monitor -b 48 -p 1 -w 100 -C 2 -S 7

# Fast scan of Band 4, skip quiet channels, verbose errors
sudo ./cell_monitor -b 4 -f -k -V
```

> **Note:** Running as root (`sudo`) is typically required for real-time USB access to the B210.

---

## Output Format

Each successful measurement prints a JSON object on stdout:

```json
{"t_ms":1716900000123,"band":48,"earfcn":55240,"freq_mhz":3560.00,"pci":1,"n_prb":100,"ports":2,"rsrp_dbm":-85.3,"rsrq_db":-10.1,"snr_db":18.4,"cfo_hz":142.7,"frame_type":"TDD"}
```

Measurement failures emit an `"error":"measure_failed"` key instead of signal metrics.

---

## Signal Handling

- **Ctrl+C once** — graceful stop (finishes current measurement)
- **Ctrl+C twice** — forces RF stream stop
- **Ctrl+C three times** — immediate exit (`_exit(1)`)

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Failed to open RF device` | UHD not finding B210 | Run `uhd_find_devices`; check USB 3.0 connection |
| `MIB failed after 3 retries` | BW unknown | Use `-w <prb>` to force bandwidth |
| No cells found | Wrong band / gain | Try `-g 70` or `-g 50`; verify EARFCN range |
| `libsrsran_phy.so not found` | Library path missing | `sudo ldconfig` or set `LD_LIBRARY_PATH` |
| TDD cells show only errors | Wrong TDD config | Try `-C 1` or `-C 2` with matching `-S` |

---

## License

`cell_monitor` is licensed under the **GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later)**.

This is required because it links against [srsRAN 4G](https://github.com/srsran/srsRAN_4G), which is itself AGPL-3.0. Any project that incorporates or distributes this code — including network services — must also release their source under AGPL-3.0.

See [LICENSE](LICENSE) for the full license text.
