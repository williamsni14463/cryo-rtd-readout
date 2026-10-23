# Cryogenic Temperature Readout — PT100 and Cernox

Temperature instrumentation for the MAMBA bubble chamber prototype at Fermilab:
PT100 RTDs read through MAX31865 amplifiers on a Raspberry Pi, and a Cernox
CX-1080 read through a Lake Shore 240.

Two sensors because they cover different ground. A PT100 is cheap, linear, and
useless below about 70 K where its resistance flattens out. A Cernox stays
sensitive down to the liquid-helium range but needs an instrument that knows its
calibration curve. The chamber needs both: PT100s for the warm plumbing and
cooldown monitoring, Cernox for the cold mass.

---

## What came out of it

**The MAX31865 does what it claims.** Timed against the DRDY pin in continuous
conversion mode: 16.689 ms mean interval over 100 conversions, σ = 1.524 ms. That
is 59.92 Hz against a nominal 60 Hz.

Some of that 1.5 ms spread is the Pi's own interrupt latency detecting the DRDY
edge rather than jitter in the chip, so treat it as an upper bound on the
amplifier's variation rather than a measurement of it.

**The probe, not the electronics, sets the response time.** A step change in
sensor temperature reached 63.2% in **2.42 s** and 90% in 7.86 s, so the PT100
time constant is about 2.4 seconds.

| Stage | Time |
|---|---|
| Modbus round trip to the PLC | 0.6 ms |
| MAX31865 conversion | 16.7 ms |
| **PT100 thermal response** | **2,420 ms** |

The probe is 145× slower than the conversion and roughly 4,000× slower than the
communication. Any effort spent optimising scan cycles or communication latency
for a temperature loop on this hardware is wasted — the thermal mass of the probe
dominates by three orders of magnitude.

That is specific to temperature. The same reasoning applied to pressure gives the
opposite answer, because a piezoresistive pressure sensor responds in
microseconds and the communication path becomes the limit.

---

## The thing that cost the most time

Two MAX31865 amplifiers share the Pi's SPI bus. SPI is a bus: SDI, SDO and CLK
are common to every device on it, and the Pi selects which one it is talking to
by pulling that device's **CS (chip select)** line low.

So two amplifiers need two separate CS pins. Everything else is shared. Wiring
both CS lines to the same GPIO means both chips answer at once and the data is
garbage in a way that looks like a bad sensor rather than a bad bus.

Details and the working pin map are in
[docs/dual-sensor-setup.md](docs/dual-sensor-setup.md).

Separately, and worse: **starting OpenPLC with the hardware layer set to
"Raspberry Pi" destroys the SPI bus.** The runtime hardcodes GPIO 5, 9, 10 and 11
to digital inputs and 7, 8 to outputs the instant the PLC starts, regardless of
what the program does. PT100 readings go to 0 Ω and −242 °C and only a full power
cycle recovers them. The fix is to set the hardware layer to "Blank for Linux".
Full diagnosis with before/after pin states is in the
[PLC benchmarks repo](https://github.com/williamsni14463/raspberry-pi-plc-benchmarks).

---

## Reading a Cernox

The CX-1080 goes through a Lake Shore 240-8P rather than a MAX31865, because
below about 70 K a PT100's resistance curve flattens and the sensor stops being
able to tell you much.

The 240 is a Windows-only instrument as shipped. Getting it to talk to Linux
needed a driver, which is its own repo:
[lakeshore240-linux](https://github.com/williamsni14463/lakeshore240-linux).

`calibration/X208461.340` is the factory curve for Cernox serial X208461
(CX-1080-SD-HT-20L), 100 breakpoints in log ohms versus Kelvin, valid to 325 K.
The instrument needs this loaded against the right channel or it will apply some
other sensor's curve to your resistance and report a confident wrong number.

---

## Layout

```
docs/        Setup and experiment writeups
scripts/     Sensor checks, timing tests, data reduction
data/        Raw CSV from each run
calibration/ Factory sensor curves
```

Start with [docs/pt100-single-sensor.md](docs/pt100-single-sensor.md) for one
sensor, then [docs/dual-sensor-setup.md](docs/dual-sensor-setup.md) for two.

---

## Hardware

- PT100 RTD, 4-wire, through an Adafruit MAX31865 (430 Ω reference resistor)
- Cernox CX-1080-SD-HT-20L through a Lake Shore 240-8P
- Raspberry Pi 4, Raspberry Pi OS Bookworm 64-bit

Enable SPI with `sudo raspi-config` → Interface Options → SPI, then confirm
`/dev/spidev0.0` and `/dev/spidev0.1` exist.

## Software

```bash
pip3 install --break-system-packages adafruit-blinka \
    adafruit-circuitpython-max31865
```

Bookworm's pip guard means installs need `--break-system-packages` or a virtual
environment. If you use a venv, remember it needs reactivating in every new
terminal before any script will run.

---

## Sanity checks worth knowing

A PT100 reads about 100 Ω at 0 °C and climbs roughly 0.385 Ω/°C, so 108–110 Ω is
normal at room temperature.

**−242 °C with near-zero resistance is a wiring fault**, not a cold sensor. It
means the amplifier sees an open or shorted RTD. Check for bridged pads before
suspecting anything else.

If room temperature reads about 23 °C, touch the sensor and watch it climb. That
single test catches wiring, SPI, and curve problems in about five seconds.
