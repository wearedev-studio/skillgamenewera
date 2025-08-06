# API Specifications and Endpoints

## Overview

This document defines the comprehensive REST API specifications for the gaming platform backend. The API follows RESTful principles with proper HTTP status codes, error handling, and authentication mechanisms.

## Base Configuration

- **Base URL**: `https://api.gaming-platform.com/v1`
- **Content-Type**: `application/json`
- **Authentication**: JWT Bearer tokens
- **Rate Limiting**: Implemented per endpoint
- **API Versioning**: URL-based versioning (`/v1/`)

## Authentication Flow

### JWT Token Structure
```typescript
interface JWTPayload {
  userId: string;
  username: string;
  email: string;
  role: 'user' | 'admin' | 'moderator';
  isVerified: boolean;
  iat: number;
  exp: number;
}

interface RefreshTokenPayload {
  userId: string;
  tokenId: string;
  iat: number;
  exp: number;
}
```

## API Endpoints

### 1. Authentication Endpoints

#### POST /auth/register
Register a new user account.

**Request Body:**
```typescript
{
  username: string;     // 3-20 chars, alphanumeric + underscore
  email: string;        // Valid email format
  password: string;     // Min 8 chars, must include uppercase, lowercase, number
  language?: 'en' | 'ru'; // Default: 'en'
}
```

**Response (201):**
```typescript
{
  success: true;
  message: string;
  data: {
    user: {
      id: string;
      username: string;
      email: string;
      isVerified: boolean;
      createdAt: string;
    };
  };
}
```

**Errors:**
- `400` - Validation errors
- `409` - Username or email already exists
- `429` - Too many registration attempts

#### POST /auth/login
Authenticate user and return tokens.

**Request Body:**
```typescript
{
  login: string;        // Username or email
  password: string;
  rememberMe?: boolean; // Extends refresh token expiry
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    user: UserProfile;
    tokens: {
      accessToken: string;  // 15 minutes expiry
      refreshToken: string; // 7 days (30 days if rememberMe)
    };
  };
}
```

**Errors:**
- `400` - Invalid credentials
- `401` - Account not verified
- `423` - Account locked due to failed attempts
- `429` - Too many login attempts

#### POST /auth/refresh
Refresh access token using refresh token.

**Request Body:**
```typescript
{
  refreshToken: string;
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    accessToken: string;
    refreshToken: string; // New refresh token
  };
}
```

#### POST /auth/logout
Invalidate current session.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Logged out successfully";
}
```

#### POST /auth/forgot-password
Request password reset.

**Request Body:**
```typescript
{
  email: string;
}
```

**Response (200):**
```typescript
{
  success: true;
  message: "Password reset email sent";
}
```

#### POST /auth/reset-password
Reset password with token.

**Request Body:**
```typescript
{
  token: string;
  newPassword: string;
}
```

**Response (200):**
```typescript
{
  success: true;
  message: "Password reset successfully";
}
```

#### GET /auth/verify-email/:token
Verify email address.

**Response (200):**
```typescript
{
  success: true;
  message: "Email verified successfully";
}
```

### 2. User Management Endpoints

#### GET /users/profile
Get current user profile.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    user: UserProfile;
  };
}
```

#### PUT /users/profile
Update user profile.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  username?: string;
  email?: string;
  preferences?: {
    language?: 'en' | 'ru';
    notifications?: NotificationPreferences;
    privacy?: PrivacySettings;
  };
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    user: UserProfile;
  };
}
```

#### POST /users/upload-avatar
Upload user avatar.

**Headers:** 
- `Authorization: Bearer <token>`
- `Content-Type: multipart/form-data`

**Request Body:**
```typescript
{
  avatar: File; // Max 5MB, jpg/png/gif
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    avatarUrl: string;
  };
}
```

#### PUT /users/change-password
Change user password.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  currentPassword: string;
  newPassword: string;
  confirmPassword: string;
}
```

**Response (200):**
```typescript
{
  success: true;
  message: "Password changed successfully";
}
```

