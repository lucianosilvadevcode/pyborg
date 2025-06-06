
# pyBOrg: Python Brain Organoid Simulator

**pyBOrg** is a Python module designed for simulating brain organoids, leveraging the power and flexibility of the Brian2 spiking neural network simulator. It provides a structured framework to model neurons and synapses within organoids, simulate their development and activity, and interact with them using simulated Microelectrode Arrays (MEAs) and calcium imaging techniques.

## Objective

The primary goal of `pyBOrg` is to provide researchers with an accessible, modular, and extensible in-silico platform to:
*   Model the formation and maturation of neural networks in brain organoids.
*   Simulate electrophysiological experiments, including MEA-based stimulation and recording.
*   Simulate calcium imaging experiments to observe network activity with high spatial resolution.
*   Investigate the effects of various stimuli, genetic modifications, or disease states on organoid network dynamics.
*   Facilitate the exploration of structural and synaptic plasticity mechanisms.

## Key Features

*   **Brian2-Powered Core:** Utilizes Brian2 for efficient and accurate simulation of spiking neuron and synapse models.
*   **Organoid Construction:** Flexible tools to create 3D organoid structures with defined neuronal populations and spatial arrangements.
*   **MEA Integration:**
    *   Model MEA geometries.
    *   Simulate targeted electrical stimulation of organoids via MEA electrodes.
    *   Record spike activity and Local Field Potential (LFP) proxies from MEA electrodes.
