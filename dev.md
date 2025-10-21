# 🧠 NanoChat 로컬 학습 세팅 (RTX 3060 환경)

## 💻 작성자 하드웨어

| 항목    | 사양                                    |
| ------- | --------------------------------------- |
| **CPU** | Intel Core i7-12700K                    |
| **RAM** | Samsung DDR5 4800MHz 16GB × 2 (총 32GB) |
| **SSD** | Samsung PM9A1 1TB (NVMe)                |
| **VGA** | NVIDIA GeForce RTX 3060 12GB            |
| **OS**  | Windows 11 25H2                         |
| **WSL** | Ubuntu 24.04 LTS                        |

---

## 1️⃣ WSL 진입 후 초기 세팅

```bash
# 프로젝트 클론 및 진입
cd ~/
git clone https://github.com/karpathy/nanochat.git
cd nanochat

# 파이썬 가상환경 생성 및 활성화
python3 -m venv venv
source venv/bin/activate

# CUDA 인식 확인
nvidia-smi
```

> 💡 만약 `nvidia-smi` 명령이 없다고 나오면 다음을 실행 후 재부팅:
>
> ```bash
> sudo add-apt-repository ppa:graphics-drivers/ppa
> sudo ubuntu-drivers autoinstall
> sudo reboot
> nvidia-smi
> ```

이후 진행을 위해 필수
```bash
cd ~/.cache/nanochat
curl -L -o eval_bundle.zip https://karpathy-public.s3.us-west-2.amazonaws.com/eval_bundle.zip
unzip eval_bundle.zip
rm eval_bundle.zip
```

---

## 2️⃣ 데이터셋 샘플 다운로드 (10 샤드만 테스트용)

```bash
python -m nanochat.dataset -n 10 -w 2
```

- 기본 저장 경로: `~/.cache/nanochat/base_data`
- 약 1GB 내외의 텍스트 데이터가 다운로드됨

---

## 3️⃣ Rust 기반 토크나이저 빌드 (`rustbpe`)

```bash
pip install maturin
maturin develop --release --manifest-path rustbpe/Cargo.toml
```

> ✅ 성공 시 출력 예시:
>
> ```
> Finished `release` profile [optimized] target(s) in Xs
> 📦 Built wheel for CPython 3.12 ...
> 🛠 Installed nanochat-0.1.0
> ```

### 토크나이저 정상 동작 확인

```bash
python - <<'PY'
import rustbpe
print("rustbpe OK. has Tokenizer?", hasattr(rustbpe, "Tokenizer"))
print(dir(rustbpe))
PY
```

> ✅ 출력 예시
>
> ```
> rustbpe OK. has Tokenizer? True
> ['Tokenizer', '__all__', ...]
> ```

---

## 4️⃣ 토크나이저 학습 및 평가

```bash
python -m scripts.tok_train --max_chars=200000000
python -m scripts.tok_eval
```

> ✅ 출력 예시:
>
> ```
> Saved tokenizer encoding to ~/.cache/nanochat/tokenizer/tokenizer.pkl
> Ours: vocab_size=65536 (GPT-2: 50257, GPT-4: 100277)
> ```
>
> - 정상적으로 저장되면 토크나이저 단계 완료
> - GPT-2/4 대비 압축률 비교 결과도 표시됨

---

## 5️⃣ CUDA 메모리 최적화 환경 변수 설정

```bash
export PYTORCH_ALLOC_CONF="expandable_segments:True,max_split_size_mb:64,garbage_collection_threshold:0.8"
```

- PyTorch 2.9 이상에서는 `PYTORCH_CUDA_ALLOC_CONF` 대신 `PYTORCH_ALLOC_CONF` 사용
- 메모리 단편화 최소화 및 VRAM 누수 방지 효과

---

## 6️⃣ Base 모델 사전학습 (테스트용 소형 설정)

```bash
# 배치/시퀀스에 맞춰 total_batch_size를 배수로 설정 (384 * 1024 = 393216)
python -m scripts.base_train \
  --run=dummy \
  --depth=6 \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=1000
```

### ⚙️ 주요 파라미터 설명

