<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>True Final Snow AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>

    <style>
      /* スタイルリセット：背景を黒にして余白をなくす */
      body, html { margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; background-color: #000; }

      /* 1階層目：スマホのカメラ映像（背景） */
      /* これが一番後ろにあって、画面全体に広がる */
      #real-camera {
        position: fixed;
        top: 0; left: 0;
        width: 100%; height: 100%;
        object-fit: cover;
        z-index: 1;
      }

      /* 2階層目：A-Frameの雪（前景） */
      /* カメラの上に透明なレイヤーとして乗っかる */
      #ar-scene {
        position: fixed;
        top: 0; left: 0;
        width: 100%; height: 100%;
        z-index: 2;
        pointer-events: none; /* 操作を邪魔しない */
      }
    </style>

    <script>
      // 1. カメラ起動スクリプト
      async function startRealCamera() {
        const videoElement = document.getElementById('real-camera');
        try {
          // 背面カメラを要求
          const stream = await navigator.mediaDevices.getUserMedia({ 
            video: { facingMode: 'environment' }, 
            audio: false 
          });
          videoElement.srcObject = stream;
        } catch (err) {
          console.error("カメラダメやったわ:", err);
          alert("カメラが起動できへん。設定を確認してな！");
        }
      }

      // 2. 強制自動再生スクリプト
      function forcePlaySnow() {
        const snowVideo = document.getElementById('snow-video-asset');
        // まずは普通に再生を試みる
        snowVideo.play().then(() => {
            console.log("自動再生いけたで！");
        }).catch(err => {
            console.log("自動再生ブロックされた！タップ待ちの罠を張るで。");
            // ブロックされたら、画面のどこを触っても再生するように仕向ける
            document.body.addEventListener('touchstart', () => snowVideo.play(), {once:true});
            document.body.addEventListener('click', () => snowVideo.play(), {once:true});
        });
      }

      // 読み込み完了したら実行
      window.onload = function() {
        startRealCamera();
        forcePlaySnow();
      };
    </script>
  </head>

  <body>

    <video id="real-camera" autoplay muted playsinline></video>

    <a-scene id="ar-scene" embedded renderer="alpha: true;" vr-mode-ui="enabled: false">
      
      <a-assets>
        <video 
          id="snow-video-asset" 
          src="snow.mp4" 
          preload="auto" 
          autoplay loop muted playsinline webkit-playsinline
          crossOrigin="anonymous">
        </video>
      </a-assets>

      <a-entity camera position="0 0 0" look-controls="enabled: false">
        
        <a-video 
          src="#snow-video-asset" 
          position="0 0 -4"
          width="8" 
          height="4.5"
          material="shader: flat; transparent: true; blending: additive; opacity: 1.0;">
        </a-video>

      </a-entity>

    </a-scene>

  </body>
</html>
