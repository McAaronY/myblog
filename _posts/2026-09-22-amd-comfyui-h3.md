---
title: 免费的ai模型
date: 2026-11-12 19:00 +0800
categories: [AI]
tags: [AMD anruicloud 部署comfyui+H3]
---

#### AMD nruicloud 部署comfyUI+H3 生成视频

### 第一步
1. 打开 https://developer.amd.com.cn/ 这个网址注册amd开发者账号
2. 第二部领取开发账号的开发者积分，默认开通的有200积分，1积分相当于服务器的1个小时，开发积分可以通过amd的活动进行领取
3. 积分兑换算力，打开 https://developer.amd.com.cn/points/redeem ，看到 ***立即兑换*** 按钮，点击后输入需要兑换的小时数
4. AMD 会生成兑换码在***立即兑换*** 按钮下方，需要时间，5分钟左右
5. 兑换码兑换算力 ***立即兑换*** 上面有具体的操作指引
6. 启动服务器，打开 https://developer.amd.com.cn/radeon/，下方有可以选择服务器类型，选择者**one-click**，后点击启动，启动后打开python的Notebook 开发环境
7. 新建一个start.sh 文件
8. 复制下方脚本，粘贴的start.sh 上
9. 打开终端执行一下命令
  ```bash
   chomd +x  start.sh
  ```
10. 等待命令执行完毕即可，免费获取一个可以制作视频的工具


### 可以使用的脚本

#### 脚本一

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

COMFY_DIR="${COMFY_DIR:-/root/ComfyUI}"
VENV_DIR="${VENV_DIR:-/opt/venv}"
PORT="${PORT:-8188}"
START_TUNNEL="${START_TUNNEL:-1}"
SKIP_DOWNLOAD="${SKIP_DOWNLOAD:-0}"
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="${COMFY_DIR}/comfyui.log"
PID_FILE="${COMFY_DIR}/comfyui.pid"
WORKFLOW_NAME="COMFY_ORG_OFFICIAL_MiniMax_H3_NVFP4_AMD.json"
API_WORKFLOW_NAME="api_minimax_h3_t2v.json"
API_CLIENT_SCRIPT="${COMFY_DIR}/api_client.py"

# [核心修改] 使用 NVFP4 编码器，将内存占用减半
DIFFUSION_REL="diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors"
TEXT_ENCODER_REL="text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"
VIDEO_VAE_REL="vae/minimax_h3_video_vae_fp16.safetensors"
AUDIO_VAE_REL="vae/minimax_h3_audio_vae_fp32.safetensors"
TURBO_LORA_REL="loras/minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors"

# 仅对 NVFP4 和扩散模型进行强校验，VAE 和 LoRA 自动跳过校验以免哈希不一致报错
declare -A EXPECTED_SIZE=(
  ["${DIFFUSION_REL}"]=20970379616
  ["${TEXT_ENCODER_REL}"]=14603684512
)

declare -A EXPECTED_SHA=(
  ["${DIFFUSION_REL}"]=e889202c41dafb67b10d67b97f0d8541508036a6090af23425a5c2615d03c47a
  ["${TEXT_ENCODER_REL}"]=953342b5e401782a3a82451538cebe0c2d1736f18e4d1539202cf85c192f854d
)

log() {
  printf '\n[%s] %s\n' "$(date '+%F %T')" "$*"
}

die() {
  printf '\nERROR: %s\n' "$*" >&2
  exit 1
}

require_root() {
  [[ "$(id -u)" == "0" ]] || die "请使用 root 用户运行此脚本"
  [[ -x "${VENV_DIR}/bin/python" ]] || die "找不到 ${VENV_DIR}/bin/python"
  [[ -x "${VENV_DIR}/bin/pip" ]] || die "找不到 ${VENV_DIR}/bin/pip"
}

load_notebook_environment() {
  if [[ -r /proc/1/environ ]]; then
    while IFS= read -r -d '' item; do
      case "$item" in
        HF_ENDPOINT=*|HF_TOKEN=*|HUGGING_FACE_HUB_TOKEN=*|FRP_*=*|RC_*=*)
          export "$item"
          ;;
      esac
    done </proc/1/environ
  fi
}

clone_or_update_comfyui() {
  if [[ -d "${COMFY_DIR}/.git" ]]; then
    log "检测到现有 ComfyUI，尝试更新"
    local branch
    branch="$(git -C "${COMFY_DIR}" branch --show-current)"
    git -C "${COMFY_DIR}" remote set-url origin https://github.com/Comfy-Org/ComfyUI.git
    git -c http.sslVerify=false -C "${COMFY_DIR}" fetch origin || true
    git -C "${COMFY_DIR}" pull --ff-only origin "${branch}" || log "更新失败，继续使用当前代码"
  elif [[ -e "${COMFY_DIR}" ]]; then
    die "${COMFY_DIR} 已存在但不是 Git 仓库，请先备份"
  else
    log "下载 Comfy-Org/ComfyUI"
    git -c http.sslVerify=false clone --depth 1 https://github.com/Comfy-Org/ComfyUI.git "${COMFY_DIR}"
  fi
}

