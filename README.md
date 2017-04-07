HOW TO START YOUR OWN VR APP
 - Pretty simeple here, we just need to reference the Google vr api by adding the following to the gradle build file

 allprojects {
    repositories {
        // For GVR components, the official source of the .aars is JCenter.
        jcenter()
    }
}

dependencies {
    compile 'com.google.vr:sdk-audio:1.10.0'
    compile 'com.google.vr:sdk-base:1.10.0'
}

MANIFEST FILE

 - Start up notes
 - <uses-sdk android:minSdkVersion="19"/> indicates that the device must be running API Level 19 (KitKat) or higher.
 - <uses-sdk android:targetSdkVersion="24"/> indicates our app is targetting API Level 24 (Nougat).
 - <uses-feature android:glEsVersion="0x00020000" android:required="true" /> indicates that the device must support OpenGL ES 2.0 to run the sample app.
 - <uses-feature android:name="android.software.vr.mode" android:required="false" /> indicates (optional) use of Android N's VR mode, when available.
 - <uses-feature android:name="android.software.vr.high_performance" android:required="false" /> indicates (optional) use of Daydream-ready device features, when available.
 - android:screenOrientation="landscape" indicates that the activity's required screen orientation is "landscape." This is the orientation you must set for VR apps. The view used by the Google VR SDK, GvrView, only renders on fullscreen and landscape modes.
 - Use of Android N's VR mode is signaled with android:enableVrMode="gvr_vr_mode_component". This attribute will have no effect on older versions of Android.
 - The Google VR SDK-provided VR theme, specified with android:theme="@style/VrActivityTheme", should always be used to ensure a consistent fullscreen experience, particularly when transitioning between Daydream-compatible apps.
 - android:resizeableActivity="false" disable resizing support
 - android.permission.READ_EXTERNAL_STORAGE pair the users phone to the vr viewer, only needed when google vr virew service is unavailable

GVR ACTIVITY
 - starting point for any vr app (google VR SDK), exposes event to the vr app and handles details required when creating and activity for vr rendering.
 - sample apps MAINACTIVITY extends GVRACTIVITY, MAINACTIVITY implments following interface
 - gvrview.stereorender: interface that regulates sterostopic view.

DEFINE GVRVIEW
 - renders content in stereo
 - IT IS DEFINED IN LAYOUT.XML file of project!!

 <com.google.vr.sdk.base.GvrView
    android:id="@+id/gvr_view"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:layout_alignParentTop="true"
    android:layout_alignParentLeft="true" />

 - Main activity class inits gvrview in the ONCREATE() method

 /**
 * Sets the view to our GvrView and initializes the transformation matrices
 * we will use to render our scene.
 */
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.common_ui);
    GvrView gvrView = (GvrView) findViewById(R.id.gvr_view);
    // Associate a GvrView.StereoRenderer with gvrView.
    gvrView.setRenderer(this);
    // Associate the gvrView with this activity.
    setGvrView(gvrView);

    // Initialize other objects here.
    ...
}


RENDER THE VIEW
 - Once you load the GrView you associate it with a render, then you associate the gvrview with the activity

  - Usually the stereo render is GVRVIEW.STEREORENDER (two methods)
  - onEyeFrame() called every time the app renders
  - onEyeDraw() called for each eye with different eye parameters

ONNEWFRAME()
- this method call implments logic before the eyes are rendered for the user, good place to update the model, this method can be seen in the code if the user is looking at a treasure


/**
 * Prepares OpenGL ES before we draw a frame.
 * @param headTransform The head transformation in the new frame.
 */
@Override
public void onNewFrame(HeadTransform headTransform) {
    ...
    headTransform.getHeadView(mHeadView, 0);
    ...
}


ONDRAWEYE()

 - per eye configuration method, usually contains the meat of the code, EYE object contains projection matricies for the eye



 /**
 * Draws a frame for an eye.
 *
 * @param eye The eye to render. Includes all required transformations.
 */
@Override
public void onDrawEye(Eye eye) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);
    ...
    // Apply the eye transformation to the camera.
    Matrix.multiplyMM(mView, 0, eye.getEyeView(), 0, mCamera, 0);

    // Set the position of the light
    Matrix.multiplyMV(mLightPosInEyeSpace, 0, mView, 0, LIGHT_POS_IN_WORLD_SPACE, 0);

    // Build the ModelView and ModelViewProjection matrices
    // for calculating cube position and light.
    float[] perspective = eye.getPerspective(Z_NEAR, Z_FAR);
    Matrix.multiplyMM(mModelView, 0, mView, 0, mModelCube, 0);
    Matrix.multiplyMM(mModelViewProjection, 0, perspective, 0, mModelView, 0);
    drawCube();

    // Draw the rest of the scene.
    ...
}

This is the sequence of events:

 - The treasure comes into eye space.
 - We apply the projection matrix. This provides the scene rendered for the specified eye.
 - The Google VR SDK applies distortion automatically, to render the final scene.

RENDERING SPATIAL AUDIO
 - oncreate() method initializes audio engine
 - GvrAudioEngine allows the user to specify a rendering mode defining the spatialization fidelity.

 gvrAudioEngine =
    new GvrAudioEngine(this, GvrAudioEngine.RenderingMode.BINAURAL_HIGH_QUALITY);

 - To disable audio when the user pauses the app, and enable it again when they resume, call gvrAudioEngine.pause(); and gvrAudioEngine.resume(); in the onPause() and onResume() functions respectively

 new Thread(
       new Runnable() {
         @Override
         public void run() {
           gvrAudioEngine.preloadSoundFile(SOUND_FILE);
         }
       })
   .start();


 - we may need to preload audio files in order to not let the phone render the files on disk, to fix this it uses the following code.

 // Start spatial audio playback of SOUND_FILE at the model postion. The returned
// sourceId handle allows for repositioning the sound object whenever the cube
// position changes.
sourceId = gvrAudioEngine.createSoundObject(SOUND_FILE);
gvrAudioEngine.setSoundObjectPosition(
    sourceId, modelPosition[0], modelPosition[1], modelPosition[2]);
gvrAudioEngine.playSound(sourceId, true /* looped playback */);



HANDLING INPUTS

 - to provide functionaltiy for when the user touches the screen, we should use GvrActivity.onCardboardTrigger()



 /**
 * Called when the Cardboard trigger is pulled.
 */
@Override
public void onCardboardTrigger() {
    if (isLookingAtObject()) {
        hideObject();
    }

    // Always give user feedback
    mVibrator.vibrate(50);
}
