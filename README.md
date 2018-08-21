---
title: "Snakemake READ.ME"
output:
  github_document:
    pandoc_args: --webtex
---

# Snakemake workflow: quantum-monte-carlo
This workflow performs a quantum monte carlo (QMC) calculation using "trial" wave function (WFN)
generated by QCHEM and ORCA.

## Authors

* Vladimir Konjkov

## Usage

### Step 1: Install workflow and additional utilities

Download workflow from [repository](https://github.com/Konjkov/snakerules)

Download [molden2qmc.py](https://github.com/Konjkov/molden2qmc) utility required to convert "trial" WFN from MOLDEN output format to CASINO input format.

Download [multideterminant.py](https://github.com/Konjkov/molden2qmc) utility required to convert multi determinant information available in CASSCF and post-HF methods to CASINO format.

Copy them to the directory in the current path.

Choose program to generate "trial" WFN (QCHEM > 4.0 or ORCA 4.0.1) and install it.


### Step 2: Configure workflow

Configure the workflow according to your needs via editing the file `ORCA/Snakefile` or `QCHEM/Snakefile` depending on what program you will use.

The basic information necessary for calculations is contained in global variables:

* MOLECULES is a list of molecular geometry file names in xyz-format (without extension) located in the `chem_database` directory for which the calculation will be performed.

* METHODS is a list of quantum chemistry methods to obtain a "trial" WFN.

* BASES is a list of bases in which the "trial" function will be represented.

* JASTROWS is a filename of template (without extension) defining JASTROW factor

Depending on the program used to generate "trial" WFN, the following rules are available:

#### ORCA rules:

* calculation to generate gwfn.data file and in case of CASSCF method correlation.data.
```
rule ALL_ORCA:
    input: '{method}/{basis}/{molecule}/gwfn.data'
```
Where:
* __method__ - method available in ORCA to calculate "trial" WFN like HF, any DFT methods (i.e. B3LYP, CAM-BLYP, PBE0),
OO-RI-MP2, CASSCF(N,M) for multideterminant extension. CASSCF(N,M) should be coded as CASSCF_N_M
* __basis__ - any basis available in ORCA (i.e. cc-pVDZ, aug-cc-pVQZ, def2-SVP).
* __molecule__ - molecular geometry file name in xyz-format (without extension) located in the `chem_database` directory.

#### QCHEM rules

* calculation to generate gwfn.data file and in case of OD, OD(2), VOD, VOD(2), QCCD, QCCD(2), VQCCD correlation.data.
```
rule ALL_QCHEM:
    input: '{method}/{basis}/{molecule}/gwfn.data'
```
Where:
* __method__ - method available in QCHEM to calculate "trial" WFN including HF, any DFT methods (i.e. B3LYP, CAM-BLYP, PBE0), OO-RI-MP2, any orbital optimized methods from the list (OD, OD(2), VOD, VOD(2), QCCD, QCCD(2), VQCCD).
in case of OO-method T2-amplitudes where used as determinant's weights but some type of active space truncation should be specified.\
It should be done with two ways:
    - __OD_10__ first 10 active orbitals were taken.
    - __OD\_\.01__ all orbitals with T2-amplitudes greater then 0.01 were taken.
* __basis__ - any basis available in QCHEM (i.e. cc-pVDZ, aug-cc-pVQZ, pc-1).
* __molecule__ - molecular geometry file names in xyz-format (without extension) located in the `chem_database` directory.

#### QMC rules

* pure VMC calculation (without JASTROW)
    ```
    rule ALL_VMC:
        input: '{method}/{basis}/{molecule}/VMC/{nstep}/out'
    ```
    Where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule intended to check whether the conversion of the "trial" WFN to the CASINO format is correct and HF energy is equal to pure VMC one.
    It is not required to calculate the DMC energy.
* JASTROW coefficients optimization using some optimization plan
    ```
    rule ALL_VMC_OPT:
        input: '{method}/{basis}/{molecule}/VMC_OPT/{opt_plan}/{jastrow}/out'
    ```
    Where:
    * __opt_plan__ - name (without extension) of CASINO input file in `opt_plan` directory which define JASTROW optimization plan.
    * __jastrow__ - name (without extension) of template for `parameters.casl` file in `casl` directory.
* VMC energy calculation with optimized JASTROW coefficients.
    ```
    rule ALL_VMC_OPT_ENERGY:
        input: '{method}/{basis}/{molecule}/VMC_OPT_ENERGY/{opt_plan}/{jastrow}/{nstep}/out'
    ```
    Where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule is not required to calculate the DMC energy, but this may be necessary if you need the VMC energy with high accuracy.
* FN-DMC energy calculation to achieve desired accuracy (specified in the variable STD_ERR).
    ```
    rule ALL_VMC_DMC:
        input: '{method}/{basis}/{molecule}/VMC_DMC/{opt_plan}/{jastrow}/tmax_{dt}_{nconfig}_{i}/out'
    ```
    Where:
    * __dt__ - part of denominator (integer) to calculate the DMC time step using the formula $1.0/(max_Z^2 * 3.0 * dt)$.
    * __nconfig__ - number of configuration in DMC colculation (1024 is recomended)
    * __i__ - stage of DMC calculation (1 - only DMC equilibration and fixed step (50000) of DMC accumulation run, 2 - additional DMC accumulation run to achieve desired accuracy)
* JASTROW coefficients optimization using some optimization plan with BACKFLOW transformed WFN.
    ```
    rule ALL_VMC_OPT_BF:
        input: '{method}/{basis}/{molecule}/VMC_OPT_BF/{opt_plan}/{jastrow}__{backflow}/out'
    ```
    Where:
    * __backflow__ - combination of {ETA-term}\_{MU-term}\_{PHI-term-electron-nucleus}{PHI-term-electron-electron} expansion orders:
        - 3_3_00 - ETA-term expansion of order 3 and MU-term of order 3, no PHI-term
        - 9_9_33 - ETA-term expansion of order 9 and MU-term of order 9, PHI-term with electron-nucleus and electron-electron expansion of order 3 both.

    It should be noted that the BACKFLOW transformation in CASINO is implemented only up to f-orbitals.
* VMC energy calculation with optimized JASTROW coefficients with BACKFLOW transformation.
    ```
    rule ALL_VMC_OPT_ENERGY_BF:
        input: '{method}/{basis}/{molecule}/VMC_OPT_ENERGY_BF/{opt_plan}/{jastrow}__{backflow}/{nstep}/out'
    ```
    Where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule is not required to calculate the DMC energy.
* FN-DMC energy calculation to achieve desired accuracy (specified in the variable STD_ERR) with BACKFLOW transformed WFN.
    ```
    rule ALL_VMC_DMC_BF:
        input: '{method}/{basis}/{molecule}/VMC_DMC_BF/{opt_plan}/{jastrow}__{backflow}/tmax_{dt}_{nconfig}_{i}/out'
    ```

### Step 3: Execute workflow

All you need to execute this workflow is to install Snakemake via the [Conda package manager](http://snakemake.readthedocs.io/en/stable/getting_started/installation.html#installation-via-conda). Software needed by this workflow is automatically deployed into isolated environments by Snakemake.

Test your configuration by performing a dry-run via

    snakemake <rule> -n

Execute the workflow locally via

    snakemake <rule>

## Examples

To demonstrate the possibilities of this workflow, examples of calculations are given.

### ORCA

#### HF "trial" WFN for H-Ne atoms

* H $(^2S)$

  "trial" WFN for the ground state of H-atom is nodeless, so DMC energy is exact, also as no electron correlations are present VMC energy is always exact. 

* He $(^1S)$

  spacial part of "trial" WFN for the ground state of He-atom is symmetric so has no nodal surface, thus DMC energy is exact.

* Li $(^2S)$

* Be $(^1S)$

* B $(^2P_{1/2})$

* C $(^3P_0)$

### QCHEM
