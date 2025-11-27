<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Final Ultimate)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000;
        font-family: sans-serif;
        overscroll-behavior: none;
      }

      /* 素材（非表示） */
      .hidden-source {
        position: absolute; top: 0; left: 0;
        width: 10px; height: 10px;
        opacity: 0.01;
        pointer-events: none;
        z-index: -99;
      }

      /* メイン表示＆録画用キャンバス */
      #work-canvas {
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        min-width: 100%; min-height: 100%;
        width: auto; height: auto;
        z-index: 1;
        display: block;
      }

      /* --- UIパーツ --- */
      
      /* リロードボタン（新機能） */
      #reload-btn {
        position: fixed; top: 20px; right: 20px;
        z-index: 500;
        width: 44px; height: 44px;
        background: rgba(255, 255, 255, 0.3);
        border: 1px solid rgba(255, 255, 255, 0.5);
        border-radius: 50%;
        color: white;
        font-size: 20px;
        cursor: pointer;
        display: flex; justify-content: center; align-items: center;
        backdrop-filter: blur(4px);
        -webkit-tap-highlight-color: transparent; 
      }
      #reload-btn:active { background: rgba(255, 255, 255, 0.6); }

      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.9);
        z-index: 999;
        display: flex; justify-content: center; align-items: center; flex-direction: column;
        color: white;
        padding: 20px;
        box-sizing: border-box;
      }
      #start-btn {
        padding: 15px 40px; font-size: 18px;
        background: #ff3b30; color: white;
        border: none; border-radius: 30px;
        cursor: pointer; font-weight: bold;
        margin-bottom: 20px;
      }
      #status-msg { 
        color: #ffffff; font-size: 16px; text-align: center; line-height: 1.5;
        white-space: pre-wrap;
      }

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

    <button id="reload-btn" onclick="location.reload()">↻</button>

    <div id="start-screen">
      <button id="start-btn">カメラを起動する</button>
      <div id="status-msg"></div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline></video>
    
    <video id="snow-1" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline></video>
    <video id="snow-2" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline></video>

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
      // 動画を2つ取得
      const snowV1 = document.getElementById('snow-1');
      const snowV2 = document.getElementById('snow-2');
      let activeSnowVideo = snowV1; // 現在描画に使っている動画

      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      const bufferCanvas = document.createElement('canvas');
      const bufferCtx = bufferCanvas.getContext('2d');

      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const statusMsg = document.getElementById('status-msg');
      
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let selectedMimeType = '';
      
      // 誤操作・連写防止用フラグ
      let pressTimer;
      let isLongPress = false;
      let isPressing = false; // マウス/タッチが押されているか
      let shutterLock = false; // 連写防止ロック
      const LONG_PRESS_DURATION = 500;

      function log(msg) {
        statusMsg.innerHTML = msg;
        console.log(msg);
      }

      // ==========================================
      // 0. シームレスループ管理（二刀流ロジック）
      // ==========================================
      function initSeamlessLoop() {
        // メイン動画の準備
        snowV1.loop = false; // 手動で切り替えるのでloop属性は切る
        snowV2.loop = false;

        // 動画1の監視
        snowV1.addEventListener('timeupdate', () => {
          checkAndSwitch(snowV1, snowV2);
        });
        // 動画2の監視
        snowV2.addEventListener('timeupdate', () => {
          checkAndSwitch(snowV2, snowV1);
        });
      }

      function checkAndSwitch(current, next) {
        // 残り時間が0.3秒を切ったら次の動画を再生開始
        if (current.duration - current.currentTime < 0.3) {
          if (next.paused) {
            next.currentTime = 0;
            next.play().catch(()=>{});
          }
        }
        // 現在の動画が終わったら、描画対象を次に切り替え
        if (current.ended) {
          activeSnowVideo = next; // 描画対象切り替え
          current.pause();
          current.currentTime = 0;
        }
      }

      // ==========================================
      // 1. 起動処理
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";
        log("初期化中...");

        try {
          // 動画ループシステムの準備
          initSeamlessLoop();
          await snowV1.play(); // 最初はV1から
          activeSnowVideo = snowV1;

          // カメラ取得
          let stream = null;
          try {
            stream = await navigator.mediaDevices.getUserMedia({
              video: { facingMode: 'environment', width: { ideal: 1280 }, height: { ideal: 720 } },
              audio: false 
            });
          } catch (err1) {
            log("高画質NG...標準設定で再トライ");
            stream = await navigator.mediaDevices.getUserMedia({
              video: { facingMode: 'environment' },
              audio: false 
            });
          }

          cameraVideo.srcObject = stream;
          
          cameraVideo.onloadedmetadata = () => {
             startScreen.style.display = 'none';
             shutterContainer.style.display = 'block';
             drawCompositeFrame(); 
          };

        } catch (err) {
          console.error(err);
          startBtn.disabled = false;
          startBtn.textContent = "再試行";
          log("エラー: " + err.message);
        }
      });

      // ==========================================
      // 2. 合成ループ
      // ==========================================
      function drawCompositeFrame() {
        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;

        if (cw === 0 || ch === 0) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
          bufferCanvas.width = cw;
          bufferCanvas.height = ch;
        }

        // --- 手順1: 裏キャンバスに雪動画を描く ---
        // activeSnowVideo（現在再生中のやつ）を使う
        const videoToDraw = activeSnowVideo;
        const vw = videoToDraw.videoWidth;
        const vh = videoToDraw.videoHeight;
        
        if (vw > 0 && vh > 0 && !videoToDraw.paused) {
          const videoAspect = vw / vh;
          const canvasAspect = cw / ch;
          let sx, sy, sw, sh;

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

          bufferCtx.globalCompositeOperation = 'source-over';
          bufferCtx.fillStyle = '#000000';
          bufferCtx.fillRect(0, 0, cw, ch);
          bufferCtx.drawImage(videoToDraw, sx, sy, sw, sh, 0, 0, cw, ch);
        }

        // --- 手順2: メイン合成 ---
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, cw, ch);

        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(bufferCanvas, 0, 0);

        requestAnimationFrame(drawCompositeFrame);
      }

      // ==========================================
      // 3. 撮影・録画機能
      // ==========================================
      function takePhoto() {
        if (shutterLock) return; // ロック中なら何もしない
        shutterLock = true; // ロック開始

        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);

        // 1秒後にロック解除（連打防止）
        setTimeout(() => { shutterLock = false; }, 1000);
      }

      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();

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
      // 4. イベント管理（連写防止ロジック強化）
      // ==========================================
      const startPress = (e) => {
        if(e.cancelable) e.preventDefault();
        
        // 既に押されてたら無視
        if (isPressing) return;
        isPressing = true;
        
        isLongPress = false;
        pressTimer = setTimeout(() => startRecording(), LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        
        // 押してないのに離したイベントが来た場合（マウスが外に出た後など）は無視
        if (!isPressing) return;
        isPressing = false;

        clearTimeout(pressTimer);

        if (isRecording) {
          stopRecording();
        } else {
          // 長押し判定じゃなかった場合のみ撮影
          if (!isLongPress) {
            takePhoto();
          }
        }
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
