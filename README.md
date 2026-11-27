# Medical Imaging: X-ray CT Acquisition, Reconstruction and Analysis

Experiments in Python on X-ray computed tomography (CT), medical image formats, denoising, segmentation, depth estimation and 3D surface extraction. Carried out as part of the project of *Acquisition, Reconstruction & Medical Imaging* (M.Sc. AI for Image Processing and Computer Vision, IPCVai, Université de Bordeaux).

The notebooks implement the methods, measures them against a known reference (a phantom or a clean image) and explains the results.

**Tools:** Python, NumPy, SciPy, scikit-image, OpenCV, PyTorch, pydicom, nibabel, matplotlib

## Contents

| Notebook | Topic |
|---|---|
| `01_xray_ct_projections_and_reconstruction.ipynb` | X-ray projections (Radon transform), filtered back-projection, SART |
| `02_radon_fbp_sart_from_scratch.ipynb` | Radon transform, filtered back-projection and algebraic reconstruction written from scratch |
| `03_dicom_reading_and_denoising.ipynb` | DICOM headers and pixel data, intensity normalisation, denoising |
| `04_mri_windowing_nifti_segmentation.ipynb` | DICOM windowing, 3D NIfTI volumes, segmentation |
| `05_stereo_and_monocular_depth.ipynb` | Depth from stereo matching and from monocular deep networks |
| `06_surface_extraction_marching_cubes.ipynb` | 3D surface meshes from a voxel volume |

## 1. X-ray projections and reconstruction

A CT scanner measures X-ray projections of an object at many angles, and the image is reconstructed from them. Using the Shepp-Logan head phantom (300 × 300 px, where the true image is known):

- Simulated projections with the Radon transform (sinogram over 360°) and checked their physics: every projection keeps the total mass of the object (projection sums 11,088 ± 0.5), and the second half of a 360° scan mirrors the first.
- Studied how the number of projections controls quality with filtered back-projection: SNR rose from −6.5 dB with 8 projections to 19.1 dB with 256. The angular sampling limit for this image size is about 470 projections over 180°.
- Showed why the ramp filter is needed: without it, back-projection blurs the image and its peak value is 87× too high.
- Compared filtered back-projection with SART, an iterative method. SART raised SNR to 22.2 dB after 1 iteration and 25.4 dB after 6, but the contrast-to-noise ratio of a small structure fell from 24.4 to 12.0 and background noise tripled. Better SNR does not always mean a more readable image.

## 2. Reconstruction algorithms from scratch

- **Radon transform.** Profiled a loop-based implementation that was 390× slower than scikit-image (48% of the time in an inner summation), then wrote a vectorised version with inverse mapping and bilinear interpolation. It matches scikit-image to 299 dB SNR at 1.3× its runtime.
- **Filtered back-projection.** Implemented back-projection, ramp and Shepp-Logan filters and normalisation. A half-pixel difference in the rotation centre ((N−1)/2 vs N/2) was the main source of error. After fixing it, the result matches scikit-image's `iradon` to 42 dB SNR.
- **Algebraic reconstruction (ART).** The given version stopped improving at 9.0 dB after 3 iterations. A corrected update rule reached 26.9 dB at iteration 3. With noisy projections, quality peaked at iteration 2 (15.5 dB) and then fell (14.3 dB at iteration 6), so the iteration count acts as regularisation.

| Method | SNR (dB) | SSIM |
|---|---|---|
| Filtered back-projection (ramp) | 16.2 | 0.94 |
| SART, 1 iteration | 18.6 | 0.97 |

## 3. DICOM images and denoising

- Read DICOM headers and pixel data with pydicom: modality, pixel spacing, slice thickness, bit depth, and a check that the patient fields are anonymised. Displayed a 9-slice series (5 mm spacing) and normalised 16-bit intensities to 8 bits.
- Denoised a noisy MRI slice and compared two methods. Total-variation denoising raised PSNR from 18.2 dB to 24.5 dB (SSIM inside the head 0.17 → 0.82), and non-local means reached 24.6 dB. The non-local means setting was chosen from a sweep of 120 parameter sets, and the effect of the stopping rule was measured.

## 4. Windowing, 3D volumes and segmentation

- Applied display windowing from the DICOM header (window centre and width), and showed why the stored window did not fit this image's intensity range.
- Loaded a 3D NIfTI volume with nibabel (160 × 256 × 256 voxels, 1 mm), read its orientation and voxel size from the header, and extracted and normalised slices.
- Segmented the brain on one slice. Thresholding (including Otsu) and morphology could not separate brain from skull and scalp. Watershed segmentation on a Sobel gradient map reached a Dice score of 0.83 against a morphological brain mask. Studied the effect of the compactness and connectivity parameters.

## 5. Depth estimation: stereo and monocular deep networks

- Computed disparity maps from a stereo pair (1920 × 1080) with OpenCV. Semi-global matching (SGBM) gave valid depth for 81% of pixels in 106 ms, against 52% in 38 ms for block matching. The disparity search range was the most important parameter.
- Estimated depth from a single image with two pretrained deep networks, MiDaS and Monodepth2, and compared them with the stereo result. The monocular maps had no holes and a plausible depth order, but blurred edges and some objects placed at the wrong depth.

## 6. 3D surface extraction

- Read a 3D Shepp-Logan volume (64³, PGM3D format) and extracted the boundary surface of each tissue as a triangle mesh, built from the faces between voxels of different values. Checked that all normals point outwards: the volume enclosed by each surface equals its voxel count. Exported the meshes as OBJ files.
- Compared with marching cubes (scikit-image), which gives a smooth surface instead of a staircase. For the outer shell, surface area was 17,190 against 23,456 voxel units.
