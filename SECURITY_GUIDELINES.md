# The Gurukul Academy - Security Guidelines & Best Practices

## Comprehensive Security Implementation Guide

### **1. Authentication & Authorization Security**

#### **1.1 Firebase Phone Authentication**

**Best Practices:**
- Enable App Check to prevent unauthorized API usage
- Implement rate limiting for OTP requests
- Use secure random generation for verification codes
- Set proper timeout values (60-120 seconds)

**Implementation:**
```dart
// Secure OTP configuration
await _auth.verifyPhoneNumber(
  phoneNumber: phoneNumber,
  timeout: const Duration(seconds: 60), // Reasonable timeout
  forceResendingToken: resendToken, // Prevent spam
  verificationCompleted: (credential) => {
    // Auto-verify only on trusted devices
  },
  verificationFailed: (exception) => {
    // Log failed attempts for monitoring
    _logSecurityEvent('otp_verification_failed', {
      'phone': phoneNumber,
      'error': exception.code,
      'timestamp': DateTime.now().toIso8601String(),
    });
  }
);
```

**Security Measures:**
- Monitor failed authentication attempts
- Block suspicious IP addresses
- Implement account lockout after multiple failures
- Use secure storage for authentication tokens

#### **1.2 Role-Based Access Control (RBAC)**

**Security Rules Implementation:**
```javascript
// Firestore Security Rules - Enhanced
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Security logging function
    function logSecurityEvent(action, resource) {
      return request.auth != null && 
             get(/databases/$(database)/documents/security_logs/$(request.auth.uid + '_' + action)).data != null;
    }
    
    // Enhanced user role checking
    function getUserData() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data;
    }
    
    function isValidRole(role) {
      return role in ['student', 'teacher', 'admin'] && 
             getUserData().role == role &&
             getUserData().isVerified == true;
    }
    
    // Rate limiting check
    function checkRateLimit(resource, maxRequests) {
      return resource.data.requestCount < maxRequests &&
             resource.data.lastRequest < request.time - duration.value(1, 'm');
    }
    
    // Users collection with enhanced security
    match /users/{userId} {
      allow read: if isAuthenticated() && 
        (request.auth.uid == userId || isValidRole('admin'));
      
      allow create: if isAuthenticated() && 
        request.auth.uid == userId &&
        validateUserData(request.resource.data);
      
      allow update: if isAuthenticated() && 
        (request.auth.uid == userId || isValidRole('admin')) &&
        validateUserUpdate(request.resource.data, resource.data);
      
      allow delete: if false; // Prevent user deletion via client
    }
    
    // Data validation functions
    function validateUserData(userData) {
      return userData.keys().hasAll(['phone', 'name', 'role']) &&
             userData.role in ['student', 'teacher'] &&
             userData.phone.matches('^\\+91[6-9]\\d{9}$') &&
             userData.name.size() > 0 &&
             userData.name.size() <= 100;
    }
    
    function validateUserUpdate(newData, oldData) {
      return !('role' in newData.diff(oldData).affectedKeys()) ||
             isValidRole('admin');
    }
  }
}
```

### **2. Payment Security**

#### **2.1 Razorpay Integration Security**

