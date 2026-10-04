# Clean Architecture — Code Showcase

> Real code examples from the **Inbox** and **Auth** features of the Dr. Ahmed Soliman Cardiology Clinics Dashboard (Flutter/Dart + Firebase).

---

## Why Clean Architecture?

The dashboard is a **medical-grade** web application that manages PHI (Protected Health Information). Clean Architecture enforces:

- **Independence**: UI and framework changes don't touch business logic.
- **Testability**: every layer can be unit-tested in isolation.
- **PHI safety**: the cache layer and data-source layer are the only places that touch raw Firestore data — the domain layer never sees `Map<String, dynamic>`.

---

## Layer Map

```
┌──────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                      │
│   Screen → Cubit (State) → UseCase calls                 │
├──────────────────────────────────────────────────────────┤
│                    DOMAIN LAYER                          │
│   Entities (pure Dart) │ UseCases │ Repository interface │
├──────────────────────────────────────────────────────────┤
│                     DATA LAYER                           │
│   RepositoryImpl │ RemoteDataSource │ Models (fromJson)  │
└──────────────────────────────────────────────────────────┘
```

Dependencies always point **inward**: Data → Domain ← Presentation. The domain layer has zero imports from Flutter or Firebase.

---

## Feature 1: Inbox (Unified WhatsApp Inbox)

### 1.1 Domain — Entity

The `ConversationPreview` entity is a **pure Dart class** — no Firebase, no JSON, no Flutter.

```dart
// dashboard/lib/features/inbox/domain/entities/conversation_preview.dart

/// AI context derived from session state — no LLM call involved.
class AiContext {
  final String intent;
  final String? aiActionBookingDate;
  final String? aiActionBookingTime;
  final bool escalated;

  const AiContext({
    this.intent = '',
    this.aiActionBookingDate,
    this.aiActionBookingTime,
    this.escalated = false,
  });
}

/// Represents a conversation summary shown in the inbox sidebar list.
class ConversationPreview {
  final String patientId;
  final String name;
  final String phone;
  final String lastMessage;
  final String? lastMessageType; // 'text' | 'image' | 'audio' | 'document'
  final DateTime? lastMessageAt;
  final int unreadCount;
  final String channel; // 'WhatsApp'
  final String? channelId;
  final bool aiEnabled;
  final String? activeIntent; // e.g. 'staff_handoff'
  final AiContext? aiContext;
  final DateTime? lastInboundAt;

  const ConversationPreview({
    required this.patientId,
    required this.name,
    required this.phone,
    required this.lastMessage,
    this.lastMessageType,
    required this.lastMessageAt,
    required this.unreadCount,
    required this.channel,
    this.channelId,
    this.aiEnabled = true,
    this.activeIntent,
    this.aiContext,
    this.lastInboundAt,
  });

  /// Returns initials from the patient display name.
  String get initials {
    final parts = name.trim().split(' ');
    if (parts.length >= 2) return '${parts[0][0]}${parts[1][0]}'.toUpperCase();
    if (parts.isNotEmpty && parts[0].isNotEmpty) return parts[0][0].toUpperCase();
    return '?';
  }
}
```

**Key insight**: No `fromJson`, no `toJson` here — those live in the **Model** (data layer).

---

### 1.2 Domain — Repository Interface

The interface lives in the domain layer — it describes *what* can be done, not *how*.

```dart
// dashboard/lib/features/inbox/domain/repositories/inbox_repository.dart

abstract class InboxRepository {
  Stream<Either<Failure, List<ConversationPreview>>> watchConversations({
    int limit = 30,
    String? channelId,
  });

  Stream<Either<Failure, List<Message>>> watchMessages(
    String patientId, {
    required String channelId,
    DateTime? since,
  });

  Future<Either<Failure, Unit>> sendMessage({
    required String patientId,
    required String toPhone,
    required String text,
    required String channelId,
  });

  Future<Either<Failure, Unit>> markAsRead(String patientId, String channelId);

  Future<Either<Failure, Unit>> updateAiState(
    String patientId, String channelId, bool isAiEnabled,
  );

  Future<Either<Failure, Unit>> clearStaffHandoff(
    String patientId, { String? channelId }
  );

  Future<String> getSignedMediaUrl(String storagePath);
  // ... and more
}
```

