# BharatContent AI - Design Document

## Amazon AI for Bharat Hackathon 2026

---

## 1. Design Overview

### 1.1 Architecture Philosophy
BharatContent AI follows a serverless, microservices-based architecture leveraging AWS managed services. The design prioritizes:

- **Voice-First Experience:** Minimal typing, maximum voice interaction
- **Regional Language Support:** Native support for 8+ Indian languages
- **Real-Time Processing:** Sub-10-second end-to-end content generation
- **Scalability:** Serverless architecture for automatic scaling
- **Security:** End-to-end encryption and AWS security best practices

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Interface Layer                          │
│         (Next.js - Mobile-First Web Application)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS/TLS 1.3
┌────────────────────────▼────────────────────────────────────────┐
│                  API Gateway + AWS WAF                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼────────┐ ┌───▼──────┐ ┌─────▼──────────┐
│  Authentication │ │  Content │ │   Analytics    │
│     Service     │ │  Service │ │    Service     │
│  (AWS Lambda)   │ │ (Lambda) │ │   (Lambda)     │
└────────┬────────┘ └───┬──────┘ └─────┬──────────┘
         │              │               │
┌────────▼──────────────▼───────────────▼──────────┐
│           AWS AI/ML Services Layer                │
│  Transcribe | Bedrock | Comprehend | Translate   │
│              SageMaker (Prediction)               │
└───────────────────────┬───────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────┐
│         Data & Storage Layer                      │
│  DynamoDB | S3 | CloudWatch                       │
└───────────────────────────────────────────────────┘
```

### 1.3 Technology Stack

**Frontend:** Next.js 14+, Tailwind CSS, Zustand, Recharts  
**Backend:** AWS Lambda (Python 3.12 / Node.js 20.x), API Gateway  
**AI/ML:** Transcribe, Bedrock, Comprehend, Translate, SageMaker  
**Data:** DynamoDB, S3, CloudWatch  
**Security:** AWS WAF, Cognito, Secrets Manager

---

## 2. System Architecture

### 2.1 Frontend Architecture

#### Component Structure
```
src/
├── app/
│   ├── (auth)/              # Login, Register, Verify
│   ├── (dashboard)/         # Create, History, Analytics, Profile
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── voice/               # VoiceRecorder, AudioVisualizer
│   ├── content/             # ContentGenerator, PlatformSelector
│   ├── analytics/           # DashboardChart, PerformanceMetrics
│   └── common/              # Button, Input, Modal
├── lib/
│   ├── api/                 # API client functions
│   ├── hooks/               # Custom React hooks
│   ├── store/               # Zustand stores
│   └── utils/
└── types/
```

#### Key Components

**VoiceRecorder:**
- Handles microphone permissions
- Records audio using MediaRecorder API
- Real-time waveform visualization
- Uploads to S3 via pre-signed URL

**ContentGenerator:**
- Displays transcribed text for editing
- Platform selection
- Tone adjustment controls
- Real-time content generation
- Engagement score visualization

**Analytics Dashboard:**
- Content history with filters
- Engagement trends chart
- Platform performance comparison
- Growth insights

#### State Management (Zustand)

```typescript
// authStore.ts
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
}

// contentStore.ts
interface ContentState {
  currentContent: GeneratedContent | null;
  contentHistory: Content[];
  isGenerating: boolean;
  generateContent: (input: ContentInput) => Promise<void>;
}
```

---

### 2.2 Backend Architecture

#### Lambda Function Structure

```
backend/
├── functions/
│   ├── auth/                # register, login, verify-otp
│   ├── voice/               # upload, transcribe
│   ├── content/             # generate, optimize, translate
│   ├── analytics/           # dashboard, trends
│   └── prediction/          # engagement, timing
├── layers/
│   ├── common/              # Shared utilities
│   └── aws-sdk/
└── shared/
    ├── models/
    ├── schemas/
    └── constants/
