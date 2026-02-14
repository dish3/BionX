# Design Document: BionX Healthcare Platform

## Overview

The BionX Healthcare Platform is a serverless, cloud-native healthcare queue management system built on AWS infrastructure. The architecture follows a microservices pattern with API Gateway routing requests to specialized Lambda functions, DynamoDB for data persistence, and integration with AWS Bedrock for AI capabilities, AWS Location Service for hospital discovery, and AWS Cognito for authentication.

The system is designed for high availability, scalability, and cost efficiency, targeting a monthly operational cost of ₹1,000–₹5,000 through serverless architecture and intelligent caching strategies.

### Key Design Principles

1. **Serverless-First**: Leverage AWS Lambda for compute to minimize costs and maximize scalability
2. **Event-Driven**: Use asynchronous messaging for notifications and queue updates
3. **Stateless Services**: All Lambda functions are stateless, storing state in DynamoDB
4. **Security by Default**: Implement authentication, authorization, and encryption at every layer
5. **Offline-First Mobile**: Cache critical data on mobile devices for offline access
6. **Cost-Conscious**: Implement caching, connection pooling, and resource optimization

## Architecture

### High-Level Architecture

```
┌─────────────────┐         ┌─────────────────┐
│   Mobile App    │         │    Web App      │
│ (React Native)  │         │    (React)      │
└────────┬────────┘         └────────┬────────┘
         │                           │
         └───────────┬───────────────┘
                     │ HTTPS
         ┌───────────▼────────────┐
         │   AWS API Gateway      │
         │  (REST + WebSocket)    │
         └───────────┬────────────┘
                     │
         ┌───────────▼────────────┐
         │   AWS Lambda Layer     │
         │  ┌──────────────────┐  │
         │  │ Auth Service     │  │
         │  │ Token Service    │  │
         │  │ Queue Service    │  │
         │  │ Hospital Service │  │
         │  │ AI Service       │  │
         │  │ Notification Svc │  │
         │  └──────────────────┘  │
         └───────────┬────────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼────┐    ┌─────▼──────┐   ┌────▼─────┐
│DynamoDB│    │AWS Bedrock │   │AWS       │
│        │    │(AI Model)  │   │Location  │
└────────┘    └────────────┘   └──────────┘
    │
┌───▼────────┐
│AWS Cognito │
│(Auth)      │
└────────────┘
    │
┌───▼────────┐
│AWS SNS/FCM │
│(Notifications)│
└────────────┘
```

### Component Interaction Flow

**Token Booking Flow:**
```
Mobile App → API Gateway → Token Lambda → DynamoDB (write token)
                                       → Queue Lambda (update queue)
                                       → SNS (send notification)
```

**Queue Status Updates:**
```
Staff Dashboard → API Gateway → Queue Lambda → DynamoDB (update)
                                             → WebSocket (broadcast to patients)
                                             → SNS (notify affected patients)
```

**AI Assistant Flow:**
```
Mobile App → API Gateway → AI Lambda → AWS Bedrock (generate response)
                                    → DynamoDB (log interaction)
```

## Components and Interfaces

### 1. API Gateway

**Responsibilities:**
- Route HTTP requests to appropriate Lambda functions
- Handle WebSocket connections for real-time updates
- Validate request format and authentication tokens
- Rate limiting and throttling

**REST Endpoints:**

```
POST   /auth/send-otp          - Send OTP to phone number
POST   /auth/verify-otp        - Verify OTP and create session
POST   /auth/refresh           - Refresh authentication token

GET    /hospitals              - List hospitals (with location filter)
GET    /hospitals/{id}         - Get hospital details
GET    /hospitals/{id}/queues  - Get hospital queue status

POST   /tokens                 - Book a new token
GET    /tokens/{id}            - Get token details
DELETE /tokens/{id}            - Cancel token

GET    /queue/{hospitalId}/{dept} - Get queue status
POST   /queue/control          - Pause/resume/delay queue (staff only)
PUT    /queue/advance          - Mark patient as served (staff only)

POST   /ai/chat                - Send message to AI assistant
GET    /ai/history             - Get conversation history

POST   /notifications/register - Register device for push notifications
```

