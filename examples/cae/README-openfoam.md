<!--
Copyright 2026 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# OpenFOAM CAE Blueprint with Visualization

This folder contains a reference blueprint (`openfoam-blueprint.yaml`) for deploying an HPC cluster specifically tailored for OpenFOAM workloads and visualization.

It provisions a Slurm cluster using Google Cloud's C2D instance family (AMD EPYC processors), ensuring a high memory bandwidth suitable for computation-heavy workloads. This solution is particularly designed to automate the installation of OpenFOAM and Paraview (with `osmesa` support) using the Spack package manager, and includes a pre-configured Chrome Remote Desktop (CRD) VM for visualizing results without transferring data back locally.

## Architecture and Components

The architecture consists of the following key Google Cloud and cluster components:

- **VPC Network**: A dedicated network and subnetwork for the compute cluster.
- **Filestore (NFS)**: Two centralized storage mounts used across all VMs:
  - `/home`: For user profiles, case files, and simulation output data.
  - `/sw`: For shared software installation, including Spack environments.
- **Spack Builder VM**: A dedicated `c2d-standard-16` machine configured to dynamically build OpenFOAM (`openfoam-org@8`) and Paraview (`paraview@5.11.2`) using Spack into the shared `/sw` mount.
- **Slurm Controller & Login Node**: Manages the batch scheduling and provides user access points.
- **Slurm Compute Partitions**:
  - `lowcost`: For small-scale testing and preprocessing using smaller VMs.
  - `compute` (default): Autoscaling partition utilizing `c2d-standard-112` VMs for large scale MPI simulations.
- **Chrome Remote Desktop (CRD) VM**: A standalone visual node (`c2d-standard-8`) equipped with the Spack environment injected into `/etc/profile.d/`, ensuring immediate availability of `paraview` and `openfoam` upon login, mounting both `/home` and `/sw`.

## Installed Software
Through automated initialization scripts using `spack`, the cluster automatically installs:
- **GCC** (13.1.0) and other compile tools.
- **OpenMPI** (4.1.3) built with Slurm integration (pmi/legacylaunchers) optimized for AMD AOCC environments.
- **AOCC Compiler** (3.2.0) for optimized numerical compute binaries.
- **OpenFOAM** (`openfoam-org@8`) using Spack, automatically placed in an isolated environment.
- **Paraview** (`paraview@5.11.2` with `+osmesa` target `zen3`) allowing hardware-independent robust 3D rendering.

## Deployment Instructions

1. **Clone the repository and build the Toolkit:**

   ```bash
   git clone https://github.com/GoogleCloudPlatform/cluster-toolkit.git
   cd cluster-toolkit
   make
   ```

2. **Configure Variables:**

   Edit the `openfoam-blueprint.yaml` to set your `project_id`. Alternatively, pass it dynamically during generation.

3. **Generate Deployment and Deploy:**

   ```bash
   ./gcluster create examples/cae/openfoam-blueprint.yaml -w --vars project_id=<your-project-id>
   ./gcluster deploy openfoam-v6
   ```

   Follow the prompts to deploy the components.

4. **Wait for Software Installation:**

   After the cluster comes up, the `spack_builder` will automatically execute Spack compilation in the background. Because Paraview and OpenFOAM are compiled from source, this process will take several hours. You can monitor the progress by SSH'ing into the builder node and tailing `/var/log/spack.log`:

   ```bash
   gcloud compute ssh spack-builder-0 --zone us-central1-a
   tail -f /var/log/spack.log
   ```

## Example Workflow: Running OpenFOAM and Visualizing with Paraview

Once the installation is complete, follow these steps to execute a simulation and visualize it.

### Step 1: Run the OpenFOAM Simulation on Slurm

Log into the Slurm login node:

```bash
gcloud compute ssh openfoam-v6-login-0 --zone us-central1-a
```

Since the environment script loads OpenFOAM automatically, you can immediately prepare a case. For example, using the standard `motorBike` tutorial:

```bash
# Set up a working directory
mkdir -p ~/openfoam_jobs
cd ~/openfoam_jobs

# Copy the OpenFOAM tutorial
cp -r $WM_PROJECT_DIR/tutorials/incompressible/simpleFoam/motorBike .
cd motorBike

# The Allrun script in the tutorial prepares the mesh and runs the solver.
# Let's write a batch script to execute it on the compute nodes using Slurm.
cat << 'EOF' > submit.sh
#!/bin/bash
#SBATCH --job-name=motorbike
#SBATCH --partition=compute
#SBATCH --nodes=1
#SBATCH --ntasks=6

# The environment is already active via /etc/profile.d/
./Allrun
EOF

# Submit the job
sbatch submit.sh
```

Monitor your job using `squeue`. The results and case structure will be saved in `~/openfoam_jobs/motorBike`.

### Step 2: Set up Chrome Remote Desktop

To visualize your data using the CRD VM, you first need to configure a Remote Desktop pin.
1. In your local browser, navigate to [remotedesktop.google.com/headless](https://remotedesktop.google.com/headless).
2. Follow the prompt to authenticate, and copy the "Debian Linux" installation command provided (it looks like `DISPLAY= /opt/google/chrome-remote-desktop/start-host --code="..." --redirect-url="..." --name=$(hostname)`).
3. Connect to the CRD VM via SSH:

   ```bash
   gcloud compute ssh openfoam-v6-chrome-remote-desktop-0 --zone us-central1-a
   ```

4. Paste and execute the command. When prompted, define a 6-digit PIN.

### Step 3: Visualize with Paraview

1. In your local browser, navigate back to [remotedesktop.google.com/access](https://remotedesktop.google.com/access).
2. You will see `openfoam-v6-chrome-remote-desktop-0` in your list of devices. Click it and enter your PIN.
3. You are now presented with a desktop environment running on the cloud. Open a Terminal within the desktop environment.
4. Because the shared `/home` is mounted on this VM and `/etc/profile.d/spack.sh` handles initialization, simply launch Paraview by running:

   ```bash
   paraview
   ```

5. In Paraview, navigate to File -> Open. Since `/home` is shared, navigate to `/home/<your-user>/openfoam_jobs/motorBike`.
6. To load the dataset natively into Paraview, you can create a dummy `.foam` file:

   ```bash
   touch ~/openfoam_jobs/motorBike/motorBike.foam
   ```

   Select `motorBike.foam` in Paraview, click Apply, and you can now interactively analyze the simulation geometry and fluid dynamics.
