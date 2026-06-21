---
layout: post
title: "코딩 에이전트는 같은 코드를 몇 번이나 다시 읽을까 — AST 기반 KV cache 재사용을 실험하며"
date: 2026-06-22
categories: [capstone, llm-systems]
tags: [vLLM, KV-cache, AST, prefix-caching, capstone-design]
---

캡스톤디자인 프로젝트로 LLM 서빙 최적화를 다루면서, 가장 먼저 부딪힌 벽은 알고리즘이 아니라 **환경설정**이었다. 이 글은 그 시행착오와, 그 끝에 확인한 실험 결과, 그리고 직접 재현해볼 수 있는 환경 구성 과정을 정리한 기록이다. 코드와 실험 데이터는 [GitHub](https://github.com/capstone-2026-ewha/def)에 공개되어 있다.

---

## 왜 코딩 에이전트의 KV cache를 들여다봤나

요즘 Cursor나 Copilot 같은 AI 코딩 도구를 쓰다 보면 이상한 점이 하나 있다. 에이전트는 파일을 읽고, 코드를 고치고, 테스트를 돌리는 과정을 한 세션 안에서 수십 번 반복한다. 그런데 그 과정에서 **같은 파일을 여러 번 다시 읽는** 일이 굉장히 흔하다.

LLM이 텍스트를 처리할 때는 입력 토큰마다 KV cache(어텐션 연산에서 나온 중간 결과로, Key와 Value 값을 저장해두는 캐시)를 계산하는데, 같은 입력이 반복되면 이걸 재사용해서 연산량을 줄일 수 있다. 그런데 vLLM 같은 서빙 엔진의 기본 prefix caching은 **앞부분 토큰이 완전히 일치할 때만** 캐시를 재사용한다. 문제는 멀티턴 세션에서는 매 턴마다 타임스탬프나 도구 호출 ID 같은 동적인 값이 끼어들어서, 코드 내용이 똑같아도 이 "완전 일치" 조건이 계속 깨진다는 점이다.

실제로 48개 trace를 분석해보니 코딩 에이전트가 쓰는 전체 토큰의 **83.3%가 파일 읽기**에서 나왔다. 같은 코드가 반복해서 등장하는데도 매번 처음부터 계산되고 있었던 거다. 이 지점이 우리 연구의 출발점이었다.

---

## 선행 연구는 이 문제를 어떻게 풀어왔나

KV cache 재사용 자체는 새로운 주제가 아니다. 우리가 참고한 세 갈래 접근을 짚어본다.

### PagedAttention (vLLM) — 정확한 prefix 재사용

Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*, SOSP 2023.

vLLM의 핵심 메커니즘으로, 운영체제의 가상 메모리 페이징 개념을 KV cache에 적용해 메모리 단편화를 줄인 연구다. 이를 기반으로 한 prefix caching은 연속된 앞부분 토큰이 완전히 일치할 때만 캐시를 재사용하는 방식이라 구현이 단순하고 정확도 손실이 없다는 장점이 있다. 다만 멀티턴 세션에서 동적 메타정보가 prefix를 깨뜨리면 hit rate가 급격히 떨어진다는 한계가 있다 — 우리 실험에서도 직접 확인한 부분이다.

### PromptCache — 모듈형 attention state 재사용

Gim et al., *Prompt Cache: Modular Attention Reuse for Low-Latency Inference*, MLSys 2024.

Yale와 Google이 함께 발표한 연구로, 시스템 메시지나 프롬프트 템플릿처럼 여러 요청에 걸쳐 반복되는 텍스트 조각(prompt module)의 attention state를 미리 계산해 저장해두고 재사용한다. 다만 어떤 부분이 재사용 가능한 모듈인지 **스키마로 사전에 명시**해야 한다는 전제가 있다. 코딩 에이전트가 실행 중 동적으로 생성하는 파일 읽기 결과나 터미널 출력에는 이 전제가 성립하지 않는다.

### SnapKV — attention score 기반 압축

Li et al., *SnapKV: LLM Knows What You are Looking for Before Generation*, NeurIPS 2024.

긴 입력을 처리할 때 KV cache 크기 자체가 커지는 문제를 다룬 연구로, attention head가 일관되게 주목하는 prompt attention feature를 관찰 윈도우에서 찾아내 중요도가 낮은 KV를 선택적으로 압축한다. 16K 토큰 입력 기준 디코딩 속도 3.6배, 메모리 효율 8.2배 개선을 보고했다. 다만 **KV를 버리는 방식**이기 때문에 압축률을 높일수록 정확도 손실 위험이 커지고, 코드 구조 같은 도메인 지식은 직접 활용하지 않는다.

### 세 접근의 공통된 빈틈

세 연구 모두 강점이 분명하지만, **코딩 에이전트의 동적 경계 + 코드의 구조적 반복성 + 세션 수준 재사용**이라는 세 조건을 동시에 만족하지는 못한다는 게 우리가 내린 결론이었다. PagedAttention은 동적 경계 자체를 다루지 못하고, PromptCache는 경계를 사전에 알아야 하며, SnapKV는 코드 구조를 활용하지 않는다. 우리는 이 빈틈을 메우기 위해 **AST(추상 구문 트리) 서브트리를 재사용 단위로 삼는 방식**을 시도했다.

---

## 본론 전에: 환경설정에서 일주일을 날린 이야기

연구 아이디어보다 먼저 풀어야 했던 건 **돌아가는 환경을 만드는 것**이었다. 아래는 실제로 우리가 거친 시행착오와, 최종적으로 정착한 환경 구성 순서다. 그대로 따라 하면 동일한 환경을 재현할 수 있다.

### 1) vLLM 설치가 자꾸 실패했다

OpenHands(코딩 에이전트 프레임워크)를 실행할 환경을 만드는데, git 기반으로 vLLM을 설치하는 게 계속 실패했다. 원인을 추적해보니 의존성 패키지 문제였다. 의존성을 먼저 따로 설치한 다음, 기존에 빌드해둔 vLLM을 `PYTHONPATH`로 우회 연결하는 방식으로 겨우 해결했다. 완전히 새로 설치하는 대신 "이미 동작하는 게 있는데 재활용할 수 없을까"를 먼저 점검한 게 시간을 아낀 선택이었다.

처음부터 환경을 새로 만든다면 아래 순서를 추천한다.

```bash
# 레포 클론
git clone https://github.com/capstone-2026-ewha/def.git
cd def

# vLLM 서빙용 conda 환경
conda create -n vllm python=3.10
conda activate vllm
pip install vllm==0.8.x humming-kernels[cu13]

# trace 수집·분석용 conda 환경 (별도 분리 권장)
conda create -n openhands python=3.10
conda activate openhands
pip install open-hands-ai tree-sitter transformers blake3
```

서빙 환경과 분석 환경을 분리한 이유는, vLLM 자체가 의존성이 무겁고 충돌이 잦아서 trace 분석용 패키지(tree-sitter, blake3 등)와 같은 환경에 두면 디버깅이 훨씬 어려워지기 때문이다.

### 2) 모델 크기를 정하는 데만 일주일

처음엔 7B 모델로 trace를 수집했는데, 실제 이벤트 로그를 읽어보니 **할루시네이션이 너무 심했다**. 모델이 존재하지 않는 파일 경로를 만들어내거나, JSON 파싱 자체가 깨지는 경우가 잦았다. 14B로 올려도 비슷한 문제가 반복됐다.

공식 문서를 참고해서 30B로 올리기로 했는데, 이번엔 GPU VRAM이 발목을 잡았다. RTX 5090 두 장(각 31.84GB)으로도 30B 모델의 KV cache 공간을 잡을 수가 없었다. 처음엔 `--dtype bfloat16` 옵션으로 즉석 양자화를 시도했지만 이마저도 실패했고, 결국 공식적으로 배포된 FP8 양자화 모델을 따로 받아서 적용하고 나서야 안정적으로 서빙할 수 있었다.

```bash
# 공식 FP8 양자화 모델 다운로드
huggingface-cli download Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8 \
    --local-dir ./models/Qwen3-Coder-30B-A3B-Instruct-FP8

# vLLM 서버 실행 (2-GPU tensor parallel)
conda activate vllm
vllm serve ./models/Qwen3-Coder-30B-A3B-Instruct-FP8 \
    --tool-call-parser qwen3_coder \
    --enable-expert-parallel \
    --tensor-parallel-size 2 \
    --max-model-len 131072 \
    --gpu-memory-utilization 0.9 \
    --port 8000
```

`--tool-call-parser qwen3_coder`는 OpenHands가 보내는 tool call을 모델이 올바르게 파싱하도록 지정하는 옵션이고, `--tensor-parallel-size 2`로 두 GPU에 모델을 분산했다. `--max-model-len 131072`는 멀티턴 세션이 길어져도 컨텍스트가 잘리지 않도록 충분히 크게 잡은 값이다.

이 과정에서 작은 모델로는 애초에 "정상적인 코딩 에이전트 작업 기록"을 얻을 수가 없다는 걸 체감했다. trace 품질이 모델 크기에 크게 좌우된다는 건 문서로 읽었을 때와 직접 망가진 로그를 들여다봤을 때의 체감이 완전히 달랐다.

### 3) "정상 종료"의 기준을 다시 세워야 했다

trace 50개를 수집한 뒤, 어떤 게 정상이고 어떤 게 비정상인지 걸러내는 작업이 필요했다. 처음엔 단순하게 생각했다 — 마지막 이벤트가 MessageEvent로 끝나면 정상이라고.

그런데 실제 로그를 하나하나 읽어보니, 모델이 "I have successfully fixed the issue"라고 완료를 선언해놓고도, 그 직후에 명시했어야 할 도구 호출(`<tool_call>`)이 실제로는 실행되지 않은 채 끝난 trace가 있었다. 텍스트 패턴만 보고 정상/비정상을 가르려 했다면 이런 경우를 정상으로 잘못 분류할 뻔했다.

그래서 기준을 바꿨다. **텍스트 선언이 아니라 실제 도구 호출 여부로 판단**해야 한다고. 이 기준으로 다시 걸러낸 결과, 수집한 50개 trace 중 2개(`django__django-11019_run1`, `matplotlib__matplotlib-23563_run0`)가 도구 호출 실패로 인한 비정상 종료였고, 이를 제외한 48개를 최종 분석에 사용했다.

각 trace를 수집할 때는 이슈마다 git 상태를 초기화해서 trace 간 오염을 방지했다.

```bash
git checkout . && git clean -fd
```

### `collect_traces_30b_fp8.py` 뜯어보기

trace 수집은 OpenHands SDK로 에이전트를 만들고, SWE-bench 이슈 25개를 각 2회씩 돌리면서 전체 작업 과정을 JSON으로 저장하는 구조다.

```python
llm = LLM(
    model="openai/Qwen3-Coder-30B-A3B-Instruct-FP8",
    api_key="dummy",
    base_url="http://localhost:8002/v1",
)
```
OpenHands SDK의 `LLM` 객체가 OpenAI 호환 API 형식으로 로컬 vLLM 서버를 호출하도록 설정한다. vLLM은 OpenAI API와 호환되는 엔드포인트를 제공하기 때문에 `openai/` 프리픽스로 모델명을 지정하면 별도 어댑터 없이 바로 붙는다. `api_key="dummy"`는 로컬 서버라 실제 인증이 필요 없어서 형식만 맞춘 값이다.

```python
subprocess.run(['git', 'checkout', '.'], cwd=cwd, check=True)
subprocess.run(['git', 'clean', '-fd'], cwd=cwd, check=True)
```
이슈 하나당 run 1, run 2 두 번씩 돌리는데, 그 사이마다 레포지토리를 깨끗한 상태로 되돌린다. `checkout .`은 수정된 파일을 원본으로 복원하고, `clean -fd`는 에이전트가 새로 만든 파일·디렉토리까지 지운다. 이게 없으면 run 1의 수정 흔적이 run 2에 그대로 남아서 두 run이 사실상 같은 trace가 되어버린다 — 독립적인 두 번의 시도를 비교한다는 실험 설계 자체가 무너지는 셈이다.

```python
agent = Agent(
    llm=llm,
    tools=[
        Tool(name=TerminalTool.name),
        Tool(name=FileEditorTool.name),
    ],
)
```
에이전트가 쓸 수 있는 도구를 터미널 명령 실행과 파일 편집(읽기/수정/생성) 두 가지로 제한했다. 이 두 도구로 발생하는 호출이 바로 뒤에서 다룰 `ActionEvent`·`ObservationEvent`의 실체다.

```python
trace = {
    'instance_id': instance_id,
    'repo': repo,
    'run_idx': run_idx,
    'events': [e.model_dump() for e in conversation.state.events],
}
```
`conversation.state.events`에는 OpenHands가 세션 동안 쌓은 모든 이벤트(시스템 프롬프트, 사용자 메시지, 에이전트 행동, 관찰 결과)가 들어있다. `model_dump()`로 각 이벤트 객체를 JSON 직렬화 가능한 딕셔너리로 바꾼 뒤, instance_id별로 파일에 저장한다. 이 trace JSON 파일이 이후 hit rate 분석과 TTFT replay 실험 모두의 원천 데이터가 된다.

이미 존재하는 trace는 건너뛰도록 처리해뒀는데(`if os.path.exists(trace_path): continue`), 30B 모델로 50개 trace를 수집하는 데 시간이 꽤 걸려서 중간에 끊겨도 이어서 돌릴 수 있게 해둔 안전장치다.

```bash
# SWE-bench trace 수집 (25개 이슈, 각 2회씩 → 총 50개)
conda activate openhands
python experiments/scripts/collect_traces_30b_fp8.py
```

---

## 실험 1: 코드 구조는 정말로 반복되는가 (Hit Rate 분석)

본격적인 가설 검증에 들어가기 전, 가장 먼저 확인해야 할 건 "애초에 재사용할 만한 코드 반복이 충분히 존재하는가"였다. 아무리 좋은 캐시 메커니즘을 만들어도 재사용할 게 없으면 의미가 없으니까.

### 방법

Tree-sitter로 코드를 AST(추상 구문 트리)로 파싱하고, 다음 조건을 만족하는 서브트리만 재사용 후보로 골랐다.

- 노드 타입이 함수 정의, 클래스 정의, import문, 함수 호출 등 의미 있는 구조일 것
- AST 깊이 2 이상, 토큰 수 8개 이상 (Qwen3-Coder-30B tokenizer 기준)

여기에 한 가지를 더했다. 변수명이 `price`든 `amount`든 구조가 같으면 같은 코드로 인식하도록, **α-rename 정규화**를 적용해서 식별자를 `VAR_0`, `VAR_1` 같은 형태로, 리터럴을 `STR`/`NUM`으로 통일한 뒤 S-expression으로 직렬화하고 BLAKE3 해시로 32바이트 지문을 만들었다. 텍스트가 완전히 같지 않아도 "구조가 같은 코드"를 잡아내기 위한 장치다.

### `astkv_fingerprint.py` 뜯어보기 — 지문은 어떻게 만들어지나

이 스크립트가 파이프라인의 핵심이다. 코드 한 덩어리를 받아서 "재사용 가능한 서브트리"를 골라내고, 그걸 지문(fingerprint)으로 바꾸는 일을 한다.

```python
ALLOWED_TYPES = {
    "function_definition", "class_definition",
    "import_statement", "import_from_statement",
    "call", "assignment",
}
```
AST의 모든 노드를 재사용 후보로 삼으면 너무 잘게 쪼개진 의미 없는 조각들(예: 단순 변수 하나)까지 캐시 대상이 돼버린다. 그래서 함수/클래스 정의, import문, 함수 호출, 할당문처럼 그 자체로 의미가 완결된 노드 타입만 후보로 제한했다.

```python
def traverse(n):
    if n.type in ALLOWED_TYPES:
        depth = get_depth(n)
        if depth >= min_depth:
            text = code_bytes[n.start_byte:n.end_byte].decode("utf-8", "ignore")
            if count_tokens(text) >= min_tokens:
                results.append(n)
                return  # 자격 노드 -> 자식으로 안 내려감 (최상위만)
    for child in n.children:
        traverse(child)
```
이 함수가 "최상위 서브트리만" 추출하는 부분이다. 자격 조건(허용된 타입, 깊이 2 이상, 토큰 8개 이상)을 만족하는 노드를 찾으면 그 즉시 결과에 담고 `return`으로 **자식 노드 탐색을 멈춘다**. 예를 들어 함수 정의 안에 또 다른 함수 호출이 있어도, 함수 정의 자체가 이미 자격을 통과했다면 그 안의 함수 호출까지 따로 캐시 후보로 잡지 않는다. 부모 노드의 KV를 재사용하면 자식 노드 내용은 자연히 포함되기 때문에, 중복으로 카운트할 필요가 없다는 설계다.

```python
def _is_variable_identifier(node):
    ...
    if ptype in ("function_definition", "class_definition") and field == "name":
        return False
    if ptype == "call" and field == "function":
        return False
    if ptype == "attribute" and field in ("attribute", "object"):
        return False
    if ptype == "keyword_argument" and field == "name":
        return False
    return True
```
`vars_only` 모드에서 "이 식별자가 진짜 변수인가, 아니면 보존해야 할 이름인가"를 가르는 휴리스틱이다. 함수/클래스 이름, 호출되는 함수 이름, `obj.attr` 형태의 모듈명·속성명, 키워드 인자 이름은 의미가 식별자 자체에 있기 때문에 치환하지 않고 그대로 둔다. 반대로 이 조건에 안 걸리면 "변수"로 간주해 `VAR_N`으로 치환한다.

코드 안의 주석으로도 남겨뒀듯, 이 휴리스틱에는 한계가 있다. `obj.method()`에서 `obj`가 진짜 변수여도 attribute의 object라는 이유로 보존되는 경우가 있다. 완벽한 변수 판별보다는 "대부분의 흔한 패턴을 잡아내는" 실용적 절충안이다.

```python
def normalize(node, code_bytes, rename_mode="vars_only"):
    ...
    if t == "string":
        return "STR"
    if t in ("integer", "float"):
        return "NUM"
    ...
    if t == "identifier":
        ...
        if rename_mode == "all":
            return rename_id(name)
        if _is_variable_identifier(n):
            return rename_id(name)
        return name
    ...
    inner = " ".join(walk(c) for c in n.children)
    return f"({t} {inner})"
```
AST를 재귀적으로 순회하면서 S-expression 문자열로 직렬화하는 부분이다. 문자열·숫자 리터럴은 값과 무관하게 `STR`/`NUM`으로 뭉개고, 식별자는 위에서 판별한 기준에 따라 치환하거나 보존한다. 최종적으로 `(function_definition (identifier VAR_0) (parameters ...) ...)` 같은 형태의 문자열이 나오는데, 이 문자열이 코드의 "구조"만 남긴 표현이다.

```python
def fingerprint(node, code_bytes, rename_mode="vars_only"):
    norm = normalize(node, code_bytes, rename_mode=rename_mode)
    return blake3.blake3(norm.encode("utf-8")).hexdigest()
```
정규화된 문자열을 BLAKE3로 해싱해서 32바이트(64자리 hex) 지문을 만든다. 변수명이 달라도 구조가 같으면 정규화 결과가 동일한 문자열이 되므로, 해시값도 동일해진다 — 이게 "구조 기반 캐시 조회 키"의 핵심 메커니즘이다.

스크립트 맨 아래 `__main__` 블록에는 직접 검증해볼 수 있는 예제가 있다. `parse_json`과 `parse_data`는 변수명만 다를 뿐 구조가 같은 함수라 `vars_only` 모드에서 같은 지문이 나오고, `parse_pickle`은 `json.loads` 대신 `pickle.loads`를 쓰기 때문에(모듈명이 보존되는 부분) 다른 지문이 나온다. 직접 실행해서 확인할 수 있다.

```bash
python experiments/scripts/astkv_fingerprint.py ~/local-proj/ast-kv/models/Qwen3-Coder-30B-A3B-Instruct
```

### `hitrate_astkv.py` 뜯어보기 — 4가지 조건을 한 번에 비교하기

trace 48개에 대해 hit rate를 계산하는 스크립트로, "분석 소스(observation만 vs observation+action)"와 "α-rename 방식(all vs vars_only)"을 조합한 4가지 조건을 한 번의 실행으로 모두 비교한다.

```python
EXCLUDE = {"django__django-11019_run1", "matplotlib__matplotlib-23563_run0"}
GO_HIT_THRESHOLD = 25.0
GO_TOKEN_THRESHOLD = 30.0
```
앞서 trace 정상성 기준으로 걸러낸 비정상 trace 2개를 여기서도 명시적으로 제외한다. GO 기준(hit rate 25% 이상, token ratio 30% 이상)도 상수로 박아둬서 코드만 봐도 판정 기준이 바로 드러나게 했다.

```python
def iter_text_blocks(trace):
    for ev in trace.get("events", []):
        kind = ev.get("kind")
        if kind == "ObservationEvent":
            ...
            yield ("observation", strip_line_numbers(txt))
        elif kind == "ActionEvent":
            act = ev.get("action") or {}
            if act.get("kind") != "FileEditorAction":
                continue
            for field in ("old_str", "new_str", "file_text"):
                ...
                yield (f"action_{field}", txt)
```
trace JSON에서 실제 코드 텍스트가 들어있는 부분만 뽑아낸다. `ObservationEvent`는 파일을 읽은 결과(즉 에이전트가 "본" 코드)이고, `ActionEvent`의 `old_str`/`new_str`은 에이전트가 코드를 수정할 때 사용한 "고친 전/후 코드"다. `strip_line_numbers`는 `file_editor`가 출력에 붙이는 `cat -n` 형식의 줄 번호(`   12\tdef foo():`)를 제거해서 순수 코드만 남긴다 — 줄 번호가 안 지워지면 같은 함수라도 등장 위치(줄 번호)가 달라질 때마다 다른 텍스트로 인식돼 지문이 깨진다.

```python
class Stat:
    def add_session(self, name, blocks_by_mode):
        session_seen = set()
        ...
        for src_kind, fps, elig_tok, all_tok in data:
            ...
            for fp in fps:
                self.global_total += 1
                if fp in self.global_seen:
                    self.global_hit += 1
                else:
                    self.global_seen.add(fp)
                s_total += 1
                if fp in session_seen:
                    s_hit += 1
                else:
                    session_seen.add(fp)
```
`Stat` 클래스가 4가지 조건 각각에 대해 hit/miss를 누적 집계한다. 중요한 설계 포인트는 hit 판정이 **세션(trace) 단위로 초기화**된다는 점이다(`session_seen = set()`이 매 trace마다 새로 만들어짐). 즉 "이 작업 세션 안에서 같은 코드 구조가 다시 등장했는가"를 보는 거지, trace를 넘나들며 캐시를 공유하지는 않는다. 실제 KV cache도 세션이 끝나면 의미가 없어지므로 이 설계가 현실적인 시나리오를 반영한다.

```python
def summary(self):
    valid = [r for _, t, _, r in self.per_session if t > 0]
    macro = sum(valid) / len(valid) if valid else 0.0
```
hit rate는 두 가지 방식으로 계산할 수 있다 — 전체 이벤트를 다 모아서 한 번에 비율을 내는 micro 방식과, 세션별로 비율을 낸 뒤 그걸 평균 내는 macro 방식. 이 스크립트는 macro를 메인 지표로 쓴다. trace마다 이벤트 수가 다른데(최소 9개, 최대 160개), micro로 계산하면 이벤트가 많은 trace 하나가 전체 평균을 과도하게 좌우할 수 있어서, "각 작업이 동등한 비중을 갖도록" macro 평균을 택했다.

```bash
python experiments/scripts/hitrate_astkv.py \
    ./experiments/data/traces/30b_direct_run01 \
    ~/local-proj/ast-kv/models/Qwen3-Coder-30B-A3B-Instruct
```

### 결과

가장 보수적인 조건(파일 읽기 이벤트만 분석, 모듈명·메서드명은 보존하는 `vars_only` 정규화)에서도 **hit rate 32.6%, token ratio 34.5%**가 나왔다. 우리가 사전에 세운 GO 기준(hit rate 25% 이상 AND token ratio 30% 이상)을 모두 통과한 결과였다.

| 조건 | Hit Rate | Token Ratio | 판정 |
|------|----------|-------------|------|
| obs 단독 / vars_only | 32.6% | 34.5% | ✅ GO |
| obs 단독 / all | 42.4% | 34.5% | ✅ GO |
| obs+action / vars_only | 42.6% | 40.0% | ✅ GO |
| obs+action / all | 51.1% | 40.0% | ✅ GO |

흥미로웠던 건 α-rename의 효과였다. 변수명까지 모두 정규화하는 방식(`all`)이 모듈명·메서드명을 보존하는 방식(`vars_only`)보다 hit rate가 약 9.8%p 더 높게 나왔다. 다만 이건 "더 좋다"가 아니라 "더 공격적이다"에 가깝다 — `all` 방식은 `json.loads`의 `loads`까지 변수명으로 취급해 치환하기 때문에, 의미가 다른 코드를 같은 코드로 잘못 묶을 위험(false positive)이 있다. 그래서 우리는 보수적인 `vars_only` 쪽을 권장 기준으로 채택했다.

---

## 실험 2: 그럼 기존 prefix caching은 왜 부족한가 (TTFT 비교)

두 번째 실험은 "기존 방식이 정말로 한계가 있는가"를 직접 측정하는 것이었다.

### 방법

KV 재사용을 아예 끈 vanilla decode와, vLLM의 기본 prefix caching을 켠 조건을 비교했다. 가설은 단순했다 — vanilla는 턴이 늘수록 TTFT(Time-To-First-Token, 첫 토큰이 나오기까지 걸리는 시간)가 선형으로 증가할 것이고, prefix caching은 이전 컨텍스트를 재사용하니 어느 정도 안정될 것이다.

### `vanilla_replay.py` / `prefix_caching_replay.py` 뜯어보기

두 스크립트는 구조가 거의 동일하다. 차이는 vLLM 서버를 띄울 때 `--enable-prefix-caching` 플래그를 켰는지 여부뿐이고, 같은 trace를 같은 방식으로 재생(replay)해서 TTFT를 측정한다. 이미 수집해둔 trace를 "다시 재생"하는 이유는, 실제 에이전트를 처음부터 다시 돌리면 trace 내용 자체가 달라질 수 있어서 — vanilla와 prefix caching 조건을 **완전히 동일한 입력**으로 비교하려면 이미 고정된 trace를 그대로 replay하는 게 맞다.

```python
def extract_turns(events):
    turns = []
    current_messages = []

    for e in events:
        kind = e.get('kind', '')

        if kind == 'SystemPromptEvent':
            ...
            current_messages = [{'role': 'system', 'content': text}]

        elif kind == 'MessageEvent' and e.get('source') == 'user':
            ...
            current_messages = current_messages + [{'role': 'user', 'content': text}]

        elif kind == 'ActionEvent' and e.get('source') == 'agent':
            ...
            if text or tool_call:
                turns.append(list(current_messages))
                ...
                current_messages = current_messages + [assistant_msg]

        elif kind == 'ObservationEvent':
            ...
            current_messages = current_messages + [{...}]

    return turns
```
trace에 저장된 이벤트 시퀀스를, 실제 LLM에 보냈을 법한 "누적 메시지 컨텍스트" 형태로 재구성하는 함수다. 핵심은 `ActionEvent`를 만날 때마다 **그 시점까지 쌓인 `current_messages`를 턴으로 저장**한다는 점이다(`turns.append(list(current_messages))`). 즉 trace 안의 events 리스트를 한 번 쭉 훑으면서, 에이전트가 다음 행동을 결정하기 직전 시점의 컨텍스트 스냅샷을 턴 단위로 잘라내는 것이다. 이렇게 만들어진 turn 1개가 곧 "에이전트가 다음 응답을 생성하기 위해 모델에 보냈을 입력"이 된다.

`list(current_messages)`로 복사본을 저장하는 이유도 중요하다 — 그냥 `current_messages`를 저장하면 같은 리스트 객체를 참조하게 되어, 이후 메시지가 계속 append될 때 이미 저장해둔 과거 턴의 내용까지 덩달아 바뀌어버린다.

```python
def measure_ttft(messages):
    url = f"{VLLM_BASE_URL}/v1/chat/completions"
    payload = {
        "model": MODEL_NAME,
        "messages": messages,
        "max_tokens": 1,
        "stream": True,
        "temperature": 0,
    }
    start = time.time()
    first_token_time = None
    with requests.post(url, json=payload, stream=True, timeout=120) as resp:
        for line in resp.iter_lines():
            if line and line != b'data: [DONE]':
                first_token_time = time.time()
                break
    return (first_token_time - start) * 1000
```
TTFT 측정의 핵심이다. `max_tokens: 1`로 딱 한 토큰만 생성하게 하고(생성 시간 자체가 아니라 prefill 시간을 보는 게 목적이므로), `stream: True`로 토큰이 생성되는 즉시 스트리밍 응답을 받는다. 요청을 보낸 시각(`start`)부터 첫 번째 응답 라인이 도착한 시각까지의 차이가 곧 TTFT다. `temperature: 0`은 같은 입력에 대해 항상 같은 결과가 나오도록 재현성을 확보하기 위한 설정이다.

```python
def get_prefix_cache_hit_rate():
    resp = requests.get(f"{VLLM_BASE_URL}/metrics", timeout=10)
    hits, queries = None, None
    for line in resp.text.splitlines():
        if 'vllm:prefix_cache_hits_total' in line and 'created' not in line:
            hits = float(line.split()[-1])
        if 'vllm:prefix_cache_queries_total' in line and 'created' not in line:
            queries = float(line.split()[-1])
    if hits is not None and queries is not None and queries > 0:
        return hits / queries
```
vLLM은 `/metrics` 엔드포인트로 Prometheus 형식의 내부 통계를 노출하는데, 거기서 `prefix_cache_hits_total`과 `prefix_cache_queries_total` 두 값을 직접 읽어서 누적 hit rate를 계산한다(prefix_caching_replay.py에만 있는 함수다). 매 턴마다 이 값을 같이 기록해서, "지금까지 캐시가 얼마나 적중했는지"와 "그런데도 TTFT는 어떻게 변하는지"를 나란히 비교할 수 있게 했다 — hit rate 75.8%인데 TTFT 개선은 미미했던 모순을 포착할 수 있었던 게 바로 이 계측 덕분이다.

```bash
# vLLM 서버를 두 조건으로 각각 띄운 뒤 실행
# (vanilla: --no-enable-prefix-caching / prefix caching: --enable-prefix-caching)
python experiments/scripts/vanilla_replay.py
python experiments/scripts/prefix_caching_replay.py

# 결과 시각화
python experiments/scripts/visualize/plot_ttft.py
python experiments/scripts/visualize/plot_ttft_avg_comparison.py
```

이 측정은 30B 모델 대신 Qwen2.5-Coder-0.5B-Instruct로 진행했다. 30B는 시스템 프롬프트 캐시 효과가 워낙 압도적이라 코드 블록 구간만의 효과를 분리해서 보기가 어려웠고, 작은 모델은 그 비중이 상대적으로 낮아 구간별 효과를 더 명확하게 볼 수 있었다.

### 결과는 예상과 달랐다

48개 trace, 총 1,721턴을 대상으로 측정한 결과, **두 조건의 TTFT 증가 패턴이 사실상 동일**했다.

| 조건 | 전체 평균 TTFT | 최종 prefix cache hit rate |
|------|----------------|------------------------------|
| Vanilla decode | 30.5 ms | 0% |
| vLLM prefix caching | 29.7 ms | 75.8% |

차이는 단 0.8ms. 더 인상적인 건 prefix cache hit rate가 75.8%로 꽤 높게 측정됐는데도 이 결과가 나왔다는 점이다. 캐시는 적중하고 있는데 TTFT는 줄지 않는다? 이 모순을 들여다보니 답이 보였다 — 캐시 적중의 대부분이 모든 세션에 공통으로 들어가는 시스템 프롬프트 구간에서만 일어나고 있었다. 실제로 매 턴 누적되는 코드 내용 구간에서는 `tool_call_id` 같은 동적 값이 앞부분을 계속 갱신시키면서 "완전 일치"라는 조건을 매번 깨뜨리고 있었던 거다.

다시 말해, prefix caching은 **켜져 있어도 정작 중요한 코드 블록 구간에서는 작동하지 않는다.** 이게 우리 연구의 핵심 출발점이 됐다.

---

## 두 실험을 합치면

1번 실험은 "재사용할 코드 구조가 충분히 존재한다"는 걸, 2번 실험은 "그런데 기존 방식은 그걸 못 잡는다"는 걸 보여줬다. 둘을 합치면 결론이 명확해진다 — 토큰 단위 완전 일치가 아니라 **코드의 문법적 구조(AST) 단위로 재사용 여부를 판단해야 한다**는 것.

이 구조 인식을 위해 우리는 α-rename 정규화로 변수명 변화를 흡수하고, RoPE re-positioning(재사용한 KV가 새로운 위치에 삽입될 때 발생하는 위치 정보 불일치를, 위치 오프셋 Δ를 이용한 회전 행렬 `K_new = R(Δ) · K_cached`로 보정하는 기법)으로 위치 왜곡 문제를 해결하는 방향을 설계했다. 이 부분은 다음 단계 — 실제 KV 재사용 시스템(SessionCacheTable + RoPE re-positioning) 구현 — 에서 검증할 예정이다.

---

## 돌아보며

한 학기 동안 가장 크게 배운 건 알고리즘보다 오히려 **"신뢰할 수 있는 데이터를 어떻게 만들 것인가"**였다. 모델이 작으면 trace 자체가 망가지고, trace 정상성 기준을 허술하게 잡으면 분석 결과 전체가 흔들린다. 환경 하나, 기준 하나를 제대로 세우는 데 들인 시간이 결과적으로 실험의 신뢰도를 받쳐줬다고 생각한다.

다음 단계는 실제로 KV를 재사용하는 시스템을 구현하고, 목표로 잡은 TTFT 30% 이상 감축과 코드 생성 정확도(HumanEval pass@1) 손실 1.5pp 이내를 동시에 달성할 수 있는지 확인하는 것이다.

---

## 참고문헌

- Kwon, W. et al. *Efficient Memory Management for Large Language Model Serving with PagedAttention.* SOSP, 2023.
- Gim, I. et al. *Prompt Cache: Modular Attention Reuse for Low-Latency Inference.* MLSys, 2024.
- Yao, J. et al. *CacheBlend: Fast Large Language Model Serving for RAG with Cached Knowledge Fusion.* EuroSys, 2025.
- Li, Y. et al. *SnapKV: LLM Knows What You are Looking for Before Generation.* NeurIPS, 2024.

---

*본 프로젝트는 2026년 1학기 캡스톤디자인(연구트랙) 과제로 진행되었으며, 이화여자대학교 컴퓨터공학과 14팀(def)에서 수행하였다. 코드와 실험 데이터는 [GitHub 레포](https://github.com/capstone-2026-ewha/def)에서 확인할 수 있다. 코드 작성과 디버깅 과정에서 Claude(Anthropic)의 도움을 받았으나, 실험 설계와 결과 해석, 핵심 판단은 모두 팀이 직접 진행하고 검증하였다.*
