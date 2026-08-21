# GP77 13VG: Arreglo del zumbido de audio en Ubuntu (`snd_intel_dspcfg.dsp_driver=1`)

- **Fecha:** 2026-08-05
- **Fuente:** Luis, arreglo práctico en el GP77 13VG (i7-13700H + RTX 4070), Ubuntu. Verificado en vivo.
- **Estado:** Funcionando. El zumbido fuerte al arrancar desapareció.

## Síntoma

Zumbido/ruido fuerte al iniciar Ubuntu en el GP77 13VG. Causa raíz: la pila de audio de Intel comportándose mal en este hardware (driver DSP SOF Sound Open Firmware contra HDA legacy).

## Arreglo

Forzar el driver Intel HDA legacy en vez del driver DSP mediante parámetro de kernel:

```
snd_intel_dspcfg.dsp_driver=1
```

Valores: `0` = automático (por defecto de la plataforma), `1` = HDA legacy, `2` = SST, `3` = SOF.

## Pasos

```bash
sudo nano /etc/default/grub
```

Agregar el parámetro a `GRUB_CMDLINE_LINUX_DEFAULT`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd_intel_dspcfg.dsp_driver=1"
```

Aplicar (Ubuntu):

```bash
sudo update-grub
```

Reiniciar. El zumbido debería desaparecer.

## Notas

- Si se reinstala el SO o una actualización de kernel/grub reescribe la línea de comandos, el parámetro debe volver a agregarse.
- Este es el **laptop gaming** GP77, no es un nodo del homelab. No confundir con el servidor Proxmox.
