# 硅基流动 Embedding 和 Rerank 支持实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 daily-paper-reader 项目添加硅基流动 (SiliconFlow) API 的 embedding 和 rerank 模型支持

**Architecture:** 在现有代码基础上扩展，保持向后兼容性，通过环境变量切换提供商。为 SiliconflowClient 添加 rerank 方法，为 model_loader 添加硅基流动 embedding 支持，修改 3.rank_papers.py 支持多提供商。

**Tech Stack:** Python, requests, 硅基流动 OpenAI 兼容 API

---

## 文件结构

| 文件 | 操作 | 职责 |
|------|------|------|
| `src/llm.py` | 修改 | 为 SiliconflowClient 添加 rerank() 方法 |
| `src/model_loader.py` | 修改 | 添加硅基流动 embedding 支持 |
| `src/3.rank_papers.py` | 修改 | 支持多提供商 rerank 客户端 |

---

### Task 1: 为 SiliconflowClient 添加 rerank 支持

**Files:**
- Modify: `src/llm.py:582-584`

**背景:** 当前 SiliconflowClient 只继承了基础 rerank 方法（抛出 NotImplementedError），需要添加与 BltClient 类似的实现。

- [ ] **Step 1: 在 SiliconflowClient 类中添加 rerank() 方法**

在 `src/llm.py` 的 `SiliconflowClient` 类（第 582-584 行）后添加：

```python
class SiliconflowClient(LLMClient):
    def __init__(self, api_key: str, model: str, base_url: str = "https://api.siliconflow.cn/v1"):
        super().__init__(api_key=api_key, model=model, base_url=base_url)

    def rerank(
        self,
        query: str,
        documents: List[str],
        top_n: Optional[int] = None,
        model: Optional[str] = None,
    ) -> dict:
        """
        调用硅基流动 Rerank 接口（/v1/rerank）。

        :param query: 查询文本
        :param documents: 待排序文档列表
        :param top_n: 返回的 Top N（可选）
        :param model: 重排模型名（可选，默认使用 self.model）
        """
        if not query:
            raise ValueError("rerank: query 不能为空")
        if not documents:
            raise ValueError("rerank: documents 不能为空")

        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }
        payload: Dict[str, Any] = {
            "model": model or self.model,
            "query": query,
            "documents": documents,
        }
        if top_n is not None:
            payload["top_n"] = int(top_n)

        request_bases = self._iter_retry_bases(total_attempts=6)
        last_error: Exception | None = None
        for attempt_idx, req_base in enumerate(request_bases, start=1):
            request_url = f"{req_base.rstrip('/')}/rerank"
            try:
                response = requests.post(request_url, headers=headers, json=payload, timeout=120)
                response.raise_for_status()
                try:
                    response_data = response.json()
                except ValueError:
                    print("Rerank 响应无法解析为 JSON，原始文本预览:", response.text[:500])
                    raise

                if isinstance(response_data, dict) and 'error' in response_data:
                    err = response_data.get('error') or {}
                    print("Rerank 返回错误:", {
                        'type': err.get('type'),
                        'code': err.get('code'),
                        'message': err.get('message') or err,
                    })
                    raise requests.exceptions.HTTPError(f"Rerank API error: {err}")

                return response_data
            except Exception as e:
                last_error = e
                if attempt_idx < len(request_bases):
                    next_base = request_bases[attempt_idx] if attempt_idx < len(request_bases) else ''
                    print(
                        f"Rerank 请求失败（base={req_base}，第 {attempt_idx} 次），"
                        f"将回退到 {next_base}"
                    )
                    continue
                print(f"通过 requests 调用 Rerank API 时出错: {e}")
                print("Rerank 请求摘要:", {
                    "url": request_url,
                    "model": payload.get("model"),
                    "query_len": len(query or ""),
                    "documents": len(documents),
                    "top_n": payload.get("top_n"),
                })
                if hasattr(e, "response") and e.response is not None:
                    try:
                        print("错误详情(JSON):", e.response.json())
                    except ValueError:
                        try:
                            print("错误详情(TEXT):", e.response.text[:500])
                        except Exception:
                            pass
                else:
                    print("错误详情: 未收到服务端响应（可能是网络/SSL问题）。")
                raise

        if last_error is not None:
            raise last_error
        raise RuntimeError("rerank 未命中可用 base")
```

