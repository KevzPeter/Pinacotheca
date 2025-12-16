# Pinacotheca - Photo Index

![Homepage](./media/homepage.png)

An intelligent photo album web application that enables natural language search capabilities using AWS services including Lex, ElasticSearch, and Rekognition.

## Overview

Pinacotheca allows users to search their photo collection using natural language queries. The application automatically detects and indexes objects, people, actions, and landmarks in uploaded photos, making them searchable through an intuitive interface.

## Architecture

### Components

1. **Frontend (S3 Bucket B1)**

   - Static website hosting
   - Upload interface with custom label support
   - Search interface with natural language queries
   - Photo gallery display

2. **Backend Services**

   - **Lambda Function (LF1) - index-photos**: Indexes photos using Rekognition labels
   - **Lambda Function (LF2) - search-photos**: Processes search queries using Amazon Lex
   - **API Gateway**: RESTful API with two endpoints
   - **OpenSearch**: Photo metadata and label storage
   - **S3 Bucket (B2)**: Photo storage with PUT event triggers

3. **CI/CD Pipeline**

   - **CodePipeline (P1)**: Automated deployment for Lambda functions
   - **CodePipeline (P2)**: Automated deployment for frontend code

4. **Infrastructure as Code**
   - CloudFormation template for resource provisioning

## Features

### Natural Language Search

- Search photos using keywords: "trees", "dogs"
- Search using natural sentences: "show me trees and birds"
- Support for multiple keywords per query
- Automatic label detection via AWS Rekognition

### Custom Labels

- Add custom labels during photo upload
- Custom labels stored in S3 metadata (`x-amz-meta-customLabels`)
- Searchable alongside auto-detected labels

### Automated Indexing

- Automatic photo indexing on upload
- Rekognition integration for label detection
- ElasticSearch indexing for fast retrieval

## API Specification

### Endpoints

#### PUT /photos

Uploads photos directly to S3 bucket via API Gateway S3 Proxy.

**Headers:**

```
x-amz-meta-customLabels: "label1, label2, label3"
```

#### GET /search?q={query}

Searches photos using natural language queries.

**Parameters:**

- `q`: Search query (keywords or natural language)

**Response:**

```json
{
  "results": [
    {
      "objectKey": "photo.jpg",
      "bucket": "photo-bucket",
      "createdTimestamp": "2025-12-15T12:40:02",
      "labels": ["dog", "park", "person"]
    }
  ]
}
```

## ElasticSearch Schema

Photos are indexed in the "photos" index with the following structure:

```json
{
  "objectKey": "my-photo.jpg",
  "bucket": "my-photo-bucket",
  "createdTimestamp": "2018-11-05T12:40:02",
  "labels": ["person", "dog", "ball", "park"]
}
```

## Setup Instructions

### Deployment

#### Option 1: CloudFormation Template

```bash
aws cloudformation create-stack \
  --stack-name photo-album-stack \
  --template-body file://photo-album-stack.yml \
  --capabilities CAPABILITY_IAM
```

The CloudFormation template provisions:

- Lambda functions (LF1, LF2)
- API Gateway with endpoints
- S3 buckets (frontend and storage)
- IAM roles and policies

#### Option 2: Manual Setup

1. **Create ElasticSearch Domain**

   ```bash
   aws es create-elasticsearch-domain --domain-name photos
   ```

2. **Create S3 Buckets**

   - Frontend bucket (B1) with static website hosting
   - Photos bucket (B2) with PUT event trigger

3. **Deploy Lambda Functions**

   - Deploy LF1 (index-photos) with S3 PUT trigger
   - Deploy LF2 (search-photos) connected to API Gateway

4. **Configure Amazon Lex**

   - Create bot with SearchIntent
   - Add training utterances for keyword and sentence searches

5. **Setup API Gateway**

   - Create PUT /photos endpoint (S3 Proxy)
   - Create GET /search endpoint (Lambda proxy to LF2)
   - Generate and deploy API key
   - Generate SDK for frontend integration

6. **Configure CodePipeline**
   - Pipeline P1: Backend Lambda deployment
   - Pipeline P2: Frontend S3 deployment

### Environment Variables

**Lambda LF1 (index-photos):**

- `ES_ENDPOINT`: ElasticSearch domain endpoint
- `ES_INDEX`: "photos"

**Lambda LF2 (search-photos):**

- `ES_ENDPOINT`: ElasticSearch domain endpoint
- `LEX_BOT_NAME`: Name of Lex bot
- `LEX_BOT_ALIAS`: Bot alias

## Lambda Function Details

### LF1: index-photos

**Trigger:** S3 PUT event on photos bucket

**Process:**

1. Receives S3 PUT event
2. Calls Rekognition `detectLabels` on uploaded photo
3. Retrieves custom labels from S3 object metadata
4. Combines Rekognition labels with custom labels
5. Indexes photo metadata in ElasticSearch

### LF2: search-photos

**Trigger:** API Gateway GET /search request

**Process:**

1. Receives search query
2. Disambiguates query using Amazon Lex
3. Extracts keywords from Lex response
4. Queries ElasticSearch photos index
5. Returns matching photos

## Amazon Lex Configuration

**Bot Name:** PhotoSearchBot

**Intent:** SearchIntent

**Sample Utterances:**

- "show me {keyword}"
- "find {keyword}"
- "photos with {keyword}"
- "show me {keyword} and {keyword}"
- "{keyword}"

**Slots:**

- keyword1 (required)
- keyword2 (optional)

## Usage

### Uploading Photos

1. Navigate to the frontend URL
2. Click "Upload Photo"
3. Select image file
4. (Optional) Add custom labels (comma-separated)
5. Click "Submit"

### Searching Photos

1. Enter search query in search bar
   - Examples: "dogs", "show me cats and trees"
2. Press Enter or click Search
3. View results in photo gallery

## Testing

### Test Photo Upload and Indexing

```bash
# Upload test photo
aws s3 cp test-photo.jpg s3://photo-bucket/ \
  --metadata customLabels="test,sample"

# Check LF1 logs
aws logs tail /aws/lambda/index-photos --follow
```

### Test Search Functionality

```bash
# Test search endpoint
curl -X GET "https://api-gateway-url/search?q=dogs" \
  -H "x-api-key: YOUR_API_KEY"
```

## Project Structure

```
.
├── backend/
│   ├── lf1/
│   │   └── index.js          # index-photos Lambda
│   ├── lf2/
│   │   └── index.js          # search-photos Lambda
│   └── buildspec.yml         # CodeBuild spec for backend
├── frontend/
│   ├── index.html            # Main application page
│   ├── styles.css            # Styling
│   └── buildspec.yml         # CodeBuild spec for frontend
└── photo-album-stack.yml     # CloudFormation template
└── README.md
```

## Technologies Used

- **AWS Lambda**: Serverless compute for indexing and search
- **AWS Rekognition**: Image analysis and label detection
- **Amazon Lex**: Natural language understanding
- **Amazon ElasticSearch**: Photo metadata indexing and search
- **Amazon S3**: Photo and frontend hosting
- **API Gateway**: RESTful API endpoints
- **AWS CodePipeline**: CI/CD automation
- **AWS CloudFormation**: Infrastructure as Code
