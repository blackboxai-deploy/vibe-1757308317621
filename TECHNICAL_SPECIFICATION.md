# The Gurukul Academy - Technical Specification Document

## Executive Summary

The Gurukul Academy is a production-ready Flutter e-learning application designed to scale from 1,000 to 50,000+ users. This comprehensive technical specification outlines the architecture, implementation details, and development roadmap for building a secure, cost-effective, and feature-rich educational platform.

### Technology Stack
- **Frontend**: Flutter 3.x with Riverpod state management
- **Backend**: Firebase (Auth, Firestore, Cloud Functions, Storage, FCM)
- **Video Hosting**: Bunny Stream with HLS adaptive streaming
- **Payments**: Razorpay (India-focused)
- **Offline Storage**: Hive for local caching
- **CI/CD**: GitHub Actions
- **Additional**: Admin web dashboard (optional Flutter Web)

---

## 1. System Architecture

### 1.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    The Gurukul Academy Architecture              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │   Flutter App   │    │  Admin Web UI   │    │   Firebase   │ │
│  │                 │    │  (Flutter Web)  │    │   Console    │ │
│  │ • Student UI    │    │                 │    │              │ │
│  │ • Teacher UI    │◄───┤ • User Mgmt     │◄───┤ • Analytics  │ │
│  │ • Basic Admin   │    │ • Course Mgmt   │    │ • Monitoring │ │
│  └─────────────────┘    │ • Analytics     │    └──────────────┘ │
│           │              └─────────────────┘                     │
│           │                       │                             │
│           ▼                       ▼                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Firebase Services                        │ │
│  │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────┐ │ │
│  │ │ Firebase    │ │ Firestore   │ │ Cloud       │ │ Cloud   │ │ │
│  │ │ Auth        │ │ Database    │ │ Functions   │ │ Storage │ │ │
│  │ │             │ │             │ │             │ │         │ │ │
│  │ │ • Phone     │ │ • Users     │ │ • Payments  │ │ • PDFs  │ │ │
│  │ │ • Roles     │ │ • Courses   │ │ • Video URLs│ │ • Images│ │ │
│  │ │ • JWT       │ │ • Lessons   │ │ • Webhooks  │ │ • Docs  │ │ │
│  │ └─────────────┘ │ • Chats     │ │ • Security  │ └─────────┘ │ │
│  │                 │ • Payments  │ └─────────────┘             │ │
│  │                 │ • Analytics │                             │ │
│  │                 └─────────────┘                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│           │                       │                             │
│           ▼                       ▼                             │
│  ┌─────────────────┐    ┌─────────────────────────────────────┐ │
│  │  Bunny Stream   │    │           External APIs             │ │
│  │                 │    │                                     │ │
│  │ • Video Storage │    │ ┌─────────────┐ ┌─────────────────┐ │ │
│  │ • HLS Streaming │    │ │  Razorpay   │ │      FCM        │ │ │
│  │ • CDN Delivery  │    │ │             │ │                 │ │ │
│  │ • Signed URLs   │    │ │ • Payments  │ │ • Push Notifs   │ │ │
│  │ • Analytics     │    │ │ • Webhooks  │ │ • Real-time     │ │ │
│  └─────────────────┘    │ │ • Refunds   │ │ • Messaging     │ │ │
│                         │ └─────────────┘ └─────────────────┘ │ │
│                         └─────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Interactions

**Data Flow:**
1. **Authentication**: Phone OTP → Firebase Auth → Role assignment → Firestore profile
2. **Content Upload**: Teacher uploads → Cloud Storage → Bunny Stream → Signed URL generation
3. **Payments**: Razorpay checkout → Webhook → Cloud Function → Firestore enrollment
4. **Real-time**: Firestore listeners → UI updates → FCM notifications

**Security Layers:**
- Firebase Auth tokens
- Firestore security rules
- Bunny Stream signed URLs
- Cloud Function validation
- Client-side role checks

---

## 2. Database Schema (Firestore)

### 2.1 Collections Structure

```typescript
// Firestore Collections Overview
/users/{userId}
/courses/{courseId}
/courses/{courseId}/lessons/{lessonId}
/enrollments/{enrollmentId}
/payments/{paymentId}
/chats/{chatId}
/chat_messages/{messageId}
/quizzes/{quizId}
/quiz_attempts/{attemptId}
/assignments/{assignmentId}
/assignment_submissions/{submissionId}
/notifications/{notificationId}
/analytics/{analyticsId}
```

### 2.2 Detailed Schema with Sample Documents