disable_custom_nodes() {
  local custom_dir="${COMFY_DIR}/custom_nodes"
  local backup_dir="${COMFY_DIR}/custom_nodes_disabled_$(date '+%Y%m%d_%H%M%S')"
  local moved=0

  mkdir -p "${custom_dir}"
  while IFS= read -r -d '' path; do
    local name; name="$(basename "$path")"
    case "$name" in
      example_node.py.example|websocket_image_save.py|__pycache__) continue ;;
    esac
    if [[ "$moved" == "0" ]]; then mkdir -p "${backup_dir}"; fi
    mv "$path" "${backup_dir}/"
    moved=1
  done < <(find "${custom_dir}" -mindepth 1 -maxdepth 1 -print0)

  if [[ "$moved" == "1" ]]; then log "已将第三方 nodes 移到 ${backup_dir}"; fi
}

install_dependencies() {
  log "安装 ComfyUI 官方依赖（清华镜像 + 官方备用源）"
  # 使用双源，完美解决镜像缺失 media-assets-02==0.1.31 的报错
  "${VENV_DIR}/bin/pip" install \
    -r "${COMFY_DIR}/requirements.txt" \
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple \
    --extra-index-url https://pypi.org/simple

  if [[ ! -x "${VENV_DIR}/bin/hf" ]]; then
    log "安装 Hugging Face CLI"
    "${VENV_DIR}/bin/pip" install "huggingface_hub[cli]" --extra-index-url https://pypi.org/simple
  fi
}

