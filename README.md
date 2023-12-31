<!DOCTYPE html>
<html>

<head>
    <title>YOLO_V5 TFJS</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest"></script>
</head>

<body>
    <div id="info">모델을 불러오고 있습니다. 잠시 기다려 주세요...</div>
    <div id="main">
        <div class="container">
            <div class="canvas-wrapper">
                <canvas id="output"></canvas>
                <video id="video" playsinline style="
                  -webkit-transform: scaleX(-1);
                  transform: scaleX(-1);
                  visibility: hidden;
                  width: auto;
                  height: auto;
                  ">
                </video>
            </div>
        </div>
    </div>
</body>
<script>
    /* ----------------------------------------------------------
    1. 목표 : 도장 불량 데이터 학습 수행 AI 시스템 구축
      - 일관성 있는 도장 품질 검사 기준 적용
      - 도장 관련 지식 부족 인원도 도장 품질 판단
      - 카메라 연동을 통해 신속한 검사 가능 시스템 구축
    
    2. 진행 현황
      - YOLO V5 모델 -> Tensorflow.js 모델로 변환 (완료)
      - Tensorflow.js 모델을 웹 브라우저에서 구현 (완료)
    
    */ 

    // COCO 데이터셋 라벨링
    const coco_names = ['person', 'bicycle', 'car', 'motorcycle', 'airplane', 'bus', 'train', 'truck', 'boat', 'traffic light',
        'fire hydrant', 'stop sign', 'parking meter', 'bench', 'bird', 'cat', 'dog', 'horse', 'sheep', 'cow',
        'elephant', 'bear', 'zebra', 'giraffe', 'backpack', 'umbrella', 'handbag', 'tie', 'suitcase', 'frisbee',
        'skis', 'snowboard', 'sports ball', 'kite', 'baseball bat', 'baseball glove', 'skateboard', 'surfboard',
        'tennis racket', 'bottle', 'wine glass', 'cup', 'fork', 'knife', 'spoon', 'bowl', 'banana', 'apple',
        'sandwich', 'orange', 'broccoli', 'carrot', 'hot dog', 'pizza', 'donut', 'cake', 'chair', 'couch',
        'potted plant', 'bed', 'dining table', 'toilet', 'tv', 'laptop', 'mouse', 'remote', 'keyboard', 'cell phone',
        'microwave', 'oven', 'toaster', 'sink', 'refrigerator', 'book', 'clock', 'vase', 'scissors', 'teddy bear',
        'hair drier', 'toothbrush']

    // 페인트 데이터셋 라벨링
    const names = [
        '워터스포팅',
        '흐름',
        '도막분리',
        '핀홀',
        '균열',
        '부풀음',
        '이물질포함',
        '용접손상',
        '스크래치',
        '도막떨어짐'
    ]

    // 클라이언트 체크
    function isiOS() {
        return /iPhone|iPad|iPod/i.test(navigator.userAgent);
    }

    function isAndroid() {
        return /Android/i.test(navigator.userAgent);
    }

    function isMobile() {
        return isAndroid() || isiOS();
    }

    // WebRTC 카메라 클래스
    class Camera {
        constructor() {
            this.video = document.getElementById('video');
            this.canvas = document.getElementById('output');
            this.ctx = this.canvas.getContext('2d');
        }

        static async setupCamera() {
            if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
                throw new Error('Browser API navigator.mediaDevices.getUserMedia not available');
            }

            // 카메라와 모델 이미지 크기 설정
            const $size = { width: 640, height: 640 };
            const $m_size = { width: 640, height: 640 };
            const videoConfig = {
                'audio': false,
                'video': {
                    facingMode: 'environment',
                    width: isMobile() ? $m_size.width : $size.width,
                    height: isMobile() ? $m_size.height : $size.height,
                }
            };

            const stream = await navigator.mediaDevices.getUserMedia(videoConfig);
            const camera = new Camera();
            camera.video.srcObject = stream;

            await new Promise((resolve) => {
                camera.video.onloadedmetadata = () => {
                    resolve(video);
                };
            });

            camera.video.play();

            const videoWidth = camera.video.videoWidth;
            const videoHeight = camera.video.videoHeight;

            camera.video.width = videoWidth;
            camera.video.height = videoHeight;

            camera.canvas.width = videoWidth;
            camera.canvas.height = videoHeight;
            const canvasContainer = document.querySelector('.canvas-wrapper');
            canvasContainer.style = `width: ${videoWidth}px; height: ${videoHeight}px`;
            return camera;
        }

        drawCtx() {
            this.ctx.drawImage(this.video, 0, 0, this.video.videoWidth, this.video.videoHeight);
        }

        clearCtx() {
            this.ctx.clearRect(0, 0, this.video.videoWidth, this.video.videoHeight);
        }

        drawResult(res) {
            const font = "16px sans-serif";
            this.ctx.font = font;
            this.ctx.textBaseline = "top";

            const [boxes, scores, classes, valid_detections] = res;
            const boxes_data = boxes.dataSync();
            const scores_data = scores.dataSync();
            const classes_data = classes.dataSync();
            const valid_detections_data = valid_detections.dataSync()[0];

            tf.dispose(res);

            let i;
            for (i = 0; i < valid_detections_data; ++i) {
                let [x1, y1, x2, y2] = boxes_data.slice(i * 4, (i + 1) * 4);
                x1 *= this.canvas.width;
                x2 *= this.canvas.width;
                y1 *= this.canvas.height;
                y2 *= this.canvas.height;
                const width = x2 - x1;
                const height = y2 - y1;
                // const klass = coco_names[classes_data[i]];
                const klass = names[classes_data[i]];
                const score = scores_data[i].toFixed(2);

                // Draw the bounding box.
                this.ctx.strokeStyle = "#00FFFF";
                this.ctx.lineWidth = 4;
                this.ctx.strokeRect(x1, y1, width, height);

                // Draw the label background.
                this.ctx.fillStyle = "#00FFFF";
                const textWidth = this.ctx.measureText(klass + ":" + score).width;
                const textHeight = parseInt(font, 10); // base 10
                this.ctx.fillRect(x1, y1, textWidth + 4, textHeight + 4);

            }
            for (i = 0; i < valid_detections_data; ++i) {
                let [x1, y1, ,] = boxes_data.slice(i * 4, (i + 1) * 4);
                x1 *= this.canvas.width;
                y1 *= this.canvas.height;
                // const klass = coco_names[classes_data[i]];
                const klass = names[classes_data[i]];
                const score = scores_data[i].toFixed(2);

                // Draw the text last to ensure it's on top.
                this.ctx.fillStyle = "#000000";
                this.ctx.fillText(klass + ":" + score, x1, y1);
            }
        }
    }


    let detector, camera, stats;
    let startInferenceTime, numInferences = 0;
    let inferenceTimeSum = 0, lastPanelUpdate = 0;
    let rafId;

    // YOLO V5 기본모델을 tfjs로 export한 weight파일 경로
    // const yolov5n_weight = "https://raw.githubusercontent.com/Sein-Oh/MANDLOH/main/Javascript/tfjs_weights/yolov5s_web_model/model.json";

    // YOLO V5 선박 도장데이터셋 9000장으로 학습한 weight
    const yolov5n_weight = "https://raw.githubusercontent.com/Sein-Oh/MANDLOH/main/Javascript/tfjs_weights/yolov5_paint/model.json";

    async function createDetector() {
        return tf.loadGraphModel(yolov5n_weight);
    }

    async function renderResult() {
        if (camera.video.readyState < 2) {
            await new Promise((resolve) => {
                camera.video.onloadeddata = () => {
                    resolve(video);
                };
            });
        }
        let detect_res = null;
        let [modelWidth, modelHeight] = detector.inputs[0].shape.slice(1, 3);
        const input = tf.tidy(() => {
            return tf.image.resizeBilinear(tf.browser.fromPixels(camera.video), [modelWidth, modelHeight])
                .div(255.0).expandDims(0);
        });
        
        if (detector != null) {
            try {
                detect_res = await detector.executeAsync(
                    input,
                );
            } catch (error) {
                detector.dispose();
                detector = null;
                alert(error);
            }
        }
        camera.drawCtx();
        camera.drawResult(detect_res);
        tf.dispose(input);
    }

    async function renderPrediction() {
        await renderResult();
        // rafId = requestAnimationFrame(renderPrediction);
        setTimeout(renderPrediction, 200);
    };

    async function app() {
        camera = await Camera.setupCamera();
        detector = await createDetector();
        document.querySelector("#info").innerHTML = "YOLO V5S 모델 불러오기 완료. 앱을 시작합니다.";
        renderPrediction();
    };
    app();
</script>
</html>