**WebSocket Endpoints:**
```
/ws/queue/{queueId}  - Subscribe to queue updates
```

### 2. Authentication Service (Lambda)

**Responsibilities:**
- Generate and validate OTPs
- Create and manage user sessions
- Integrate with AWS Cognito for user management
- Handle staff authentication

**Interface:**

```typescript
interface AuthService {
  sendOTP(phoneNumber: string): Promise<{
    success: boolean;
    expiresAt: number;
  }>;
  
  verifyOTP(phoneNumber: string, otp: string): Promise<{
    success: boolean;
    token: string;
    refreshToken: string;
    userId: string;
  }>;
  
  refreshToken(refreshToken: string): Promise<{
    token: string;
    expiresAt: number;
  }>;
  
  validateToken(token: string): Promise<{
    valid: boolean;
    userId: string;
    role: 'patient' | 'staff';
  }>;
}
```

**Implementation Details:**
- Use AWS Cognito Custom Auth Flow for OTP
- Store OTPs in DynamoDB with 5-minute TTL
- JWT tokens with 24-hour expiration
- Refresh tokens with 30-day expiration
- Rate limit: 3 OTP requests per phone number per hour

### 3. Token Service (Lambda)

**Responsibilities:**
- Create digital tokens for patients
- Validate booking constraints (no duplicates)
- Calculate estimated wait times
- Handle token cancellations

**Interface:**

```typescript
interface TokenService {
  createToken(request: {
    userId: string;
    hospitalId: string;
    department: string;
    preferredTime?: string;
  }): Promise<{
    tokenId: string;
    tokenNumber: number;
    queuePosition: number;
    estimatedWaitTime: number;
  }>;
  
  getToken(tokenId: string): Promise<{
    tokenId: string;
    tokenNumber: number;
    status: 'waiting' | 'approaching' | 'ready' | 'served' | 'cancelled';
    queuePosition: number;
    estimatedWaitTime: number;
    hospitalId: string;
    department: string;
  }>;
  
  cancelToken(tokenId: string, userId: string): Promise<{
    success: boolean;
  }>;
  
  getUserTokens(userId: string): Promise<Token[]>;
}
```

**Implementation Details:**
- Generate sequential token numbers per queue per day
- Check for duplicate bookings before creation
- Trigger queue update after token creation
- Store tokens with composite key: `hospitalId#department#date`

### 4. Queue Service (Lambda)

**Responsibilities:**
- Manage queue state and progression
- Calculate queue positions and wait times
- Handle queue control operations (pause/resume/delay)
- Broadcast updates via WebSocket

**Interface:**

```typescript
interface QueueService {
  getQueueStatus(hospitalId: string, department: string): Promise<{
    queueId: string;
    status: 'active' | 'paused' | 'delayed';
    totalPatients: number;
    currentPosition: number;
    averageWaitTime: number;
    tokens: QueueToken[];
  }>;
  
  pauseQueue(queueId: string, staffId: string): Promise<{
    success: boolean;
  }>;
  
  resumeQueue(queueId: string, staffId: string): Promise<{
    success: boolean;
  }>;
  
  addDelay(queueId: string, delayMinutes: number, staffId: string): Promise<{
    success: boolean;
  }>;
  
  advanceQueue(queueId: string, tokenId: string, staffId: string): Promise<{
    success: boolean;
    nextToken?: string;
  }>;
  
  updatePosition(tokenId: string): Promise<{
    newPosition: number;
    estimatedWaitTime: number;
  }>;
}
```

**Implementation Details:**
- Use DynamoDB streams to trigger position updates
- Calculate wait time: `averageServiceTime * queuePosition`
- Broadcast updates to WebSocket subscribers
- Log all control operations with staff ID and timestamp
- Use optimistic locking for concurrent updates

### 5. Hospital Service (Lambda)

**Responsibilities:**
- Retrieve hospital information
- Search hospitals by location
- Manage hospital metadata
- Cache hospital data

**Interface:**

