<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>CSS Screen Blend AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">

    <style>
      /* 全体を黒くして余白を消す */
      body, html {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        background-color: #000;
        overflow: hidden;
      }

      /* レイヤー共通設定 */
      .full-screen-video {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        object-fit: cover; /* 画面いっぱいに広げる */
      }

      /* 1階：カメラ映像（一番下） */
      #camera-video {
        z-index: 1;
      }

      /* 2階：雪の動画（上） */
      #snow-video {
        z-index: 2;
        /* 【ここが魔法の呪文や！】
           mix-blend-mode: screen; 
           → これぞ「スクリーン合成」！黒を透明にして、明るい色だけ残す。
           もし薄すぎたら 'lighten' や 'plus-lighter' も試せるで。
        */
        mix-blend-mode: screen;
        
        /* タップ操作を透過させる（下のボタン等が押せるように）
           今回は関係ないけど、おまじないとして入れとくわ 
        */
        pointer-events: none; 
      }

      /* スタートボタン（自動再生されなかった時用） */
      #start-btn {
        position: absolute;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        z-index: 999;
        padding: 15px 30px;
        font-size: 18px;
        background: rgba(255, 255, 255, 0.2);
        color: white;
        border: 1px solid white;
        border-radius: 30px;
        display: none; /* 最初は隠しておく */
      }
    </style>
  </head>

  <body>

    <video id="camera-video" autoplay muted playsinline></video>

    <video id="snow-video" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <button id="start-btn">TAP TO START SNOW</button>

    <script>
      // カメラ起動
      async function startCamera() {
        const video = document.getElementById('camera-video');
        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment' },
            audio: false
          });
          video.srcObject = stream;
        } catch (err) {
          alert("カメラの許可が必要やで！設定見てみてな。");
        }
      }

      // 雪動画の再生管理
      function initSnowVideo() {
        const snow = document.getElementById('snow-video');
        const btn = document.getElementById('start-btn');

        // まずは強制再生を試みる
        const playPromise = snow.play();

        if (playPromise !== undefined) {
          playPromise.then(() => {
            console.log("自動再生成功！");
          }).catch(error => {
            console.log("自動再生ブロックされたわ。ボタン出すで。");
            btn.style.display = 'block'; // ボタン表示
          });
        }

        // ボタンが押されたら再生
        btn.addEventListener('click', () => {
          snow.play();
          btn.style.display = 'none';
        });
        
        // 画面タッチでも再生試行（念のため）
        document.body.addEventListener('touchstart', () => {
             if(snow.paused) { snow.play(); btn.style.display = 'none'; }
        }, {once:true});
      }

      window.onload = function() {
        startCamera();
        initSnowVideo();
      };
    </script>
  </body>
</html>
