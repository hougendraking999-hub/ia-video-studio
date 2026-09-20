// Google Apps Script: Code.gs
function doPost(e) {
  const params = JSON.parse(e.postData.contents);
  
  // 1. Validate user & payment (Stripe) - you can add later
  const userId = params.userId;
  const prompt = params.prompt;
  const duration = params.duration || 15; // seconds
  
  // 2. Generate AI video via Replicate
  const replicateKey = 'your_replicate_api_key';
  const replicatePayload = {
    "version": "stability-ai/stable-video-diffusion:3f0457e4619daac51203dedb472816fd4af51f3149fa7a9e0b5ffcf1b8172438",
    "input": {
      "prompt": prompt,
      "frames": duration * 30,
      "fps": 30,
      "num_frames": duration * 30
    }
  };
  
  const replicateResponse = UrlFetchApp.fetch('https://api.replicate.com/v1/predictions', {
    method: 'post',
    headers: {
      'Authorization': 'Token ' + replicateKey,
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(replicatePayload)
  });
  
  const prediction = JSON.parse(replicateResponse.getContentText());
  const videoUrl = prediction.output; // wait for completion in production
  
  // 3. Download video and upload to Bunny Storage
  const videoBlob = UrlFetchApp.fetch(videoUrl).getBlob();
  const bunnyStorageApiKey = 'your_bunny_storage_api_key';
  const storageZone = 'your_storage_zone';
  const fileName = userId + '_' + Date.now() + '.mp4';
  
  const uploadUrl = 'https://storage.bunnycdn.com/' + storageZone + '/' + fileName;
  const uploadResponse = UrlFetchApp.fetch(uploadUrl, {
    method: 'put',
    headers: {
      'AccessKey': bunnyStorageApiKey,
      'Content-Type': 'video/mp4'
    },
    payload: videoBlob
  });
  
  // 4. Create Bunny Flow stream
  const bunnyFlowApiKey = 'your_bunny_flow_api_key';
  const libraryId = 'your_library_id';
  
  const streamPayload = {
    "title": prompt.substring(0, 50),
    "collectionId": "your_collection_id",
    "thumbnailTime": 1
  };
  
  const streamResponse = UrlFetchApp.fetch('https://video.bunnycdn.com/library/' + libraryId + '/videos', {
    method: 'post',
    headers: {
      'AccessKey': bunnyFlowApiKey,
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(streamPayload)
  });
  
  const streamData = JSON.parse(streamResponse.getContentText());
  const videoId = streamData.guid;
  
  // 5. Fetch video file URL from Bunny Storage and tell Bunny Flow to pull it
  const fetchUrl = 'https://video.bunnycdn.com/library/' + libraryId + '/videos/' + videoId + '/fetch';
  const fetchPayload = {
    "url": 'https://your-storage-zone.b-cdn.net/' + fileName
  };
  
  UrlFetchApp.fetch(fetchUrl, {
    method: 'post',
    headers: {
      'AccessKey': bunnyFlowApiKey,
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(fetchPayload)
  });
  
  // 6. Return embed URL to user
  const embedUrl = `https://iframe.mediadelivery.net/embed/${libraryId}/${videoId}`;
  
  return ContentService.createTextOutput(JSON.stringify({
    success: true,
    embedUrl: embedUrl,
    videoId: videoId
  })).setMimeType(ContentService.MimeType.JSON);
}

function doGet() {
  return HtmlService.createHtmlOutputFromFile('index');
}
