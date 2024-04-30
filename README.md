# DPRc Tutorial

This tutorial shows a simple example to train a [DPRc model](https://doi.org/10.1021/acs.jctc.1c00201) and perform simulations.

This tutorial is aimed to show readers how to use DPRc. It is not for production. For simplicity, the tutorial takes only one window from the ethylene phosphate reaction. The detail of this reaction is described in [J. Phys. Chem. A 2022, 126, 45, 8519–8533](https://doi.org/10.1021/acs.jpca.2c06201).

Before this tutorial, assume you have some basic knowledge of AMBER. If not, follow [AMBER tutorials](https://ambermd.org/tutorials/), especially the sections for setting up a system, running umbrella sampling, and running QM/MM simulations.

## Software

Assume you have a local machine and a remote machine separately, you need to install the following software.

| Software                                                  | Machine          | Version   | Documentation | Additional notes |
| --------------------------------------------------------- | ---------------- | --------- | ------------- | ------------ |
| [DP-GEN](https://github.com/deepmodeling/dpgen)           | local            | >= 0.12.0 | [DPRc](https://docs.deepmodeling.com/projects/dpgen/en/latest/run/param.html#run-jdata-model-devi-engine-amber)
| [dpamber](https://github.com/njzjz/dpamber)               | local and remote | >= 0.3.0  | [README](https://github.com/njzjz/dpamber)
| [DeePMD-kit](https://github.com/deepmodeling/deepmd-kit)  | remote           | >= 2.2.8  | [DPRc](https://docs.deepmodeling.com/projects/deepmd/en/latest/model/dprc.html) | Python interface; C++ interface to Ambertools
| [AmberTools](https://ambermd.org/)                        | remote           | >= 2024   | [Manual](https://ambermd.org/doc12/Amber24.pdf)<br/>[AmberDPRc](https://gitlab.com/RutgersLBSR/AmberDPRc/) | Enable [DeePMD-kit](https://github.com/deepmodeling/deepmd-kit) and [QUICK](https://github.com/merzlab/QUICK) interface

Please give the proper credits to all the software above.

## Initial data

### Prepare initial systems

Firstly, one needs to use `tleap` to generate a parm7 file, minimize the structure, and then use a semi-empirical QM/MM method to generate the starting structures for umbrella windows. This process has been introduced in the [AMBER tutorials](https://ambermd.org/tutorials/), so this tutorial will not pay attention to it.

For convenience, the parm7 file and the starting structure have been added to this repository, located in [`parm7/ETP_ETH.parm7`](parm7/ETP_ETH.parm7) and [`rst7/init_-1.50.rst7`](rst7/init_-1.50.rst7).

### Generate initial training data

Here assume you have had `ETP_ETH.parm7` and `init_-1.50.rst7`.
Then you need to use `sander` to run several fast, semi-empirical QM/MM simulations in different random seeds, and print both energy and forces. The mdin file is provided in [`mdin/low_level_md.mdin`](mdin/low_level_md.mdin).

```sh
sander -O -p ETP_ETH.parm7 -c init_-1.50.rst7 -i low_level_md.mdin -o rc.mdout -r rc.rst7 -x mndod.nc -inf rc.mdinfo -ref init_-1.50.rst7 -frc mndod.mdfrc -e mndod.mden
```

Then, you need to use `sander` to run *ab initio* QM/MM calculation from the given trajectory `mndod.nc` with `imin = 6` (note: `imin=6` may not be supported in old AMBER versions). An example mdin file is provided in [`mdin/high_level.mdin`](mdin/high_level.mdin), but you need to modify it to match your DFT software.

```sh
sander -O -p ETP_ETH.parm7 -c init_-1.50.rst7 -i high_level_relabel.mdin -o high_level.mdout -r high_level.rst7 -x high_level.nc -y mndod.nc -frc high_level.mdfrc -inf high_level.mdinfo -e high_level.mden
```

Now you have both high-level and low-level data. Then use [`dpamber corr`](https://github.com/njzjz/dpamber) to generate the initial training data:

```sh
dpamber corr --cutoff 6. --qm_region ":1-2" --parm7_file ETP_ETH.param7 --nc mndod.nc --hl pbe0 --ll mndod --out init_data.hdf5
```

For convenience, we provide an example of the initial data in [`init_data/init_data.hdf5.tar.bz2`](init_data/init_data.hdf5.tar.bz2), and you can extract it and jump to the next step.

```sh
cd init_data
tar vxjf init_data.hdf5.tar.bz2
```

## Setup initial files

Here we have five directories:

- [`init_data`](init_data): initial training data.
- [`parm7`](parm7): contains parm7 files.
- [`mdin`](mdin): contains mdin files for simulations and relabeling.
  - `ml.mdin` is the template for DPRc QM/MM simulation;
  - `high_level.mdin` is the template for high-level *ab initio* QM/MM calculation; 
  - `low_level.mdin` is the template for low-level semi-empirical QM/MM calculation.
- [`rst7`](rst7): contains starting structures. 
- [`disang`](disang): contains distance and angle constraints for umbrella sampling.

These files have been prepared in advance, but you need to modify the mdin file for the high-level calculation to match your software package.

## Running DP-GEN

You need to prepare two JSON files, one for [parameters](https://docs.deepmodeling.com/projects/dpgen/en/latest/run/param.html) and one for the [machine](https://docs.deepmodeling.com/projects/dpgen/en/latest/run/mdata.html). You need to modify the machine file to match your machines. Click the link for a detailed explanation.

The explanation for DeePMD-kit training parameters can be found [here](https://docs.deepmodeling.com/projects/deepmd/en/master/train/train-input.html).

After these files are ready, run DP-GEN on the local machine:

```sh
dpgen run param.json machine.json
```

For simplicity, we only run two iterations, and the output will look like

```
INFO:dpgen:-------------------------iter.000000 task 06--------------------------
INFO:dpgen:system 000 candidate :   1112 in   2000  55.60 %
INFO:dpgen:system 000 failed    :     28 in   2000   1.40 %
INFO:dpgen:system 000 accurate  :    860 in   2000  43.00 %
INFO:dpgen:system 000 accurate_ratio:   0.4300    thresholds: 1.0000 and 1.0000   eff. task min and max   -1 1000   number of fp tasks:   1000
INFO:dpgen:-------------------------iter.000001 task 06--------------------------
INFO:dpgen:system 000 candidate :    619 in   2000  30.95 %
INFO:dpgen:system 000 failed    :      0 in   2000   0.00 %
INFO:dpgen:system 000 accurate  :   1381 in   2000  69.05 %
INFO:dpgen:system 000 accurate_ratio:   0.6905    thresholds: 1.0000 and 1.0000   eff. task min and max   -1 1000   number of fp tasks:    619
```

The ratio of accurate frames is increased in the second iteration. If the active learning cycle continues, the accurate ratio will coverage to 100%.
