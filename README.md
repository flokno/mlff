# MLFF
Repository for training, testing and developing machine learned force fields using the `So3krates` transformer (check
out the [NeurIPS paper](https://proceedings.neurips.cc/paper_files/paper/2022/hash/bcf4ca90a8d405201d29dd47d75ac896-Abstract-Conference.html)).
## Installation
Assuming you have already set up an virtual environment with python version `>= 3.8.` In order to ensure compatibility
with CUDA `jax/jaxlib` have to be installed manually. Therefore **before** you install `MLFF` run 
```
pip install --upgrade pip
# Installs the wheel compatible with CUDA 11 and cuDNN 8.6 or newer.
# Note: wheels only available on linux.
pip install --upgrade "jax[cuda]" -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html
```
for details (e.g. CUDA 12) check the official [`JAX`](https://github.com/google/jax#pip-installation-gpu-cuda) repository.

Since the `optax` package has a `tensorflow` dependency for checkpointing and does not install it itself, we also have to 
install `tensorflow`, although the models in `mlff` are written in `JAX`. It can be installed running
```
pip install tensorflow
```
**Apple ARM M1/M2** 

In case you have an Apple ARM M1/M2 chip, the designated `tensorflow` package called `tensorflow-maxos` has to be 
installed. More details on doing so can be found e.g. [here](https://stackoverflow.com/questions/72964800/what-is-the-proper-way-to-install-tensorflow-on-apple-m1-in-2022)
or [here](https://developer.apple.com/metal/tensorflow-plugin/) (the metal plugin is not needed). 
Unfortunately, we didn't found an M1 for testing, so we are happy about reported issues.

Next clone the `mlff` repository by running
```
git clone https://github.com/thorben-frank/mlff.git
```
 

Now do
```
cd mlff
pip install -e .
```
which completes the installation and installs remaining dependencies.
## Weights and Bias
If you do not have a weights and bias account already you can create on [here](https://wandb.ai/site). After installing
``mlff`` run
```
wandb login
```
and log in with your account.
# Quickstart
Following we will give a quick start how to train, evaluate and run an MD simulation with the 
`So3krates` model.
## Training
Train your fist `So3krates` model by running
```
train_so3krates --data_file data.xyz  --n_train 1000 --n_valid 100 --wandb_init project=so3krates,name=first_run
```
The `--data_file` can be any format digestible by the `ase.io.read` method. In case minimal image convention
should be applied, add `--mic` to the command. The model parameters will be saved per default to `/module`. Another 
directory can be specified using `--ckpt_dir $CKPT_DIR`, which will safe the model parameters to `$CKPT_DIR/module`. 
More details on training can be found in the detailed training section below.
## Evaluation
After training, change into the model directory, e.g. and run the `evaluate` command
```
cd module
evaluate
``` 
As before, when your data is not in eV and Angstrom add the `--units` keyword. The reported metrics are then in eV and
Angstrom (e.g. `--units energy='kcal/mol',force='kcal/(mol*Ang)'` if the energy in your data is in `kcal/mol`).
## Molecular Dynamics
You can use the `mdx` package which is the `mlff` internal MD package, fully relying on `jax` and thus fully 
optimized for XLA compilation on GPU. Before you can use it make sure you install the [`glp`](https://github.com/sirmarcel/glp) 
package by cloning the `glp` repository and install it
```
git clone git@github.com:sirmarcel/glp.git
cd glp

pip install .
```
First, lets create a relaxed structure, using the LBFGS optimizer
```
run_relaxation  --qn_max_steps 1000 --qn_tol 0.0001 --use_mdx
```
which will save the relaxed geometry to `relaxed_structure.h5`. Next, convert the `.h5` file to an 
`xyz` file, by running
```
trajectory_to_xyz --trajectory relaxed_structure.h5 --output relaxed_structure.xyz
```
We now run an MD with the relaxed structure as start geometry
```
run_md --start_geometry relaxed_structure.xyz --thermostat velocity_verlet --temperature_init 600 --time_step 0.5 --total_time 1 --use_mdx
```
Temperature is in Kelvin, time step in femto seconds and total time in nano seconds. It will save a `trajectory.h5` 
file to the current working directory.
### Analysis
After the MD is finished you can either work with the `trajectory.h5` using e.g. a `jupyter notebook` and `h5py`. 
Alternatively, you can run
```
trajectory_to_xyz --trajectory trajectory.h5 --output trajectory.xyz
```  
which will create an `xyz` file. The resulting `xyz` file can be used as input to the 
[`MDAnalysis`](https://docs.mdanalysis.org/stable/index.html) python package, which provides a broad range of functions 
to analyse the MD simulations. The central `Universe` object can be creates easily as
```python
import MDAnalysis as mda

# Load MD simulation results from xyz
u = mda.Universe('trajectory.xyz')
```
# Deep Dive
In the quickstart section we went through a few basic steps, allowing to train, validate a `so3krates` model as well 
as running an MD simulation. If you want to learn more about each of the steps, check the following sections.
## Training
Lets start start from the training command already shown in the quickstart section
```
train_so3krates --data_file atoms.xyz --n_train 1000 --n_valid 100
```
for which all data in `atoms.xyz` is loaded an split into `1000` data points for training, 
`100` data points for validation and the remaining data points `n_test = n_tot - 1000 - 100` is hold back for testing 
the potential after training. The validation data points are used to determine the best performing model during training,
for which the parameters are saved to `module/checkpoint_loss_XXX` where XXX denotes the training step for which the best performing
model was found. We will show later, how to load the checkpoint such that one use the trained potential directly in 
`Python`.
### Input Data Files
`mlff` can deal with any input file that can be read by the [`ase.io.read`](https://wiki.fysik.dtu.dk/ase/ase/io/io.html#ase.io.read) method.
Further `--data_file` admits to pass `*.npz` files. `*.npz` files allow to store `numpy.ndarray` under different `keys`. 
Thus, `mlff` needs to "know" under which key to find e.g. the positions, the forces and so on .. Per default, `mlff` 
assumes the following relations between property and key
```
{
 atomic_position: R,     # shape: (n_data, n, 3) 
 atomic_type: z,  # shape: (n_data, n) or (n)
 energy: E,       # shape: (n_data, 1)
 force: F         # shape: (n_data, n, 3)
}
```
If you have an `*.npz` file which uses a different convention, you can specify the keys customizing the property keys
via 
```
train_so3krates --data_file file.npz --n_train 1000 --n_valid 100 --prop_keys atomic_position=pos,atomic_type=numbers 
``` 
The above examples would assume that the properties `energy` and `force` are still found under the `keys` `E` and `F`,
respectively but `position` and `atomic_type` are found under `pos` and `numbers`.
### Units
Per default, `mlff` assumes the ASE default units which are `eV` for energy and `Angstrom` for coordinates. Some data
sets, however, differ from these convention, e.g. the MD17 or the MD22 data set. You can download the corresponding 
`*.npz` files [here](http://sgdml.org/#datasets) (Note that the `*.xyz` files provided there are not formatted in a 
way that allows reading them via `ase.io.read` method). For both data sets the energy is is in `kcal/mol` such that 
the forces are in `kcal/(mol*Ang)`. You can either pre-process the data yourself by applying the proper conversion 
factors and pass the data directly into the `train_so3krates` command. Alternatively, you can set them manually by
```
train_so3krates --train_file dha.npz --n_train 1000 --n_valid 500 --units energy='kcal/mol',force='kcal/(mol*Ang)'
```
Note the character strings for the units, which are necessary in the course of internal processing. This will internally
rescale the energy and the forces to `eV` and `eV/Ang`.
### Minimal Image Convention
In [1] `So3krates` was used to calculate EOS and heat flux in solids, such that it must be capable of
handling periodic boundary conditions. If you want to apply the minimal image convention, you can specify this by 
adding the corresponding flag to the training command 
```
train_so3krates --train_file file_with_pbc.xyz --n_train 100 --n_valid 100 --mic
```
Internally, `mlff` uses the [`ase.neighborlist.primitive_neighborlist`](https://wiki.fysik.dtu.dk/ase/ase/neighborlist.html#ase.neighborlist.primitive_neighbor_list) 
for computing the atomic neighborhoods.
### Energy Shifts
The energy scale of the data, tend to increase rapidly with the number of atoms in the system. However, relevant scale
of energy changes is usually on the scale of a few eV, such that one typically shifts the energy by the mean of the 
energies in the `--n_train` training data points. This is done by default when running the `train_so3krates` command. 
However, it might be desired to atom type specific shifts, which is possible via
```
train_so3krates --atoms.xyz --n_train 1000 --n_valid 200 --shifts 1=-500.30, 6=-6000.25
```
and would shift the energy by `-500.30` for each hydrogen in the structure and by `-6000.25` for each carbon.
### `So3krates` Hyperparameters
One can further vary different model hyperparameters: `--L` sets the number of message passing steps, `--F` sets the
feature dimension, `--degrees` sets the degrees for spherical harmonic coordinates and `--r_cut` sets the cutoff for the 
atomic neighborhoods. E.g. a model with 2 message passing layers, feature dimension 64, degree 1 and 2 and cutoff of 4 Angstrom
can be trained by running
```
train_so3krates --atoms.xyz --n_train 1000 --n_valid 200 --L 2 --F 64 --degrees 1 2 --r_cut 4
```
### Optimization Hyperparameters
TODO
### Weight and Bias
To keep track of the performed experiments, you can organize your trainings using weights and bias. You can specify the 
project and name of the current trainings run via
```
train_so3krates --atoms.xyz --n_train 1000 --n_valid 200 --wandb_init project=so3krates,name=deep_dive_run
```
The arguments passed to `--wandb_init` are passed as is to the `wandb.init` for which you can find all possible 
arguments [here](https://docs.wandb.ai/ref/python/init).
## Validation
In order to validate the models performance, the errors on the unseen test set are often a good starting point. Go to 
model you want to evaluate by going to the model directory (the directory in which the checkpoint_loss_XXX file lies)
and use the `evaluate` command
```
cd module
evaluate
``` 
This command will evaluate the model on the test data points and print the mean absolute error (MAE), the root mean squared 
error (RMSE) and the R2 spearson correlation. It will further save two files in the current directory which are called 
`metrics.json` and `evaluate_predictions.npz`. The former contains the metrics which have also been printed and the latter
contains the per data point predictions of the model. Thus, the metrics can be calculated from the `evaluate_predictions.npz`
by using the following code snippet
```python
import numpy as np

eval_data = np.load('evaluate_predictions.npz', allow_pickle=True)
predicted_energies = eval_data['predictions'].item()['E']
target_energies = eval_data['targets'].item()['E']
mae = np.abs(predicted_energies - target_energies).mean()
print(f'MAE: {mae:.4f} (eV)')
```
## Use `So3krates` in Python
While it is convenient to train and validate a model using the `CLI`, playing around with the potential is often much
easier in `Python` in particular if it is of interest to couple it with other methods and packages. 
### GLP
Below you find a code snippet how to load a trained `So3krates` model and use it in `Python`.
```python
import jax.numpy as jnp
from mlff.mdx import MLFFPotential

ckpt_dir = 'path/to/module/' 
dtype = jnp.float64
pot = MLFFPotential.create_from_ckpt_dir(ckpt_dir=ckpt_dir, dtype=dtype)
```
The resulting potential is a [`Potential`](https://github.com/sirmarcel/glp#potential) in the sense of the 
the [`glp`](https://github.com/sirmarcel/glp) package. Thus, you can directly use your trained `so3krates` model to 
perform energy, force, stress and heat flux calculations. `glp` also offers a binding to [`vibes`](https://vibes-developers.gitlab.io/vibes/), which 
further allows you to do Phonon calculations and much more. To calculate thermal conductivities you can use the [`gkx`](https://github.com/sirmarcel/gkx) 
package.
### MDx
TODO
## Run the tests
The test suite can be run with pytest as:
```
pytest tests/
```
## Cite
If you use parts of the code please cite the corresponding paper
```
@article{frank2022so3krates,
  title={So3krates: Equivariant attention for interactions on arbitrary length-scales in molecular systems},
  author={Frank, Thorben and Unke, Oliver and M{\"u}ller, Klaus-Robert},
  journal={Advances in Neural Information Processing Systems},
  volume={35},
  pages={29400--29413},
  year={2022}
}
```
## References
* [1] M.F. Langer, J.T. Frank, and F. Knoop.  [*Stress and heat flux via automatic differentiation*](https://arxiv.org/abs/2305.01401), arxiv 2305.01401 (2023)
