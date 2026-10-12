# snom D785 — Hardware & Firmware Interface Reference

A practical reference for driving the snom D785 desk phone's hardware from Linux user space, detailed
enough to build an alternative firmware application (a replacement for the stock telephony app). It
covers every device node, sysfs path, input code, LED protocol and the on-board audio control protocol,
with concrete examples.

- **SoC:** Broadcom **BCMRing** (ARMv6, soft-float, single core).
- **Kernel:** Linux **2.6.27.47**. The interfaces below are from that era (legacy fbdev, sysfs GPIO,
  evdev).
- **Telephony DSP:** a Broadcom **EPT** endpoint plus **HAL Audio**, exposed as `/dev/ept0` and
  `/dev/halaudio`. Those are held open exclusively by the on-board **`media_daemon`** process. Audio is
  controlled by talking to `media_daemon` over its **LPCP** socket — you do not open the DSP yourself.

A minimal replacement application therefore needs to: draw to two framebuffers, drive the LEDs, read
the keypad (including the hook), and drive audio/RTP through `media_daemon`. Each is below.

Byte-order note: this is a **little-endian** ARM target. RTP and the SRTP keys are big-endian on the
wire (shown explicitly). `struct input_event` and framebuffer pixels are native (little-endian).

---

## 1. Main display — `/dev/fb0`

Standard Linux fbdev: **480 × 272, 32 bpp**, pixel layout `0x00RRGGBB` (the top byte is ignored by the
panel). Open, query geometry, `mmap`, and write pixels directly — the mapped memory is the visible
buffer, so a write is immediately on screen. For tear-free updates, render into a RAM backbuffer and
`memcpy` whole frames.

```c
#include <linux/fb.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <stdint.h>

int fd = open("/dev/fb0", O_RDWR);
struct fb_var_screeninfo v; struct fb_fix_screeninfo f;
ioctl(fd, FBIOGET_VSCREENINFO, &v);          /* v.xres=480, v.yres=272, v.bits_per_pixel=32 */
ioctl(fd, FBIOGET_FSCREENINFO, &f);          /* f.line_length = bytes per row */
uint8_t *fb = mmap(0, f.smem_len, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

static inline void putpixel(uint8_t *fb, const struct fb_fix_screeninfo *f,
                            int x, int y, uint32_t rgb) {
    uint32_t *row = (uint32_t *)(fb + y * f->line_length);
    row[x] = rgb | 0xff000000u;              /* force opaque; top byte ignored by HW */
}
```

Fonts, anti-aliasing and alpha compositing are entirely up to the application (composite in your
backbuffer before writing). **Backlight:** `/sys/class/backlight/<dev>/bl_power` (blank/unblank) and
`brightness` (0..`max_brightness`), e.g. `echo 0 > .../bl_power; echo 200 > .../brightness`.

## 2. Side display — `/dev/fb1` — and the line keys

The narrow vertical display beside the six line keys is an SPI panel (`snom_fb2.0`): the framebuffer
reports **254 × 100, 1 bpp**, but the panel is mounted so the natural orientation is **portrait,
100 × 254**. Render portrait into your own buffer and rotate 90° when copying to the device.

Rotation used in practice (logical portrait `W=100 × H=254` → device `254 × 100`):

```c
/* logical pixel (lx, ly), lx in [0,100), ly in [0,254)  ->  device (px, py) */
int px = 254 - 1 - ly;      /* device X, 0..253 */
int py = lx;                /* device Y, 0..99  */
dev32[py * 254 + px] = logical32[ly * 100 + lx];
```

Then pack the rotated 32-bit image down to the panel's **1 bpp**: **8 pixels per byte, MSB = leftmost**,
true stride `(width + 7) / 8` (the fbdev `line_length` is not usable for this panel). Threshold on
luminance (e.g. `lum = (R*77 + G*150 + B*29) >> 8; bit = lum > 96`):

```c
int stride = (254 + 7) / 8;
for (int y = 0; y < 100; y++) {
    uint8_t *drow = devfb + (size_t)y * stride;
    memset(drow, 0, stride);
    for (int x = 0; x < 254; x++) {
        uint32_t p = dev32[y*254 + x];
        unsigned lum = (((p>>16)&0xff)*77 + ((p>>8)&0xff)*150 + (p&0xff)*29) >> 8;
        if (lum > 96) drow[x>>3] |= (uint8_t)(0x80 >> (x & 7));
    }
}
```

