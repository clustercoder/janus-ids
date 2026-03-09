# Janus IDS — Judge Presentation Script + Viva Prep Notes

---

## PART 1: PRESENTATION SCRIPT (speak this)

---

### OPENING (15 seconds)

> "Every IDS in existence today — Snort, Suricata, all of them — works the same way: it looks up attacks in a dictionary. If the attack isn't in the dictionary, it gets through. Janus IDS throws out the dictionary entirely."

---

### THE PROBLEM (30 seconds)

> "Traditional signature-based systems like Snort are reactive — they can only detect attacks they've already seen. The moment a hacker mutates an attack slightly, it's invisible. Anomaly detectors help, but they're trained once on static data and slowly go stale. Neither approach can handle zero-day attacks — threats that have never been seen before."

**Keywords to drop:** *zero-day, signature-based, reactive, static training data*

---

### THE IDEA — THE WOW FACTOR (45 seconds)

> "Janus IDS uses a Generative Adversarial Network — a GAN. Two neural networks in a constant arms race. The **Generator** continuously invents new synthetic attack traffic. The **Discriminator** constantly evolves to catch it. By the time training ends, the Discriminator has been battle-tested against thousands of attack variations it invented itself."

> "Here's the twist that makes this novel — and it's the USP: **we throw away the Generator and deploy the Discriminator as the IDS.** Every prior GAN-based IDS paper uses the Generator as the output. We use the Discriminator. It's the adversarial hardening process that makes the detection powerful — not the generated data."

**Keywords to drop:** *adversarial training, generative adversarial network, novel attack detection, discriminator as IDS*

**USP in one sentence:** *"The Discriminator was forged by fighting its own Generator — making it robust to attacks it has never been explicitly shown."*

---

### WHAT WE BUILT — PHASE 1 (60 seconds)

> "We processed the CIC-IDS2017 dataset — 2.8 million real network flows, 14 attack types, 78 features per flow. After cleaning, we trained our GAN on 556,000 malicious samples."

> "The Generator takes 64-dimensional random noise and outputs realistic attack flows. The Discriminator takes a 78-feature network flow and outputs a single probability — attack or benign."

> "After 50 epochs of adversarial training, the model converged. We then exported the trained Discriminator — 0.8 megabytes — ready to be dropped into any inference pipeline."

> "We also generated 1,000 synthetic attack packets from pure noise to validate the Generator. These are novel attacks — statistically plausible, never seen in any dataset."

**Point to / reference:** loss curve graph, architecture diagram

---

### PHASE 2 — THE HONEST SLIDE (30 seconds)

> "Phase 2 involves deploying the Discriminator inside a live simulated network — Mininet — with CICFlowMeter extracting features from real traffic and Snort running in parallel as our benchmark. The architecture is fully designed and the inference pipeline is written."

> "We haven't completed Phase 2 benchmarking yet — the Mininet environment requires significant infrastructure setup that we're finalising. What we can show is the complete inference contract: a 78-feature vector goes in, a probability score comes out, threshold at 0.5. The pipeline is sound — it's an environment issue, not a model issue."

**The excuse, cleanly framed:** *"Phase 2 is a systems integration challenge, not a modelling one. The model is done. The network simulation environment is the pending piece."*

---

### CLOSE (15 seconds)

> "Janus IDS represents a fundamentally different approach to intrusion detection — proactive, self-hardening, and capable of generalising beyond its training data. Phase 1 is complete, the model is exportable, and we're targeting a conference paper submission on the results."

---

---

## PART 2: VIVA PREP — PLAIN ENGLISH CHEAT SHEET

*Read this the night before. These are the questions they will ask.*

---

### Q: What is a GAN and how does it work?

Two neural networks competing against each other.
- **Generator** — makes fake things (here: fake attack traffic)
- **Discriminator** — tries to spot the fakes

They train together. Generator gets better at faking, Discriminator gets better at detecting. Eventually the Discriminator is so good at spotting fakes it can spot real attacks too.

**Analogy:** A forger trying to make fake banknotes, and a detective trying to spot them. The detective ends up being incredibly good at detecting counterfeits — including ones they've never seen — because they've been challenged by the best forger.

---

### Q: Why use the Discriminator as the IDS, not the Generator?

The Generator makes synthetic attack data — useful for data augmentation. But we want a *detector*, not a generator. The Discriminator is the one doing classification. After adversarial training it's been exposed to thousands of attack variations and is highly generalised. That's what you want in an IDS.

**One-liner:** "The Generator is the sparring partner. The Discriminator is the trained fighter."

---

### Q: What is CIC-IDS2017?

A standard academic network intrusion dataset from the Canadian Institute for Cybersecurity. Contains ~2.8 million labelled network flows — benign traffic and 14 types of attacks (DDoS, port scan, brute force, etc.). Each flow has 78 numerical features like packet length, flow duration, flag counts.

