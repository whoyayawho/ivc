## 📋 목차

- [개요](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [주요 기능](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [시스템 아키텍처](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [설치 방법](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [사용 방법](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [ROS2 인터페이스](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [파라미터 설정](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)
- [기술 스택](https://www.notion.so/3D-Human-Detection-and-Tracking-System-264488fcddd08080a9bcf9c127860a83?pvs=21)

## 개요

본 패키지는 ROS2 환경에서 카메라와 라이다 센서 데이터를 융합하여 실시간으로 사람을 감지, 추적하고 제스처를 인식하는 통합 시스템입니다. YOLOv8, SAM2, MediaPipe 등 최신 딥러닝 모델을 활용하며, Kalman Filter 기반 다중 객체 추적을 통해 안정적인 3D 궤적 추적을 제공합니다.

## 주요 기능

### 1. 🎯 객체 감지 및 세그멘테이션

- **YOLOv8**: 실시간 객체 감지 (person 클래스)
- **SAM2 (Segment Anything Model 2)**: 정밀한 인스턴스 세그멘테이션
- 바운딩 박스 및 마스크 기반 3D 포인트 추출

### 2. 🏃 스켈레톤 및 제스처 인식

- **MediaPipe Pose**: 33개 랜드마크 기반 스켈레톤 감지
- 손 들기 제스처 인식 (양손/한손 구분)
- 실시간 스켈레톤 시각화

### 3. 📍 다중 객체 추적 (MOT)

- **Kalman Filter**: 6차원 상태 공간 (위치 + 속도) 예측
- **Hungarian Algorithm**: 최적 데이터 연관
- 개인별 고유 ID 부여 및 색상 구분
- 3D 궤적 기록 및 시각화

### 4. 🌐 3D 포인트 클라우드 처리

- 카메라-라이다 캘리브레이션 기반 센서 융합
- DBSCAN 클러스터링을 통한 노이즈 제거
- Statistical Outlier Removal (SOR) 필터링
- 색상 매핑된 3D 포인트 클라우드 생성

### 5. 📊 RViz2 3D 시각화

- 3D 바운딩 박스 마커
- 실시간 궤적 라인 표시
- 제스처 상태 텍스트 표시
- 추적 ID 표시

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     DetectHumanNode (ROS2)                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐      ┌──────────────┐                     │
│  │ Image Input  │      │PointCloud    │                     │
│  │ (Compressed) │      │   Input      │                     │
│  └──────┬───────┘      └──────┬───────┘                     │
│         │                      │                             │
│         ▼                      ▼                             │
│  ┌─────────────────────────────────────┐                    │
│  │    Synchronization Buffer            │                    │
│  │    (25ms tolerance)                  │                    │
│  └──────────────┬──────────────────────┘                    │
│                 ▼                                            │
│  ┌─────────────────────────────────────────────────┐        │
│  │            Detection Pipeline                     │        │
│  ├───────────────────────────────────────────────────┤       │
│  │                                                   │       │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │       │
│  │  │  YOLOv8  │→ │   SAM2   │→ │MediaPipe │      │       │
│  │  │Detection │  │Segmentation│ │  Pose    │      │       │
│  │  └──────────┘  └──────────┘  └──────────┘      │       │
│  └─────────────────────┬─────────────────────────┘        │
│                        ▼                                    │
│  ┌─────────────────────────────────────────────────┐       │
│  │          Multi-Person Tracker                    │       │
│  ├───────────────────────────────────────────────────┤      │
│  │  ┌──────────────┐      ┌──────────────┐         │      │
│  │  │Kalman Filter │  ←→  │  Hungarian   │         │      │
│  │  │  (per ID)    │      │  Algorithm   │         │      │
│  │  └──────────────┘      └──────────────┘         │      │
│  └─────────────────────┬─────────────────────────┘       │
│                        ▼                                   │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Output Publishers                     │    │
│  ├────────────────────────────────────────────────────┤   │
│  │ • /human_detections (Detection2DArray)             │   │
│  │ • /human_segmentation_image (Image)                │   │
│  │ • /human_pointcloud (PointCloud2)                  │   │
│  │ • /masked_pointcloud (PointCloud2)                 │   │
│  │ • /human_bounding_boxes (MarkerArray)              │   │
│  │ • /human_trajectories (MarkerArray)                │   │
│  │ • /yolo_detection_image (Image)                    │   │
│  │ • /projected_image (Image)                         │   │
│  │ • /colored_pointcloud (PointCloud2)                │   │
│  └────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

```

### 클래스 구조

### PersonTracker

- 개별 사람 추적을 위한 Kalman Filter 기반 트래커
- 6차원 상태 벡터: [x, y, z, vx, vy, vz]
- 궤적 히스토리 관리 (최대 100포인트)

### MultiPersonTracker

- 다중 PersonTracker 관리
- Hungarian Algorithm을 통한 detection-to-tracker 매칭
- 트래커 생성/삭제 관리

### DetectHumanNode

- 메인 ROS2 노드
- 센서 데이터 동기화
- AI 모델 통합 및 실행
- 결과 퍼블리싱

## 프로젝트 구조

```
detect_human/
├── config/
│   └── detection_params.yaml      # 감지 및 추적 파라미터 설정
├── data/
│   └── calibration_params.json    # 카메라-LiDAR 캘리브레이션 파라미터
├── detect_human/
│   ├── __init__.py
│   ├── detect_human_node.py       # 기본 융합 노드 (감지 미포함)
│   ├── detect_human_node_with_detection.py  # AI 통합 메인 노드
│   └── sam2/                      # SAM2 모델 디렉토리
├── launch/
│   └── detect_human.launch.py     # RViz2 통합 런치 파일
├── models/                         # AI 모델 가중치
│   └── sam2.1_hiera_base_plus.pt
├── resource/
│   └── detect_human               # ROS2 패키지 마커
├── rviz/
│   └── config.rviz                # RViz2 시각화 설정
├── package.xml                    # 패키지 의존성
├── setup.py                       # Python 패키지 설정
└── setup.cfg                      # 빌드 설정

```

## 설치 방법

### 1. 시스템 요구사항

- Ubuntu 22.04
- ROS2 Humble
- Python 3.10+
- CUDA 11.8+ (GPU 사용 시)

### 2. 의존성 패키지 설치

```bash
# ROS2 패키지
sudo apt update
sudo apt install ros-humble-vision-msgs ros-humble-visualization-msgs

# Python 패키지
pip install torch torchvision ultralytics mediapipe opencv-python scipy open3d

```

### 3. SAM2 모델 설치

```bash
cd /root/ros2_ws/src/detect_human/detect_human
git clone <https://github.com/facebookresearch/segment-anything-2.git> sam2
cd sam2
pip install -e .

# 모델 가중치 다운로드
cd /root/ros2_ws/src/detect_human/models
wget <https://dl.fbaipublicfiles.com/segment_anything_2/072824/sam2.1_hiera_base_plus.pt>

```

### 4. 패키지 빌드

```bash
cd /root/ros2_ws
colcon build --packages-select detect_human
source install/setup.bash

```

## 사용 방법

### 1. 기본 실행

```bash
# Launch 파일로 실행 (RViz2 포함)
ros2 launch detect_human detect_human.launch.py

# 또는 노드만 실행
ros2 run detect_human detect_human_node_with_detection

```

### 2. 파라미터 수정하여 실행

```bash
# YAML 설정 파일 수정
nano /root/ros2_ws/src/detect_human/config/detection_params.yaml

# 파라미터 오버라이드
ros2 run detect_human detect_human_node_with_detection \\
  --ros-args \\
  -p enable_tracking:=true \\
  -p yolo_confidence:=0.6 \\
  -p trajectory_max_length:=50

```

### 3. 캘리브레이션 파라미터 설정

```bash
# calibration_params.json 수정
nano /root/ros2_ws/src/detect_human/data/calibration_params.json

```

## ROS2 인터페이스

### 입력 토픽

| 토픽명                                                   | 메시지 타입                   | 설명                      |
| -------------------------------------------------------- | ----------------------------- | ------------------------- |
| `/my_camera/pylon_ros2_camera_node/image_raw/compressed` | `sensor_msgs/CompressedImage` | 압축된 카메라 이미지      |
| `/pointcloud`                                            | `sensor_msgs/PointCloud2`     | 3D 라이다 포인트 클라우드 |

### 출력 토픽

| 토픽명                      | 메시지 타입                      | 설명                                      |
| --------------------------- | -------------------------------- | ----------------------------------------- |
| `/human_detections`         | `vision_msgs/Detection2DArray`   | 2D 감지 결과 배열                         |
| `/human_segmentation_image` | `sensor_msgs/Image`              | 세그멘테이션 + 스켈레톤 오버레이 이미지   |
| `/human_pointcloud`         | `sensor_msgs/PointCloud2`        | 감지된 사람의 3D 포인트                   |
| `/masked_pointcloud`        | `sensor_msgs/PointCloud2`        | 마스크 색상이 적용된 전체 포인트 클라우드 |
| `/human_bounding_boxes`     | `visualization_msgs/MarkerArray` | RViz2용 3D 바운딩 박스                    |
| `/human_trajectories`       | `visualization_msgs/MarkerArray` | RViz2용 궤적 시각화                       |
| `/yolo_detection_image`     | `sensor_msgs/Image`              | YOLO 감지 결과 이미지                     |
| `/projected_image`          | `sensor_msgs/Image`              | 포인트 클라우드 투영 이미지               |
| `/colored_pointcloud`       | `sensor_msgs/PointCloud2`        | RGB 색상이 매핑된 포인트 클라우드         |

## 파라미터 설정

### 주요 파라미터

### 감지 파라미터

- `enable_human_detection` (bool, default: true): 사람 감지 활성화
- `yolo_model_path` (string, default: "[yolov8m.pt](http://yolov8m.pt/)"): YOLO 모델 경로
- `yolo_confidence` (float, default: 0.5): YOLO 신뢰도 임계값
- `sam2_model_size` (string, default: "base_plus"): SAM2 모델 크기

### 포즈/제스처 파라미터

- `enable_pose_detection` (bool, default: true): 포즈 감지 활성화
- `enable_gesture_detection` (bool, default: true): 제스처 인식 활성화
- `hands_up_y_offset` (int, default: 20): 손 들기 판단 픽셀 오프셋

### 추적 파라미터

- `enable_tracking` (bool, default: true): 다중 객체 추적 활성화
- `max_tracking_distance` (float, default: 1.0): 최대 매칭 거리 (m)
- `max_frames_to_skip` (int, default: 30): 트래커 제거 전 최대 프레임
- `trajectory_max_length` (int, default: 30): 궤적 최대 길이

### 클러스터링 파라미터

- `enable_clustering` (bool, default: true): DBSCAN 클러스터링 활성화
- `cluster_eps` (float, default: 0.01): 클러스터 최대 거리 (m)
- `cluster_min_points` (int, default: 10): 클러스터 최소 포인트 수

### 성능 파라미터

- `skip_frames` (int, default: 0): 프레임 스킵 수
- `max_detections` (int, default: 10): 최대 감지 인원
- `use_gpu` (bool, default: true): GPU 가속 사용

### 설정 파일 예시

```yaml
detect_human_node:
  ros__parameters:
    # 감지 설정
    enable_human_detection: true
    yolo_model_path: "yolov8m.pt" # n, s, m, l, x 사용 가능
    yolo_confidence: 0.5
    sam2_model_size: "base_plus" # tiny, small, base_plus, large

    # 추적 설정
    enable_tracking: true
    trajectory_max_length: 50
    max_frames_to_skip: 30

    # 포즈/제스처
    enable_pose_detection: true
    enable_gesture_detection: true

    # 성능 최적화
    use_gpu: true
    skip_frames: 0
```

## 캘리브레이션 설정

캘리브레이션 파일 (`data/calibration_params.json`) 내용:

```json
{
  "camera_matrix": {
    "fx": 1147.424,
    "fy": 1147.424,
    "cx": 960.0,
    "cy": 600.0,
    "skew": 0.0
  },
  "dist_coeffs": {
    "k1": -0.098,
    "k2": -0.078,
    "k3": 0.0,
    "p1": -0.005,
    "p2": 0.002
  },
  "quaternions": {
    "x": 0.514,
    "y": -0.506,
    "z": 0.496,
    "w": 0.484
  },
  "tvecs": {
    "tx": 0.116,
    "ty": -0.014,
    "tz": -0.02
  }
}
```

## 기술 스택

### AI/ML 프레임워크

- **YOLOv8**: Ultralytics의 최신 객체 감지 모델
- **SAM2**: Meta의 Segment Anything Model 2.1
- **MediaPipe**: Google의 포즈 감지 솔루션
- **PyTorch**: 딥러닝 프레임워크

### 컴퓨터 비전

- **OpenCV**: 이미지 처리 및 시각화
- **Open3D**: 포인트 클라우드 필터링 (옵션)

### 추적 알고리즘

- **Kalman Filter**: 상태 예측 및 업데이트
- **Hungarian Algorithm**: 최적 할당 문제 해결
- **DBSCAN**: 밀도 기반 클러스터링

### ROS2 패키지

- `vision_msgs`: 감지 결과 메시지
- `visualization_msgs`: RViz2 마커
- `cv_bridge`: OpenCV-ROS 브리지
