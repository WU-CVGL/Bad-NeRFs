<h1 align=center> 😈𝘽𝙖𝙙-𝙍𝙁𝙨: 𝘽undle-𝙖𝙙justed 𝙍adiance 𝙁ield𝙨 from degraded images with continuous-time motion models</h1>

This repo contains:
- An implementation of our ArXiv 2024 paper [**BAD-Gaussians**: Bundle Adjusted Deblur Gaussian Splatting](https://lingzhezhao.github.io/BAD-Gaussians/),
- An accelerated reimplementation of our CVPR 2023 paper [**BAD-NeRF**: Bundle Adjusted Deblur Neural Radiance Fields](https://wangpeng000.github.io/BAD-NeRF/),

based on the [nerfstudio](https://github.com/nerfstudio-project/nerfstudio) framework.

In the future, we will continue to explore *bundle-adjusted radience fields*, add more accelerated implementations
to this repo, such as a reimplementation of our ICLR 2024 paper [**USB-NeRF**: Unrolling Shutter Bundle Adjusted Neural Radiance Fields](https://arxiv.org/abs/2310.02687).

## Demo

Deblurring & novel-view synthesis results on [Deblur-NeRF](https://github.com/limacv/Deblur-NeRF/)'s real-world motion-blurred data:

<video src="https://github.com/WU-CVGL/Bad-RFs/assets/43722188/d0ff1c69-1c7c-4ac2-bcf4-d625f95e06bd"></video>

> Left: BAD-Gaussians deblured novel-view renderings;
>
> Right: Input images.


## Quickstart

### 1. Installation

You may check out the original [`nerfstudio`](https://github.com/nerfstudio-project/nerfstudio) repo for prerequisites and dependencies. 
Currently, our codebase is build on top of the latest version of nerfstudio (v1.0.2),
so if you have an older version of nerfstudio installed,
please `git clone` the main branch and install the latest version.

Besides, we use [pypose](https://github.com/pypose/pypose) to implement the pose interpolation. You can install it with:

```bash
pip install pypose
```

Our `bad-gaussians` currently relies on [our fork of `gsplat`](https://github.com/LingzheZhao/gsplat), you can install it with:

```bash
git clone https://github.com/LingzheZhao/gsplat
cd gsplat
pip install -e .
```

Then you can clone and install this repo as a Python package with:

```bash
git clone https://github.com/WU-CVGL/Bad-RFs
cd Bad-RFs
pip install -e .
```

### 2. Prepare the dataset

#### Deblur-NeRF Synthetic Dataset (Re-rendered)

As described in the BAD-NeRF paper, we re-rendered Deblur-NeRF's synthetic dataset with 51 interpolations per blurry image.

Additionally, in the original BAD-NeRF paper, we directly run COLMAP on blurry images only, with neither ground-truth camera intrinsics nor sharp novel-view images. We find this is quite challenging for COLMAP - it may fail to reconstruct the scene and we need to re-run COLMAP for serval times. To this end, we provided a new set of data, where we ran COLMAP with ground-truth camera intrinsics over both blurry and sharp novel-view images, named `bad-nerf-gtK-colmap-nvs`:

[Download link](https://westlakeu-my.sharepoint.com/:f:/g/personal/cvgl_westlake_edu_cn/EoCe3vaC9V5Fl74DjbGriwcBKj1nbB0HQFSWnVTLX7qT9A)

#### Deblur-NeRF Real Dataset

You can directly download the `real_camera_motion_blur` folder from [Deblur-NeRF](https://limacv.github.io/deblurnerf/).

#### Your Custom Dataset

1. Use the [`ns-process-data` tool from Nerfstudio](https://docs.nerf.studio/reference/cli/ns_process_data.html)
    to process deblur-nerf training images. 

    For example, if the
    [dataset from BAD-NeRF](https://westlakeu-my.sharepoint.com/:f:/g/personal/cvgl_westlake_edu_cn/EsgdW2cRic5JqerhNbTsxtkBqy9m6cbnb2ugYZtvaib3qA?e=bjK7op)
    is in `llff_data`, execute:

    ```
    ns-process-data images \
        --data llff_data/blurtanabata/images \
        --output-dir data/my_data/blurtanabata
    ```

2. Copy the testing images (ground truth sharp images) to the new folder

    ```
    cp llff_data/blurtanabata/images_test data/my_data/blurtanabata/
    ```

3. The folder `data/my_data/blurtanabata` is ready.

> Note1: If you do not have the testing images, e.g. when training with real-world data
> (like those in [Deblur-NeRF](https://limacv.github.io/deblurnerf/)), you can skip the step 2.
>
> Note2: In our `Dataparser`s, since nerfstudio does not model the NDC scene contraction for LLFF data,
> we set `scale_factor = 0.25`, which works well on LLFF datasets.
> If your data is not captured in a LLFF fashion (i.e. forward-facing), such as object-centric like Mip-NeRF 360,
> you can set the `scale_factor = 1.`, 
> e.g., `ns-train bad-gaussians --data data/my_data/my_seq --vis viewer+tensorboard image-restore-data --scale_factor 1`

### 3. Training

#### BAD-Gaussians

For `Deblur-NeRF synthetic` dataset, train with:

```bash
ns-train bad-gaussians \
    --data data/bad-nerf-gtK-colmap-nvs/blurtanabata \
    --vis viewer+tensorboard \
    deblur-nerf-data
```

For `Deblur-NeRF real` dataset with `downscale_factor=4`, train with:
```bash
ns-train bad-gaussians \
    --data data/real_camera_motion_blur/blurdecoration \
    --pipeline.model.camera-optimizer.mode "cubic" \
    --vis viewer+tensorboard \
    deblur-nerf-data \
    --downscale_factor 4
```

For `Deblur-NeRF real` dataset with full resolution, train with:
```bash
ns-train bad-gaussians \
    --data data/real_camera_motion_blur/blurdecoration \
    --pipeline.model.camera-optimizer.mode "cubic" \
    --pipeline.model.camera-optimizer.num_virtual_views 15 \
    --pipeline.model.densify_grad_thresh 2.6667e-4 \
    --pipeline.model.num_downscales 2 \
    --pipeline.model.resolution_schedule 3000 \
    --vis viewer+tensorboard \
    deblur-nerf-data
```

For custom data processed with `ns-process-data`, train with:

```bash
ns-train bad-gaussians \
    --data data/my_data/blurtanabata \
    --vis viewer+tensorboard \
    image-restore-data
```

#### BAD-nerfacto

For `Deblur-NeRF synthetic` dataset and `Deblur-NeRF real` dataset, train with:

```bash
ns-train bad-nerfacto \
    --data data/bad-nerf-gtK-colmap-nvs/blurtanabata \
    --vis viewer+tensorboard \
    deblur-nerf-data
```

```bash
ns-train bad-nerfacto \
    --pipeline.model.camera-optimizer.mode "cubic" \
    --pipeline.model.camera-optimizer.num_virtual_views 15 \
    --data data/real_camera_motion_blur/blurdecoration \
    --vis viewer+tensorboard \
    deblur-nerf-data
```

For custom data processed with `ns-process-data`, train with:

```bash
ns-train bad-nerfacto \
    --data data/my_data/blurtanabata \
    --vis viewer+tensorboard \
    image-restore-data
```

### 4. Render videos

```bash
ns-render interpolate \
  --load-config outputs/tanabata/bad-gaussians/<your_experiment_date_time>/config.yml \
  --render-nearest-camera True \
  --order-poses True \
  --output-path renders/<your_filename>.mp4
```

### 5. Debug with your IDE

Open this repo with your IDE, create a configuration, and set the executing python script path to
`<nerfstudio_path>/nerfstudio/scripts/train.py`, with the parameters above.


## Citation

If you find this useful, please consider citing:

```bibtex
@article{zhao2024badgaussians,
      author    = {Zhao, Lingzhe and Wang, Peng and Liu, Peidong},
      title     = {{BAD-Gaussians: Bundle Adjusted Deblur Gaussian Splatting}},
      journal   = {arXiv preprint arXiv: 2403.xxxxx},
      year      = {2024},
}

@misc{zhao2023badrfs,
    title     = {{Bad-RFs: Bundle-adjusted Radiance Fields from Degraded Images with Continuous-time Motion Models}},
    author    = {Zhao, Lingzhe and Wang, Peng and Liu, Peidong},
    year      = {2023},
    note      = {{https://github.com/WU-CVGL/Bad-RFs}}
}

@InProceedings{wang2023badnerf,
    title     = {{BAD-NeRF: Bundle Adjusted Deblur Neural Radiance Fields}},
    author    = {Wang, Peng and Zhao, Lingzhe and Ma, Ruijie and Liu, Peidong},
    month     = {June},
    year      = {2023},
    booktitle = {Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)},
    pages     = {4170-4179}
}
```

## Acknowledgment

- Kudos to the [Nerfstudio](https://github.com/nerfstudio-project/nerfstudio) team for their amazing framework:

```bibtex
@inproceedings{nerfstudio,
	title        = {Nerfstudio: A Modular Framework for Neural Radiance Field Development},
	author       = {
		Tancik, Matthew and Weber, Ethan and Ng, Evonne and Li, Ruilong and Yi, Brent
		and Kerr, Justin and Wang, Terrance and Kristoffersen, Alexander and Austin,
		Jake and Salahi, Kamyar and Ahuja, Abhik and McAllister, David and Kanazawa,
		Angjoo
	},
	year         = 2023,
	booktitle    = {ACM SIGGRAPH 2023 Conference Proceedings},
	series       = {SIGGRAPH '23}
}
```