#### GET /users/statistics
Get user game statistics.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `gameType?: string` - Filter by game type
- `period?: string` - 'week' | 'month' | 'year' | 'all'

**Response (200):**
```typescript
{
  success: true;
  data: {
    statistics: UserStatistics;
  };
}
```

#### GET /users/game-history
Get user game history.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `page?: number` - Default: 1
- `limit?: number` - Default: 10, Max: 50
- `gameType?: string` - Filter by game type
- `result?: string` - 'win' | 'loss' | 'draw'

**Response (200):**
```typescript
{
  success: true;
  data: {
    games: GameHistoryItem[];
    pagination: {
      page: number;
      limit: number;
      total: number;
      totalPages: number;
    };
  };
}
```

#### GET /users/transaction-history
Get user transaction history.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `page?: number` - Default: 1
- `limit?: number` - Default: 10, Max: 50
- `type?: string` - Filter by transaction type
- `status?: string` - Filter by status

**Response (200):**
```typescript
{
  success: true;
  data: {
    transactions: TransactionHistoryItem[];
    pagination: PaginationInfo;
  };
}
```

### 3. KYC Management Endpoints

#### POST /kyc/submit
Submit KYC documents.

**Headers:** 
- `Authorization: Bearer <token>`
- `Content-Type: multipart/form-data`

**Request Body:**
```typescript
{
  documents: File[]; // Max 4 files, 10MB each
  documentTypes: string[]; // Corresponding types for each file
}
```

**Response (200):**
```typescript
{
  success: true;
  message: "KYC documents submitted successfully";
  data: {
    submissionId: string;
    status: 'pending';
  };
}
```

#### GET /kyc/status
Get KYC verification status.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    status: 'not_submitted' | 'pending' | 'approved' | 'rejected';
    submittedAt?: string;
    reviewedAt?: string;
    rejectionReason?: string;
    documents: KYCDocument[];
  };
}
```

### 4. Financial Endpoints

#### GET /financial/balance
Get user balance.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    balances: {
      main: number;
      internal: number;
    };
  };
}
```

#### POST /financial/deposit
Initiate deposit transaction.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  amount: number;
  method: 'card' | 'bank_transfer' | 'crypto' | 'paypal';
  currency?: string; // Default: USD
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    transactionId: string;
    paymentUrl?: string; // For external payment processing
    status: 'pending';
  };
}
```

#### POST /financial/withdraw
Request withdrawal.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  amount: number;
  method: 'card' | 'bank_transfer' | 'crypto' | 'paypal';
  destination: {
    // Method-specific destination details
    cardNumber?: string;
    bankAccount?: string;
    cryptoAddress?: string;
    paypalEmail?: string;
  };
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    transactionId: string;
    status: 'pending';
    estimatedProcessingTime: string;
  };
}
```

### 5. Game Lobby Endpoints

#### GET /lobby/rooms
Get available game rooms.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `gameType?: string` - Filter by game type
- `minStake?: number` - Minimum stake filter
- `maxStake?: number` - Maximum stake filter
- `page?: number` - Default: 1
- `limit?: number` - Default: 20

**Response (200):**
```typescript
{
  success: true;
  data: {
    rooms: GameRoom[];
    pagination: PaginationInfo;
  };
}
```

#### POST /lobby/create-room
Create a new game room.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  name: string;
  gameType: 'chess' | 'checkers' | 'backgammon' | 'tictactoe';
  stake: number;
  isPrivate?: boolean;
  password?: string;
  timeControl?: {
    initialTime: number;
    increment: number;
  };
  maxSpectators?: number;
}
```

**Response (201):**
```typescript
{
  success: true;
  data: {
    room: GameRoom;
  };
}
```

#### POST /lobby/join-room/:roomId
Join an existing room.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  password?: string; // Required for private rooms
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    room: GameRoom;
    gameId?: string; // If game starts immediately
  };
}
```

#### DELETE /lobby/leave-room/:roomId
Leave a room.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Left room successfully";
}
```

### 6. Game Management Endpoints

#### GET /games/:gameId
Get game details.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    game: GameDetails;
  };
}
```

