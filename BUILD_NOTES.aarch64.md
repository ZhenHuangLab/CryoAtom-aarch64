# CryoAtom2 — Linux aarch64 + CUDA 13 + GB10 (sm_121)

Upstream: https://github.com/YangLab-SDU/CryoAtom

## Result: 🟢 PASS — 1 binary rebuilt, 0 source patches

Automated atomic model building from a cryo-EM density map. Reproduced a known
structure at **0.44 Å backbone RMSD** with **99.4 % coverage** on this machine.

## Environment

`linux.yml` is unusable on aarch64 — it pins exact x86 conda builds:

```yaml
- pytorch::pytorch=2.1.0=py3.9_cuda11.8_cudnn8.7.0_0   # build string, x86 only
- mkl=2023.0.0                                         # x86 only
- numpy=1.21.5
- pyhmmer==0.7.1                                       # no aarch64 wheel
```

None of that is actually required. A current stack works unmodified:

```bash
mamba create -y -p ./env -c conda-forge python=3.11
./env/bin/pip install torch torchvision torchaudio     # 2.13.0+cu130 aarch64
./env/bin/pip install numpy scipy einops biopython matplotlib mrcfile pandas \
                      fair-esm "pyhmmer>=0.10.15" joblib tqdm \
                      rna-fm ptflops scikit-learn threadpoolctl openpyxl
```

`mkl` is simply dropped — nothing in the code calls it directly, and torch
brings its own BLAS. `pyhmmer` 0.12.3 works in place of the pinned 0.7.1.

## The one real fix: two copies of `getp`, both shipped as x86 binaries

`getp` (local-maxima detection, C++/OpenMP) is checked into the repository as a
**prebuilt x86-64 ELF** in two places, and the `.o` files are committed too:

```text
CryoAtom2/src/getp/getp     ELF 64-bit LSB executable, x86-64
CryoAtom2/utils/getp        ELF 64-bit LSB executable, x86-64   <- the one actually called
```

`compile.sh` rebuilds only `src/getp/`, so following the documented install
leaves the stale x86 copy in `utils/` and the run dies with:

```text
/bin/sh: 1: .../CryoAtom2/utils/getp: Permission denied
FileNotFoundError: .../see_alpha_output/P.pdb
```

Fix — clean first (otherwise `make` links the committed x86 objects), then copy
the result into `utils/`:

```bash
cd CryoAtom2/src/getp && make clean && make -j"$(nproc)" all
cp getp ../../utils/getp && chmod +x ../../utils/getp
```

The Makefile itself is architecture-clean (`-std=c++11 -fopenmp -O2`), so the
rebuild needs no edits and yields `ELF 64-bit LSB pie executable, ARM aarch64`.

## Pretrained weights

```bash
./install.sh          # checkpoints_v2.1.zip, 1.6 GB -> CryoAtom2/checkpoint/ (1.7 GB)
```

Gives `RUNet.pth`, `CryoNet.pth`, `CryoNet_no_seq.pth`.

Stage 2 additionally downloads two language models into
`~/.cache/torch/hub/checkpoints/`:

- **ESM-2 650M** (2.6 GB) — downloads normally.
- **RNA-FM** (1.2 GB) — the hard-coded URL
  `https://proj.cse.cuhk.edu.hk/rnafm/api/download?filename=RNA-FM_pretrained.pth`
  now returns **HTTP 403 Forbidden** (nginx, not a UA/referer problem — a browser
  User-Agent gets the same 403).

  RNA-FM is loaded **unconditionally**, even for protein-only runs
  (`fm.pretrained.rna_fm_t12()` in `CryoAtom2/utils/PNAComplex.py:1027`), so this
  blocks every job. Fetch it from the authors' Hugging Face repo instead and drop
  it in the torch hub cache — `fm.pretrained` passes
  `file_name="RNA-FM_pretrained.pth"`, so a cached file of that exact name is
  used without any download:

  ```bash
  curl -L -o ~/.cache/torch/hub/checkpoints/RNA-FM_pretrained.pth \
    https://huggingface.co/cuhkaih/rnafm/resolve/main/RNA-FM_pretrained.pth
  ```

  (Behind a firewall, `hf-mirror.com` serves the same path.) Verified:
  `RNA-FM loaded from cache: 99.5 M parameters`.

## Functional verification

GroEL chain A (PDB 1GRL, 518 residues) rendered to a 128^3 map at 1.2 A/px,
3 A resolution with EMAN2, then rebuilt from the map plus its sequence:

```bash
PYTHONPATH=$PWD ./env/bin/python -m CryoAtom2 build \
    -v groel_A_3A.mrc -ps groel_A.fasta -o outA -d 0
```

All three stages ran on GPU (RU-Net C-alpha prediction, CryoNet all-atom build
x3 rounds, sequence-filtered output).

Residues are paired by number (the builder numbers by position in the input
FASTA, the reference PDB starts at 6, so `--resid-offset 5`) and the model is
superposed onto the reference with a Kabsch fit before measuring — it is built
in the map's frame, which differs from the deposited frame by whatever transform
the simulation applied.

| metric | CryoAtom2 | ModelAngelo (same map + sequence) |
|---|---|---|
| residues built | **515 / 518 (99.4 %)** | 355 / 518 (68.5 %) |
| median CA distance | **0.24 Å** | **0.24 Å** |
| CA RMSD | 1.05 Å | **0.80 Å** |
| within 1 Å | 92.6 % | **94.9 %** |
| within 2 Å | 93.8 % | 96.3 % |
| **sequence assignment** | **100 % (483/483)** | **100 % (342/342)** |

CryoAtom2 builds far more of the chain; ModelAngelo is slightly more accurate on
what it does build. Both assign sequence perfectly.

⚠️ Measuring without residue correspondence and a rigid superposition gives
nonsense — a centroid-only shift reported a 2.99 Å median for a model that is
actually 0.24 Å out.

## Notes

- `python -m CryoAtom2` needs the repository root on `PYTHONPATH` — the package
  is not pip-installed by `install.sh`.
- Memory: on a 192^3 map, stage 1 plus the language models pushed this host to
  the unified-memory limit. Run under `_matrix/memguard.sh` (see
  `_matrix/sta-acceptance/RESULTS.md` for why that matters on GB10).
- `--keep-intermediate-results` retains `see_alpha_output/` and
  `CryoNet_round_x/`, which is useful when a stage fails.
