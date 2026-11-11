# Spec sheet: <circuit name>

Fill this in before you start the schematic, then go over it with a project lead. Keep it to about one page.

## About the circuit

- **Designer(s):**
- **Circuit:** what it is, in one or two sentences.
- **Source:** the course, paper, textbook section or idea it comes from. Add a link or citation.
- **Area:** your estimate of the layout area, against the 4,000 µm² outline.

## Ports used

| Port | Used? | What it does in your circuit |
| --- | --- | --- |
| `en` |     |     |
| `ain` |     |     |
| `aout` |     |     |
| `aux` |     |     |
| `ibias` |     |     |
| `ctrl[5:0]` |     |     |
| `dout[1:0]` |     |     |

## Targets

List every number you'll measure on the chip. Each needs a target and the range you'd accept, across corners, supply and temperature.

| Parameter | Target | Acceptable range | Conditions |
| --- | --- | --- | --- |
|     |     |     |     |

## Power

- Current from `VDPWR` while selected:
- Current while `en` is low:

## How to measure it

Which instruments from the UofT lab you'll need, what you'll drive into which port, and what you'll look at. Say how the extra resistance and capacitance between your design and the pins could affect the measurement. The README's "Electrical limits" section lists them.

## Risks

What could make it fail on silicon: mismatch, corners, parasitics, or something specific to this circuit.