*   **Calcium Imaging Simulation:**
    *   Neuron models incorporating intracellular calcium dynamics.
    *   Simulation of fluorescent calcium indicator signals (e.g., GCaMP's ΔF/F).
*   **Network Plasticity:**
    *   Implement common synaptic plasticity rules (e.g., STDP).
    *   Simulate structural plasticity for activity-dependent network formation and pruning.
*   **Electrophysiology Suite:**
    *   Generate complex stimulus patterns.
    *   Comprehensive data recording capabilities.
    *   Persistent data storage using SQLite for simulation results.
*   **Analysis & Visualization:**
    *   Tools for spike train analysis (firing rates, synchrony, bursting).
    *   Calcium trace analysis (ΔF/F, event detection).
    *   Functional network inference from spike or calcium activity.
    *   Plotting utilities for raster plots, LFP, calcium traces, and connectivity graphs.
*   **Modularity and Extensibility:** Designed to easily add new neuron models, synapse types, plasticity rules, or analysis techniques.
*   **Jupyter Notebook Examples:** A comprehensive suite of examples from basic setup to advanced simulations.

## Quick Start Examples

### 1. Stimulating the Organoid with an MEA

This snippet shows how to set up a simple organoid, an MEA, and stimulate a specific region.

```python
import brian2 as b2
from pyborg.organoid import Organoid, spatial
from pyborg.core import neuron_models
from pyborg.mea import MEA
from pyborg.simulation import Simulator
from pyborg.electrophysiology import stimulus_generator

# Ensure reproducible results
b2.seed(42)

# 1. Create an Organoid
lif_model = neuron_models.LIFNeuron()
my_organoid = Organoid(name="SimpleOrganoid")
positions = spatial.random_positions_in_sphere(N=100, radius=200*b2.um)
my_organoid.add_neurons(
    num_neurons=100,
    model_template=lif_model,
    positions=positions,
    name="lif_neurons"
)
# (Add synapses as needed...)

# 2. Create an MEA
mea_layout = mea.generate_grid_layout(rows=8, cols=8, spacing=50*b2.um)
my_mea = MEA(electrode_positions=mea_layout)

# 3. Setup the Simulator
sim = Simulator(organoid=my_organoid, mea=my_mea)
sim.setup_simulation_entry(
    organoid_config_details={"neuron_type": "LIF"},
    mea_config_details={"layout": "8x8_grid"}
) # For SQLite logging

# 4. Define a stimulus
pulse_stim = stimulus_generator.create_pulse_train(
    amplitude=0.8*b2.nA,
    frequency=10*b2.Hz,
    pulse_width=2*b2.ms,
    duration=100*b2.ms,
    dt=sim.brian_dt # Use simulator's dt
)

# 5. Add stimulus to an MEA electrode targeting nearby neurons
# (Assuming electrode 0 is defined and has targetable neurons)
sim.add_stimulus(
    electrode_id=0,
    stimulus_waveform=pulse_stim,
    target_group_name="lif_neurons",
    influence_radius=60*b2.um # Neurons within this radius of electrode 0
)

# 6. Add a spike monitor
sim.add_recording(
    monitor_name="all_spikes",
    monitor_type="spike",
    target_brian_obj=my_organoid.get_neuron_group("lif_neurons")
)

# 7. Run simulation
sim.run(100*b2.ms)

# 8. Retrieve and plot data (example)
# spike_data = sim.get_data("all_spikes")
# from pyborg.visualization import spike_plotter
# spike_plotter.plot_raster(spike_data.i, spike_data.t, duration=100*b2.ms)
# b2.show()

sim.close() # Closes SQLite connection if used
```

### 2. Reading Activity: MEA Spikes and Calcium Imaging

This snippet demonstrates how to record spikes from an MEA region and simulated calcium fluorescence.

```python
import brian2 as b2
from pyborg.organoid import Organoid, spatial
from pyborg.core import neuron_models # Assuming LIFCalciumFluorNeuron exists
from pyborg.mea import MEA
from pyborg.simulation import Simulator

# Ensure reproducible results
b2.seed(123)

# 1. Create an Organoid with calcium-enabled neurons
# (Assuming LIFCalciumFluorNeuron includes 'Ca' and 'F' state variables)
calcium_lif_model = neuron_models.LIFCalciumFluorNeuron(tau_ca=50*b2.ms, tau_F=100*b2.ms)
calcium_organoid = Organoid(name="CalciumOrganoid")
positions_ca = spatial.random_positions_in_cube(N=50, side_length=150*b2.um)
calcium_organoid.add_neurons(
    num_neurons=50,
    model_template=calcium_lif_model,
    positions=positions_ca,
    name="calcium_neurons"
)
# (Add synapses and some baseline input current for activity)
calcium_organoid.get_neuron_group("calcium_neurons").I = 0.7 * b2.nA

# 2. Create an MEA (optional for calcium, but can be used for LFP)
# mea_layout_ca = mea.generate_grid_layout(rows=4, cols=4, spacing=70*b2.um)
# my_mea_ca = MEA(electrode_positions=mea_layout_ca)

# 3. Setup the Simulator
# sim_ca = Simulator(organoid=calcium_organoid, mea=my_mea_ca)
sim_ca = Simulator(organoid=calcium_organoid) # MEA is optional here
sim_ca.setup_simulation_entry(
    organoid_config_details={"neuron_type": "LIFCalciumFluor"}
)

# 4. Add MEA-based spike recording (e.g., for a specific electrode)
# This is a conceptual example; actual MEA recording might involve LFP proxies
# or spike detection from neurons near an electrode.
# For simplicity, we'll record all spikes here.
sim_ca.add_recording(
    monitor_name="mea_spikes_region",
    monitor_type="spike",
    target_brian_obj=calcium_organoid.get_neuron_group("calcium_neurons") # Or a subgroup
)

# 5. Add Calcium/Fluorescence recording
sim_ca.add_recording(
    monitor_name="fluorescence_trace",
    monitor_type="state", # Using StateMonitor for calcium variables
    target_brian_obj=calcium_organoid.get_neuron_group("calcium_neurons"),
    variables_to_record=['F', 'Ca'], # Record Fluorescence and Calcium
    record_indices=list(range(5)) # Record first 5 neurons for example
)

# 6. Run simulation
sim_ca.run(200*b2.ms)

# 7. Retrieve and plot data
# spikes_mea_region = sim_ca.get_data("mea_spikes_region")
# from pyborg.visualization import spike_plotter
# spike_plotter.plot_raster(spikes_mea_region.i, spikes_mea_region.t, duration=200*b2.ms)
# b2.show()

# fluorescence_data = sim_ca.get_data("fluorescence_trace")
# from pyborg.visualization import calcium_plotter
# from pyborg.analysis import calcium_analysis
# F_traces = fluorescence_data.F
# t_traces = fluorescence_data.t
# delta_F_over_F = calcium_analysis.calculate_deltaF_over_F(F_traces, t_traces)
# calcium_plotter.plot_calcium_traces(
#     traces_dict={i: delta_F_over_F[i] for i in range(delta_F_over_F.shape[0])},
#     timestamps=t_traces,
#     title="Simulated ΔF/F Traces"
# )
# b2.show()

sim_ca.close()
```

## Project Directory Structure

```
pyBOrg/
├── pyborg/
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── neuron_models.py
│   │   ├── synapse_models.py
│   │   └── network_builder.py
│   ├── organoid/
│   │   ├── __init__.py
│   │   ├── organoid.py
│   │   └── spatial.py
│   ├── mea/
│   │   ├── __init__.py
│   │   └── mea.py
│   ├── electrophysiology/
│   │   ├── __init__.py
│   │   ├── stimulus_generator.py
│   │   ├── brian_monitors.py
│   │   └── data_persistence/
│   │       ├── __init__.py
│   │       ├── db_schema.py
│   │       ├── sqlite_writer.py
│   │       └── sqlite_reader.py
│   ├── network_plasticity/
│   │   ├── __init__.py
│   │   ├── base_plasticity_rule.py
│   │   ├── stdp.py
│   │   ├── homeostatic.py
│   │   ├── structural_plasticity.py
│   │   └── growth_guidance.py
│   ├── simulation/
│   │   ├── __init__.py
│   │   └── simulator.py
│   ├── analysis/
│   │   ├── __init__.py
│   │   ├── spike_analysis.py
│   │   ├── calcium_analysis.py
│   │   └── network_inference/
│   │       ├── __init__.py
│   │       ├── base_inferrer.py
│   │       ├── spike_based_inference.py
│   │       └── calcium_based_inference.py
│   ├── visualization/
│   │   ├── __init__.py
│   │   ├── spike_plotter.py
│   │   ├── calcium_plotter.py
│   │   └── network_plotter.py
│   └── utils/
│       ├── __init__.py
│       └── config_parser.py
│
├── examples/
│   ├── README.md
│   ├── data/
│   │   └── (empty or example_config.json)
│   ├── 00_Installation_and_Setup.ipynb
│   ├── 01_Creating_Your_First_Organoid.ipynb
│   ├── 02_Running_a_Simple_Simulation_and_Recording_Spikes.ipynb
│   ├── 03_Exploring_Neuron_and_Synapse_Models.ipynb
│   ├── 04_Spatial_Arrangement_and_Connectivity_Rules.ipynb
│   ├── 05_MEA_Stimulation_and_Basic_Recording.ipynb
│   ├── 06_Simulating_Spontaneous_Activity_and_LFP_Proxy.ipynb
│   ├── 07_Patterned_Stimulation_and_Response_Analysis.ipynb
│   ├── 08_Implementing_Synaptic_Plasticity_STDP.ipynb
│   ├── 09_Simulating_Structural_Plasticity_Network_Formation.ipynb
│   ├── 10_Modeling_Calcium_Dynamics_and_Fluorescence_Imaging.ipynb
│   ├── 11_Data_Persistence_Saving_and_Loading_with_SQLite.ipynb
│   ├── 12_Inferring_Functional_Networks_from_Spike_Data.ipynb
│   ├── 13_Inferring_Functional_Networks_from_Calcium_Data.ipynb
│   ├── 14_Advanced_Analysis_Bursting_and_Synchrony.ipynb
│   └── 15_Example_Modeling_a_Simplified_Disease_Phenotype.ipynb
│
├── tests/
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   └── test_neuron_models.py
│   ├── organoid/
│   │   ├── __init__.py
│   │   └── test_organoid.py
│   ├── mea/
│   │   ├── __init__.py
│   │   └── test_mea.py
│   ├── electrophysiology/
│   │   ├── __init__.py
│   │   └── test_stimulus_generator.py
│   │   └── data_persistence/
│   │       ├── __init__.py
│   │       └── test_sqlite_writer.py
│   ├── network_plasticity/
│   │   ├── __init__.py
│   │   └── test_stdp.py
│   ├── simulation/
│   │   ├── __init__.py
│   │   └── test_simulator.py
│   ├── analysis/
│   │   ├── __init__.py
│   │   └── test_spike_analysis.py
│   │   └── network_inference/
│   │       ├── __init__.py
│   │       └── test_spike_based_inference.py
│   └── (Other test files matching module structure)
│
├── docs/
│   ├── conf.py
│   ├── index.rst
│   ├── installation.rst
│   ├── tutorials/
│   │   └── (links to or copies of notebooks)
│   └── api/
│       └── (autogenerated API documentation for each module)
│
├── .gitignore
├── LICENSE
├── README.md
├── setup.py
└── requirements.txt
```

## Installation

```bash
# Clone the repository
git clone https://github.com/lucianosilvadevcode/pyborg/pyborg.git
cd pyBOrg

# Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Install pyBOrg in editable mode (for development)
pip install -e .
```

## Usage

Please refer to the Jupyter Notebooks in the `examples/` directory for detailed usage instructions and demonstrations. Start with `00_Installation_and_Setup.ipynb`.

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues.

## License

This project is licensed under the [MIT License]. See the `LICENSE` file for details.