- [ ] **Step 2: 验证代码语法**

Run: `python -m py_compile src/llm.py`
Expected: 无错误输出

- [ ] **Step 3: Commit**

```bash
git add src/llm.py
git commit -m "feat: add rerank support to SiliconflowClient"
```

---

### Task 2: 修改 model_loader.py 添加硅基流动 embedding 支持

**Files:**
- Modify: `src/model_loader.py:17-24` (添加常量)
- Modify: `src/model_loader.py:319-367` (修改 load_sentence_transformer)

**背景:** 当前只支持自定义远程 embedding 端点，需要添加硅基流动 API 支持。

- [ ] **Step 1: 添加硅基流动配置常量**

在 `src/model_loader.py` 第 17-24 行后添加：

```python
_DEFAULT_SILICONFLOW_EMBED_ENDPOINT = "https://api.siliconflow.cn/v1"
```

- [ ] **Step 2: 修改 load_sentence_transformer 函数**

替换 `src/model_loader.py` 的 `load_sentence_transformer` 函数（第 319-367 行）：

```python
def load_sentence_transformer(
  model_name: str,
  *,
  device: str,
  allow_remote: bool = True,
  retries: int | None = None,
  log: Callable[[str], None] = _log_default,
  providers: tuple[tuple[str, str], ...] = (
    ("huggingface", HUGGINGFACE_ENDPOINT),
    ("modelscope", MODELSCOPE_ENDPOINT),
  ),
):
  embed_provider = os.getenv("EMBED_PROVIDER", "custom").strip().lower()

  if embed_provider == "siliconflow":
    siliconflow_api_key = os.getenv("SILICONFLOW_API_KEY", "").strip()
    siliconflow_model = os.getenv("SILICONFLOW_EMBED_MODEL", model_name).strip()
    siliconflow_endpoint = os.getenv("SILICONFLOW_EMBED_ENDPOINT", _DEFAULT_SILICONFLOW_EMBED_ENDPOINT).strip()
    remote_timeout_text = os.getenv("DPR_EMBED_API_TIMEOUT", str(_DEFAULT_REMOTE_TIMEOUT_SECONDS))
    try:
      remote_timeout = int(remote_timeout_text)
    except ValueError:
      log(
        f"[WARN] 环境变量 DPR_EMBED_API_TIMEOUT 无效：{remote_timeout_text}，"
        f"回退默认 {_DEFAULT_REMOTE_TIMEOUT_SECONDS}"
      )
      remote_timeout = _DEFAULT_REMOTE_TIMEOUT_SECONDS
    log(
      f"[INFO] 使用硅基流动 embedding 服务：model={siliconflow_model} "
      f"endpoint={str(siliconflow_endpoint).strip()} timeout={remote_timeout}s device={device}"
    )
    return SiliconFlowSentenceTransformer(
      model_name=siliconflow_model,
      endpoint=str(siliconflow_endpoint).strip(),
      api_key=siliconflow_api_key,
      timeout=remote_timeout,
      local_device=device,
      local_retries=retries,
      local_providers=providers,
      log=log,
    )

  remote_endpoint = _DEFAULT_REMOTE_EMBED_ENDPOINT
  remote_api_key = _DEFAULT_REMOTE_EMBED_API_KEY
  if allow_remote and remote_endpoint:
    remote_timeout_text = os.getenv("DPR_EMBED_API_TIMEOUT", str(_DEFAULT_REMOTE_TIMEOUT_SECONDS))
    try:
      remote_timeout = int(remote_timeout_text)
    except ValueError:
      log(
        f"[WARN] 环境变量 DPR_EMBED_API_TIMEOUT 无效：{remote_timeout_text}，"
        f"回退默认 {_DEFAULT_REMOTE_TIMEOUT_SECONDS}"
      )
      remote_timeout = _DEFAULT_REMOTE_TIMEOUT_SECONDS
    log(
      f"[INFO] 使用远程 embedding 服务：model={model_name} "
      f"endpoint={str(remote_endpoint).strip()} timeout={remote_timeout}s device={device}"
    )
    return RemoteSentenceTransformer(
      model_name=model_name,
      endpoint=str(remote_endpoint).strip(),
      api_key=remote_api_key,
      timeout=remote_timeout,
      local_device=device,
      local_retries=retries,
      local_providers=providers,
      log=log,
    )

  if remote_endpoint and not allow_remote:
    log(f"[INFO] 已禁用远程 embedding，强制使用本地模型：{model_name} (device={device})")

  return _load_local_sentence_transformer(
    model_name,
    device=device,
    retries=retries,
    log=log,
    providers=providers,
  )
```

