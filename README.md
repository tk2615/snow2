<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Fixed Snow AR Camera</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: black;
        font-family: sans-serif;
      }

      /* 動画表示エリア */
      .responsive-video {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        object-fit: cover;
      }
      #camera-feed { z-index: 1; }
      #snow-layer { z-index: 2; mix-blend-mode: screen; pointer-events: none; }

      /* --- スタート画面（エラー対策の切り札） --- */
      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: #000;
        z-index: 9999;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        color: white;
      }
      #start-btn {
        padding: 20px 40px;
        font-size: 24px;
        font-weight: bold;
        background: #ff3b30;
        color: white;
        border: none;
        border-radius: 50px;
        cursor: pointer;
        box-shadow: 0 0 20px rgba(255, 59, 48, 0.5);
      }
      #debug-log {
        margin-top: 20px;
        font-size: 12px;
        color: #aaa;
        width: 80%;
        text-align: center;
        white-space: pre-wrap;
      }

      /* --- UIパーツ --- */
      #shutter-container {
        display: none; /* 最初は隠す */
        position: fixed; bottom: 40px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100;
        cursor: pointer;
        -webkit-tap-highlight-color: transparent; 
        user-select: none;
      }

      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear;
        stroke: #ff3b30;
        stroke-width: 5;
        fill: transparent;
      }

      #shutter-btn {
        position: absolute; top: 10px; left: 10px;
        width: 60px; height: 60px;
        background-color: white;
        border-radius: 50%;
        transition: all 0.2s;
        box-shadow: 0 0 10px rgba(0,0,0,0.3);
      }
      
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px;
        top: 25px; left: 25px;
        border-radius: 4px;
        background-color: #ff3b30;
      }

      /* プレビュー画面 */
      #preview-modal {
        display: none;
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: rgba(0,0,0,0.95);
        z-index: 300;
        flex-direction: column;
        align-items: center;
        justify-content: center;
      }
      #preview-img, #preview-video {
        max-width: 90%; max-height: 70%;
        border-radius: 10px;
        pointer-events: auto; 
        -webkit-user-select: none; user-select: none;
        -webkit-touch-callout: default; 
      }
      .preview-controls { margin-top: 20px; display: flex; gap: 20px; }
      .btn { padding: 12px 24px; border-radius: 30px; font-size: 16px; font-weight: bold; border: none; cursor: pointer; }
      .btn-cancel { background: #333; color: white; }
      .btn-save { background: white; color: black; }
      .guide-text { color: #aaa; font-size: 12px; margin-top: 10px; text-align: center; }

      #flash { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: white; opacity: 0; pointer-events: none; z-index: 200; transition: opacity 0.2s; }
      #work-canvas { display: none; }
    </script>
  </head>

  <body>

    <div id="start-screen">
      <button id="start-btn">カメラ起動</button>
      <div id="debug-log">待機中...</div>
    </div>

    <video id="camera-feed" class="responsive-video" autoplay muted playsinline></video>
    <video id="snow-layer" class="responsive-video" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <div id="shutter-container">
      <svg class="progress-ring"><circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/></svg>
      <div id="shutter-btn"></div>
    </div>
    <div id="flash"></div>

    <div id="preview-modal">
      <img id="preview-img" style="display:none;">
      <video id="preview-video" style="display:none;" controls playsinline loop></video>
      <p class="guide-text">長押しで保存 / または下のボタン</p>
      <div class="preview-controls">
        <button class="btn btn-cancel" id="btn-retake">撮り直す</button>
        <button class="btn btn-save" id="btn-download">保存する</button>
      </div>
    </div>

    <canvas id="work-canvas"></canvas>

    <script>
      // ログ表示用関数
      function log(msg) {
        const logBox = document.getElementById('debug-log');
        logBox.innerText += "\n" + msg;
        console.log(msg);
      }

      // エラーハンドリング
      window.onerror = function(msg, url, line) {
        log("エラー発生: " + msg);
        return false;
      };

      const cameraVideo = document.getElementById('camera-feed');
      const snowVideo = document.getElementById('snow-layer');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      
      // UI要素
      const previewModal = document.getElementById('preview-modal');
      const previewImg = document.getElementById('preview-img');
      const previewVideo = document.getElementById('preview-video');
      const btnRetake = document.getElementById('btn-retake');
      const btnDownload = document.getElementById('btn-download');

      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let animationFrameId;
      let pressTimer;
      let isLongPress = false;
      let currentBlob = null;
      let currentFileType = '';

      // ==========================================
      // 1. スタートボタンでの起動処理
      // ==========================================
      startBtn.addEventListener('click', async () => {
        log("起動開始...");
        startBtn.disabled = true;
        startBtn.innerText = "起動中...";

        try {
          // カメラ起動
          log("カメラ権限を要求中...");
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment', width: {ideal: 1280}, height: {ideal: 720} },
            audio: false 
          });
          log("カメラ取得成功！");
          cameraVideo.srcObject = stream;
          
          // カメラのロード待ち
          await new Promise(r => cameraVideo.onloadedmetadata = r);
          cameraVideo.play();

          // 雪動画の再生
          log("雪動画を再生...");
          await snowVideo.play();
          
          // 準備完了
          log("全システム正常。");
          startScreen.style.display = 'none'; // スタート画面消去
          shutterContainer.style.display = 'block'; // シャッター表示

        } catch (err) {
          log("【致命的エラー】: " + err.name + " - " + err.message);
          startBtn.innerText = "再試行";
          startBtn.disabled = false;
          alert("起動に失敗しました。\nログを確認してください。\n" + err.message);
        }
      });

      // ==========================================
      // 2. 描画ループ
      // ==========================================
      function drawCompositeFrame() {
        if (canvas.width !== cameraVideo.videoWidth && cameraVideo.videoWidth > 0) {
          canvas.width = cameraVideo.videoWidth;
          canvas.height = cameraVideo.videoHeight;
        }

        if(canvas.width > 0) {
            ctx.globalCompositeOperation = 'source-over';
            ctx.drawImage(cameraVideo, 0, 0, canvas.width, canvas.height);
            ctx.globalCompositeOperation = 'screen';
            ctx.drawImage(snowVideo, 0, 0, canvas.width, canvas.height);
        }

        if (isRecording) {
          animationFrameId = requestAnimationFrame(drawCompositeFrame);
        }
      }

      // ==========================================
      // 3. 撮影機能
      // ==========================================
      function takePhoto() {
        drawCompositeFrame(); 
        
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        canvas.toBlob((blob) => {
          if(!blob) { log("画像生成失敗"); return; }
          currentBlob = blob;
          currentFileType = 'image';
          showPreview();
        }, 'image/png');
      }

      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();
        recordedChunks = [];
        drawCompositeFrame(); 

        // Safari対応のストリーム取得
        const stream = canvas.captureStream ? canvas.captureStream(30) : canvas.webkitCaptureStream(30);
        
        let options = { mimeType: 'video/mp4' };
        if (!MediaRecorder.isTypeSupported('video/mp4')) {
          options = { mimeType: 'video/webm' };
          if(!MediaRecorder.isTypeSupported(options.mimeType)) {
             options = undefined; // ブラウザにお任せ
          }
        }

        try {
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          log("MediaRecorderエラー: " + e.message);
          mediaRecorder = new MediaRecorder(stream);
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          const mimeType = mediaRecorder.mimeType || 'video/webm';
          currentBlob = new Blob(recordedChunks, { type: mimeType });
          currentFileType = 'video';
          showPreview();
        };

        mediaRecorder.start();
        updateGauge();
      }

      function stopRecording() {
        isRecording = false;
        shutterContainer.classList.remove('recording');
        cancelAnimationFrame(animationFrameId);
        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
          mediaRecorder.stop();
        }
        progressCircle.style.strokeDashoffset = circumference;
      }

      function updateGauge() {
        if (!isRecording) return;
        const MAX_TIME = 10000;
        const elapsed = Date.now() - recordingStartTime;
        const progress = Math.min(elapsed / MAX_TIME, 1);
        const offset = circumference - (progress * circumference);
        progressCircle.style.strokeDashoffset = offset;

        if (progress < 1) requestAnimationFrame(updateGauge);
        else stopRecording();
      }

      // ==========================================
      // 4. プレビュー & 保存
      // ==========================================
      function showPreview() {
        shutterContainer.style.display = 'none';
        previewModal.style.display = 'flex';
        
        const url = URL.createObjectURL(currentBlob);

        if (currentFileType === 'image') {
          previewImg.src = url;
          previewImg.style.display = 'block';
          previewVideo.style.display = 'none';
          previewVideo.src = "";
        } else {
          previewVideo.src = url;
          previewVideo.style.display = 'block';
          previewImg.style.display = 'none';
          previewVideo.play();
        }
      }

      btnRetake.addEventListener('click', () => {
        previewModal.style.display = 'none';
        shutterContainer.style.display = 'block';
        previewVideo.pause();
        URL.revokeObjectURL(previewImg.src);
        URL.revokeObjectURL(previewVideo.src);
      });

      btnDownload.addEventListener('click', () => {
        if (!currentBlob) return;
        const url = URL.createObjectURL(currentBlob);
        const a = document.createElement('a');
        a.style.display = 'none';
        a.href = url;
        
        const timestamp = new Date().getTime();
        if (currentFileType === 'image') {
          a.download = `snow_photo_${timestamp}.png`;
        } else {
          a.download = `snow_video_${timestamp}.mp4`; 
        }
        
        document.body.appendChild(a);
        a.click();
        setTimeout(() => {
          document.body.removeChild(a);
          window.URL.revokeObjectURL(url);
        }, 100);
      });

      // ==========================================
      // 5. 操作イベント
      // ==========================================
      const startPress = (e) => {
        if(e.target.closest('#preview-modal') || e.target.closest('#start-screen')) return;
        e.preventDefault();
        isLongPress = false;
        pressTimer = setTimeout(() => { startRecording(); }, 500);
      };

      const endPress = (e) => {
        if(e.target.closest('#preview-modal') || e.target.closest('#start-screen')) return;
        e.preventDefault();
        clearTimeout(pressTimer);
        if (isRecording) stopRecording();
        else takePhoto();
      };

      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress);
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('touchend', endPress);

    </script>
  </body>
</html>
