# BIOS 278: Reproducibility and Data Science in Computational Brain Image Analysis
## Lecture 2: Python Environments and Containers

---

## Learning Objectives

By the end of this lecture, students will be able to:
1. Understand what Python environments are and why they matter for reproducibility
2. Identify advantages and disadvantages of virtual environments
3. Explain why containerization is necessary for computational research
4. Describe what containers are and how they work
5. Compare and contrast Docker and Singularity/Apptainer
6. Choose appropriate tools for different reproducibility scenarios

---

## Part 1: Python Environments

### What is a Python Environment?

A Python environment is an isolated space where you can install specific versions of Python and packages without affecting other projects or your system Python installation.

**Analogy:** Think of it like having separate laboratories for different experiments. You wouldn't want chemicals from one experiment contaminating another.

### Why Do We Need Python Environments?

#### The Dependency Problem

Consider this scenario:
- **Project A** (from 2020) requires NumPy 1.18 and scikit-learn 0.22
- **Project B** (current) requires NumPy 1.24 and scikit-learn 1.3
- Your brain imaging pipeline requires specific versions of FSL, ANTs, and NiBabel that were validated together

Without environments, you'd have constant conflicts and couldn't reproduce older analyses.

### Common Python Environment Tools

1. **venv** (built into Python 3.3+)
   - Lightweight, no additional installation needed
   - Creates isolated Python environments

2. **conda/mamba**
   - Manages both Python packages and non-Python dependencies
   - Popular in scientific computing
   - Can install system libraries, R packages, etc.

3. **virtualenv**
   - More features than venv
   - Compatible with older Python versions

4. **pipenv**
   - Combines pip and virtualenv
   - Uses Pipfile for dependency management

### Creating a Python Environment Example

```bash
# Using venv
python -m venv myproject_env
source myproject_env/bin/activate  # On Windows: myproject_env\Scripts\activate
pip install numpy pandas nibabel

# Using conda
conda create -n brain_analysis python=3.9 numpy pandas nibabel
conda activate brain_analysis

# Using mamba (faster conda alternative)
mamba create -n fmri_pipeline python=3.10 nilearn nipype
mamba activate fmri_pipeline
```

### Advantages of Python Environments

1. **Isolation**: Each project has its own dependencies
2. **Version Control**: Pin specific package versions for reproducibility
3. **Easy Cleanup**: Delete environment to remove all packages
4. **Multiple Python Versions**: Run Python 3.8 for one project, 3.11 for another
5. **No Administrator Rights Needed**: Install packages in user space
6. **Shareable**: Export environment specifications (requirements.txt, environment.yml)

### Disadvantages of Python Environments

1. **Not Completely Isolated**: 
   - Still relies on system libraries (CUDA, MKL, system Python)
   - Operating system differences can cause issues
   - Doesn't capture system-level dependencies

2. **Disk Space**: 
   - Each environment duplicates packages
   - Large scientific packages (PyTorch, TensorFlow) take significant space

3. **Reproducibility Limitations**:
   - Python version must be compatible with your OS
   - System libraries may differ between machines
   - Doesn't guarantee bit-for-bit reproducibility

4. **Complexity for Beginners**: 
   - Must remember to activate/deactivate
   - Can have multiple environments and forget which is which

5. **Doesn't Capture Everything**:
   - External tools (FSL, FreeSurfer, ANTs) not included
   - System configuration not preserved
   - Hardware differences (GPUs) not addressed

### Example: When Environments Aren't Enough

Imagine you're analyzing fMRI data with this stack:
- Ubuntu 20.04
- Python 3.8 with NiBabel, NiPype, NiLearn
- FSL 6.0.4
- FreeSurfer 7.1.1
- CUDA 11.2 for GPU acceleration

A Python environment captures only the Python packages. Your collaborator on macOS or Windows can't reproduce your exact setup.

---

## Part 2: Why We Need Containers

### The Reproducibility Crisis in Computational Science

Research has shown that:
- 70% of researchers have failed to reproduce another scientist's experiments
- More than half have failed to reproduce their own experiments
- Software is a major contributor to irreproducibility

### Problems That Environments Don't Solve

1. **Operating System Differences**
   - Library versions differ across Ubuntu, CentOS, macOS, Windows
   - System calls and file paths behave differently

2. **System Dependencies**
   - Neuroimaging tools like FSL require specific system libraries
   - GPU drivers and CUDA versions must match exactly