#### POST /games/:gameId/move
Make a game move.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  move: GameMove; // Game-specific move format
  timeLeft?: number; // Remaining time after move
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    gameState: GameState;
    isGameFinished: boolean;
    result?: GameResult;
  };
}
```

#### POST /games/:gameId/resign
Resign from game.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    result: GameResult;
  };
}
```

#### POST /games/:gameId/offer-draw
Offer a draw.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Draw offer sent";
}
```

#### POST /games/:gameId/accept-draw
Accept draw offer.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    result: GameResult;
  };
}
```

#### POST /games/:gameId/decline-draw
Decline draw offer.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Draw offer declined";
}
```

#### POST /games/:gameId/offer-revenge
Offer revenge game.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    revengeOfferId: string;
    expiresAt: string;
  };
}
```

#### POST /games/:gameId/accept-revenge
Accept revenge offer.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    newGameId: string;
  };
}
```

### 7. Tournament Endpoints

#### GET /tournaments
Get available tournaments.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `status?: string` - Filter by status
- `gameType?: string` - Filter by game type
- `page?: number` - Default: 1
- `limit?: number` - Default: 10

**Response (200):**
```typescript
{
  success: true;
  data: {
    tournaments: Tournament[];
    pagination: PaginationInfo;
  };
}
```

#### GET /tournaments/:tournamentId
Get tournament details.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    tournament: TournamentDetails;
  };
}
```

#### POST /tournaments/:tournamentId/join
Join a tournament.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    tournament: TournamentDetails;
    position: number; // Registration position
  };
}
```

#### DELETE /tournaments/:tournamentId/leave
Leave a tournament (before it starts).

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Left tournament successfully";
}
```

#### GET /tournaments/:tournamentId/bracket
Get tournament bracket.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    bracket: TournamentBracket;
  };
}
```

### 8. Bot Management Endpoints

#### GET /bots
Get available bots for matchmaking.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `gameType?: string` - Filter by game type
- `difficulty?: string` - Filter by difficulty
- `stakeRange?: string` - Filter by preferred stake range

**Response (200):**
```typescript
{
  success: true;
  data: {
    bots: BotInfo[];
  };
}
```

#### POST /bots/challenge/:botId
Challenge a specific bot.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```typescript
{
  gameType: 'chess' | 'checkers' | 'backgammon' | 'tictactoe';
  stake: number;
  timeControl?: {
    initialTime: number;
    increment: number;
  };
}
```

**Response (200):**
```typescript
{
  success: true;
  data: {
    gameId: string;
    roomId: string;
  };
}
```

### 9. Notification Endpoints

#### GET /notifications
Get user notifications.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `page?: number` - Default: 1
- `limit?: number` - Default: 20
- `unreadOnly?: boolean` - Default: false

**Response (200):**
```typescript
{
  success: true;
  data: {
    notifications: Notification[];
    unreadCount: number;
    pagination: PaginationInfo;
  };
}
```

#### PUT /notifications/:notificationId/read
Mark notification as read.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "Notification marked as read";
}
```

#### PUT /notifications/mark-all-read
Mark all notifications as read.

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```typescript
{
  success: true;
  message: "All notifications marked as read";
}
```

### 10. Admin/CRM Endpoints

#### GET /admin/users
Get users list (Admin only).

**Headers:** `Authorization: Bearer <admin_token>`

**Query Parameters:**
- `page?: number` - Default: 1
- `limit?: number` - Default: 20
- `search?: string` - Search by username/email
- `status?: string` - Filter by status
- `role?: string` - Filter by role

**Response (200):**
```typescript
{
  success: true;
  data: {
    users: AdminUserInfo[];
    pagination: PaginationInfo;
  };
}
```

#### PUT /admin/users/:userId/ban
Ban/unban user.

**Headers:** `Authorization: Bearer <admin_token>`

**Request Body:**
```typescript
{
  banned: boolean;
  reason?: string;
  duration?: number; // Hours, 0 for permanent
}
```

