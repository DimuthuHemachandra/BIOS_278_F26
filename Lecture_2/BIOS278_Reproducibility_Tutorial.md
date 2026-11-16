# BIOS 278: From Bad to Good - A Reproducible Neuroimaging Workflow

## Tutorial: Building a Reproducible Brain Image Visualization Script

This tutorial demonstrates the progression from poorly written, non-reproducible code to a fully containerized, reproducible neuroimaging analysis.

---

## Table of Contents

1. [The Bad Script: What Not to Do](#the-bad-script)
2. [Why It's Bad: Reproducibility Issues](#reproducibility-issues)
3. [The Improved Script: Best Practices](#the-improved-script)
4. [BIDS Compatibility](#bids-compatibility)
5. [Python Environments](#python-environments)
6. [Requirements Management](#requirements-management)
7. [Dockerization](#dockerization)
8. [Complete Workflow Example](#complete-workflow)

---

## The Bad Script: What Not to Do

Here's a typical script written by someone new to reproducible research:

**File: `bad_brain_plot.py`**

```python
import nilearn
from nilearn import plotting
import nibabel

# Load the image
img = nibabel.load('/Users/john/Desktop/my_data/scan1.nii.gz')

# Plot it
plotting.plot_anat(img)
plotting.show()

# Save figure
display = plotting.plot_stat_map(img, threshold=3)
display.savefig('output.png')
```

**How to run it (according to the author):**
```bash
python bad_brain_plot.py
```

---

## Reproducibility Issues

Let's identify everything wrong with this script from a reproducibility standpoint:

### Issue 1: Hard-coded Absolute Paths
```python
img = nibabel.load('/Users/john/Desktop/my_data/scan1.nii.gz')
```

**Why it's bad:**
- Only works on John's computer
- Won't work on Windows (path separators)
- Won't work on anyone else's machine (no `/Users/john/`)
- Can't be run on HPC clusters
- Breaks if John reorganizes his Desktop

**Impact:** Script fails immediately on any other system

---

### Issue 2: No Version Information
```python
import nilearn
from nilearn import plotting
import nibabel
```

**Why it's bad:**
- No documentation of which versions were used
- Nilearn 0.9.0 vs 0.10.0 have different APIs
- Results may differ with different versions
- Breaking changes can make script fail entirely

**Impact:** Cannot reproduce exact results months or years later

---

### Issue 3: No Input Validation
```python
img = nibabel.load('/Users/john/Desktop/my_data/scan1.nii.gz')
```

**Why it's bad:**
- No check if file exists
- No verification of image dimensions or type
- Script crashes with cryptic error if file is missing
- No validation of expected data format

**Impact:** Difficult to debug, unclear error messages

---

### Issue 4: Magic Numbers
```python
plotting.plot_stat_map(img, threshold=3)
```

**Why it's bad:**
- What does threshold=3 mean? Z-score? T-stat? Arbitrary units?
- No documentation of why 3 was chosen
- No way to reproduce the decision process
- Hard to modify systematically

**Impact:** Methods not transparent, difficult to replicate analysis logic

---

### Issue 5: No Documentation
- No docstrings
- No comments explaining the purpose
- No usage instructions
- No expected input/output formats

**Impact:** Others cannot understand or use the code

---

### Issue 6: No Error Handling
- No try/except blocks
- No informative error messages
- Fails silently or with confusing errors

**Impact:** Difficult to troubleshoot, poor user experience

---

### Issue 7: Non-descriptive Output Names
```python
display.savefig('output.png')
```

**Why it's bad:**
- `output.png` will be overwritten each run
- No timestamp or identifier
- No connection to input file
- Can't process multiple subjects without manual renaming

**Impact:** Data loss, confusion, manual processes

---

### Issue 8: Missing Metadata
- No logging of:
  - When script was run
  - What parameters were used
  - What versions of software were used
  - What system it ran on

**Impact:** Cannot audit or verify past analyses

---

### Issue 9: Not BIDS Compatible
- Assumes non-standard file organization
- No subject/session structure
- Can't integrate with BIDS ecosystem

**Impact:** Isolated from neuroimaging best practices

---

### Issue 10: No Environment Specification
- No requirements.txt
- No conda environment file
- No documentation of system requirements

**Impact:** "Works on my machine" syndrome

---

## The Improved Script: Best Practices

Now let's see a properly written, reproducible version:

**File: `plot_brain_image.py`**

```python
#!/usr/bin/env python3
"""
Brain Image Visualization Script

This script loads a structural brain MRI image in NIfTI format and creates
anatomical visualizations using nilearn.

Usage:
    python plot_brain_image.py --input <path_to_nifti> --output <output_dir>

Example:
    python plot_brain_image.py --input data/sub-01/anat/sub-01_T1w.nii.gz --output results/

Requirements:
    - Python 3.8+
    - nilearn >= 0.10.0
    - nibabel >= 4.0.0
    - matplotlib >= 3.5.0

Author: Your Name
Date: 2024-11-16
Version: 1.0.0
"""

import argparse
import logging
import sys
from pathlib import Path
from datetime import datetime
import json

import nibabel as nib
import nilearn
from nilearn import plotting
import matplotlib
matplotlib.use('Agg')  # Non-interactive backend for server environments
import matplotlib.pyplot as plt


# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


def validate_input_image(image_path):
    """
    Validate that the input image exists and is a valid NIfTI file.
    
    Parameters
    ----------
    image_path : Path
        Path to the NIfTI image file
        
    Returns
    -------
    nibabel.Nifti1Image
        Loaded NIfTI image
        
    Raises
    ------
    FileNotFoundError
        If image file does not exist
    ValueError
        If image cannot be loaded or has invalid dimensions
    """
    if not image_path.exists():
        raise FileNotFoundError(f"Image file not found: {image_path}")
    
    if not str(image_path).endswith(('.nii', '.nii.gz')):
        raise ValueError(f"File must be NIfTI format (.nii or .nii.gz): {image_path}")
    
    try:
        img = nib.load(str(image_path))
        logger.info(f"Loaded image with shape: {img.shape}")
        logger.info(f"Image affine:\n{img.affine}")
    except Exception as e:
        raise ValueError(f"Failed to load NIfTI image: {e}")
    
    # Validate dimensions (expect 3D or 4D image)
    if len(img.shape) < 3 or len(img.shape) > 4:
        raise ValueError(f"Expected 3D or 4D image, got shape: {img.shape}")
    
    return img


def create_output_directory(output_dir):
    """
    Create output directory if it doesn't exist.
    
    Parameters
    ----------
    output_dir : Path
        Path to output directory
        
    Returns
    -------
    Path
        Validated output directory path
    """
    output_dir = Path(output_dir)
    output_dir.mkdir(parents=True, exist_ok=True)
    logger.info(f"Output directory: {output_dir}")
    return output_dir


def plot_anatomical_image(img, output_path, title=None):
    """
    Create and save anatomical visualization of brain image.
    
    Parameters
    ----------
    img : nibabel.Nifti1Image
        Brain image to visualize
    output_path : Path
        Path where plot will be saved
    title : str, optional
        Title for the plot
    """
    logger.info(f"Creating anatomical plot: {output_path}")
    
    display = plotting.plot_anat(
        img,
        title=title,
        display_mode='ortho',
        cut_coords=None,
        annotate=True,
        draw_cross=True
    )
    
    display.savefig(str(output_path), dpi=300)
    display.close()
    logger.info(f"Saved anatomical plot to: {output_path}")


def plot_slices(img, output_path, title=None):
    """
    Create and save multi-slice visualization.
    
    Parameters
    ----------
    img : nibabel.Nifti1Image
        Brain image to visualize
    output_path : Path
        Path where plot will be saved
    title : str, optional
        Title for the plot
    """
    logger.info(f"Creating slice plot: {output_path}")
    
    display = plotting.plot_anat(
        img,
        title=title,
        display_mode='z',
        cut_coords=7,
        annotate=True
    )
    
    display.savefig(str(output_path), dpi=300)
    display.close()
    logger.info(f"Saved slice plot to: {output_path}")


def save_metadata(output_dir, image_path, img):
    """
    Save processing metadata for reproducibility.
    
    Parameters
    ----------
    output_dir : Path
        Output directory
    image_path : Path
        Input image path
    img : nibabel.Nifti1Image
        Loaded image
    """
    metadata = {
        'timestamp': datetime.now().isoformat(),
        'script_version': '1.0.0',
        'input_file': str(image_path.absolute()),
        'image_shape': list(img.shape),
        'image_dtype': str(img.get_data_dtype()),
        'voxel_size': img.header.get_zooms()[:3],
        'software_versions': {
            'python': sys.version,
            'nibabel': nib.__version__,
            'nilearn': nilearn.__version__,
            'matplotlib': matplotlib.__version__
        }
    }
    
    metadata_path = output_dir / 'processing_metadata.json'
    with open(metadata_path, 'w') as f:
        json.dump(metadata, f, indent=2, default=str)
    
    logger.info(f"Saved metadata to: {metadata_path}")


def parse_arguments():
    """
    Parse command-line arguments.
    
    Returns
    -------
    argparse.Namespace
        Parsed arguments
    """
    parser = argparse.ArgumentParser(
        description='Visualize brain MRI images',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
    # Single subject
    python plot_brain_image.py -i data/sub-01/anat/sub-01_T1w.nii.gz -o results/sub-01/

    # With custom title
    python plot_brain_image.py -i brain.nii.gz -o outputs/ -t "Subject 001 T1w"
        """
    )
    
    parser.add_argument(
        '-i', '--input',
        type=Path,
        required=True,
        help='Path to input NIfTI image (.nii or .nii.gz)'
    )
    
    parser.add_argument(
        '-o', '--output',
        type=Path,
        required=True,
        help='Path to output directory'
    )
    
    parser.add_argument(
        '-t', '--title',
        type=str,
        default=None,
        help='Title for the plots (default: filename)'
    )
    
    parser.add_argument(
        '-v', '--verbose',
        action='store_true',
        help='Enable verbose logging'
    )
    
    return parser.parse_args()


def main():
    """
    Main execution function.
    """
    # Parse arguments
    args = parse_arguments()
    
    # Set logging level
    if args.verbose:
        logger.setLevel(logging.DEBUG)
    
    # Log start
    logger.info("="*60)
    logger.info("Brain Image Visualization Script v1.0.0")
    logger.info("="*60)
    logger.info(f"Input image: {args.input}")
    logger.info(f"Output directory: {args.output}")
    
    try:
        # Validate input
        img = validate_input_image(args.input)
        
        # Create output directory
        output_dir = create_output_directory(args.output)
        
        # Generate title if not provided
        title = args.title if args.title else args.input.stem
        
        # Create base filename from input
        base_name = args.input.stem.replace('.nii', '')
        
        # Generate plots
        plot_anatomical_image(
            img,
            output_dir / f'{base_name}_ortho.png',
            title=f'{title} - Orthogonal View'
        )
        
        plot_slices(
            img,
            output_dir / f'{base_name}_slices.png',
            title=f'{title} - Axial Slices'
        )
        
        # Save metadata
        save_metadata(output_dir, args.input, img)
        
        logger.info("="*60)
        logger.info("Processing completed successfully!")
        logger.info(f"Results saved to: {output_dir}")
        logger.info("="*60)
        
    except Exception as e:
        logger.error(f"Error during processing: {e}", exc_info=True)
        sys.exit(1)


if __name__ == '__main__':
    main()
```

---

## BIDS Compatibility

Now let's create a BIDS-compatible version that works with standard neuroimaging datasets:

**File: `plot_brain_bids.py`**

```python
#!/usr/bin/env python3
"""
BIDS-Compatible Brain Image Visualization

This script processes anatomical MRI images from a BIDS-structured dataset
and creates visualizations for quality control.

BIDS Format Expected:
    dataset/
        sub-01/
            anat/
                sub-01_T1w.nii.gz
                sub-01_T2w.nii.gz
        sub-02/
            anat/
                sub-02_T1w.nii.gz

Usage:
    # Process single subject
    python plot_brain_bids.py --bids-dir /path/to/dataset --subject 01
    
    # Process all subjects
    python plot_brain_bids.py --bids-dir /path/to/dataset --all
    
    # Specific modality
    python plot_brain_bids.py --bids-dir /path/to/dataset --subject 01 --modality T2w

Author: Your Name
Date: 2024-11-16
Version: 1.0.0
"""

import argparse
import logging
from pathlib import Path
import json
from datetime import datetime
import sys

import nibabel as nib
import nilearn
from nilearn import plotting
import matplotlib
matplotlib.use('Agg')


logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class BIDSDataset:
    """Handle BIDS dataset structure and validation."""
    
    def __init__(self, bids_dir):
        """
        Initialize BIDS dataset handler.
        
        Parameters
        ----------
        bids_dir : Path
            Root directory of BIDS dataset
        """
        self.bids_dir = Path(bids_dir)
        self.validate_bids_structure()
    
    def validate_bids_structure(self):
        """Validate that directory follows BIDS structure."""
        if not self.bids_dir.exists():
            raise FileNotFoundError(f"BIDS directory not found: {self.bids_dir}")
        
        # Check for dataset_description.json (BIDS requirement)
        desc_file = self.bids_dir / 'dataset_description.json'
        if not desc_file.exists():
            logger.warning(
                f"No dataset_description.json found. "
                f"This may not be a valid BIDS dataset."
            )
        else:
            with open(desc_file) as f:
                self.dataset_info = json.load(f)
                logger.info(f"Dataset: {self.dataset_info.get('Name', 'Unknown')}")
    
    def get_subjects(self):
        """
        Get list of all subjects in dataset.
        
        Returns
        -------
        list of str
            Subject IDs (without 'sub-' prefix)
        """
        subject_dirs = sorted(self.bids_dir.glob('sub-*'))
        subjects = [d.name.replace('sub-', '') for d in subject_dirs if d.is_dir()]
        logger.info(f"Found {len(subjects)} subjects: {subjects}")
        return subjects
    
    def get_anat_files(self, subject, modality='T1w'):
        """
        Get anatomical files for a subject.
        
        Parameters
        ----------
        subject : str
            Subject ID (without 'sub-' prefix)
        modality : str
            MRI modality (T1w, T2w, etc.)
            
        Returns
        -------
        list of Path
            Paths to anatomical NIfTI files
        """
        subject_dir = self.bids_dir / f'sub-{subject}' / 'anat'
        
        if not subject_dir.exists():
            raise FileNotFoundError(f"No anat directory for sub-{subject}")
        
        # Find files matching pattern
        pattern = f'sub-{subject}_*{modality}.nii.gz'
        files = sorted(subject_dir.glob(pattern))
        
        if not files:
            raise FileNotFoundError(
                f"No {modality} files found for sub-{subject} in {subject_dir}"
            )
        
        return files


def process_subject(bids_dir, subject, modality, output_dir):
    """
    Process anatomical images for one subject.
    
    Parameters
    ----------
    bids_dir : Path
        BIDS dataset directory
    subject : str
        Subject ID
    modality : str
        MRI modality
    output_dir : Path
        Output directory
    """
    logger.info(f"Processing sub-{subject}, modality: {modality}")
    
    # Initialize BIDS dataset
    dataset = BIDSDataset(bids_dir)
    
    # Get anatomical files
    anat_files = dataset.get_anat_files(subject, modality)
    
    # Create subject output directory
    subject_output = output_dir / f'sub-{subject}'
    subject_output.mkdir(parents=True, exist_ok=True)
    
    # Process each anatomical file
    for anat_file in anat_files:
        logger.info(f"Processing: {anat_file.name}")
        
        # Load image
        img = nib.load(str(anat_file))
        
        # Create filename base (BIDS-style)
        base_name = anat_file.stem.replace('.nii', '')
        
        # Generate orthogonal view
        display = plotting.plot_anat(
            img,
            title=f"sub-{subject} {modality}",
            display_mode='ortho',
            annotate=True,
            draw_cross=True
        )
        output_file = subject_output / f'{base_name}_ortho.png'
        display.savefig(str(output_file), dpi=300)
        display.close()
        logger.info(f"Saved: {output_file}")
        
        # Generate slice view
        display = plotting.plot_anat(
            img,
            title=f"sub-{subject} {modality}",
            display_mode='z',
            cut_coords=7,
            annotate=True
        )
        output_file = subject_output / f'{base_name}_slices.png'
        display.savefig(str(output_file), dpi=300)
        display.close()
        logger.info(f"Saved: {output_file}")
    
    # Save processing metadata
    metadata = {
        'timestamp': datetime.now().isoformat(),
        'subject': subject,
        'modality': modality,
        'files_processed': [str(f) for f in anat_files],
        'software_versions': {
            'python': sys.version,
            'nibabel': nib.__version__,
            'nilearn': nilearn.__version__,
        }
    }
    
    metadata_file = subject_output / f'sub-{subject}_{modality}_metadata.json'
    with open(metadata_file, 'w') as f:
        json.dump(metadata, f, indent=2)
    
    logger.info(f"Completed sub-{subject}")


def main():
    """Main execution function."""
    parser = argparse.ArgumentParser(
        description='Process BIDS anatomical MRI data',
        formatter_class=argparse.RawDescriptionHelpFormatter
    )
    
    parser.add_argument(
        '--bids-dir',
        type=Path,
        required=True,
        help='Path to BIDS dataset directory'
    )
    
    parser.add_argument(
        '--subject',
        type=str,
        help='Subject ID to process (without sub- prefix)'
    )
    
    parser.add_argument(
        '--all',
        action='store_true',
        help='Process all subjects in dataset'
    )
    
    parser.add_argument(
        '--modality',
        type=str,
        default='T1w',
        choices=['T1w', 'T2w', 'FLAIR', 'PD'],
        help='MRI modality to process (default: T1w)'
    )
    
    parser.add_argument(
        '--output-dir',
        type=Path,
        default=Path('derivatives/visualizations'),
        help='Output directory (default: derivatives/visualizations)'
    )
    
    args = parser.parse_args()
    
    # Validate arguments
    if not args.subject and not args.all:
        parser.error("Must specify either --subject or --all")
    
    # Initialize dataset
    dataset = BIDSDataset(args.bids_dir)
    
    # Determine subjects to process
    if args.all:
        subjects = dataset.get_subjects()
    else:
        subjects = [args.subject]
    
    # Create output directory
    output_dir = args.bids_dir / args.output_dir
    output_dir.mkdir(parents=True, exist_ok=True)
    
    logger.info(f"Output directory: {output_dir}")
    logger.info(f"Processing {len(subjects)} subject(s)")
    
    # Process each subject
    for subject in subjects:
        try:
            process_subject(args.bids_dir, subject, args.modality, output_dir)
        except Exception as e:
            logger.error(f"Error processing sub-{subject}: {e}", exc_info=True)
            continue
    
    logger.info("All processing complete!")


if __name__ == '__main__':
    main()
```

---

## Python Environments

### Creating a Virtual Environment

**Step 1: Create a new environment**

```bash
# Using venv (built-in)
python3 -m venv brain_imaging_env

# Activate on Linux/Mac
source brain_imaging_env/bin/activate

# Activate on Windows
brain_imaging_env\Scripts\activate
```

**Step 2: Verify activation**

```bash
# Your prompt should change to show (brain_imaging_env)
which python  # Should point to environment's Python
```

**Step 3: Install packages**

```bash
pip install --upgrade pip
pip install nibabel nilearn matplotlib numpy
```

### Using Conda/Mamba

```bash
# Create environment with conda
conda create -n brain_imaging python=3.10

# Activate
conda activate brain_imaging

# Install packages
conda install -c conda-forge nibabel nilearn matplotlib
```

### Using Mamba (faster alternative)

```bash
# Install mamba
conda install -n base -c conda-forge mamba

# Create environment with mamba
mamba create -n brain_imaging python=3.10

# Activate
mamba activate brain_imaging

# Install packages (much faster than conda)
mamba install -c conda-forge nibabel nilearn matplotlib
```

---

## Requirements Management

### Creating requirements.txt

**File: `requirements.txt`**

```txt
# Brain Imaging Visualization Requirements
# Generated: 2024-11-16
# Python version: 3.10

# Core neuroimaging
nibabel==5.2.0
nilearn==0.10.3

# Plotting and visualization
matplotlib==3.8.2

# Scientific computing
numpy==1.26.2
scipy==1.11.4

# Image processing
Pillow==10.1.0

# Optional: for BIDS handling
pybids==0.16.4
```

**Best practices for requirements.txt:**

1. **Pin exact versions** for full reproducibility:
```txt
nibabel==5.2.0  # Exact version
```

2. **Use minimum versions** if you want flexibility:
```txt
nibabel>=5.0.0,<6.0.0  # Any 5.x version
```

3. **Document why** each package is needed:
```txt
nibabel==5.2.0  # NIfTI file I/O
nilearn==0.10.3  # Brain image visualization
```

### Generating requirements.txt

```bash
# From your environment
pip freeze > requirements.txt

# Or manually curated (recommended)
pip list --format=freeze | grep -E 'nibabel|nilearn|matplotlib' > requirements.txt
```

### Installing from requirements.txt

```bash
# In a fresh environment
pip install -r requirements.txt

# Or upgrade existing packages
pip install --upgrade -r requirements.txt
```

---

### Creating environment.yml (Conda)

**File: `environment.yml`**

```yaml
name: brain_imaging
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - pip>=23.0
  
  # Neuroimaging packages
  - nibabel=5.2.0
  - nilearn=0.10.3
  
  # Scientific stack
  - numpy=1.26.2
  - scipy=1.11.4
  - matplotlib=3.8.2
  
  # Optional: BIDS support
  - pip:
      - pybids==0.16.4
```

### Using environment.yml

```bash
# Create environment from file
conda env create -f environment.yml

# Activate
conda activate brain_imaging

# Update existing environment
conda env update -f environment.yml --prune

# Export your current environment
conda env export > environment.yml

# Export without build numbers (more portable)
conda env export --no-builds > environment.yml
```

---

## Project Structure

Here's how to organize your project for maximum reproducibility:

```
brain_imaging_project/
│
├── README.md                 # Project documentation
├── requirements.txt          # Python dependencies (pip)
├── environment.yml           # Conda environment specification
├── Dockerfile               # Docker container definition
│
├── src/                     # Source code
│   ├── plot_brain_image.py
│   └── plot_brain_bids.py
│
├── data/                    # Data directory (add to .gitignore)
│   └── .gitkeep
│
├── results/                 # Output directory (add to .gitignore)
│   └── .gitkeep
│
├── tests/                   # Unit tests
│   └── test_plotting.py
│
├── docs/                    # Additional documentation
│   └── usage_examples.md
│
└── .gitignore              # Git ignore file
```

**File: `.gitignore`**

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
brain_imaging_env/
.env

# Data (too large for git)
data/
*.nii
*.nii.gz

# Results
results/
*.png
*.jpg
*.pdf

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

**File: `README.md`**

```markdown
# Brain Imaging Visualization

Reproducible brain MRI visualization pipeline.

## Requirements

- Python 3.10+
- See `requirements.txt` for Python packages

## Installation

### Option 1: Using venv

```bash
python3 -m venv brain_imaging_env
source brain_imaging_env/bin/activate
pip install -r requirements.txt
```

### Option 2: Using conda

```bash
conda env create -f environment.yml
conda activate brain_imaging
```

### Option 3: Using Docker

```bash
docker build -t brain-imaging:v1.0 .
docker run -v $(pwd)/data:/data brain-imaging:v1.0
```

## Usage

### Single image

```bash
python src/plot_brain_image.py \
    --input data/brain.nii.gz \
    --output results/
```

### BIDS dataset

```bash
python src/plot_brain_bids.py \
    --bids-dir data/bids_dataset \
    --subject 01 \
    --modality T1w
```

## Output

- Orthogonal view: `*_ortho.png`
- Axial slices: `*_slices.png`
- Metadata: `processing_metadata.json`

## Citation

If you use this code, please cite: [Your paper/DOI]

## License

MIT License
```

---

## Dockerization

Now let's containerize our script for complete reproducibility.

### Dockerfile

**File: `Dockerfile`**

```dockerfile
# Start from official Python image
FROM python:3.10-slim

# Set metadata labels
LABEL maintainer="your.email@university.edu" \
      version="1.0.0" \
      description="Brain imaging visualization container"

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements first (for better caching)
COPY requirements.txt .

# Install Python packages
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/

# Create directories for data and results
RUN mkdir -p /data /results

# Set working directory to /data
WORKDIR /data

# Add script to PATH
ENV PATH="/app/src:${PATH}"

# Make scripts executable
RUN chmod +x /app/src/*.py

# Default command - show help
CMD ["python", "/app/src/plot_brain_image.py", "--help"]
```

### Building the Docker Image

```bash
# Build the image
docker build -t brain-imaging:v1.0 .

# Build with specific tag
docker build -t myusername/brain-imaging:1.0.0 .

# View images
docker images
```

### Running the Docker Container

```bash
# Run with help message
docker run brain-imaging:v1.0

# Process a single file
docker run -v $(pwd)/data:/data -v $(pwd)/results:/results \
    brain-imaging:v1.0 \
    python /app/src/plot_brain_image.py \
    --input /data/brain.nii.gz \
    --output /results/

# Process BIDS dataset
docker run -v $(pwd)/bids_data:/data -v $(pwd)/results:/results \
    brain-imaging:v1.0 \
    python /app/src/plot_brain_bids.py \
    --bids-dir /data \
    --subject 01 \
    --output-dir /results

# Interactive shell in container
docker run -it -v $(pwd)/data:/data brain-imaging:v1.0 /bin/bash
```

### Advanced Dockerfile (Production-Ready)

**File: `Dockerfile.advanced`**

```dockerfile
# Multi-stage build for smaller image size
FROM python:3.10-slim as builder

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install Python packages
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt


# Final stage - minimal image
FROM python:3.10-slim

# Metadata
LABEL maintainer="your.email@university.edu" \
      version="1.0.0" \
      description="Brain imaging visualization" \
      org.opencontainers.image.source="https://github.com/yourusername/brain-imaging"

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv

# Set environment variables
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Create non-root user for security
RUN useradd -m -u 1000 -s /bin/bash brainuser && \
    mkdir -p /app /data /results && \
    chown -R brainuser:brainuser /app /data /results

# Copy application
COPY --chown=brainuser:brainuser src/ /app/src/
RUN chmod +x /app/src/*.py

# Switch to non-root user
USER brainuser

# Set working directory
WORKDIR /data

# Add to PATH
ENV PATH="/app/src:${PATH}"

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import nilearn; import nibabel" || exit 1

# Default command
ENTRYPOINT ["python", "/app/src/plot_brain_image.py"]
CMD ["--help"]
```

### Docker Compose for Complex Workflows

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  brain-imaging:
    build:
      context: .
      dockerfile: Dockerfile
    image: brain-imaging:v1.0
    container_name: brain_analysis
    volumes:
      - ./data:/data:ro  # Read-only data
      - ./results:/results:rw  # Read-write results
    environment:
      - SUBJECT_ID=01
      - MODALITY=T1w
    command: >
      python /app/src/plot_brain_bids.py
      --bids-dir /data
      --subject ${SUBJECT_ID}
      --modality ${MODALITY}
      --output-dir /results
```

**Running with Docker Compose:**

```bash
# Run the analysis
docker-compose up

# Run in background
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

---

## Converting Docker to Singularity

For HPC environments, convert your Docker image to Singularity:

### Method 1: Direct Conversion

```bash
# Pull from Docker Hub and convert
singularity pull docker://myusername/brain-imaging:1.0.0

# This creates: brain-imaging_1.0.0.sif
```

### Method 2: Build from Docker Image

```bash
# Build from local Docker daemon
singularity build brain-imaging.sif docker-daemon://brain-imaging:v1.0

# Build from Docker Hub
singularity build brain-imaging.sif docker://myusername/brain-imaging:1.0.0
```

### Running Singularity Container

```bash
# Execute command
singularity exec brain-imaging.sif \
    python /app/src/plot_brain_image.py \
    --input /data/brain.nii.gz \
    --output /results/

# With bind mounts
singularity exec --bind $(pwd)/data:/data,$(pwd)/results:/results \
    brain-imaging.sif \
    python /app/src/plot_brain_image.py \
    --input /data/brain.nii.gz \
    --output /results/

# Interactive shell
singularity shell brain-imaging.sif

# On HPC with SLURM
#!/bin/bash
#SBATCH --job-name=brain_viz
#SBATCH --time=01:00:00
#SBATCH --mem=8GB

singularity exec brain-imaging.sif \
    python /app/src/plot_brain_bids.py \
    --bids-dir /scratch/data \
    --subject ${SLURM_ARRAY_TASK_ID} \
    --output-dir /scratch/results
```

---

## Complete Workflow Example

Let's put it all together with a real-world example.

### Directory Structure

```
project/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── environment.yml
├── README.md
├── .gitignore
│
├── src/
│   ├── plot_brain_image.py
│   └── plot_brain_bids.py
│
├── data/                    # Your BIDS dataset
│   ├── dataset_description.json
│   ├── sub-01/
│   │   └── anat/
│   │       └── sub-01_T1w.nii.gz
│   └── sub-02/
│       └── anat/
│           └── sub-02_T1w.nii.gz
│
└── results/                 # Will be created
```

### Workflow 1: Local Development with venv

```bash
# Step 1: Clone repository
git clone https://github.com/yourusername/brain-imaging.git
cd brain-imaging

# Step 2: Create environment
python3 -m venv venv
source venv/bin/activate

# Step 3: Install dependencies
pip install -r requirements.txt

# Step 4: Run analysis
python src/plot_brain_bids.py \
    --bids-dir data/ \
    --subject 01 \
    --output-dir results/

# Step 5: Deactivate when done
deactivate
```

### Workflow 2: Using Docker

```bash
# Step 1: Build image
docker build -t brain-imaging:v1.0 .

# Step 2: Run analysis
docker run \
    -v $(pwd)/data:/data:ro \
    -v $(pwd)/results:/results:rw \
    brain-imaging:v1.0 \
    python /app/src/plot_brain_bids.py \
    --bids-dir /data \
    --subject 01 \
    --output-dir /results

# Step 3: View results
ls results/sub-01/
```

### Workflow 3: HPC with Singularity

```bash
# Step 1: Convert Docker to Singularity (on your machine)
docker build -t brain-imaging:v1.0 .
docker save brain-imaging:v1.0 | gzip > brain-imaging.tar.gz
scp brain-imaging.tar.gz hpc-cluster:~/

# Step 2: On HPC cluster
ssh hpc-cluster
singularity build brain-imaging.sif docker-archive://brain-imaging.tar.gz

# Step 3: Create SLURM job script
cat > run_analysis.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=brain_viz
#SBATCH --array=1-10
#SBATCH --time=01:00:00
#SBATCH --mem=8GB
#SBATCH --cpus-per-task=1

# Format subject ID with zero-padding
SUBJECT=$(printf "%02d" ${SLURM_ARRAY_TASK_ID})

singularity exec \
    --bind /scratch/data:/data \
    --bind /scratch/results:/results \
    brain-imaging.sif \
    python /app/src/plot_brain_bids.py \
    --bids-dir /data \
    --subject ${SUBJECT} \
    --output-dir /results
EOF

# Step 4: Submit job
sbatch run_analysis.sh

# Step 5: Monitor jobs
squeue -u $USER

# Step 6: Check results
ls /scratch/results/sub-*/
```

---

## Testing Your Container

Create a test script to verify everything works:

**File: `test_container.sh`**

```bash
#!/bin/bash
# Test script for brain imaging container

set -e  # Exit on error

echo "Testing brain imaging container..."

# Create test data directory
mkdir -p test_data test_results

# Download sample MNI template (small test file)
echo "Downloading test data..."
wget -q -O test_data/MNI152_T1_2mm_brain.nii.gz \
    https://www.bic.mni.mcgill.ca/~vfonov/icbm/2009/mni_icbm152_nlin_asym_09c_nifti.zip

# Test Docker container
echo "Testing Docker container..."
docker run \
    -v $(pwd)/test_data:/data \
    -v $(pwd)/test_results:/results \
    brain-imaging:v1.0 \
    python /app/src/plot_brain_image.py \
    --input /data/MNI152_T1_2mm_brain.nii.gz \
    --output /results

# Verify output files exist
if [ -f "test_results/MNI152_T1_2mm_brain_ortho.png" ]; then
    echo "✓ Docker test passed!"
else
    echo "✗ Docker test failed!"
    exit 1
fi

# Clean up
echo "Cleaning up..."
rm -rf test_data test_results

echo "All tests passed!"
```

**Run tests:**

```bash
chmod +x test_container.sh
./test_container.sh
```

---

## Publishing and Sharing

### Publishing to Docker Hub

```bash
# Step 1: Tag your image
docker tag brain-imaging:v1.0 yourusername/brain-imaging:1.0.0
docker tag brain-imaging:v1.0 yourusername/brain-imaging:latest

# Step 2: Login to Docker Hub
docker login

# Step 3: Push to Docker Hub
docker push yourusername/brain-imaging:1.0.0
docker push yourusername/brain-imaging:latest

# Step 4: Others can now pull and use
docker pull yourusername/brain-imaging:1.0.0
```

### Publishing to GitHub Container Registry

```bash
# Step 1: Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u yourusername --password-stdin

# Step 2: Tag for GitHub
docker tag brain-imaging:v1.0 ghcr.io/yourusername/brain-imaging:1.0.0

# Step 3: Push
docker push ghcr.io/yourusername/brain-imaging:1.0.0
```

### Creating a Zenodo DOI

1. Push code to GitHub
2. Create a release (v1.0.0)
3. Link GitHub repository to Zenodo
4. Zenodo automatically creates DOI for each release
5. Cite your containerized pipeline with DOI

---

## Summary Checklist

### For Reproducible Code:

- [ ] Use command-line arguments (not hard-coded paths)
- [ ] Add comprehensive error handling
- [ ] Include logging and metadata
- [ ] Document all parameters and functions
- [ ] Validate inputs
- [ ] Use BIDS-compatible structure
- [ ] Include version information

### For Python Environment:

- [ ] Create isolated virtual environment
- [ ] Pin exact package versions in requirements.txt
- [ ] Document Python version requirement
- [ ] Test installation in fresh environment

### For Containerization:

- [ ] Create Dockerfile with pinned base image
- [ ] Document container build process
- [ ] Include all dependencies
- [ ] Test container locally
- [ ] Push to registry with version tags
- [ ] Provide usage examples
- [ ] Create Singularity version for HPC

### For Sharing:

- [ ] Comprehensive README
- [ ] Clear license (MIT, GPL, etc.)
- [ ] Example data or test script
- [ ] Citation information
- [ ] GitHub repository
- [ ] DOI from Zenodo
- [ ] Publication/preprint link

---

## Comparison: Before and After

| Aspect | Bad Script | Good Script + Container |
|--------|-----------|------------------------|
| **Portability** | Only works on author's machine | Works anywhere Docker/Singularity runs |
| **Reproducibility** | Impossible to reproduce exact results | Bit-for-bit reproducible |
| **Documentation** | None | Comprehensive docs + examples |
| **Maintenance** | Breaks with updates | Frozen, stable environment |
| **Sharing** | Email script, pray it works | Pull container, run immediately |
| **Version Control** | Unknown software versions | All versions documented |
| **Testing** | No way to verify | Can test on multiple systems |
| **HPC Compatibility** | Manual setup required | Single Singularity file |
| **Data Organization** | Chaotic | BIDS-compliant |
| **Long-term Usability** | Unusable after 6 months | Usable for years/decades |

---

## Additional Resources

### Documentation
- BIDS Specification: https://bids-specification.readthedocs.io
- Nilearn Documentation: https://nilearn.github.io
- Docker Best Practices: https://docs.docker.com/develop/dev-best-practices/
- Singularity User Guide: https://sylabs.io/docs/

### Tools
- BIDS Validator: https://bids-standard.github.io/bids-validator/
- Neurodocker: Generate Dockerfiles for neuroimaging
- ReproNim: Reproducible neuroimaging toolkit
- DataLad: Version control for data

### Example Repositories
- fMRIPrep: https://github.com/nipreps/fmriprep
- MRIQC: https://github.com/nipreps/mriqc
- Neurostars: Community forum for help

---

*End of Tutorial*