```typescript
interface HospitalService {
  searchHospitals(request: {
    latitude: number;
    longitude: number;
    radius?: number; // km, default 10
    emergency?: boolean;
  }): Promise<{
    hospitals: Hospital[];
  }>;
  
  getHospital(hospitalId: string): Promise<{
    id: string;
    name: string;
    address: string;
    phone: string[];
    departments: string[];
    location: {
      latitude: number;
      longitude: number;
    };
    emergencyServices: boolean;
    currentQueues: QueueSummary[];
  }>;
  
  getHospitalQueues(hospitalId: string): Promise<{
    queues: QueueSummary[];
  }>;
}
```

**Implementation Details:**
- Integrate with AWS Location Service for distance calculations
- Cache hospital data in DynamoDB with 24-hour TTL
- Use geohash for efficient location queries
- Return results sorted by distance
- Include real-time queue status in results

### 6. AI Assistant Service (Lambda)

**Responsibilities:**
- Process user queries and generate first-aid guidance
- Maintain conversation context
- Detect emergency situations
- Log interactions for compliance

**Interface:**

```typescript
interface AIService {
  sendMessage(request: {
    userId: string;
    conversationId: string;
    message: string;
    language: 'en' | 'hi';
  }): Promise<{
    response: string;
    isEmergency: boolean;
    conversationId: string;
  }>;
  
  getConversationHistory(conversationId: string): Promise<{
    messages: Message[];
  }>;
  
  endConversation(conversationId: string): Promise<{
    success: boolean;
  }>;
}
```

**Implementation Details:**
- Use AWS Bedrock with Claude or Llama model
- System prompt: "You are a medical first-aid assistant. Provide clear, actionable first-aid guidance. If the situation is life-threatening, immediately advise calling emergency services (911/108). Keep responses concise and in simple language."
- Store last 10 messages in DynamoDB for context
- Limit conversation to 10 exchanges to control costs
- Detect keywords: "bleeding", "unconscious", "chest pain", "breathing difficulty" → flag as emergency
- Log all interactions with timestamp and user ID

### 7. Notification Service (Lambda)

**Responsibilities:**
- Send push notifications via FCM and AWS SNS
- Handle notification delivery failures and retries
- Register and manage device tokens

**Interface:**

```typescript
interface NotificationService {
  registerDevice(userId: string, deviceToken: string, platform: 'ios' | 'android'): Promise<{
    success: boolean;
  }>;
  
  sendNotification(request: {
    userId: string;
    title: string;
    body: string;
    data?: Record<string, string>;
    priority: 'high' | 'normal';
  }): Promise<{
    success: boolean;
    messageId: string;
  }>;
  
  sendBulkNotification(userIds: string[], notification: Notification): Promise<{
    successCount: number;
    failureCount: number;
  }>;
}
```

**Implementation Details:**
- Use AWS SNS for iOS (APNS) and Android (FCM)
- Store device tokens in DynamoDB
- Retry failed deliveries up to 2 times with exponential backoff
- Remove invalid device tokens after 3 consecutive failures
- Support notification types: booking_confirmed, approaching_turn, your_turn, queue_paused, queue_delayed

## Data Models

### User

```typescript
interface User {
  userId: string;              // PK: USER#{userId}
  phoneNumber: string;         // GSI: phoneNumber
  role: 'patient' | 'staff';
  name?: string;
  language: 'en' | 'hi';
  hospitalId?: string;         // For staff users
  createdAt: number;
  lastLoginAt: number;
  deviceTokens: string[];
}
```

### Token

```typescript
interface Token {
  tokenId: string;             // PK: TOKEN#{tokenId}
  userId: string;              // GSI: userId
  hospitalId: string;
  department: string;
  queueId: string;             // GSI: queueId
  tokenNumber: number;
  status: 'waiting' | 'approaching' | 'ready' | 'served' | 'cancelled';
  queuePosition: number;
  estimatedWaitTime: number;   // minutes
  bookedAt: number;
  servedAt?: number;
  date: string;                // YYYY-MM-DD
}
```

### Queue

