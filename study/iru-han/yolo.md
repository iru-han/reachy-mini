```
import time
from pathlib import Path

import cv2
import mediapipe as mp
from mediapipe.tasks.python import BaseOptions
from mediapipe.tasks.python.vision import (
    FaceLandmarker,
    FaceLandmarkerOptions,
    HandLandmarker,
    HandLandmarkerOptions,
    RunningMode,
)
from ultralytics import YOLO

MODEL_DIR = Path.home() / "mediapipe_models"

# ---------------------------------------------------------------------------
# 1) YOLO: 사람 및 방안 사물 인식 (COCO 사전 학습 모델)
# ---------------------------------------------------------------------------
yolo_model = YOLO("yolo11n.pt")

# ---------------------------------------------------------------------------
# 2) MediaPipe FaceLandmarker: 얼굴 랜드마크 + 표정(blendshapes)
# ---------------------------------------------------------------------------
face_landmarker = FaceLandmarker.create_from_options(
    FaceLandmarkerOptions(
        base_options=BaseOptions(model_asset_path=str(MODEL_DIR / "face_landmarker.task")),
        running_mode=RunningMode.VIDEO,
        num_faces=2,
        output_face_blendshapes=True,
        output_facial_transformation_matrixes=False,
    )
)

# ---------------------------------------------------------------------------
# 3) MediaPipe HandLandmarker: 손 랜드마크
# ---------------------------------------------------------------------------
hand_landmarker = HandLandmarker.create_from_options(
    HandLandmarkerOptions(
        base_options=BaseOptions(model_asset_path=str(MODEL_DIR / "hand_landmarker.task")),
        running_mode=RunningMode.VIDEO,
        num_hands=2,
    )
)

def draw_hand_landmarks(frame, hand_result):
    h, w = frame.shape[:2]
    for hand_landmarks in hand_result.hand_landmarks:
        points = [(int(lm.x * w), int(lm.y * h)) for lm in hand_landmarks]
        for x, y in points:
            cv2.circle(frame, (x, y), 3, (0, 255, 0), -1)


def draw_face_landmarks(frame, face_result):
    h, w = frame.shape[:2]
    for face_landmarks in face_result.face_landmarks:
        for lm in face_landmarks:
            x, y = int(lm.x * w), int(lm.y * h)
            cv2.circle(frame, (x, y), 1, (255, 200, 0), -1)


EXPRESSION_THRESHOLD = 0.5


def classify_expression(blendshapes, threshold=EXPRESSION_THRESHOLD):
    """ARKit blendshape 조합을 웃음/화남/슬픔/놀람 등 감정 라벨로 매핑."""
    bs = {c.category_name: c.score for c in blendshapes}

    smile = (bs.get("mouthSmileLeft", 0) + bs.get("mouthSmileRight", 0)) / 2  # 입꼬리 올라감
    frown = (bs.get("mouthFrownLeft", 0) + bs.get("mouthFrownRight", 0)) / 2  # 입꼬리 내려감
    brow_down = (bs.get("browDownLeft", 0) + bs.get("browDownRight", 0)) / 2  # 눈썹 사이 찡그림/눈썹 내려감
    nose_sneer = (bs.get("noseSneerLeft", 0) + bs.get("noseSneerRight", 0)) / 2  # 코 찡그림
    cheek_squint = (bs.get("cheekSquintLeft", 0) + bs.get("cheekSquintRight", 0)) / 2  # 팔자주름
    jaw_open = bs.get("jawOpen", 0)  # 입 벌어짐
    eye_wide = (bs.get("eyeWideLeft", 0) + bs.get("eyeWideRight", 0)) / 2  # 눈 커짐
    brow_up = (
        bs.get("browInnerUp", 0) + bs.get("browOuterUpLeft", 0) + bs.get("browOuterUpRight", 0)
    ) / 3  # 눈썹 올라감

    scores = {
        "웃음": smile,
        "화남": (nose_sneer + brow_down) / 2,
        "슬픔": (frown + cheek_squint + brow_down) / 3,
        "놀람": (eye_wide + jaw_open + brow_up) / 3,
    }

    label, score = max(scores.items(), key=lambda kv: kv[1])
    if score < threshold:
        return "무표정", score
    return label, score


def top_expression(face_result, face_idx=0):
    if not face_result.face_blendshapes:
        return None
    return classify_expression(face_result.face_blendshapes[face_idx])


def main():
    cap = cv2.VideoCapture(0)
    if not cap.isOpened():
        raise RuntimeError("웹캠을 열 수 없습니다. 카메라 연결 및 인덱스를 확인하세요.")

    start_time = time.time()

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        timestamp_ms = int((time.time() - start_time) * 1000)
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)

        # --- YOLO: 사람/사물 탐지 ---
        yolo_results = yolo_model(frame, verbose=False)
        annotated = yolo_results[0].plot()

        # --- MediaPipe: 얼굴 표정 ---
        face_result = face_landmarker.detect_for_video(mp_image, timestamp_ms)
        draw_face_landmarks(annotated, face_result)
        expr = top_expression(face_result)
        if expr:
            name, score = expr
            cv2.putText(
                annotated, f"expression: {name} ({score:.2f})", (10, 30),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 200, 0), 2,
            )

        # --- MediaPipe: 손 랜드마크 ---
        hand_result = hand_landmarker.detect_for_video(mp_image, timestamp_ms)
        draw_hand_landmarks(annotated, hand_result)
        if hand_result.handedness:
            names = ", ".join(h[0].category_name for h in hand_result.handedness)
            cv2.putText(
                annotated, f"hands: {names}", (10, 60),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2,
            )

        cv2.imshow("YOLO + MediaPipe (Face/Hand)", annotated)

        if cv2.waitKey(1) & 0xFF == ord("q"):
            break

    cap.release()
    cv2.destroyAllWindows()
    face_landmarker.close()
    hand_landmarker.close()


if __name__ == "__main__":
    main()

```