3. **Configuration Drift**
   - System updates can break previously working code
   - "It worked on my machine" syndrome

4. **Complex Workflows**
   - Brain imaging pipelines often combine Python, shell scripts, and compiled tools
   - Each component has its own dependencies

### The Container Solution

Containers package your entire computational environment:
- Operating system (user space)
- System libraries
- Python/R/other language runtimes
- All software dependencies
- Your code

**Key Insight:** "Package the computer, not just the code"

---

## Part 3: What Are Containers?

### Container Basics

A container is a lightweight, standalone, executable package that includes everything needed to run an application: code, runtime, system tools, libraries, and settings.

### Containers vs. Virtual Machines

| Feature | Container | Virtual Machine |
|---------|-----------|-----------------|
| **Includes** | Application + dependencies | Entire OS + application |
| **Size** | Megabytes to few GB | Gigabytes to tens of GB |
| **Startup Time** | Seconds | Minutes |
| **Performance** | Near-native | Some overhead |
| **Isolation** | Process-level | Hardware-level |
| **Resource Usage** | Lightweight | Heavy |

### How Containers Work

Containers use OS-level virtualization:
1. **Kernel Sharing**: All containers share the host OS kernel
2. **Namespaces**: Isolate process trees, network, users, file systems
3. **Control Groups (cgroups)**: Limit and isolate resource usage
4. **Union File Systems**: Layer file systems efficiently

### Container Images vs. Containers

- **Image**: Read-only template with instructions (like a recipe)
- **Container**: Running instance of an image (like a meal made from the recipe)

You can create multiple containers from one image.

---

## Part 4: Docker

### What is Docker?

Docker is the most popular container platform, developed in 2013. It revolutionized application deployment and is now the standard for containerization.

### Key Docker Concepts

1. **Dockerfile**: Text file with instructions to build an image
2. **Image**: Built from Dockerfile, contains all dependencies
3. **Container**: Running instance of an image
4. **Registry**: Storage for images (Docker Hub, GitHub Container Registry)
5. **Volumes**: Persistent data storage

### Example Dockerfile for Brain Imaging

```dockerfile
# Start from Ubuntu 20.04 base
FROM ubuntu:20.04

# Install system dependencies
RUN apt-get update && apt-get install -y \
    python3.8 \
    python3-pip \
    dcm2niix \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip3 install \
    numpy==1.21.0 \
    nibabel==3.2.1 \
    nilearn==0.8.1 \
    nipype==1.7.0

# Install FSL (simplified)
RUN wget -O- https://fsl.fmrib.ox.ac.uk/fsldownloads/fsl-6.0.4-centos7_64.tar.gz | \
    tar xz -C /opt

# Set up environment variables
ENV FSLDIR=/opt/fsl
ENV PATH=$FSLDIR/bin:$PATH

# Set working directory
WORKDIR /data

# Default command
CMD ["/bin/bash"]
```

### Basic Docker Commands

```bash
# Build an image
docker build -t my-fmri-analysis:v1.0 .

# Run a container
docker run -it my-fmri-analysis:v1.0

# Run with data mounted
docker run -v /home/user/data:/data my-fmri-analysis:v1.0 python analysis.py

# List images
docker images

# List running containers
docker ps

# Remove a container
docker rm container_id
```

### Docker Advantages

1. **Portability**: "Build once, run anywhere"
2. **Reproducibility**: Exact same environment every time
3. **Ecosystem**: Huge community, Docker Hub with pre-built images
4. **Development-Production Parity**: Same environment for development and deployment
5. **Easy Sharing**: Push/pull images from registries
6. **Version Control**: Tag images with versions
7. **Resource Efficiency**: Lighter than VMs

### Docker Disadvantages (for HPC/Research)

1. **Requires Root Access**: Security concern on shared systems
2. **Daemon Required**: Must run Docker daemon with elevated privileges
3. **Not HPC-Friendly**: Most HPC centers don't allow Docker
4. **Security Concerns**: Containers can potentially access host resources
5. **Networking Complexity**: Can be challenging on HPC clusters
6. **GPU Support**: Requires nvidia-docker, additional configuration

---

## Part 5: Singularity (Now Apptainer)

### What is Singularity/Apptainer?

Singularity was created in 2015 specifically for HPC and scientific computing. In 2021, it became part of the Linux Foundation and was renamed Apptainer (though "Singularity" is still widely used).

### Why Singularity for Scientific Computing?

