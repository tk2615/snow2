<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Crossfade Loop)</title>
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
      
      /* リロードボタン */
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
      // 動画2つ
      const snowV1 = document.getElementById('snow-1');
      const snowV2 = document.getElementById('snow-2');
      
      // クロスフェード管理用の変数
      let currentSnowVideo = snowV1; // メインで流れてる方
      let nextSnowVideo = snowV2;    // 次に待機してる方
      const FADE_DURATION = 1.0;     // 1秒かけてクロスフェードさせる

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
      
      let pressTimer;
      let isLongPress = false;
      let isPressing = false;
      let shutterLock = false; 
      const LONG_PRESS_DURATION = 500;

      function log(msg) {
        statusMsg.innerHTML = msg;
        console.log(msg);
      }

      // ==========================================
      // 1. 起動処理
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";
        log("初期化中...");

        try {
          // 動画の準備
          snowV1.loop = false; // ループはJSで制御するから切る
          snowV2.loop = false;
          
          await snowV1.play();
          
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
      // 2. 描画ループ（クロスフェードロジック）
      // ==========================================
      function drawCompositeFrame() {
        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;

        if (cw === 0 || ch === 0) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        // サイズ同期
        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
          bufferCanvas.width = cw;
          bufferCanvas.height = ch;
        }

        // --- バッファ（裏画面）のクリア ---
        bufferCtx.globalCompositeOperation = 'source-over';
        bufferCtx.fillStyle = '#000000';
        bufferCtx.fillRect(0, 0, cw, ch);

        // ===========================================
        // ★クロスフェード計算ロジック
        // ===========================================
        
        const duration = currentSnowVideo.duration;
        const currentTime = currentSnowVideo.currentTime;

        // メタデータ読み込み前などでdurationがNaNのときは何もしない
        if (duration && duration > 0) {
          
          // 1. 残り時間が FADE_DURATION を切ったら、次の動画を再生開始＆重ねて描画
          const timeLeft = duration - currentTime;
          
          if (timeLeft <= FADE_DURATION) {
            // 次の動画が止まってたら再生開始
            if (nextSnowVideo.paused) {
              nextSnowVideo.currentTime = 0;
              nextSnowVideo.play().catch(()=>{});
            }

            // 透明度の計算
            // currentは 1.0 -> 0.0 へ
            // next は 0.0 -> 1.0 へ
            const alphaCurrent = Math.max(0, timeLeft / FADE_DURATION); // フェードアウト
            const alphaNext = 1.0 - alphaCurrent;                       // フェードイン

            // --- 現在の動画を描画（薄くなっていく） ---
            drawSnowToBuffer(currentSnowVideo, cw, ch, alphaCurrent);
            
            // --- 次の動画を描画（濃くなっていく） ---
            drawSnowToBuffer(nextSnowVideo, cw, ch, alphaNext);

          } else {
            // まだフェード期間じゃないなら、現在の動画だけを全力(alpha=1)で描画
            drawSnowToBuffer(currentSnowVideo, cw, ch, 1.0);
            
            // 次の動画は待機（念のため止めておく）
            if (!nextSnowVideo.paused) {
              nextSnowVideo.pause();
              nextSnowVideo.currentTime = 0;
            }
          }

          // 2. 現在の動画が終わったら、役割を交代（スワップ）
          if (currentSnowVideo.ended || timeLeft <= 0) {
            // 交代！
            const temp = currentSnowVideo;
            currentSnowVideo = nextSnowVideo;
            nextSnowVideo = temp;

            // 終わった方は停止して巻き戻し
            nextSnowVideo.pause();
            nextSnowVideo.currentTime = 0;
          }
        }

        // --- メイン合成（カメラの上にバッファをスクリーン合成） ---
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, cw, ch); // カメラ

        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(bufferCanvas, 0, 0); // 雪（クロスフェード済み）

        requestAnimationFrame(drawCompositeFrame);
      }

      // ヘルパー関数：雪動画をバッファに描画する（クロップ＆透明度指定）
      function drawSnowToBuffer(video, cw, ch, alpha) {
        if (alpha <= 0.01) return; // ほぼ見えないなら描かない

        const vw = video.videoWidth;
        const vh = video.videoHeight;
        if (vw === 0 || vh === 0) return;

        // クロップ計算
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

        // 指定された透明度で描画
        bufferCtx.globalAlpha = alpha;
        bufferCtx.drawImage(video, sx, sy, sw, sh, 0, 0, cw, ch);
        bufferCtx.globalAlpha = 1.0; // リセット
      }


      // ==========================================
      // 3. 撮影・録画機能（前回と同じ）
      // ==========================================
      function takePhoto() {
        if (shutterLock) return;
        shutterLock = true;

        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);

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

      // イベント
      const startPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (isPressing) return;
        isPressing = true;
        isLongPress = false;
        pressTimer = setTimeout(() => startRecording(), LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (!isPressing) return;
        isPressing = false;
        clearTimeout(pressTimer);
        if (isRecording) {
          stopRecording();
        } else {
          if (!isLongPress) takePhoto();
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
