# Network-Traffic-Anomaly-Detection-Using-GANs

Unsupervised anomaly detection in network traffic using a GAN-based autoencoder trained exclusively on normal data. No attack labels required during training.

---

## Background

Signature-based intrusion detection systems fail against zero-day attacks because they rely on known patterns. This project takes a different approach — instead of learning what attacks look like, the model learns what *normal* traffic looks like, then flags anything that doesn't fit.

The core idea: train an encoder-decoder (autoencoder) adversarially on clean traffic. At inference time, anything that reconstructs poorly is probably anomalous.

---

## Dataset

**NSL-KDD** — an improved version of the KDD Cup 1999 dataset with duplicate records removed.

- 41 features per record (connection duration, protocol type, byte counts, flags, etc.)
- Labels: `normal` and a range of attack categories (DoS, Probe, R2L, U2R)
- During training: **only normal samples are used**
- At test time: mixed normal + attack samples

```
data/
├── KDDTrain+.txt
└── KDDTest+.txt
```

You can get the dataset from the [Canadian Institute for Cybersecurity](https://www.unb.ca/cic/datasets/nsl.html).

---

## How it works

### Preprocessing

- Categorical features (`protocol_type`, `service`, `flag`) → one-hot encoded
- Numerical features → `StandardScaler`
- Training set filtered to normal class only
- Test set includes both normal and anomaly samples

### Architecture

Three components work together:

**Encoder** — maps input to a compressed latent representation
```
Input(n_features) → Dense(256, ReLU) → Dense(64) → Latent(32)
```

**Decoder** — reconstructs input from latent vector
```
Latent(32) → Dense(256, ReLU) → Dense(n_features)
```

**Discriminator** — distinguishes real samples from reconstructed ones; used to sharpen the generator during training
```
Input(n_features) → Dense(256, LeakyReLU) → Dense(1, Sigmoid)
```

### Training

The model trains in two alternating phases:

1. **Discriminator step** — train discriminator to separate real vs reconstructed
2. **Encoder-Decoder step** — minimize reconstruction loss while fooling the discriminator

Combined loss:
```
L_total = L_reconstruction + 0.1 × L_adversarial + 0.5 × L_latent
```

Where `L_latent` penalizes inconsistency in the encoded space.

### Anomaly scoring

At test time, each sample gets a score based on how poorly it reconstructs:
```
score = reconstruction_error + 0.5 × latent_consistency_error
```

Threshold is set at the **99th percentile** of scores on held-out normal data. Anything above the threshold is flagged as anomalous.

---

## Setup

```bash
git clone https://github.com/yourname/network-anomaly-detection.git
cd network-anomaly-detection

pip install -r requirements.txt
```

**Dependencies:**
- Python 3.9+
- TensorFlow 2.x
- scikit-learn
- pandas, numpy
- matplotlib, seaborn

---

## Usage

**Preprocess data:**
```bash
python src/preprocess.py --train data/KDDTrain+.txt --test data/KDDTest+.txt
```

**Train the model:**
```bash
python src/train.py --epochs 50 --batch_size 256 --latent_dim 32
```

**Evaluate:**
```bash
python src/evaluate.py --model checkpoints/gan_model.h5
```

**Outputs:**
- Confusion matrix
- ROC-AUC and Precision-Recall curves
- Anomaly score distribution histogram
- Threshold visualization

---

## Results

| Metric | Score |
|---|---|
| ROC-AUC | ~0.97 |
| PR-AUC | ~0.95 |
| F1 Score | ~0.92 |

*Results vary with random seed and threshold selection. Reported on NSL-KDD test set.*

---

## Project structure

```
├── data/                  # raw dataset files
├── src/
│   ├── preprocess.py      # data loading and feature engineering
│   ├── model.py           # encoder, decoder, discriminator definitions
│   ├── train.py           # training loop
│   └── evaluate.py        # scoring, threshold, metrics
├── notebooks/
│   └── exploration.ipynb  # EDA and visualization
├── checkpoints/           # saved model weights
├── requirements.txt
└── README.md
```

---

## Known issues / limitations

- GAN training can be unstable. If loss diverges, try lowering the discriminator learning rate or reducing adversarial loss weight.
- Threshold selection is sensitive to the validation set distribution — results may shift if the normal data isn't representative.
- The model doesn't currently capture temporal patterns in connection sequences. Adding an LSTM encoder would help with sequential flows.

---

## Possible extensions

- **WGAN-GP** for more stable adversarial training
- **LSTM encoder** for sequence-aware anomaly detection
- Real-time scoring via a lightweight inference server
- Integration with packet capture tools (e.g., Zeek, Suricata)

---

## References

- Schlegl et al., [Unsupervised Anomaly Detection with Generative Adversarial Networks](https://arxiv.org/abs/1703.05921) (AnoGAN, 2017)
- Akcay et al., [GANomaly: Semi-Supervised Anomaly Detection via Adversarial Training](https://arxiv.org/abs/1805.06725) (2019)
- Tavallaee et al., [A Detailed Analysis of the KDD CUP 99 Data Set](https://www.researchgate.net/publication/220565197) (NSL-KDD paper)

---

## License

MIT