| 옵션                        | 설명                            |
| --------------------------- | ------------------------------- |
| `--depth=6`                 | Transformer 블록 6층            |
| `--max_seq_len=384`         | 입력 시퀀스 최대 토큰 길이      |
| `--device_batch_size=1`     | GPU 단일 배치 (VRAM 절약용)     |
| `--total_batch_size=393216` | 배치/시퀀스 곱의 배수 조건 필수 |
| `--num_iterations=1000`     | 학습 스텝 수                    |

### 🧩 실행 요약

- **예상 VRAM 사용량:** 7~9GB
- **예상 소요시간:** 약 9\~11시간 (성능 70% 제한 시 12\~14시간)
- **학습 출력 경로:** `./runs/dummy`
- **로그 및 체크포인트 자동 저장**

---

## ⚙️ (옵션) VRAM 제한 설정

```bash
# PATH 환경변수에 nvidia-smi 경로 추가
export PATH="/usr/lib/wsl/lib:$PATH"

# GPU 모니터링
watch -n 1 nvidia-smi
```

---

## ✅ 학습 완료 후

- 결과 모델과 로그는 `./runs/day1_base_small` 또는 `./runs/dummy` 경로에 저장됨
- 이후 단계:

# 📚 NanoChat 진행 플랜 — 권장 / 탄탄 / 전체 (분기 가능한 체크포인트 운영)

> ✅ 전제: 아래 베이스라인(이미 진행)
>
> ```bash
> # Base Pretrain 1차: 1,000 스텝(완료)
> python -m scripts.base_train \
>   --run=dummy \
>   --depth=6 \
>   --max_seq_len=384 \
>   --device_batch_size=1 \
>   --total_batch_size=393216 \
>   --num_iterations=1000
> ```

---

## 🔀 분기(Branch) 저장/재개 — 공통 루틴

> 각 Day 종료 직후 최신 체크포인트를 변수에 담아 **분기용으로 보관**합니다.  
> 일부 스크립트는 `--init_from`를 지원하지 않을 수 있습니다. **지원되면 사용**, 없으면 **동일 `--run`으로 이어서** 진행하세요.

```bash
# 최신 체크포인트 경로 잡기
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"

# (옵션) 안전 보관용 복사
mkdir -p ckpt_archive
[ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/

# (옵션) 분기용 run 이름으로 시작하고 싶으면
# 동일 옵션 + --run=<새이름> 으로 재시작 (지원 시 --init_from 사용)
# 예시:
# python -m scripts.base_train \
#   --run=dummy_branchA \
#   --max_seq_len=384 \
#   --device_batch_size=1 \
#   --total_batch_size=393216 \
#   --num_iterations=500 \
#   --init_from="$CKPT"     # 스크립트에 없으면 이 줄은 빼세요
```

---

# 🟢 권장(Recommended) 버전 — 약 1~2일 / 6h 블록 × 2~3회

### Day 1 (6h)

**Base Pretrain 1차: +1,500 스텝 (누적 ~2,500 / ~0.98B 토큰)**

```bash
export PYTORCH_ALLOC_CONF="expandable_segments:True,max_split_size_mb:64,garbage_collection_threshold:0.8"

python -m scripts.base_train \
  --run=dummy \
  --depth=6 \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=1500
```

**분기 저장**

```bash
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 2 (6h)

**Base Pretrain 2차: +500 스텝 (선택, 여유시간용)**

> 누적 3,000 스텝까지 올리면 일반적으로 안정감 ↑ (원하면 생략 가능)

```bash
python -m scripts.base_train \
  --run=dummy \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=500
```

**빠른 평가**

```bash
python -m scripts.base_eval   # 레포 버전에 따라 유효/무효
python -m scripts.base_loss   # 로스/샘플 확인
```

**분기 저장**

```bash
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 3 (6h)

**Mid-Train 1차: 1,000 스텝**

```bash
python -m scripts.mid_train \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1000
python -m scripts.chat_eval -- -i mid
```

**SFT 1차: 1,000 스텝**

```bash
python -m scripts.chat_sft \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1000
python -m scripts.chat_eval -- -i sft
```

**웹UI**

```bash
python -m scripts.chat_web
# http://127.0.0.1:8000
```

---

# 🔵 탄탄(Solid) 버전 — 약 2~3일 / 6h 블록 × 3~4회

### Day 1 (6h)

