# amd-pstate-idle-floor

Lowering the CPU idle frequency floor on the `amd-pstate-epp` driver, on Ubuntu 24.04 LTS, for AMD Ryzen 7000 series.

## Problem

In `active` mode, the `amd-pstate-epp` driver uses the CPPC *Lowest Non-linear Frequency* as the policy minimum by default (~2.99 GHz on a Ryzen 9 7950X3D), not the hardware minimum (~417 MHz). As a result, even on an idle system, cores stay around 3 GHz, which causes unnecessary power draw and heat.

## Tested environment

- AMD Ryzen 9 7950X3D
- Ubuntu 24.04 LTS Server (minimal install)
- Kernel 6.17 (HWE)
- `amd-pstate-epp` driver, `active` mode
- `powersave` governor, `balance_performance` EPP

## Diagnosis

```bash
# Verify driver — should be amd-pstate-epp
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

# Mode — should be active
cat /sys/devices/system/cpu/amd_pstate/status

# Current governor and EPP
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference

# Compare hardware limits vs. policy limits
cpupower frequency-info
```

In the `cpupower frequency-info` output, look at this section:

```
amd-pstate limits:
  Lowest Non-linear Frequency: 2.99 GHz   <-- used as policy minimum
  Lowest Frequency:              545 MHz   <-- actual hardware floor
```

And:

```
current policy: frequency should be within 2.99 GHz and 5.76 GHz.
```

If the lower bound of `current policy` is higher than `cpuinfo_min_freq`, the fix applies.

## Solution

Set `scaling_min_freq` to `cpuinfo_min_freq` (417 MHz on the 7950X3D), so the `powersave` governor is allowed to scale all the way down to the hardware floor.

### Manual, non-persistent (for testing)

```bash
echo 417000 | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_min_freq
```

Reverts to default after reboot.

### Persistent, via systemd unit

See [`cpu-min-freq.service`](cpu-min-freq.service).

Install:

```bash
sudo cp cpu-min-freq.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cpu-min-freq.service
```

Verify:

```bash
systemctl status cpu-min-freq.service
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq   # 417000
grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
```

When idle, most cores should report 417 MHz. A few cores (typically the preferred cores — see `Preferred Core Ranking` in the `cpupower` output) will stay at a higher frequency. This is normal: CPPC keeps them ready for fast response to incoming load.

## On a different CPU

The `cpuinfo_min_freq` value depends on the CPU. For other Zen 3/4/5 chips, read the actual value:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq
```

Use that number (in kHz) in the service's `ExecStart` line.

## Reverting

```bash
sudo systemctl disable --now cpu-min-freq.service
sudo rm /etc/systemd/system/cpu-min-freq.service
sudo systemctl daemon-reload
sudo reboot
```

## Notes

- EPP (`balance_performance`) is left unchanged — under load, the CPU still ramps up to maximum boost just as quickly as before.
- This fix only lowers the *floor*; it does not affect the upper bound or the scaling dynamics.
- Not relevant for the `acpi-cpufreq` driver — that one works differently.

## References

- [AMD P-State driver kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/pm/amd-pstate.html)
- [Ubuntu Server cpupower docs](https://documentation.ubuntu.com/server/explanation/performance/perf-tune-cpupower/)
- [Arch Wiki: CPU frequency scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling)
