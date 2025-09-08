# The Gurukul Academy - Implementation Guide

## Flutter Code Examples and Integration Details

### 1. Flutter Project Setup

#### pubspec.yaml Dependencies
```yaml
name: gurukul_academy
description: A comprehensive e-learning platform

dependencies:
  flutter:
    sdk: flutter
  
  # State Management
  flutter_riverpod: ^2.4.9
  riverpod_annotation: ^2.3.3
  
  # Firebase
  firebase_core: ^2.24.2
  firebase_auth: ^4.15.3
  cloud_firestore: ^4.13.6
  firebase_storage: ^11.5.6
  firebase_messaging: ^14.7.10
  firebase_analytics: ^10.7.4
  cloud_functions: ^4.6.1
  
  # Video Player
  video_player: ^2.8.1
  chewie: ^1.7.4
  
  # Payments
  razorpay_flutter: ^1.3.7
  
  # Offline Storage
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  
  # Networking
  dio: ^5.4.0
  dio_cache_interceptor: ^3.4.4
  dio_cache_interceptor_hive_store: ^3.2.2
  
  # UI/UX
  flutter_screenutil: ^5.9.0
  cached_network_image: ^3.3.0
  shimmer: ^3.0.0
  lottie: ^2.7.0
  
  # Utilities
  intl: ^0.19.0
  url_launcher: ^6.2.2
  permission_handler: ^11.1.0
  device_info_plus: ^9.1.1
  package_info_plus: ^4.2.0
  
  # Local Storage
  shared_preferences: ^2.2.2
  path_provider: ^2.1.2
  
  # File handling
  file_picker: ^6.1.1
  image_picker: ^1.0.7
  
  # PDF
  flutter_pdfview: ^1.3.2
  
  # Charts and Analytics
  fl_chart: ^0.66.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  build_runner: ^2.4.7
  riverpod_generator: ^2.3.9
  hive_generator: ^2.0.1
  json_annotation: ^4.8.1
  json_serializable: ^6.7.1

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/animations/
    - assets/icons/
```

### 2. Firebase Phone Authentication

