# Sensor chain timing

Two separate questions that are easy to conflate:

1. How fast does the **amplifier** produce a new number? (MAX31865 conversion rate)
2. How fast does the **probe** notice that the world changed? (PT100 thermal response)

The second is 145× slower than the first, and that ratio is the whole point of
measuring both.

---

## 1. MAX31865 conversion rate

**Method.** The DRDY pin on the breakout goes LOW when a new conversion result is
available in the data register, and returns HIGH when that register is read. In
continuous mode the cycle is:

```
chip finishes conversion  ->  DRDY goes LOW
Pi reads the register     ->  DRDY returns HIGH
chip starts the next conversion
```

Timing successive falling edges gives the conversion interval directly, without
assuming anything about the Pi's polling rate.

**Data.** `data/Conversion Time Test Data`, 100 pulses.

| | Value |
|---|---|
| Mean interval | 16.689 ms |
| Median | 16.582 ms |
| Standard deviation | 1.524 ms |
| Minimum | 14.006 ms |
| Maximum | 19.356 ms |

16.689 ms is **59.92 Hz** against the 60 Hz continuous conversion rate the part
is configured for. Agreement to 0.13%.

**On the spread.** 1.5 ms of standard deviation is larger than a hardware-timed
conversion should produce on its own. The measurement includes however long the
Pi took to notice each falling edge, and Raspberry Pi OS is not a real-time
system. So this number bounds the amplifier's jitter from above rather than
measuring it. Separating the two would need a hardware timestamp on the edge —
worth doing if the conversion jitter ever mattered, which so far it does not.

---

## 2. PT100 thermal response

**Method.** Step change in the sensor's environment, logging resistance and
temperature continuously and expressing each sample as a percentage of the total
step.

**Data.** `data/Thermal Response Data`, 1,798 samples over 30 s.

| | Time |
|---|---|
| 63.2% of step (one time constant) | 2.42 s |
| 90% of step | 7.86 s |
| Settled | ~30 s |

So **τ ≈ 2.4 s**.

For a first-order thermal system, 63.2% is one time constant and 90% should
arrive at 2.30 τ. Measured 90% at 7.86 s against 2.30 × 2.42 = 5.57 s predicted,
so the response is slower in the tail than a single exponential.

That is expected and it is informative. A sheathed probe is not one thermal mass,
it is at least two in series: the sheath and the element inside it. Two time
constants in series produce exactly this shape — close to first-order early,
dragging late. Fitting a two-pole model to this data would give both, and that
would be a better description than the single number above.

**What it depends on.** This τ is for this probe in whatever medium the step was
applied in. In a different fluid, at a different flow rate, or potted into a
different fitting, the number changes — the sheath-to-medium coupling is usually
the dominant term. The ratio between probe time and electronics time is robust;
the 2.4 s itself is not transferable.

---

## 3. Why the comparison matters

| Stage | Time | Relative |
|---|---|---|
| Modbus round trip to the PLC | 0.6 ms | 1× |
| MAX31865 conversion | 16.7 ms | 28× |
| PT100 thermal response | 2,420 ms | 4,000× |

For any control loop closed on temperature with this hardware, the probe
dominates by three orders of magnitude. Reducing the PLC scan cycle from 20 ms to
1 ms improves the loop by less than one part in a hundred. That effort is better
spent on probe mounting and thermal contact, which is where the time constant
actually lives.

The general rule, and the reason this was worth measuring at all: **find the
slowest element before optimising any of the others.** The latency work in the
[PLC benchmarks repo](https://github.com/williamsni14463/raspberry-pi-plc-benchmarks)
is genuinely useful, but for pressure, not for temperature. A piezoresistive
pressure sensor has a natural frequency in the hundreds of kilohertz, which flips
the table completely and makes the communication path the limiting element.

Same hardware, same software, opposite conclusion, depending on which sensor is
in the loop.
