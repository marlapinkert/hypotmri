# Functional pipeline 
First make sure you have the correct project activated 
```bash
source set_project.sh <project-name>
```

## [1] Susceptibility distortion correction - (afni)
Runs susceptibility distortion correction using AFNI. This corrects for the stretching/squishing of BOLD images that occurs along the phase-encoding axis. AFNI does this by comparing an image acquired in one phase-encoding direction with another which was acquired in the opposite direction. 

We call this command - specifying the subject, session and task to run through, and where to put the outputs. Note - you can also specify to run with the AFNI docker, by calling --afni-docker.
### Options
``` 
--bids-dir            BIDS root directory
--output-file         Output derivatives file
--sub                 Subject label (e.g. sub-01)
--ses                 Session label (default='ses-01')
--task                Task label.  Omit to process all tasks. (default=None)
--afni-docker         AFNI Docker image tag, or "local" to run on host (default=os.environ.get('AFNI_IMAGE', 'afni/afni_make_build:latest')
--overwrite           Force re-run for named step(s)
--overwrite-all       Force re-run for all steps
``` 

### Examples
#### Run SDC locally 
```bash
# Activate correct python env
$PYPACKAGE_MANAGER activate preproc
# Run SDC (using afni docker)
s01_sdc_AFNI.py --bids-dir $BIDS_DIR \
    --sub <sub-id> --ses <ses-id> \
    --task <task-id> \
    --output-file s1_AFNI_sdc \ 
    --afni-docker $AFNI_IMAGE
```
#### Run SDC on the HPC
```bash
s01_sdc_hpc.sh --bids-dir $BIDS_DIR \
    --sub <sub-id> --ses <ses-id> \
    --task <task-id> \
    --output-file s1_AFNI_sdc 

```


## [2] Motion correction (fsl) and [3] Coregistration
Next, we can run motion correction for a whole session using fsl. Motion correction and coregistration use the FreeSurfer T1 and surface projection, so freesurfer must have already been run before this step.

### Coregistration strategy
1) Select first sbref as "master" - coregister to anatomy with bbregister
2) Coregister each sbref_i to sbref_master   (FLIRT, normcorr, DOF 6)
3) MCFLIRT per run, referencing the corresponding sbref_i
4) Concatenate transforms:  VOL -> sbref_i -> sbref_master -> FS_T1

If no sbref image is available, the script automatically chooses the middle volume of each run as a synthetic sbref.

### Environment: preproc 

Make sure to activate this environment before running the script!

```$PYPACKAGE_MANAGER activate preproc```

### Options:
```
--bids-dir           BIDS directory
--input-file         Input directory containing SDC-corrected BOLD + SBREF files
--output-file        Output derivatives file
--sub                Subject label (e.g. sub-01)
--ses                Session label (default: ses-01)
--subjects-dir       FreeSurfer SUBJECTS_DIR (default: $SUBJECTS_DIR)
--docker             Docker image for FSL/FreeSurfer tools, or "local" (default=os.environ.get('FSL_FREESURFER_IMAGE', 'local'))
--task               Task label to process (e.g. rest). Omit to process all tasks. (default=None)
--run                Run label to process (e.g. 01 or run-01). Omit to process all runs. (default=None)
```


### Examples
Locally:
```bash
$PYPACKAGE_MANAGER activate preproc
s02_coreg.py --sub <sub-id> --ses <ses-id> \
    --input-file s1_AFNI_sdc \
    --output-file s2_coreg \
    --bids-dir $BIDS_DIR \
    --docker local
```
On the cluster:
```bash
s02_coreg_hpc.sh --sub <sub-id> --ses <ses-id> \
    --input-file s1_AFNI_sdc \
    --output-file s2_coreg \
    --bids-dir $BIDS_DIR
```
You can also submit runs & tasks separately - if you want to break it up into smaller stages 
```bash
# Local
s02_coreg.py --sub <sub-id> --ses <ses-id> \
    --input-file s1_AFNI_sdc \
    --output-file s2_coreg \
    --bids-dir $BIDS_DIR \
    --task pRFLE --run 1

# Cluster
# Arguments after " -- " are passed to s02_coreg.py
s02_coreg_hpc.sh --sub hp01 --ses 01 \
    --input-file s1_AFNI_sdc \
    --output-file s2_coreg \
    --bids-dir $BIDS_DIR \
    -- --task pRFLE --run 1


# you can loop over a submmission job like this
for TASK in "CSFLE" "CSFRE" "pRFLE" "pRFRE"; do
    for run in "01" "02" "03"; do
        s02_coreg_hpc.sh --sub hp01 --ses 01 \
            --input-file s1_AFNI_sdc \
            --output-file s2_coreg \
            --bids-dir $BIDS_DIR \
            --skip-sync \
            -- --task $TASK --run $run
    done
done
```

## [3] fmriprep (for confounds)
Locally 
```bash
$PYPACKAGE_MANAGER activate preproc
s03_fmriprep_func.sh --sub hp01 --ses 01 \
    --input-file s2_coreg --bids-dir $BIDS_DIR
```
On the cluster
```bash
s03_fmriprep_func_hpc.sh --sub hp01 --ses 01 \
    --input-file s2_coreg --bids-dir $BIDS_DIR
```

## [4] confounds 
- Grabs the confounds & bold-brainmasks from fmriprep
- Removes the motion related ones 
- Replace them with our own, FSL derived motion confounds (from moco file)
- Save as .tsv file 
- Run PCA on select components of confound file, apply sg-filter, save as .tsv 
- Volume data: Apply sg-filter & regress out PCA noise components
- Surface data: Apply sg-filter & regress out PCA noise components

Locally: 
```bash
s04_confounds.py --sub hp01 --ses 01 \
    --bids-dir $BIDS_DIR \
    --moco-file s2_coreg \
    --output-file s4_denoised
```

On the cluster:

```
s04_confounds_hpc.sh --sub hp01 --ses 01 \
    --bids-dir $BIDS_DIR \
    --moco-file s2_coreg \
    --output-file s4_denoised
```