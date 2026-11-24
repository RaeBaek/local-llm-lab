# 로컬 LLM 실험실

Apple Silicon Mac에서 **Ollama + Qwen2.5 7B + Open WebUI**를 구성하고, Python 코드와 브라우저 양쪽에서 같은 모델을 사용해 봅니다.

이미 학습된 모델을 다운로드하여 실행하는 실습입니다. 모델을 새로 학습시키는 과정은 포함하지 않습니다.

## 완성할 구성

```text
Python 코드 ── OpenAI 호환 API ──┐
                              ↓
                         Mac의 Ollama
                              ↓
                       Qwen2.5 7B 모델
                              ↑
브라우저 ── Open WebUI ── Ollama API ─┘
            (Docker)
```

| 구성 요소 | 역할 |
| --- | --- |
| Qwen2.5 7B | 질문에 대한 답변을 생성하는 모델 |
| Ollama | 모델 다운로드·실행 및 API 제공 |
| OpenAI Python SDK | Python 코드에서 Ollama API 호출 |
| Open WebUI | 브라우저에서 사용하는 채팅 화면 |
| Docker Desktop | Open WebUI 컨테이너 실행 |

OpenAI SDK를 사용하지만, 아래 코드의 요청 대상은 로컬 Ollama입니다. 실제 답변은 Mac의 Qwen이 생성하며 OpenAI API 키는 필요하지 않습니다.

## 준비 사항

- Apple Silicon Mac
- Python 3와 터미널 앱(iTerm2 또는 Terminal)
- 모델과 Docker 이미지 다운로드를 위한 인터넷 연결 및 디스크 여유 공간
- 모델을 실행할 수 있는 메모리 여유

`qwen2.5:7b` 모델 파일은 약 4.7GB입니다. 실행 중에는 모델 외에 문맥 처리와 작업 공간을 위한 메모리가 추가로 필요합니다. 이 실습에서 관찰한 실행 메모리는 6.4GB였지만, 설정과 버전에 따라 달라집니다. 실제 전체 메모리 요구량은 모델 파일 크기와 차이가 있습니다.

로컬 모델 추론에는 외부 AI API 사용량 요금이 발생하지 않습니다. 장비·전기 비용은 별도이며, 외부 API나 검색 기능을 추가하면 데이터 전송과 비용 조건이 달라질 수 있습니다.

## 1. Ollama 설치 및 실행

[Ollama 다운로드](https://ollama.com/download)에서 macOS 버전을 설치하고 앱을 실행합니다.

```bash
open -a Ollama
```

터미널에서 연결을 확인합니다.

```bash
ollama list
```

아직 모델을 받지 않았다면 목록이 비어 있어도 정상입니다.

## 2. 모델 다운로드 및 터미널 대화

```bash
ollama pull qwen2.5:7b
ollama run qwen2.5:7b
```

입력 대기 화면에서 질문합니다.

```text
Swift의 Optional을 간단한 코드 예시와 함께 설명해 줘.
```

답변이 나오면 로컬 모델 실행에 성공한 것입니다. 대화를 종료하려면 다음을 입력합니다.

```text
/bye
```

다운로드한 모델과 메모리에 올라간 모델은 각각 다음으로 확인합니다.

```bash
ollama list
ollama ps
```

| 항목 | 의미 |
| --- | --- |
| `ollama list`의 `SIZE` | 디스크에 저장된 모델 크기 |
| `ollama ps`의 `SIZE` | 실행 중인 모델의 메모리 사용 정보 |
| `PROCESSOR`의 `100% GPU` | 모델이 GPU에 모두 올라간 상태. GPU 사용률 100%라는 뜻은 아님 |
| `CONTEXT` | 현재 설정된 최대 문맥 길이(토큰 단위) |
| `UNTIL` | 추가 요청이 없을 때 모델을 메모리에서 내릴 예정 시점 |

문맥은 질문, 이전 대화 등 모델이 답변에 참고하는 내용의 범위입니다. 길이를 늘리면 메모리 요구량도 증가합니다. 토큰 수는 한국어 글자 수나 단어 수와 일치하지 않습니다.

## 3. Python에서 같은 모델 호출

### 실험 폴더 및 가상환경 준비

```bash
mkdir -p ~/Documents/local-llm-lab
cd ~/Documents/local-llm-lab

python3 -m venv .venv
source .venv/bin/activate
python -m pip install openai
```

이미 저장소를 내려받았다면 해당 저장소 폴더에서 가상환경을 만들어도 됩니다. 가상환경 폴더 `.venv/`는 Git에 포함하지 않습니다.

### chat.py 작성

작업 폴더에 `chat.py` 파일을 만들고 저장합니다.

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="unused",
)

response = client.chat.completions.create(
    model="qwen2.5:7b",
    messages=[
        {
            "role": "user",
            "content": "Swift의 Optional을 간단한 코드 예시와 함께 설명해 줘.",
        }
    ],
)