**Server-Side Verification:**
```javascript
// Cloud Function for secure payment verification
exports.verifyPayment = functions.https.onCall(async (data, context) => {
  // Input validation
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'Authentication required');
  }
  
  const { paymentId, orderId, signature, courseId } = data;
  
  // Validate input parameters
  if (!paymentId || !orderId || !signature || !courseId) {
    throw new functions.https.HttpsError('invalid-argument', 'Missing required parameters');
  }
  
  try {
    // Verify Razorpay signature
    const generatedSignature = crypto
      .createHmac('sha256', functions.config().razorpay.webhook_secret)
      .update(`${orderId}|${paymentId}`)
      .digest('hex');
    
    if (generatedSignature !== signature) {
      // Log security incident
      await logSecurityIncident('payment_signature_mismatch', {
        userId: context.auth.uid,
        paymentId: paymentId,
        orderId: orderId,
        timestamp: new Date().toISOString()
      });
      throw new functions.https.HttpsError('permission-denied', 'Invalid payment signature');
    }
    
    // Verify payment amount and course price match
    const [paymentDoc, courseDoc] = await Promise.all([
      admin.firestore().collection('payments').doc(orderId).get(),
      admin.firestore().collection('courses').doc(courseId).get()
    ]);
    
    if (!paymentDoc.exists || !courseDoc.exists) {
      throw new functions.https.HttpsError('not-found', 'Payment or course not found');
    }
    
    const payment = paymentDoc.data();
    const course = courseDoc.data();
    
    if (payment.amount !== course.price || payment.userId !== context.auth.uid) {
      await logSecurityIncident('payment_amount_mismatch', {
        userId: context.auth.uid,
        paymentAmount: payment.amount,
        coursePrice: course.price,
        timestamp: new Date().toISOString()
      });
      throw new functions.https.HttpsError('permission-denied', 'Payment verification failed');
    }
    
    // Create enrollment atomically
    const batch = admin.firestore().batch();
    
    // Update payment status
    batch.update(paymentDoc.ref, {
      paymentId: paymentId,
      status: 'verified',
      verifiedAt: admin.firestore.FieldValue.serverTimestamp(),
      verificationSignature: signature
    });
    
    // Create enrollment
    const enrollmentRef = admin.firestore().collection('enrollments')
      .doc(`${context.auth.uid}_${courseId}`);
    batch.set(enrollmentRef, {
      userId: context.auth.uid,
      courseId: courseId,
      paymentId: paymentId,
      status: 'active',
      enrolledAt: admin.firestore.FieldValue.serverTimestamp()
    });
    
    await batch.commit();
    
    // Log successful payment
    await logSecurityEvent('payment_verified', {
      userId: context.auth.uid,
      courseId: courseId,
      amount: payment.amount
    });
    
    return { verified: true, enrollmentId: enrollmentRef.id };
    
  } catch (error) {
    console.error('Payment verification error:', error);
    await logSecurityIncident('payment_verification_error', {
      userId: context.auth.uid,
      error: error.message,
      timestamp: new Date().toISOString()
    });
    throw new functions.https.HttpsError('internal', 'Payment verification failed');
  }
});
```

**Security Measures:**
- Never store payment credentials in the client
- Use HTTPS for all payment communications
- Implement webhook signature verification
- Log all payment events for audit trails
- Use atomic transactions for payment updates

#### **2.2 Payment Data Protection**

```dart
// Secure payment data handling
class PaymentSecurityService {
  // Encrypt sensitive payment data before storage
  static Future<String> encryptPaymentData(Map<String, dynamic> data) async {
    final key = await _getEncryptionKey();
    final encrypted = encrypt(json.encode(data), key);
    return encrypted;
  }
  
  // Secure storage for payment receipts
  static Future<void> storePaymentReceipt(String paymentId, String encryptedData) async {
    await FirebaseFirestore.instance.collection('payment_receipts').add({
      'paymentId': paymentId,
      'encryptedData': encryptedData,
      'createdAt': FieldValue.serverTimestamp(),
      'userId': FirebaseAuth.instance.currentUser?.uid,
    });
  }
  
  // Validate payment before processing
  static bool validatePaymentData(PaymentData data) {
    return data.amount > 0 &&
           data.currency == 'INR' &&
           data.courseId.isNotEmpty &&
           RegExp(r'^[a-zA-Z0-9_]+$').hasMatch(data.courseId);
  }
}
```

### **3. Video Content Security**

#### **3.1 Bunny Stream Security Implementation**