#### Users Collection
```typescript
// /users/{userId}
{
  "userId": "user123",
  "phone": "+919876543210",
  "email": "student@example.com", // Optional
  "name": "Rajesh Kumar",
  "role": "student", // admin | teacher | student
  "profilePicture": "https://storage.googleapis.com/...",
  "isVerified": true,
  "createdAt": "2024-01-15T10:30:00Z",
  "lastLoginAt": "2024-01-20T09:15:00Z",
  "metadata": {
    "deviceId": "device123",
    "appVersion": "1.0.0",
    "platform": "android"
  },
  // Student-specific fields
  "studentData": {
    "grade": "12",
    "subjects": ["mathematics", "physics", "chemistry"],
    "preferences": {
      "notifications": true,
      "downloadQuality": "medium"
    }
  },
  // Teacher-specific fields
  "teacherData": {
    "specialization": ["mathematics", "physics"],
    "qualification": "M.Sc. Mathematics",
    "experience": 5,
    "isApproved": true,
    "rating": 4.8,
    "totalStudents": 150
  },
  // Admin-specific fields
  "adminData": {
    "permissions": ["user_management", "course_approval", "analytics"],
    "level": "super_admin" // super_admin | admin | moderator
  }
}
```

#### Courses Collection
```typescript
// /courses/{courseId}
{
  "courseId": "course123",
  "title": "Complete JEE Mathematics",
  "description": "Comprehensive mathematics course for JEE preparation",
  "teacherId": "teacher123",
  "teacherName": "Prof. Sharma",
  "category": "mathematics",
  "level": "advanced", // beginner | intermediate | advanced
  "price": 2999, // In INR paisa (₹29.99)
  "discountPrice": 1999,
  "currency": "INR",
  "isPaid": true,
  "isPublished": true,
  "thumbnail": "https://storage.googleapis.com/workspace-0f70711f-8b4e-4d94-86f1-2a93ccde5887/image/441e4897-2b8f-44da-a935-e1d8af3e29bf.png",
  "duration": 7200, // Total duration in seconds
  "lessonsCount": 45,
  "studentsEnrolled": 1250,
  "rating": 4.7,
  "totalRatings": 340,
  "tags": ["jee", "mathematics", "algebra", "calculus"],
  "prerequisites": ["Basic Algebra", "Trigonometry"],
  "whatYouWillLearn": [
    "Advanced calculus concepts",
    "Complex number theory",
    "Coordinate geometry mastery"
  ],
  "createdAt": "2024-01-10T10:00:00Z",
  "updatedAt": "2024-01-18T15:30:00Z",
  "metadata": {
    "totalVideoSize": 1024000000, // In bytes
    "totalPDFSize": 50000000,
    "language": "hindi", // hindi | english | bilingual
    "subtitles": ["hindi", "english"]
  }
}
```

#### Lessons Collection (Subcollection)
```typescript
// /courses/{courseId}/lessons/{lessonId}
{
  "lessonId": "lesson123",
  "courseId": "course123",
  "title": "Limits and Continuity",
  "description": "Understanding the fundamental concepts of limits",
  "order": 3, // Lesson sequence
  "type": "video", // video | pdf | quiz | assignment
  "duration": 1800, // In seconds (30 minutes)
  "isPreview": false, // Free preview lesson
  "content": {
    // Video content
    "videoId": "bunny_video_123",
    "videoUrl": "", // Generated dynamically with signed URL
    "thumbnail": "https://storage.googleapis.com/workspace-0f70711f-8b4e-4d94-86f1-2a93ccde5887/image/40c81b74-fe77-4ad3-b641-f060f85db0c2.png",
    "quality": ["720p", "480p", "360p"],
    "size": {
      "720p": 150000000, // In bytes
      "480p": 85000000,
      "360p": 45000000
    },
    // PDF content
    "pdfUrl": "https://storage.googleapis.com/gurukul/lesson123.pdf",
    "pdfSize": 2048000,
    // Quiz content
    "quizId": "quiz123",
    "passingScore": 70
  },
  "watchTime": {
    "averageWatchTime": 1620, // seconds
    "completionRate": 85 // percentage
  },
  "createdAt": "2024-01-12T14:20:00Z",
  "updatedAt": "2024-01-12T14:20:00Z"
}
```

### 2.3 Indexing Strategy

