# PAI — 환경 도메인 적응형 VLA 로봇 시스템

> 한성대학교 캡스톤 디자인 (2026.03–06), 3인 팀 프로젝트 · 정진환: 데이터 수집·학습부터 실로봇 배포까지 전 과정

자연어 명령 한 줄로 SO-101 로봇 팔이 **분리수거**(캔·플라스틱·종이)와 **식기 정리**(접시·포크·숟가락)를 하는 시스템입니다.
Gemma 4 E2B VLM이 장면을 보고 집을 물체를 문장으로 묘사하면, 그 문장을 조건으로 받은 ACT 정책이 관절 궤적을 만듭니다.
VLM, ACT 정책, ROS2 노드 전체가 **Jetson Orin Nano 8GB 한 대**에서 클라우드 없이 돌아갑니다.

![실로봇 동작](docs/demo.gif)

## 한눈에 보기

| 항목 | 내용 |
|---|---|
| 도메인 | 분리수거, 식기 정리 (도메인별 Gemma LoRA + 도메인별 ACT 정책) |
| ACT 학습 데이터 | 실물 teleop 시연 — 분리수거 571 에피소드, 식기 정리 233 에피소드 |
| VLM | Gemma 4 E2B (GGUF Q4) + 도메인 LoRA hot-swap, llama.cpp로 약 28 tok/s |
| 배포 | ROS2 Humble 노드 6개 + llama-server가 Jetson Orin Nano 8GB에 공존 |
| 시뮬레이션 | Isaac Sim 5.1 SO-101 디지털 트윈 + Domain Randomization (sim-to-real은 시도 단계) |

## 동작 방식

1. **도메인 선택** — 사용자 명령을 [`domain_map.yaml`](deploy/config/domain_map.yaml)의 키워드와 매칭해 도메인을 고르고, llama-server에 해당 LoRA를 hot-swap합니다.
2. **장면 보기 (`look_scene`)** — Gemma가 RGB 이미지를 보고 타겟을 `"the {attribute} {object} {spatial}"` 형식의 영어 문장으로 묘사합니다. (예: `"the blue can on the right side of the table"`)
3. **임베딩** — `embedding_node`가 문장을 384차원 벡터로 바꿉니다 (`paraphrase-multilingual-MiniLM-L12-v2`).
4. **실행 (`execute_subtask`)** — `act_node`가 RGB + 관절 상태 + 언어 임베딩으로 관절 궤적을 생성합니다 (chunk 100, temporal ensemble).
5. **검증 (`verify_result`)** — 팔을 home 자세로 돌린 뒤, Gemma의 시각 판정과 **목적지 ROI의 depth 변화**를 합쳐 성공 여부를 판단합니다. depth는 실패를 거부할 권한만 갖습니다(단방향 veto).
6. **재시도** — 실패하면 최대 3회 다시 시도하고, 그래도 안 되면 사용자 개입을 기다립니다.

## ROS2 노드

| 노드 | 역할 |
|---|---|
| `camera_node` | RealSense RGB-D 프레임 발행 (30 Hz) |
| `agent_node` | Gemma 함수 호출 루프 `look_scene → execute_subtask → verify_result` 오케스트레이션 |
| `llama_server_manager` | llama-server `/lora-adapters` HTTP로 도메인 LoRA hot-swap |
| `embedding_node` | 문장 → 벡터 임베딩 서비스 (`/embed_text`) |
| `act_node` | 도메인별 ACT 정책 추론 (30 Hz) |
| `robot_controller_node` | SO-101 팔로워 모터 제어 (1000 Hz), `/go_home` 서비스 |

커스텀 메시지는 `robot_arm_msgs` 패키지에 있습니다 (`SubtaskCommand`, `ExecutionStatus`, `DomainSelection`, `AgentLog`, `EmbedText`).

## 학습

**ACT 정책 (실물 데이터만 사용)**
- SO-101 Leader–Follower teleop으로 시연을 수집했습니다. 라벨은 [`label_vocab.yaml`](deploy/config/label_vocab.yaml) 어휘 안에서만 고르게 해 학습·배포 문장 분포를 맞췄습니다.
- 한 장면에 후보 물체를 여러 개 두고 명령에 따라 다른 물체를 집게 해서, 정책이 언어를 무시하고 화면만 보는 shortcut을 막으려 했습니다.
- LeRobot 0.4.4로 도메인별 정책 2개를 학습했습니다.