**Signed URL Generation:**
```javascript
// Cloud Function for secure video URL generation
exports.getSignedVideoUrl = functions.https.onCall(async (data, context) => {
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'Authentication required');
  }
  
  const { videoId, courseId, lessonId } = data;
  
  try {
    // Verify user enrollment
    const enrollmentDoc = await admin.firestore()
      .collection('enrollments')
      .doc(`${context.auth.uid}_${courseId}`)
      .get();
    
    if (!enrollmentDoc.exists || enrollmentDoc.data().status !== 'active') {
      await logSecurityIncident('unauthorized_video_access', {
        userId: context.auth.uid,
        videoId: videoId,
        courseId: courseId,
        timestamp: new Date().toISOString()
      });
      throw new functions.https.HttpsError('permission-denied', 'Course enrollment required');
    }
    
    // Generate signed URL with expiration
    const expirationTime = Math.floor(Date.now() / 1000) + (2 * 60 * 60); // 2 hours
    const authKey = functions.config().bunnystream.auth_key;
    const pullZone = functions.config().bunnystream.pull_zone;
    
    // Create signature
    const signatureString = `${authKey}${videoId}${expirationTime}`;
    const signature = crypto.createHash('sha256').update(signatureString).digest('hex');
    
    const signedUrl = `https://${pullZone}.b-cdn.net/${videoId}?token=${signature}&expires=${expirationTime}`;
    
    // Log video access
    await logVideoAccess(context.auth.uid, videoId, courseId);
    
    return { 
      url: signedUrl, 
      expiresAt: expirationTime,
      restrictions: {
        maxViews: 5,
        allowDownload: false,
        domainRestriction: true
      }
    };
    
  } catch (error) {
    console.error('Video URL generation error:', error);
    throw new functions.https.HttpsError('internal', 'Failed to generate video URL');
  }
});

// Video access logging for analytics and security
async function logVideoAccess(userId, videoId, courseId) {
  await admin.firestore().collection('video_access_logs').add({
    userId: userId,
    videoId: videoId,
    courseId: courseId,
    accessTime: admin.firestore.FieldValue.serverTimestamp(),
    ipAddress: functions.https.onCall.request?.ip,
    userAgent: functions.https.onCall.request?.headers['user-agent']
  });
}
```

**Client-Side Security:**
```dart
class VideoSecurityService {
  // Secure video player initialization
  static Future<VideoPlayerController> initSecureVideoPlayer({
    required String videoId,
    required String courseId,
    required String lessonId,
  }) async {
    try {
      // Get signed URL from secure endpoint
      final result = await FirebaseFunctions.instance
          .httpsCallable('getSignedVideoUrl')
          .call({
        'videoId': videoId,
        'courseId': courseId,
        'lessonId': lessonId,
      });
      
      final videoUrl = result.data['url'] as String;
      final expiresAt = result.data['expiresAt'] as int;
      
      // Verify URL hasn't expired
      if (DateTime.now().millisecondsSinceEpoch / 1000 > expiresAt) {
        throw Exception('Video URL has expired');
      }
      
      // Initialize player with secure URL
      final controller = VideoPlayerController.networkUrl(Uri.parse(videoUrl));
      
      // Add security listeners
      controller.addListener(() {
        _trackVideoProgress(videoId, controller.value.position);
        _detectSuspiciousActivity(controller);
      });
      
      return controller;
      
    } catch (e) {
      // Log security error
      await FirebaseFirestore.instance.collection('security_logs').add({
        'event': 'video_access_failed',
        'videoId': videoId,
        'error': e.toString(),
        'timestamp': FieldValue.serverTimestamp(),
        'userId': FirebaseAuth.instance.currentUser?.uid,
      });
      rethrow;
    }
  }
  
  // Detect suspicious video activity
  static void _detectSuspiciousActivity(VideoPlayerController controller) {
    final position = controller.value.position;
    final duration = controller.value.duration;
    
    // Detect rapid seeking (potential piracy attempt)
    if (_lastPosition != null) {
      final timeDiff = position.inMilliseconds - _lastPosition!.inMilliseconds;
      if (timeDiff.abs() > 30000 && timeDiff.abs() < 60000) {
        _reportSuspiciousActivity('rapid_seeking', {
          'position': position.inSeconds,
          'jump': timeDiff,
        });
      }
    }
    
    _lastPosition = position;
  }
  
