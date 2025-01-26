# Step-by-Step Guide to Uploading a Video to YouTube Using Python and YouTube DATA API

### Introduction
This guide will walk you through the process of uploading a video to YouTube using Python and the YouTube Data API. By following these steps, you will learn how to:

- Set up the Google APIs Client Library for Python
- Create and configure a Google Cloud project for OAuth 2.0
- Write and execute a Python script to upload your video

### Requirements
1. Python 2.5 or higher (Python 3.x recommended).
2. Install the Google APIs Client Library for Python (`google-api-python-client`).
3. Create and configure a `client_secrets.json` file.

---

### Step 1: Install the Required Libraries

First, install the necessary Python libraries:
```bash
pip install google-api-python-client oauth2client
```

---

### Step 2: Set Up Your Google Cloud Project
1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project or select an existing one.
3. Enable the **YouTube Data API v3** for your project.
4. Navigate to **APIs & Services > Credentials** and create an **OAuth 2.0 Client ID**:
   - Choose "Web Application" as the application type.
   - Add a redirect URI (e.g., `http://localhost:8080`).
5. Download the `client_secrets.json` file and place it in the same directory as your Python script.

---

### Step 3: Prepare the `client_secrets.json` File
Ensure your `client_secrets.json` file looks like this:

```json
{
  "web": {
    "client_id": "YOUR_CLIENT_ID",
    "project_id": "YOUR_PROJECT_ID",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "YOUR_CLIENT_SECRET",
    "redirect_uris": ["http://localhost:8080"]
  }
}
```

Replace `YOUR_CLIENT_ID` and `YOUR_CLIENT_SECRET` with the values from your Google Cloud project.

---

### Step 4: Write the Python Script

Create a Python script named `upload_video.py` with the following content:

```python
import httplib2
import os
import random
import sys
import time

from apiclient.discovery import build
from apiclient.errors import HttpError
from apiclient.http import MediaFileUpload
from oauth2client.client import flow_from_clientsecrets
from oauth2client.file import Storage
from oauth2client.tools import argparser, run_flow

httplib2.RETRIES = 1
MAX_RETRIES = 10
RETRIABLE_EXCEPTIONS = (httplib2.HttpLib2Error, IOError)
RETRIABLE_STATUS_CODES = [500, 502, 503, 504]

CLIENT_SECRETS_FILE = "client_secrets.json"
YOUTUBE_UPLOAD_SCOPE = "https://www.googleapis.com/auth/youtube.upload"
YOUTUBE_API_SERVICE_NAME = "youtube"
YOUTUBE_API_VERSION = "v3"

VALID_PRIVACY_STATUSES = ("public", "private", "unlisted")

def get_authenticated_service(args):
    flow = flow_from_clientsecrets(CLIENT_SECRETS_FILE, scope=YOUTUBE_UPLOAD_SCOPE)
    storage = Storage(f"{sys.argv[0]}-oauth2.json")
    credentials = storage.get()

    if credentials is None or credentials.invalid:
        credentials = run_flow(flow, storage, args)

    return build(YOUTUBE_API_SERVICE_NAME, YOUTUBE_API_VERSION, http=credentials.authorize(httplib2.Http()))

def initialize_upload(youtube, options):
    tags = options.keywords.split(",") if options.keywords else None

    body = dict(
        snippet=dict(
            title=options.title,
            description=options.description,
            tags=tags,
            categoryId=options.category
        ),
        status=dict(
            privacyStatus=options.privacyStatus
        )
    )

    insert_request = youtube.videos().insert(
        part=",".join(body.keys()),
        body=body,
        media_body=MediaFileUpload(options.file, chunksize=-1, resumable=True)
    )

    resumable_upload(insert_request)

def resumable_upload(insert_request):
    response = None
    retry = 0

    while response is None:
        try:
            print("Uploading file...")
            status, response = insert_request.next_chunk()

            if response is not None and 'id' in response:
                print(f"Video id '{response['id']}' was successfully uploaded.")
                return
        except HttpError as e:
            if e.resp.status in RETRIABLE_STATUS_CODES:
                error = f"A retriable HTTP error {e.resp.status} occurred: {e.content}"
            else:
                raise
        except RETRIABLE_EXCEPTIONS as e:
            error = f"A retriable error occurred: {e}"

        if retry > MAX_RETRIES:
            sys.exit("No longer attempting to retry.")

        retry += 1
        max_sleep = 2 ** retry
        sleep_seconds = random.random() * max_sleep
        print(f"Sleeping {sleep_seconds} seconds and then retrying...")
        time.sleep(sleep_seconds)

if __name__ == "__main__":
    argparser.add_argument("--file", required=True, help="Video file to upload")
    argparser.add_argument("--title", help="Video title", default="Test Title")
    argparser.add_argument("--description", help="Video description", default="Test Description")
    argparser.add_argument("--category", default="22", help="Numeric video category")
    argparser.add_argument("--keywords", help="Video keywords, comma separated", default="")
    argparser.add_argument("--privacyStatus", choices=VALID_PRIVACY_STATUSES, default="public", help="Video privacy status.")

    args = argparser.parse_args()

    if not os.path.exists(args.file):
        sys.exit("Please specify a valid file using the --file= parameter.")

    youtube = get_authenticated_service(args)
    initialize_upload(youtube, args)
```

---

### Step 5: Execute the Script

Run the script from your terminal, providing the required arguments:
```bash
python upload_video.py \
  --file="path/to/your/video.mp4" \
  --title="Your Video Title" \
  --description="Your Video Description" \
  --keywords="keyword1,keyword2" \
  --category="22" \
  --privacyStatus="private"
```

Replace the placeholder values with the appropriate information.

---

### Step 6: Troubleshooting
- Ensure the `client_secrets.json` file is correctly configured and present in the same directory as the script.
- Check the console for detailed error messages if the upload fails.
- Verify that the YouTube Data API is enabled in your Google Cloud project.

---

### Conclusion
By following this guide, you can upload videos to YouTube programmatically using Python. Customize the script further to meet your specific needs, such as automating uploads or integrating it into a larger application.

Happy coding!