print(response.choices[0].message.content)
```

| 설정 | 의미 |
| --- | --- |
| `localhost` | Python 코드를 실행하는 Mac |
| `11434` | Ollama API 포트 |
| `/v1` | OpenAI 호환 API 경로 |
| `api_key="unused"` | SDK에 필요한 자리 채움 값. 로컬 Ollama는 이 값을 무시 |
| `model` | Ollama에 다운로드한 모델 이름 |

### 실행

Ollama가 켜진 상태에서 실행합니다.

```bash
python chat.py
```

터미널에 답변이 표시되면 코드에서 로컬 모델을 호출하는 데 성공한 것입니다.

이 코드는 실행할 때마다 한 번의 질문만 전송합니다. 이전 실행의 대화를 기억하거나 질문·답변을 파일에 저장하지 않습니다. 여러 차례의 대화를 구현하려면 이전 메시지도 `messages`에 함께 전달해야 합니다.

## 4. Docker Desktop 설치

[Docker Desktop 설치 안내](https://docs.docker.com/desktop/setup/install/mac-install/)에서 **Mac with Apple silicon** 버전을 설치하고 실행합니다.

설치 후 새 터미널 탭을 열어 확인합니다.

```bash
docker info
```

Client와 Server 정보가 표시되면 준비된 것입니다.

### docker: command not found가 나오는 경우

먼저 새 터미널 탭에서 다시 시도합니다. 설치 전에 열어 둔 탭에는 변경된 검색 경로가 반영되지 않을 수 있습니다.

계속 실패하고 Docker CLI가 `~/.docker/bin`에 설치되어 있다면 현재 탭에 경로를 추가합니다.

```bash
export PATH="$HOME/.docker/bin:$PATH"
docker info
```

이 방법으로 해결된다면 `~/.zshrc`에 다음 줄을 한 번 추가하여 새 터미널에도 적용합니다.

```bash
export PATH="$HOME/.docker/bin:$PATH"
```

명령은 인식되지만 Docker 서버에 연결하지 못하면 Docker Desktop이 실행 중이고 준비가 끝났는지 확인합니다.

## 5. Open WebUI 실행

Ollama와 Docker Desktop이 켜진 상태에서 다음 명령을 **최초 한 번** 실행합니다. 작업 폴더와 관계없이 실행할 수 있습니다.

```bash
docker run -d \
  --name open-webui \
  -p 127.0.0.1:3000:8080 \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -e WEBUI_SECRET_KEY="$(openssl rand -hex 32)" \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

| 설정 | 역할 |
| --- | --- |
| `-d` | 백그라운드에서 실행 |
| `--name open-webui` | 컨테이너 이름 지정 |
| `127.0.0.1:3000:8080` | 내 Mac에서만 접근 가능한 3000번 포트를 컨테이너의 8080번 포트에 연결 |
| `OLLAMA_BASE_URL` | 컨테이너에서 Mac의 Ollama에 접근할 주소 |
| `WEBUI_SECRET_KEY` | 세션 등에 사용하는 비밀 키. 이 명령으로 생성한 컨테이너를 재시작해도 유지됨 |
| `-v open-webui:/app/backend/data` | 대화·계정·설정을 Docker 볼륨에 저장 |

컨테이너 안의 `localhost`는 컨테이너 자신을 가리킵니다. Mac에서 실행 중인 Ollama에는 `host.docker.internal`로 접근합니다.

처음에는 이미지 다운로드와 초기 준비에 시간이 걸립니다. 로그는 다음으로 확인합니다.

```bash
docker logs -f open-webui
```

로그 확인을 끝내려면 `Control+C`를 누릅니다. 컨테이너는 계속 실행됩니다.

> 이 예제는 이동하는 `main` 이미지 태그를 사용하므로 설치 시점에 따라 화면이나 동작이 달라질 수 있습니다. 재현 가능한 환경을 배포하려면 검증한 이미지 버전이나 다이제스트와 Python 패키지 버전을 고정하세요. 컨테이너를 새로 만들 때는 기존 비밀 키도 유지해야 합니다.

## 6. 브라우저에서 대화