#### Authentication Service
```dart
// lib/services/auth_service.dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

enum AuthStatus { initial, loading, authenticated, unauthenticated, error, otpSent }

class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;
  
  User? get currentUser => _auth.currentUser;
  String? _verificationId;
  
  // Stream of auth state changes
  Stream<User?> get authStateChanges => _auth.authStateChanges();
  
  // Send OTP to phone number
  Future<bool> sendOTP(String phoneNumber) async {
    try {
      await _auth.verifyPhoneNumber(
        phoneNumber: phoneNumber,
        verificationCompleted: (PhoneAuthCredential credential) async {
          // Auto-verification on Android
          await _signInWithCredential(credential);
        },
        verificationFailed: (FirebaseAuthException e) {
          throw Exception('Verification failed: ${e.message}');
        },
        codeSent: (String verificationId, int? resendToken) {
          _verificationId = verificationId;
        },
        codeAutoRetrievalTimeout: (String verificationId) {
          _verificationId = verificationId;
        },
        timeout: const Duration(seconds: 60),
      );
      return true;
    } catch (e) {
      throw Exception('Failed to send OTP: $e');
    }
  }
  
  // Verify OTP and sign in
  Future<UserCredential> verifyOTP(String otp) async {
    try {
      if (_verificationId == null) {
        throw Exception('Verification ID not found. Please request OTP again.');
      }
      
      PhoneAuthCredential credential = PhoneAuthProvider.credential(
        verificationId: _verificationId!,
        smsCode: otp,
      );
      
      return await _signInWithCredential(credential);
    } catch (e) {
      throw Exception('OTP verification failed: $e');
    }
  }
  
  // Sign in with credential
  Future<UserCredential> _signInWithCredential(PhoneAuthCredential credential) async {
    try {
      UserCredential userCredential = await _auth.signInWithCredential(credential);
      
      // Check if user profile exists, create if not
      if (userCredential.additionalUserInfo?.isNewUser == true) {
        await _createUserProfile(userCredential.user!);
      }
      
      return userCredential;
    } catch (e) {
      throw Exception('Sign in failed: $e');
    }
  }
  
  // Create user profile in Firestore
  Future<void> _createUserProfile(User user) async {
    try {
      await _firestore.collection('users').doc(user.uid).set({
        'userId': user.uid,
        'phone': user.phoneNumber,
        'name': '', // To be filled during onboarding
        'role': 'student', // Default role
        'isVerified': false,
        'createdAt': FieldValue.serverTimestamp(),
        'lastLoginAt': FieldValue.serverTimestamp(),
        'metadata': {
          'deviceId': '', // To be filled with device info
          'appVersion': '', // To be filled with app version
          'platform': '', // To be filled with platform info
        },
      });
    } catch (e) {
      throw Exception('Failed to create user profile: $e');
    }
  }
  
  // Update user profile
  Future<void> updateUserProfile({
    required String name,
    String? email,
    required String role,
    Map<String, dynamic>? additionalData,
  }) async {
    try {
      final user = currentUser;
      if (user == null) throw Exception('No authenticated user');
      
      Map<String, dynamic> updateData = {
        'name': name,
        'role': role,
        'updatedAt': FieldValue.serverTimestamp(),
      };
      
      if (email != null) updateData['email'] = email;
      if (additionalData != null) updateData.addAll(additionalData);
      
      await _firestore.collection('users').doc(user.uid).update(updateData);
    } catch (e) {
      throw Exception('Failed to update profile: $e');
    }
  }
  
  // Get user profile
  Future<Map<String, dynamic>?> getUserProfile([String? userId]) async {
    try {
      final uid = userId ?? currentUser?.uid;
      if (uid == null) return null;
      
      final doc = await _firestore.collection('users').doc(uid).get();
      return doc.exists ? doc.data() : null;
    } catch (e) {
      throw Exception('Failed to get user profile: $e');
    }
  }
  
  // Sign out
  Future<void> signOut() async {
    try {
      await _auth.signOut();
    } catch (e) {
      throw Exception('Sign out failed: $e');
    }
  }
}

// Riverpod providers
final authServiceProvider = Provider<AuthService>((ref) => AuthService());

final authStateProvider = StreamProvider<User?>((ref) {
  final authService = ref.watch(authServiceProvider);
  return authService.authStateChanges;
});

final userProfileProvider = FutureProvider.family<Map<String, dynamic>?, String?>((ref, userId) {
  final authService = ref.watch(authServiceProvider);
  return authService.getUserProfile(userId);
});
```

### 3. Bunny Stream Video Integration

#### Video Service
```dart
// lib/services/video_service.dart
import 'dart:convert';
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:cloud_functions/cloud_functions.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class VideoService {
  final Dio _dio = Dio();
  final FirebaseFunctions _functions = FirebaseFunctions.instance;
  
  // Upload video to Bunny Stream
  Future<String> uploadVideo({
    required File videoFile,
    required String courseId,
    required String lessonId,
    required String title,
    void Function(double)? onProgress,
  }) async {
    try {
      // Get signed upload URL from Cloud Function
      final result = await _functions.httpsCallable('getSignedUploadUrl').call({
        'courseId': courseId,
        'lessonId': lessonId,
        'fileName': videoFile.path.split('/').last,
        'contentType': 'video/mp4',
      });
      
      final uploadUrl = result.data['uploadUrl'] as String;
      final videoId = result.data['videoId'] as String;
      
      // Upload video to Bunny Stream
      final formData = FormData.fromMap({
        'file': await MultipartFile.fromFile(
          videoFile.path,
          filename: videoFile.path.split('/').last,
        ),
        'title': title,
      });
      
      await _dio.post(
        uploadUrl,
        data: formData,
        onSendProgress: (sent, total) {
          if (onProgress != null && total > 0) {
            onProgress(sent / total);
          }
        },
        options: Options(
          headers: {
            'Content-Type': 'multipart/form-data',
          },
        ),
      );
      
      return videoId;
    } catch (e) {
      throw Exception('Video upload failed: $e');
    }
  }
  
  // Get signed playback URL
  Future<String> getSignedPlaybackUrl({
    required String videoId,
    required String courseId,
    required String lessonId,
  }) async {
    try {
      final result = await _functions.httpsCallable('getSignedPlaybackUrl').call({
        'videoId': videoId,
        'courseId': courseId,
        'lessonId': lessonId,
      });
      
      return result.data['playbackUrl'] as String;
    } catch (e) {
      throw Exception('Failed to get playback URL: $e');
    }
  }
}

// Providers
final videoServiceProvider = Provider<VideoService>((ref) => VideoService());
```

