Copyright Georgia Tech 2024

[![test status](https://github.com/AI4OPT/ML4OPF/workflows/tests/badge.svg)](https://github.com/AI4OPT/ML4OPF/actions/workflows/tests.yaml)
[![coverage](.github/coverage.svg)](https://github.com/AI4OPT/ML4OPF/actions/workflows/tests.yaml)
[![docs status](https://github.com/AI4OPT/ML4OPF/workflows/docs/badge.svg)](https://pages.github.com/AI4OPT/ML4OPF/index.html)

# ML4OPF

ML4OPF is a Python package for developing machine learning proxy models for optimal power flow. It is built on top of PyTorch and PyTorch Lightning, and is designed to be modular and extensible. The main components are:

- [`formulations`](ml4opf/formulations): The main interface to the OPF formulations [ACOPF](ml4opf/formulations/acp), [DCOPF](ml4opf/formulations/dcp), and [Economic Dispatch](ml4opf/formulations/ed).

  - Each OPF formulation has three main component classes: `OPFProblem`, `OPFViolation`, and `OPFModel`. The `OPFProblem` class loads and parses data from disk, the `OPFViolation` class calculates constraints residuals, incidence matrices, objective value, etc., and the `OPFModel` class is an abstract base class for proxy models.

<!-- - [`functional`](ml4opf/functional): The functional interface underlying the object-oriented interface in `formulations`, which makes less assumptions (i.e., all problem data may vary but the user is responsible for keeping track of it). -->

- [`loss_functions`](ml4opf/loss_functions): Various loss functions including [LDFLoss](ml4opf/loss_functions/ldf.py) and the self-supervised [ObjectiveLoss](ml4opf/loss_functions/objective.py).

- [`layers`](ml4opf/layers): Various feasibility recovery layers including [BoundRepair](ml4opf/layers/bound_repair.py) and [HyperSimplexRepair](ml4opf/layers/hypersimplex_repair.py).

- [`models`](ml4opf/models): Various proxy models including [BasicNeuralNet](ml4opf/models/basic_nn), [LDFNeuralNet](ml4opf/models/ldf_nn), and [E2ELR](ml4opf/models/e2elr).

- [`parsers`](ml4opf/parsers): Parsers for data generated by [AI4OPT/OPFGenerator](https://github.com/AI4OPT/OPFGenerator). <!-- Note that when selecting the test set the seed is fixed to ensure reproducibility. -->

- [`viz`](ml4opf/viz): Visualization helpers for plots and tables.

Documentation based on docstrings is live [here](https://pages.github.com/AI4OPT/ML4OPF/index.html).


## Installation

To install `ml4opf` on macOS (CPU/MPS) and Windows (CPU), run:
```bash
pip install git+ssh://git@github.com/AI4OPT/ML4OPF.git

# or, to install with optional dependencies (options: "all", "dev", "viz"):
pip install "ml4opf[all] @ git+ssh://git@github.com/AI4OPT/ML4OPF.git"
```
If you don't already have PyTorch on Linux (CPU/CUDA/ROCm) or Windows (CUDA), make sure to provide the correct `--index-url` which you can find [here](https://pytorch.org/get-started/locally/). For example, to install from scratch with CUDA 12.1 and all optional dependencies:
```bash
pip install "ml4opf[all] @ git+ssh://git@github.com/AI4OPT/ML4OPF.git" \
     --index-url https://download.pytorch.org/whl/cu121                \
     --extra-index-url https://pypi.python.org/simple/
```

For development, the recommended installation method is using Conda environment files provided at [environment.yml](environment.yml) and [environment_cuda.yml](environment_cuda.yml). Using [Mambaforge](https://mamba.readthedocs.io/en/latest/mamba-installation.html#mamba-install) is recommended for super fast installation:
```bash
git clone git@github.com:AI4OPT/ML4OPF.git # clone this repo
cd ML4OPF                                            # cd into the repo
mamba env create -f environment.yml                  # create the environment
conda activate ml4opf                                # activate the environment
pip install -e ".[all]"                              # install ML4OPF
```

## Usage


### Training a `BasicNeuralNet` for ACOPF
```python
import torch

# load data
from ml4opf import ACPProblem

data_path = ...
network = "300_ieee"

problem = ACPProblem(data_path, network)

# make a basic neural network model
from ml4opf.models.acp.basic_nn import BasicNeuralNet # requires pytorch-lightning

config = {
    "optimizer": "adam",
    "init_lr": 1e-3,
    "loss": "mse",
    "hidden_sizes": [500,300,500], # encoder-decoder structure
    "activation": "sigmoid",
    "boundrepair": "none" # optionally clamp outputs to bounds (choices: "sigmoid", "relu", "clamp")
}

model = BasicNeuralNet(config, problem)

model.train(trainer_kwargs={'max_epochs': 100, 'accelerator': 'auto'}) # pass args to the PyTorch Lightning Trainer

evals = model.evaluate_model()

from ml4opf.viz import make_stats_df
print(make_stats_df(evals))

model.save_checkpoint("./basic_300bus") # creates a folder called "basic_300bus" with two files in it, trainer.ckpt and config.json
```

### Manually Loading Data
```python
import torch

from ml4opf import ACPProblem

data_path = ...
network = "300_ieee"

# parse HDF5/JSON
problem = ACPProblem(data_path, network)

# get train/test set:
train_data = problem.train_data
test_data = problem.test_data

train_data['input/pd'].shape # torch.Size([52863, 201])
test_data['input/pd'].shape # torch.Size([5000, 201])

# if needed, convert the HDF5 data to a tree dictionary instead of a flat dictionary:
from ml4opf.parsers import H5Parser
h5_tree = H5Parser.make_tree(train_data) # this tree structure should
                                         # exactly mimic the
                                         # structure of the HDF5 file.
h5_tree['input']['pd'].shape # torch.Size([52863, 201])
```

## Acknowledgements

This material is based upon work supported by the National Science Foundation under Grant No. 2112533 NSF AI Institute for Advances in Optimization (AI4OPT). Any opinions, findings, and conclusions or recommendations expressed in this material are those of the author(s) and do not necessarily reflect the views of the National Science Foundation.