```typescript
interface Queue {
  queueId: string;             // PK: QUEUE#{hospitalId}#{department}#{date}
  hospitalId: string;
  department: string;
  date: string;                // YYYY-MM-DD
  status: 'active' | 'paused' | 'delayed';
  currentPosition: number;
  totalPatients: number;
  averageServiceTime: number;  // minutes
  delayMinutes: number;
  tokens: string[];            // Array of tokenIds in order
  createdAt: number;
  updatedAt: number;
}
```

### Hospital

```typescript
interface Hospital {
  hospitalId: string;          // PK: HOSPITAL#{hospitalId}
  name: string;
  address: string;
  city: string;
  state: string;
  phone: string[];
  location: {
    latitude: number;
    longitude: number;
    geohash: string;           // For location queries
  };
  departments: string[];
  emergencyServices: boolean;
  operatingHours: {
    open: string;              // HH:MM
    close: string;             // HH:MM
  };
  createdAt: number;
  updatedAt: number;
}
```

### AIConversation

```typescript
interface AIConversation {
  conversationId: string;      // PK: CONV#{conversationId}
  userId: string;              // GSI: userId
  messages: Message[];
  language: 'en' | 'hi';
  createdAt: number;
  lastMessageAt: number;
  ttl: number;                 // Auto-delete after 24 hours
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: number;
  isEmergency?: boolean;
}
```

### QueueControlLog

```typescript
interface QueueControlLog {
  logId: string;               // PK: LOG#{logId}
  queueId: string;             // GSI: queueId
  staffId: string;
  action: 'pause' | 'resume' | 'delay' | 'advance';
  details: Record<string, any>;
  timestamp: number;
}
```

### DynamoDB Table Design

**Single Table Design:**

```
Table: BionXPlatform

PK (Partition Key): Entity type + ID
SK (Sort Key): Additional identifier or timestamp
GSI1PK: Secondary access pattern
GSI1SK: Secondary sort key

Examples:
- User: PK=USER#{userId}, SK=PROFILE
- Token: PK=TOKEN#{tokenId}, SK=METADATA, GSI1PK=USER#{userId}, GSI1SK=BOOKED#{timestamp}
- Queue: PK=QUEUE#{hospitalId}#{dept}, SK=DATE#{date}
- Hospital: PK=HOSPITAL#{hospitalId}, SK=METADATA
- AI Conv: PK=CONV#{convId}, SK=METADATA, GSI1PK=USER#{userId}
```

**Indexes:**
- GSI1: For user-based queries (get all tokens for user)
- GSI2: For queue-based queries (get all tokens in queue)
- GSI3: For location-based queries (geohash)



## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property Reflection

After analyzing all acceptance criteria, I identified the following redundancies:
- Property 4.2 (approaching turn notification) is redundant with 3.3 (same requirement)
- Property 14.6 (conversation length limit) is redundant with 7.4 (same requirement)

These redundant properties have been consolidated to avoid duplicate testing.

### Authentication Properties

**Property 1: OTP Generation for Valid Phone Numbers**
*For any* valid phone number format, when requesting an OTP, the Auth_Service should successfully generate and send an OTP with a 5-minute expiration.
**Validates: Requirements 1.1**

**Property 2: Valid OTP Authentication**
*For any* valid OTP that has not expired, when verified with the correct phone number, the Auth_Service should authenticate the user and create a valid session token.
**Validates: Requirements 1.2**

**Property 3: Invalid OTP Rejection**
*For any* invalid OTP (wrong code, expired, or mismatched phone number), the Auth_Service should reject the authentication attempt and maintain retry count up to 3 attempts.
**Validates: Requirements 1.3**

**Property 4: Staff Authentication**
*For any* authorized staff member with valid credentials, the Auth_Service should authenticate them and grant access only to their assigned facility's data.
**Validates: Requirements 1.5**

### Token Booking Properties

**Property 5: Token Uniqueness**
*For any* set of token bookings, all generated token IDs should be unique across the entire system.
**Validates: Requirements 2.2**

**Property 6: Token Response Completeness**
*For any* successfully created token, the response should include token number, queue position, and estimated wait time.
**Validates: Requirements 2.3**

**Property 7: Duplicate Booking Prevention**
*For any* patient and hospital-department-date combination, attempting to create a second token should be rejected while the first token is still active (not served or cancelled).
**Validates: Requirements 2.4**

