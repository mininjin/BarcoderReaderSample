<template>
  <div class="fixed w-full h-full bg-black bg-opacity-90 top-0 left-0">
    <div class="w-full h-full relative" ref="cameraDOM">
      <video
        class="w-full h-full absolute object-contain"
        autoplay="true"
      ></video>
    </div>
    <div v-if="cameraStart" class="absolute bottom-5 text-center w-full">
      <button
        class="
          p-2
          bg-gray-200
          text-gray-800
          font-bold
          rounded-lg
          border-2 border-gray-800
          shadow-lg
          mb-2
        "
        @click="stop"
      >
        Stop Camera
      </button>
    </div>
  </div>
</template>
<script lang="ts">
import Quagga from "quagga";
import { defineComponent, onMounted, ref } from "vue";

const SNAP_TIME = 300;

export default defineComponent({
  setup(_, context) {
    const cameraStart = ref<boolean>(false);
    const cameraDOM = ref<HTMLDivElement>();

    let interval: number;
    let code: string | null = null;
    let quaggaCamera = true;

    onMounted(async () => {
      if (cameraDOM.value) {
        // OSで場合分け
        const userAgent = navigator.userAgent;
        if (/iPhone/.test(userAgent)) {
          // iPhoneならQuaggaをLivestreamで起動
          await _start(cameraDOM.value);
        } else {
          // Androidの場合
          // 使用したいカメラを選択
          const devices = await navigator.mediaDevices.enumerateDevices();
          const videoDevices = devices.filter(
            (v) =>
              v.kind == "videoinput" &&
              v.label.indexOf("camera2 0, facing back") >= 0 // カメラの選定
          );
          if (videoDevices.length == 0) {
            // 該当のカメラがないならQuaggaをLivestreamで起動
            await _start(cameraDOM.value);
          } else {
            quaggaCamera = false;
            // 該当のカメラを起動
            const stream = await navigator.mediaDevices.getUserMedia({
              audio: false,
              video: {
                deviceId: videoDevices[0].deviceId,
              },
            });
            cameraStart.value = true;
            // ビデオトラックを取得
            const track = stream.getVideoTracks()[0];
            // 読み取り用canvasの作成
            const canvas = document.createElement("canvas");
            const trackSettings = track.getSettings();
            canvas.width = trackSettings.width || 0;
            canvas.height = trackSettings.height || 0;

            // videoにstreamを入力
            const video = cameraDOM.value.querySelector("video");
            const ctx = canvas.getContext("2d");
            if (video && ctx) {
              video.autoplay = true;
              video.playsInline = true;
              video.srcObject = stream;
              // 0.3秒ごとにcanvasにスナップショットをとり、Quaggaでバーコードを読み取る
              interval = window.setInterval(() => {
                // canvasにスナップショットを作成
                ctx.drawImage(
                  video,
                  0,
                  0,
                  trackSettings.width || 0,
                  trackSettings.height || 0
                );
                // canvasの画像をblobに変換
                canvas.toBlob(
                  (blob) => {
                    if (blob) {
                      // blobをurlに変換
                      const reader = new FileReader();
                      reader.readAsDataURL(blob);
                      reader.onload = () => {
                        // quaggaを起動
                        const config = {
                          decoder: {
                            readers: ["ean_reader"],
                            multiple: false,
                          },
                          locate: true,
                          src: reader.result,
                          singleChannel: false,
                        };
                        Quagga.decodeSingle(config);
                      };
                    }
                  },
                  "image/jpeg",
                  1
                );
              }, SNAP_TIME);
            }
          }
        }

        // quaggaが読み取りに成功した場合の設定
        let count = 0;
        let detected = false;
        Quagga.onDetected((result: any) => {
          if (detected) return;
          // 3連続で同じコードを得られるまで検知
          if (code == result.codeResult.code) {
            count++;
          } else {
            // 前回のコードと違った場合
            count = 0;
            code = result.codeResult.code;
          }
          // 3連続で同じコードを得られた場合はカメラを停止
          if (count >= 3) {
            detected = true;
            // 画像の読み取りを停止
            if (interval) window.clearInterval(interval);
            // カメラを停止
            stop();
          }
        });
      }
    });

    const _start = (camera: HTMLDivElement): Promise<void> => {
      const config = {
        inputStream: {
          type: "LiveStream",
          target: camera,
          constraints: {
            audio: false,
            video: {
              facingMode: "environment",
            },
          },
        },
        decoder: {
          readers: ["ean_reader"],
        },
      };
      return new Promise((resolve, reject) => {
        Quagga.init(
          config,
          // カメラの起動後に実行
          async (err: any) => {
            if (err) {
              reject(err);
            } else {
              cameraStart.value = true;
              // video設定
              const video = camera.querySelector("video");
              if (video) {
                video.width = camera.clientWidth;
                video.height = camera.clientHeight;
                // バーコード読み取りをスタート
                Quagga.start();
                resolve();
              }
            }
          }
        );
      });
    };

    const stop = async () => {
      if (cameraDOM.value) {
        // Quaggaの停止
        Quagga.stop();
        if (quaggaCamera) {
          // QuaggaをLivestreamで起動した場合の初期化
          cameraDOM.value.innerHTML = `<video id="stream" autoplay="true"></video>`;
        } else {
          // カメラをgetUserMediaで起動した場合の初期化
          const video = cameraDOM.value.querySelector("video");
          if (video) {
            const stream = video.srcObject;
            if (stream instanceof MediaStream) {
              const tracks = stream.getVideoTracks();
              for (let i = 0; i < tracks.length; i++) {
                tracks[i].stop();
              }
            }
          }
        }
        cameraStart.value = false;
        context.emit("emitUp", code);
      }
    };
    return {
      cameraStart,
      cameraDOM,
      stop,
    };
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