- [ ] **Step 3: 添加 SiliconFlowSentenceTransformer 类**

在 `src/model_loader.py` 的 `RemoteSentenceTransformer` 类后（第 254 行后）添加：

```python
class SiliconFlowSentenceTransformer:
  """兼容 SentenceTransformer.encode 接口的硅基流动 embedding 包装器。"""

  is_remote = True

  def __init__(
    self,
    model_name: str,
    endpoint: str,
    api_key: str = "",
    timeout: int = _DEFAULT_REMOTE_TIMEOUT_SECONDS,
    default_batch_size: int = 8,
    local_device: str = "cpu",
    local_retries: int | None = None,
    local_providers: tuple[tuple[str, str], ...] = (
      ("huggingface", HUGGINGFACE_ENDPOINT),
      ("modelscope", MODELSCOPE_ENDPOINT),
    ),
    log: Callable[[str], None] = _log_default,
  ):
    self.model_name = model_name
    self.endpoint = self._normalize_endpoint(endpoint)
    self.api_key = str(api_key or "").strip()
    self.timeout = max(int(timeout or _DEFAULT_REMOTE_TIMEOUT_SECONDS), 1)
    self.default_batch_size = max(int(default_batch_size or 1), 1)
    self.max_seq_length = None
    self.local_device = str(local_device or "cpu")
    self.local_retries = local_retries
    self.local_providers = local_providers
    self._local_model = None
    self._log = log
    self._remote_available = True
    self._remote_disabled_reason = ""

  @staticmethod
  def _normalize_endpoint(endpoint: str) -> str:
    text = str(endpoint or "").strip().rstrip("/")
    if not text:
      raise ValueError("硅基流动 embedding 服务地址不能为空")
    if text.endswith("/embeddings"):
      return text
    return f"{text}/embeddings"

  def _headers(self) -> dict[str, str]:
    headers = {
      "Content-Type": "application/json",
    }
    if self.api_key:
      headers["Authorization"] = f"Bearer {self.api_key}"
    return headers

  def _get_local_model(self):
    if self._local_model is None:
      self._log(
        f"[WARN] 硅基流动 embedding 不可用，回退本地模型：{self.model_name} "
        f"(device={self.local_device})"
      )
      self._local_model = _load_local_sentence_transformer(
        self.model_name,
        device=self.local_device,
        retries=self.local_retries,
        log=self._log,
        providers=self.local_providers,
      )
      if self.max_seq_length is not None and hasattr(self._local_model, "max_seq_length"):
        try:
          self._local_model.max_seq_length = self.max_seq_length
        except Exception:
          pass
    return self._local_model

  def _disable_remote(self, reason: Exception | str) -> None:
    self._remote_available = False
    self._remote_disabled_reason = str(reason or "").strip()

  def _encode_via_local(
    self,
    texts,
    *,
    convert_to_numpy: bool,
    normalize_embeddings: bool,
    batch_size: int,
    show_progress_bar: bool,
    **kwargs,
  ):
    local_model = self._get_local_model()
    result = local_model.encode(
      texts,
      convert_to_numpy=convert_to_numpy,
      normalize_embeddings=normalize_embeddings,
      batch_size=batch_size,
      show_progress_bar=show_progress_bar,
      **kwargs,
    )
    if convert_to_numpy and not isinstance(result, np.ndarray):
      try:
        result = np.asarray(result, dtype=np.float32)
      except Exception:
        pass
    return result

  def encode(
    self,
    texts,
    convert_to_numpy: bool = True,
    normalize_embeddings: bool = True,
    batch_size: int = 8,
    show_progress_bar: bool = False,
    **kwargs,
  ):
    if isinstance(texts, str):
      texts = [texts]
    if not isinstance(texts, list):
      texts = list(texts or [])
    if not texts:
      empty = np.zeros((0, 0), dtype=np.float32)
      return empty if convert_to_numpy else empty.tolist()

    safe_batch_size = max(int(batch_size or self.default_batch_size), 1)
    if not self._remote_available:
      return self._encode_via_local(
        texts,
        convert_to_numpy=convert_to_numpy,
        normalize_embeddings=normalize_embeddings,
        batch_size=safe_batch_size,
        show_progress_bar=show_progress_bar,
        **kwargs,
      )
    try:
      siliconflow_max_batch = 32
      adjusted_batch_size = min(safe_batch_size, siliconflow_max_batch)
      chunks = [texts[i : i + adjusted_batch_size] for i in range(0, len(texts), adjusted_batch_size)]
      outputs: list[np.ndarray] = []

      self._log(
        f"[INFO] 硅基流动 embedding：model={self.model_name} "
        f"endpoint={self.endpoint} total={len(texts)} batch={adjusted_batch_size}"
      )

      for chunk_index, chunk in enumerate(chunks, start=1):
        headers = self._headers()
        response = requests.post(
          self.endpoint,
          headers=headers,
          json={
            "model": self.model_name,
            "input": chunk,
            "encoding_format": "float"
          },
          timeout=self.timeout,
        )
        response.raise_for_status()
        data = response.json()
        data_list = data.get("data")
        if not isinstance(data_list, list):
          raise RuntimeError("硅基流动 embedding 服务返回缺少 data 字段")

        embeddings = []
        for item in data_list:
          if isinstance(item, dict) and "embedding" in item:
            embeddings.append(item["embedding"])

        if len(embeddings) != len(chunk):
          raise RuntimeError(
            f"硅基流动 embedding 返回条数异常：expected={len(chunk)} actual={len(embeddings)}"
          )

        try:
          arr = np.asarray(embeddings, dtype=np.float32)
        except Exception as exc:
          raise RuntimeError(f"硅基流动 embedding 返回无法转换为 float32：{exc}") from exc

        if arr.ndim != 2:
          raise RuntimeError(f"硅基流动 embedding 返回维度异常：shape={getattr(arr, 'shape', None)}")

        if normalize_embeddings:
          norms = np.linalg.norm(arr, axis=1, keepdims=True)
          arr = arr / np.clip(norms, 1e-12, None)
        outputs.append(arr)
        self._log(
          f"[INFO] 硅基流动 embedding 批次完成：{chunk_index}/{len(chunks)} "
          f"count={len(chunk)} dim={arr.shape[1]}"
        )

      merged = np.vstack(outputs) if outputs else np.zeros((0, 0), dtype=np.float32)
      return merged if convert_to_numpy else merged.tolist()
    except Exception as exc:
      self._log(f"[WARN] 硅基流动 embedding 请求失败，将自动回退本地模型：{exc}")
      self._disable_remote(exc)
      return self._encode_via_local(
        texts,
        convert_to_numpy=convert_to_numpy,
        normalize_embeddings=normalize_embeddings,
        batch_size=safe_batch_size,
        show_progress_bar=show_progress_bar,
        **kwargs,
      )

  def start_multi_process_pool(self, target_devices=None):
    del target_devices
    return None

  def encode_multi_process(
    self,
    texts,
    pool=None,
    batch_size: int = 8,
    normalize_embeddings: bool = True,
    **kwargs,
  ):
    del pool
    return self.encode(
      texts,
      convert_to_numpy=True,
      normalize_embeddings=normalize_embeddings,
      batch_size=batch_size,
      **kwargs,
    )

  def stop_multi_process_pool(self, pool):
    del pool
    return None
```