1. Chrome 또는 Safari에서 [http://localhost:3000](http://localhost:3000)에 접속합니다.
2. 초기 안내에 따라 로컬 관리자 계정을 만듭니다.
3. 모델 선택 메뉴에서 `qwen2.5:7b`를 선택합니다.
4. 질문을 보내 답변을 확인합니다.

대화 문맥도 확인해 봅니다.

```text
첫 질문: Swift의 Optional을 간단히 설명해 줘.
후속 질문: 방금 설명한 개념을 사용하는 짧은 코드 예시를 보여 줘.
```

모델이 보이지 않으면 관리자 설정의 **Connections → Ollama**에서 연결 주소를 확인합니다. 메뉴 이름은 버전에 따라 다를 수 있습니다.

```text
http://host.docker.internal:11434
```

이 연결은 Ollama 전용 API를 사용하므로 `/v1`을 붙이지 않습니다.

## 7. 다음에 다시 사용할 때

### 웹 채팅 사용

1. Ollama 앱을 실행합니다.
2. Docker Desktop을 실행하고 준비될 때까지 기다립니다.
3. 기존 컨테이너를 시작합니다.

```bash
docker start open-webui
```

4. [http://localhost:3000](http://localhost:3000)에 접속합니다.

처음 설치할 때의 `docker run`을 반복할 필요는 없습니다. 이미 같은 이름의 컨테이너가 있다면 이름 충돌 오류가 발생합니다.

### Python만 사용

Ollama를 실행한 뒤 다음을 실행합니다. Docker는 필요하지 않습니다.

```bash
cd ~/Documents/local-llm-lab
source .venv/bin/activate
python chat.py
```

모델이 메모리에서 내려가 있더라도 요청을 받으면 다시 로드됩니다. 첫 응답에는 로딩 시간이 추가될 수 있습니다.

## 8. 사용 종료 및 메모리 정리

```bash
# Open WebUI 컨테이너 중지
docker stop open-webui

# Qwen 모델을 메모리에서 내리기
ollama stop qwen2.5:7b
```

추가로 메모리를 확보하려면 Mac 메뉴 막대의 Docker 아이콘에서 **Quit Docker Desktop**을 선택합니다. Ollama 앱도 사용하지 않으면 종료할 수 있습니다.

| 동작 | 영향 |
| --- | --- |
| `docker stop open-webui` | 웹 채팅 앱 실행 중지. Ollama 모델은 별도로 관리 |
| `ollama stop qwen2.5:7b` | 모델을 메모리에서 내림. 모델 파일은 유지 |
| Docker Desktop 종료 | Docker의 가상머신과 백그라운드 실행 종료 |

## 9. 데이터 저장 위치

| 데이터 | 위치 |
| --- | --- |
| Qwen 모델 파일 | 기본값 `~/.ollama/models` |
| Open WebUI 대화·계정·설정 | Docker의 `open-webui` 볼륨 |
| Python 코드 및 가상환경 | 이 안내에서는 `~/Documents/local-llm-lab` |

모델 폴더를 Finder에서 열려면 다음을 실행합니다.

```bash
open ~/.ollama/models
```

Open WebUI 데이터는 컨테이너 안에서 `/app/backend/data`로 보입니다. Mac에서는 Docker의 가상머신 디스크 내부에 있으므로 일반 폴더처럼 Finder에서 바로 열지 않습니다. Docker Desktop의 **Volumes → open-webui**에서 확인할 수 있습니다.

앱과 컨테이너를 중지해도 모델 파일과 볼륨 데이터는 유지됩니다. 볼륨을 삭제하거나 Docker 데이터를 초기화하면 대화 기록을 잃을 수 있습니다.

Python과 Open WebUI는 같은 모델을 사용하지만 대화 기록은 자동 공유되지 않습니다. 대화는 모델의 학습 데이터나 가중치에 자동 반영되지 않습니다.

## 문제 해결

| 증상 | 확인할 내용 |
| --- | --- |
| `ollama` 명령을 찾지 못함 | 설치 완료 후 새 터미널에서 재시도 |
| Ollama 연결 오류 | Ollama 앱 실행 여부 확인 |
| `model not found` | `ollama list` 확인 후 `ollama pull qwen2.5:7b` |
| `No module named 'openai'` | `.venv` 활성화 후 `python -m pip install openai` |
| Docker 서버 연결 오류 | Docker Desktop 실행 및 준비 상태 확인 |
| `open-webui` 이름 충돌 | `docker ps -a`로 확인 후 기존 컨테이너를 `docker start open-webui`로 시작 |
| 브라우저 접속 실패 | `docker ps -a`와 `docker logs open-webui` 확인 |
| Open WebUI에 모델이 안 보임 | Ollama 실행 여부와 Ollama 연결 주소 확인 |
| 답변이 느리거나 메모리가 부족함 | 다른 앱을 정리하고 문맥 길이를 줄이거나 더 작은 모델 사용 |

## 완료 기준

- [ ] `ollama run qwen2.5:7b`로 터미널에서 답변을 받는다.
- [ ] `python chat.py`로 같은 모델의 답변을 받는다.
- [ ] Open WebUI에서 `qwen2.5:7b`를 선택하여 대화한다.
- [ ] 컨테이너 중지·재시작 후 기존 대화 기록이 유지된다.
- [ ] `ollama stop qwen2.5:7b` 후 `ollama ps`에서 모델이 내려간 것을 확인한다.

이 안내는 개인용 Mac에서 기본 환경을 구성하는 범위입니다. vLLM 전환이나 팀 공유용 게이트웨이는 포함하지 않습니다.

## 공식 문서

- [Ollama 다운로드](https://ollama.com/download)
- [Ollama FAQ — 저장 위치와 모델 메모리 관리](https://docs.ollama.com/faq)
- [Ollama OpenAI 호환 API](https://docs.ollama.com/api/openai-compatibility)
- [Qwen2.5 7B 모델 정보](https://ollama.com/library/qwen2.5:7b)
- [Docker Desktop for Mac 설치](https://docs.docker.com/desktop/setup/install/mac-install/)
- [Open WebUI 설치](https://docs.openwebui.com/getting-started/quick-start/)
- [Open WebUI의 Ollama 연결](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama/)
- [Docker Desktop for Mac 데이터 저장 방식](https://docs.docker.com/desktop/troubleshoot-and-support/faqs/macfaqs/)
