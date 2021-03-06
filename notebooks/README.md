# Denoising the BodyPix Overlay

This notebook contains code used to translate noisy colored masks into properly formatted segmentation masks. The generated masks contain only 25 shades of gray from rgb(0,0,0) to rgb(24,24,24). 

## Hacking the BodyPix demo to dump images and overlays
To access the raw data (images and overlays) from the browser demo, we used the [AWS SDK for javascript](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/s3-example-photo-album.html) to write each image asset to an S3 bucket. Follow the instructions to correctly configure AWS service to allow dumping images to an S3 bucket from a browser application.




First, we separate each asset into its own canvas. 
```
// in the segmentBodyInRealTime() function, add:
const canvas_overlay = document.getElementById('output_overlay');   // canvas with id 'output_overlay' in html

// in the 'partmap' case, under the else clause
else {
      bodyPix.drawMask(
        canvas, video, coloredPartImageData, 0, //display raw image only
        maskBlurAmount, flipHorizontally);
      bodyPix.drawMask(
        canvas_overlay, video, coloredPartImageData, 1, //display overlay only
        maskBlurAmount, flipHorizontally);
```

Next, we initialize AWS S3 assets and include custom functions to format the images correctly for upload:
```
// near the top of the index.js file, we add:

var BucketName = "YOUR-BUCKET-NAME";
var bucketRegion = "us-west-2";
var IdentityPoolId = "YOUR_IDENTITY_POOL_ID";

AWS.config.update({
  region: bucketRegion,
  credentials: new AWS.CognitoIdentityCredentials({
    IdentityPoolId: IdentityPoolId
  })
});

var s3 = new AWS.S3({
  apiVersion: "2006-03-01",
  params: { Bucket: BucketName }
});

// Declare the following functions:

// Function sending canvas objects as image files to S3
function sendCanvas(canvas, time, prefix){
    var dataUrl = canvas.toDataURL("image/jpeg");
    var blobData = dataURItoBlob(dataUrl);
    var fileName = prefix + time.toString() + '.jpg';
    var params = {Key: fileName, ContentType: "image/jpeg", Body: blobData, Bucket: BucketName};
    s3.putObject(params, function(err, data){
        if (err) console.log(err, err.stack);                        // an error occurred
        else     console.log("successfully uploaded " + filename);   //successful response
        });
}

// Helper function to convert canvas data objects to blobs
function dataURItoBlob(dataURI) {
    var binary = atob(dataURI.split(',')[1]);
    var array = [];
    for(var i = 0; i < binary.length; i++) {
        array.push(binary.charCodeAt(i));
    }
    return new Blob([new Uint8Array(array)], {type: 'image/jpeg'});
}
```

Finally, we added the AWS SDK references to upload each canvas asset to S3 
```
// in the segmentBodyInRealTime() function, after the last else statement, we add:
var time_now = Date.now()
sendCanvas(canvas, time_now, 'raw-');
sendCanvas(canva_overlay, time_now, 'overlay-');
```

## Formatting dumped assets from BodyPix demo
After collecting samples for both the source and target subjects, use this notebook to convert the fuzzy colored overlay masks.