```

#### Lambda Handler Pattern

```python
import json
import boto3
from lib.auth import verify_token
from lib.db import DynamoDBClient
from lib.utils import create_response, handle_error

# Initialize clients outside handler
dynamodb = DynamoDBClient()
s3_client = boto3.client('s3')

def handler(event, context):
    try:
        # 1. Extract and validate input
        body = json.loads(event.get('body', '{}'))
        headers = event.get('headers', {})
        
        # 2. Authenticate user
        user_id = verify_token(headers.get('Authorization'))
        
        # 3. Validate input
        validated_input = validate_input(body)
        
        # 4. Business logic
        result = process_request(user_id, validated_input)
        
        # 5. Return response
        return create_response(200, result)
        
    except ValueError as e:
        return handle_error(400, str(e))
    except Exception as e:
        return handle_error(500, "Internal server error")
```

#### API Gateway Structure

```yaml
/api/v1:
  /auth:
    /register: POST (Lambda: auth-register)
    /login: POST (Lambda: auth-login)
    /verify-otp: POST (Lambda: auth-verify-otp)
  
  /voice:
    /upload: POST (Lambda: voice-upload)
    /transcribe/{id}: GET (Lambda: voice-transcribe)
  
  /content:
    /generate: POST (Lambda: content-generate)
    /history: GET (Lambda: content-history)
  
  /analytics:
    /dashboard: GET (Lambda: analytics-dashboard)
```

---

### 2.3 Data Layer Design

#### DynamoDB Tables

**Users Table:**
```
Table: bharatcontent-users
Partition Key: user_id (String)

Attributes:
- user_id, phone_number, email, name
- preferred_language, location, business_type
- created_at, updated_at, is_active

GSI: phone_number-index, email-index
```

**Content Table:**
```
Table: bharatcontent-content
Partition Key: user_id (String)
Sort Key: created_at (Number)

Attributes:
- content_id, transcription, language, platform
- generated_content (caption, hashtags, CTA)
- engagement_score, sentiment

GSI: content_id-index
```

**Analytics Table:**
```
Table: bharatcontent-analytics
Partition Key: user_id (String)
Sort Key: date (String)

Attributes:
- metrics (content_generated, avg_engagement_score)
- best_performing_content_id
- recommendations
```

#### S3 Bucket Structure

```
bharatcontent-voice-recordings/
└── {user_id}/{year}/{month}/{content_id}.wav

bharatcontent-static-assets/
└── images/, icons/, fonts/
```

---

## 3. AI/ML Service Integration

### 3.1 Amazon Transcribe

```python
import boto3
import uuid

transcribe_client = boto3.client('transcribe')

def transcribe_audio(audio_file_key: str, language_code: str) -> str:
    """Transcribe audio using Amazon Transcribe"""
    job_name = f"transcribe-{uuid.uuid4()}"
    
    transcribe_client.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={'MediaFileUri': f's3://bucket/{audio_file_key}'},
        MediaFormat='wav',
        LanguageCode=language_code,
        Settings={'ShowSpeakerLabels': False}
    )
    
    return job_name

# Supported Languages
LANGUAGES = {
    'hi-IN': 'Hindi',
    'en-IN': 'English',
    'ta-IN': 'Tamil',
    'te-IN': 'Telugu',
    'mr-IN': 'Marathi',
    'bn-IN': 'Bengali',
    'gu-IN': 'Gujarati',
    'kn-IN': 'Kannada'
}
```

### 3.2 Amazon Bedrock

```python
import boto3
import json

bedrock_runtime = boto3.client('bedrock-runtime')

def generate_content(
    transcription: str,
    platform: str,
    language: str,
    tone: str = 'professional'
) -> dict:
    """Generate content using Amazon Bedrock"""
    
    prompt = build_prompt(transcription, platform, language, tone)
    
    response = bedrock_runtime.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 1024,
            'messages': [{'role': 'user', 'content': prompt}],
            'temperature': 0.7
        })
    )
    
    result = json.loads(response['body'].read())
    return parse_generated_content(result['content'][0]['text'])

