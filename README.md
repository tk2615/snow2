<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera Final</title>
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

      /* --- UIパーツ（撮影画面） --- */
      #shutter-container {
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

      /* フラッシュ演出 */
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
      }
      
      /* --- プレビュー画面（撮影確認） --- */
      #preview-modal {
        display: none; /* 最初は隠す */
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: rgba(0,0,0,0.95);
        z-index: 300;
        flex-direction: column;
        align-items: center;
        justify-content: center;
      }

      /* プレビュー画像・動画のスタイル */
      #preview-img, #preview-video {
        max-width: 90%;
        max-height: 70%;
        border-radius: 10px;
        box-shadow: 0 0 20px rgba(255,255,255,0.1);
        /* 長押し保存を有効にするためポインターイベントを許可 */
        pointer-events: auto; 
        -webkit-user-select: none;
        user-select: none;
        /* iOSで長押しメニューが出やすいように */
        -webkit-touch-callout: default; 
      }

      /* ボタン群 */
      .preview-controls {
        margin-top: 20px;
        display: flex;
        gap: 20px;
      }

      .btn {
        padding: 12px 24px;
        border-radius: 30px;
        font-size: 16px;
        font-weight: bold;
        border: none;
        cursor: pointer;
      }
      .btn-cancel { background: #333; color: white; }
      .btn-save { background: white; color: black; }
      
      .guide-text {
        color: #aaa; font-size: 12px; margin-top: 10px;
        text-align: center;
      }

      /* 処理用キャンバス */
      #work-canvas { display: none; }
    </script>
  </head>

  <body>

    <video id="camera-feed" class="responsive-video" autoplay muted playsinline></video>
    <video id="snow-layer" class="responsive-video" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/>
      </svg>
      <div id="shutter-btn"></div>
    </div>

    <div id="flash"></div>

    <div id="preview-modal">
      <img id="preview-img" style="display:none;">
      <video id="preview-video" style="display:none;" controls playsinline loop></video>
      
      <p class="guide-text">長押ししてカメラロールに保存できるで</p>

      <div class="preview-controls">
        <button class="btn btn-cancel" id="btn-retake">撮り直す</button>
        <button class="btn btn-save" id="btn-download">保存する</button>
      </div>
    </div>

    <canvas id="work-canvas"></canvas>

    <script>
      const cameraVideo = document.getElementById('camera-feed');
      const snowVideo = document.getElementById('snow-layer');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      
      // UI要素
      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const previewModal = document.getElementById('preview-modal');
      const previewImg = document.getElementById('preview-img');
      const previewVideo = document.getElementById('preview-video');
      const btnRetake = document.getElementById('btn-retake');
      const btnDownload = document.getElementById('btn-download');

      // ゲージ計算
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      // 録画関連変数
      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let animationFrameId;
      let pressTimer;
      let isLongPress = false;
      let currentBlob = null; // 保存用データ
      let currentFileType = ''; // 'image' or 'video'

      // ==========================================
      // 1. 起動処理
      // ==========================================
      async function initApp() {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment', width: {ideal: 1280}, height: {ideal: 720} },
            audio: false 
          });
          cameraVideo.srcObject = stream;
          snowVideo.play().catch(() => {
            document.body.addEventListener('touchstart', () => snowVideo.play(), {once:true});
          });
        } catch (err) {
          alert("カメラの許可が必要やで！");
        }
      }

      // ==========================================
      // 2. 描画ループ
      // ==========================================
      function drawCompositeFrame() {
        if (canvas.width !== cameraVideo.videoWidth) {
          canvas.width = cameraVideo.videoWidth;
          canvas.height = cameraVideo.videoHeight;
        }

        // カメラを描く
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, canvas.width, canvas.height);

        // 雪をスクリーン合成
        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(snowVideo, 0, 0, canvas.width, canvas.height);

        if (isRecording) {
          animationFrameId = requestAnimationFrame(drawCompositeFrame);
        }
      }

      // ==========================================
      // 3. 静止画撮影
      // ==========================================
      function takePhoto() {
        drawCompositeFrame(); // 1フレーム描画
        
        // フラッシュ
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        // 画像データ作成
        canvas.toBlob((blob) => {
          currentBlob = blob;
          currentFileType = 'image';
          showPreview();
        }, 'image/png');
      }

      // ==========================================
      // 4. 動画撮影（MP4対応努力版）
      // ==========================================
      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();
        recordedChunks = [];

        drawCompositeFrame(); // 録画ループ開始

        const stream = canvas.captureStream(30);
        
        // 【重要】MP4での録画を試みる
        let options = { mimeType: 'video/mp4' }; // まずMP4を要求
        
        if (!MediaRecorder.isTypeSupported('video/mp4')) {
          // ダメならWebM（Androidなど）
          console.log("MP4非対応やからWebMでいくで");
          options = { mimeType: 'video/webm;codecs=vp9' };
          if(!MediaRecorder.isTypeSupported(options.mimeType)) {
             options = { mimeType: 'video/webm' }; // 最終手段
          }
        }

        try {
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          mediaRecorder = new MediaRecorder(stream);
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          // 録画停止後の処理
          // ここで MIMEタイプに関わらずBlobを作る
          // Androidでmp4として保存させるためにtypeを工夫する手もあるが
          // 再生互換性のため、記録された通りの型でBlob化する
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
      // 5. プレビュー画面の表示・保存
      // ==========================================
      function showPreview() {
        // UI切り替え
        shutterContainer.style.display = 'none';
        previewModal.style.display = 'flex';
        
        const url = URL.createObjectURL(currentBlob);

        if (currentFileType === 'image') {
          // 画像表示
          previewImg.src = url;
          previewImg.style.display = 'block';
          previewVideo.style.display = 'none';
          previewVideo.src = "";
        } else {
          // 動画表示
          previewVideo.src = url;
          previewVideo.style.display = 'block';
          previewImg.style.display = 'none';
          previewVideo.play(); // プレビュー再生
        }
      }

      // 「撮り直す」ボタン
      btnRetake.addEventListener('click', () => {
        previewModal.style.display = 'none';
        shutterContainer.style.display = 'block';
        previewVideo.pause();
        // メモリ開放
        URL.revokeObjectURL(previewImg.src);
        URL.revokeObjectURL(previewVideo.src);
      });

      // 「保存する」ボタン（長押ししない人用）
      btnDownload.addEventListener('click', () => {
        if (!currentBlob) return;
        
        const url = URL.createObjectURL(currentBlob);
        const a = document.createElement('a');
        a.style.display = 'none';
        a.href = url;
        
        // ファイル名を決定
        const timestamp = new Date().getTime();
        if (currentFileType === 'image') {
          a.download = `snow_photo_${timestamp}.png`;
        } else {
          // 動画の場合、iPhoneならmp4でOK。
          // Androidでwebmで録画されてても、無理やり.mp4をつけると
          // 最近のプレイヤーは賢いから再生できることが多い。
          a.download = `snow_video_${timestamp}.mp4`; 
        }
        
        document.body.appendChild(a);
        a.click();
        
        // スマホによってはa.click()だけでは動かん場合があるから
        // その場合は「長押ししてな！」のガイドが効いてくる
        setTimeout(() => {
          document.body.removeChild(a);
          window.URL.revokeObjectURL(url);
        }, 100);
      });

      // ==========================================
      // 6. タップイベント管理
      // ==========================================
      const startPress = (e) => {
        if(e.target.closest('#preview-modal')) return; // プレビュー中は反応させない
        e.preventDefault();
        isLongPress = false;
        pressTimer = setTimeout(() => { startRecording(); }, 500);
      };

      const endPress = (e) => {
        if(e.target.closest('#preview-modal')) return;
        e.preventDefault();
        clearTimeout(pressTimer);
        if (isRecording) stopRecording();
        else takePhoto();
      };

      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress);
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('touchend', endPress);

      window.onload = initApp;

    </script>
  </body>
</html>
