<script lang="ts">
    import './styles.css'
    import { onMount } from "svelte";
    import {
    FaceDetector,
    FilesetResolver,
    } from "@mediapipe/tasks-vision";
let score = 0;
onMount(async () => { 

  const demosSection = document.getElementById("demos");

let faceDetector: FaceDetector;
let runningMode: string = "IMAGE";

// Initialize the object detector
const initializefaceDetector = async () => {
  const vision = await FilesetResolver.forVisionTasks(
    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm"
  );
  faceDetector = await FaceDetector.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: `https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite`,
      delegate: "GPU"
    },
    runningMode: runningMode
  });
  demosSection.classList.remove("invisible");
};
initializefaceDetector();
 
/********************************************************************
 // Demo 2: Continuously grab image from webcam stream and detect it.
 ********************************************************************/

let video = document.getElementById("webcam");
const liveView = document.getElementById("liveView");
let enableWebcamButton: HTMLButtonElement;

// Check if webcam access is supported.
const hasGetUserMedia = () => !!navigator.mediaDevices?.getUserMedia;

// Keep a reference of all the child elements we create
// so we can remove them easilly on each render.
var children = [];

// If webcam supported, add event listener to button for when user
// wants to activate it.
if (hasGetUserMedia()) {
  enableWebcamButton = document.getElementById("webcamButton");
  enableWebcamButton.addEventListener("click", enableCam);
} else {
  console.warn("getUserMedia() is not supported by your browser");
}

// Enable the live webcam view and start detection.
async function enableCam(event) {
  if (!faceDetector) {
    alert("Face Detector is still loading. Please try again..");
    return;
  }

  // Hide the button.
  enableWebcamButton.classList.add("removed");

  // getUsermedia parameters
  const constraints = {
    video: true
  };

  // Activate the webcam stream.
  navigator.mediaDevices
    .getUserMedia(constraints)
    .then(function (stream) {
      video.srcObject = stream;
      video.addEventListener("loadeddata", predictWebcam);
    })
    .catch((err) => {
      console.error(err);
    });
}

let lastVideoTime = -1;
async function predictWebcam() {
  // if image mode is initialized, create a new classifier with video runningMode
  if (runningMode === "IMAGE") {
    runningMode = "VIDEO";
    await faceDetector.setOptions({ runningMode: "VIDEO" });
  }
  let startTimeMs = performance.now();

  // Detect faces using detectForVideo
  if (video.currentTime !== lastVideoTime) {
    lastVideoTime = video.currentTime;
    const detections = faceDetector.detectForVideo(video, startTimeMs)
      .detections;
    displayVideoDetections(detections);
  }

  // Call this function again to keep predicting when the browser is ready
  window.requestAnimationFrame(predictWebcam);
}

function displayVideoDetections(detections: Detection[]) {
  // Remove any highlighting from previous frame.

  for (let child of children) {
    liveView.removeChild(child);
  }
  children.splice(0);

  // Iterate through predictions and draw them to the live view
  for (let detection of detections) {
    const p = document.createElement("p");
    p.innerText =
      "Confidence: " +
      Math.round(parseFloat(detection.categories[0].score) * 100) +
      "% .";
      score = Math.round(parseFloat(detection.categories[0].score) * 100)

    p.style =
      "left: " +
      (video.offsetWidth -
        detection.boundingBox.width -
        detection.boundingBox.originX) +
      "px;" +
      "top: " +
      (detection.boundingBox.originY - 30) +
      "px; " +
      "width: " +
      (detection.boundingBox.width - 10) +
      "px;";

    const highlighter = document.createElement("div");
    highlighter.setAttribute("class", "highlighter");
    highlighter.style =
      "left: " +
      (video.offsetWidth -
        detection.boundingBox.width -
        detection.boundingBox.originX) +
      "px;" +
      "top: " +
      detection.boundingBox.originY +
      "px;" +
      "width: " +
      (detection.boundingBox.width - 10) +
      "px;" +
      "height: " +
      detection.boundingBox.height +
      "px;";

    liveView.appendChild(highlighter);
    liveView.appendChild(p);

    // Store drawn objects in memory so they are queued to delete at next call
    children.push(highlighter);
    children.push(p);
    for (let keypoint of detection.keypoints) {
         //Taking the photo
  if (score > 70){
      document.getElementById("snap").disabled = false;
      // Elementos para tomar la foto
      var canvas = document.getElementById('canvas');
      var context = canvas.getContext('2d');
      // Evento de escucha para el botón
      document.getElementById("snap").addEventListener("click", async function() {
        context.drawImage(video, 0, 0, 640, 480);
        const imageBase64 = canvas.toDataURL();
        console.log(imageBase64);
        // Ocultar el canvas después de tomar la foto
        canvas.style.display = 'none';
        if (navigator.onLine) {
            // Send the image data to the server
            const response = await fetch('/api/inPhoto', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ imageData: imageBase64 })
            });
            if (response.ok) {
                console.log("Data inserted successfully");
            } else {
                console.error("Error inserting data:", response.statusText); 
            }
        } else {
            // Save the photo locally
            localStorage.setItem('photo', imageBase64);
            console.log("Photo saved locally");
        }

      });
    } else {
      document.getElementById("snap").disabled = true;
    }
//End taking the photo
      const keypointEl = document.createElement("spam");
      keypointEl.className = "key-point";
      keypointEl.style.top = `${keypoint.y * video.offsetHeight - 3}px`;
      keypointEl.style.left = `${
        video.offsetWidth - keypoint.x * video.offsetWidth - 3
      }px`;
      liveView.appendChild(keypointEl);
      children.push(keypointEl);
    }
  }
}

// Add this function to send the locally saved photo to the server
async function sendLocalPhoto() {
        const localPhoto = localStorage.getItem('photo');
        if (localPhoto) {
            const response = await fetch('/api/inPhoto', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ imageData: localPhoto })
            });
            if (response.ok) {
                console.log("Local photo sent to server");
                localStorage.removeItem('photo');
            } else {
                console.error("Error sending local photo:", response.statusText);
            }
        }
    }

// Listen for when the device goes back online
window.addEventListener('online', function() {
            setTimeout(sendLocalPhoto, 5000); // waits 5 seconds before calling sendLocalPhoto function
        });
});

</script>

<h1>Face detection using the MediaPipe Face Detector task</h1>

<section id="demos" class="invisible">
	<div id="liveView" class="videoView">
		<button id="webcamButton" class="mdc-button mdc-button--raised">
			<span class="mdc-button__ripple" />
			<span class="mdc-button__label">ENABLE WEBCAM</span>
		</button>
    <button id="snap">Snap Photo</button>
		<video id="webcam" autoplay playsinline />
    <canvas id="canvas" width="640" height="480"></canvas>
	</div>
</section>
