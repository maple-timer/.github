# Maple Timer — 메이플스토리 실시간 알림 도우미

> **https://maple-timer.com**

## 프로젝트 개요

Maple Timer는 메이플스토리 플레이 화면을 **브라우저 안에서 실시간 분석**해
스킬 재설치·룬 출현·버프 종료 같은 순간을 놓치지 않게 알려주는 웹 서비스입니다.

- 사용자가 직접 선택한 **화면 공유 영역만** 분석합니다. 게임 클라이언트,
  메모리, 패킷, 키보드/마우스 입력에는 일절 접근하지 않습니다.
- 모든 감지 모델(YOLO 버프 감지, CNN 룬 감지, 숫자 OCR)은 직접 수집·학습한
  자체 모델이며, 기본 동작은 **온디바이스**(WebGPU/WASM)로 실행됩니다.
- 무거운 parser 연산은 **원격 인식**으로 전용 추론 서버에 오프로드할 수
  있어, 저사양·CPU 모드 환경에서도 정밀 감지를 사용할 수 있습니다.
- 설치가 필요 없는 웹앱으로, 브라우저 탭 하나로 동작합니다.

## 영상

https://github.com/user-attachments/assets/84975d67-0301-4757-a4f9-7913a1ad0d07

## 성과

**2026-07-26 14:19 (KST)** — 실시간 동시 접속 **1,000명**(최근 30분 기준), 알림 재생 이벤트 30분당 1.7만 회.

