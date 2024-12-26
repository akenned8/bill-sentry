# Bill-Sentry

## System Overview
The system will be built as a modern cloud-native application with these key components:
- React-based frontend dashboard
- Event-driven serverless backend for file processing
- REST API for dashboard data and user management
- Secure file storage
- Document processing pipeline
- Results database and analytics engine

## Core Components

### Frontend Architecture
- **Framework**: React with TypeScript
- **State Management**: Redux Toolkit for global state
- **UI Components**: Tailwind CSS with shadcn/ui
- **File Upload**: React Dropzone for drag-and-drop functionality
- **Visualization**: Recharts for dashboard metrics
- **Authentication**: AWS Cognito with JWT tokens

### Backend Services

#### API Layer (AWS ECS with Fargate)
- Fast-responding REST API built with FastAPI
- Containerized for scalability and consistency
- Endpoints for:
  - User management
  - Dashboard metrics
  - File upload URLs (pre-signed S3 URLs)
  - Verification results
  - Historical data access

#### File Processing Pipeline

1. **Upload Flow**
   - Frontend requests pre-signed S3 URL from API
   - Direct browser-to-S3 upload using pre-signed URL
   - S3 event triggers Lambda notification
   - Files stored in separate buckets based on type (PDF/CSV)

2. **PDF Processing Pipeline**
   - Lambda trigger from S3 PDF upload
   - PDF extraction using Python's `pdfplumber` library
   - OCR capability using AWS Textract for poor quality scans
   - Structured data extraction with custom regex patterns
   - Results stored in JSON format in processing bucket

3. **CSV Processing Pipeline**
   - Lambda trigger from S3 CSV upload
   - Validation of CSV structure and required fields
   - Pandas for data normalization
   - Results stored in processing bucket

4. **Verification Engine** (ECS Service)
   - Watches processing bucket for completed extractions
   - Matches PDF bill data against CSV source-of-truth
   - Identifies discrepancies using configurable rules
   - Generates detailed comparison reports
   - Stores results in RDS

### Data Storage

1. **Amazon RDS (PostgreSQL)**
   - User accounts and preferences
   - Processed results and discrepancies
   - Historical analysis data
   - Audit logs

2. **Amazon S3**
   - Raw file storage (PDFs, CSVs)
   - Processed results
   - Temporary processing files
   - Archive bucket for historical files

3. **ElastiCache (Redis)**
   - API response caching
   - Session management
   - Rate limiting

### Security & Compliance

- AWS WAF for API protection
- VPC with private subnets for processing
- KMS encryption for sensitive data
- IAM roles with least privilege
- Audit logging to CloudWatch

## Detailed Process Flows

### File Upload Flow

1. **Frontend Preparation**
```typescript
// Frontend code snippet
const uploadFile = async (file: File) => {
  // Get pre-signed URL from API
  const uploadUrl = await api.getUploadUrl(file.name);
  
  // Upload directly to S3
  await axios.put(uploadUrl, file, {
    headers: {
      'Content-Type': file.type,
      'x-amz-meta-userId': currentUser.id,
    }
  });
};
```

2. **Backend Pre-signed URL Generation**
```python
# FastAPI endpoint
@router.post("/upload-url")
async def generate_upload_url(
    filename: str,
    file_type: FileType,
    current_user: User = Depends(get_current_user)
):
    bucket = PDF_BUCKET if file_type == FileType.PDF else CSV_BUCKET
    key = f"{current_user.id}/{uuid4()}/{filename}"
    
    url = s3_client.generate_presigned_url(
        ClientMethod='put_object',
        Params={
            'Bucket': bucket,
            'Key': key,
            'ContentType': file_type.mime_type,
        },
        ExpiresIn=3600
    )
    return {"uploadUrl": url, "key": key}
```

3. **Processing Pipeline**
```python
# Lambda handler for PDF processing
def handle_pdf_upload(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Extract text from PDF
    text = extract_pdf_content(bucket, key)
    
    # Parse into structured data
    structured_data = parse_bill_data(text)
    
    # Store results
    store_structured_data(structured_data, key)
    
    # Trigger verification if matching CSV exists
    trigger_verification(key)
```

### Verification Engine

The verification engine runs as a containerized service that:
1. Maintains state of processed files
2. Matches PDF bills with corresponding CSV data
3. Applies verification rules:
   - Price matching
   - Quantity verification
   - Date alignment
   - Tax calculation validation
4. Generates detailed reports with:
   - Line-item discrepancies
   - Total amount differences
   - Suggested corrections
   - Confidence scores

## Dashboard Structure

1. **Main Metrics**
   - Total processed bills
   - Total discrepancies found
   - Potential savings identified
   - Processing status

2. **File Management**
   - Upload interface
   - File processing status
   - Historical file browser

3. **Analysis Views**
   - Discrepancy details
   - Trend analysis
   - Vendor performance
   - Cost impact visualization

## Scaling Considerations

- Auto-scaling groups for API containers
- Lambda concurrency management
- S3 lifecycle policies for cost optimization
- RDS read replicas for heavy query loads
- CloudFront for static asset delivery
- DynamoDB for real-time processing status

## Monitoring & Operations

- CloudWatch dashboards for system metrics
- X-Ray for distributed tracing
- SNS notifications for processing errors
- Regular backup policies
- Retention policies for each storage type