def build_prompt(transcription, platform, language, tone):
    """Build platform-specific prompt"""
    return f"""Create {platform} content in {language}.
    
Input: "{transcription}"
Tone: {tone}

Requirements:
- Platform-specific format
- Culturally relevant for Indian audience
- Include hashtags and call-to-action
- Use appropriate emojis

Output JSON:
{{"caption": "...", "hashtags": [...], "call_to_action": "..."}}
"""
```

### 3.3 Amazon Comprehend

```python
comprehend_client = boto3.client('comprehend')

def analyze_sentiment(text: str, language_code: str) -> dict:
    """Analyze sentiment and emotions"""
    
    # Detect sentiment
    sentiment = comprehend_client.detect_sentiment(
        Text=text,
        LanguageCode=language_code
    )
    
    # Detect key phrases
    phrases = comprehend_client.detect_key_phrases(
        Text=text,
        LanguageCode=language_code
    )
    
    return {
        'sentiment': sentiment['Sentiment'],
        'scores': sentiment['SentimentScore'],
        'key_phrases': [p['Text'] for p in phrases['KeyPhrases']],
        'emotion_scores': {
            'trust': sentiment['SentimentScore']['Positive'] * 0.8,
            'excitement': sentiment['SentimentScore']['Positive'] * 0.9,
            'urgency': sentiment['SentimentScore']['Negative'] * 0.3
        }
    }

def check_content_safety(text: str) -> dict:
    """Check content for safety issues"""
    sentiment = comprehend_client.detect_sentiment(
        Text=text,
        LanguageCode='en'
    )
    
    is_safe = sentiment['SentimentScore']['Negative'] < 0.8
    
    return {
        'is_safe': is_safe,
        'safety_score': 1 - sentiment['SentimentScore']['Negative'],
        'issues': [] if is_safe else ['high_negativity']
    }
```

### 3.4 Amazon Translate

```python
translate_client = boto3.client('translate')

def translate_content(
    text: str,
    source_language: str,
    target_languages: list
) -> dict:
    """Translate content to multiple languages"""
    translations = {}
    
    for target_lang in target_languages:
        if target_lang == source_language:
            translations[target_lang] = text
            continue
        
        response = translate_client.translate_text(
            Text=text,
            SourceLanguageCode=source_language,
            TargetLanguageCode=target_lang
        )
        
        translations[target_lang] = response['TranslatedText']
    
    return translations
```

### 3.5 Amazon SageMaker - Engagement Prediction

```python
import boto3
import json

sagemaker_runtime = boto3.client('sagemaker-runtime')

def predict_engagement(content_features: dict) -> dict:
    """Predict engagement score"""
    
    features = [
        content_features['caption_length'],
        content_features['hashtag_count'],
        content_features['emoji_count'],
        content_features['sentiment_score'],
        content_features['has_cta'],
        content_features['posting_hour'],
        content_features['platform_code']
    ]
    
    response = sagemaker_runtime.invoke_endpoint(
        EndpointName='engagement-predictor',
        ContentType='application/json',
        Body=json.dumps({'features': features})
    )
    
    result = json.loads(response['Body'].read())
    score = result['predictions'][0]
    
    return {
        'engagement_score': round(score, 1),
        'category': categorize_score(score),
        'suggestions': suggest_improvements(content_features)
    }

def categorize_score(score: float) -> str:
    if score >= 86: return 'Excellent'
    elif score >= 71: return 'Good'
    elif score >= 41: return 'Average'
    else: return 'Poor'

def suggest_improvements(features: dict) -> list:
    suggestions = []
    if features['hashtag_count'] < 15:
        suggestions.append("Add more hashtags")
    if features['emoji_count'] < 3:
        suggestions.append("Add 2-3 emojis")
    if not features['has_cta']:
        suggestions.append("Include call-to-action")
    return suggestions
