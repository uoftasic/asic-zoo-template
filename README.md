# ASIC Zoo project template

ASIC Zoo is a UofT ASIC Team project. Each member designs one small analog circuit, takes it from schematic to layout, and gets it fabricated on a shared chip through [Tiny Tapeout](https://tinytapeout.com), a service that puts many small designs on one chip. We're using SkyWater's open-source 130 nm process, SKY130. When the chips come back, you measure your own circuit on the bench and compare the results with your simulations.

You pick the circuit. It can come from a course, a paper, or your own idea, as long as it fits the design rules below. If you don't have an idea yet, see the [project ideas](#project-ideas).

This repository is a template, based on Tiny Tapeout's [ttsky-analog-template](https://github.com/TinyTapeout/ttsky-analog-template). Each circuit gets its own repository made from it (step 1 below), so that Tiny Tapeout's GitHub Actions build and check your layout on every push.

## How the chip works

You design and check your circuit in your own repository. At the end, the integration group collects all the designs into one shared Tiny Tapeout project:

- the shared project holds up to **12 designs**;
- digital select pins choose which design is active;
- an analog multiplexer connects the active design's three analog ports to the chip's analog pins, through output buffers;
- a bypass channel connects a pin straight through the multiplexer, so we can measure the multiplexer itself and subtract its effect from every other measurement;
- a shared bias network supplies a reference current to every design.

Because all 12 designs share one project, the space each one gets is limited. The next section explains where the numbers come from.

## Design rules

### Area

A Tiny Tapeout analog project can be at most 2x2 tiles: 334.88 µm × 225.76 µm, according to Tiny Tapeout's [layout templates](https://github.com/TinyTapeout/tt-support-tools/tree/main/tech/sky130A/def/analog). Power stripes must run from 5 µm above the bottom edge to 5 µm below the top, which leaves about 215 µm of usable height, or about 72,250 µm².

30% of that goes to the multiplexer, decoder, buffers, bias network and routing. The remaining 70% is split 12 ways:

72,250 µm² × 0.7 ÷ 12 ≈ **4,200 µm² per design**

In a 4 × 3 grid, that works out to an outline of **80 µm × 50 µm (4,000 µm²)**. Your layout must fit inside it. The integration group routes over the top of the designs on metal 4, so there are no routing gaps between outlines.

For scale, a complete ADC on a recent Tiny Tapeout SKY130 chip, with two op-amps, a comparator and a bias generator, took about 1,253 µm², roughly a third of one outline.

### Ports

Every design has the same ports. You don't have to use them all; tie unused inputs to `VGND`.

| Port | Type | What it's for |
| --- | --- | --- |
| `VDPWR` | supply | 1.8 V |
| `VGND` | ground |     |
| `en` | digital input, 1.8 V | High when your design is selected. Use it to power your circuit down when it isn't. |
| `ain` | analog | Main input, driven from the bench through the multiplexer. |
| `aout` | analog | Main output. It goes to a buffer, not straight to a pin, so it only has to drive the buffer's input. |
| `aux` | analog | A third terminal, used however your design needs: the − input of an op-amp or comparator, a reference voltage, a second output, or an external bias that overrides the shared one. Tie it to `VGND` if you don't need it. |
| `ibias` | analog input | Reference current from the shared bias network. It's an internal connection, not one of the chip's pins. |
| `ctrl[5:0]` | digital inputs, 1.8 V | Control bits shared by all designs, such as DAC codes or trim bits. |
| `dout[1:0]` | digital outputs, 1.8 V | Digital results, such as a comparator decision or a divided-down clock. |

### Electrical limits

- **Supply:** 1.8 V only. This template doesn't use Tiny Tapeout's optional 3.3 V supply (`VAPWR`).
- **Signal range:** 0 V to 1.8 V on every port.
- **Current:** only one design is powered at a time, so Tiny Tapeout's power network sets the limit: about 0.1 V of droop at 20 mA, or about 5 Ω. Keep your circuit under **1 mA** while selected (about 5 mV of droop) and under **1 µA** while `en` is low.
- **Pins:** Tiny Tapeout specifies under 500 Ω and under 5 pF between each analog pad and the project, and at most 4 mA through any analog pin. The multiplexer adds its own resistance and capacitance on top of that; its numbers will go here once it's designed. Plan your output impedance and bandwidth measurements around that extra load.

### Layout

- Stay inside the 80 µm × 50 µm outline.
- Use li1 and metal 1 to metal 3 only. Tiny Tapeout's pins and power stripes are on metal 4, and the integration group needs metal 4 to route over your design. Tiny Tapeout doesn't allow metal 5 at all.
- Put your ports on the outline's edge, using the names in the table above.
- On matched pairs and current mirrors, use common-centroid placement (interleave the matched transistors around a shared centre) and dummy devices at the ends. A project lead checks this when reviewing your layout.
- Keep the Magic layout hierarchical, not flattened. Tiny Tapeout's instructions say flattening causes DRC problems.

## Getting started

1. **Make your repository.** Click **Use this template** at the top of this page, then **Create a new repository**. Name it after your circuit, for example `zoo-cascode-mirror`.
2. **Set up the tools.** Use the team's [workspace](https://github.com/uoftasic/workspace), the same Docker environment every UofT ASIC course uses. It's built on the IIC-OSIC-TOOLS image, with xschem, ngspice, Magic, KLayout and a pinned version of the SKY130 PDK already configured. Course [IC101](https://uoftasic.com/ic101/) walks you through setting it up.
3. **Write a spec sheet.** Copy [docs/spec.md](docs/spec.md) and fill it in: what the circuit does, where it comes from, what you'll measure and the numbers you're aiming for. A project lead reviews it with you before you start the schematic.
4. **Design and simulate.** Draw the schematic in xschem and simulate it in ngspice. It has to meet your spec at all process corners (tt, ff, ss, fs, sf), over supply and temperature, and in a Monte Carlo mismatch run.
5. **Lay it out.** Draw your design in Magic inside the 80 µm × 50 µm outline, then place it in the 1x2 test project described below. Run three checks: DRC (the layout follows the foundry's geometry rules), LVS (the layout matches your schematic), and a simulation with the parasitic resistance and capacitance extracted from the layout (PEX). Your circuit is done when the post-PEX simulation still meets your spec. If you haven't done layout before, take [AD104](https://uoftasic.com/ad104/) and come to the [layout workshop](#full-custom-layout-workshop) first.
6. **Fill in the Tiny Tapeout files.** Edit `info.yaml`, `src/project.v` and `docs/info.md` as described in [Tiny Tapeout files](#tiny-tapeout-files). Export your GDS and LEF so the GitHub Actions checks run on your design.
7. **Hand it over.** Post a link to your repository in the UofT ASIC Discord. The integration group places your design on the shared chip.

## Testing your design on its own

Your repository is a complete Tiny Tapeout analog project at the smallest analog size, 1x2 tiles (161.0 µm × 225.76 µm). Your 80 µm × 50 µm design sits inside it, and the rest of the space is left empty. Tiny Tapeout names the project's pins `ua[]` for analog, `ui_in[]` for digital inputs, `uo_out[]` for digital outputs, and `uio_*` for bidirectional digital pins. Wire your design to them like this, so that everyone's test projects use the same pinout:

| Design port | Project pin |
| --- | --- |
| `ain` | `ua[0]` |
| `aout` | `ua[1]` |
| `aux` | `ua[2]` |
| `ibias` | `ua[3]` (drive it from an external current source when testing) |
| `en` | `ui_in[0]` |
| `ctrl[5:0]` | `ui_in[6:1]` |
| `dout[1:0]` | `uo_out[1:0]` |

Tie every unused `uo_out`, `uio_out` and `uio_oe` pin to `VGND`; Tiny Tapeout doesn't allow floating digital outputs. Its [analog specs page](https://tinytapeout.com/specs/analog/) explains how to do that in Magic.

This test project is never submitted to a shuttle. Only the shared project is. The test project exists so that Tiny Tapeout's `gds` and `precheck` actions check your layout early.

## Tiny Tapeout files

These follow Tiny Tapeout's [analog specs page](https://tinytapeout.com/specs/analog/), which has the full instructions.

- **`info.yaml`:** set `title`, `author`, `discord` and `description`. Keep `tiles: "1x2"` and set `analog_pins: 4`. Set `top_module` to `tt_um_uoftasic_zoo_<yourcircuit>`; the name must start with `tt_um_` and be unique on the shuttle. Fill in the `pinout` section using the table above.
- **`src/project.v`:** rename `tt_um_example` to your `top_module` name. This file is only a black box for Tiny Tapeout's integration; it contains no logic.
- **`docs/info.md`:** explain how your circuit works and how to test it.
- **`gds/` and `lef/`:** export your layout from Magic with `gds write ../gds/<top_module>.gds` and `lef write ../lef/<top_module>.lef -pinonly`. The file names must match `top_module`.

Until you add the GDS and LEF files, the `gds` check fails on every push. That's expected for a new repository.

## Project ideas

Bring your own circuit if you have one. If you don't, pick one of these. There are more ideas than the 12 places on the chip. They're grouped by difficulty, so a first-year with one circuits course can take a current mirror and a fourth-year can take the bandgap.

### Tier 1: first circuit

| Circuit | What you'd measure |
| --- | --- |
| NMOS / PMOS test structures | I-V curves, threshold voltage, transconductance |
| Simple current mirror | Output resistance, current matching |
| Cascode current mirror | Output resistance, compliance voltage |
| Inverter-chain delay line | Propagation delay against supply |
| Ring oscillator | Frequency against supply and temperature |

### Tier 2: some analog experience

| Circuit | What you'd measure |
| --- | --- |
| Resistor and MIM capacitor ladder | Absolute value, matching, temperature coefficient |
| Source-follower buffer | Gain, output impedance, level shift |
| 5-transistor OTA | Gain, gain-bandwidth, phase margin, input offset |
| Two-stage Miller-compensated op-amp | Gain, gain-bandwidth, phase margin, slew rate, PSRR |
| Comparator with hysteresis | Offset, hysteresis width, propagation delay |
| Relaxation oscillator | Frequency, supply sensitivity, jitter |

### Tier 3: advanced

| Circuit | What you'd measure |
| --- | --- |
| Bandgap voltage reference | Output voltage, temperature coefficient, PSRR |
| Low-dropout regulator | Dropout, line and load regulation, stability |
| PTAT temperature sensor | Sensitivity, linearity, absolute accuracy |
| Charge pump | Output voltage, ripple, efficiency |
| 6-bit R-2R DAC | DNL, INL, settling time (uses `ctrl[5:0]` for the code) |

Two people can pair on one circuit; that's a good idea for a first project.

## After tapeout

When the chips arrive, you measure your own circuit in a fully equipped lab at UofT, with bench supplies, signal generators and oscilloscopes. For quick checks outside the lab, an Analog Discovery (a USB oscilloscope and signal generator) also works; Tiny Tapeout has a [guide for that setup](https://tinytapeout.com/guides/analog-discovery/). Your schematic, layout, simulated and measured results go into the Silicon Bestiary, a public catalogue of every circuit on the chip with your name on your page. If your measurements don't match your simulations, write down the difference and why you think it happened; that's useful to the next person who designs one.

## Resources

### Learn analog design with UofT ASIC

The team's [education site](https://edu.uoftasic.com) has self-paced courses that start from no background and use the same tools as this project. Start with IC101, then work through the analog track:

| Course | What it covers |
| --- | --- |
| [IC101: Onboarding onto Tools](https://uoftasic.com/ic101/) | Setting up the shared Docker workspace and SKY130 tools. Every other course assumes it. |
| [AD101: Signals](https://uoftasic.com/ad101/) | Waveforms, spectra and Bode plots |
| [AD102: Linear Circuits & Fabrication](https://uoftasic.com/ad102/) | Linear circuit analysis, and how passives are fabricated on-chip |
| [AD103: Nonlinear Circuits](https://uoftasic.com/ad103/) | The diode and MOSFET in xschem: regions of operation and key device parameters |
| [AD104: Layout](https://uoftasic.com/ad104/) | Full-custom layout: stick diagrams, DRC/LVS-clean cells, PEX, and manual place and route |

AD103 and AD104 cover most of what a tier 1 circuit needs. Two more analog courses, on op-amps and filters (AD201) and on ADCs and DACs (AD202), are planned.

### Full-Custom Layout Workshop

A hands-on day of full-custom IC layout led by Nanik Adnani of StarIC, on Saturday, November 21, 2026, 10 AM to 4 PM, at the Bahen Centre. It's aimed at members working on the team's analog projects, including this one. Register at [luma.com/x8vm0yuu](https://luma.com/x8vm0yuu).

### Other resources

- [Tiny Tapeout analog specs](https://tinytapeout.com/specs/analog/): pins, tile sizes, power and submission rules
- [IIC-OSIC-TOOLS](https://github.com/iic-jku/IIC-OSIC-TOOLS): the tool image the workspace is built on
- [Zero to ASIC analog course](https://zerotoasiccourse.com/analog) by Matt Venn
- [Magic cheatsheet](https://github.com/iic-jku/osic-multitool/blob/main/magic-cheatsheet/magic_cheatsheet.pdf) and [Circuit designer's etiquette](https://github.com/iic-jku/Circuit-Designers-Etiquette) by Harald Pretl
- [Xschem tutorial](https://www.youtube.com/watch?v=q3ZcpSkVVuc) by Stefan Schippers (video)
- [Analog design tutorials](https://www.youtube.com/playlist?list=PLybHXZ9FyEhZfwQTKrLhm6ZZm4IDfGGla) by Carsten Wulff (video)
- Razavi, *Design of Analog CMOS Integrated Circuits*: the standard reference for tier 1 and 2 circuits
- Baker, *CMOS: Circuit Design, Layout, and Simulation*: layout practice and matching

Questions go in the UofT ASIC Discord.