**Property 8: Booking Confirmation Notification**
*For any* successfully created token, a confirmation notification should be sent to the patient's registered device.
**Validates: Requirements 2.6, 4.1**

### Queue Management Properties

**Property 9: Queue Position Display**
*For any* active token, when retrieving token details, the response should include current queue position and estimated wait time.
**Validates: Requirements 3.1**

**Property 10: Approaching Turn Notification**
*For any* token where the queue position becomes 3 or less, an "approaching turn" notification should be triggered.
**Validates: Requirements 3.3, 4.2**

**Property 11: Wait Time Calculation**
*For any* queue, the estimated wait time for a token should equal the average service time multiplied by the queue position.
**Validates: Requirements 3.4**

**Property 12: Queue Status Propagation**
*For any* queue, when staff changes the status (pause/resume/delay), all tokens in that queue should reflect the updated status.
**Validates: Requirements 3.5**

**Property 13: Your Turn Notification**
*For any* token where the queue position becomes 1, a "your turn" notification should be triggered.
**Validates: Requirements 4.3**

**Property 14: Queue Control Notifications**
*For any* queue control action (pause, resume, delay), all patients with active tokens in that queue should receive a notification.
**Validates: Requirements 4.4, 9.4**

**Property 15: Notification Retry Logic**
*For any* failed notification delivery, the system should retry up to 2 times with exponential backoff before marking as failed.
**Validates: Requirements 4.6**

### Hospital Discovery Properties

**Property 16: Location-Based Search Radius**
*For any* user location and search radius, all returned hospitals should be within the specified radius (default 10km) and sorted by distance in ascending order.
**Validates: Requirements 5.1**

**Property 17: Hospital Result Completeness**
*For any* hospital in search results, the response should include name, distance, address, and contact number.
**Validates: Requirements 5.3**

**Property 18: Distance Calculation Accuracy**
*For any* two geographic coordinates, the calculated distance should match the haversine formula result within 1% tolerance.
**Validates: Requirements 5.4**

**Property 19: Hospital Detail Completeness**
*For any* hospital detail request, the response should include available departments and current queue status for each department.
**Validates: Requirements 5.5**

**Property 20: Call Event Logging**
*For any* hospital call initiation, an event log should be created with user ID, hospital ID, and timestamp.
**Validates: Requirements 6.3**

**Property 21: Multiple Contact Numbers Display**
*For any* hospital with multiple contact numbers, all numbers should be included in the response with their associated department labels.
**Validates: Requirements 6.4**

### AI Assistant Properties

**Property 22: Emergency Keyword Detection**
*For any* user message containing emergency keywords ("bleeding", "unconscious", "chest pain", "breathing difficulty"), the AI_Assistant should flag the response as emergency and include emergency contact numbers.
**Validates: Requirements 7.3**

**Property 23: Conversation Context Limit**
*For any* conversation, the system should maintain at most 10 message exchanges (20 messages total: 10 user + 10 assistant).
**Validates: Requirements 7.4, 14.6**

**Property 24: AI Interaction Logging**
*For any* AI message exchange, a log entry should be created with conversation ID, user ID, message content, and timestamp.
**Validates: Requirements 7.7**

### Staff Dashboard Properties

**Property 25: Facility-Scoped Queue Access**
*For any* hospital staff member, when retrieving queues, only queues belonging to their assigned facility should be returned.
**Validates: Requirements 8.1**

**Property 26: Queue Display Completeness**
*For any* queue viewed by staff, the response should include all token numbers, booking times, and current status for each patient.
**Validates: Requirements 8.2**

**Property 27: Patient Detail Access**
*For any* token selected by staff, the system should return detailed patient information including name, phone number, and booking details.
**Validates: Requirements 8.4**

**Property 28: Queue Statistics Calculation**
*For any* queue, the displayed statistics (total patients, average wait time, patients served) should accurately reflect the current queue state.
**Validates: Requirements 8.5**

### Queue Control Properties

**Property 29: Queue Pause Effect**
*For any* queue, when staff pauses it, the queue status should change to "paused" and no automatic queue advancement should occur.
**Validates: Requirements 9.1**