#### Video Player Widget
```dart
// lib/widgets/video_player_widget.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:chewie/chewie.dart';
import 'package:video_player/video_player.dart';
import '../services/video_service.dart';

class VideoPlayerWidget extends ConsumerStatefulWidget {
  final String videoId;
  final String courseId;
  final String lessonId;
  final String title;
  final bool autoPlay;
  final bool showControls;
  final void Function(Duration)? onPositionChanged;
  final void Function()? onVideoCompleted;

  const VideoPlayerWidget({
    super.key,
    required this.videoId,
    required this.courseId,
    required this.lessonId,
    required this.title,
    this.autoPlay = false,
    this.showControls = true,
    this.onPositionChanged,
    this.onVideoCompleted,
  });

  @override
  ConsumerState<VideoPlayerWidget> createState() => _VideoPlayerWidgetState();
}

class _VideoPlayerWidgetState extends ConsumerState<VideoPlayerWidget> {
  VideoPlayerController? _videoPlayerController;
  ChewieController? _chewieController;
  bool _isLoading = true;
  String? _error;
  Duration _lastPosition = Duration.zero;

  @override
  void initState() {
    super.initState();
    _initializePlayer();
  }

  @override
  void dispose() {
    _videoPlayerController?.dispose();
    _chewieController?.dispose();
    super.dispose();
  }

  Future<void> _initializePlayer() async {
    try {
      setState(() {
        _isLoading = true;
        _error = null;
      });

      // Get signed playback URL
      final videoService = ref.read(videoServiceProvider);
      final playbackUrl = await videoService.getSignedPlaybackUrl(
        videoId: widget.videoId,
        courseId: widget.courseId,
        lessonId: widget.lessonId,
      );

      // Initialize video player
      _videoPlayerController = VideoPlayerController.networkUrl(
        Uri.parse(playbackUrl),
      );

      await _videoPlayerController!.initialize();

      // Setup Chewie controller
      _chewieController = ChewieController(
        videoPlayerController: _videoPlayerController!,
        autoPlay: widget.autoPlay,
        looping: false,
        showControls: widget.showControls,
        materialProgressColors: ChewieProgressColors(
          playedColor: Theme.of(context).primaryColor,
          handleColor: Theme.of(context).primaryColor,
          bufferedColor: Theme.of(context).primaryColor.withOpacity(0.3),
          backgroundColor: Colors.grey.withOpacity(0.3),
        ),
        placeholder: Container(
          color: Colors.black,
          child: const Center(
            child: CircularProgressIndicator(),
          ),
        ),
        errorBuilder: (context, errorMessage) {
          return Container(
            color: Colors.black,
            child: Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(
                    Icons.error_outline,
                    color: Colors.white,
                    size: 48,
                  ),
                  const SizedBox(height: 16),
                  const Text(
                    'Video playback error',
                    style: TextStyle(color: Colors.white),
                  ),
                  const SizedBox(height: 8),
                  Text(
                    errorMessage,
                    style: const TextStyle(color: Colors.white70, fontSize: 12),
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: _initializePlayer,
                    child: const Text('Retry'),
                  ),
                ],
              ),
            ),
          );
        },
      );

      // Listen to position changes
      _videoPlayerController!.addListener(() {
        final position = _videoPlayerController!.value.position;
        if (position != _lastPosition) {
          _lastPosition = position;
          widget.onPositionChanged?.call(position);
        }

        // Check if video completed
        if (_videoPlayerController!.value.position >= 
            _videoPlayerController!.value.duration) {
          widget.onVideoCompleted?.call();
        }
      });

      setState(() {
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _isLoading = false;
        _error = e.toString();
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return Container(
        height: 200,
        color: Colors.black,
        child: const Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              CircularProgressIndicator(color: Colors.white),
              SizedBox(height: 16),
              Text(
                'Loading video...',
                style: TextStyle(color: Colors.white),
              ),
            ],
          ),
        ),
      );
    }

    if (_error != null) {
      return Container(
        height: 200,
        color: Colors.black,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(
                Icons.error_outline,
                color: Colors.white,
                size: 48,
              ),
              const SizedBox(height: 16),
              const Text(
                'Failed to load video',
                style: TextStyle(color: Colors.white),
              ),
              const SizedBox(height: 8),
              Text(
                _error!,
                style: const TextStyle(color: Colors.white70, fontSize: 12),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: _initializePlayer,
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
      );
    }

    if (_chewieController == null) {
      return Container(
        height: 200,
        color: Colors.black,
        child: const Center(
          child: Text(
            'Video player not available',
            style: TextStyle(color: Colors.white),
          ),
        ),
      );
    }

    return AspectRatio(
      aspectRatio: _videoPlayerController!.value.aspectRatio,
      child: Chewie(controller: _chewieController!),
    );
  }
}
```

