// 📤 YouTube Upload Manager via Google Apps Script
// This document serves as a complete user guide for setting up and customizing the YouTube Upload Automation Script using Google Apps Script and Google Sheets.

// ✅ Overview
// This script automatically uploads videos to your YouTube channel based on information stored in a Google Spreadsheet. It fetches video files from a designated Google Drive folder and uses YouTube Data API to publish videos with the metadata you define.

//📋 Prerequisites
//To use this script, you need:
//1. A Google Cloud project with YouTube Data API v3 enabled.
//2. OAuth2 authorization for the script with YouTube scope.
//3. A Google Sheet containing metadata columns.
//4. A Google Drive folder containing your video files.
//5. Proper authorization to run YouTube.Videos.insert().

//🧰 Setup Instructions

//📁 Step 1: Prepare Google Drive Folder
//1. Create a new folder in Google Drive.
//2. Upload your video files to this folder.
//3. Copy the folder ID from the URL (after /folders/).
//Example: https://drive.google.com/drive/folders/YOUR_FOLDER_ID

//📊 Step 2: Prepare Google Spreadsheet
//1. Create a new Google Sheet.
//2. Rename one sheet to your preferred name, e.g., VideoSchedule.
//3. Create the following headers in Row 1:

//A: File Name, B: Title, C: Description, D: Tags, E: Category ID, F: Video ID, G: Status

//Fill in rows below with video metadata:
//File Name: must match the exact file name in your Drive folder
//Tags: comma-separated (e.g., education, tutorial)
//Category ID: see list in section below

//Video ID and Status will be auto-filled by the script

//🔑 Step 3: Enable YouTube Data API
//1. Go to Google Cloud Console
//2. Create a new project (or use an existing one)
//3. Enable the YouTube Data API v3 for the project
//4. Add OAuth2 credentials and authorize the Apps Script with YouTube scope
 
//🧠 Step 4: Use the Script
//1. Open Apps Script Editor from your Google Sheet: Extensions > Apps Script
//2. Paste the full script and replace: YOUR_DRIVE_FOLDER_ID, YOUR_SPREADSHEET_ID, YOUR_SHEET_NAME
//3. Save the script
//4. Run the function uploadVideoAuto()
//- The script will:
//- Check the next video in the list that hasn't been uploaded
//- Upload it to YouTube with metadata
//- Fill in the Video ID and set the status to Done

//📦 YouTube Category ID Reference
//Here are common category IDs: https://developers.google.com/youtube/v3/docs/videoCategories/list

//⚠️ Common Issues
//- File not found in folder → ensure file name matches exactly.
//- Status not updated → check quota and permission to write to sheet.
//- Script error YouTube is not defined → ensure you enabled YouTube Data API v3 in Cloud Console.

//🛠️ Optional Customizations
//- Change privacyStatus to private or unlisted in the script.
//- Add scheduled publish time via status.publishAt if needed.
//- Add email notifications upon successful upload using GmailApp.


//SCRIPT:


function uploadVideoAuto() {
  const driveFolderId = "YOUR_DRIVE_FOLDER_ID"; // Replace with your Google Drive folder ID
  const spreadsheetId = "YOUR_SPREADSHEET_ID"; // Replace with your Google Sheet ID
  const sheetName = "YOUR_SHEET_NAME"; // Replace with your target sheet name

  const folder = DriveApp.getFolderById(driveFolderId);
  const sheet = SpreadsheetApp.openById(spreadsheetId).getSheetByName(sheetName);
  const data = sheet.getDataRange().getValues();

  let lastUploadedRowIndex = 0;

  // Find the last row where the status is "Done"
  for (let i = 1; i < data.length; i++) {
    if (data[i][6] === "Done") { // Column G = Status
      lastUploadedRowIndex = i;
    }
  }

  const nextRowIndex = lastUploadedRowIndex + 1;

  // Skip if there's no new video or Video ID already exists
  if (nextRowIndex >= data.length || data[nextRowIndex][5]) { // Column F = Video ID
    Logger.log("No new video to upload.");
    return;
  }

  const [fileName, title, description, tags, categoryId] = data[nextRowIndex];

  // Find the video file in Google Drive
  const files = folder.getFilesByName(fileName);
  if (!files.hasNext()) {
    Logger.log("File not found: " + fileName);
    return;
  }

  const file = files.next();
  const mediaBlob = file.getBlob();

  try {
    // Upload the video to YouTube
    const video = YouTube.Videos.insert(
      {
        snippet: {
          title: title,
          description: description,
          tags: tags ? tags.split(", ") : [],
          categoryId: categoryId
        },
        status: {
          privacyStatus: "public", // Set to "private" or "unlisted" if needed
          selfDeclaredMadeForKids: false
        }
      },
      "snippet,status",
      mediaBlob
    );

    // Save the video ID and update status to "Done"
    sheet.getRange(nextRowIndex + 1, 6).setValue(video.id); // Column F = Video ID
    sheet.getRange(nextRowIndex + 1, 7).setValue("Done");   // Column G = Status

    Logger.log("Video uploaded successfully: " + video.id);

  } catch (e) {
    Logger.log("Upload failed: " + e.message);
  }
}
