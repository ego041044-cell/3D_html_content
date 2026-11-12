# Three.js 디지털 트윈 HTML 분석 및 실행 가이드

본 문서는 첨부된 두 개의 단일 파일 Three.js 웹 애플리케이션(`smart_classroom_camera_stable.html`, `ThreeJS_korean_bus.html`)의 **실행 방법**, **공통 아키텍처**, 그리고 **핵심 문법 구조**를 상세히 분석한 기술 가이드입니다.

---

## 1. 파일 개요

두 파일 모두 외부 3D 모델(`.glb`, `.gltf`)이나 외부 이미지 파일 없이 **순수 JavaScript(Three.js ES Modules)의 기하학 도형(Procedural Geometry)과 HTML5 Canvas 동적 텍스처**만으로 구현된 단일 파일(Single-File) 디지털 트윈 시뮬레이터입니다.

| 구분 | `smart_classroom_camera_stable.html` | `ThreeJS_korean_bus.html` |
| :--- | :--- | :--- |
| **주제** | 스마트 학교 교실 환경/IoT 디지털 트윈 | 한국형 저상 전기 시내버스 조립공장 디지털 트윈 |
| **Three.js 버전** | `r164.1` (CDN ES Module) | `r180.0` (CDN ES Module) |
| **핵심 기능** | 온도·$CO_2$·조도 실시간 연산, 시나리오 제어, JSON/CSV/PNG 내보내기 | 8단계 순차 조립 애니메이션, 분해 보기(Explode), X-RAY 투시, 부품 클릭(Raycaster) |
| **특이사항** | 카메라 이동 시 벽면 자동 절개(Wall Cutaway) 및 Blender 연동용 스펙 내장 | 조립 공정 상태 머신(State Machine) 및 오프라인 대비 2D SVG Fallback 화면 내장 |

---

## 2. 실행 방법

두 파일 모두 `<script type="importmap">`과 ES Module(`<script type="module">`)을 사용하며, CDN에서 Three.js 라이브러리를 불러옵니다.

### ① 가장 간편한 실행 방법 (더블 클릭)
1. 인터넷이 연결된 상태에서 각 `.html` 파일을 더블 클릭하여 **Chrome, Edge, Whale 등 최신 브라우저**로 엽니다.
2. 별도의 빌드 과정이나 패키지 설치 없이 즉시 3D 화면이 렌더링됩니다.

### ② 로컬 웹 서버를 통한 실행 (권장)
브라우저의 보안 정책(CORS)이나 안정적인 모듈 로딩을 위해 로컬 서버를 통해 실행하는 것이 가장 좋습니다.
* **VS Code 사용자**: `Live Server` 확장 프로그램 설치 후, HTML 파일 우클릭 $\rightarrow$ **Open with Live Server** 클릭
* **Python 설치 환경**: 터미널(CMD)에서 파일이 있는 폴더로 이동 후 아래 명령어 실행
  ```bash
  python -m http.server 8000
  ```
  이후 브라우저 주소창에 `http://localhost:8000/smart_classroom_camera_stable.html` 입력

> **⚠️ 주의사항**: 최초 실행 시 CDN(`cdn.jsdelivr.net`)에서 Three.js 모듈을 다운로드하므로 **인터넷 연결**이 필요합니다. 연결이 차단된 환경에서는 로딩 화면 이후 오류 진단 안내창(또는 2D SVG 대체 화면)이 표시됩니다.

---

## 3. 전체적인 코드 및 문법 구조

두 파일은 유지보수와 이식성을 극대화하기 위해 **HTML + CSS + JS가 하나의 파일에 통합된 5단계 계층 구조**를 따릅니다.

```text
[1. <head> 메타데이터 및 CSS 스타일시트]
       │
[2. <script type="importmap"> 모듈 경로 매핑]
       │
[3. <body> UI 레이아웃 및 3D 캔버스 컨테이너]
       │
[4. <script> 일반 스크립트 (초기 로딩 예외 처리)]
       │
[5. <script type="module"> Three.js 메인 엔진 로직]
       ├── ① 초기화 (Scene, Camera, Renderer, OrbitControls, Light)
       ├── ② 헬퍼 팩토리 함수 (Mesh 생성 및 Canvas 텍스처 생성)
       ├── ③ 3D 공간 및 오브젝트 계층 조립 (Group 트리 구조)
       ├── ④ 상태 관리(State) 및 인터랙션 이벤트 바인딩
       └── ⑤ 애니메이션 루프 (requestAnimationFrame / setAnimationLoop)
```