#### GET /admin/kyc/pending
Get pending KYC submissions.

**Headers:** `Authorization: Bearer <admin_token>`

**Response (200):**
```typescript
{
  success: true;
  data: {
    submissions: KYCSubmission[];
  };
}
```

#### PUT /admin/kyc/:submissionId/review
Review KYC submission.

**Headers:** `Authorization: Bearer <admin_token>`

**Request Body:**
```typescript
{
  status: 'approved' | 'rejected';
  notes?: string;
  rejectionReason?: string;
}
```

#### POST /admin/tournaments
Create tournament.

**Headers:** `Authorization: Bearer <admin_token>`

**Request Body:**
```typescript
{
  name: string;
  description?: string;
  gameType: GameType;
  maxPlayers: number;
  entryFee: number;
  prizeDistribution: PrizeDistribution[];
  startTime?: string;
  settings: TournamentSettings;
}
```

#### GET /admin/analytics
Get platform analytics.

**Headers:** `Authorization: Bearer <admin_token>`

**Query Parameters:**
- `period?: string` - 'day' | 'week' | 'month' | 'year'
- `startDate?: string` - ISO date string
- `endDate?: string` - ISO date string

**Response (200):**
```typescript
{
  success: true;
  data: {
    analytics: PlatformAnalytics;
  };
}
```

## Error Handling

### Standard Error Response Format
```typescript
{
  success: false;
  error: {
    code: string;
    message: string;
    details?: any;
    timestamp: string;
    requestId: string;
  };
}
```

### Common Error Codes
- `VALIDATION_ERROR` - Input validation failed
- `AUTHENTICATION_REQUIRED` - No valid token provided
- `AUTHORIZATION_FAILED` - Insufficient permissions
- `RESOURCE_NOT_FOUND` - Requested resource doesn't exist
- `RESOURCE_CONFLICT` - Resource already exists or conflict
- `RATE_LIMIT_EXCEEDED` - Too many requests
- `INSUFFICIENT_BALANCE` - Not enough funds
- `GAME_NOT_FOUND` - Game doesn't exist
- `INVALID_MOVE` - Move is not valid
- `TOURNAMENT_FULL` - Tournament has reached capacity
- `KYC_REQUIRED` - KYC verification required
- `ACCOUNT_LOCKED` - Account is temporarily locked
- `MAINTENANCE_MODE` - System under maintenance

## Rate Limiting

### Rate Limit Headers
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
```

### Rate Limits by Endpoint Category
- **Authentication**: 5 requests per minute
- **Game Moves**: 1 request per second
- **General API**: 100 requests per minute
- **File Uploads**: 10 requests per hour
- **Admin Operations**: 1000 requests per hour

## WebSocket Events (Socket.io)

### Connection Authentication
```typescript
// Client connects with JWT token
socket.auth = {
  token: 'jwt_token_here'
};
```

### Game Events
```typescript
// Client to Server
socket.emit('game:join', { gameId: string });
socket.emit('game:move', { gameId: string, move: GameMove });
socket.emit('game:offer-draw', { gameId: string });

// Server to Client
socket.on('game:state-update', (gameState: GameState) => {});
socket.on('game:move-made', (move: GameMove) => {});
socket.on('game:game-ended', (result: GameResult) => {});
```

### Lobby Events
```typescript
// Client to Server
socket.emit('lobby:join', { gameType: GameType });
socket.emit('lobby:create-room', roomConfig);

// Server to Client
socket.on('lobby:room-list', (rooms: Room[]) => {});
socket.on('lobby:player-joined', (player: Player) => {});
```

### Tournament Events
```typescript
// Server to Client
socket.on('tournament:bracket-update', (bracket: TournamentBracket) => {});
socket.on('tournament:match-started', (match: TournamentMatch) => {});
socket.on('tournament:round-completed', (roundInfo: RoundInfo) => {});
```

This comprehensive API specification provides the foundation for building a robust, scalable gaming platform with proper authentication, real-time features, and administrative capabilities.