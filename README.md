# Simplify re-implementation of 3DGS 

## 1.1 3D Gaussian Rasterization (35 points)

In this section, we will implement a 3D Gaussian rasterization pipeline in PyTorch. The official implementation uses custom CUDA code and several optimizations to make the rendering very fast. For simplicity, our implementation avoids many of the tricks and optimizations used by the official implementation and hence would be much slower. Additionally, instead of using all the spherical harmonic coefficients to model view dependent effects, we will only use the view independent compoenents.

Inspite of these limitations, once you complete this section successfully, you will find that our simplified 3D Gaussian rasterizer can still produce renderings of pre-trained 3D Gaussians that were trained using the original codebase reaonsably well!

For this section, you will have to complete the code in the files `model.py` and `render.py`. The file `model.py` contains code that manipulates and renders Gaussians. The file `render.py` uses functionality from `model.py` to render a pre-trained 3D Gaussian representation of an object.

In sections `1.1.1` to `1.1.4`, you will have to complete the code in the classes `Gaussians` and `Scene` in the file `model.py`. In section `1.1.5`, you will have to complete the code in the file `render.py`. It might be helpful to first skim both the files before starting the assignment to get a rough idea of the workflow.

> **Note**: All the functions in `model.py` perform operations on a batch of N Gaussians instead of 1 Gaussian at a time. As such, it is recommended to write loopless vectorized code to maximize performance. However, a solution with loops will also be accepted as long as it works and produces the desired output.

### 1.1.1 Project 3D Gaussians to Obtain 2D Gaussians

We will begin our implementation of a 3D Gaussian rasterizer by first creating functionality to project 3D Gaussians in the world space to 2D Gaussians that lie on the image plane of a camera. 

A 3D Gaussian is parameterized by its mean (a 3 dimensional vector) and covariance (a 3x3 matrix). Following equations (5) and (6) of the [original paper](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf), we can obtain a 2D Gaussian (parameterized by a 2D mean vector and 2x2 covariance matrix) that represents an approximation of the projection of a 3D Gaussian to the image plane of a camera.

For this section, you will need to complete the code in the functions `compute_cov_3D`, `compute_cov_2D` and `compute_means_2D` of the class `Gaussians`. 

### 1.1.2 Evaluate 2D Gaussians

In the previous section, we had implemented code to project 3D Gaussians to obtain 2D Gaussians. Now, we will write code to evaluate the 2D Gaussian at a particular 2D pixel location.

A 2D Gaussian is represented by the following expression:

$$
  f(\mathbf{x}; \boldsymbol{\mu}\_{i}, \boldsymbol{\Sigma}\_{i}) = \frac{1}{2 \pi \sqrt{ | \boldsymbol{\Sigma}\_{i} |}} \exp \left ( {-\frac{1}{2}} (\mathbf{x} - \boldsymbol{\mu}\_{i})^T \boldsymbol{\Sigma}\_{i}^{-1} (\mathbf{x} - \boldsymbol{\mu}\_{i}) \right ) = \frac{1}{2 \pi \sqrt{ | \boldsymbol{\Sigma}\_{i} |}} \exp \left ( P_{(\mathbf{x}, i)} \right )
$$

Here, $\mathbf{x}$ is a 2D vector that represents the pixel location, $\boldsymbol{\mu}$ represents is a 2D vector representing the mean of the $i$-th 2D Gaussian, and $\boldsymbol{\Sigma}$ represents the covariance of the 2D Gaussian. The exponent part $P_{(\mathbf{x}, i)}$ is referred to as **power** in the code.

$$
  P_{(\mathbf{x}, i)} = {-\frac{1}{2}} (\mathbf{x} - \boldsymbol{\mu}\_{i})^T \mathbf{\Sigma}\_{i}^{-1} (\mathbf{x} - \boldsymbol{\mu}\_{i})
$$

The function `evaluate_gaussian_2D` of the class `Gaussians` is used to compute the power. In this section, you will have to complete this function.

**Unit Test**: To check if your implementation is correct so far, we have provided a unit test. Run `python unit_test_gaussians.py` to see if you pass all 4 test cases.

### 1.1.3 Filter and Sort Gaussians

Now that we have implemented functionality to project 3D Gaussians, we can start implementing the rasterizer!

