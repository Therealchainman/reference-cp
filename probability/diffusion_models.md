# Diffusion models

## 1. What is a diffusion model?

A diffusion model is a **generative machine learning model** that creates new data, such as images, by learning how to transform random noise into structured data.

It involves two processes:

- **Forward diffusion:** Gradually corrupt real images by adding random noise.
- **Reverse diffusion:** Learn to reverse that corruption, allowing the model to generate new images starting from random noise.

**Key insight:** Diffusion models learn to generate complex data by solving a series of simpler denoising problems.

## 2. What is noise in an image?

An image is represented by numerical pixel values. Noise consists of random numerical values added to those pixels.

For example:

$$
120 + 35 = 155
$$

Here, 120 is the original pixel value, 35 is random noise, and 155 is the noisy pixel.

Adding enough noise eventually makes the original image unrecognizable.

### Why Gaussian noise?

Diffusion models commonly use Gaussian noise, where random values follow a normal distribution.

This is useful because:

- Gaussian distributions are mathematically tractable.
- The sum of independent Gaussian random variables is Gaussian.
- We can calculate the result of many noise-adding steps directly.
- Gaussian noise naturally connects to Brownian motion and stochastic differential equations.

Gaussian noise isn't the only possibility, but it makes conventional diffusion models easier to formulate and train.

## 3. Why is it called diffusion?

The name comes from diffusion in physics.

In physical diffusion, particles spread through random motion, progressively reducing differences in concentration.

In machine learning, random noise progressively destroys the structure of data until its distribution approaches a simple noise distribution.

Both processes are described using related mathematics from probability theory and stochastic processes.

**Important:** Image diffusion doesn't literally move pixels around. It changes their numerical values through a controlled random process.

## 4. How are diffusion models trained?

The clever idea is to deliberately corrupt real images so the model has a well-defined prediction problem:

1. Start with a real training image.
2. Add a known amount of random noise.
3. Have the neural network predict the noise.
4. Compare the prediction with the actual noise and update the weights.

Because we generated the noise ourselves, we know exactly what the correct prediction should be. This provides an effective training objective without requiring humans to label the noise.

By repeating this process across many images and noise levels, the network learns the statistical structure of the training data.

## 5. How does a diffusion model generate an image from a text prompt?

Suppose you enter:

> A golden retriever running through a snowy forest.

The model doesn't directly convert those words into pixels. Instead:

1. **Encode the prompt:** A text encoder converts the description into a numerical representation.
2. **Generate random noise:** The model creates a random starting state.
3. **Perform reverse diffusion:** A trained neural network repeatedly updates the noisy representation, guided by the prompt.
4. **Produce the image:** The final representation becomes an image, possibly after decoding from a compressed latent space.

The prompt guides the denoising process at each step.

### Training vs. generation

| Training | Generation |
|----------|------------|
| Start with real images | Start with random noise |
| Add noise | Remove predicted noise |
| Learn to predict noise or clean data | Use those learned predictions |
| Update neural network weights | Usually keep weights fixed |
| Learn the data distribution | Generate a new sample |

The model is called a diffusion model because of both how it is trained and how it generates images.

## 6. Why transform randomness into structured data?

**Generating new data is fundamentally different from recognizing or classifying existing data.** The objective of generative modeling is to produce new samples from a complicated probability distribution.

For images, we want to approximate:

$$
p_{\text{data}}(x)
$$

But sampling directly from the distribution of realistic images is difficult.

Sampling from Gaussian noise is easy:

$$
z \sim \mathcal{N}(0,I)
$$

Diffusion models learn a process that transforms samples from this simple distribution into samples approximating the real data distribution:

$$
\boxed{z \sim \mathcal{N}(0,I) \quad\xrightarrow{\text{learned denoising}}\quad x \sim p_\theta(x)}
$$

**It transforms a probability distribution that's easy to sample from into one that's extremely complicated.** Randomness provides diversity, while learned denoising provides structure. This is why the same prompt can produce many different images.

### Different starting points produce different outputs

Imagine asking an AI to generate a picture of a cat.

The model has learned what cats look like, but there are practically unlimited possible cat images.

If the model always started from the same state and followed a deterministic process, it would produce the same image every time.

Instead, it starts with random noise.