- [ ] **Step 4: 验证代码语法**

Run: `python -m py_compile src/model_loader.py`
Expected: 无错误输出

- [ ] **Step 5: Commit**

```bash
git add src/model_loader.py
git commit -m "feat: add siliconflow embedding support"
```

---

### Task 3: 修改 3.rank_papers.py 支持多提供商 rerank

**Files:**
- Modify: `src/3.rank_papers.py:11` (修改导入)
- Modify: `src/3.rank_papers.py:240-246` (修改 process_file 参数)
- Modify: `src/3.rank_papers.py:389-447` (修改 main 函数)

**背景:** 当前硬编码使用 BltClient，需要支持通过环境变量选择提供商。

- [ ] **Step 1: 修改导入语句**

替换 `src/3.rank_papers.py` 第 11 行：

```python
from llm import BltClient, SiliconflowClient, parse_provider_model, ClientFactory
```

- [ ] **Step 2: 修改 process_file 函数签名**

替换 `src/3.rank_papers.py` 第 240-246 行：

```python
def process_file(
  reranker,
  input_path: str,
  output_path: str,
  top_n: Optional[int],
  rerank_model: str,
) -> None:
```

- [ ] **Step 3: 修改 main 函数**

替换 `src/3.rank_papers.py` 的 `main()` 函数（第 389-447 行）：