**Property 30: Queue Pause-Resume Round Trip**
*For any* queue, pausing and then immediately resuming should restore the queue to active status with the same queue positions.
**Validates: Requirements 9.2**

**Property 31: Delay Propagation**
*For any* queue with a delay added, all estimated wait times for tokens in that queue should increase by the delay amount.
**Validates: Requirements 9.3**

**Property 32: Queue Control Audit Logging**
*For any* queue control action, a log entry should be created with staff ID, action type, queue ID, and timestamp.
**Validates: Requirements 9.5**

**Property 33: Patient Served Advancement**
*For any* token marked as served, the token status should change to "served", it should be removed from the active queue, and the next patient's position should advance.
**Validates: Requirements 9.6**

### Emergency Features Properties

**Property 34: Emergency Hospital Limit**
*For any* emergency hospital search, the results should return at most 5 hospitals with emergency services enabled.
**Validates: Requirements 10.1**

**Property 35: Emergency Hospital Response Completeness**
*For any* emergency hospital in results, the response should include distance, estimated travel time, and contact numbers.
**Validates: Requirements 10.2**

**Property 36: Emergency Service Prioritization**
*For any* emergency hospital search results, hospitals with 24/7 emergency services should appear before hospitals without 24/7 services.
**Validates: Requirements 10.3**

### Data Consistency Properties

**Property 37: Write Retry Logic**
*For any* failed database write operation, the system should retry up to 3 times before returning an error to the user.
**Validates: Requirements 11.2**

**Property 38: Token-Queue Consistency**
*For any* token in a queue, the token's queue position should match its position in the queue's token array.
**Validates: Requirements 11.3**

**Property 39: Optimistic Locking for Concurrent Updates**
*For any* concurrent updates to the same queue, the system should detect conflicts and ensure only one update succeeds per version.
**Validates: Requirements 11.4**

### Security Properties

**Property 40: Authentication Token Validation**
*For any* API request to a protected endpoint without a valid authentication token, the request should be rejected with 401 Unauthorized.
**Validates: Requirements 13.3**

**Property 41: Role-Based Access Control**
*For any* hospital staff member attempting to access data from a different facility, the request should be rejected with 403 Forbidden.
**Validates: Requirements 13.4**

**Property 42: Data Access Audit Logging**
*For any* access to patient data (token details, queue information), an audit log should be created with accessor ID, resource ID, and timestamp.
**Validates: Requirements 13.7**

### Monitoring Properties

**Property 43: Lambda Invocation Logging**
*For any* Lambda function invocation, a log entry should be created in CloudWatch with function name, request ID, and execution time.
**Validates: Requirements 15.1**

**Property 44: Error Logging Completeness**
*For any* error that occurs, the error log should include user ID, request ID, error message, and stack trace.
**Validates: Requirements 15.2**

### Offline Capability Properties

**Property 45: Offline Token Display**
*For any* cached token, when the mobile app is offline, the token details should be displayed from cache with an offline indicator.
**Validates: Requirements 16.1**

**Property 46: Token Caching**
*For any* token retrieved while online, the token data should be cached locally on the mobile device.
**Validates: Requirements 16.2**

**Property 47: Connectivity Restoration Sync**
*For any* mobile app that regains connectivity after being offline, the app should automatically sync cached data with the server.
**Validates: Requirements 16.4**

**Property 48: Offline Hospital Information**
*For any* hospital data retrieved while online, the hospital contact information should be cached and accessible offline.
**Validates: Requirements 16.5**

### Multi-Language Properties

**Property 49: AI Language Response**
*For any* AI assistant request with a specified language preference, the response should be in the requested language (English or Hindi).
**Validates: Requirements 17.3**

**Property 50: Language Preference Persistence**
*For any* user who sets a language preference, the preference should be saved and restored in subsequent sessions.
**Validates: Requirements 17.4**

**Property 51: Translation Fallback**
*For any* UI text or content where translation is unavailable in the user's selected language, the system should display the English version.
**Validates: Requirements 17.5**

### Analytics Properties

