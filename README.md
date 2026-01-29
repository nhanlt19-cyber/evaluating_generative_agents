# Generative Agents chạy local (Ollama / Llama.cpp / GPT4All) — Hướng dẫn triển khai & demo (Tiếng Việt)

![Smallville](cover.png)

Repo này là một fork của dự án đi kèm paper **“Generative Agents: Interactive Simulacra of Human Behavior”**. Bản gốc mặc định dùng OpenAI API (tốn chi phí); fork này đã tích hợp để chạy **LLM local** thông qua các framework như **Ollama**, **llama.cpp**, **GPT4All** (thông qua LangChain).

![oss_llm](https://github.com/rlancemartin/generative_agents/assets/122662504/be4d142f-a4f1-4819-ae7b-628e28207fc6)

---

## Tổng quan kiến trúc (cần chạy 2 server song song)

- **Environment server (Django)**: `environment/frontend_server`  
  Phục vụ UI map, demo, replay; giao tiếp với backend bằng các endpoint như `process_environment` / `update_environment`.
- **Simulation server (Python)**: `reverie/backend_server/reverie.py`  
  Tạo/tiếp tục simulation, gọi LLM + embeddings, ghi movement vào `environment/frontend_server/storage`.

Bạn cần **mở 2 cửa sổ terminal** và chạy đồng thời cả hai.

---

## Yêu cầu hệ thống

- **Python**: khuyến nghị **Python 3.9.x** (repo gốc test 3.9.12)
- **LLM local (khuyến nghị)**: **Ollama** (HTTP `http://localhost:11434`)
- **RAM/CPU**: tuỳ theo model + số agent (xem mục “Cấu hình server” phía dưới)

---

## Bước 1 — Cài LLM local (khuyến nghị: Ollama)

### 1.1 Cài Ollama và pull model

- Cài Ollama theo hướng dẫn chính thức của Ollama.
- Pull model, ví dụ:

```bash
ollama pull llama2:13b
```

### 1.2 Chọn model dùng trong project

Project mặc định đang cấu hình Ollama trong `reverie/backend_server/persona/prompt_template/gpt_structure.py`:

- `Ollama(base_url="http://localhost:11434", model=ollama_model, ...)`

Tên model lấy từ biến `ollama_model` trong `reverie/backend_server/utils.py` (repo đã có sẵn file này).

---

## (Tuỳ chọn) Dùng LLM từ server khác (OpenAI-compatible `/v1/chat/completions`)

Nếu bạn có một server LLM cung cấp API tương thích OpenAI Chat Completions (endpoint dạng `/v1/chat/completions`), bạn có thể cấu hình để Reverie gọi sang server đó thay vì Ollama local.

### 1) Cấu hình `utils.py`

Mở `reverie/backend_server/utils.py` và đặt:

- `llm_provider = "openai_compat"`
- `openai_compat_api_base = "https://<host>/v1"`  (chỉ base `/v1`, không thêm `/chat/completions`)
- `openai_compat_model = "<ten-model>"` (ví dụ model bạn được server hỗ trợ)

### 2) Set API key qua biến môi trường (không commit key)

Trên Ubuntu:

```bash
export LLM_API_KEY="YOUR_KEY_HERE"
```

Sau đó chạy backend như bình thường (`python reverie.py`).

---

## Bước 2 — Tạo môi trường Python & cài dependencies

### 2.1 Tạo virtualenv

#### Ubuntu (khuyến nghị khi bạn deploy trên server)

Tại thư mục root của repo:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

#### Windows PowerShell (nếu bạn chạy local trên Windows)

Tại thư mục root của repo:

```bash
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

### 2.2 Cài requirements

Khuyến nghị cài bằng file root:

```bash
pip install -r requirements.txt
```

### 2.3 Kiểm tra file cấu hình `utils.py` (bắt buộc cho backend)

Backend simulation import `from utils import *`, nên bạn **phải có**:

- `reverie/backend_server/utils.py`

Trong file này, cần chú ý:

- **`ollama_model`**: đặt đúng tên model bạn đã `ollama pull`
- **`fs_storage` / `fs_temp_storage`**: mặc định đã trỏ sang `environment/frontend_server/...`

---

## Bước 3 — Chạy Environment Server (Django)

Mở terminal #1:

```bash
cd environment\frontend_server
python manage.py runserver 0.0.0.0:8000
```

### Truy cập từ laptop qua IP server

Trên laptop, mở:

- `http://<IP_SERVER>:8000/`
- `http://<IP_SERVER>:8000/simulator_home`

### Nếu Django báo `DisallowedHost`

Vì bạn truy cập qua IP, Django có thể chặn. Cách nhanh nhất cho môi trường nội bộ:

Sửa `environment/frontend_server/frontend_server/settings/local.py` (hoặc file settings đang dùng) và thêm:

- `ALLOWED_HOSTS = ["<IP_SERVER>", "localhost", "127.0.0.1"]`

Hoặc tạm thời (chỉ nội bộ/dev) dùng:

- `ALLOWED_HOSTS = ["*"]`

### Mở port trên Ubuntu (nếu có firewall)

Nếu server bật `ufw`:

```bash
sudo ufw allow 8000/tcp
sudo ufw status
```

Mở trình duyệt:

- `http://localhost:8000/` (landing)

---

## Bước 4 — Chạy Simulation Server (backend)

Mở terminal #2:

```bash
cd reverie\backend_server
python reverie.py
```

Bạn sẽ nhập:

- `Enter the name of the forked simulation:` (ví dụ `base_the_ville_isabella_maria_klaus`)
- `Enter the name of the new simulation:` (ví dụ `test-sim-001`)

Giữ terminal backend chạy, nó sẽ hiện: `Enter option:`

---

## Bước 5 — Mở UI simulator và chạy step

Mở:

- `http://localhost:8000/simulator_home`

Chạy simulation bằng lệnh trong terminal backend:

```text
run <step-count>
```

Ví dụ: `run 100` (1 step = 10 giây thời gian trong game)

Các lệnh hay dùng:

- `run 100`: chạy 100 steps
- `save`: lưu trạng thái
- `fin`: lưu và thoát
- `exit`: thoát và xoá folder simulation vừa tạo
- `autorun <max_steps> <save_frequency>`: tự chạy và thỉnh thoảng lưu bản copy (vd `autorun 1440 60`)

---

## Replay một simulation đã chạy

Khi Django server đang chạy, mở:

- `http://localhost:8000/replay/<simulation-name>/<starting-time-step>/`

Ví dụ (có sẵn trong repo):

- `http://localhost:8000/replay/July1_the_ville_isabella_maria_klaus-step-3-20/1/`

---

## Demo “đẹp” (có sprite đúng) — cần compress simulation

Replay chủ yếu để debug, visuals có thể không tối ưu. Để demo đúng sprite, cần **compress** simulation trước.

### 1) Compress

Mở terminal tại thư mục `reverie`:

```bash
cd reverie
python -c "from compress_sim_storage import compress; compress('test-sim-001')"
```

Output nằm ở:

- `environment/frontend_server/compressed_storage/<sim_code>/`

### 2) Mở demo

Mở:

- `http://localhost:8000/demo/<simulation-name>/<starting-time-step>/<simulation-speed>/`

Trong đó:

- `simulation-speed`: 1 chậm nhất, 5 nhanh hơn

Ví dụ:

- `http://localhost:8000/demo/July1_the_ville_isabella_maria_klaus-step-3-20/1/3/`

---

## Vị trí dữ liệu simulation

- **Simulation lưu (khi bạn `save`/`fin`)**: `environment/frontend_server/storage/`
- **Demo đã compress**: `environment/frontend_server/compressed_storage/`

---

## Tuỳ biến (agents/history)

### 1) Load history cho agent (khi đang ở prompt `Enter option:`)

```text
call -- load history the_ville/<history_file_name>.csv
```

Gợi ý file mẫu (trong assets):

- `the_ville/agent_history_init_n3.csv` (base 3 agents)
- `the_ville/agent_history_init_n25.csv` (base 25 agents)

History CSV đặt tại:

- `environment/frontend_server/static_dirs/assets/the_ville/`

### 2) Tạo base simulation mới

Cách đơn giản nhất: copy một folder base trong `environment/frontend_server/storage/base_*` rồi đổi tên/sửa dữ liệu.
Nếu muốn đổi map / tăng số agent hỗ trợ, có thể cần chỉnh map bằng [Tiled](https://www.mapeditor.org/).

---

## Troubleshooting nhanh

- **Mở `http://localhost:8000/simulator_home` thấy trang báo chưa chạy backend**:
  - Backend `python reverie.py` chưa chạy, hoặc chưa tạo `temp_storage/curr_step.json`.
  - Hãy chạy backend và nhập fork/new simulation xong rồi refresh trang.
- **Lỗi `ModuleNotFoundError: utils` ở backend**:
  - Đảm bảo có file `reverie/backend_server/utils.py` và bạn đang `cd reverie\backend_server` khi chạy `python reverie.py`.
- **LLM không trả lời / không kết nối Ollama**:
  - Đảm bảo Ollama đang chạy và truy cập được `http://localhost:11434`.
  - Đảm bảo `ollama_model` trong `reverie/backend_server/utils.py` đúng tên model bạn đã `ollama pull`.

---

## Gợi ý triển khai nội bộ an toàn hơn (khuyến nghị)

Nếu bạn không muốn mở port 8000 ra LAN, bạn có thể dùng SSH port-forward từ laptop:

```bash
ssh -L 8000:127.0.0.1:8000 <user>@<IP_SERVER>
```

Sau đó trên laptop mở `http://localhost:8000/` (traffic được tunnel vào server).

---

## Cấu hình server 32 vCPU / 32GB RAM có phù hợp không?

- **Chạy simulation + Django**: **phù hợp** (32 vCPU / 32GB RAM là dư cho phần server + I/O + Python).
- **Chạy LLM local**: phụ thuộc model và có GPU hay không.

Gợi ý nhanh:

- **CPU-only**: nên dùng model nhỏ/quantized (7B–13B) để tốc độ chấp nhận được; chạy `base_the_ville_n25` (25 agents) sẽ rất chậm.
- **Có GPU**: tốc độ inference cải thiện rõ rệt, phù hợp chạy nhiều agents/steps hơn.

Nếu bạn cho mình biết server có **GPU (VRAM bao nhiêu)** và bạn định dùng model nào (7B/13B/70B), mình sẽ ước lượng tốc độ và gợi ý cấu hình model hợp lý cho demo.

---

## Authors and Citation

**Authors:** Joon Sung Park, Joseph C. O'Brien, Carrie J. Cai, Meredith Ringel Morris, Percy Liang, Michael S. Bernstein

Please cite our paper if you use the code or data in this repository.

```bibtex
@inproceedings{Park2023GenerativeAgents,
author = {Park, Joon Sung and O'Brien, Joseph C. and Cai, Carrie J. and Morris, Meredith Ringel and Liang, Percy and Bernstein, Michael S.},
title = {Generative Agents: Interactive Simulacra of Human Behavior},
year = {2023},
publisher = {Association for Computing Machinery},
address = {New York, NY, USA},
booktitle = {In the 36th Annual ACM Symposium on User Interface Software and Technology (UIST '23)},
keywords = {Human-AI interaction, agents, generative AI, large language models},
location = {San Francisco, CA, USA},
series = {UIST '23}
}
```

## Acknowledgements

We encourage you to support the following three amazing artists who have designed the game assets for this project, especially if you are planning to use the assets included here for your own project:

- Background art: [PixyMoon (@_PixyMoon_)](https://twitter.com/_PixyMoon_)
- Furniture/interior design: [LimeZu (@lime_px)](https://twitter.com/lime_px)
- Character design: [ぴぽ (@pipohi)](https://twitter.com/pipohi)

In addition, we thank Lindsay Popowski, Philip Guo, Michael Terry, and the Center for Advanced Study in the Behavioral Sciences (CASBS) community for their insights, discussions, and support. Lastly, all locations featured in Smallville are inspired by real-world locations that Joon has frequented as an undergraduate and graduate student---he thanks everyone there for feeding and supporting him all these years.
