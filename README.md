<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Canvas Preview)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: black;
        font-family: sans-serif;
        overscroll-behavior: none;
      }

      /* * 動画素材は隠す（opacity:0）。
       * display:noneにすると再生が止まるスマホがあるから、
       * 透明にして裏で動かしておくのがコツや。
       */
      .hidden-source {
        position: absolute; top: 0; left: 0;
        width: 1px; height: 1px;
        opacity: 0;
        pointer-events: none;
        z-index: -1;
      }

      /* * メイン表示用のキャンバス 
       * CSSで「画面いっぱいに広げる（object-fit: cover相当）」を実現する
       */
      #work-canvas {
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        /* 短い辺に合わせてアスペクト比を維持しつつ埋める */
        min-width: 100%;
        min-height: 100%;
        width: auto;
        height: auto;
        z-index: 1;
        display: block; /* 最初から表示 */
      }
      
      /* 画面のアスペクト比によっては、これだけだと隙間ができる場合があるから
         JS側でもサイズ調整はしてるけど、CSSで中央寄せ拡大が基本や */

      /* --- UIパーツ --- */
      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.8);
        z-index: 999;
        display: flex; justify-content: center; align-items: center; flex-direction: column;
        color: white;
      }
      #start-btn {
        padding: 15px 40px; font-size: 18px;
        background: #ff3b30; color: white;
        border: none; border-radius: 30px;
        cursor: pointer; font-weight: bold;
      }
      #error-msg { margin-top: 20px; color: #ff3b30; font-size: 14px; text-align: center; }

      #shutter-container {
        position: fixed; bottom: 30px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100;
        cursor: pointer;
        -webkit-tap-highlight-color: transparent; 
        user-select: none;
        display: none;
      }

      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear;
        stroke: #ff3b30; stroke-width: 4; fill: transparent;
      }

      #shutter-btn {
        position: absolute; top: 10px; left: 10px;
        width: 60px; height: 60px;
        background-color: white; border-radius: 50%;
        transition: all 0.2s;
      }
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px; top: 25px; left: 25px;
        border-radius: 4px; background-color: #ff3b30;
      }

      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
      }
    </style>
  </head>

  <body>

    <div id="start-screen">
      <button id="start-btn">カメラを起動する</button>
      <div id="error-msg"></div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline></video>
    <video id="snow-layer" class="hidden-source" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <canvas id="work-canvas"></canvas>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/>
      </svg>
      <div id="shutter-btn"></div>
    </div>

    <div id="flash"></div>

    <script>
      const cameraVideo = document.getElementById('camera-feed');
      const snowVideo = document.getElementById('snow-layer');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d'); // ここで2Dコンテキスト取得
      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const errorMsg = document.getElementById('error-msg');
      
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let selectedMimeType = '';
      let pressTimer;
      let isLongPress = false;
      const LONG_PRESS_DURATION = 500;

      // ==========================================
      // 1. 起動処理
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";

        try {
          // 動画再生
          await snowVideo.play();

          // カメラ取得
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: 'environment', 
              width: { ideal: 1280 },
              height: { ideal: 720 } 
            },
            audio: false 
          });
          cameraVideo.srcObject = stream;
          
          // カメラのメタデータ読み込み完了を待ってからループ開始
          cameraVideo.onloadedmetadata = () => {
             startScreen.style.display = 'none';
             shutterContainer.style.display = 'block';
             drawCompositeFrame(); // ループ開始！
          };

        } catch (err) {
          console.error(err);
          startBtn.disabled = false;
          startBtn.textContent = "再試行";
          errorMsg.textContent = "エラー: " + (err.message || "起動失敗");
        }
      });

      // ==========================================
      // 2. 合成ループ（常に回す）
      // ==========================================
      function drawCompositeFrame() {
        // カメラの準備ができてなければ待つ
        if (cameraVideo.readyState < 2) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;

        // キャンバスサイズ更新
        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
        }

        // 1. カメラ描画
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, cw, ch);

        // 2. 雪の合成（スクリーン）
        // CSSではなくCanvasAPIで行うため、確実に合成される
        ctx.globalCompositeOperation = 'screen';

        const vw = snowVideo.videoWidth;
        const vh = snowVideo.videoHeight;

        if (vw > 0 && vh > 0) {
          // アスペクト比計算とクロップ
          const videoAspect = vw / vh;
          const canvasAspect = cw / ch;
          let sx, sy, sw, sh;

          // object-fit: cover のロジック
          if (canvasAspect > videoAspect) {
            sw = vw;
            sh = vw / canvasAspect;
            sx = 0;
            sy = (vh - sh) / 2;
          } else {
            sh = vh;
            sw = vh * canvasAspect;
            sx = (vw - sw) / 2;
            sy = 0;
          }
          // 雪を描画
          ctx.drawImage(snowVideo, sx, sy, sw, sh, 0, 0, cw, ch);
        }

        // ループ継続（録画中じゃなくても回す）
        requestAnimationFrame(drawCompositeFrame);
      }

      // ==========================================
      // 3. 撮影・録画機能
      // ==========================================
      function takePhoto() {
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);
      }

      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();
        // ループは既に回ってるのでここでは何もしなくてOK

        const stream = canvas.captureStream(30);
        
        const mimeTypes = [
          'video/mp4;codecs=avc1',
          'video/mp4',
          'video/webm;codecs=h264',
          'video/webm'
        ];
        selectedMimeType = mimeTypes.find(type => MediaRecorder.isTypeSupported(type)) || '';

        try {
          const options = selectedMimeType ? { mimeType: selectedMimeType } : undefined;
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          mediaRecorder = new MediaRecorder(stream);
          selectedMimeType = 'video/webm';
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          const blob = new Blob(recordedChunks, { type: selectedMimeType || 'video/webm' });
          const url = URL.createObjectURL(blob);
          let ext = (selectedMimeType && selectedMimeType.includes('mp4')) ? 'mp4' : 'webm';
          downloadFile(url, `snow_video_${Date.now()}.${ext}`);
          recordedChunks = [];
        };

        mediaRecorder.start();
        updateGauge();
      }

      function stopRecording() {
        isRecording = false;
        shutterContainer.classList.remove('recording');
        // ループは止めない
        if (mediaRecorder && mediaRecorder.state !== 'inactive') mediaRecorder.stop();
        progressCircle.style.strokeDashoffset = circumference;
      }

      function updateGauge() {
        if (!isRecording) return;
        const MAX_RECORD_TIME = 10000;
        const elapsed = Date.now() - recordingStartTime;
        const progress = Math.min(elapsed / MAX_RECORD_TIME, 1);
        progressCircle.style.strokeDashoffset = circumference - (progress * circumference);

        if (progress < 1) requestAnimationFrame(updateGauge);
        else stopRecording();
      }

      // ==========================================
      // 4. イベント管理
      // ==========================================
      const startPress = (e) => {
        if(e.cancelable) e.preventDefault();
        isLongPress = false;
        pressTimer = setTimeout(() => startRecording(), LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        clearTimeout(pressTimer);
        if (isRecording) stopRecording();
        else takePhoto();
      };

      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress, {passive: false});
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('mouseleave', endPress);
      shutterContainer.addEventListener('touchend', endPress, {passive: false});

      function downloadFile(url, filename) {
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
      }
    </script>
  </body>
</html>