**Base Pretrain 1차: +1,500 스텝 (누적 ~2,500)**

```bash
export PYTORCH_ALLOC_CONF="expandable_segments:True,max_split_size_mb:64,garbage_collection_threshold:0.8"

python -m scripts.base_train \
  --run=dummy \
  --depth=6 \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=1500
```

**분기 저장**

```bash
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 2 (6h)

**Base Pretrain 2차: +1,500 스텝 (누적 ~4,000 / ~1.57B 토큰)**

```bash
python -m scripts.base_train \
  --run=dummy \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=1500
```

**빠른 평가 & 분기 저장**

```bash
python -m scripts.base_eval
python -m scripts.base_loss
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 3 (6h)

**Mid-Train 1차: 1,000 스텝**

```bash
python -m scripts.mid_train \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1000
python -m scripts.chat_eval -- -i mid
```

**SFT 1차: 1,000 스텝**

```bash
python -m scripts.chat_sft \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1000
python -m scripts.chat_eval -- -i sft
```

---

### Day 4 (선택, 3h 내외)

**(옵션) RL 500~1000 스텝 + 리포트/웹UI**

```bash
# RL (옵션: GSM8K)
# python -m scripts.chat_rl -- --run=dummy  # 레포 버전에 따라 인자 상이 가능

python -m nanochat.report generate   # 종합 리포트 (레포 지원 시)
python -m scripts.chat_web
# http://127.0.0.1:8000
```

---

# 🟣 전체(Full-ish) 버전 — 약 3~4일 / 6h 블록 × 4~5회

### Day 1 (6h)

**Base Pretrain 1차: +2,000 스텝 (누적 ~3,000)**

```bash
export PYTORCH_ALLOC_CONF="expandable_segments:True,max_split_size_mb:64,garbage_collection_threshold:0.8"

python -m scripts.base_train \
  --run=dummy \
  --depth=6 \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=2000
```

**분기 저장**

```bash
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 2 (6h)

**Base Pretrain 2차: +2,000 스텝 (누적 ~5,000 / ~1.96B 토큰)**

```bash
python -m scripts.base_train \
  --run=dummy \
  --max_seq_len=384 \
  --device_batch_size=1 \
  --total_batch_size=393216 \
  --num_iterations=2000
```

**빠른 평가 & 분기 저장**

```bash
python -m scripts.base_eval
python -m scripts.base_loss
CKPT=$(ls -1t runs/dummy/*.pt 2>/dev/null | head -n 1)
echo "LATEST CKPT: $CKPT"
mkdir -p ckpt_archive && [ -n "$CKPT" ] && cp -v "$CKPT" ckpt_archive/
```

---

### Day 3 (6h)

**Mid-Train 1차: 1,500 스텝** _(Full 권장: 조금 더 태움)_

```bash
python -m scripts.mid_train \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1500
python -m scripts.chat_eval -- -i mid
```

---

### Day 4 (6h)

**SFT 1차: 1,500 스텝**

```bash
python -m scripts.chat_sft \
  --run=dummy \
  --device_batch_size=1 \
  --max_steps=1500
python -m scripts.chat_eval -- -i sft
```

---

### Day 5 (선택, 3h 내외)

**(옵션) RL 1000 스텝 + 리포트/웹UI**

```bash
# RL (옵션)
# python -m scripts.chat_rl -- --run=dummy

python -m nanochat.report generate
python -m scripts.chat_web
# http://127.0.0.1:8000
```

---

## 🧩 실전 팁 (공통)

- **리소스 여유가 필요할 때**
  `--max_seq_len=256` + `--total_batch_size=262144` 로 한 단계 낮춰 스파이크/프레임드랍 완화.
- **WSL I/O 최적화**
  프로젝트/데이터를 **리눅스 홈(`~`)**에 두기(`/mnt/c` 경유 I/O 지양).
- **웹서핑/동영상 병행 시**
  브라우저 **하드웨어 가속 OFF**, 탭 수 최소화, 전력 캡(`sudo nvidia-smi -pl 120`) 유지.
- **분기 실험**
  Day마다 `ckpt_archive/`에 체크포인트 백업 후, 필요 시 새 `--run`으로 가지치기.
  _`--init_from` 옵션이 없으면 생략하고 동일 `--run`으로 이어서 학습._