![GA 실시간 1,000명](https://github.com/maple-timer/.github/raw/main/profile/ga-realtime.png)

## 구조

### 메인 앱 — 모든 인식이 브라우저 안에서

핵심은 앱 자체입니다. 화면 공유 한 번으로 아래 파이프라인 전체가
사용자 기기 안에서 돌아갑니다.

```mermaid
flowchart LR
    subgraph B["Browser (on-device)"]
        direction LR
        C["Screen Share<br/>region select · viewport calibration"] --> S["Frame Sampling<br/>per-feature loops"]
        S --> W1["Buff-slot Pipeline<br/>YOLO detect → deep matcher"]
        S --> W2["Rune Detection<br/>CNN cascade"]
        S --> W3["Digit Readers<br/>cooldown · EXP OCR"]
        W1 --> J["Judgment Loop<br/>evidence · false-positive gates"]
        W2 --> J
        W3 --> J
        J --> A["Alert Scheduler"]
        A --> O1["Sound Alerts"]
        A --> O2["PiP Timer"]
    end
```

- 인식 모델은 Web Worker + onnxruntime-web(WebGPU, WASM 폴백)으로 실행됩니다.
- 판정 루프가 프레임 증거를 모아 오탐 게이트를 통과한 것만 알림으로 이어집니다.

### 서비스 아키텍처

```mermaid
flowchart LR
    subgraph CLIENT["Client"]
        USER["User"] --> APP["Maple Timer Web App<br/>on-device recognition"]
    end

    subgraph CF["Cloudflare"]
        PAGES["Pages<br/>maple-timer.com"]
        FN["Pages Functions<br/>report API"]
        BOT["Workers<br/>Discord Bot"]
        D1[("D1")]
        TUNNEL["Tunnel"]
    end

    subgraph STUDIO["Mac Studio"]
        subgraph BG["Docker — blue/green"]
            RT["Router"] --> GW["API Gateway<br/>seat-based node assignment"]
        end
        subgraph SLOTS_S["Native service — launchd a/b slots"]
            SA["slot a<br/>CoreML parser"]
            SB["slot b<br/>CoreML parser"]
        end
        subgraph MON["Monitoring"]
            ALLOY1["Alloy"] --> LOKI["Loki"]
            PROM["Prometheus"] --> GRAF["Grafana"]
            LOKI --> GRAF
        end
    end

    subgraph MINI["Mac mini"]
        subgraph SLOTS_M["Native service — launchd a/b slots"]
            MA["slot a<br/>CoreML parser"]
            MB["slot b<br/>CoreML parser"]
        end
        ALLOY2["Alloy"]
    end

    DISCORD["Discord Community"]

    APP -->|"load app & ONNX models"| PAGES
    APP -->|"send reports"| FN
    APP -->|"remote frames · VP8 1 Hz"| TUNNEL
    TUNNEL --> RT
    GW -->|"admit & infer<br/>(active generation)"| SLOTS_S
    GW -->|"admit & infer<br/>via SSH tunnel"| SLOTS_M
    BOT --> D1
    BOT -->|"issue invite codes<br/>publish patch notes"| DISCORD
    BOT -.->|"poll release feed"| PAGES
    ALLOY2 -.->|"ship event logs<br/>via SSH tunnel"| LOKI
    PROM -.->|scrape| GW

    classDef clientZone stroke:#1d6ff2,stroke-width:2px,fill:none
    classDef cfZone stroke:#f38020,stroke-width:2px,fill:none
    classDef studioZone stroke:#2da44e,stroke-width:2px,fill:none
    classDef miniZone stroke:#8250df,stroke-width:2px,fill:none
    class CLIENT clientZone
    class CF cfZone
    class STUDIO studioZone
    class MINI miniZone
```

- 원격 인식은 무거운 parser 연산만 전용 서버로 오프로드하는 선택
  기능입니다(초대 코드 기반). 게이트웨이는 각 노드의 실시간 좌석
  현황을 보고 배정하며, 노드마다 a/b 두 세대를 함께 띄워 **무중단
  롤링 업데이트**로 교체합니다.
- 두 노드의 이벤트 로그는 Alloy가 라벨을 정제해 Loki로 모으고,
  Grafana 대시보드에서 함께 조회합니다.

## 주요 기능 요약

| 기능 | 설명 |
| --- | --- |
| 스킬 알림 (정밀) | 버프칸을 분석해 솔 야누스·에르다 파운틴 등 설치기 재설치 시점을 감지 |
| 룬 알림 | 미니맵에서 룬 출현을 CNN으로 감지 |
| 사냥 멈춤 알림 | 경험치·쿨타임 변화가 멈추면 알림 |
| 버프 종료 알림 | 유니온의 부·행운, 물약, 경험치 쿠폰 종료 시점 감지 |
| 특수 코어 · 부스터 종료 | 쿨타임/판독 기반 카운트다운 알림 |
| 울티마 스쿼드 알림 | 장비 가방·보스 등장 화면 감지 |
| PiP 타이머 | 게임 위에 항상 떠 있는 소형 타이머 창 |
| 원격 인식 | 무거운 parser 연산만 전용 서버로 오프로드 — 저사양·CPU 모드 사용자 지원 (초대 코드 기반) |

## 시스템 구성 요소

| 저장소 | 역할 |
| --- | --- |
| `maple-timer` | 웹앱 본체 (Cloudflare Pages) |
| `maple-timer-api` | 원격 인식 게이트웨이 · 운영 API (Docker blue/green, 무중단 a/b 롤링) |
| `maple-timer-remote-recognition` | 네이티브 추론 서비스 (CoreML/ONNX, 2노드 좌석 풀) |
| `maple-timer-discord` | 커뮤니티 봇 — 초대 코드 발급, 패치노트 자동 게시, 문의 채널 |
| `maple-feedback-desk-operational-status` | 관리자 웹 — 문의·제보 처리, 운영 공지 편집 |
| `maple-lab-*` | 모델 학습 랩 — 버프 아이콘 추출/매칭, 룬 감지, 쿨다운 OCR |

## 기술 스택

| 영역 | 기술 |
| --- | --- |
| Frontend | React, TypeScript, Vite, onnxruntime-web (WebGPU/WASM), Web Workers |
| Edge | Cloudflare Pages · Functions · Workers · D1 |
| Gateway | Node.js, Docker blue/green, 좌석 기반 동적 노드 배정 |
| Native | Python, ONNX Runtime (CoreML), numba 전처리, launchd a/b 슬롯 |
| ML | YOLOv8n 버프 감지, CenterNet/Cascade 룬 감지, 자체 OCR — 전 모델 자체 학습 |
| Observability | Grafana, Loki, Prometheus, Alloy |