### 3.1. ES Module 및 Import Map 설정
빌드 도구(Webpack, Vite 등) 없이 브라우저에서 `import * as THREE from 'three'` 문법을 사용하기 위해 `<script type="importmap">`을 선언합니다.

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.180.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.180.0/examples/jsm/"
  }
}
</script>
```

### 3.2. UI 레이아웃 및 반응형 CSS 구조
* **교실 파일 (`smart_classroom`)**: 전체 화면(`position: fixed; inset: 0`) 3D 캔버스 위에 `position: absolute`와 `backdrop-filter: blur()`를 적용한 글래스모피즘(Glassmorphism) 플로팅 패널을 배치했습니다.
* **버스 파일 (`korean_bus`)**: CSS Grid(`grid-template-columns: 292px minmax(0,1fr) 272px`)를 사용해 **[좌측 공정 제어 - 중앙 3D 뷰포트 - 우측 관제 패널]**의 3단 대시보드 레이아웃을 구성했습니다.
* 두 파일 모두 `@media` 쿼리를 통해 태블릿 및 모바일 화면에서 패널이 슬라이드 드로어(Drawer) 형태로 전환되도록 설계되었습니다.

### 3.3. 핵심 Three.js 문법 및 디자인 패턴

#### ① 절차적 기하학 헬퍼 함수 (Procedural Geometry Helpers)
수백 개의 3D 부품을 효율적으로 조립하기 위해 `BoxGeometry`, `CylinderGeometry`, `SphereGeometry`, `TubeGeometry` 생성과 그림자 설정, 부모 그룹 추가를 한 번에 처리하는 래퍼(Wrapper) 함수를 정의하여 사용합니다.

```javascript
// 버스 파일의 박스 생성 헬퍼 예시
function bx(parent, w, h, d, mat, x = 0, y = 0, z = 0) {
  const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), mat);
  m.position.set(x, y, z);
  m.castShadow = true;
  m.receiveShadow = true;
  parent.add(m);
  return m;
}
```

#### ② HTML5 Canvas 기반 동적 텍스처 (`THREE.CanvasTexture`)
외부 이미지 파일 없이 전자칠판의 실시간 센서 수치, 버스의 행선지 전광판(`701 서울 시내버스`), 공장 안전 표지판 등을 구현하기 위해 메모리 상의 `<canvas>`에 2D 그래픽을 그린 뒤 3D 재질의 `map`으로 주입합니다.

```javascript
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
// ctx.fillText(...) 등으로 텍스트 및 그래프 렌더링
const texture = new THREE.CanvasTexture(canvas);
texture.colorSpace = THREE.SRGBColorSpace;
```

#### ③ 계층적 그룹화 (`THREE.Group`) 및 상태 제어
* **교실 파일**: `School_Architecture`, `Window_System`, `Student_Desks_24`, `IoT_Sensors` 등 기능별 그룹으로 트리 구조를 형성하며, 카메라 좌표가 벽면 바깥으로 이동하면 `wallCutaway` 로직이 작동해 앞벽과 창가 벽의 `visible` 속성을 자동으로 끕니다.
* **버스 파일**: 버스의 각 부품을 `part()` 함수로 등록할 때 `stage`(조립 단계), `from`(진입 시작 좌표), `explode`(분해 시 이동 좌표) 벡터를 함께 저장합니다. 이후 렌더링 루프에서 `Vector3.lerp()` 보간 함수를 이용해 부드러운 조립 및 분해 애니메이션을 구현합니다.

#### ④ 광선 투사(Raycasting) 및 카메라 보간(Tweening)
* **부품 클릭 감지**: `THREE.Raycaster`와 정규화된 장치 좌표(NDC, $-1 \sim +1$)를 활용해 사용자가 3D 버스 부품을 클릭하면 해당 `partId`를 찾아 우측 패널에 상세 설명을 출력합니다.
* **시점 전환**: 프리셋 버튼 클릭 시 `camera.position`과 `controls.target`을 목표 좌표로 선형 보간(`lerp` / Cubic Ease)하여 부드럽게 이동시킵니다.

#### ⑤ 방어적 예외 처리 (Fallback & Context Loss)
* 3D 모듈 로딩이 12~14초 이상 지연되거나 WebGL 초기화에 실패할 경우, 전역 플래그(`window.__busReady`, `window.__SMART_CLASSROOM_READY__`)를 검사하여 사용자에게 명확한 오류 원인과 재시도 UI를 제공합니다.
* 교실 파일에서는 GPU 컨텍스트 손실(`webglcontextlost`) 이벤트 리스너까지 구현되어 안정성을 높였습니다.
```

---

두 파일의 실행 방법과 핵심 문법 구조를 정리한 마크다운 문서를 생성해 드렸습니다. 특정 코드 블록의 상세한 작동 원리나 두 파일의 기능을 결합하는 방법이 필요하시면 말씀해 주세요!