Designed with these HPC requirements in mind:
- **No root/sudo required** for running containers
- **User context preserved** (you are you inside the container)
- **Easy integration** with HPC schedulers (SLURM, PBS)
- **MPI support** for parallel computing
- **GPU support** without extra tools

### Singularity Architecture

Unlike Docker:
- Single-file images (SIF format)
- No daemon process needed
- Runs as regular user
- Integrates with existing authentication

### Creating Singularity Containers

#### From a Definition File

```singularity
Bootstrap: docker
From: ubuntu:20.04

%post
    apt-get update && apt-get install -y \
        python3.8 \
        python3-pip \
        dcm2niix
    
    pip3 install numpy==1.21.0 nibabel==3.2.1 nilearn==0.8.1

%environment
    export LC_ALL=C
    export PATH=/opt/software/bin:$PATH

%runscript
    echo "Running fMRI analysis pipeline"
    exec python3 /data/analysis.py "$@"

%labels
    Author your.name@university.edu
    Version v1.0

%help
    This container includes tools for fMRI preprocessing:
    - Python 3.8
    - NiBabel, NiLearn
    - dcm2niix
```

#### Building Singularity Images

```bash
# Build from definition file (requires root/sudo)
sudo singularity build fmri-analysis.sif fmri-analysis.def

# Build from Docker Hub (no root needed!)
singularity build fmri-analysis.sif docker://neurodebian:latest

# Build remotely (uses Sylabs Cloud)
singularity build --remote fmri-analysis.sif fmri-analysis.def
```

### Running Singularity Containers

```bash
# Execute a command
singularity exec fmri-analysis.sif python3 myscript.py

# Run the default runscript
singularity run fmri-analysis.sif

# Interactive shell
singularity shell fmri-analysis.sif

# With GPU support
singularity exec --nv fmri-analysis.sif python3 gpu_analysis.py

# Bind mount directories
singularity exec --bind /scratch/project:/data fmri-analysis.sif python3 analysis.py

# On HPC with SLURM
srun singularity exec fmri-analysis.sif python3 parallel_analysis.py
```

### Singularity Advantages

1. **HPC-Friendly**: No root needed to run, works on shared systems
2. **Security**: Designed for multi-user environments
3. **Simplicity**: Single-file images, easy to share
4. **Integration**: Works with HPC schedulers, MPI, GPUs
5. **User Permissions**: Maintains your user identity
6. **Docker Compatible**: Can convert Docker images
7. **Immutable**: Images are read-only by default (reproducibility)

### Singularity Disadvantages

1. **Building Requires Root**: Need sudo/root or remote builder for image creation
2. **Smaller Ecosystem**: Fewer pre-built images than Docker
3. **Less Tooling**: Fewer third-party tools and integrations
4. **Learning Curve**: Slightly different concepts from Docker
5. **Limited Windows Support**: Primarily Linux-focused

---

## Part 6: Docker vs. Singularity - Choosing the Right Tool

### Comparison Table

| Aspect | Docker | Singularity/Apptainer |
|--------|--------|----------------------|
| **Target Use Case** | Cloud, web services, microservices | HPC, scientific computing, research |
| **Root Required** | Yes (to run daemon) | Only to build, not to run |
| **Image Format** | Layered (multiple files) | Single SIF file |
| **Daemon** | Required | Not required |
| **User Inside Container** | root by default | Same as host user |
| **File System** | Isolated | Host FS accessible |
| **HPC Clusters** | Usually not allowed | Designed for HPC |
| **GPU Support** | nvidia-docker needed | Built-in (--nv flag) |
| **MPI Support** | Complex | Native support |
| **Image Registries** | Docker Hub, extensive | Sylabs Cloud, limited |
| **Ecosystem** | Huge | Smaller but scientific |

### Decision Guide for Brain Imaging Research

**Use Docker when:**
- Developing on your local machine
- Creating web applications or APIs
- Need the largest ecosystem of pre-built images
- Working with cloud platforms (AWS, Google Cloud)
- Deploying services in production

**Use Singularity when:**
- Running on HPC clusters or supercomputers
- You don't have root access
- Need MPI for parallel neuroimaging workflows
- Working with sensitive data (better security model)
- Need simple, single-file distribution

**Use Both:**
Many researchers develop with Docker (local) and deploy with Singularity (HPC):
1. Create Dockerfile for development
2. Build and test Docker image locally
3. Push to Docker Hub
4. Convert to Singularity on HPC cluster