  static Duration? _lastPosition;
  
  static void _reportSuspiciousActivity(String activity, Map<String, dynamic> data) {
    FirebaseFirestore.instance.collection('security_incidents').add({
      'type': 'video_security',
      'activity': activity,
      'data': data,
      'userId': FirebaseAuth.instance.currentUser?.uid,
      'timestamp': FieldValue.serverTimestamp(),
    });
  }
}
```

### **4. Data Protection & Privacy**

#### **4.1 Data Encryption**

```dart
// Encryption service for sensitive data
class EncryptionService {
  static final _encrypter = Encrypter(AES(Key.fromSecureRandom(32)));
  
  // Encrypt user sensitive data
  static String encryptSensitiveData(String data) {
    try {
      final encrypted = _encrypter.encrypt(data, iv: IV.fromSecureRandom(16));
      return encrypted.base64;
    } catch (e) {
      throw Exception('Encryption failed: $e');
    }
  }
  
  // Decrypt user data
  static String decryptSensitiveData(String encryptedData) {
    try {
      final encrypted = Encrypted.fromBase64(encryptedData);
      return _encrypter.decrypt(encrypted, iv: IV.fromSecureRandom(16));
    } catch (e) {
      throw Exception('Decryption failed: $e');
    }
  }
  
  // Secure key storage
  static Future<void> storeEncryptionKey(String key) async {
    const secureStorage = FlutterSecureStorage(
      aOptions: AndroidOptions(
        encryptedSharedPreferences: true,
        keyCipherAlgorithm: KeyCipherAlgorithm.RSA_ECB_PKCS1Padding,
        storageCipherAlgorithm: StorageCipherAlgorithm.AES_GCM_NoPadding,
      ),
      iOptions: IOSOptions(
        accessibility: IOSAccessibility.first_unlock_this_device,
        accountName: 'GurukulAcademy',
      ),
    );
    
    await secureStorage.write(key: 'master_key', value: key);
  }
}
```

#### **4.2 Personal Data Protection**

```dart
// GDPR and privacy compliance service
class PrivacyComplianceService {
  // Data minimization - collect only necessary data
  static Map<String, dynamic> sanitizeUserData(Map<String, dynamic> userData) {
    final allowedFields = {
      'name', 'phone', 'email', 'role', 'grade', 'subjects',
      'preferences', 'createdAt', 'lastLoginAt'
    };
    
    return Map.fromEntries(
      userData.entries.where((entry) => allowedFields.contains(entry.key))
    );
  }
  
  // Data anonymization for analytics
  static Map<String, dynamic> anonymizeUserData(Map<String, dynamic> userData) {
    return {
      'hashedUserId': _hashUserId(userData['userId']),
      'role': userData['role'],
      'grade': userData['grade'],
      'subjects': userData['subjects'],
      'registrationMonth': _getRegistrationMonth(userData['createdAt']),
      'isActive': userData['lastLoginAt'] != null,
    };
  }
  
  // User consent management
  static Future<void> recordConsent({
    required String userId,
    required String consentType,
    required bool granted,
  }) async {
    await FirebaseFirestore.instance.collection('user_consents').add({
      'userId': userId,
      'consentType': consentType,
      'granted': granted,
      'timestamp': FieldValue.serverTimestamp(),
      'ipAddress': await _getClientIP(),
    });
  }
  
  // Data deletion (right to be forgotten)
  static Future<void> deleteUserData(String userId) async {
    final batch = FirebaseFirestore.instance.batch();
    
    // List all collections containing user data
    final collections = [
      'users', 'enrollments', 'payments', 'chat_messages',
      'quiz_attempts', 'video_progress', 'user_consents'
    ];
    
    for (final collection in collections) {
      final docs = await FirebaseFirestore.instance
          .collection(collection)
          .where('userId', isEqualTo: userId)
          .get();
      
      for (final doc in docs.docs) {
        batch.delete(doc.reference);
      }
    }
    
    await batch.commit();
    
    // Log data deletion
    await FirebaseFirestore.instance.collection('data_deletion_logs').add({
      'userId': userId,
      'deletedAt': FieldValue.serverTimestamp(),
      'reason': 'user_request',
    });
  }
  