---

### 1.3 Domain — Use Cases

Use cases are the **only** entry point from Presentation into Domain. They are thin coordinators — no business logic leaks into the cubit.

```dart
// dashboard/lib/features/inbox/domain/usecases/inbox_usecases.dart

@lazySingleton // injectable DI annotation
class InboxUsecases {
  final InboxRepository _repository;

  const InboxUsecases(this._repository);

  // Real-time stream of all conversations
  Stream<Either<Failure, List<ConversationPreview>>> watchConversations({
    int limit = 30,
    String? channelId,
  }) => _repository.watchConversations(limit: limit, channelId: channelId);

  // Real-time stream of messages for a single conversation
  Stream<Either<Failure, List<Message>>> watchMessages(
    String patientId, {
    required String channelId,
    DateTime? since,
  }) => _repository.watchMessages(patientId, channelId: channelId, since: since);

  // Send a text message via Cloud Function
  Future<Either<Failure, Unit>> sendMessage({
    required String patientId,
    required String toPhone,
    required String text,
    required String channelId,
  }) => _repository.sendMessage(
    patientId: patientId, toPhone: toPhone, text: text, channelId: channelId,
  );

  // Toggle AI on/off for a conversation
  Future<Either<Failure, Unit>> updateAiState(
    String patientId, String channelId, bool isAiEnabled,
  ) => _repository.updateAiState(patientId, channelId, isAiEnabled);

  // Clear staff handoff flag once a staff member opens the chat
  Future<Either<Failure, Unit>> clearStaffHandoff(
    String patientId, { String? channelId }
  ) => _repository.clearStaffHandoff(patientId, channelId: channelId);
}
```

---

### 1.4 Data — Remote DataSource Interface

The data source interface sits **inside the data layer** — it is the contract between `RepositoryImpl` and the Firebase/API implementation.

```dart
// dashboard/lib/features/inbox/data/datasources/remote/inbox_remote_data_source.dart

abstract class InboxRemoteDataSource {
  /// Real-time stream of conversations, ordered by lastMessageAt desc.
  Stream<List<ConversationPreviewModel>> watchConversations({
    int limit = 30,
    String? channelId,
  });

  /// Real-time stream of messages for a patient in a specific channel.
  Stream<List<MessageModel>> watchMessages(
    String patientId, {
    required String channelId,
    int limit = 50,
    DateTime? since,
  });

  /// Calls the sendOutboundMessage Cloud Function.
  Future<void> sendMessage({
    required String patientId,
    required String toPhone,
    required String text,
    required String channelId,
  });

  /// Calls the markConversationAsRead Cloud Function.
  Future<void> markAsRead(String patientId, String channelId);

  /// Updates aiEnabled on the Firestore conversation document.
  Future<void> updateAiState(String patientId, String channelId, bool isAiEnabled);

  /// Calls getMediaSignedUrl Cloud Function — never returns a public URL for PHI.
  Future<String> getSignedMediaUrl(String storagePath);
}
```

---

### 1.5 Data — Repository Implementation

The implementation translates between `Model` (data) and `Entity` (domain) and wraps everything in `Either<Failure, T>`.

```dart
// dashboard/lib/features/inbox/data/repositories/inbox_repository_impl.dart

@LazySingleton(as: InboxRepository)
class InboxRepositoryImpl implements InboxRepository {
  final InboxRemoteDataSource _remote;
  final InboxLocalDataSource _local;

  InboxRepositoryImpl(this._remote, this._local);

  @override
  Stream<Either<Failure, List<ConversationPreview>>> watchConversations({
    int limit = 30,
    String? channelId,
  }) {
    return _remote
        .watchConversations(limit: limit, channelId: channelId)
        .map((models) => Right<Failure, List<ConversationPreview>>(
              models.map((m) => m.toEntity()).toList(),
            ))
        .handleError((e) => Left<Failure, List<ConversationPreview>>(
              ServerFailure(e.toString()),
            ));
  }

  @override
  Future<Either<Failure, Unit>> sendMessage({
    required String patientId,
    required String toPhone,
    required String text,
    required String channelId,
  }) async {
    try {
      await _remote.sendMessage(
        patientId: patientId,
        toPhone: toPhone,
        text: text,
        channelId: channelId,
      );
      return const Right(unit);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}
```

