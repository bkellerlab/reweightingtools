# reweightingtools
Software package collection for dynamical Girsanov path reweighting  
   
## Girsanov path reweighting
Dynamical reweighting methods are becoming more important to study complex mechanisms of chemical processes, whose direct simulations of the full state-space at the correct statistical weight are impossible. 
In order to gain a better understanding of the applicability of Girsanov path reweighting to chemical processes in complex environments, a broad network between already high-quality methods must be created.
The [article](https://pubs.acs.org/doi/full/10.1021/acs.jpcb.4c01702) presents the basis for how Girsanov path reweighting can be integrated into any simulation and analysis software. 
We also provide a ready-to-use simulation program that enables on-the-fly estimation of reweighting factors and a unified procedure for estimating a reweighted dynamical model from this data.

## Software
- simulation programs: [openmmtools](https://github.com/anyschaefer/openmmtools);
  we started a [pull request](https://github.com/choderalab/openmmtools/pull/729)
  - reporter function: [ReweightingReporter](https://github.com/bkellerlab/reweightingtools/blob/main/reweightingtools/simulation/openmmSimulation/reweightingreporter.py)
- analysis package: [deeptime](https://github.com/deeptime-ml/deeptime)

### Setup
1. create new environment (e.g. RWGHTSoftware) with `conda create -n RWGHTSoftware python=3.11` 
2. install the simulation and analysis tools

   a) download [existing simulation software](https://github.com/anyschaefer/openmmtools) with reweighting modification

   b) clone the respective package by
   	- copying the "HTTPS clone URL" link that appears after selecting the code icon and 
   	- run `git clone [url]` in the directory you want to download it to

   c) install the openmmtools extension in cloned openmmtools folder via `pip insatll .`, further possible dependencies are
   
     `conda install -c conda-forge openmm`

     `conda install -c conda-forge mdtraj`

     `conda install -c conda-forge netcdf4`

     `conda install -c conda-forge mpiplus`

     `conda install -c omnia pymbar`

     `conda install -c numba numba`
   

   d) go to [deeptime](https://github.com/deeptime-ml/deeptime) and follow the installation instructions

4. install further dependencies, e.g. `conda install -c anaconda matplotlib`, `conda install -c anaconda spyder`
  
5. create scripts for simulation and analysis with Girsanov reweighting or use the setup and templates in [reweightingtools](https://github.com/bkellerlab/reweightingtools)

   Example studies are discussed in [Implementation of Girsanov reweighting in OpenMM and Deeptime](https://pubs.acs.org/doi/full/10.1021/acs.jpcb.4c01702).

a) The propagation of a system, cf. Fig. 3 and Fig. 5 b-d, according to a Langevin symmetric splittig scheme at a unbiased (`openMM_simulation()`) or biased potential (`openMM_biased_simulation()`) can be realized with the help of some top-level functions of the [reweightingtools simulation-api](https://github.com/bkellerlab/reweightingtools/blob/main/reweightingtools/simulation/openmmSimulation/api.py). A detailed description of the setup can be found in the respective function documentation, e.g. here for the biased simulation:

  ```py
   # Import
   from reweightingtools.simulation import openMM_biased_simulation, restraints_harmonic_force
   from openmm.unit import nanometer
            
   # MD input
   forcefield='amber99.ff'
   equisteps= 5000
   nsteps= 5000000000
   integrator_scheme= 'LangevinWithGirsanov'
   integrator_splitting='R V O V R'
   nstxout=100
   solvent='HOH'

   restraints_kwargs=dict()   
   restraints_kwargs['multiple_restraints']=True
   restraints_kwargs['atom_name']=[('CA'),('CL')] 
   restraints_kwargs['force_constants']=[200000.0,500000.0]
   restraints_kwargs['restraint_x']=[True, True]
   restraints_kwargs['restraint_y']=[True, True]
   restraints_kwargs['restraint_z']=[False, True]

   # Run openMM simulation
   openMM_biased_simulation(forcefield,
                            equisteps,
                            nsteps,
                            integrator_scheme,
                            integrator_splitting,
                            nstxout,
                            gro_input='ClCaCl',
                            plumed_file='plumedCOLVAR.dat',
                            externalForce_file='bias_05z.txt',
                            restraints_func= [restraints_harmonic_force,restraints_harmonic_force], 
                            restraints_kwargs=restraints_kwargs,
                            fix_system=True,
                            #restart=True,
                            dcd_output=False,
                            pdp_output=False,
                            data_output=False
                            )
  ```


    
b) Dynamic properties of a system, such as implicit timescales (`deeptime_implied_timescales()`) or dominant eigenvectors (`deeptime_eigenvectors()`) of a Markov state models (MSM) (cf. Fig. 4 or Fig. 5 b-d) can be analyzed with the help of some top-level functions of the [reweightingtools analysis-api](https://github.com/bkellerlab/reweightingtools/blob/main/reweightingtools/analysis/deeptimeMSM/api.py). For more customization of the analysis see the [deeptime documentation](https://deeptime-ml.github.io/latest/api/index_markov_tools.html#msm-analysis). Here an example for evaluating implied timescales: 

```py
# Import
from reweightingtools.analysis.discretisation import *
from reweightingtools.analysis.api import *
        
# Set input
cwd='/home/project/'
analysisMD=cwd+'analysis_MD' 
gridsize=100
reversible=True
nstxout=100
lagtimes=[1,3,5,7,9,10] 
max2D, min2D = np.array([ 1.5 ,  3.5]), np.array([-3.5, -1.5])
traj  = np.load(MD_directory+'/traj.npy') 
dtraj = trajectory2D_MBP(traj,
                         gridsize,
                         min2D,
                         max2D)
np.save(analysisMD+'/discretized_'+str(gridsize),dtraj)
        
# Prepare directory, eg. /discretized, containing discretized 
# trajectory, g and M factor output from openMM simulation using 
# the LangevinSplittingGirsanov integrator
deeptime_implied_timescales(lagtimes=lagtimes,
                            gridsize=gridsize,
                            reversible=reversible,
                            reweighting=True,
                            its_file='/its_rwght',
                            analysisMD=analysisMD,
                            dtraj_file='/discretized',
                            gF_file='/g_factors',
                            MF_file='/M_factors',
                            save_its=True)
        
  # Visualize reweighted implied timescales 
  its_rwght = np.load(analysisMD+'/its_rwght_'+str(gridsize)+'.npy')
  i= 0 # slowest process
  plt.plot(np.array(lagtimes),its_rwght[:,i],label='rwght')
  ```

To get the first two dominant eigenvectors of a $36\times 36$ MSM constructed on trajectories resulting from a biased simulation with Girsanov reweighting import the analysis api `from reweightingtools.analysis.api import *`. Prepare a directory `analysisMD:str='analysis_MD'`, with the discretized trajectory ` dtraj_file:str='/discretized'`, as well with both $g$ and $M$ factor (`gF_file:str='/g_factors'`, `MF_file:str='/M_factors'`) from openMM simulation using `LangevinSplittingGirsanov` integrator.

```py           
deeptime_eigenvectors(lagtime=150,
                      gridsize=36,
                      number_eigenvectors=2,
                      reversible=True,
                      reweighting=True,
                      analysisMD:str='analysis_MD',
                      eigenvector_file:str='/evecs',
                      dtraj_file:str='/discretized',
                      gF_file:str='/g_factors',
                      MF_file:str='/M_factors'
                      )
```