```typescript
// Firestore Indexes for Performance
// These should be created via Firebase Console or CLI

// Compound Indexes
1. courses: [teacherId, isPublished, createdAt (desc)]
2. enrollments: [userId, status, enrolledAt (desc)]
3. payments: [userId, status, createdAt (desc)]
4. chat_messages: [chatId, timestamp (desc)]
5. quiz_attempts: [userId, quizId, attemptNumber (desc)]

// Single Field Indexes (Auto-created)
- users.role
- courses.category
- courses.isPaid
- lessons.type
- payments.status
- notifications.userId
```

---

## 3. Key Features Implementation

### 3.1 Authentication & Role Management
- Firebase Phone Authentication with OTP verification
- Role-based access control (Admin, Teacher, Student)
- Secure user profile management
- Device registration and tracking

### 3.2 Course Management System
- Course creation and publishing workflow
- Multi-media content support (videos, PDFs, quizzes)
- Lesson sequencing and progress tracking
- Course categorization and search

### 3.3 Video Streaming Security
- Bunny Stream integration with signed URLs
- Domain restriction and token expiry
- Adaptive streaming (HLS) support
- Video analytics and progress tracking

### 3.4 Payment Processing
- Razorpay integration for Indian market
- Secure payment verification
- Enrollment automation
- Refund and subscription management

### 3.5 Real-time Communication
- 1:1 chat between students and teachers
- Group announcements
- FCM push notifications
- Message history and file attachments

---

## 4. Security Implementation

### 4.1 Firebase Security Rules
- Role-based document access control
- Server-side validation for critical operations
- Prevention of unauthorized data access
- Audit logging for sensitive operations

### 4.2 Payment Security
- Server-side payment verification
- Webhook signature validation
- Secure API key management
- PCI DSS compliance considerations

### 4.3 Video Content Protection
- Signed URL generation with expiry
- Domain-based access restriction
- Anti-piracy measures
- Download permission controls

---

## 5. Scalability & Performance

### 5.1 Database Optimization
- Efficient indexing strategy
- Data denormalization for read performance
- Batch operations for bulk updates
- Caching strategy implementation

### 5.2 Cost Optimization
- Firebase usage optimization
- Bunny Stream bandwidth management
- Efficient Cloud Function deployment
- Storage cost minimization

### 5.3 Performance Monitoring
- Real-time analytics dashboard
- Error tracking and alerting
- Performance metrics collection
- User behavior analysis

---

## 6. Development Roadmap

### Phase 1: Foundation (Weeks 1-3)
- Project setup and architecture
- Firebase configuration
- Authentication system
- Basic UI framework

### Phase 2: Core Features (Weeks 4-7)
- Course management system
- Video integration
- Payment processing
- User role management

### Phase 3: Advanced Features (Weeks 8-10)
- Real-time chat
- Quiz and assignment system
- Offline content support
- Analytics dashboard

### Phase 4: Optimization & Launch (Weeks 11-12)
- Performance optimization
- Security audit
- Testing and bug fixes
- Production deployment

---

## 7. Cost Estimates

### For 1,000 Users/Month
- Firebase: $25-50
- Bunny Stream: $10-20
- Cloud Functions: $5-15
- **Total: $40-85/month**

### For 10,000 Users/Month
- Firebase: $150-300
- Bunny Stream: $80-150
- Cloud Functions: $25-50
- **Total: $255-500/month**

### For 50,000 Users/Month
- Firebase: $600-1,200
- Bunny Stream: $300-600
- Cloud Functions: $100-200
- **Total: $1,000-2,000/month**

---

## 8. Testing Strategy

### 8.1 Testing Levels
- Unit tests for business logic
- Integration tests for API endpoints
- E2E tests for critical user journeys
- Performance tests for scalability

### 8.2 Testing Tools
- Flutter Test for unit tests
- Integration testing framework
- Firebase Test Lab for device testing
- Load testing for performance validation

---

## 9. Deployment & CI/CD

### 9.1 Build Pipeline
- Automated Flutter builds
- Code quality checks
- Security vulnerability scanning
- Automated testing execution

### 9.2 Deployment Strategy
- Staging environment validation
- Blue-green deployment
- Rollback capabilities
- Performance monitoring

---

## 10. Monitoring & Analytics

### 10.1 Application Monitoring
- Crash reporting
- Performance metrics
- User engagement analytics
- Revenue tracking

### 10.2 Business Intelligence
- Course completion rates
- User retention analysis
- Revenue optimization
- Feature usage statistics

---

This technical specification provides a comprehensive foundation for building The Gurukul Academy e-learning platform. Each section includes detailed implementation guidelines, security considerations, and scalability planning to ensure successful deployment and operation.