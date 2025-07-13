
# 🖼️ Serverless Image Processing with S3, Lambda, and DynamoDB

This project demonstrates a serverless architecture using AWS Lambda to process images uploaded to an S3 bucket. Upon upload, the image is resized, stored in a separate destination S3 bucket, and a log record is added to a DynamoDB table.

---

## 📊 Architecture Diagram

> *(Include a visual diagram here if used in GitHub or documentation platform)*

---

## 🛠️ Components

- **S3 Source Bucket**: Where original images are uploaded.
- **Lambda Function**: Triggered by S3 event, resizes the image, uploads to destination, and logs to DynamoDB.
- **S3 Destination Bucket**: Stores resized images.
- **DynamoDB Table**: Logs metadata for each processed image.

---

## ✅ Prerequisites

- AWS Account
- IAM Role with permissions for:
  - `s3:GetObject`, `s3:PutObject`
  - `dynamodb:PutItem`
  - `logs:*` (optional for debugging with CloudWatch)

---

## 📂 Setup Instructions (Manual)

### 1. Create Two S3 Buckets
- Source Bucket: Example: `source-bucket-yourname`
- Destination Bucket: Example: `dest-bucket-yourname`

### 2. Create DynamoDB Table
- Table Name: `ImageProcessingLog`
- Partition Key: `ImageKey` (String)

### 3. Create Lambda Function
- Runtime: Python 3.11
- Trigger: S3 `ObjectCreated` event from the source bucket
- Attach an IAM role with access to S3 and DynamoDB

### Use this code:

```python
import boto3
import os
from PIL import Image
import io
from datetime import datetime

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
DEST_BUCKET = 'dest-bucket-yourname'
DDB_TABLE = 'ImageProcessingLog'

def get_image_format(key):
    extension = key.lower().split('.')[-1]
    return {
        'jpg': 'JPEG',
        'jpeg': 'JPEG',
        'png': 'PNG',
        'gif': 'GIF',
        'bmp': 'BMP',
        'tiff': 'TIFF',
        'webp': 'WEBP'
    }.get(extension, 'JPEG')

def lambda_handler(event, context):
    table = dynamodb.Table(DDB_TABLE)

    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']

        response = s3.get_object(Bucket=bucket, Key=key)
        image_content = response['Body'].read()

        image = Image.open(io.BytesIO(image_content))
        image = image.resize((300, 300))

        buffer = io.BytesIO()
        image_format = get_image_format(key)
        image.save(buffer, format=image_format)
        buffer.seek(0)

        s3.put_object(
            Bucket=DEST_BUCKET,
            Key=key,
            Body=buffer,
            ContentType=response['ContentType']
        )

        table.put_item(
            Item={
                'ImageKey': key,
                'SourceBucket': bucket,
                'DestinationBucket': DEST_BUCKET,
                'ProcessedAt': datetime.utcnow().isoformat(),
                'ImageSize': f"{image.width}x{image.height}",
                'ImageFormat': image_format
            }
        )

    return {
        'statusCode': 200,
        'body': 'Image processed and logged successfully.'
    }
```

---

## 🧪 Test the System

1. Upload a supported image to your source S3 bucket.
2. Confirm it appears in the destination bucket (resized).
3. Check DynamoDB table for a new log record.

---

## 📌 Notes

- ✅ Supports image types: JPG, PNG, GIF, BMP, TIFF, WEBP
- ✅ Make sure your Lambda has enough timeout and memory for larger images
- ✅ Resize dimensions can be customized in the script