Before starting the rasterization procedure, we should first sort the 3D Gaussians in increasing order by their depth value. We should also discard 3D Gaussians whose depth value is less than 0 (we only want to project 3D Gaussians that lie in front on the image plane).

Complete the functions `compute_depth_values` and `get_idxs_to_filter_and_sort` of the class `Scene` in `model.py`. You can refer to the function `render` in class `Scene` to see how these functions will be used.

### 1.1.4 Compute Alphas and Transmittance

Using these `N` ordered and filtered 2D Gaussians, we can compute their alpha and transmittance values at each pixel location in an image.

The alpha value of a 2D Gaussian $i$ at a single pixel location $\mathbf{x}$ can be calculated using:

$$
  \alpha_{(\mathbf{x}, i)} = o_i \exp(P_{(\mathbf{x}, i)})
$$

Here, $o_i$ is the opacity of each Gaussian, which is a learnable parameter.

Given `N` ordered 2D Gaussians, the transmittance value of a 2D Gaussian $i$ at a single pixel location $\mathbf{x}$ can be calculated using:

$$
  T_{(\mathbf{x}, i)} = \prod_{j \lt i} (1 - \alpha_{(\mathbf{x}, j)})
$$

In this section, you will need to complete the functions `compute_alphas` and `compute_transmittance` of the class `Scene` in `model.py` so that alpha and transmittance values can be computed.

> Note: In practice, when `N` is large and when the image dimensions are large, we may not be able to compute all alphas and transmittance in one shot since the intermediate values may not fit within GPU memory limits. In such a scenario, it might be beneficial to compute the alphas and transmittance in **mini-batches**. In our codebase, we provide the user the option to perform splatting `num_mini_batches` times, where we splat `K` Gaussians at a time (except at the last iteration, where we could possibly splat less than `K` Gaussians). Please refer to the functions `splat` and `render` of class `Scene` in `model.py` to see how splatting and mini-batching is performed.

### 1.1.5 Perform Splatting

Finally, using the computed alpha and transmittance values, we can blend the colour value of each 2D Gaussian to compute the colour at each pixel. The equation for computing the colour of a single pixel is (which is the same as equation (3) from the [original paper](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf)).

More formally, given `N` ordered 2D Gaussians, we can compute the colour value at a single pixel location $\mathbf{x}$ by:

$$
  C_{\mathbf{x}} = \sum_{i = 1}^{N} c_i \alpha_{(\mathbf{x}, i)} T_{(\mathbf{x}, i)}
$$

Here, $c_i$ is the colour contribution of each Gaussian, which is a learnable parameter. Instead of using the colour contribution of each Gaussian, we can also use other attributes to compute the depth and silhouette mask at each pixel as well!

In this section, you will need to complete the function `splat` of the class `Scene` in `model.py` and return the colour, depth and silhouette (mask) maps. While the equation for colour is given in this section, you will have to think and implement similar equations for computing the depth and silhouette (mask) maps as well. You can refer to the function `render` in the same class to see how the function `splat` will be used.

Once you have finished implementing the functions, you can open the file `render.py` and complete the rendering code in the function `create_renders` (this task is very simple, you just have to call the `render` function of the object of the class `Scene`)

After completing `render.py`, you can test the rendering code by running `python render.py`. This script will take a few minutes to render views of a scene represented by pre-trained 3D Gaussians!

For reference, here is one frame of the GIF that you can expect to see:

<p align="center">
  <img src="ref_output/q1_render_example.png" alt="Q1_Render" width=60%/>
</p>

Do note that while the reference we have provided is a still frame, we expect you to submit the GIF that is output by the rendering code.

**GPU Memory Usage**: This task (with default paramter settings) may use approximately 6GB GPU memory. You can decrease/increase GPU memory utilization for performance by using the `--gaussians_per_splat` argument.

> **Submission:** In your webpage, attach the **GIF** that you obtained by running `render.py`

## 1.2 Training 3D Gaussian Representations (15 points)

Now, we will use our 3D Gaussian rasterizer to train a 3D representation of a scene given posed multi-view data. 

More specifically, we will train a 3D representation of a toy cow given multi-view data and a point cloud. The folder `./data/cow_dataset` contains images, poses and a point cloud of a toy cow. The point cloud is used to initialize the means of the 3D Gaussians. 

