# amd-pstate-idle-floor

CPU idle frekvencia padlójának leszállítása `amd-pstate-epp` driver-en, Ubuntu 24.04 LTS-en, AMD Ryzen 7000 sorozaton.

## Probléma

`amd-pstate-epp` `active` módban a driver alapértelmezetten a CPPC *Lowest Non-linear Frequency* értékét használja policy-minimumnak (Ryzen 9 7950X3D-n ez ~2.99 GHz), nem a hardveres minimumot (~417 MHz). Emiatt idle gépen is a core-ok 3 GHz körül ragadnak, ami felesleges fogyasztás és hő.

## Tesztelt környezet

- AMD Ryzen 9 7950X3D
- Ubuntu 24.04 LTS Server (minimal install)
- Kernel 6.17 (HWE)
- `amd-pstate-epp` driver, `active` mód
- `powersave` governor, `balance_performance` EPP

## Diagnózis

```bash
# Driver ellenőrzése — amd-pstate-epp legyen
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

# Mód — active legyen
cat /sys/devices/system/cpu/amd_pstate/status

# Aktuális governor és EPP
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference

# Hardveres limitek vs. policy limitek összehasonlítása
cpupower frequency-info
```

A `cpupower frequency-info` kimenetében figyeld meg ezt a részt:

```
amd-pstate limits:
  Lowest Non-linear Frequency: 2.99 GHz   <-- ezt használja policy-minimumnak
  Lowest Frequency:              545 MHz   <-- ez a tényleges hardveres alja
```

És:

```
current policy: frequency should be within 2.99 GHz and 5.76 GHz.
```

Ha a `current policy` alsó határa magasabb, mint a `cpuinfo_min_freq` értéke, akkor jogos a fix.

## Megoldás

A `scaling_min_freq`-et a `cpuinfo_min_freq` értékére állítjuk (417 MHz a 7950X3D esetén), így a `powersave` governor lemehet egészen a hardveres aljáig.

### Manuális, perzisztens nélkül (teszteléshez)

```bash
echo 417000 | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_min_freq
```

Reboot után visszaáll a default.

### Perzisztens, systemd unit-tal

Lásd [`cpu-min-freq.service`](cpu-min-freq.service).

Telepítés:

```bash
sudo cp cpu-min-freq.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cpu-min-freq.service
```

Ellenőrzés:

```bash
systemctl status cpu-min-freq.service
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq   # 417000
grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
```

Idle-ben a legtöbb core 417 MHz-en kell legyen. Néhány core (általában a preferred core-ok, lásd `Preferred Core Ranking` a `cpupower` kimenetben) magasabb frekvencián marad — ez normális, a CPPC tartja őket készenlétben gyors válaszhoz.

## Más CPU-n

A `cpuinfo_min_freq` értéke CPU-tól függ. Ha más Zen 3/4/5 procit használsz, ezt írd be a service-be:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq
```

Az ott látott számot (kHz-ben) használd a service `ExecStart` sorában.

## Visszavonás

```bash
sudo systemctl disable --now cpu-min-freq.service
sudo rm /etc/systemd/system/cpu-min-freq.service
sudo systemctl daemon-reload
sudo reboot
```

## Megjegyzések

- Az EPP (`balance_performance`) változatlan marad — terhelésnél a CPU továbbra is gyorsan felugrik max boost-ra.
- A megoldás csak a *padlót* tolja le, a felső határt és a skálázás dinamikáját nem befolyásolja.
- `acpi-cpufreq` driver-en nem releváns, az másképp működik.

## Referenciák

- [AMD P-State driver kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/pm/amd-pstate.html)
- [Ubuntu Server cpupower docs](https://documentation.ubuntu.com/server/explanation/performance/perf-tune-cpupower/)
- [Arch Wiki: CPU frequency scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling)