```bash
# On your local machine
docker build -t myuser/fmri-pipeline:v1.0 .
docker push myuser/fmri-pipeline:v1.0

# On HPC cluster
singularity pull docker://myuser/fmri-pipeline:v1.0
singularity run fmri-pipeline_v1.0.sif
```

---

## Part 7: Best Practices for Reproducibility

### Container Best Practices

1. **Version Everything**
   - Tag images with versions (not just "latest")
   - Pin package versions in Dockerfile/definition file
   - Document build date and environment

2. **Keep Images Small**
   - Use minimal base images (alpine, slim variants)
   - Clean up in same layer: `RUN apt-get update && apt-get install && rm -rf /var/lib/apt/lists/*`
   - Multi-stage builds for compiled software

3. **Document Thoroughly**
   - Include README with container
   - Add labels/metadata
   - Provide example commands

4. **Share Openly**
   - Push images to public registries
   - Include Dockerfile/definition file in repository
   - Use DOIs (Zenodo) for long-term preservation

5. **Test Containers**
   - Include test data and scripts
   - Verify outputs match expected results
   - Test on different systems

### Real-World Example: Neuroimaging Pipeline

```dockerfile
FROM nipreps/fmriprep:23.1.3

# Add custom analysis scripts
COPY analysis/ /opt/analysis/
RUN pip install -r /opt/analysis/requirements.txt

# Add metadata
LABEL maintainer="researcher@university.edu" \
      version="1.0" \
      description="Custom fMRI analysis pipeline built on fMRIPrep"

# Set entrypoint
ENTRYPOINT ["/opt/analysis/run_analysis.sh"]
```

This approach:
- Builds on validated, versioned base (fMRIPrep)
- Adds custom code
- Documents versions
- Provides clear entrypoint

---

## Summary

### Key Takeaways

1. **Python environments** provide basic isolation but have limitations
2. **Containers** solve the "works on my machine" problem completely
3. **Docker** is the industry standard with the largest ecosystem
4. **Singularity/Apptainer** is designed specifically for HPC and scientific computing
5. Both tools are complementary, not competing solutions
6. Containerization is essential for computational reproducibility

### The Reproducibility Hierarchy

1. **Level 0**: Shared code only → Cannot reproduce
2. **Level 1**: Code + requirements.txt → Might reproduce
3. **Level 2**: Code + environment.yml → Better, still depends on OS
4. **Level 3**: Code + container → Highly reproducible
5. **Level 4**: Code + container + data + workflow → Gold standard

### Next Steps for Students

1. Practice creating Python environments for a small project
2. Install Docker/Singularity and run a pre-built neuroimaging container
3. Write a simple Dockerfile for your own analysis
4. Explore NeuroDebian, BrainLife, or other neuroimaging container repositories

---

## Additional Resources

### Documentation
- Docker Docs: https://docs.docker.com
- Singularity/Apptainer Docs: https://apptainer.org/docs
- Neurodocker (generate containers for neuro): https://github.com/ReproNim/neurodocker

### Pre-built Neuroimaging Containers
- NeuroDebian: http://neuro.debian.net
- fMRIPrep: https://fmriprep.org
- MRIQC: https://mriqc.readthedocs.io
- BrainLife: https://brainlife.io

### Research Papers
- Glatard et al. (2015). Reproducibility of neuroimaging analyses across operating systems
- Gronenschild et al. (2012). The effects of FreeSurfer version, workstation type, and macOS version on anatomical volume and cortical thickness measurements

---

## Discussion Questions

1. What aspects of your current research workflow are difficult to reproduce? How could containers help?

2. Why might a Python environment be sufficient for some projects but not others?

3. What are the tradeoffs between flexibility and reproducibility when choosing whether to pin exact package versions?

4. How would you decide between Docker and Singularity for a new brain imaging project?

5. What ethical considerations arise from making computational research reproducible?

---

## Hands-On Exercise

**Exercise**: Create a simple container for a neuroimaging analysis

1. Choose a basic analysis (e.g., loading and plotting a brain image)
2. Write requirements.txt with packages needed
3. Create a Dockerfile that:
   - Uses an appropriate base image
   - Installs your dependencies
   - Copies your analysis script
4. Build and test your container
5. Share the Dockerfile with a classmate and verify they can reproduce your results

**Bonus**: Convert your Docker container to Singularity format and test it!

---

*End of Lecture 2*
