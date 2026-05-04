# Lmbic — Interactive Projection System

A passion project that explores spatial AR through projection mapping: an
NVIDIA Jetson, two cameras, and a DLP projector packaged into a lamp-style
form factor that points at a desk or a wall and projects context-aware
overlays onto whatever sits below it.

This is a write-up of the implementation journey. It is intentionally
high-level — the project is still being pursued by others as a startup
without me — so I describe outcomes, challenges, and the shape of the
solutions, not the specifics that may carry commercial value.

The story is split into chapters. This is chapter one.

---

## 1. Hardware bring-up

When I joined the project we had freshly received the hardware and a
not-yet-working pile of components on a desk. The goal of this first
milestone was simple to state and surprisingly hard to reach: connect
everything together, drive it from a single computer, and get the projector
to display a frame that originated on that computer.

### What was on the desk

**Display path.** An [eViewTek P2](https://www.eviewtek.com/) optical engine
containing a Texas Instruments digital micromirror device (DMD), driven by a
TI DLPC controller mounted on a driver board. The same driver board carries
an [ITE IT6801FN](https://www.ite.com.tw/en/product/cate1/IT6801) HDMI
receiver responsible for accepting the video signal that the DLPC ultimately
projects.

**Capture path.** Two
[Arducam IMX477](https://www.arducam.com/arducam-12mp-imx477-motorized-focus-high-quality-camera-for-jetson.html)
12-megapixel cameras with motorised focus, intended to give the system its
view of the world below the lamp.

**Compute.** An NVIDIA Jetson Nano Developer Kit, talking to the projector's
driver board over I²C for control and HDMI for the video signal, and to the
cameras over MIPI CSI-2.

![Hardware on the desk](images/01-components-on-desk.jpg)

### The MCU died

The driver board ships with a small microcontroller whose only job is to run
the DLPC's initialisation sequence at boot. Within days of receiving the
hardware that microcontroller failed in a way we could not recover from. The
projector was inert: video signal arriving on HDMI had nowhere to go, and we
had no path to reach the DLPC to bring it up another way.

The manufacturer was not responsive on the timescales we needed. We had no
full register-level documentation for the driver board as a system. Buying a
second unit and waiting for it to arrive was an option but a slow one, and
it would not solve the underlying problem the next time something blew up.
We needed a path that did not depend on the original MCU at all.

### Pivoting onto the TI reference design

eViewTek manufactures the optical engine, but the rest of the driver board
is built on a publicly documented Texas Instruments reference design — the
same one TI publishes for evaluation kits using their DLP chips. That
reference design comes with an open initialisation routine: how to wake the
DLPC up, how to configure the HDMI receiver feeding it, and the order in
which all of that has to happen.

The catch was that the reference code targets the original MCU. Our compute
sat on the other side of the I²C lines, on the Jetson, running Linux. We
re-targeted the boot sequence:

- Replaced the bare-metal MCU communication layer with Linux-side I²C
  transactions issued from the Jetson, using the kernel's standard interface
  to the bus
- Kept the high-level initialisation logic — register sets, sequence order,
  timing — from the reference
- Resolved the differences the reference did not cover: bus discovery,
  device addressing on the specific layout we had, and the ITE part's own
  configuration (for which we never found a programming guide and inferred
  enough behaviour from the reference code to make it work)

After enough iteration, an HDMI signal sent from the Jetson found its way
through the ITE receiver, into the DLPC, and out of the optical engine as a
projected image. First light.

![First image projected 1](images/02-first-light-1.jpg)
![First image projected 2](images/02-first-light-2.jpg)

### Cameras: the same shape of problem, smaller scale

The cameras went through their own less dramatic but similarly trial-and-
error pass. The IMX477 sensor is well-supported in the Linux ecosystem but
not entirely plug-and-play on the Jetson Nano: it needs the
[RidgeRun V4L2 driver](https://developer.ridgerun.com/wiki/index.php?title=Raspberry_Pi_HQ_camera_IMX477_Linux_driver_for_Jetson),
the right capture pipeline (NVIDIA's Argus API and the
hardware-accelerated GStreamer plugins), and a careful pass over focus —
manual coarse adjustment first, then software-driven fine adjustment, since
the motorised range is too narrow to reach focus from cold on its own.

![Camera live preview](images/03-camera-preview.jpg)

### Outcome

We ended this chapter with a closed loop: cameras feeding pixels into the
Jetson; the Jetson driving the DLPC over I²C and pushing video into it over
HDMI; the projector drawing whatever the Jetson asked it to. None of it was
yet doing anything intelligent — but the substrate was finally there, and
everything that follows is software.

---

_Next: turning the projector and the cameras into a closed-loop system that
can find itself in space. Calibration, homographies, and getting projected
content to land where you actually intended it to._