**Property 52: Daily Statistics Calculation**
*For any* date, the dashboard statistics (total patients served, average wait time, peak hours) should accurately reflect all tokens for that date.
**Validates: Requirements 18.1**

**Property 53: Custom Date Range Reports**
*For any* valid date range, the system should generate a report including all tokens and queue activities within that range.
**Validates: Requirements 18.2**

**Property 54: Queue Abandonment Rate**
*For any* queue and date, the abandonment rate should equal (tokens marked as no-show / total tokens booked) × 100.
**Validates: Requirements 18.3**

**Property 55: Trend Calculation**
*For any* metric and time period, the trend should show the change in metric values over consecutive time intervals.
**Validates: Requirements 18.4**

**Property 56: CSV Export Format**
*For any* report exported to CSV, the file should be valid CSV format with proper headers and escaped values.
**Validates: Requirements 18.5**

## Error Handling

### Error Categories

1. **Client Errors (4xx)**
   - 400 Bad Request: Invalid input format or missing required fields
   - 401 Unauthorized: Missing or invalid authentication token
   - 403 Forbidden: Insufficient permissions for requested resource
   - 404 Not Found: Requested resource does not exist
   - 409 Conflict: Duplicate booking or concurrent update conflict
   - 429 Too Many Requests: Rate limit exceeded

2. **Server Errors (5xx)**
   - 500 Internal Server Error: Unexpected server error
   - 502 Bad Gateway: Downstream service failure (AWS Bedrock, Location Service)
   - 503 Service Unavailable: Service temporarily unavailable
   - 504 Gateway Timeout: Request timeout

### Error Response Format

All errors should follow a consistent format:

```typescript
interface ErrorResponse {
  error: {
    code: string;           // Machine-readable error code
    message: string;        // Human-readable error message
    details?: any;          // Additional error context
    requestId: string;      // Request ID for tracking
    timestamp: number;      // Error timestamp
  };
}
```

### Error Handling Strategies

**Authentication Errors:**
- Invalid OTP: Return 401 with retry count remaining
- Expired OTP: Return 401 with message to request new OTP
- Expired session: Return 401 with message to re-authenticate
- Rate limit exceeded: Return 429 with retry-after header

**Booking Errors:**
- Duplicate booking: Return 409 with existing token details
- Invalid hospital/department: Return 404 with available options
- Queue full: Return 503 with alternative hospitals

**Queue Control Errors:**
- Unauthorized staff: Return 403 with facility mismatch details
- Invalid queue state: Return 400 with current state
- Concurrent update conflict: Return 409 with latest version

**AI Assistant Errors:**
- AWS Bedrock timeout: Return 504 with retry suggestion
- AWS Bedrock rate limit: Return 429 with exponential backoff
- Invalid conversation: Return 404 with message to start new conversation

**Location Service Errors:**
- Invalid coordinates: Return 400 with valid range
- AWS Location Service failure: Return 502 with fallback to cached data
- No hospitals found: Return 200 with empty array and suggestion to expand radius

**Notification Errors:**
- Invalid device token: Log error and remove token from database
- FCM/SNS failure: Retry with exponential backoff, log after final failure
- User opted out: Skip notification and log event

### Retry Policies

**Database Operations:**
- Retry up to 3 times with exponential backoff (100ms, 200ms, 400ms)
- Use jitter to prevent thundering herd
- Log all retry attempts

**External Service Calls:**
- AWS Bedrock: Retry up to 2 times with 1s, 2s backoff
- AWS Location Service: Retry up to 2 times with 500ms, 1s backoff
- SNS/FCM: Retry up to 2 times with 1s, 2s backoff

**Idempotency:**
- All POST/PUT/DELETE operations should be idempotent
- Use idempotency keys for critical operations (token booking, queue control)
- Store idempotency keys in DynamoDB with 24-hour TTL

## Testing Strategy

### Dual Testing Approach

The BionX platform requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests:**
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, special characters)
- Error conditions and exception handling
- Integration points between components
- Mock external services (AWS Bedrock, Location Service, SNS)

**Property-Based Tests:**
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization
- Minimum 100 iterations per property test
- Each test references its design document property
- Tag format: **Feature: bionx-healthcare-platform, Property {number}: {property_text}**