In this section, for ease of implementation and because the scene is simple, we will perform training using **isotropic** Gaussians. Do recall that you had already implemented all the necessary functionality for this in the previous section! In the training code, we just simply set `isotropic` to True while initializing `Gaussians` so that we deal with isotropic Gaussians.

For all questions in this section, you will have to complete the code in `train.py`

### 1.2.1 Setting Up Parameters and Optimizer

First, we must make our 3D Gaussian parameters trainable. You can do this by setting `requires_grad` to True on all necessary parameters in the function `make_trainable` in `train.py` (you will have to implement this function). 

Next, you will have to setup the optimizer. It is recommended to provide different learning rates for each type of parameter (for example, it might be preferable to use a much smaller learning rate for the means as compared to opacities or colours). You can refer to pytorch documentation on how to set different learning rates for different sets of parameters. 

Your task is to complete the function `setup_optimizer` in `train.py` by passing all trainable parameters and setting appropriate learning rates. Feel free to experiment with different settings of learning rates.

### 1.2.2 Perform Forward Pass and Compute Loss

We are almost ready to start training. All that is left is to complete the function `run_training`. Here, you are required to call the relevant function to render the 3D Gaussians to predict an image rendering viewed from a given camera. Also, you are required to implement a loss function that compared the predicted image rendering to the ground truth image. Standard L1 loss should work fine for this question, but you are free to experiment with other loss functions as well.

Finally, we can now start training. You can do so by running `python train.py`. This script would save two GIFs (`q1_training_progress.gif` and `q1_training_final_renders.gif`). 

For reference, here is one frame from the training progress GIF from our reference implementation. The top row displays renderings obtained from Gaussians that are being trained and the bottom row displayes the ground truth. The top row looks good in this reference because this frame is from near the end of the optimization procedure. You can expect the top row to look bad during the start of the optimization procedure.

<p align="center">
  <img src="ref_output/q1_training_example_1.png" alt="Q1_Training_1" width=60%/>
</p>

Also, for reference, here is one frame from the final rendering GIF created after training is complete

<p align="center">
  <img src="ref_output/q1_training_example_2.png" alt="Q1_Training_2" width=15%/>
</p>

Do note that while the reference we have provided is a still frame, we expect you to submit the GIFs output by the rendering code. 

Feel free to experiment with different learning rate values and number of iterations. After training is completed, the script will save the trained gaussians and compute the PSNR and SSIM on some held out views. 

**GPU Memory Usage**: This task (with default paramter settings) may use approximately 15.5GB GPU memory. You can decrease/increase GPU memory utilization for performance by using the `--gaussians_per_splat` argument.

> **Submission:** In your webpage, include the following details:
> - Learning rates that you used for each parameter. If you had experimented with multiple sets of learning rates, just mention the set that obtains the best performance in the next question.
> - Number of iterations that you trained the model for.
> - The PSNR and SSIM.
> - Both the GIFs output by `train.py`.

## 1.3 Extensions **(Choose at least one! More than one is extra credit)**

### 1.3.1 Rendering Using Spherical Harmonics (10 Points)

In the previous sections, we implemented a 3D Gaussian rasterizer that is view independent. However, scenes often contain elements whose apperance looks different when viewed from a different direction (for example, reflections on a shiny surface). To model these view dependent effects, the authors of the 3D Gaussian Splatting paper use spherical harmonics.

The 3D Gaussians trained using the original codebase often come with learnt spherical harmonics components as well. Infact, for the scene used in section 1.1, we do have access to the spherical harmonic components! For simplicity, we had extracted only the view independent part of the spherical harmonics (the 0th order coefficients) and returned that as colour. As a result, the renderings we obtained in the previous section can only represent view independent information, and might also have lesser visual quality.

In this section, we will explore rendering 3D Gaussians with associated spherical harmonic components. We will add support for spherical harmonics such that we can run inference on 3D Gaussians that are already pre-trained using the original repository (we will **not** focus on training spherical harmonic coefficients).

In this section, you will complete code in `model.py` and `data_utils.py` to enable the utilization of spherical harmonics. You will also have to modify parts of `model.py`. Please lookout for the tag `[Q 1.3.1]` in the code to find parts of the code that need to be modified and/or completed.