  static String _hashUserId(String userId) {
    return crypto.sha256.convert(utf8.encode(userId)).toString();
  }
  
  static String _getRegistrationMonth(Timestamp? timestamp) {
    if (timestamp == null) return 'unknown';
    final date = timestamp.toDate();
    return '${date.year}-${date.month.toString().padLeft(2, '0')}';
  }
  
  static Future<String> _getClientIP() async {
    // Implementation to get client IP
    return 'unknown';
  }
}
```

### **5. Security Monitoring & Incident Response**

#### **5.1 Real-time Security Monitoring**

```javascript
// Cloud Function for security monitoring
exports.securityMonitor = functions.firestore
  .document('security_logs/{logId}')
  .onCreate(async (snapshot, context) => {
    
    const logData = snapshot.data();
    const severity = calculateSeverity(logData);
    
    if (severity === 'HIGH' || severity === 'CRITICAL') {
      // Immediate alert
      await sendSecurityAlert(logData, severity);
      
      // Auto-block if critical
      if (severity === 'CRITICAL') {
        await autoBlockUser(logData.userId, logData.event);
      }
    }
    
    // Update security metrics
    await updateSecurityMetrics(logData);
  });

async function calculateSeverity(logData) {
  const highRiskEvents = [
    'payment_signature_mismatch',
    'unauthorized_video_access',
    'rapid_authentication_attempts',
    'suspicious_video_activity'
  ];
  
  const criticalEvents = [
    'data_breach_attempt',
    'admin_privilege_escalation',
    'payment_fraud_detected'
  ];
  
  if (criticalEvents.includes(logData.event)) return 'CRITICAL';
  if (highRiskEvents.includes(logData.event)) return 'HIGH';
  
  // Check for patterns
  const recentLogs = await admin.firestore()
    .collection('security_logs')
    .where('userId', '==', logData.userId)
    .where('timestamp', '>', new Date(Date.now() - 60000)) // Last minute
    .get();
  
  if (recentLogs.size > 5) return 'HIGH'; // Rapid events
  
  return 'MEDIUM';
}

async function sendSecurityAlert(logData, severity) {
  // Send to security team
  const alertData = {
    severity: severity,
    event: logData.event,
    userId: logData.userId,
    timestamp: logData.timestamp,
    details: logData,
    requiresImmediateAction: severity === 'CRITICAL'
  };
  
  // Multiple alert channels
  await Promise.all([
    sendEmailAlert(alertData),
    sendSlackAlert(alertData),
    logToSIEM(alertData)
  ]);
}
```

#### **5.2 Automated Incident Response**

```dart
class IncidentResponseService {
  // Automated security response system
  static Future<void> handleSecurityIncident({
    required String incidentType,
    required String userId,
    required Map<String, dynamic> incidentData,
  }) async {
    
    switch (incidentType) {
      case 'multiple_failed_logins':
        await _handleFailedLogins(userId, incidentData);
        break;
      case 'suspicious_payment':
        await _handleSuspiciousPayment(userId, incidentData);
        break;
      case 'unauthorized_access_attempt':
        await _handleUnauthorizedAccess(userId, incidentData);
        break;
      case 'content_piracy_detected':
        await _handleContentPiracy(userId, incidentData);
        break;
    }
    
    // Log incident for forensics
    await _logIncidentForForensics(incidentType, userId, incidentData);
  }
  