### 4. Razorpay Payment Integration

#### Payment Service
```dart
// lib/services/payment_service.dart
import 'dart:convert';
import 'package:razorpay_flutter/razorpay_flutter.dart';
import 'package:cloud_functions/cloud_functions.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class PaymentService {
  final Razorpay _razorpay = Razorpay();
  final FirebaseFunctions _functions = FirebaseFunctions.instance;
  
  PaymentService() {
    _razorpay.on(Razorpay.EVENT_PAYMENT_SUCCESS, _handlePaymentSuccess);
    _razorpay.on(Razorpay.EVENT_PAYMENT_ERROR, _handlePaymentError);
    _razorpay.on(Razorpay.EVENT_EXTERNAL_WALLET, _handleExternalWallet);
  }
  
  // Payment callbacks
  void Function(PaymentSuccessResponse)? onPaymentSuccess;
  void Function(PaymentFailureResponse)? onPaymentError;
  void Function(ExternalWalletResponse)? onExternalWallet;
  
  // Create order and initiate payment
  Future<void> createPayment({
    required String courseId,
    required int amount, // Amount in paisa
    required String currency,
    required String userEmail,
    required String userPhone,
    required String userName,
  }) async {
    try {
      // Create order via Cloud Function
      final result = await _functions.httpsCallable('createRazorpayOrder').call({
        'courseId': courseId,
        'amount': amount,
        'currency': currency,
        'receipt': 'course_${courseId}_${DateTime.now().millisecondsSinceEpoch}',
      });
      
      final orderId = result.data['orderId'] as String;
      final orderAmount = result.data['amount'] as int;
      
      // Razorpay payment options
      final options = {
        'key': 'rzp_test_YOUR_KEY_ID', // Replace with your actual key
        'order_id': orderId,
        'amount': orderAmount,
        'currency': currency,
        'name': 'Gurukul Academy',
        'description': 'Course Purchase',
        'prefill': {
          'contact': userPhone,
          'email': userEmail,
          'name': userName,
        },
        'theme': {
          'color': '#2563eb', // Your primary color
        },
      };
      
      _razorpay.open(options);
    } catch (e) {
      throw Exception('Failed to create payment: $e');
    }
  }
  
  // Handle payment success
  void _handlePaymentSuccess(PaymentSuccessResponse response) {
    onPaymentSuccess?.call(response);
  }
  
  // Handle payment error
  void _handlePaymentError(PaymentFailureResponse response) {
    onPaymentError?.call(response);
  }
  
  // Handle external wallet
  void _handleExternalWallet(ExternalWalletResponse response) {
    onExternalWallet?.call(response);
  }
  
  // Verify payment on server
  Future<bool> verifyPayment({
    required String paymentId,
    required String orderId,
    required String signature,
    required String courseId,
  }) async {
    try {
      final result = await _functions.httpsCallable('verifyPayment').call({
        'paymentId': paymentId,
        'orderId': orderId,
        'signature': signature,
        'courseId': courseId,
      });
      
      return result.data['verified'] as bool;
    } catch (e) {
      throw Exception('Payment verification failed: $e');
    }
  }
  
  void dispose() {
    _razorpay.clear();
  }
}

// Payment state management
enum PaymentStatus {
  initial,
  loading,
  success,
  error,
  externalWallet,
  verifying,
  verified,
}

class PaymentState {
  final PaymentStatus status;
  final String? error;
  final String? paymentId;
  final String? orderId;
  final String? signature;
  
  PaymentState._({
    required this.status,
    this.error,
    this.paymentId,
    this.orderId,
    this.signature,
  });
  
  factory PaymentState.initial() => PaymentState._(status: PaymentStatus.initial);
  factory PaymentState.loading() => PaymentState._(status: PaymentStatus.loading);
  factory PaymentState.success({
    required String paymentId,
    required String orderId,
    required String signature,
  }) => PaymentState._(
    status: PaymentStatus.success,
    paymentId: paymentId,
    orderId: orderId,
    signature: signature,
  );
  factory PaymentState.error(String error) => PaymentState._(
    status: PaymentStatus.error,
    error: error,
  );
  factory PaymentState.verifying() => PaymentState._(status: PaymentStatus.verifying);
  factory PaymentState.verified() => PaymentState._(status: PaymentStatus.verified);
}

// Provider
final paymentServiceProvider = Provider<PaymentService>((ref) => PaymentService());
```

