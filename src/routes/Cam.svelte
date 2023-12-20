
<script>
    // @ts-ignore
    import { onMount } from "svelte";
    import {
    FaceDetector,
    FilesetResolver,
    } from "@mediapipe/tasks-vision";
    // @ts-ignore
    let stream;
    // @ts-ignore
    let videoRef;
    // @ts-ignore
    /**
	 * @type {HTMLCanvasElement}
	 */
    let canvasRef;

    
  //@ts-ignore
  let faceDetector;
  let runningMode = "IMAGE";
  
  // Initialize the object detector
  const initializefaceDetector = async () => {
    const demosSection = document.getElementById("demos");
    const vision = await FilesetResolver.forVisionTasks(
      "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm"
    );
    faceDetector = await FaceDetector.createFromOptions(vision, {
      baseOptions: {
        modelAssetPath: `https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite`,
        delegate: "GPU"
      },
      // @ts-ignore
      runningMode: runningMode
    });
    //@ts-ignore
    demosSection.classList.remove("invisible");
  };

  let lastVideoTime = -1;
  async function predictWebcam() {
    // if image mode is initialized, create a new classifier with video runningMode
    if (runningMode === "IMAGE") {
      runningMode = "VIDEO";
      await faceDetector.setOptions({ runningMode: "VIDEO" });
    }
    let startTimeMs = performance.now();
  
    // Detect faces using detectForVideo
    //@ts-ignore
    if (videoRef.currentTime !== lastVideoTime) {
        //@ts-ignore
      lastVideoTime = videoRef.currentTime;
      //@ts-ignore
    // Ahora el video está listo para ser reproducido y puedes detectar caras
     videoRef.addEventListener('canplay', function() {
        const detections = faceDetector.detectForVideo(videoRef, startTimeMs).detections;
        displayVideoDetections(detections);
        });
    }
     // Call this function again to keep predicting when the browser is ready
    window.requestAnimationFrame(predictWebcam);
  }
  
  function displayVideoDetections(detections) {
    const liveView = document.getElementById("liveView");
    const children = [];
    // Remove any highlighting from previous frame.
    //@ts-ignore
    for (let child of children) {
    //@ts-ignore
      liveView.removeChild(child);
    }
    //@ts-ignore
    children.splice(0);
  
    // Iterate through predictions and draw them to the live view
    for (let detection of detections) {
      const p = document.createElement("p");
      p.innerText =
        "Confidence: " +
        Math.round(parseFloat(detection.categories[0].score) * 100) +
        "% .";
      p.style =
        "left: " +
        (videoRef.offsetWidth -
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
      highlighter.style.setProperty("left", (videoRef.offsetWidth - detection.boundingBox.width - detection.boundingBox.originX) + "px");
      highlighter.style.setProperty("top", detection.boundingBox.originY + "px");
      highlighter.style.setProperty("width", (detection.boundingBox.width - 10) + "px");
      highlighter.style.setProperty("height", detection.boundingBox.height + "px");
        "left: " +
        (videoRef.offsetWidth -
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
  
      if (liveView) {
        document.body.appendChild(highlighter);
      }
      if (liveView) {
        liveView.appendChild(p);
      }  
      // Store drawn objects in memory so they are queued to delete at next call
      children.push(highlighter);
      children.push(p);
      for (let keypoint of detection.keypoints) {
        console.log(keypoint);
        const keypointEl = document.createElement("spam");
        keypointEl.className = "key-point";
        keypointEl.style.top = `${keypoint.y * videoRef.offsetHeight - 3}px`;
        keypointEl.style.left = `${
          videoRef.offsetWidth - keypoint.x * videoRef.offsetWidth - 3
        }px`;
        if (liveView) {
          liveView.appendChild(keypointEl);
        }
        children.push(keypointEl);
      }
    }
  }
  
  async function getStream() {
        try {
            stream = await navigator.mediaDevices.getUserMedia({
                video: true,
                audio: false
            });
            // @ts-ignore
            videoRef.srcObject = stream;
        } catch (err) {
            console.error(err);
        }
        // @ts-ignore
        console.log(stream.getTracks()[0])
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

    async function takePhoto() {
        // @ts-ignore
        canvasRef.getContext('2d').drawImage(videoRef, 0, 0, canvasRef.width, canvasRef.height);
        const dataUrl = canvasRef.toDataURL('image/png');

        if (navigator.onLine) {
            // Send the image data to the server
            const response = await fetch('/api/inPhoto', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ imageData: dataUrl })
            });
            if (response.ok) {
                console.log("Data inserted successfully");
            } else {
                console.error("Error inserting data:", response.statusText); 
            }
        } else {
            // Save the photo locally
            localStorage.setItem('photo', dataUrl);
            console.log("Photo saved locally");
        }
    }

    onMount(async () => {
        getStream();
        // Listen for when the device goes back online
        window.addEventListener('online', function() {
            setTimeout(sendLocalPhoto, 5000); // waits 5 seconds before calling sendLocalPhoto function
        });
        await initializefaceDetector();
        predictWebcam();
    });

</script>

<section class="container mx-auto px-4" id="demos">
    <h1 class="text-4xl text-blue-500 my-4">Webcam Stream Mastery</h1>
    <button class="rounded-sm bg-blue-600 text-white px-4 py-2" on:click|preventDefault={takePhoto}>Take Photo</button>
    <!-- svelte-ignore a11y-media-has-caption -->
    <div id="liveView" class="videoView">
    <video class="mt-4 rounded-sm " width="640" height="480" autoplay={true} bind:this={videoRef} />
    <canvas class="mt-4 rounded-sm" width="640" height="480" bind:this={canvasRef}></canvas>
</section>