Unblank: `ioctl(fd, FBIOBLANK, FB_BLANK_UNBLANK)` and `echo 0 > /sys/class/graphics/fb1/blank`. Its
backlight is unusual — `/sys/class/backlight/<dev>/bl_power` where **the value is the level** (≈ 0–3 =
off, 4..30 = on), not a blank flag.

**Line keys:** the six hardware keys next to this panel emit ordinary input key codes (see §5), top →
bottom: **`100, 99, 98, 86, 85, 83`**.

## 3. Line-key LEDs — `/dev/snomled`

The six two-colour (green/red) line-key LEDs are driven through a dedicated character device,
**`/dev/snomled`**, with a simple binary protocol — not GPIO. Each LED is addressed by its key number
`n` (1–6); the green and red elements are set independently by writing **two 4-byte records**:

```
green element :  { 0x00, 0x00,   n,   g }     # g = 1 to light key n's green element, 0 to clear
red   element :  {   n,   r,  0x00, 0x00 }     # r = 1 to light key n's red element,   0 to clear
```

```c
/* colour: 0=off, 1=green, 2=red, 3=orange */
void set_linekey_led(int n /*1..6*/, int colour) {
    int fd = open("/dev/snomled", O_WRONLY);
    unsigned char g = colour & 1, r = (colour >> 1) & 1;
    unsigned char a[4] = { 0, 0, (unsigned char)n, g };
    unsigned char b[4] = { (unsigned char)n, r, 0, 0 };
    write(fd, a, 4); write(fd, b, 4);
    close(fd);
}
```

Visible states: **off / green / red / orange** (green + red). The device has no blink mode; blink in
software by rewriting the records with the element forced off on the off-phase (e.g. toggle at ~2 Hz).

## 4. Status / function LEDs — GPIO (sysfs)

The fixed indicator LEDs are exported **GPIO output** lines via the legacy sysfs GPIO interface:

```sh
echo 32 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio32/direction
echo 1   > /sys/class/gpio/gpio32/value      # Ring/Call LED on
```

| LED                         | GPIO |
|-----------------------------|------|
| Message / voicemail (MWI)   | 18   |
| Mute                        | 21   |
| Speaker (hands-free)        | 27   |
| Headset                     | 28   |
| Ring / call indicator       | 32   |

The SoC has GPIOs 0–60 on two controllers (`gpiochip0`, base 0, 32 lines; `gpiochip32`, base 32, 29
lines). Most lines are claimed by drivers and won't export; the five above are the LED outputs.

## 5. Keypad and key codes

All front-panel keys arrive as Linux `EV_KEY` events on **`/dev/input/event0`** (device name `keypad`).
Capabilities are `EV_SYN` + `EV_KEY` only — there is **no `EV_SW`**. You can `EVIOCGRAB` the device for
exclusive access. On this 32-bit kernel, `struct input_event` is **16 bytes** (`struct timeval` = two
32-bit words, then `__u16 type`, `__u16 code`, `__s32 value`). `value` = 1 press, 0 release, 2 repeat.

```c
#include <linux/input.h>
int fd = open("/dev/input/event0", O_RDONLY);
ioctl(fd, EVIOCGRAB, 1);                     /* optional: grab exclusively */
struct input_event ev;
while (read(fd, &ev, sizeof ev) == sizeof ev) {
    if (ev.type == EV_KEY && ev.value == 1)  handle_key(ev.code);   /* press */
    if (ev.type == EV_KEY && ev.code == 102) handle_hook(ev.value); /* hook, see §6 */
}
```

Key-code map:

```
Navigation / edit : UP=84  DOWN=67  LEFT=101  RIGHT=50  OK=120  X / Back=97
Digits            : 0=5  1=51  2=49  3=48  4=38  5=37  6=36  7=22  8=21  9=20   *=6   #=4
Soft keys         : SK1=116  SK2=115  SK3=114  SK4=113
Audio keys        : VOL+=119  VOL-=16  MUTE=3  SPEAKER=17  HEADSET=18
Paging keys       : PAGE-LEFT=54  PAGE-RIGHT=53
Function keys     : MENU/SETTINGS=19  DIRECTORY=33
Line keys         : 100  99  98  86  85  83   (top → bottom)
Message button    : 52
DND button        : 35
```

Notes: **35** is the dedicated DND key (not a generic home/exit key); **52** is the dedicated
message/voicemail key; the X/Back key is **97**.

## 6. Handset hook (lift-off detection)

The hook switch is **not** an `EV_SW` switch and **not** a readable GPIO. It is reported on the keypad
input device (`/dev/input/event0`) as an **`EV_KEY` event with code `102`**:

- `value == 0` → handset **lifted** (off-hook)
- `value == 1` → handset **on the cradle** (on-hook)

(On the cradle the switch is held "pressed" = 1; lifting releases it = 0.) Only transitions emit events;
the resting state is not announced at startup, so assume on-hook until the first event. There is no
hardware debounce guarantee — debounce in software if needed.

## 7. Audio control — `media_daemon` over LPCP

You never touch the codec/DSP directly. `media_daemon` owns `/dev/ept0` + `/dev/halaudio` and exposes a
control channel over **LPCP** on TCP **`127.0.0.1:1299`**. Connect a stream socket and exchange
line-oriented text messages framed as:

```
<VERB> LPCP/1.2\r\n
CSEQ: <monotonic integer>\r\n
<Header>: <value>\r\n
...
CLEN: <body-length>\r\n      (only if there is a body)
\r\n
<body>                       (optional)
```

Replies and asynchronous notifications come back on the same socket; correlate by `CSEQ` and/or match
the response verb (e.g. `RESPONSE_RTP_SETUP`). Example sender:

```c
int md = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in a = {0}; a.sin_family = AF_INET;
a.sin_port = htons(1299); a.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
connect(md, (struct sockaddr*)&a, sizeof a);

int cseq = 1;
void md_send(const char *verb, const char *body, const char *headers) {
    char m[2048]; int n = snprintf(m, sizeof m, "%s LPCP/1.2\r\nCSEQ: %d\r\n%s", verb, cseq++, headers);
    if (body) n += snprintf(m+n, sizeof m-n, "CLEN: %d\r\n\r\n%s", (int)strlen(body), body);
    else      n += snprintf(m+n, sizeof m-n, "\r\n");
    write(md, m, n);
}
```

### 7.1 Setting up a call's audio

Three steps: reserve a local RTP port, bind the stream to the remote endpoint, route the mix to an
output.

**(a) `RTP_SETUP`** — reserve a local RTP port; the response carries the chosen port in `RTP-ADDRESS`:

```
md_send("RTP_SETUP", NULL,
  "TRANSPORT: 0\r\nRTP-NAME: RP7\r\nRTCP-NAME: RC7\r\n"
  "PORT-MIN: 49152\r\nPORT-MAX: 65534\r\nTYPE-OF-SERVICES: 160\r\n");
/* read reply; expect RESPONSE_RTP_SETUP with e.g. "RTP-ADDRESS: Udp:[::]:51002" -> local port 51002 */
```

`RP7`/`RC7` are arbitrary stream names you choose and reuse in later commands.

**(b) `STREAM_SETUP`** — bind the named stream to the remote media `IP:port` and configure codec/DTMF/
SRTP. Full header set that works:

```
md_send("STREAM_SETUP", NULL,
  "TRANSPORT: 0\r\nINDEX: 0\r\nSSRC: 1066511124\r\nRTP-NAME: RP7\r\nRTCP-NAME: RC7\r\n"
  "RTCP-XR: \r\nFORCE-RTCP-XR-MONITORING: 0\r\n"
  "RTP-ADDRESS: Udp:203.0.113.5:40000\r\nRTCP-ADDRESS: Udp:[::]:0\r\n"
  "READ: 1\r\nWRITE: 1\r\nHOLD: 0\r\nHOLD-STATE: 0\r\nRECORD_TO_USB: 0\r\nUSE_ENCRYPTION: 0\r\n"
  "CAPTURE_FILENAME: \r\nDTMF-TYPE: 2\r\nDTMF-PLTYPE: 101\r\n"
  "DO-SRTCP-TX: 0\r\nDO-SSRTP-RX: 0\r\nDO-SSRTP-TX: 0\r\n"
  "SRTP-KEY-RX: \r\nSRTP-KEY-TX: \r\nSRTP-TAG-LEN: 0\r\n"
  "SRTP-MKI-LEN-RX: 0\r\nSRTP-MKI-LEN-TX: 0\r\nSRTP-MKI-RX: 0\r\nSRTP-MKI-TX: 0\r\n"
  "FRAME-LENGTH: 160\r\nINITIAL-KEEPALIVES: 1\r\nTYPE-OF-SERVICES: 160\r\nG726-TYPE: 1\r\n");
md_send("RTP_SYMMETRIC", NULL, "NAME: RP7\r\nENABLED: 1\r\n");
```