### 5. Firebase Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }
    
    function getUserRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    
    function isAdmin() {
      return isAuthenticated() && getUserRole() == 'admin';
    }
    
    function isTeacher() {
      return isAuthenticated() && getUserRole() == 'teacher';
    }
    
    function isStudent() {
      return isAuthenticated() && getUserRole() == 'student';
    }
    
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }
    
    function isCourseTeacher(teacherId) {
      return isAuthenticated() && request.auth.uid == teacherId;
    }
    
    function isEnrolled(courseId) {
      return exists(/databases/$(database)/documents/enrollments/$(request.auth.uid + '_' + courseId));
    }

    // Users collection
    match /users/{userId} {
      // Users can read their own profile, admins can read all
      allow read: if isOwner(userId) || isAdmin();
      
      // Users can create their own profile during signup
      allow create: if isOwner(userId) && 
        request.resource.data.keys().hasAll(['phone', 'name', 'role']) &&
        request.resource.data.role in ['student', 'teacher'];
      
      // Users can update their own profile (except role), admins can update all
      allow update: if (isOwner(userId) && 
        !('role' in request.resource.data.diff(resource.data).affectedKeys())) ||
        isAdmin();
      
      // Only admins can delete users
      allow delete: if isAdmin();
    }

    // Courses collection
    match /courses/{courseId} {
      // All authenticated users can read published courses
      allow read: if isAuthenticated() && resource.data.isPublished == true;
      
      // Teachers can read their own courses, admins can read all
      allow read: if isCourseTeacher(resource.data.teacherId) || isAdmin();
      
      // Only teachers can create courses
      allow create: if isTeacher() && 
        request.resource.data.teacherId == request.auth.uid;
      
      // Teachers can update their own courses, admins can update all
      allow update: if isCourseTeacher(resource.data.teacherId) || isAdmin();
      
      // Only admins can delete courses
      allow delete: if isAdmin();
      
      // Lessons subcollection
      match /lessons/{lessonId} {
        // Students can read lessons if enrolled, teachers can read their course lessons
        allow read: if (isStudent() && isEnrolled(courseId)) ||
          isCourseTeacher(get(/databases/$(database)/documents/courses/$(courseId)).data.teacherId) ||
          isAdmin();
        
        // Only course teacher can create/update/delete lessons
        allow write: if isCourseTeacher(get(/databases/$(database)/documents/courses/$(courseId)).data.teacherId) || isAdmin();
      }
    }

    // Enrollments collection
    match /enrollments/{enrollmentId} {
      // Users can read their own enrollments
      allow read: if isOwner(resource.data.userId) || isAdmin();
      
      // Enrollments are created via Cloud Functions after payment
      allow create: if false;
      
      // Users can update their own enrollment progress
      allow update: if isOwner(resource.data.userId) && 
        request.resource.data.diff(resource.data).affectedKeys().hasOnly(['progress', 'downloads']);
      
      // Only admins can delete enrollments
      allow delete: if isAdmin();
    }

    // Payments collection
    match /payments/{paymentId} {
      // Users can read their own payments, admins can read all
      allow read: if isOwner(resource.data.userId) || isAdmin();
      
      // Payments are created via Cloud Functions
      allow create, update, delete: if false;
    }
  }
}
```

### 6. Cloud Functions Examples

```javascript
// functions/src/payments.js
const functions = require('firebase-functions');
const admin = require('firebase-admin');
const Razorpay = require('razorpay');
const crypto = require('crypto');