### Property-Based Testing Configuration

**Framework Selection:**
- **TypeScript/JavaScript**: fast-check
- **Python**: Hypothesis

**Test Configuration:**
```typescript
// Example fast-check configuration
fc.assert(
  fc.property(
    fc.string(), // Generate random phone numbers
    (phoneNumber) => {
      // Feature: bionx-healthcare-platform, Property 1: OTP Generation for Valid Phone Numbers
      const result = authService.sendOTP(phoneNumber);
      return result.success && result.expiresAt > Date.now();
    }
  ),
  { numRuns: 100 } // Minimum 100 iterations
);
```

### Test Coverage Requirements

**Authentication Service:**
- Unit tests: Valid OTP flow, invalid OTP, expired OTP, rate limiting
- Property tests: Properties 1, 2, 3, 4

**Token Service:**
- Unit tests: Single booking, cancellation, token retrieval
- Property tests: Properties 5, 6, 7, 8

**Queue Service:**
- Unit tests: Queue creation, single patient advancement, pause/resume
- Property tests: Properties 9, 10, 11, 12, 13, 14, 15, 29, 30, 31, 32, 33

**Hospital Service:**
- Unit tests: Single hospital retrieval, empty search results
- Property tests: Properties 16, 17, 18, 19, 20, 21

**AI Service:**
- Unit tests: Single message exchange, conversation creation
- Property tests: Properties 22, 23, 24, 49

**Notification Service:**
- Unit tests: Single notification, device registration
- Property tests: Property 15

**Staff Dashboard:**
- Unit tests: Dashboard load, single queue view
- Property tests: Properties 25, 26, 27, 28

**Emergency Features:**
- Unit tests: Emergency trigger, single hospital selection
- Property tests: Properties 34, 35, 36

**Data Consistency:**
- Unit tests: Single write, single read
- Property tests: Properties 37, 38, 39

**Security:**
- Unit tests: Valid token, invalid token, role check
- Property tests: Properties 40, 41, 42

**Monitoring:**
- Unit tests: Single log entry
- Property tests: Properties 43, 44

**Offline Capability:**
- Unit tests: Cache write, cache read, sync
- Property tests: Properties 45, 46, 47, 48

**Multi-Language:**
- Unit tests: Language switch, fallback
- Property tests: Properties 50, 51

**Analytics:**
- Unit tests: Single day statistics, single report
- Property tests: Properties 52, 53, 54, 55, 56

### Integration Testing

**End-to-End Flows:**
1. Complete booking flow: Register → Login → Search Hospital → Book Token → Receive Notification
2. Queue management flow: Staff Login → View Queue → Pause Queue → Patients Notified → Resume Queue
3. Emergency flow: Trigger Emergency → Search Hospitals → View Details → Initiate Call
4. AI assistant flow: Open Assistant → Send Message → Receive Response → Emergency Detection

**Performance Testing:**
- Load testing: 1000 concurrent users booking tokens
- Stress testing: Queue updates with 100 simultaneous staff actions
- Latency testing: API response times under normal load
- Cost testing: Monitor AWS costs during test runs

### Mocking Strategy

**External Services:**
- AWS Cognito: Mock OTP generation and validation
- AWS Bedrock: Mock AI responses with predefined templates
- AWS Location Service: Mock distance calculations with test data
- SNS/FCM: Mock notification delivery
- DynamoDB: Use local DynamoDB for unit tests

**Test Data:**
- Generate realistic test data using faker libraries
- Use property-based testing generators for random inputs
- Maintain test fixtures for common scenarios
- Seed test database with sample hospitals and departments

### Continuous Integration

**CI Pipeline:**
1. Lint code (ESLint, Prettier)
2. Run unit tests (Jest)
3. Run property-based tests (fast-check with 100 iterations)
4. Check code coverage (minimum 80%)
5. Run integration tests
6. Deploy to staging environment
7. Run smoke tests
8. Deploy to production (manual approval)

**Test Execution Time:**
- Unit tests: < 2 minutes
- Property tests: < 5 minutes
- Integration tests: < 10 minutes
- Total CI pipeline: < 20 minutes