```python
def main() -> None:
  parser = argparse.ArgumentParser(
    description="步骤 3：使用 Rerank API 对候选论文做重排序（支持 BLT 和硅基流动）。",
  )
  parser.add_argument(
    "--input",
    type=str,
    default=os.path.join(FILTERED_DIR, f"arxiv_papers_{TODAY_STR}.json"),
    help="筛选结果 JSON 路径。",
  )
  parser.add_argument(
    "--output",
    type=str,
    default=os.path.join(RANKED_DIR, f"arxiv_papers_{TODAY_STR}.json"),
    help="打分后的输出 JSON 路径。",
  )
  parser.add_argument(
    "--top-n",
    type=int,
    default=None,
    help="最终保留的 Top N（默认保留全部候选）。",
  )
  parser.add_argument(
    "--rerank-model",
    type=str,
    default=None,
    help="Rerank 模型名称（支持 provider/model 格式，如 siliconflow/BAAI/bge-reranker-v2-m3）。",
  )
  parser.add_argument(
    "--rerank-provider",
    type=str,
    default=None,
    choices=["blt", "siliconflow"],
    help="Rerank 提供商（blt 或 siliconflow），如不指定则从模型名称推断。",
  )

  args = parser.parse_args()

  input_path = args.input
  if not os.path.isabs(input_path):
    input_path = os.path.abspath(os.path.join(ROOT_DIR, input_path))

  output_path = args.output
  if not os.path.isabs(output_path):
    output_path = os.path.abspath(os.path.join(ROOT_DIR, output_path))

  if not os.path.exists(input_path):
    log(f"[WARN] 输入文件不存在（今天可能没有新论文）：{input_path}，将跳过 Step 3。")
    return

  rerank_provider = args.rerank_provider or os.getenv("RERANK_PROVIDER", "blt").strip().lower()
  rerank_model = args.rerank_model

  if rerank_model and "/" in rerank_model and not args.rerank_provider:
    provider_from_model, model_from_model = parse_provider_model(rerank_model)
    rerank_provider = provider_from_model
    rerank_model = model_from_model

  if not rerank_model:
    if rerank_provider == "siliconflow":
      rerank_model = os.getenv("SILICONFLOW_RERANK_MODEL") or "BAAI/bge-reranker-v2-m3"
    else:
      rerank_model = os.getenv("BLT_RERANK_MODEL") or os.getenv("RERANK_MODEL") or "qwen3-reranker-4b"

  if rerank_provider == "siliconflow":
    api_key = os.getenv("SILICONFLOW_API_KEY")
    if not api_key:
      raise RuntimeError("缺少 SILICONFLOW_API_KEY 环境变量，无法调用硅基流动 Rerank API。")
    reranker = SiliconflowClient(api_key=api_key, model=rerank_model)
  else:
    api_key = os.getenv("BLT_API_KEY")
    if not api_key:
      raise RuntimeError("缺少 BLT_API_KEY 环境变量，无法调用 BLT Rerank API。")
    reranker = BltClient(api_key=api_key, model=rerank_model)

  log(f"[INFO] 使用 Rerank 提供商：{rerank_provider}，模型：{rerank_model}")

  process_file(
    reranker=reranker,
    input_path=input_path,
    output_path=output_path,
    top_n=args.top_n,
    rerank_model=rerank_model,
  )
```