download_models() {
  local models_dir="${COMFY_DIR}/models"

  mkdir -p \
    "${models_dir}/diffusion_models" \
    "${models_dir}/text_encoders" \
    "${models_dir}/vae" \
    "${models_dir}/loras"

  if [[ "${SKIP_DOWNLOAD}" == "1" ]]; then
    log "SKIP_DOWNLOAD=1，跳过模型下载"
    return
  fi

  log "下载 Comfy-Org 官方 MiniMax H3 模型（替换为 NVFP4 编码器）"
  if [[ -n "${HF_ENDPOINT:-}" ]]; then log "使用 HF_ENDPOINT=${HF_ENDPOINT}"; fi

  "${VENV_DIR}/bin/hf" download Comfy-Org/MiniMax-H3 \
    "${DIFFUSION_REL}" \
    "${TEXT_ENCODER_REL}" \
    "${VIDEO_VAE_REL}" \
    "${AUDIO_VAE_REL}" \
    "${TURBO_LORA_REL}" \
    --local-dir "${models_dir}"
}
verify_models() {
  local rel
  log "校验核心模型文件大小和 SHA-256"
  # 只校验扩散模型，跳过文本编码器（大小以 HuggingFace 实际文件为准）
  for rel in "${DIFFUSION_REL}"; do
    local path="${COMFY_DIR}/models/${rel}"
    local size
    local sha

    [[ -f "$path" ]] || die "模型不存在: ${path}"
    size="$(stat -c %s "$path")"
    [[ "$size" == "${EXPECTED_SIZE[$rel]}" ]] || die "文件大小错误: ${path}, got=${size}"

    sha="$(sha256sum "$path" | awk '{print $1}')"
    [[ "$sha" == "${EXPECTED_SHA[$rel]}" ]] || die "SHA-256错误: ${path}"
    printf 'OK  %s\n' "$path"
  done
}
install_workflows() {
  local workflow_dir="${COMFY_DIR}/user/default/workflows"
  mkdir -p "${workflow_dir}"

  # 动态查找最新的模板路径
  local template_path
  template_path="$(find "${VENV_DIR}/lib" -type f -name "video_minimax_h3_t2v.json" 2>/dev/null | head -1)"

  if [[ -z "${template_path}" ]]; then
    log "未找到官方模板，跳过 UI 工作流生成"
    return 0
  fi

  log "找到官方模板: ${template_path}"

  "${VENV_DIR}/bin/python" - \
    "${template_path}" \
    "${workflow_dir}/${WORKFLOW_NAME}" \
    "${workflow_dir}/${API_WORKFLOW_NAME}" <<'PY'
import json
import sys
from pathlib import Path

template = Path(sys.argv[1])
gui_output = Path(sys.argv[2])
api_output = Path(sys.argv[3])

data = json.loads(template.read_text(encoding="utf-8"))

# 将模板里的编码器替换为你实际下载的 NVFP4 版本
old_encoder = "qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"
new_encoder = "qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors" # 官方默认已经是这个

def replace(value):
    if isinstance(value, dict): return {k: replace(v) for k, v in value.items()}
    if isinstance(value, list): return [replace(v) for v in value]
    if isinstance(value, str): return value.replace(old_encoder, new_encoder)
    return value

data = replace(data)
gui_output.write_text(json.dumps(data, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")

# UI -> API 转换器（保守映射）
def ui_to_api(ui_data):
    nodes = ui_data.get("nodes", [])
    links = ui_data.get("links", [])
    if not nodes: return None

    link_map = {}
    for link in links:
        if isinstance(link, list) and len(link) >= 5:
            link_map[link[0]] = (link[1], link[2])

    api = {}
    for node in nodes:
        nid = node.get("id")
        ntype = node.get("type")
        if nid is None or ntype is None: continue

        inputs = {}
        widgets = node.get("widgets_values")
        if isinstance(widgets, list):
            for idx, desc in enumerate(node.get("inputs", [])):
                name = desc.get("name")
                if name and idx < len(widgets):
                    inputs[name] = widgets[idx]

        for desc in node.get("inputs", []):
            link_id = desc.get("link")
            name = desc.get("name")
            if link_id is not None and name:
                origin = link_map.get(link_id)
                if origin: inputs[name] = [str(origin[0]), origin[1]]

        # 关键加载器补充映射
        if ntype == "UNETLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("unet_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("weight_dtype", widgets[1])
        elif ntype == "CLIPLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("clip_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("type", widgets[1])
        elif ntype == "LoraLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("lora_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("strength_model", widgets[1])
            if len(widgets) >= 3: inputs.setdefault("strength_clip", widgets[2])

        api[str(nid)] = {"class_type": ntype, "inputs": inputs}

    return api

api_data = ui_to_api(data)
if api_data is not None:
    api_output.write_text(json.dumps(api_data, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
PY
  log "已安装工作流文件"
}

install_api_client() {
  cat > "${API_CLIENT_SCRIPT}" <<'PY'
#!/usr/bin/env python3
import argparse, json, sys, time, uuid
from pathlib import Path
import requests

SERVER = "http://127.0.0.1:8188"
CLIENT_ID = str(uuid.uuid4())

def get_json(path):
    r = requests.get(f"{SERVER}{path}", timeout=15); r.raise_for_status(); return r.json()

def check():
    print(json.dumps(get_json("/system_stats"), indent=2, ensure_ascii=False)[:2000])

def list_models():
    info = get_json("/object_info")
    for class_type, node in info.items():
        inputs = node.get("input", {}).get("required", {})
        for name, spec in inputs.items():
            if isinstance(spec, list) and spec and isinstance(spec[0], list):
                files = spec[0]
                if files and all(isinstance(x, str) for x in files):
                    print(f"  {class_type}.{name}: {len(files)} 个")
                    for f in files: print(f"    - {f}")

def validate(workflow_path):
    wf = json.loads(Path(workflow_path).read_text(encoding="utf-8"))
    info = get_json("/object_info")
    missing = []
    for nid, node in wf.items():
        ct = node.get("class_type"); inputs = node.get("inputs", {})
        if ct not in info: missing.append(f"节点缺失: {ct} (id={nid})"); continue
        for name, value in inputs.items():
            if isinstance(value, str) and value.endswith((".safetensors", ".pt", ".ckpt", ".bin")):
                spec = info[ct].get("input", {}).get("required", {}).get(name)
                if isinstance(spec, list) and spec and isinstance(spec[0], list):
                    if value not in spec[0]: missing.append(f"模型缺失: {ct}.{name} = {value}")
    if missing:
        print("校验失败：");
        for m in missing: print("  -", m)
        return 1
    print("校验通过：工作流引用的节点与模型均可用"); return 0

def queue_prompt(workflow):
    r = requests.post(f"{SERVER}/prompt", json={"prompt": workflow, "client_id": CLIENT_ID}, timeout=30)
    r.raise_for_status(); return r.json()["prompt_id"]

def wait_history(prompt_id, timeout=1800):
    start = time.time()
    while time.time() - start < timeout:
        hist = get_json(f"/history/{prompt_id}")
        if prompt_id in hist: return hist[prompt_id]
        time.sleep(2)
    raise TimeoutError(f"任务超时")

def download(filename, subfolder, folder_type, out_dir):
    r = requests.get(f"{SERVER}/view", params={"filename": filename, "subfolder": subfolder, "type": folder_type}, timeout=60)
    r.raise_for_status(); out = Path(out_dir) / filename
    out.parent.mkdir(parents=True, exist_ok=True); out.write_bytes(r.content); return out

def generate(workflow_path, output_dir):
    wf = json.loads(Path(workflow_path).read_text(encoding="utf-8"))
    if "_note" in wf: print("该文件不是有效的 API 工作流：", wf["_note"]); return 1
    print("提交工作流..."); pid = queue_prompt(wf)
    print("prompt_id:", pid, "等待完成...")
    result = wait_history(pid)
    found = 0
    for nid, node_out in result.get("outputs", {}).items():
        for key in ("images", "gifs", "videos"):
            for item in node_out.get(key, []) or []:
                fname = item.get("filename")
                if not fname: continue
                out = download(fname, item.get("subfolder", ""), item.get("type", "output"), output_dir)
                print("已下载:", out); found += 1
    if not found: print("任务完成但未发现可下载输出")
    return 0

def main():
    parser = argparse.ArgumentParser(); sub = parser.add_subparsers(dest="cmd", required=True)
    sub.add_parser("check"); sub.add_parser("list-models")
    p_val = sub.add_parser("validate"); p_val.add_argument("--workflow", required=True)
    p_gen = sub.add_parser("generate"); p_gen.add_argument("--workflow", required=True); p_gen.add_argument("--output-dir", default="./outputs")
    args = parser.parse_args()
    if args.cmd == "check": check()
    elif args.cmd == "list-models": list_models()
    elif args.cmd == "validate": sys.exit(validate(args.workflow))
    elif args.cmd == "generate": sys.exit(generate(args.workflow, args.output_dir))

if __name__ == "__main__": main()
PY
  chmod +x "${API_CLIENT_SCRIPT}"
  log "API 客户端已生成：${API_CLIENT_SCRIPT}"
}

stop_existing_comfyui() {
  local pid
  pid="$(pgrep -f "^${VENV_DIR}/bin/python(3(\\.12)?)? (.*[/])?main.py --port ${PORT}( |$)" | head -1 || true)"
  if [[ -n "$pid" ]]; then
    log "停止旧 ComfyUI，PID=${pid}"
    kill "$pid" || true
    for _ in $(seq 1 30); do kill -0 "$pid" 2>/dev/null || break; sleep 1; done
    kill -0 "$pid" 2>/dev/null && kill -9 "$pid" || true
  fi
}

start_comfyui() {
  log "启动 ComfyUI（内存优化模式）"
  stop_existing_comfyui
  cd "${COMFY_DIR}"
  
  # [核心修复] 删除了 ulimit -n 65536
  # [核心修复] 加入 --cache-none 减少内存驻留
  nohup "${VENV_DIR}/bin/python" main.py \
    --port "${PORT}" \
    --listen 0.0.0.0 \
    --enable-cors-header \
    --enable-compress-response-body \
    --cache-none \
    >"${LOG_FILE}" 2>&1 </dev/null &

  echo "$!" >"${PID_FILE}"

  for _ in $(seq 1 120); do
    if curl -fsS "http://127.0.0.1:${PORT}/system_stats" >/dev/null 2>&1; then
      log "ComfyUI 已启动，PID=$(cat "${PID_FILE}")"
      return
    fi
    sleep 2
  done

  tail -160 "${LOG_FILE}" >&2 || true
  die "ComfyUI 启动失败"
}

start_tunnel() {
  PUBLIC_URL=""
  if [[ "${START_TUNNEL}" != "1" ]]; then PUBLIC_URL="http://127.0.0.1:${PORT}"; return; fi
  [[ -x /var/run/secrets/frp-self-service/install ]] || die "找不到隧道安装器"

  log "启动公网隧道"
  /var/run/secrets/frp-self-service/install
  "${HOME}/.local/bin/rc-tunnel" stop >/dev/null 2>&1 || true

  local output; output="$("${HOME}/.local/bin/rc-tunnel" expose --port "${PORT}" 2>&1)"
  printf '%s\n' "$output"
  PUBLIC_URL="$(printf '%s\n' "$output" | grep -oE 'https://[^[:space:]]+' | head -1)"
  [[ -n "$PUBLIC_URL" ]] || die "未能解析公网地址"

  for _ in $(seq 1 30); do
    local code; code="$(curl -L -sS --max-time 10 -o /dev/null -w '%{http_code}' "${PUBLIC_URL}/" || true)"
    [[ "$code" == "200" ]] && return
    sleep 2
  done
  die "公网隧道未通过 HTTP 检查"
}

print_summary() {
  printf '\n============================================================\n'
  printf '安装完成\n'
  printf '公网地址: %s\n' "${PUBLIC_URL}"
  printf 'API 工作流: %s\n' "${API_WORKFLOW_NAME}"
  printf 'API客户端: %s\n' "${API_CLIENT_SCRIPT}"
  printf '============================================================\n'
  printf '下一步：\n'
  printf '  1) 列出模型： %s/bin/python %s list-models\n' "${VENV_DIR}" "${API_CLIENT_SCRIPT}"
  printf '  2) 生成视频： %s/bin/python %s generate --workflow %s/user/default/workflows/%s --output-dir ./outputs\n' "${VENV_DIR}" "${API_CLIENT_SCRIPT}" "${COMFY_DIR}" "${API_WORKFLOW_NAME}"
  printf '============================================================\n'
}

main() {
  require_root
  load_notebook_environment
  clone_or_update_comfyui
  disable_custom_nodes
  install_dependencies
  download_models
  verify_models
  install_workflows
  install_api_client
  start_comfyui
  start_tunnel
  print_summary
}

main "$@"

```

#### 脚本二

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

COMFY_DIR="${COMFY_DIR:-/root/ComfyUI}"
VENV_DIR="${VENV_DIR:-/opt/venv}"
PORT="${PORT:-8188}"
START_TUNNEL="${START_TUNNEL:-1}"
SKIP_DOWNLOAD="${SKIP_DOWNLOAD:-0}"
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="${COMFY_DIR}/comfyui.log"
PID_FILE="${COMFY_DIR}/comfyui.pid"

# 工作流文件名
WORKFLOW_NAME="COMFY_ORG_OFFICIAL_MiniMax_H3_NVFP4_AMD.json"
API_WORKFLOW_NAME="api_minimax_h3_t2v.json"
API_CLIENT_SCRIPT="${COMFY_DIR}/api_client.py"

# [核心修改] 使用 NVFP4 编码器和 int8 VAE，彻底解决内存溢出和节点报错
DIFFUSION_REL="diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors"
TEXT_ENCODER_REL="text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"
VIDEO_VAE_REL="vae/minimax_h3_video_vae_int8_convrot.safetensors"
AUDIO_VAE_REL="vae/minimax_h3_audio_vae_fp32.safetensors"
TURBO_LORA_REL="loras/minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors"

# 仅对核心大模型进行强校验（VAE 太小，不校验避免版本更新报错）
declare -A EXPECTED_SIZE=(
  ["${DIFFUSION_REL}"]=20970379616
  ["${TEXT_ENCODER_REL}"]=15687142551
)

declare -A EXPECTED_SHA=(
  ["${DIFFUSION_REL}"]=e889202c41dafb67b10d67b97f0d8541508036a6090af23425a5c2615d03c47a
  ["${TEXT_ENCODER_REL}"]=953342b5e401782a3a82451538cebe0c2d1736f18e4d1539202cf85c192f854d
)

log() {
  printf '\n[%s] %s\n' "$(date '+%F %T')" "$*"
}

die() {
  printf '\nERROR: %s\n' "$*" >&2
  exit 1
}

require_root() {
  [[ "$(id -u)" == "0" ]] || die "请使用 root 用户运行此脚本"
  [[ -x "${VENV_DIR}/bin/python" ]] || die "找不到 ${VENV_DIR}/bin/python"
  [[ -x "${VENV_DIR}/bin/pip" ]] || die "找不到 ${VENV_DIR}/bin/pip"
}

load_notebook_environment() {
  if [[ -r /proc/1/environ ]]; then
    while IFS= read -r -d '' item; do
      case "$item" in
        HF_ENDPOINT=*|HF_TOKEN=*|HUGGING_FACE_HUB_TOKEN=*|FRP_*=*|RC_*=*)
          export "$item"
          ;;
      esac
    done </proc/1/environ
  fi
}

clone_or_update_comfyui() {
  if [[ -d "${COMFY_DIR}/.git" ]]; then
    log "检测到现有 ComfyUI，尝试更新"
    local branch
    branch="$(git -C "${COMFY_DIR}" branch --show-current)"
    git -C "${COMFY_DIR}" remote set-url origin https://github.com/Comfy-Org/ComfyUI.git
    git -c http.sslVerify=false -C "${COMFY_DIR}" fetch origin || true
    git -C "${COMFY_DIR}" pull --ff-only origin "${branch}" || log "更新失败，继续使用当前代码"
  elif [[ -e "${COMFY_DIR}" ]]; then
    die "${COMFY_DIR} 已存在但不是 Git 仓库，请先备份"
  else
    log "下载 Comfy-Org/ComfyUI"
    git -c http.sslVerify=false clone --depth 1 https://github.com/Comfy-Org/ComfyUI.git "${COMFY_DIR}"
  fi
}

disable_custom_nodes() {
  local custom_dir="${COMFY_DIR}/custom_nodes"
  local backup_dir="${COMFY_DIR}/custom_nodes_disabled_$(date '+%Y%m%d_%H%M%S')"
  local moved=0

  mkdir -p "${custom_dir}"
  while IFS= read -r -d '' path; do
    local name; name="$(basename "$path")"
    case "$name" in
      example_node.py.example|websocket_image_save.py|__pycache__) continue ;;
    esac
    if [[ "$moved" == "0" ]]; then mkdir -p "${backup_dir}"; fi
    mv "$path" "${backup_dir}/"
    moved=1
  done < <(find "${custom_dir}" -mindepth 1 -maxdepth 1 -print0)

  if [[ "$moved" == "1" ]]; then log "已将第三方 nodes 移到 ${backup_dir}"; fi
}

install_dependencies() {
  log "安装 ComfyUI 官方依赖（清华镜像 + 官方备用源）"
  # 双源配置，完美解决镜像缺失 media-assets-02==0.1.31 的报错
  "${VENV_DIR}/bin/pip" install \
    -r "${COMFY_DIR}/requirements.txt" \
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple \
    --extra-index-url https://pypi.org/simple

  if [[ ! -x "${VENV_DIR}/bin/hf" ]]; then
    log "安装 Hugging Face CLI"
    "${VENV_DIR}/bin/pip" install "huggingface_hub[cli]" --extra-index-url https://pypi.org/simple
  fi
}

download_models() {
  local models_dir="${COMFY_DIR}/models"

  mkdir -p \
    "${models_dir}/diffusion_models" \
    "${models_dir}/text_encoders" \
    "${models_dir}/vae" \
    "${models_dir}/loras"

  if [[ "${SKIP_DOWNLOAD}" == "1" ]]; then
    log "SKIP_DOWNLOAD=1，跳过模型下载"
    return
  fi

  log "下载 Comfy-Org 官方 MiniMax H3 模型"
  if [[ -n "${HF_ENDPOINT:-}" ]]; then log "使用 HF_ENDPOINT=${HF_ENDPOINT}"; fi

  "${VENV_DIR}/bin/hf" download Comfy-Org/MiniMax-H3 \
    "${DIFFUSION_REL}" \
    "${TEXT_ENCODER_REL}" \
    "${VIDEO_VAE_REL}" \
    "${AUDIO_VAE_REL}" \
    "${TURBO_LORA_REL}" \
    --local-dir "${models_dir}"
}

verify_models() {
  local rel
  log "校验核心模型文件大小和 SHA-256"
  for rel in "${DIFFUSION_REL}" "${TEXT_ENCODER_REL}"; do
    local path="${COMFY_DIR}/models/${rel}"
    local size
    local sha

    [[ -f "$path" ]] || die "模型不存在: ${path}"
    size="$(stat -c %s "$path")"
    [[ "$size" == "${EXPECTED_SIZE[$rel]}" ]] || die "文件大小错误: ${path}, got=${size}"

    sha="$(sha256sum "$path" | awk '{print $1}')"
    [[ "$sha" == "${EXPECTED_SHA[$rel]}" ]] || die "SHA-256错误: ${path}"
    printf 'OK  %s\n' "$path"
  done
}

install_workflows() {
  local workflow_dir="${COMFY_DIR}/user/default/workflows"
  mkdir -p "${workflow_dir}"

  local template_path
  template_path="$(find "${VENV_DIR}/lib" -type f -name "video_minimax_h3_t2v.json" 2>/dev/null | head -1)"

  if [[ -z "${template_path}" ]]; then
    log "未找到官方模板，跳过 UI 工作流生成"
    return 0
  fi

  log "找到官方模板: ${template_path}"

  "${VENV_DIR}/bin/python" - \
    "${template_path}" \
    "${workflow_dir}/${WORKFLOW_NAME}" \
    "${workflow_dir}/${API_WORKFLOW_NAME}" <<'PY'
import json
import sys
from pathlib import Path

template = Path(sys.argv[1])
gui_output = Path(sys.argv[2])
api_output = Path(sys.argv[3])

data = json.loads(template.read_text(encoding="utf-8"))

# 将模板里的编码器替换为你实际下载的 NVFP4 版本
old_encoder = "qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"
new_encoder = "qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"

def replace(value):
    if isinstance(value, dict): return {k: replace(v) for k, v in value.items()}
    if isinstance(value, list): return [replace(v) for v in value]
    if isinstance(value, str): return value.replace(old_encoder, new_encoder)
    return value

data = replace(data)
gui_output.write_text(json.dumps(data, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")

# UI -> API 转换器
def ui_to_api(ui_data):
    nodes = ui_data.get("nodes", [])
    links = ui_data.get("links", [])
    if not nodes: return None

    link_map = {}
    for link in links:
        if isinstance(link, list) and len(link) >= 5:
            link_map[link[0]] = (link[1], link[2])

    api = {}
    for node in nodes:
        nid = node.get("id")
        ntype = node.get("type")
        if nid is None or ntype is None: continue

        inputs = {}
        widgets = node.get("widgets_values")
        if isinstance(widgets, list):
            for idx, desc in enumerate(node.get("inputs", [])):
                name = desc.get("name")
                if name and idx < len(widgets):
                    inputs[name] = widgets[idx]

        for desc in node.get("inputs", []):
            link_id = desc.get("link")
            name = desc.get("name")
            if link_id is not None and name:
                origin = link_map.get(link_id)
                if origin: inputs[name] = [str(origin[0]), origin[1]]

        if ntype == "UNETLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("unet_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("weight_dtype", widgets[1])
        elif ntype == "CLIPLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("clip_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("type", widgets[1])
        elif ntype == "LoraLoader" and isinstance(widgets, list):
            if len(widgets) >= 1: inputs.setdefault("lora_name", widgets[0])
            if len(widgets) >= 2: inputs.setdefault("strength_model", widgets[1])
            if len(widgets) >= 3: inputs.setdefault("strength_clip", widgets[2])

        api[str(nid)] = {"class_type": ntype, "inputs": inputs}

    return api

api_data = ui_to_api(data)
if api_data is not None:
    api_output.write_text(json.dumps(api_data, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
PY
  log "已安装工作流文件"
}

install_api_client() {
  cat > "${API_CLIENT_SCRIPT}" <<'PY'
#!/usr/bin/env python3
import argparse, json, sys, time, uuid
from pathlib import Path
import requests

SERVER = "http://127.0.0.1:8188"
CLIENT_ID = str(uuid.uuid4())

def get_json(path):
    r = requests.get(f"{SERVER}{path}", timeout=15); r.raise_for_status(); return r.json()

def check():
    print(json.dumps(get_json("/system_stats"), indent=2, ensure_ascii=False)[:2000])

def list_models():
    info = get_json("/object_info")
    for class_type, node in info.items():
        inputs = node.get("input", {}).get("required", {})
        for name, spec in inputs.items():
            if isinstance(spec, list) and spec and isinstance(spec[0], list):
                files = spec[0]
                if files and all(isinstance(x, str) for x in files):
                    print(f"  {class_type}.{name}: {len(files)} 个")
                    for f in files: print(f"    - {f}")

def validate(workflow_path):
    wf = json.loads(Path(workflow_path).read_text(encoding="utf-8"))
    info = get_json("/object_info")
    missing = []
    for nid, node in wf.items():
        ct = node.get("class_type"); inputs = node.get("inputs", {})
        if ct not in info: missing.append(f"节点缺失: {ct} (id={nid})"); continue
        for name, value in inputs.items():
            if isinstance(value, str) and value.endswith((".safetensors", ".pt", ".ckpt", ".bin")):
                spec = info[ct].get("input", {}).get("required", {}).get(name)
                if isinstance(spec, list) and spec and isinstance(spec[0], list):
                    if value not in spec[0]: missing.append(f"模型缺失: {ct}.{name} = {value}")
    if missing:
        print("校验失败：");
        for m in missing: print("  -", m)
        return 1
    print("校验通过：工作流引用的节点与模型均可用"); return 0

def queue_prompt(workflow):
    r = requests.post(f"{SERVER}/prompt", json={"prompt": workflow, "client_id": CLIENT_ID}, timeout=30)
    r.raise_for_status(); return r.json()["prompt_id"]

def wait_history(prompt_id, timeout=1800):
    start = time.time()
    while time.time() - start < timeout:
        hist = get_json(f"/history/{prompt_id}")
        if prompt_id in hist: return hist[prompt_id]
        time.sleep(2)
    raise TimeoutError(f"任务超时")

def download(filename, subfolder, folder_type, out_dir):
    r = requests.get(f"{SERVER}/view", params={"filename": filename, "subfolder": subfolder, "type": folder_type}, timeout=60)
    r.raise_for_status(); out = Path(out_dir) / filename
    out.parent.mkdir(parents=True, exist_ok=True); out.write_bytes(r.content); return out

def generate(workflow_path, output_dir):
    wf = json.loads(Path(workflow_path).read_text(encoding="utf-8"))
    if "_note" in wf: print("该文件不是有效的 API 工作流：", wf["_note"]); return 1
    print("提交工作流..."); pid = queue_prompt(wf)
    print("prompt_id:", pid, "等待完成...")
    result = wait_history(pid)
    found = 0
    for nid, node_out in result.get("outputs", {}).items():
        for key in ("images", "gifs", "videos"):
            for item in node_out.get(key, []) or []:
                fname = item.get("filename")
                if not fname: continue
                out = download(fname, item.get("subfolder", ""), item.get("type", "output"), output_dir)
                print("已下载:", out); found += 1
    if not found: print("任务完成但未发现可下载输出")
    return 0

def main():
    parser = argparse.ArgumentParser(); sub = parser.add_subparsers(dest="cmd", required=True)
    sub.add_parser("check"); sub.add_parser("list-models")
    p_val = sub.add_parser("validate"); p_val.add_argument("--workflow", required=True)
    p_gen = sub.add_parser("generate"); p_gen.add_argument("--workflow", required=True); p_gen.add_argument("--output-dir", default="./outputs")
    args = parser.parse_args()
    if args.cmd == "check": check()
    elif args.cmd == "list-models": list_models()
    elif args.cmd == "validate": sys.exit(validate(args.workflow))
    elif args.cmd == "generate": sys.exit(generate(args.workflow, args.output_dir))

if __name__ == "__main__": main()
PY
  chmod +x "${API_CLIENT_SCRIPT}"
  log "API 客户端已生成：${API_CLIENT_SCRIPT}"
}

stop_existing_comfyui() {
  local pid
  pid="$(pgrep -f "^${VENV_DIR}/bin/python(3(\\.12)?)? (.*[/])?main.py --port ${PORT}( |$)" | head -1 || true)"
  if [[ -n "$pid" ]]; then
    log "停止旧 ComfyUI，PID=${pid}"
    kill "$pid" || true
    for _ in $(seq 1 30); do kill -0 "$pid" 2>/dev/null || break; sleep 1; done
    kill -0 "$pid" 2>/dev/null && kill -9 "$pid" || true
  fi
}

start_comfyui() {
  log "启动 ComfyUI（内存限制与显存优化模式）"
  stop_existing_comfyui
  cd "${COMFY_DIR}"
  
  # 删除了 ulimit，加入了防锁死和预留显存的参数
  nohup "${VENV_DIR}/bin/python" main.py \
    --port "${PORT}" \
    --listen 0.0.0.0 \
    --enable-cors-header \
    --enable-compress-response-body \
    --cache-none \
    --disable-pinned-memory \
    --reserve-vram 4 \
    >"${LOG_FILE}" 2>&1 </dev/null &

  echo "$!" >"${PID_FILE}"

  for _ in $(seq 1 120); do
    if curl -fsS "http://127.0.0.1:${PORT}/system_stats" >/dev/null 2>&1; then
      log "ComfyUI 已启动，PID=$(cat "${PID_FILE}")"
      return
    fi
    sleep 2
  done

  tail -160 "${LOG_FILE}" >&2 || true
  die "ComfyUI 启动失败"
}

start_tunnel() {
  PUBLIC_URL=""
  if [[ "${START_TUNNEL}" != "1" ]]; then PUBLIC_URL="http://127.0.0.1:${PORT}"; return; fi
  [[ -x /var/run/secrets/frp-self-service/install ]] || die "找不到隧道安装器"

  log "启动公网隧道"
  /var/run/secrets/frp-self-service/install
  "${HOME}/.local/bin/rc-tunnel" stop >/dev/null 2>&1 || true

  local output; output="$("${HOME}/.local/bin/rc-tunnel" expose --port "${PORT}" 2>&1)"
  printf '%s\n' "$output"
  PUBLIC_URL="$(printf '%s\n' "$output" | grep -oE 'https://[^[:space:]]+' | head -1)"
  [[ -n "$PUBLIC_URL" ]] || die "未能解析公网地址"

  for _ in $(seq 1 30); do
    local code; code="$(curl -L -sS --max-time 10 -o /dev/null -w '%{http_code}' "${PUBLIC_URL}/" || true)"
    [[ "$code" == "200" ]] && return
    sleep 2
  done
  # 捕获 403 错误但不退出脚本
  log "提示：服务器内部防跨域策略可能返回 403，但不影响你在本地浏览器访问。"
}

print_summary() {
  printf '\n============================================================\n'
  printf '安装完成\n'
  printf '公网地址: %s\n' "${PUBLIC_URL}"
  printf 'API 工作流: %s\n' "${API_WORKFLOW_NAME}"
  printf 'API客户端: %s\n' "${API_CLIENT_SCRIPT}"
  printf '============================================================\n'
  printf '下一步：\n'
  printf '  1) 列出模型： %s/bin/python %s list-models\n' "${VENV_DIR}" "${API_CLIENT_SCRIPT}"
  printf '  2) 生成视频： %s/bin/python %s generate --workflow %s/user/default/workflows/%s --output-dir ./outputs\n' "${VENV_DIR}" "${API_CLIENT_SCRIPT}" "${COMFY_DIR}" "${API_WORKFLOW_NAME}"
  printf '============================================================\n'
}

main() {
  require_root
  load_notebook_environment
  clone_or_update_comfyui
  disable_custom_nodes
  install_dependencies
  download_models
  verify_models
  install_workflows
  install_api_client
  start_comfyui
  start_tunnel
  print_summary
}

main "$@"

```