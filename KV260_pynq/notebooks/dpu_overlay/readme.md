# DPU Overlay on KV260

## 1. Pynq dpu 예제 `dpu_yolov3.ipynb`에 camera를 연결하자

### 문제점

1. jupyter notebook 상에서 cv2.imshow()를 사용할 수 없다.
    - `IPython.display` 로 해결

    ```python
        from IPython.display import display, Image, clear_output
            
        # # 이미지 출력
        # cv2.imshow('cam',frame)

        # JPEG 포맷으로 인코딩 (압축률 조절로 레이턴시 감소 가능)
        _, jpeg = cv2.imencode('.jpg', frame, [int(cv2.IMWRITE_JPEG_QUALITY), 70])
        # Jupyter 화면 업데이트
        clear_output(wait=True)
        display(Image(data=jpeg.tobytes()))
    ```

2. pynq jupyter 환경에서만 사용 가능
    - `root` 로 ssh 접속
    - `root` 접속 허용 방법
        * `/etc/ssh/sshd_config`의 ₩PermitRootLogin` 항목을 `yes`로 수정

## 2. `dpu_yolov3_cam.ipynb`을 `.py` module로 만들어 실행시켜 보자

- `root`로 접속시 X11 forwarding 이 가능하다.
- `.py` module로 download 후 코드를 정리하고 진행해 보자