- Possible output 1: [European Shorthair cat on white background](https://images.openai.com/static-rsc-4/ZUR-sNkBtl5HaPc1HY6uTBUxA4gO_tU-nJi7KbbJGPYRg4DTbxA-t0iqd1zI-t2be_FoANthNzcg08IsCzHypMnucdZiH2G1pITqG3LE6GjWCrK72f7NAMCUX8egpAX_VbH1xhGp_LhQ2UwgpMgFU5qr2vYlYKQYcUF0cPP0uaADi9D4IgJpbwo8KSm2X_rv?purpose=inline)
- Possible output 2: [Captivating Beauty: Long-Haired Black Cat](https://images.openai.com/static-rsc-4/XOm8ndiQoTRp5MZOKHsZOu1sBzj8k_xDZSaiaZ4a0PBd-giCy2OfV-YVgt8gGx5-PTlqIomXlPMuaCbVLy0-OnHZzI8LdlfeHTCY6EJHINYQ9pYT2Ryn6dOxiq6rcPydNB8t7KigiV-ipVvdCU6motFvMQVa2lxl-ncpO_jWehs?purpose=inline)
- Possible output 3: [White and gray kitten](https://images.openai.com/static-rsc-4/Ezr_hC61B6_VXT9hYd0ITxctRkV47xqFcPJkXzjgMez3CRBgXn3EbJrI9l8g7JI2cLcB-0LNUKQRlQenAFwhwz5Z9J2HFtnXzT1ko3dTdMl5eP3Ud_JbDq9U8RuHtIM1pBg5CDytNUHfcmGX2RA7RB9zGvZE7xfv4xkSb_yoUdrasELuPMQE93o4p2pGcXNK?purpose=inline)

Different random starting points can lead to different plausible images.

### Why not just generate an image directly?

You can! Other generative models, such as GANs and autoregressive models, use different approaches.

Diffusion models are useful because adding noise creates a sequence of progressively simpler learning problems. Instead of learning to generate an entire image in one step, the neural network learns how to predict noise or denoise an image at different noise levels.

This provides a practical and effective training objective.

## 7. How is this connected to statistical mechanics?

Diffusion models have mathematical connections to statistical physics, especially stochastic processes, Brownian motion, and probability distributions.

In statistical mechanics, we often start with microscopic randomness and investigate how macroscopic structure emerges. Diffusion models approach a related mathematical challenge from another direction: they learn how to generate configurations that exhibit the complex statistical structure found in real data.

The [Ising model and bootstrap percolation](../mathematics/ising_model_and_bootstrap_percolation.md) provide related points of comparison:

| Model | Primary purpose |
|-------|-----------------|
| Ising model | Understand how interactions produce collective order, disorder, and phase transitions |
| Bootstrap percolation | Understand how activation spreads through local threshold rules |
| Generative diffusion model | Learn to generate structured data by reversing a noise-corruption process |

These models share themes involving randomness and collective behavior, but describe different mathematical processes.

A diffusion model is not necessarily discovering physical laws or spontaneous order. It learns a transformation based on patterns in its training data.

Generative diffusion models should not be confused with threshold diffusion models used to study information spreading through social networks.

## 8. The most important mathematical equations

### Forward diffusion

$$
x_t = \sqrt{\bar{\alpha}_t}x_0 + \sqrt{1-\bar{\alpha}_t}\epsilon
$$

Here, $x_0$ is the original data, $x_t$ is its noisy representation, and $\epsilon$ is Gaussian noise. The factor $\bar{\alpha}_t$ controls how much of the original signal remains at noise level $t$.

### Training objective

For a typical noise-prediction model:

$$
L = \mathbb{E}\left[\|\epsilon - \epsilon_\theta(x_t,t,c)\|^2\right]
$$

Here, $c$ represents optional conditioning information, such as a text prompt.

The neural network learns to predict the noise from the noisy data, noise level, and conditioning information.

## Key takeaways

1. **Adding noise makes training easier:** We can create unlimited supervised examples from existing images because we know exactly which noise was added.
2. **Gaussian noise makes the mathematics convenient:** Its properties allow efficient training and a well-defined generative process.
3. **Denoising is the generation mechanism:** When you provide a prompt, a diffusion-based image generator starts from random noise and progressively transforms it into an image consistent with your description.

Ultimately, a diffusion model learns to sample from a complicated data distribution by reversing a mathematically controlled randomization process. The goal isn't simply to remove noise. It's to **sample from a complex probability distribution that would otherwise be difficult to generate from directly**.
