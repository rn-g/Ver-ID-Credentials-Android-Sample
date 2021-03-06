# Ver-ID Credentials Sample

![](app/src/main/res/drawable-mdpi/woman_with_licence.png)

The project contains a sample application that uses Microblink's [BlinkID SDK](https://github.com/BlinkID/blinkid-android) to scan an ID card. The app uses [Ver-ID SDK](https://github.com/AppliedRecognition/Ver-ID-UI-Android) to detect a face on the captured ID card and compare it to a live selfie taken with the Android device's camera.

## Adding Ver-ID to your Android Studio project

1. [Request an API secret](https://dev.ver-id.com/admin/register) for your app.
1. Open your app module's **gradle.build** file and add:

    ```groovy
    repositories {
        maven { url 'https://dev.ver-id.com/artifactory/gradle-release' }
    }
    
    dependencies {
        implementation 'com.appliedrec.verid:rx:[1.6,2.0['
        implementation 'com.appliedrec.verid:ui:[1.14.4,2.0.0['
    }
    ```
    
1. Open your app's **AndroidManifest.xml** file and add the following tag in `<application>` replacing `[your API secret]` with the API secret your received in step 1:

    ~~~xml
    <meta-data
        android:name="com.appliedrec.verid.apiSecret"
        android:value="[your API secret]" />
    ~~~

## Adding Microblink to your Android Studio project

1. Apply for an API key on the [Microblink website](https://microblink.com/products/blinkid).
1. Open your app module's **gradle.build** file and add:


    ```groovy
    repositories {
        maven { url 'http://maven.microblink.com' }
    }
    
    dependencies {
        implementation('com.microblink:blinkid:5.0.0@aar') {
            transitive = true
        }
    }
    ```

1. Detailed instructions are available on the [BlinkID Github page](https://github.com/BlinkID/blinkid-android#-sdk-integration). 


## Example 1 – Capture ID card

~~~java
public class MyActivity extends AppCompatActivity {
	
    private RecognizerBundle recognizerBundle;
    private static final int REQUEST_CODE_ID_CAPTURE = 0;
    
    /**
     * Call this method to start the ID capture 
     * (for example in response to a button click).
     */
    void runIdCapture() {
        try {
            // Set the Microblink licence key
            // This example assumes the key is set in your build.gradle file
            MicroblinkSDK.setLicenseKey(BuildConfig.BLINK_LICENCE_KEY, this);
        } catch (InvalidLicenceKeyException e) {
            // You'll want to handle this better
            return;
        }
        // To enable high-res images in intents
        MicroblinkSDK.setIntentDataTransferMode(IntentDataTransferMode.PERSISTED_OPTIMISED);
        // To detect US or Canadian ID card
        UsdlCombinedRecognizer recognizer = new UsdlCombinedRecognizer();
        recognizer.setEncodeFullDocumentImage(true);
        // For ID cards issued outside USA or Canada uncomment the following 2 lines and delete the 2 lines above
        // BlinkIdCombinedRecognizerrecognizer = new BlinkIdCombinedRecognizer();
        // recognizer.setEncodeFullDocumentImage(true);
        SuccessFrameGrabberRecognizer successFrameGrabberRecognizer = new SuccessFrameGrabberRecognizer(recognizer);
        recognizerBundle = new RecognizerBundle(successFrameGrabberRecognizer);
        startActivityForResult(intent, REQUEST_CODE_ID_CAPTURE);
	}
	
    /**
     * Listen for the result of the ID capture and display 
     * the card and detected face in your activity.
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_ID_CAPTURE && resultCode == RESULT_OK && data != null) {
            // Load the ID capture result from the data intent
            recognizerBundle.loadFromIntent(data);

            Recognizer firstRecognizer = recognizerBundle.getRecognizers()[0];
            SuccessFrameGrabberRecognizer successFrameGrabberRecognizer = (SuccessFrameGrabberRecognizer) firstRecognizer;
            
            byte[] frontImage;
            if (successFrameGrabberRecognizer.getSlaveRecognizer() instanceof UsdlCombinedRecognizer) {
                frontImage = ((UsdlCombinedRecognizer) successFrameGrabberRecognizer.getSlaveRecognizer()).getResult().getEncodedFullDocumentImage();
            } else if (successFrameGrabberRecognizer.getSlaveRecognizer() instanceof BlinkIdCombinedRecognizer) {
                frontImage = ((BlinkIdCombinedRecognizer) successFrameGrabberRecognizer.getSlaveRecognizer()).getResult().getEncodedFrontFullDocumentImage();
            } else {
                return;
            }
            // Save the image of the front of the card in your app's files
            File imageFile = new File(getFilesDir(), "cardFront.jpg");
            FileOutputStream outputStream = new FileOutputStream(imageFile);
            ByteArrayInputStream inputStream = new ByteArrayInputStream(frontImage);
            int read;
            byte[] buffer = new byte[512];
            while ((read = inputStream.read(buffer, 0, buffer.length)) > 0) {
                outputStream.write(buffer, 0, read);
            }
            outputStream.close();
            inputStream.close();
		}
	}
}
~~~

## Example 2 – Capture live face

~~~java
public class MyActivity extends AppCompatActivity {
	
    private static final int REQUEST_CODE_LIVENESS_DETECTION = 1;
    private RxVerID rxVerID;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my);
        rxVerID = new RxVerID.Builder(this).build();
    }
        	
    /**
     * Call this method to start the liveness detection session
     * (for example in response to a button click).
     */
    void runLivenessDetection() {
        rxVerID.getVerID() // Load Ver-ID
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(
                verID -> {
                    // Create liveness detection settings
                    LivenessDetectionSessionSettings settings = new LivenessDetectionSessionSettings();
                    // Construct the liveness detection intent
                    VerIDLivenessDetectionIntent intent = new VerIDLivenessDetectionIntent(this, verID, settings);
                    // Start the liveness detection activity
                    startActivityForResult(intent, REQUEST_CODE_LIVENESS_DETECTION);
                },
                error -> {
                    // Ver-ID failed to load
                });
    }
    	
    /**
     * Listen for the result of the liveness detection
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_LIVENESS_DETECTION && resultCode == RESULT_OK) {
            rxVerID.getSessionResultFromIntent(data)
                .flatMapObservable(result -> rxVerID.getFacesAndImageUrisFromSessionResult(result, Bearing.STRAIGHT))
                .filter(detectedFace -> detectedFace.getFace() instanceof RecognizableFace)
                .firstOrError()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                        detectedFace -> {
                            // You can now use the face for face recognition:
                            RecognizableFace recognizableFace = (RecognizableFace) detectedFace.getFace();
                        },
                        error -> {
                            // Failed to get the first face from the result
                        }
                ));
        }
    }	
}
~~~

## Example 3 - Compare face on ID card with live face

Building on example 1 and 2, you can use the results of the ID capture and liveness detection sessions and compare their faces.

This class takes as input the image file of the front of the card captured in example 1 and the `recognizableFace` captured in example 2.

~~~java
class FaceComparison {
    
    private final RxVerID rxVerID;
    
    /**
     * Pass an instance of RxVerID to the constructor
     */
    FaceComparison(RxVerID rxVerID) {
        this.rxVerID = rxVerID;
    }
    
    /**
     * This function returns a Single whose value is a pair of Floats.
     * The first Float is the comparison score between the two faces.
     * The second Float in the pair is the threshold required to consider the two faces as authenticated against each other.
     */
    Single<Pair<Float,Float>> compareIDCardToLiveFace(Uri imageFileUri, RecognizableFace face) {
        return rxVerID.detectRecognizableFacesInImage(imageFileUri, 1)
            .singleOrError()
            .flatMap(cardFace -> rxVerID.compareFaceToFaces(cardFace, new RecognizableFace[]{face}))
            .flatMap(score -> rxVerID.getVerID().map(verID -> new Pair<>(score, verID.getFaceRecognition().getAuthenticationThreshold())))
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread());
    }
}
~~~

## Links

### Ver-ID
- [Github](https://github.com/AppliedRecognition/Ver-ID-UI-Android)
- [Reference documentation](https://appliedrecognition.github.io/Ver-ID-UI-Android/)

### Rx-Ver-ID
- [Github](https://github.com/AppliedRecognition/Rx-Ver-ID-Android)
- [Reference documentation](https://appliedrecognition.github.io/Rx-Ver-ID-Android/)

### BlinkID
- [Github](https://github.com/BlinkID/blinkid-android)
- [Reference documentation](https://blinkid.github.io/blinkid-android/)