---

### 1.6 Presentation — Cubit (State Management)

The cubit consumes use cases and emits typed states. It never touches Firestore or Firebase directly.

```dart
// dashboard/lib/features/inbox/presentation/cubit/inbox_cubit.dart

@injectable
class InboxCubit extends Cubit<InboxState> {
  final InboxUsecases _usecases;
  StreamSubscription<Either<Failure, List<ConversationPreview>>>? _conversationSub;

  InboxCubit(this._usecases) : super(InboxState.initial());

  void watchConversations({String? channelId}) {
    _conversationSub?.cancel();
    _conversationSub = _usecases
        .watchConversations(channelId: channelId)
        .listen((result) {
          result.fold(
            (failure) => emit(state.copyWith(error: failure.message)),
            (conversations) => emit(state.copyWith(conversations: conversations)),
          );
        });
  }

  Future<void> sendMessage({
    required String patientId,
    required String toPhone,
    required String text,
    required String channelId,
  }) async {
    final result = await _usecases.sendMessage(
      patientId: patientId,
      toPhone: toPhone,
      text: text,
      channelId: channelId,
    );
    result.fold(
      (failure) => emit(state.copyWith(sendError: failure.message)),
      (_) => null, // Firestore stream will pick up the new message
    );
  }

  @override
  Future<void> close() {
    _conversationSub?.cancel();
    return super.close();
  }
}
```

---

## Feature 2: Auth (Role-Based Access Control)

### Auth Entity

```dart
// dashboard/lib/features/auth/domain/entities/staff_user.dart

enum StaffRole { physician, leadAssistant, staffAssistant }

class StaffUser {
  final String uid;
  final String email;
  final String displayName;
  final StaffRole role;

  const StaffUser({
    required this.uid,
    required this.email,
    required this.displayName,
    required this.role,
  });

  bool get canManageStaff => role == StaffRole.physician;
}
```

### Auth Use Case (RBAC)

```dart
// dashboard/lib/features/auth/domain/usecases/auth_usecases.dart

@lazySingleton
class AuthUsecases {
  final AuthRepository _repository;

  const AuthUsecases(this._repository);

  Future<Either<Failure, StaffUser>> signInWithEmailAndPassword(
    String email, String password,
  ) => _repository.signIn(email, password);

  Stream<StaffUser?> watchAuthState() => _repository.watchAuthState();

  Future<Either<Failure, Unit>> signOut() => _repository.signOut();
}
```

The role is **never stored locally** — it is always derived from the Firebase Auth custom claim in the ID token (`request.auth.token.role`) so it cannot be spoofed by a client.

---

## Dependency Injection

All use cases and data sources are registered automatically via `injectable` + `get_it`:

```dart
// dashboard/lib/core/di/injection.dart

@InjectableInit()
void configureDependencies() => getIt.init();
```

Annotations used:
- `@lazySingleton` — created once on first use
- `@injectable` — new instance per injection
- `@LazySingleton(as: InboxRepository)` — register implementation for interface

---

## Folder Structure (per feature)

```
features/
└── inbox/
    ├── domain/
    │   ├── entities/         # Pure Dart — no Flutter, no Firebase
    │   │   ├── conversation_preview.dart
    │   │   └── message.dart
    │   ├── repositories/     # Abstract interface
    │   │   └── inbox_repository.dart
    │   └── usecases/         # Business logic coordinators
    │       └── inbox_usecases.dart
    ├── data/
    │   ├── datasources/
    │   │   ├── remote/       # Firestore + Cloud Functions calls
    │   │   │   ├── inbox_remote_data_source.dart        (interface)
    │   │   │   └── inbox_remote_data_source_impl.dart   (Firestore + http)
    │   │   └── local/        # Hive encrypted cache
    │   │       └── inbox_local_data_source.dart
    │   ├── models/           # fromJson / toJson + toEntity()
    │   │   ├── conversation_preview_model.dart
    │   │   └── message_model.dart
    │   └── repositories/     # Implements domain interface
    │       └── inbox_repository_impl.dart
    └── presentation/
        ├── cubit/            # State management (BLoC/Cubit)
        │   └── inbox_cubit.dart
        └── widgets/          # Pure UI components
```