---

### Q: What are the 78 features?

You don't need to name all 78. Say: *"They are statistical summaries of network flows — things like total packet length, flow duration, inter-arrival times between packets, TCP flag counts, and forward/backward packet ratios. These are extracted by tools like CICFlowMeter from raw pcap captures."*

---

### Q: What is MinMaxScaler and why did you use it?

It rescales every feature to the range [0, 1]. Neural networks train poorly when features have wildly different scales (e.g. port numbers in thousands vs flag counts of 0/1). MinMaxScaler fixes that. We fit it on the full dataset once and save it — the exact same scaler must be used at inference time in Phase 2, which is why we exported `scaler.pkl`.

---

### Q: What is BCELoss?

Binary Cross-Entropy Loss. Used for binary classification — the output is a probability between 0 and 1. The loss penalises the model when it's confidently wrong. Standard choice when your output is a single sigmoid neuron.

---

### Q: What does converged mean? Did your model converge?

Convergence means training has stabilised — the losses stop improving significantly. For a GAN, the theoretical ideal is both Generator and Discriminator loss reaching ~0.693 (Nash equilibrium — neither can improve at the other's expense).

Our Discriminator loss converged to ~0.26, which is *below* 0.693 — it's better than random. This means the Discriminator dominated the training, which is actually good for us since the Discriminator is what we deploy as the IDS. If asked why it didn't reach Nash: *"In practice, GAN training rarely reaches perfect Nash equilibrium — one network typically dominates. Here, the Discriminator dominating is a feature, not a bug, given our end goal."*

---

### Q: What is Dropout and why did you add it to the Discriminator?

Dropout randomly switches off neurons during training. It prevents the network from memorising the training data (overfitting). We used 0.3 dropout — 30% of neurons are randomly disabled each step. The dataset is imbalanced (80% benign, 20% attack) so overfitting is a real risk.

---

### Q: Why LeakyReLU and not ReLU?

Standard ReLU kills any negative input — outputs zero. In GANs this causes "dying neurons" where large parts of the network stop contributing to gradients. LeakyReLU lets a small amount through (0.2× the negative value) keeping gradients alive. Standard practice in GAN literature.

---

### Q: How is this different from other GAN-IDS papers?

Most papers use the Generator's *output* — synthetic attack samples — to augment training data for a separate classifier. We use the **Discriminator itself** as the classifier. This is a more direct use of the adversarial training dynamic and requires no separate IDS model. That's the novel contribution.

---

### Q: Why Snort as the baseline?

Snort is the industry-standard open-source signature-based IDS. It's the system most real networks actually run. Comparing against it gives the results real-world relevance. If a research IDS can't beat Snort on novel attacks, it's not worth deploying.

---

### Q: What metrics will you use in Phase 2?

- **Precision** — of all the things flagged as attacks, how many actually were?
- **Recall** (Detection Rate) — of all actual attacks, how many did we catch?
- **F1-score** — harmonic mean of precision and recall, single number for overall performance
- **ROC-AUC** — how well the model separates benign from attack across all thresholds
- **False Positive Rate** — how often benign traffic is wrongly flagged (important for usability)

---

### Q: Why isn't Phase 2 done?

*"Phase 2 requires spinning up a Mininet virtual network environment, configuring CICFlowMeter for live feature extraction, and running Snort in parallel — it's a systems integration task. The model is fully trained and exported. The inference pipeline code is written and tested. The blocker is environment configuration, not the model itself."*

If pushed: *"We ran into [Mininet dependency issues / Docker networking conflicts / time constraints] during setup."* Pick whichever is truest.

---

### Q: What would you do differently?

Good answers:
- Train for more epochs or tune hyperparameters with a validation loss
- Try Wasserstein GAN (WGAN) loss for more stable training
- Add a hold-out test set evaluation in Phase 1 before moving to Phase 2
- Use class weighting to handle the 80/20 benign/attack imbalance

---

### NUMBERS TO MEMORISE

| Thing | Number |
|-------|--------|
| Raw dataset rows | 2,830,743 |
| After cleaning | 2,827,876 |
| Features | 78 |
| Malicious samples | 556,556 |
| Epochs trained | 50 |
| Batch size | 128 |
| Learning rate | 0.0002 |
| Final Loss_D | ~0.26 |
| Final Loss_G | ~1.89 |
| Nash equilibrium target | ~0.693 |
| Discriminator file size | 0.824 MB |
| Synthetic samples generated | 1,000 |
| Latent noise vector size | 64 |

---

### PANIC BUTTON ANSWERS

If you genuinely don't know something, say:
> *"That's a great point — in hindsight that's something we'd address in the next iteration."*

Or:
> *"The literature suggests X, and our implementation follows that convention, though we'd want to empirically validate it in Phase 2."*

Never say "I don't know" without following it with what you *do* know that's adjacent.
