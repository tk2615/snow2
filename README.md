<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Final Snow AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>

    <style>
      /* スタイルリセット */
      body, html { margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; background-color: black; }

      /* 1階層目：スマホのカメラ映像（背景） */
      #real-camera {
        position: fixed;
        top: 0; left: 0;
        width: 100%; height: 100%;
        object-fit: cover; /* 画面いっぱいに隙間なく表示 */
        z-index: 1; /* 一番後ろ */
      }

      /* 2階層目：A-Frameの雪（前景） */
      #ar-scene {
        position: fixed;
        top: 0; left: 0;
        width: 100%; height: 100%;
        z-index: 2; /* カメラより手前 */
        pointer-events: none; /* タップ操作を邪魔しないようにする */
      }
    </style>

    <script>
      // 1. スマホのカメラを起動する関数
      async function startRealCamera() {
        const videoElement = document.getElementById('real-camera');
        try {
          // 背面カメラ(environment)を取得
          const stream = await navigator.mediaDevices.getUserMedia({ 
            video: { facingMode: 'environment' }, 
            audio: false 
          });
          videoElement.srcObject = stream;
        } catch (err) {
          console.error("カメラ起動エラー:", err);
          alert("カメラが起動できません。ブラウザの設定でカメラを許可してください。");
        }
      }

      // 2. 雪の動画を強制的に自動再生する関数
      function forcePlaySnow() {
        const snowVideo = document.getElementById('snow-video-asset');
        // とにかく再生を試みる
        snowVideo.play().then(() => {
            console.log("自動再生成功！");
        }).catch(err => {
            console.log("自動再生ブロック！タップ待ちに移行:", err);
            // 万が一ブロックされたら、画面全体どこを触っても再生するように罠を張る
            document.body.addEventListener('touchstart', () => snowVideo.play(), {once:true});
            document.body.addEventListener('click', () => snowVideo.play(), {once:true});
        });
      }

      // ページ読み込み完了時に実行
      window.onload = function() {
        startRealCamera(); // カメラ起動
        forcePlaySnow();   // 雪再生
      };
    </script>
  </head>

  <body>

    <video id="real-camera" autoplay muted playsinline></video>

    <a-scene id="ar-scene" embedded renderer="alpha: true; colorManagement: true;" vr-mode-ui="enabled: false">
      
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
          position="0 0 -0.1"
          width="10" height="10"
          material="shader: flat; transparent: true; blending: additive; depthWrite: false; opacity: 1.0;">
        </a-video>

      </a-entity>

    </a-scene>

  </body>
</html>
