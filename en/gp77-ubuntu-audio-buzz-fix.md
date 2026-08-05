# GP77 13VG — Ubuntu Audio Buzz Fix (`snd_intel_dspcfg.dsp_driver=1`)

- **Date:** 2026-08-05
- **Source:** Luis — hands-on fix on the GP77 13VG (i7-13700H + RTX 4070), Ubuntu. Verified live.
- **Status:** Working — loud buzzing at boot gone.

## Symptom

Loud buzzing/static noise when booting Ubuntu on the GP77 13VG. Root cause: the Intel audio stack — SOF (Sound Open Firmware) DSP driver vs legacy HDA — misbehaving on this hardware.

## Fix

Force the legacy Intel HDA driver instead of the DSP driver via kernel parameter:

```
snd_intel_dspcfg.dsp_driver=1
```

Values: `0` = auto (platform default), `1` = legacy HDA, `2` = SST, `3` = SOF.

## Steps

```bash
sudo nano /etc/default/grub
```

Add the parameter to `GRUB_CMDLINE_LINUX_DEFAULT`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd_intel_dspcfg.dsp_driver=1"
```

Apply (Ubuntu):

```bash
sudo update-grub
```

Reboot. Buzz should be gone.

## Notes

- If the OS is reinstalled or a kernel/grub update rewrites the cmdline, the parameter must be re-added.
- This is the GP77 **gaming laptop** — not a homelab node. Do not confuse with the Proxmox box.