  static Future<void> _handleFailedLogins(String userId, Map<String, dynamic> data) async {
    final attemptCount = data['attemptCount'] as int;
    
    if (attemptCount >= 5) {
      // Temporary account lock
      await FirebaseFirestore.instance
          .collection('users')
          .doc(userId)
          .update({
        'accountStatus': 'temporarily_locked',
        'lockedUntil': Timestamp.fromDate(
          DateTime.now().add(Duration(minutes: 30))
        ),
        'lockReason': 'multiple_failed_login_attempts'
      });
      
      // Notify user via SMS/Email
      await _sendSecurityNotification(userId, 'account_temporarily_locked');
    }
  }
  
  static Future<void> _handleSuspiciousPayment(String userId, Map<String, dynamic> data) async {
    // Flag payment for manual review
    await FirebaseFirestore.instance
        .collection('payment_reviews')
        .add({
      'userId': userId,
      'paymentId': data['paymentId'],
      'suspiciousActivity': data['flags'],
      'status': 'pending_review',
      'createdAt': FieldValue.serverTimestamp(),
      'reviewPriority': 'high'
    });
    
    // Temporarily hold enrollment
    await FirebaseFirestore.instance
        .collection('enrollments')
        .doc(data['enrollmentId'])
        .update({
      'status': 'under_review',
      'reviewReason': 'suspicious_payment_activity'
    });
  }
  
  static Future<void> _handleContentPiracy(String userId, Map<String, dynamic> data) async {
    // Immediate video access revocation
    await FirebaseFirestore.instance
        .collection('video_access_tokens')
        .where('userId', isEqualTo: userId)
        .get()
        .then((snapshot) {
      final batch = FirebaseFirestore.instance.batch();
      for (final doc in snapshot.docs) {
        batch.update(doc.reference, {'revoked': true});
      }
      return batch.commit();
    });
    
    // Flag user account
    await FirebaseFirestore.instance
        .collection('users')
        .doc(userId)
        .update({
      'securityFlags': FieldValue.arrayUnion(['content_piracy_suspected']),
      'accountStatus': 'under_investigation'
    });
  }
  
  static Future<void> _sendSecurityNotification(String userId, String notificationType) async {
    // Implementation for sending security notifications
    // via SMS, Email, and in-app notifications
  }
  
  static Future<void> _logIncidentForForensics(
    String incidentType,
    String userId,
    Map<String, dynamic> incidentData,
  ) async {
    await FirebaseFirestore.instance
        .collection('forensic_logs')
        .add({
      'incidentType': incidentType,
      'userId': userId,
      'incidentData': incidentData,
      'timestamp': FieldValue.serverTimestamp(),
      'deviceInfo': await _getDeviceInfo(),
      'networkInfo': await _getNetworkInfo(),
    });
  }
  
  static Future<Map<String, dynamic>> _getDeviceInfo() async {
    // Get device information for forensics
    return {};
  }
  
  static Future<Map<String, dynamic>> _getNetworkInfo() async {
    // Get network information for forensics
    return {};
  }
}
```

### **6. Security Checklist**

#### **Pre-Launch Security Audit**
- [ ] All API endpoints secured with proper authentication
- [ ] Payment flows tested with various fraud scenarios
- [ ] Video content protection mechanisms verified
- [ ] Data encryption implemented for sensitive information
- [ ] Security rules tested with penetration testing
- [ ] Rate limiting configured for all endpoints
- [ ] Monitoring and alerting systems operational
- [ ] Incident response procedures documented and tested
- [ ] GDPR compliance verified for user data handling
- [ ] Security headers configured on all services

#### **Ongoing Security Maintenance**
- [ ] Regular security rule audits (monthly)
- [ ] Payment security reviews (quarterly)
- [ ] Video content protection assessments (quarterly)
- [ ] Security incident response drills (bi-annually)
- [ ] Third-party security assessments (annually)
- [ ] User data audit and cleanup (quarterly)
- [ ] Security training for development team (quarterly)
- [ ] Dependencies security updates (monthly)
- [ ] Cloud infrastructure security reviews (quarterly)
- [ ] Backup and disaster recovery testing (quarterly)

This comprehensive security guide ensures that The Gurukul Academy platform maintains the highest security standards throughout development and operation, protecting user data, payment information, and content assets while providing a secure learning environment.