// Initialize Razorpay
const razorpay = new Razorpay({
  key_id: functions.config().razorpay.key_id,
  key_secret: functions.config().razorpay.key_secret,
});

// Create Razorpay order
exports.createRazorpayOrder = functions.https.onCall(async (data, context) => {
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
  }

  const { courseId, amount, currency, receipt } = data;

  try {
    // Get course details
    const courseDoc = await admin.firestore().collection('courses').doc(courseId).get();
    if (!courseDoc.exists) {
      throw new functions.https.HttpsError('not-found', 'Course not found');
    }

    const course = courseDoc.data();
    
    // Verify amount matches course price
    if (amount !== course.price) {
      throw new functions.https.HttpsError('invalid-argument', 'Amount mismatch');
    }

    // Create Razorpay order
    const order = await razorpay.orders.create({
      amount: amount,
      currency: currency,
      receipt: receipt,
      payment_capture: 1,
    });

    // Store order in Firestore
    await admin.firestore().collection('payments').doc(order.id).set({
      paymentId: null,
      razorpayOrderId: order.id,
      userId: context.auth.uid,
      courseId: courseId,
      amount: amount,
      currency: currency,
      status: 'created',
      description: `${course.title} Course Purchase`,
      receipt: receipt,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    return {
      orderId: order.id,
      amount: order.amount,
      currency: order.currency,
    };
  } catch (error) {
    console.error('Error creating order:', error);
    throw new functions.https.HttpsError('internal', 'Failed to create order');
  }
});

// Verify payment
exports.verifyPayment = functions.https.onCall(async (data, context) => {
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
  }

  const { paymentId, orderId, signature, courseId } = data;

  try {
    // Verify signature
    const generatedSignature = crypto
      .createHmac('sha256', functions.config().razorpay.key_secret)
      .update(`${orderId}|${paymentId}`)
      .digest('hex');

    if (generatedSignature !== signature) {
      throw new functions.https.HttpsError('invalid-argument', 'Invalid signature');
    }

    // Update payment record
    await admin.firestore().collection('payments').doc(orderId).update({
      paymentId: paymentId,
      status: 'captured',
      signature: signature,
      capturedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    // Create enrollment
    const enrollmentId = `${context.auth.uid}_${courseId}`;
    await admin.firestore().collection('enrollments').doc(enrollmentId).set({
      enrollmentId: enrollmentId,
      userId: context.auth.uid,
      courseId: courseId,
      status: 'active',
      enrolledAt: admin.firestore.FieldValue.serverTimestamp(),
      paymentId: paymentId,
      progress: {
        lessonsCompleted: 0,
        totalLessons: 0,
        percentageComplete: 0,
        lastAccessedLesson: null,
        totalWatchTime: 0,
      },
    });

    return { verified: true };
  } catch (error) {
    console.error('Error verifying payment:', error);
    throw new functions.https.HttpsError('internal', 'Payment verification failed');
  }
});
```

### 7. Sequence Diagrams

#### Course Purchase Flow
```
Student -> Flutter App: Select Course
Flutter App -> Cloud Function: Create Razorpay Order
Cloud Function -> Razorpay: Create Order
Razorpay -> Cloud Function: Order ID
Cloud Function -> Flutter App: Order Details
Flutter App -> Razorpay SDK: Open Payment
Razorpay SDK -> Payment Gateway: Process Payment
Payment Gateway -> Razorpay: Payment Success
Razorpay -> Flutter App: Payment Response
Flutter App -> Cloud Function: Verify Payment
Cloud Function -> Razorpay: Verify Signature
Razorpay -> Cloud Function: Verification Result
Cloud Function -> Firestore: Create Enrollment
Cloud Function -> Flutter App: Verification Success
Flutter App -> Student: Access Course Content
```

#### Video Playback Flow
```
Student -> Flutter App: Request Video
Flutter App -> Cloud Function: Get Signed URL
Cloud Function -> Bunny Stream: Generate Signed URL
Bunny Stream -> Cloud Function: Signed URL
Cloud Function -> Flutter App: Video URL
Flutter App -> Video Player: Load Video
Video Player -> Bunny Stream: Stream Video
Bunny Stream -> Video Player: Video Content
Video Player -> Student: Display Video
```

This implementation guide provides practical code examples and detailed integration steps for building The Gurukul Academy Flutter e-learning platform with all the specified features and security requirements.