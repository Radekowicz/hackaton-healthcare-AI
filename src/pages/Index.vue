<template>
  <q-page class="flex flex-center column">
    <div v-show="streaming" class="row">
      <video
        autoplay
        ref="camInput"
        id="camInput"
        height="480"
        width="640"
      ></video>
      <canvas ref="canva2" id="canvasOutput"></canvas>
    </div>
    <q-btn
      v-if="!streaming"
      color="primary"
      label="START"
      @click="streamVideo"
    />
    <q-btn v-else color="primary" label="STOP" @click="streaming = false" />
  </q-page>
</template>

<script>
import axios from "axios";

export default {
  name: "Index",
  data() {
    return {
      frameIndex: 0,
      peopleFound: [],
      streaming: false,
    };
  },
  methods: {
    stopVideo: function (videoEl) {
      const stream = videoEl.srcObject;
      const tracks = stream.getTracks();
      tracks.forEach(function (track) {
        track.stop();
      });
    },
    sendNewData: async function (peopleAmount, placeId) {
      const res = await axios({
        method: "POST",
        url: "http://localhost:3000/api/v1/data",
        data: {
          placeId,
          peopleAmount,
        },
      });
      if ((res.data.status = "success")) {
        console.log("Sent successfully!");
      }
    },
    restreamVideo: function (classifier) {
      this.streaming = true;
      let video = this.$refs.camInput;
      // console.log('video2: ', video);
      // console.log('video: ', this.video);
      navigator.mediaDevices
        .getUserMedia({ video: true, audio: false })
        .then(function (stream) {
          video.srcObject = stream;
          video.play();
        })
        .catch((err) => console.log("Error"));

      let src = new cv.Mat(video.height, video.width, cv.CV_8UC4);
      let dst = new cv.Mat(video.height, video.width, cv.CV_8UC4);
      let gray = new cv.Mat();
      let cap = new cv.VideoCapture(video);

      let lights = new cv.RectVector();

      const FPS = 30;

      const processVideo = () => {
        let begin = Date.now();
        cap.read(src); //video frame will be saved to src matrix
        src.copyTo(dst);
        cv.cvtColor(dst, gray, cv.COLOR_RGBA2GRAY, 0); //image processing
        try {
          classifier.detectMultiScale(gray, lights, 1.1, 3, 0);
        } catch (err) {
          // console.log(err);
        }
        for (let i = 0; i < lights.size(); ++i) {
          let light = lights.get(i);
          let point1 = new cv.Point(light.x, light.y);
          let point2 = new cv.Point(
            light.x + light.width,
            light.y + light.height
          );
          cv.rectangle(dst, point1, point2, [255, 0, 0, 255]);
        }
        cv.imshow("canvasOutput", dst);
        this.frameIndex++;
        console.log(this.frameIndex);
        this.peopleFound.push(lights.size());
        let delay = 1000 / FPS - (Date.now() - begin);
        if (this.frameIndex >= 40) {
          const highestAmount = Math.max(...this.peopleFound);
          this.sendNewData(highestAmount, 1);
          console.log("HIGHEST: ", highestAmount);
          this.peopleFound.length = 0;
          this.frameIndex = 0;
          this.stopVideo(video);
          return setTimeout(() => {
            this.restreamVideo(classifier);
          }, 300000);
        }
        if (this.streaming) setTimeout(processVideo, delay);
        else {
          this.stopVideo(video);
        }
      };
      // schedule first one.
      setTimeout(processVideo, 0);
    },
    streamVideo: function () {
      this.streaming = true;
      let video = this.$refs.camInput;
      // console.log('video2: ', video);
      // console.log('video: ', this.video);
      navigator.mediaDevices
        .getUserMedia({ video: true, audio: false })
        .then(function (stream) {
          video.srcObject = stream;
          video.play();
        })
        .catch((err) => console.log("Error"));

      let src = new cv.Mat(video.height, video.width, cv.CV_8UC4);
      let dst = new cv.Mat(video.height, video.width, cv.CV_8UC4);
      let gray = new cv.Mat();
      let cap = new cv.VideoCapture(video);

      let lights = new cv.RectVector();
      let classifier = new cv.CascadeClassifier();

      let lightsCascadeFile = `haarcascade_upperbody.xml`;
      this.createFileFromUrl(lightsCascadeFile, lightsCascadeFile, () => {
        classifier.load(lightsCascadeFile);
      });
      // classifier.load(lightsCascadeFile);

      const FPS = 30;

      const processVideo = () => {
        let begin = Date.now();
        cap.read(src); //video frame will be saved to src matrix
        src.copyTo(dst);
        cv.cvtColor(dst, gray, cv.COLOR_RGBA2GRAY, 0); //image processing
        try {
          classifier.detectMultiScale(gray, lights, 1.1, 3, 0);
        } catch (err) {
          // console.log(err);
        }
        for (let i = 0; i < lights.size(); ++i) {
          let light = lights.get(i);
          let point1 = new cv.Point(light.x, light.y);
          let point2 = new cv.Point(
            light.x + light.width,
            light.y + light.height
          );
          cv.rectangle(dst, point1, point2, [255, 0, 0, 255]);
        }
        cv.imshow("canvasOutput", dst);
        this.frameIndex++;
        console.log(this.frameIndex);
        this.peopleFound.push(lights.size());
        let delay = 1000 / FPS - (Date.now() - begin);
        if (this.frameIndex >= 40) {
          const highestAmount = Math.max(...this.peopleFound);
          this.sendNewData(highestAmount, 1);
          console.log("HIGHEST: ", highestAmount);
          this.peopleFound.length = 0;
          this.frameIndex = 0;
          this.stopVideo(video);
          return setTimeout(() => {
            this.restreamVideo(classifier);
          }, 300000);
        }
        if (this.streaming) setTimeout(processVideo, delay);
        else {
          this.stopVideo(video);
        }
      };
      // schedule first one.
      setTimeout(processVideo, 0);
    },
    createFileFromUrl: function (path, url, callback) {
      let request = new XMLHttpRequest();
      request.open("GET", url, true);
      request.responseType = "arraybuffer";
      request.onload = function (ev) {
        if (request.readyState === 4) {
          if (request.status === 200) {
            let data = new Uint8Array(request.response);
            cv.FS_createDataFile("/", path, data, true, false, false);
            callback();
          } else {
            console.log("Failed");
          }
        }
      };
      request.send();
    },
  },
};
</script>