**Gemma 도메인 LoRA**
- 함수 호출 시나리오와 검증 시나리오 Q&A를 70:30으로 섞어 도메인별 데이터셋을 만들었습니다.
- Unsloth QLoRA로 학습하고 GGUF로 변환해 llama-server에 올립니다.

**Isaac Sim 디지털 트윈**
- SO-101 URDF를 USD로 옮기고, 실물 Leader로 시뮬레이션 follower를 조종하는 teleop 환경과 Domain Randomization(조명·카메라·물체 질량)을 만들었습니다.
- 처음 계획은 Sim + Real co-training이었지만, Isaac Sim에서 그리퍼와 물체의 접촉 물리를 안정시키지 못해 **ACT는 실물 데이터만으로 학습**했습니다. co-training 스크립트는 뼈대만 남아 있습니다.

## 해결한 문제

- **ACT가 언어 입력을 조용히 무시하던 문제** — LeRobot ACT는 `observation.environment_state`만 ENV 입력으로 배선하고, 우리가 쓰던 `observation.language_embedding`은 STATE로 분류해 버렸습니다. 임베딩을 바꿔도 출력이 같다는 검증 실험으로 원인을 찾았고, 키 이름을 바꾸는 [`rename_feature.py`](training/imitation_data/scripts/rename_feature.py)로 고친 뒤 다시 학습했습니다.
- **8GB 보드의 GPU 메모리 단편화** — 메모리 총량은 충분한데 `NvMap error 12 (ENOMEM)`로 로드가 실패했습니다. PyTorch `expandable_segments`, 임베딩 노드 CPU 실행, 커널 VM 튜닝, GUI 끄기와 MAXN 전원 모드로 6개 노드와 llama-server를 한 보드에 같이 올렸습니다.
- **Jetson에서 Gemma 4 멀티모달 구동** — 여러 설치 방식이 실패한 뒤, llama.cpp를 다시 빌드하고 NVIDIA 권장 플래그를 조합해 텍스트와 이미지 입력을 약 28 tok/s로 돌렸습니다.

## 결과와 한계

실로봇에서 전체 파이프라인이 끝까지 동작하는 것까지 확인했지만, 배치 성공률은 아직 낮습니다.

| 도메인 | place 성공 |
|---|---|
| 분리수거 | 5 / 20 |
| 식기 정리 | 3 / 20 |

분석한 원인은 다음과 같습니다.
- 정책이 일반화보다 특정 자세를 외우는 쪽으로 학습됐습니다.
- 좌우 정렬이 자주 어긋났습니다.
- 수집 세션마다 카메라 위치가 조금씩 달랐습니다.
- 실행은 subtask 단위인데 학습 데이터는 에피소드 전체 단위라 구조가 맞지 않았습니다.

도메인 선택은 자율 인식이 아니라 키워드 매칭이고, `verify_result`의 판정 규칙(Gemma 우선 + depth veto)은 1차 버전입니다.

## 저장소 구조

```
PAI/
├── deploy/      # Jetson 배포: ROS2 패키지, 런타임 설정, 캘리브레이션, llama-server systemd
├── training/    # 시연 데이터 수집·라벨링, Gemma LoRA 파인튜닝, ACT 학습
└── sim/         # Isaac Sim 자산·씬, Domain Randomization 설정, teleop 브리지
```

- 배포·실행 방법: [`deploy/README.md`](deploy/README.md)
- 데이터 수집·학습 방법: [`training/README.md`](training/README.md)

## 실행 환경

| 구분 | 내용 |
|---|---|
| 로봇 | SO-101 Follower + Leader (LeRobot), Intel RealSense D435 |
| 엣지 | Jetson Orin Nano 8GB, JetPack 6.2.1, Ubuntu 22.04, ROS2 Humble, Python 3.10 |
| 학습 PC | Ubuntu + NVIDIA GPU (ACT·LoRA 학습, Isaac Sim 5.1) |