- [ ] **Step 4: 验证代码语法**

Run: `python -m py_compile src/3.rank_papers.py`
Expected: 无错误输出

- [ ] **Step 5: Commit**

```bash
git add src/3.rank_papers.py
git commit -m "feat: support multiple rerank providers (blt/siliconflow)"
```

---

## 环境变量配置说明

添加以下环境变量支持：

| 环境变量 | 用途 | 默认值 |
|----------|------|--------|
| `EMBED_PROVIDER` | Embedding 提供商（custom/siliconflow） | `custom` |
| `SILICONFLOW_API_KEY` | 硅基流动 API 密钥 | - |
| `SILICONFLOW_EMBED_MODEL` | 硅基流动 Embedding 模型 | `BAAI/bge-large-zh-v1.5` |
| `SILICONFLOW_EMBED_ENDPOINT` | 硅基流动 Embedding 端点 | `https://api.siliconflow.cn/v1` |
| `RERANK_PROVIDER` | Rerank 提供商（blt/siliconflow） | `blt` |
| `SILICONFLOW_RERANK_MODEL` | 硅基流动 Rerank 模型 | `BAAI/bge-reranker-v2-m3` |

---

## 使用示例

### 使用硅基流动 Embedding
```bash
export EMBED_PROVIDER=siliconflow
export SILICONFLOW_API_KEY=your_api_key
export SILICONFLOW_EMBED_MODEL=BAAI/bge-large-zh-v1.5
```

### 使用硅基流动 Rerank
```bash
export RERANK_PROVIDER=siliconflow
export SILICONFLOW_API_KEY=your_api_key
export SILICONFLOW_RERANK_MODEL=BAAI/bge-reranker-v2-m3
```

### 或使用 provider/model 格式
```bash
python src/3.rank_papers.py --rerank-model siliconflow/BAAI/bge-reranker-v2-m3
```

---

## 自验证检查

- [x] Spec coverage: 所有需求都有对应任务
- [x] 无占位符: 所有代码都是完整的
- [x] 类型一致性: 方法签名和属性名一致
- [x] 向后兼容性: 默认配置保持不变

---

Plan complete and saved to `docs/superpowers/plans/2026-04-13-siliconflow-embedding-rerank.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