- `RTP-ADDRESS: Udp:<ip>:<port>` is the **remote** RTP destination (from the far end's SDP).
- `FRAME-LENGTH: 160` = 20 ms of 8 kHz (PCMU). `DTMF-PLTYPE: 101` selects the RFC 2833 payload type.
- **SRTP** (SDES): set `USE_ENCRYPTION: 1`, put the raw 30-byte key+salt as hex in `SRTP-KEY-RX` /
  `SRTP-KEY-TX`, `SRTP-TAG-LEN: 10`, and `DO-SRTCP-TX: 1`. The SDP crypto suite is
  `AES_CM_128_HMAC_SHA1_80`.

**(c) `AUDIO_SETUP`** — route the mixed audio to an output and set volumes:

```
md_send("AUDIO_SETUP", NULL,
  "SPEAKER_DESTINATION: 6\r\nSPEAKER_VOLUME: 7\r\nIS_HIGH_SPEAKER_GAIN: 0\r\nVOLUME_FACTOR: 1\r\n"
  "MICROFONE_DESTINATION: 1\r\nMICROFONE_VOLUME: 5\r\nIS_HIGH_MIC_GAIN: 0\r\n"
  "PLAYSTATE: 1\r\nSPKR_SET_ON: 0\r\nPA1_AMP_GAIN: 0\r\n");
md_send("SPEAKER_DEFAULT", NULL, "VALUE: 6\r\nIS_HIGH_SPEAKER_GAIN: 0\r\nVOLUME_FACTOR: 1\r\n");
```

- **Output destination** (`SPEAKER_DESTINATION` / `SPEAKER_DEFAULT VALUE`): **6 = hands-free speaker,
  3 = handset earpiece** (headset via its own path). Switch output mid-call by re-sending `AUDIO_SETUP`
  with the new destination (no need to redo the stream).
- **Mute** the microphone with `MICROFONE_VOLUME: 0`.
- Volumes are small integers (≈ 0–15).

Tear down with `STREAM_CLOSE` (`SSRC: <ssrc>\r\nHOLD: 0\r\n`) and `RTP_CLOSE`.

### 7.2 Codecs

The DSP supports, in index order:

```
0 g711u   1 g711a   2 g726-16  3 g726-24  4 g726-32  5 g726-40
6 g729    7 g729a   8 g729b    9 g729ab   10 amr-wb   11 amr
12 linear-128  13 linear-256  14 g722  15 g711u-wb  16 g711a-wb  17 g7231  18 iLBC
```

The RTP-payload-type ↔ codec mapping for a stream is configured through `STREAM_SETUP` (the EPT's
per-stream ingress/egress codec tables). PCMU (G.711 µ-law, payload 0, 8 kHz) + RFC 2833
telephone-events (payload 101) is the simplest working configuration. The exact binary form of the
codec-table descriptor for wideband (e.g. G.722, which also needs a 16 kHz sample rate) is not publicly
documented.

### 7.3 DTMF to the far end

Send RFC 2833 telephone-events via `PLAY_DTMF` — start the event, then stop it after an audible
duration (e.g. ~180 ms):

```
md_send("PLAY_DTMF", NULL, "ENABLED: 1\r\nVALUE: 49\r\nTIME: <ms>\r\n");   /* '1' = ASCII 49 */
/* ... ~180 ms later ... */
md_send("PLAY_DTMF", NULL, "ENABLED: 0\r\nVALUE: 49\r\nTIME: <ms>\r\n");
```

## 8. Playing local audio (dial tone, ringback, ring melodies, recording, conference)

`media_daemon` only mixes **RTP streams** to the earpiece; it has no "play a tone" primitive for
applications. So locally generated audio is produced by **streaming your own RTP into the call's RX
path** on loopback, where the DSP mixes it like far-end media. Point a stream at a loopback port (via
`STREAM_SETUP` with `RTP-ADDRESS: Udp:127.0.0.1:<port>` and `AUDIO_SETUP` routing to the earpiece), then
send PCMU RTP packets to the stream's RX port at 20 ms intervals.

RTP packet = 12-byte header + 160 µ-law bytes (20 ms @ 8 kHz):

```c
/* build one 20ms PCMU packet; seq/ts/ssrc are big-endian on the wire */
unsigned char pkt[12 + 160];
pkt[0] = 0x80;            /* V=2, no padding/extension/CSRC */
pkt[1] = 0;              /* marker=0, payload type = 0 (PCMU) */
pkt[2] = seq >> 8;  pkt[3] = seq;                 /* sequence number */
pkt[4] = ts  >> 24; pkt[5] = ts >> 16; pkt[6] = ts >> 8; pkt[7] = ts;   /* timestamp */
pkt[8] = ssrc>> 24; pkt[9] = ssrc>>16; pkt[10]= ssrc>>8; pkt[11]= ssrc; /* SSRC */
for (int k = 0; k < 160; k++) pkt[12 + k] = linear_to_ulaw(sample[k]);  /* 8-bit G.711 µ-law */
sendto(sock, pkt, sizeof pkt, 0, (struct sockaddr*)&to, sizeof to);
/* then: seq++;  ts += 160;  sleep 20 ms (use clock_nanosleep for a stable cadence) */
```

Standard G.711 µ-law encoding (bit-exact):

```c
static unsigned char linear_to_ulaw(int s) {
    int sign = (s >> 8) & 0x80; if (sign) s = -s; if (s > 32635) s = 32635;
    s += 0x84; int exp = 7; for (int m = 0x4000; (s & m) == 0 && exp > 0; m >>= 1) exp--;
    int man = (s >> (exp + 3)) & 0x0f;
    return (unsigned char)~(sign | (exp << 4) | man);
}
```

The same mechanism is used for a **software call recorder** (set up a second stream to a loopback
receiver that writes incoming packets to a WAV file — the DSP mixes mic + far-end into it) and for
**3-way conference** mixing (a second stream bridging the two legs).

## 9. Boot process and runtime components

How the phone comes up, from power-on to the running telephony application. All of the paths and
process names below are from the stock software image.

### 9.1 Boot chain overview

```
U-Boot (NOR/NAND, mtd1)
   └─ loads + boots the kernel (mtd6)
Linux 2.6.27 (BCMRing)
   └─ busybox init (PID 1)  ──reads──▶  /etc/inittab
        └─ ::once:/etc/init.d/rcS           # the main init script
             ├─ hardware/model detect, MTD partition setup
             ├─ mount /protected (ro) and the root filesystem; software-update / rescue handling
             ├─ insmod the Broadcom telephony + peripheral drivers
             ├─ draw the boot splash
             ├─ start base daemons (syslog, connectivity, wpa_supplicant, sshd, rngd, dns-proxy)
             └─ supervise loop:  exec /snom/start.sh   (restarted if it exits)
                   └─ /snom/start.sh
                        ├─ UPX-decompress the app binaries in place
                        ├─ serial console + printk + FreeType env
                        └─ exec ./lid                 # the master phone supervisor
                             ├─ media_daemon          # audio/DSP engine (owns the EPT endpoint)
                             └─ the SIP + UI application  (and the on-board Bluetooth stack)
```

### 9.2 Flash layout (MTD / JFFS2)

The on-board flash is partitioned into MTD devices; the writable filesystems are **JFFS2**:

| Partition | Device        | Contents |
|-----------|---------------|----------|
| U-Boot    | `mtd1`        | bootloader |
| Rescue FS | `mtd4`        | minimal fallback root filesystem (recovery / failed-update boot) |
| Protected | `mtd5`        | **persistent** data that survives firmware updates (configuration, certificates, provisioning) — mounted read-only at `/protected` |
| Kernel    | `mtd6`        | Linux kernel image |
| Root FS   | `mtd7`        | main root filesystem (the application image) |

### 9.3 init and `/etc/init.d/rcS`

`init` is **busybox**. `/etc/inittab` is minimal:

```
::once:/etc/init.d/rcS
::shutdown:/bin/umount /
/dev/console::restart:/sbin/init
```

`rcS` is the large system-bringup script. In order, it:

1. **Detects the hardware/model** and selects the MTD partition set (see §9.2).
2. **Mounts** the protected partition (`mount -r -t jffs2 <mtd5> /protected`) and the root filesystem.
3. **Handles software update / rescue:** if an update image is staged (`/snom/update`) or the main root
   is invalid, it mounts the rescue filesystem (`mtd4`) to `/newroot`, moves `/proc`, `/sys`, `/tmp`
   across, and does `pivot_root . oldroot` + `chroot . kill -SIGQUIT 1` to switch root (then unmounts
   `oldroot`). The normal path proceeds into the main root filesystem.
4. **Loads kernel modules** (`insmod`) — the peripheral and Broadcom telephony stack, e.g.:
   `ehci-hcd`/`ohci-hcd` (USB), `bcmring_eth_sla` (Ethernet switch), `amxr` (audio mixer router),
   `halaudio` + `halaudio_apm` + `halaudio_pcm` (HAL audio), `pxcLdxSupport` + `pxcLdxHausware`
   (Broadcom PXC / "Hausware" DSP), and `ept_driver` + `ept_core` (the EPT telephony endpoint). USB
   handset variants also load `halaudio_usbcore`/`halaudio_usb`. (The display, keypad and side-screen
   drivers are built-in; see the kernel threads in §9.7.)
5. **Draws the boot splash** (`/snom/bmp/colored/splash_screen.png`) via an on-board display helper.
6. **Starts base daemons** via `/etc/init.d/S*` scripts and inline: system logging (`S01Syslog` →
   `rsyslogd`), `Connectivityd` (`S10Connectivityd`), `wpa_supplicant` (`S40wpa_supplicant`, Wi-Fi
   variants), `sshd`, `rngd`, and a `dns-proxy`.
7. Enters a **supervise loop** that runs `/snom/start.sh` and restarts it if it exits — this is the
   application-level watchdog.

### 9.4 `/snom/start.sh`

The application launcher. It:

1. **UPX-decompresses** the packed app binaries in place, e.g.
   `upx -lqq /snom/lid > /dev/null 2>&1 && upx -dqq /snom/lid` (same for `media_daemon` and the SIP/UI
   binary). This is why the on-flash application binaries are UPX-compressed and show no strings until
   unpacked.
2. Configures the **serial debug console** (`/dev/ttyS0`, 9600 8N1, raw) and silences kernel log output
   (`echo 0 > /proc/sys/kernel/printk`).
3. Exports font-rendering settings (`FREETYPE_PROPERTIES="truetype:interpreter-version=35"`).
4. `cd /snom; exec ./lid` — replaces itself with the master supervisor.

### 9.5 `lid` — the master phone supervisor

`lid` is the top-level phone process. It launches and supervises the telephony runtime: the
**`media_daemon`** (audio/DSP engine) and the **SIP + UI application**, and it also hosts the on-board
**Bluetooth** stack. Because `rcS` supervises `start.sh` (which `exec`s `lid`), if `lid` exits the whole
application layer is restarted.

### 9.6 `media_daemon` and the EPT endpoint

`media_daemon` is started (typically as `/snom/media_daemon --rec 0 --play 0 --log --no_headset_agc`).
On start it opens and drives the Broadcom **EPT** endpoint (`/dev/ept0`) + `/dev/halaudio`, which spawns
per-endpoint kernel threads (`EPTegrRTP_n`, `EPTegrRTCP_n`, `EPTing_n` — egress RTP/RTCP and ingress for
endpoint *n*), and it exposes the **LPCP** control socket on `127.0.0.1:1299` (see §7). All call and
local audio flows through this process.

### 9.7 Notable kernel threads

Visible in `ps` and relevant to the interfaces in this document:

- **`keypad: hook poll`** — polls the hook switch and feeds keypad input (the hook arrives as `EV_KEY`
  102, see §6).
- **`snom_fb2`** — the side-display (fb1) driver thread.
- **`EPTegrRTP_*`**, **`EPTegrRTCP_*`**, **`EPTing_*`**, **`vpm_power_queue`**, **`hausioctl`**,
  **`Cadence`**, **`frameProfiler`** — the Broadcom VPM/Hausware/EPT DSP telephony threads.
- **`jffs2_gcd_mtd7`** — JFFS2 garbage collection on the root filesystem.
- **`mtdblockd`**, **`bcmring_ddrvpmp`**, USB/Bluetooth helper threads.

### 9.8 Firmware update / rescue

Firmware updates are handled by `rcS`: a staged, **signature-verified** image (see §11) is unpacked and
applied. There are **two kinds** of update, distinguished by a marker file (`update/systemUpdate`)
inside the unpacked image:

- **Application-only ("partial") update** — *no* `systemUpdate` marker. `rcS` runs `doSnomUpdate`, which
  replaces only the **application files** (the `/snom` directory) and leaves the kernel and root
  filesystem untouched. No reboot into rescue is needed.
- **System ("full") update** — the `systemUpdate` marker is present (the image carries a kernel + root
  filesystem). `rcS` runs either:
  - `doSystemUpdate` (regular), or
  - `doAdvancedSystemUpdate`, chosen when both the image and the device support "advanced update v2"
    (`/protected/advancedUpdateVersion` = 2, an `update/updateHelper.sh` in the image, and a valid
    rescue filesystem). The advanced path uses the **rescue filesystem** (`mtd4`, via the `pivot_root`
    in §9.3 step 3) and **dual root-filesystem slots** — the active slot is selected by the U-Boot
    variable `rootfsvol` (`rootfs1` / `rootfs2`) — writing the new system to the inactive slot and
    switching to it, for a safer A/B-style update.

The persistent `/protected` partition (`mtd5`) is preserved across all updates. Images are staged (e.g.
under `/snom/update`) and picked up on the next boot.

## 10. Enabling a shell / SSH on the stock image

The stock image ships `/usr/sbin/sshd` (OpenSSH); its `sshd_config` permits root login with an empty
password (`PermitRootLogin yes`, `PermitEmptyPasswords yes`, `PasswordAuthentication yes`), and
`/etc/shadow` has an **empty root password** — so once `sshd` is running you can log in as `root` with no
password. `rcS` does **not** start `sshd` in normal operation; it only does so in **development mode**.

### 10.1 Development mode (`development=yes`)

`rcS`'s `getStartMode()` checks the U-Boot environment variable `development`. When
`development=yes`, on **every boot** it:

1. shows the splash and a **~15 s prompt** `Ok -> Phone, X -> Shell` (`getKey 15`, auto-continues to the
   phone application on timeout);
2. if you choose **Phone** (or let it time out): generates an SSH host key if missing
   (`ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ""`), starts **`/usr/sbin/sshd`**, then starts the
   phone application — i.e. SSH is reachable during normal phone operation;
3. if you choose **Shell** (press X): drops to a **root shell** on the serial console (`/dev/ttyS0`,
   9600 8N1, see §9.4) instead of the phone app.

Set it from a root shell (persists across reboots and firmware updates, as it lives in the bootloader
environment in flash):

```sh
fw_setenv development yes     # enable; prompt + sshd on every boot
# fw_setenv development no    # disable
```

Related U-Boot env / cmdline switches handled by `getStartMode()`: **`forceshell=yes`** (env) or
**`forceshell`** on the kernel command line force Shell mode; `updatetesturl=<url>` forces an update
test; `killproductionmode=yes` clears production mode.

### 10.2 Getting initial root access (bootstrap)

Development mode (§10.1) needs a shell to set `fw_setenv`, and its boot prompt only appears once
`development=yes` is set — so for the *first* root login you need one of the following. In all cases the
payoff is the same: an `sshd` running, into which you log in as `root` with **no password**.

**Method 1 — OpenVPN `up`-script (no physical access).** The standard way in without opening the phone.
It relies on three stock facts:

1. the phone ships a built-in **OpenVPN** client (`/usr/sbin/openvpn`), and OpenVPN run with
   `script-security 2` executes the config's `up` script;
2. there is a stock helper **`/usr/local/bin/startsshd`** whose whole job is to bring up networking
   (DHCP on `eth0`), generate an SSH host key if missing, and run **`/usr/sbin/sshd`** (it also pins a
   static IP and disables the watchdog);
3. `root` has an empty password and `sshd_config` allows it (§10 intro).

So: stand up an OpenVPN server you control, and install on the phone a client profile whose config
contains `script-security 2` and `up /usr/local/bin/startsshd`. When the phone brings the tunnel up it
runs `startsshd`, `sshd` starts, and you SSH in as `root`. The VPN profile is a small tar of
`ca.crt` + `client.crt` + `client.key` + the OpenVPN config file (the phone expects the certs at
`/openvpn/*`); it is installed through the phone's **web admin** (VPN settings) or via **auto-
provisioning**. A minimal lab is just an OpenVPN server plus a web server to host the profile. This is
the well-documented snom root-access technique (see the coredump.ch / encrypted.at write-ups).

**Method 2 — serial console / U-Boot (physical access).** Attach a 3.3 V TTL-serial adapter to the
debug UART (console `/dev/ttyS0`, **9600 8N1**, see §9.4). Interrupt the boot to reach the U-Boot prompt
and set **`development=yes`** — this is the recommended flag: the phone still boots the phone
application (after the 15 s prompt) *and* starts `sshd`, and it persists across reboots/updates:

```
setenv development yes
saveenv
boot
```

> **Avoid `forceshell`.** It also yields a root shell, but it boots the phone **into a bare shell
> instead of the phone application** — so the phone does not work as a phone while it is set. Use it
> only for one-off recovery and clear it afterwards. Prefer `development=yes`, which gives SSH *and*
> keeps the phone usable.

With Method 1, after your first login run `fw_setenv development yes` to persist SSH the same way
(§10.1).

## 11. Firmware image format (decrypt / encrypt / sign)

A firmware `.bin` is, from the start of the file:

| Offset        | Size        | Contents |
|---------------|-------------|----------|
| `0 .. 128`    | 128 bytes   | **RSA signature** (SHA-1) over everything after it (keyblock + payload) |
| `128 .. 256`  | 128 bytes   | **key block** — the AES password, RSA-signed (recoverable with the model public key) |
| `256 .. EOF`  | remainder   | **OpenSSL `Salted__` AES-128-CBC** blob — the encrypted `tar.gz` of the image |

RSA keys are **1024-bit** (hence the 128-byte signature and key block). The AES layer uses OpenSSL's
**legacy MD5 key-derivation** (OpenSSL 3 requires an explicit `-md md5`). Each phone model has a public
key on the device at **`/firmware1.pub`** (also embedded in the rootfs); the phone verifies uploads
against it in `rcS` (`verifySoftwareUpdateFile()` / the update handler call `openssl ... --key
/firmware1.pub`). It verifies the signature as: `dd bs=128 count=1` (the signature) vs
`dd bs=128 skip=1 | openssl dgst -sha1 -verify /firmware1.pub`, i.e. over everything after the first
128 bytes.

Both **full (system)** and **application-only** updates use this same container format; which kind an
image is is decided from a marker inside it after decryption (see §9.8).

### 11.1 Decrypt / extract

```sh
# split the image
dd if=fw.bin bs=1 skip=128 count=128  of=keyblock.bin    # the RSA-signed AES password
dd if=fw.bin bs=1 skip=256            of=payload.enc     # the Salted__ AES blob

# recover the AES password from the key block using the model public key
openssl pkeyutl -verifyrecover -pubin -inkey firmware1.pub -in keyblock.bin -out aes.pass

# decrypt the payload (legacy MD5 KDF) -> a tar.gz
openssl enc -aes-128-cbc -d -md md5 -in payload.enc -out fw.tar.gz -kfile aes.pass
tar xzf fw.tar.gz
```

### 11.2 Re-pack / sign

The signature chain is verified with the model's public key, so a repack re-signed with the **original**
private key is not possible (that key is not public). To build images the phone will accept you sign
with **your own** 1024-bit RSA key pair and make the phone trust it (§11.3):

```sh
# encrypt the payload with a chosen AES password (legacy MD5 KDF, salted)
openssl enc -aes-128-cbc -md md5 -salt -in fw.tar.gz -out payload.enc -kfile aes.pass

# key block = the AES password signed with YOUR private key (so verifyrecover with YOUR pubkey yields it)
openssl pkeyutl -sign -inkey yourkey.pem -in aes.pass -out keyblock.bin   # 128 bytes for a 1024-bit key

# overall signature = SHA-1 RSA signature over (keyblock || payload), with YOUR private key
cat keyblock.bin payload.enc > rest.bin
openssl dgst -sha1 -sign yourkey.pem -out sig.bin rest.bin                # 128 bytes

# assemble: signature || keyblock || payload
cat sig.bin keyblock.bin payload.enc > fw-custom.bin
```

(The exact padding/KDF must match what the phone's verifier expects; the layout above round-trips
against the on-device `verifySoftwareUpdateFile()`.)

### 11.3 Running self-signed firmware

To make the phone accept images signed with your own key, replace the on-device public key and disable
the integrity check that would otherwise notice:

1. The root filesystem is **JFFS2 on an MTD block, mounted read-only** (only `/snom` and `/etc` are
   writable overlays). Remount it writable and replace the key with your 1024-bit public key:
   ```sh
   mount -o remount,rw /
   cp your_pub.pem /firmware1.pub
   ```
2. `rcS` also runs an integrity check (`md5sum -c /etc/checksum.md5`, which lists `/firmware1.pub`) and
   drops to a shell on mismatch. Disable it via the U-Boot environment:
   ```sh
   fw_setenv skipmd5 yes
   ```
3. Build images with the matching private key (§11.2) and install them through the phone's
   **Software Update** (firmware URL) — they now pass verification against your key.

> This replaces the device's firmware trust anchor with your own and only applies to a phone you own;
> it is the standard way to run custom/open-source firmware on this hardware.

## 12. Interfaces that are not available

- **Wideband codecs (e.g. G.722):** require the `STREAM_SETUP` codec-table descriptor plus the DSP
  sample rate (16 kHz); the binary form of that descriptor is not publicly documented.
- **Real-time audio levels / VU metering:** the EPT endpoint is run without event/packet callbacks, so
  LPCP exposes only packet counters (`RX-TOTAL-PACKETS:` from `RTP_STATS`), not sample levels.
- **Direct EPT / codec control** (the Broadcom `pxcon` console and EPT event callbacks): not reachable
  while `media_daemon` holds `/dev/ept0` open — which is also why hook/DTMF/tone *events* are not
  available over LPCP (read the hook from the keypad instead, §6).