```

---

## 4. Security Architecture

### 4.1 Authentication & Authorization

```python
import jwt
import time

SECRET_KEY = 'your-secret-key'
ALGORITHM = 'HS256'

def create_access_token(user_id: str) -> str:
    """Create JWT access token"""
    payload = {
        'user_id': user_id,
        'exp': int(time.time()) + (24 * 60 * 60),  # 24 hours
        'iat': int(time.time()),
        'type': 'access'
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str) -> dict:
    """Verify JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise PermissionError('Token expired')
    except jwt.InvalidTokenError:
        raise PermissionError('Invalid token')
```

### 4.2 Data Encryption

```python
# DynamoDB encryption
dynamodb_config = {
    'SSESpecification': {
        'Enabled': True,
        'SSEType': 'KMS'
    }
}

# S3 encryption
s3_encryption = {
    'ServerSideEncryptionConfiguration': [{
        'Rules': [{
            'ApplyServerSideEncryptionByDefault': {
                'SSEAlgorithm': 'AES256'
            }
        }]
    }]
}
```

### 4.3 Rate Limiting

```python
def check_rate_limit(user_id: str, endpoint: str) -> bool:
    """Check rate limit"""
    redis_client = get_redis_client()
    key = f"rate_limit:{user_id}:{endpoint}"
    
    current_count = redis_client.incr(key)
    if current_count == 1:
        redis_client.expire(key, 60)  # 1 minute window
    
    return current_count <= 100  # 100 requests per minute
```

---

## 5. Performance Optimization

### 5.1 Caching Strategy

```python
import redis
import json

redis_client = redis.Redis(host='cache.amazonaws.com', port=6379)

def cache_result(ttl: int = 300):
    """Decorator for caching"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            cache_key = f"{func.__name__}:{json.dumps(args)}"
            
            # Check cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Store in cache
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

@cache_result(ttl=600)
def get_user_analytics(user_id: str) -> dict:
    return fetch_from_db(user_id)
```

### 5.2 Lambda Optimization

```python
# Initialize clients outside handler (reused across invocations)
import boto3

dynamodb = boto3.resource('dynamodb')
s3_client = boto3.client('s3')
bedrock_client = boto3.client('bedrock-runtime')

def handler(event, context):
    # Handler uses pre-initialized clients
    pass
```

### 5.3 Frontend Optimization

```typescript
// Next.js dynamic imports
import dynamic from 'next/dynamic';

const AnalyticsDashboard = dynamic(
  () => import('@/components/analytics/Dashboard'),
  { loading: () => <LoadingSpinner /> }
);

const VoiceRecorder = dynamic(
  () => import('@/components/voice/Recorder'),
  { ssr: false }  // Client-side only
);
```

---

## 6. Monitoring & Observability

### 6.1 CloudWatch Metrics

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def publish_metric(metric_name: str, value: float, dimensions: dict):
    """Publish custom metric"""
    cloudwatch.put_metric_data(
        Namespace='BharatContentAI',
        MetricData=[{
            'MetricName': metric_name,
            'Value': value,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': k, 'Value': v}
                for k, v in dimensions.items()
            ]
        }]
    )
```

### 6.2 Structured Logging

```python
import json
import logging

class StructuredLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
    
    def info(self, message: str, **kwargs):
        log_entry = {
            'level': 'INFO',
            'message': message,
            **kwargs
        }
        self.logger.info(json.dumps(log_entry))
```

---

## 7. Deployment Architecture

### 7.1 AWS SAM Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 512

Resources:
  BharatContentAPI:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Cors:
        AllowMethods: "'*'"
        AllowHeaders: "'*'"
        AllowOrigin: "'*'"

  ContentGenerateFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: functions/content/generate/
      Handler: handler.lambda_handler
      Timeout: 60
      MemorySize: 1024
      Events:
        GenerateContent:
          Type: Api
          Properties:
            Path: /api/v1/content/generate
            Method: POST

  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: bharatcontent-users
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: user_id
          AttributeType: S
      KeySchema:
        - AttributeName: user_id
          KeyType: HASH
