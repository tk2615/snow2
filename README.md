<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Manual AR Snow</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>

    <style>
      /* 全体のリセット */
      body, html { margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; }

      /* 1階：カメラ映像（背景） */
      #camera-feed {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        object-fit: cover; /* 画面いっぱいに広げる */
        z-index: 1; /* 一番後ろ */
      }

      /* 2階：A-Frame（前景） */
      #ar-scene {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        z-index: 2; /* カメラより手前 */
      }

      /* タップ案内 */
      #overlay-msg {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        z-index: 999;
        color: white;
        background: rgba(0,0,0,0.6);
        padding: 20px;
        border-radius: 8px;
        font-family: sans-serif;
        text-align: center;
        pointer-events: none; /* タップを邪魔しない */
      }
    </style>

    <script>
      // カメラを起動する自作スクリプト
      async function startCamera() {
        const video = document.getElementById('camera-feed');
        try {
          // リアカメラ（environment）を要求
          const stream = await navigator.mediaDevices.getUserMedia({ 
            video: { facingMode: 'environment' }, 
            audio: false 
          });
          video.srcObject = stream;
        } catch (err) {
          console.error("カメラ起動失敗や！:", err);
          alert("カメラが起動できへん！設定で許可してるか確認してな。");
        }
      }

      // ページ読み込み完了時に実行
      window.onload = function() {
        startCamera(); // カメラ起動

        // 動画の自動再生対策
        const snowVideo = document.getElementById('snow-video');
        const msg = document.getElementById('overlay-msg');
        
        // 画面タップで確実に再生させる
        document.body.addEventListener('click', () => {
          snowVideo.play();
          if(msg) msg.style.display = 'none'; // メッセージ消す
        });
        document.body.addEventListener('touchstart', () => {
          snowVideo.play();
          if(msg) msg.style.display = 'none';
        });
      };
    </script>
  </head>

  <body>

    <video id="camera-feed" autoplay muted playsinline></video>

    <div id="overlay-msg">画面をタップして<br>雪(snow.mp4)を降らせるで！</div>

    <a-scene id="ar-scene" embedded renderer="alpha: true;" vr-mode-ui="enabled: false">
      
      <a-assets>
        <video 
          id="snow-video" 
          src="snow.mp4" 
          preload="auto" 
          loop 
          muted 
          playsinline 
          webkit-playsinline
          crossOrigin="anonymous">
        </video>
      </a-assets>

      <a-entity camera position="0 0 0" look-controls="enabled: false">
        
        <a-video 
          src="#snow-video" 
          position="0 0 -2" 
          width="3.2" 
          height="1.8"
          scale="1.5 1.5 1.5"
          material="shader: flat; transparent: true; blending: additive; opacity: 1;">
        </a-video>

      </a-entity>

    </a-scene>

  </body>
</html>
