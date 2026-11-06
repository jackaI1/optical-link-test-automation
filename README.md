# Optical test bench automation

I built a simulated optical test bench because checking every wavelength and power setting by hand makes it easy to miss a weak link or record a reading incorrectly. It runs the same checks repeatedly, collects the readings, and produces a report I can inspect or share. The dashboard lets me run a sample plan and ask for help investigating a result.

Light gets weaker as it travels through fibre and passes connectors or splices. This reduction is called **loss**; a loss budget is the maximum reduction a link can tolerate. The optical signal-to-noise ratio (**OSNR**) compares useful light with background noise. A high enough OSNR helps a receiver distinguish the signal. This project simulates both so I can practise checking a link without laboratory hardware.

## Try it

Install Python 3.10 or newer, then from this folder:

```sh
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m optical_station --output report.html
```

Open `report.html` to see the results. To use the dashboard instead, run `python -m optical_station.web` and open http://127.0.0.1:5000. Click **Run test plan**, then download a report or choose **Triage** on a row. The dashboard loads React from a public content delivery network, so it needs an internet connection; the command-line runner works offline after installation.

Change `plans/default.yaml` to select wavelengths, launch powers, sample count and acceptance limits. Version 1 plans run every combination. Results near a limit are marked marginal; results beyond a limit fail. Missing or unreliable measurements fail rather than silently passing.

## How it works

A simulated laser sets wavelength and launch power. The link subtracts fibre attenuation per kilometre, connector loss and splice loss, and adds measurement noise. A power meter reads the received signal, and a spectrum analyser estimates OSNR against a simulated noise floor. The defaults are illustrative, not a calibrated physical model: wavelength dependence, nonlinear effects and real spectrum measurements are not modelled.

The instrument drivers send SCPI-style commands (text commands used by many laboratory instruments) through `write` and `query` methods compatible with PyVISA resources. For real equipment, open resources with PyVISA and adapt command names, units, timing and safety limits to the instrument manuals. The simulator combines the three instruments in one resource; real devices usually need separate resources. It does not enable real laser output or claim hardware validation.

The sequencer is a small open-source alternative to a graphical test sequence for this station: it runs a versioned YAML plan, gathers repeated measurements, removes missing values and median-based outliers with pandas, and applies the stated limits. An HTML report contains the results. A scikit-learn random forest also predicts pass, marginal or fail from the two margins. It is trained on synthetic examples labelled by the same limits and evaluated on a separate, seeded 25% holdout. The deterministic limits remain the authoritative result, not the model; the evaluation does not establish performance on real instruments.

The Flask service runs the same sequence through a Protobuf fixture request/response boundary. Protobuf is a compact binary message format; `fixture.proto` defines the messages and `/api/fixture` accepts serialized requests. The fixture is simulated in-process, so this is an RPC-style boundary rather than a remote networked fixture. The React and TypeScript dashboard displays results and offers triage. With `TRIAGE_API_URL` and `TRIAGE_API_KEY` set, triage sends the selected result to an OpenAI-compatible chat endpoint; `TRIAGE_MODEL` selects the model. Without them, or if the endpoint fails, it gives a local checklist. Triage only suggests checks and never changes limits or reruns a test. Do not send sensitive measurements to an external service without permission.

## Development

Run `python -m pytest -q`. After changing `web/dashboard.tsx`, run `npm ci && npm run build`; the small compiled dashboard is committed. GitHub Actions installs the Python dependencies, runs the tests and exercises the command-line report on each push and pull request. The tests cover the loss model, sweep limits, cleaning, fixture messages, injected timeouts, dropouts and outliers, reporting and API behavior.
