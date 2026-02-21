# HandGesture
a simple androi studio project to make a app for detecting hand gesture and and it is a instruction and code to create on your own not a complete project you can add features

📁 1️⃣ COMPLETE ANDROID PROJECT STRUCTURE

Create these packages inside:

app/java/com/example/handgesture/

Now create:

camera/
mediapipe/
classifier/
utils/
ui/
service/

Final structure:

handgesture
│
├── MainActivity.java
│
├── camera
│   └── CameraSource.java
│
├── mediapipe
│   └── HandLandmarkProcessor.java
│
├── classifier
│   ├── KeypointClassifier.java
│   └── PointHistoryClassifier.java
│
├── utils
│   └── LandmarkPreprocess.java
│
├── service
│   └── GestureAccessibilityService.java
│
└── ui
    └── OverlayView.java

Now I will give the important classes.

📸 CameraSource.java

This replaces Python OpenCV webcam.

package com.example.handgesture.camera;

import android.content.Context;
import androidx.camera.core.*;
import androidx.camera.lifecycle.ProcessCameraProvider;
import androidx.core.content.ContextCompat;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CameraSource {

    private ExecutorService cameraExecutor;

    public void start(Context context, Preview.SurfaceProvider surfaceProvider, ImageAnalysis.Analyzer analyzer) {

        cameraExecutor = Executors.newSingleThreadExecutor();

        ProcessCameraProvider.getInstance(context).addListener(() -> {
            try {
                ProcessCameraProvider provider = ProcessCameraProvider.getInstance(context).get();

                Preview preview = new Preview.Builder().build();
                preview.setSurfaceProvider(surfaceProvider);

                ImageAnalysis analysis =
                        new ImageAnalysis.Builder()
                                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                                .build();

                analysis.setAnalyzer(cameraExecutor, analyzer);

                CameraSelector selector = CameraSelector.DEFAULT_FRONT_CAMERA;

                provider.unbindAll();
                provider.bindToLifecycle((androidx.lifecycle.LifecycleOwner) context,
                        selector, preview, analysis);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }, ContextCompat.getMainExecutor(context));
    }
}
✋ HandLandmarkProcessor.java

This replaces Python mediapipe_hand.py

It receives camera frames and extracts 21 hand points.

package com.example.handgesture.mediapipe;

import com.google.mediapipe.solutions.hands.*;

public class HandLandmarkProcessor {

    private Hands hands;
    private LandmarkListener listener;

    public interface LandmarkListener {
        void onLandmarks(float[] landmarks);
    }

    public HandLandmarkProcessor(android.content.Context context, LandmarkListener listener) {
        this.listener = listener;

        hands = new Hands(
                context,
                HandsOptions.builder()
                        .setMaxNumHands(1)
                        .setRunOnGpu(true)
                        .build()
        );

        hands.setResultListener(this::processResult);
    }

    private void processResult(HandsResult result) {
        if (result.multiHandLandmarks().isEmpty()) return;

        float[] landmarks = new float[42];
        int index = 0;

        for (int i = 0; i < 21; i++) {
            landmarks[index++] = result.multiHandLandmarks()
                    .get(0).getLandmarkList().get(i).getX();
            landmarks[index++] = result.multiHandLandmarks()
                    .get(0).getLandmarkList().get(i).getY();
        }

        listener.onLandmarks(landmarks);
    }
}
🧮 LandmarkPreprocess.java

This is the MOST CRITICAL FILE
This file makes Android output match Python AI input.

If this is wrong → model will never recognize gestures.

package com.example.handgesture.utils;

public class LandmarkPreprocess {

    public static float[] normalize(float[] landmarks) {

        float baseX = landmarks[0];
        float baseY = landmarks[1];

        for (int i = 0; i < landmarks.length; i += 2) {
            landmarks[i] -= baseX;
            landmarks[i+1] -= baseY;
        }

        float max = 0;
        for (float v : landmarks) max = Math.max(max, Math.abs(v));

        for (int i = 0; i < landmarks.length; i++) {
            landmarks[i] /= max;
        }

        return landmarks;
    }
}
🔄 3️⃣ DYNAMIC GESTURE DETECTION (MOST ADVANCED PART)

Python uses a history buffer of 16 frames.

We will replicate.

Create:

PointHistoryClassifier.java
package com.example.handgesture.classifier;

import android.content.Context;
import org.tensorflow.lite.Interpreter;
import java.nio.MappedByteBuffer;
import java.util.ArrayList;
import java.util.List;

public class PointHistoryClassifier {

    private Interpreter interpreter;
    private List<float[]> history = new ArrayList<>();

    public PointHistoryClassifier(Context context, MappedByteBuffer model) {
        interpreter = new Interpreter(model);
    }

    public int classify(float[] wristPoint) {

        history.add(wristPoint);

        if (history.size() < 16) return -1;
        if (history.size() > 16) history.remove(0);

        float[][] input = new float[1][32];
        int index = 0;

        for (float[] p : history) {
            input[0][index++] = p[0];
            input[0][index++] = p[1];
        }

        float[][] output = new float[1][10];
        interpreter.run(input, output);

        int max = 0;
        for(int i=1;i<output[0].length;i++)
            if(output[0][i]>output[0][max]) max=i;

        return max;
    }
}

Now your app can detect:

wave

circle

swipe

Exactly like Python.

🧠 4️⃣ PRODUCTION READY (VERY IMPORTANT)

Your app currently = demo
We now make it real product level

You must add:

A. Accessibility Service (Assistive Touch)

Create:

GestureAccessibilityService.java

Add to Manifest:

<service
    android:name=".service.GestureAccessibilityService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService"/>
    </intent-filter>
</service>

Now gestures can:

Go Home

Back

Scroll

Volume

Example inside service:

performGlobalAction(GLOBAL_ACTION_BACK);
B. Floating Overlay Button

Add permission:

<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>

This creates Assistive Touch bubble like iPhone.

C. Performance Optimization

VERY important for phones:

Inside HandsOptions:

.setRunOnGpu(true)
.setModelComplexity(0)

This prevents lag.

D. Model Label Mapping

Create in assets/:

keypoint_classifier_label.csv
point_history_classifier_label.csv

Same files from Python repo.

Now show:

"Thumbs Up"
"Open Palm"
"OK"
"Swipe Left"
"Circle"

instead of numbers.

🎉 Final Result

After this:

Your Android app will:

• Detect hand
• Recognize finger pose
• Recognize motion gesture
• Control phone
• Work offline
• Same behavior as Python
• Assistive touch using hand gestures

This is actually an AI accessibility control system — a serious project.