```

### 7.2 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      
      - name: SAM Build
        run: sam build --use-container
      
      - name: SAM Deploy
        run: sam deploy --no-confirm-changeset
```

---

## 8. Error Handling

### 8.1 Error Classification

```python
class BharatContentError(Exception):
    def __init__(self, message: str, error_code: str, status_code: int):
        self.message = message
        self.error_code = error_code
        self.status_code = status_code

class ValidationError(BharatContentError):
    def __init__(self, message: str):
        super().__init__(message, 'VALIDATION_ERROR', 400)

class AuthenticationError(BharatContentError):
    def __init__(self):
        super().__init__('Authentication failed', 'AUTH_ERROR', 401)

class RateLimitError(BharatContentError):
    def __init__(self):
        super().__init__('Rate limit exceeded', 'RATE_LIMIT', 429)
```

### 8.2 Retry Logic

```python
import time
from functools import wraps

def retry_with_backoff(max_retries: int = 3):
    """Retry with exponential backoff"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if i == max_retries - 1:
                        raise
                    time.sleep(2 ** i)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3)
def call_bedrock_api(prompt: str):
    return bedrock_client.invoke_model(...)
```

---

## 9. Testing Strategy

### 9.1 Unit Testing

```python
import pytest
from unittest.mock import Mock, patch

def test_generate_content():
    """Test content generation"""
    result = generate_content(
        transcription='Test product',
        platform='instagram',
        language='en'
    )
    
    assert 'caption' in result
    assert 'hashtags' in result
    assert len(result['hashtags']) > 0
```

### 9.2 Integration Testing

```python
import requests

def test_content_generation_flow(api_endpoint, auth_token):
    """Test end-to-end flow"""
    response = requests.post(
        f'{api_endpoint}/content/generate',
        headers={'Authorization': f'Bearer {auth_token}'},
        json={
            'transcription': 'Test',
            'platform': 'instagram',
            'language': 'en'
        }
    )
    
    assert response.status_code == 200
    assert 'caption' in response.json()
```

---

## 10. Cost Optimization

### 10.1 Monthly Cost Estimates (10K users)

```
Lambda: $150
DynamoDB: $25
S3: $5
Transcribe: $200
Bedrock: $300
Comprehend: $50
Translate: $75
SageMaker: $100
API Gateway: $35
Data Transfer: $50

Total: ~$990/month
Cost per user: $0.099
```

### 10.2 Optimization Strategies

- Batch processing to reduce API calls
- Caching to avoid repeated AI service calls
- S3 lifecycle policies (delete voice files after 1 day)
- Use spot instances for batch jobs
- DynamoDB on-demand pricing

---

## 11. Design Decisions

### Key Decisions

1. **Serverless Architecture**
   - Rationale: Pay-per-use, auto-scaling, no server management
   - Trade-off: Cold start latency (mitigated with provisioned concurrency)

2. **DynamoDB over RDS**
   - Rationale: Better scalability, serverless pricing
   - Trade-off: Limited query flexibility (mitigated with GSIs)

3. **Amazon Bedrock over Self-Hosted**
   - Rationale: No infrastructure, latest models, built-in safety
   - Trade-off: Higher per-request cost (acceptable for MVP)

4. **Voice File Deletion**
   - Rationale: Privacy, cost savings, compliance
   - Trade-off: Cannot re-process original audio

---

## 12. Conclusion

This design provides a comprehensive blueprint for BharatContent AI using AWS serverless architecture. The system is scalable, secure, and optimized for the hackathon timeline while maintaining production-ready quality.

### Next Steps:
1. Review and approve design
2. Create implementation tasks
3. Set up AWS infrastructure
4. Implement core features
5. Deploy MVP for demo

---

**Document Version:** 1.0  
**Last Updated:** February 12, 2026  
**Status:** Ready for Implementation