In particular, the function `colours_from_spherical_harmonics` requires you to compute colour given spherical harmonic components and directions. You can refer to function `get_color` in this [implementation](https://github.com/thomasantony/splat/blob/0d856a6accd60099dd9518a51b77a7bc9fd9ff6b/notes/00_Gaussian_Projection.ipynb) and create a vectorized version of the same for your implementation.

Once you have completed the above tasks, run `render.py` (use the same command that you used for question 1.1.5). This script will take a few minutes to render views of a scene represented by pre-trained 3D Gaussians.

For reference, here is one frame of the rendering obtained when we use all spherical harmonic components:

<p align="center">
  <img src="ref_output/q1_with_sh.png" alt="Q1_With_SH" width=60%/>
</p>

For comparison, here is one frame of the rendering obtained when we used only the DC spherical harmonic component (i.e., what we had done for question 1.1.5):

<p align="center">
  <img src="ref_output/q1_without_sh.png" alt="Q1_Without_SH" width=60%/>
</p>

> **Submission:** In your webpage, include the following details:
> - Attach the GIF you obtained using `render.py` for questions 1.3.1 (this question) and 1.1.5 (older question).
> - Attach 2 or 3 side by side RGB image comparisons of the renderings obtained from both the cases. The images that are being compared should correspond to the same view/frame.
> - For each of the side by side comparisons that are attached, provide some explanation of differences (if any) that you notice.

### 1.3.2 Training On a Harder Scene (10 Points)

In section 1.2, we explored training 3D Gaussians to represent a scene. However, that scene is relatively simple. Furthermore, the Gaussians benefitted from being initialized using many high-quality noise free points (which we had already setup for you). Hence, we were able to get reasonably good performance by just using isotropic Gaussians and by adjusting the learning rate.

However, real scenes are much more complex. For instance, the scene may comprise of thin structures that might be hard to model using isotropic Gaussians. Furthermore, the points that are used to initialize the means of the 3D Gaussians might be noisy (since the points themselves might be estimated using algorithms that might produce noisy/incorrect predictions).

In this section, we will try training 3D Gaussians on harder data and initialization conditions. For this task, we will start with randomly initialized points for the 3D Gaussian means (which makes convergence hard). Also, we will use a scene that is more challenging than the toy cow data that you used for question 1.2. We will use the materials dataset from the NeRF synthetic dataset. While this dataset does not have thin structures, it has many objects in one scene. This makes convergence challenging due to the type of random intialization we do internally (we sample points from a single sphere centered at the origin). You can download the materials dataset from [here](https://drive.google.com/file/d/1v_0w1bx6m-SMZdqu3IFO71FEsu-VJJyb/view?usp=sharing). Unzip the materials dataset into the `./Q1/data` directory.

You will have to complete the code in `train_harder_scene.py` for this question. You can start this question by reusing your solutions from question 1.2. But you may quickly realize that this may not give good results. Your task is to experiment with techniques to improve performance as much as possible.

Here are a few possible ideas to try and improve performance:
- You can start by experimenting with different learning rates and training for much longer duration.
- You can consider using learning rate scheduling strategies (and these can be different for each parameter).
- The original paper uses SSIM loss as well to encourage performance. You can consider using similar loss functions as well.
- The original paper uses adaptive density control to add or reduce Gaussians based on some conditions. You can experiment with a simplified variant of such schemes to improve performance.
- You can experiment with the initialization parameters in the `Gaussians` class.
- You can experiment with using anisotropic Gaussians instead of isotropic Gaussians. 

> **Submission:** In your webpage, include the following details:
> - Follow the submission instructions for question 1.2.2 for this question as well.
> - As a baseline, use the training setup that you used for question 1.2.2 (isotropic Gaussians, learning rates, loss etc.) for this dataset (make sure `Gaussians` have `init_type="random"`). Save the training progress GIF and PSNR, SSIM metrics.
> - Compare the training progress GIFs and PSNR, SSIM metrics for both the baseline and your improved approach on this new dataset.
> - For your best performing method, explain all the modifications that you made to your approach compared to question 1.2 to improve performance.

# Credit to 16-825: Learning for 3D Vision class