# Follow the steps to configure audio stream to AWS S3

# Step1
- Clone the repository and navigate to the cloned repo directory.

# Step2
- Open [Source HTML](index-structured.html) file in any of your favourite editor.
- Go to the bottom of the HTML code and replace the config values mentioned below with newely generated values from your AWS account.
  ```
    var wRegion = "us-east-1"; # Region where the Pool id is present.
    var poolid = "us-east-1:XXXXXXXXX"; # Pool id
    var s3bucketName = "BUCKET_NAME"; # Your S3 Bucket name where you want the audio files to be uploaded.
  ```
  
# Step3
- Once the values are replaced, you can open [Source HTML](index-structured.html) in any web browser.
- It will prompt asking for the permission to access the microphone.
- Click Allow.

# Step4
- Done, Now you can see the Record button as enabled.
- Now you can enjoy streaming the live audio and upload it to S3.

---

# From the Blog [modified]
Source: https://www.srijan.net/resources/blog/aws-s3-audio-streaming

Through this blog, I’ll take you through the tutorial on uploading AWS live audio streaming to AWS S3 audio streaming using AWS SDK. We will use certain services of AWS which includes Amazon Cognito Identity Pools (federated identities), and S3 ofcourse.

To find a working example, refer to the blog - [uploading audio stream to AWS S3](https://github.com/triloknagvenkar/upload-audio-stream-to-aws-s3)

## AWS Configurations
### Step 1 - Create S3 Bucket
Assuming you have logged into the AWS console, let us get started by creating a S3 Bucket, where all the audio files will be stored. To create the bucket, navigate to AWS S3 –> Create bucket

### Step 2 - Create Federated Identity
Once the bucket is created, our next step is to create a Federated Identity which provides the necessary permission for a file upload from browser to S3 bucket.

To create the Federated Identity please navigate to the Cognito service - > Manage identity pools > Create new identity.

Give the Identity pool name and check the Enable access to unauthenticated identities Or Authentication Providers.

The next screen is all about setting the necessary permission for the Federated Identity via IAM roles. Here, we will create a new IAM role with specific permission defined via custom policy as policy mentioned below:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"s3:PutObject",
				"s3:GetObject",
				"s3:ListMultipartUploadParts",
				"s3:ListBucketMultipartUploads"
			],
			"Resource": "arn:aws:s3:::S3_BUCKET_NAME/*"
		}
	]
}
```

[view raw](https://gist.github.com/triloknagvenkar/82d52557f9052645481bb89b2575801b/raw/6364f573fc9d3d6803378c83157d98a46b100fbf/policy.json) [policy.json](https://gist.github.com/triloknagvenkar/82d52557f9052645481bb89b2575801b#file-policy-json) hosted by GitHub

Post creation, it will provide the Identity Pool Id. That ID is required to communicate with AWS services.

### Step 3 - Front-end App - This Repo
Now we will create a small front-end app to record and upload audio stream to S3.

#### HTML:

```html
<button type="button" class="btn kc record" id="record_q1" disabled="disabled" onclick="AudioStream.startRecording(this.id)">Record</button>
<button type="button" class="btn kc stop" id="stop_q1" disabled="disabled" onclick="AudioStream.stopRecording(this.id)">Stop</button>
```

[view raw](https://gist.github.com/triloknagvenkar/d3a4332cc387668f0c796aa09a164f02/raw/de5b77710e0475fd6896abd5f4846ffe202d7331/index.html) [index.html](https://gist.github.com/triloknagvenkar/d3a4332cc387668f0c796aa09a164f02#file-index-html) hosted by GitHub
 
#### JS:

We will create a AudioStream class which will have functions used in above HTML events and also the one used to upload the audio stream to s3.

Initialization:

1- audioStreamInitialize function is used to request the microphone permission, and on receiving the data, it will create a multi-part upload.

```js

audioStreamInitialize() {
	/*
	Feature detecting is a simple check for the existence of "navigator.mediaDevices.getUserMedia"
	 
	To use the microphone. we need to request permission.
	The parameter to getUserMedia() is an object specifying the details and requirements for each type of media you want to access.
	To use microphone it shud be {audio: true}
	 
	*/
	navigator.mediaDevices.getUserMedia(self.audioConstraints)
		.then(function (stream) {
			/*
			Creates a new MediaRecorder object, given a MediaStream to record.
			*/
			self.recorder = new MediaRecorder(stream);

			/*
			Called to handle the dataavailable event, which is periodically triggered each time timeslice milliseconds of media have been recorded
			(or when the entire media has been recorded, if timeslice wasn't specified).
			The event, of type BlobEvent, contains the recorded media in its data property.
			You can then collect and act upon that recorded media data using this event handler.
			*/
			self.recorder.addEventListener('dataavailable', function (e) {
				var normalArr = [];
				/*
				Here we push the stream data to an array for future use.
				*/
				self.recordedChunks.push(e.data);
				normalArr.push(e.data);

				/*
				here we create a blob from the stream data that we have received.
				*/
				var blob = new Blob(normalArr, {
					type: 'audio/webm'
				});

				/*
				if the length of recordedChunks is 1 then it means its the 1st part of our data.
				So we createMultipartUpload which will return an upload id.
				Upload id is used to upload the other parts of the stream
				 
				else.
				It Uploads a part in a multipart upload.
				*/
				if (self.recordedChunks.length == 1) {
					self.startMultiUpload(blob, self.filename)
				}
				else {
					/*
					self.incr is basically a part number.
					Part number of part being uploaded. This is a positive integer between 1 and 10,000.
					*/
					self.incr = self.incr + 1
					self.continueMultiUpload(blob, self.incr, self.uploadId, self.filename, self.bucketName);
				}
			})
		});
}
```
