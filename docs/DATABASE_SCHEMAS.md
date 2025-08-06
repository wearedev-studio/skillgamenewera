# Database Schemas and Data Models

## Overview

This document defines the comprehensive database schemas for the gaming platform using MongoDB with Mongoose ODM. All schemas include proper indexing, validation, and relationships for optimal performance and data integrity.

## Core Collections

### 1. Users Collection

```typescript
import { Schema, model, Document } from 'mongoose';

interface IUser extends Document {
  username: string;
  email: string;
  passwordHash: string;
  avatar?: string;
  balances: {
    main: number;
    internal: number;
  };
  statistics: {
    totalGames: number;
    gamesWon: number;
    gamesLost: number;
    gamesDrawn: number;
    hoursPlayed: number;
    moneyEarned: number;
    gameStats: {
      chess: GameStatistics;
      checkers: GameStatistics;
      backgammon: GameStatistics;
      tictactoe: GameStatistics;
    };
  };
  kyc: {
    status: 'pending' | 'approved' | 'rejected' | 'not_submitted';
    documents: Array<{
      type: 'passport' | 'utility_bill' | 'international_passport' | 'residence_permit';
      filename: string;
      originalName: string;
      uploadedAt: Date;
      reviewedAt?: Date;
      reviewedBy?: Schema.Types.ObjectId;
      notes?: string;
    }>;
    submittedAt?: Date;
    reviewedAt?: Date;
    reviewedBy?: Schema.Types.ObjectId;
    rejectionReason?: string;
  };
  preferences: {
    language: 'en' | 'ru';
    notifications: {
      email: boolean;
      push: boolean;
      gameInvites: boolean;
      tournaments: boolean;
      promotions: boolean;
    };
    privacy: {
      showOnline: boolean;
      showStatistics: boolean;
    };
  };
  security: {
    lastLogin?: Date;
    loginAttempts: number;
    lockUntil?: Date;
    passwordResetToken?: string;
    passwordResetExpires?: Date;
    emailVerificationToken?: string;
    emailVerificationExpires?: Date;
    twoFactorEnabled: boolean;
    twoFactorSecret?: string;
  };
  isVerified: boolean;
  isActive: boolean;
  isBanned: boolean;
  banReason?: string;
  banExpiresAt?: Date;
  role: 'user' | 'admin' | 'moderator';
  createdAt: Date;
  updatedAt: Date;
}

interface GameStatistics {
  played: number;
  won: number;
  lost: number;
  drawn: number;
  winRate: number;
  averageGameDuration: number;
  longestWinStreak: number;
  currentWinStreak: number;
}

const userSchema = new Schema<IUser>({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    minlength: 3,
    maxlength: 20,
    match: /^[a-zA-Z0-9_]+$/
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    match: /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  },
  passwordHash: {
    type: String,
    required: true,
    minlength: 60
  },
  avatar: {
    type: String,
    default: null
  },
  balances: {
    main: {
      type: Number,
      default: 0,
      min: 0
    },
    internal: {
      type: Number,
      default: 100, // Starting bonus
      min: 0
    }
  },
  statistics: {
    totalGames: { type: Number, default: 0 },
    gamesWon: { type: Number, default: 0 },
    gamesLost: { type: Number, default: 0 },
    gamesDrawn: { type: Number, default: 0 },
    hoursPlayed: { type: Number, default: 0 },
    moneyEarned: { type: Number, default: 0 },
    gameStats: {
      chess: {
        played: { type: Number, default: 0 },
        won: { type: Number, default: 0 },
        lost: { type: Number, default: 0 },
        drawn: { type: Number, default: 0 },
        winRate: { type: Number, default: 0 },
        averageGameDuration: { type: Number, default: 0 },
        longestWinStreak: { type: Number, default: 0 },
        currentWinStreak: { type: Number, default: 0 }
      },
      checkers: {
        played: { type: Number, default: 0 },
        won: { type: Number, default: 0 },
        lost: { type: Number, default: 0 },
        drawn: { type: Number, default: 0 },
        winRate: { type: Number, default: 0 },
        averageGameDuration: { type: Number, default: 0 },
        longestWinStreak: { type: Number, default: 0 },
        currentWinStreak: { type: Number, default: 0 }
      },
      backgammon: {
        played: { type: Number, default: 0 },
        won: { type: Number, default: 0 },
        lost: { type: Number, default: 0 },
        drawn: { type: Number, default: 0 },
        winRate: { type: Number, default: 0 },
        averageGameDuration: { type: Number, default: 0 },
        longestWinStreak: { type: Number, default: 0 },
        currentWinStreak: { type: Number, default: 0 }
      },
      tictactoe: {
        played: { type: Number, default: 0 },
        won: { type: Number, default: 0 },
        lost: { type: Number, default: 0 },
        drawn: { type: Number, default: 0 },
        winRate: { type: Number, default: 0 },
        averageGameDuration: { type: Number, default: 0 },
        longestWinStreak: { type: Number, default: 0 },
        currentWinStreak: { type: Number, default: 0 }
      }
    }
  },
  kyc: {
    status: {
      type: String,
      enum: ['pending', 'approved', 'rejected', 'not_submitted'],
      default: 'not_submitted'
    },
    documents: [{
      type: {
        type: String,
        enum: ['passport', 'utility_bill', 'international_passport', 'residence_permit'],
        required: true
      },
      filename: { type: String, required: true },
      originalName: { type: String, required: true },
      uploadedAt: { type: Date, default: Date.now },
      reviewedAt: Date,
      reviewedBy: { type: Schema.Types.ObjectId, ref: 'User' },
      notes: String
    }],
    submittedAt: Date,
    reviewedAt: Date,
    reviewedBy: { type: Schema.Types.ObjectId, ref: 'User' },
    rejectionReason: String
  },
  preferences: {
    language: {
      type: String,
      enum: ['en', 'ru'],
      default: 'en'
    },
    notifications: {
      email: { type: Boolean, default: true },
      push: { type: Boolean, default: true },
      gameInvites: { type: Boolean, default: true },
      tournaments: { type: Boolean, default: true },
      promotions: { type: Boolean, default: true }
    },
    privacy: {
      showOnline: { type: Boolean, default: true },
      showStatistics: { type: Boolean, default: true }
    }
  },
  security: {
    lastLogin: Date,
    loginAttempts: { type: Number, default: 0 },
    lockUntil: Date,
    passwordResetToken: String,
    passwordResetExpires: Date,
    emailVerificationToken: String,
    emailVerificationExpires: Date,
    twoFactorEnabled: { type: Boolean, default: false },
    twoFactorSecret: String
  },
  isVerified: { type: Boolean, default: false },
  isActive: { type: Boolean, default: true },
  isBanned: { type: Boolean, default: false },
  banReason: String,
  banExpiresAt: Date,
  role: {
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user'
  }
}, {
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});

// Indexes
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ username: 1 }, { unique: true });
userSchema.index({ 'security.passwordResetToken': 1 });
userSchema.index({ 'security.emailVerificationToken': 1 });
userSchema.index({ isActive: 1, isBanned: 1 });
userSchema.index({ role: 1 });
userSchema.index({ createdAt: -1 });

export const User = model<IUser>('User', userSchema);
```

### 2. Games Collection

```typescript
interface IGame extends Document {
  type: 'chess' | 'checkers' | 'backgammon' | 'tictactoe';
  players: {
    player1: {
      userId: Schema.Types.ObjectId;
      username: string;
      isBot: boolean;
      color?: 'white' | 'black' | 'red' | 'blue';
      ready: boolean;
    };
    player2: {
      userId: Schema.Types.ObjectId;
      username: string;
      isBot: boolean;
      color?: 'white' | 'black' | 'red' | 'blue';
      ready: boolean;
    };
  };
  gameState: {
    board: any; // Game-specific board state
    currentPlayer: 'player1' | 'player2';
    moveHistory: Array<{
      player: 'player1' | 'player2';
      move: any; // Game-specific move data
      timestamp: Date;
      notation?: string; // Human-readable move notation
    }>;
    gameSpecificData?: any; // Additional game-specific state
  };
  status: 'waiting' | 'ready' | 'playing' | 'paused' | 'finished' | 'abandoned';
  result?: {
    winner?: 'player1' | 'player2' | 'draw';
    reason: 'checkmate' | 'resignation' | 'timeout' | 'draw_agreement' | 'stalemate' | 'abandonment';
    finalPosition?: any;
  };
  financial: {
    stake: number;
    totalPot: number;
    commission: number;
    winnings: {
      player1: number;
      player2: number;
    };
  };
  timing: {
    timeControl?: {
      initialTime: number; // seconds
      increment: number; // seconds per move
    };
    timeLeft: {
      player1: number;
      player2: number;
    };
    lastMoveTime?: Date;
  };
  room: {
    id: string;
    name?: string;
    isPrivate: boolean;
    password?: string;
  };
  revenge: {
    offered: boolean;
    offeredBy?: 'player1' | 'player2';
    offeredAt?: Date;
    expiresAt?: Date;
    accepted?: boolean;
  };
  tournament?: {
    tournamentId: Schema.Types.ObjectId;
    round: number;
    matchNumber: number;
  };
  spectators: Array<{
    userId: Schema.Types.ObjectId;
    username: string;
    joinedAt: Date;
  }>;
  startedAt?: Date;
  finishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

const gameSchema = new Schema<IGame>({
  type: {
    type: String,
    enum: ['chess', 'checkers', 'backgammon', 'tictactoe'],
    required: true
  },
  players: {
    player1: {
      userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
      username: { type: String, required: true },
      isBot: { type: Boolean, default: false },
      color: String,
      ready: { type: Boolean, default: false }
    },
    player2: {
      userId: { type: Schema.Types.ObjectId, ref: 'User' },
      username: String,
      isBot: { type: Boolean, default: false },
      color: String,
      ready: { type: Boolean, default: false }
    }
  },
  gameState: {
    board: Schema.Types.Mixed,
    currentPlayer: {
      type: String,
      enum: ['player1', 'player2'],
      default: 'player1'
    },
    moveHistory: [{
      player: {
        type: String,
        enum: ['player1', 'player2'],
        required: true
      },
      move: Schema.Types.Mixed,
      timestamp: { type: Date, default: Date.now },
      notation: String
    }],
    gameSpecificData: Schema.Types.Mixed
  },
  status: {
    type: String,
    enum: ['waiting', 'ready', 'playing', 'paused', 'finished', 'abandoned'],
    default: 'waiting'
  },
  result: {
    winner: {
      type: String,
      enum: ['player1', 'player2', 'draw']
    },
    reason: {
      type: String,
      enum: ['checkmate', 'resignation', 'timeout', 'draw_agreement', 'stalemate', 'abandonment']
    },
    finalPosition: Schema.Types.Mixed
  },
  financial: {
    stake: { type: Number, required: true, min: 0 },
    totalPot: { type: Number, required: true, min: 0 },
    commission: { type: Number, required: true, min: 0 },
    winnings: {
      player1: { type: Number, default: 0 },
      player2: { type: Number, default: 0 }
    }
  },
  timing: {
    timeControl: {
      initialTime: Number,
      increment: Number
    },
    timeLeft: {
      player1: Number,
      player2: Number
    },
    lastMoveTime: Date
  },
  room: {
    id: { type: String, required: true },
    name: String,
    isPrivate: { type: Boolean, default: false },
    password: String
  },
  revenge: {
    offered: { type: Boolean, default: false },
    offeredBy: {
      type: String,
      enum: ['player1', 'player2']
    },
    offeredAt: Date,
    expiresAt: Date,
    accepted: Boolean
  },
  tournament: {
    tournamentId: { type: Schema.Types.ObjectId, ref: 'Tournament' },
    round: Number,
    matchNumber: Number
  },
  spectators: [{
    userId: { type: Schema.Types.ObjectId, ref: 'User' },
    username: String,
    joinedAt: { type: Date, default: Date.now }
  }],
  startedAt: Date,
  finishedAt: Date
}, {
  timestamps: true
});

// Indexes
gameSchema.index({ type: 1, status: 1 });
gameSchema.index({ 'players.player1.userId': 1 });
gameSchema.index({ 'players.player2.userId': 1 });
gameSchema.index({ 'tournament.tournamentId': 1 });
gameSchema.index({ status: 1, createdAt: -1 });
gameSchema.index({ 'room.id': 1 });
gameSchema.index({ createdAt: -1 });

export const Game = model<IGame>('Game', gameSchema);
```

### 3. Tournaments Collection

```typescript
interface ITournament extends Document {
  name: string;
  description?: string;
  gameType: 'chess' | 'checkers' | 'backgammon' | 'tictactoe';
  format: 'single_elimination' | 'double_elimination' | 'round_robin';
  maxPlayers: number;
  minPlayers: number;
  entryFee: number;
  prizePool: {
    total: number;
    distribution: Array<{
      position: number;
      percentage: number;
      amount: number;
    }>;
  };
  status: 'upcoming' | 'registration' | 'starting' | 'active' | 'finished' | 'cancelled';
  participants: Array<{
    userId: Schema.Types.ObjectId;
    username: string;
    isBot: boolean;
    registeredAt: Date;
    eliminated?: boolean;
    eliminatedAt?: Date;
    finalPosition?: number;
  }>;
  bracket: {
    rounds: Array<{
      roundNumber: number;
      matches: Array<{
        matchId: string;
        gameId?: Schema.Types.ObjectId;
        player1?: {
          userId: Schema.Types.ObjectId;
          username: string;
          isBot: boolean;
        };
        player2?: {
          userId: Schema.Types.ObjectId;
          username: string;
          isBot: boolean;
        };
        winner?: 'player1' | 'player2';
        status: 'pending' | 'playing' | 'finished';
        startedAt?: Date;
        finishedAt?: Date;
      }>;
    }>;
  };
  settings: {
    timeControl?: {
      initialTime: number;
      increment: number;
    };
    autoFillWithBots: boolean;
    registrationDeadline?: Date;
    startTime?: Date;
    maxDuration?: number; // hours
  };
  winner?: {
    userId: Schema.Types.ObjectId;
    username: string;
    isBot: boolean;
    prize: number;
  };
  statistics: {
    totalGames: number;
    averageGameDuration: number;
    totalDuration: number;
    humanPlayers: number;
    botPlayers: number;
  };
  createdBy: Schema.Types.ObjectId;
  startedAt?: Date;
  finishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

const tournamentSchema = new Schema<ITournament>({
  name: {
    type: String,
    required: true,
    trim: true,
    maxlength: 100
  },
  description: {
    type: String,
    maxlength: 500
  },
  gameType: {
    type: String,
    enum: ['chess', 'checkers', 'backgammon', 'tictactoe'],
    required: true
  },
  format: {
    type: String,
    enum: ['single_elimination', 'double_elimination', 'round_robin'],
    default: 'single_elimination'
  },
  maxPlayers: {
    type: Number,
    required: true,
    min: 2,
    max: 64
  },
  minPlayers: {
    type: Number,
    required: true,
    min: 2
  },
  entryFee: {
    type: Number,
    required: true,
    min: 0
  },
  prizePool: {
    total: { type: Number, required: true },
    distribution: [{
      position: { type: Number, required: true },
      percentage: { type: Number, required: true },
      amount: { type: Number, required: true }
    }]
  },
  status: {
    type: String,
    enum: ['upcoming', 'registration', 'starting', 'active', 'finished', 'cancelled'],
    default: 'upcoming'
  },
  participants: [{
    userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    username: { type: String, required: true },
    isBot: { type: Boolean, default: false },
    registeredAt: { type: Date, default: Date.now },
    eliminated: Boolean,
    eliminatedAt: Date,
    finalPosition: Number
  }],
  bracket: {
    rounds: [{
      roundNumber: { type: Number, required: true },
      matches: [{
        matchId: { type: String, required: true },
        gameId: { type: Schema.Types.ObjectId, ref: 'Game' },
        player1: {
          userId: { type: Schema.Types.ObjectId, ref: 'User' },
          username: String,
          isBot: Boolean
        },
        player2: {
          userId: { type: Schema.Types.ObjectId, ref: 'User' },
          username: String,
          isBot: Boolean
        },
        winner: {
          type: String,
          enum: ['player1', 'player2']
        },
        status: {
          type: String,
          enum: ['pending', 'playing', 'finished'],
          default: 'pending'
        },
        startedAt: Date,
        finishedAt: Date
      }]
    }]
  },
  settings: {
    timeControl: {
      initialTime: Number,
      increment: Number
    },
    autoFillWithBots: { type: Boolean, default: true },
    registrationDeadline: Date,
    startTime: Date,
    maxDuration: Number
  },
  winner: {
    userId: { type: Schema.Types.ObjectId, ref: 'User' },
    username: String,
    isBot: Boolean,
    prize: Number
  },
  statistics: {
    totalGames: { type: Number, default: 0 },
    averageGameDuration: { type: Number, default: 0 },
    totalDuration: { type: Number, default: 0 },
    humanPlayers: { type: Number, default: 0 },
    botPlayers: { type: Number, default: 0 }
  },
  createdBy: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  startedAt: Date,
  finishedAt: Date
}, {
  timestamps: true
});

// Indexes
tournamentSchema.index({ gameType: 1, status: 1 });
tournamentSchema.index({ status: 1, createdAt: -1 });
tournamentSchema.index({ createdBy: 1 });
tournamentSchema.index({ 'participants.userId': 1 });
tournamentSchema.index({ startedAt: -1 });

export const Tournament = model<ITournament>('Tournament', tournamentSchema);
```

### 4. Transactions Collection

```typescript
interface ITransaction extends Document {
  userId: Schema.Types.ObjectId;
  type: 'deposit' | 'withdrawal' | 'game_win' | 'game_loss' | 'tournament_win' | 'tournament_entry' | 'commission' | 'refund' | 'bonus';
  amount: number;
  balanceType: 'main' | 'internal';
  status: 'pending' | 'processing' | 'completed' | 'failed' | 'cancelled';
  reference?: {
    type: 'game' | 'tournament' | 'payment' | 'manual';
    id: string;
    description?: string;
  };
  payment?: {
    method: 'card' | 'bank_transfer' | 'crypto' | 'paypal';
    provider: string;
    transactionId?: string;
    fees?: number;
  };
  balances: {
    before: {
      main: number;
      internal: number;
    };
    after: {
      main: number;
      internal: number;
    };
  };
  metadata?: {
    gameType?: string;
    opponent?: string;
    tournamentName?: string;
    adminNote?: string;
    ipAddress?: string;
    userAgent?: string;
  };
  processedBy?: Schema.Types.ObjectId;
  failureReason?: string;
  createdAt: Date;
  processedAt?: Date;
  updatedAt: Date;
}

const transactionSchema = new Schema<ITransaction>({
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  type: {
    type: String,
    enum: ['deposit', 'withdrawal', 'game_win', 'game_loss', 'tournament_win', 'tournament_entry', 'commission', 'refund', 'bonus'],
    required: true
  },
  amount: {
    type: Number,
    required: true
  },
  balanceType: {
    type: String,
    enum: ['main', 'internal'],
    required: true
  },
  status: {
    type: String,
    enum: ['pending', 'processing', 'completed', 'failed', 'cancelled'],
    default: 'pending'
  },
  reference: {
    type: {
      type: String,
      enum: ['game', 'tournament', 'payment', 'manual']
    },
    id: String,
    description: String
  },
  payment: {
    method: {
      type: String,
      enum: ['card', 'bank_transfer', 'crypto', 'paypal']
    },
    provider: String,
    transactionId: String,
    fees: Number
  },
  balances: {
    before: {
      main: { type: Number, required: true },
      internal: { type: Number, required: true }
    },
    after: {
      main: { type: Number, required: true },
      internal: { type: Number, required: true }
    }
  },
  metadata: {
    gameType: String,
    opponent: String,
    tournamentName: String,
    adminNote: String,
    ipAddress: String,
    userAgent: String
  },
  processedBy: { type: Schema.Types.ObjectId, ref: 'User' },
  failureReason: String,
  processedAt: Date
}, {
  timestamps: true
});

// Indexes
transactionSchema.index({ userId: 1, createdAt: -1 });
transactionSchema.index({ type: 1, status: 1 });
transactionSchema.index({ status: 1, createdAt: -1 });
transactionSchema.index({ 'reference.type': 1, 'reference.id': 1 });
transactionSchema.index({ balanceType: 1, createdAt: -1 });

export const Transaction = model<ITransaction>('Transaction', transactionSchema);
```

### 5. Rooms Collection

```typescript
interface IRoom extends Document {
  id: string;
  name: string;
  gameType: 'chess' | 'checkers' | 'backgammon' | 'tictactoe';
  creator: {
    userId: Schema.Types.ObjectId;
    username: string;
    isBot: boolean;
  };
  settings: {
    stake: number;
    isPrivate: boolean;
    password?: string;
    maxSpectators: number;
    timeControl?: {
      initialTime: number;
      increment: number;
    };
    allowRevenge: boolean;
  };
  status: 'waiting' | 'full' | 'playing' | 'finished';
  currentGame?: Schema.Types.ObjectId;
  participants: Array<{
    userId: Schema.Types.ObjectId;
    username: string;
    isBot: boolean;
    joinedAt: Date;
    ready: boolean;
  }>;
  spectators: Array<{
    userId: Schema.Types.ObjectId;
    username: string;
    joinedAt: Date;
  }>;
  gameHistory: Array<{
    gameId: Schema.Types.ObjectId;
    winner?: 'player1' | 'player2' | 'draw';
    finishedAt: Date;
  }>;
  createdAt: Date;
  updatedAt: Date;
}

const roomSchema = new Schema<IRoom>({
  id: {
    type: String,
    required: true,
    unique: true
  },
  name: {
    type: String,
    required: true,
    trim: true,
    maxlength: 50
  },
  gameType: {
    type: String,
    enum: ['chess', 'checkers', 'backgammon', 'tictactoe'],
    required: true
  },
  creator: {
    userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    username: { type: String, required: true },
    isBot: { type: Boolean, default: false }
  },
  settings: {
    stake: { type: Number, required: true, min: 0 },
    isPrivate: { type: Boolean, default: false },
    password: String,
    maxSpectators: { type: Number, default: 10, min: 0, max: 100 },
    timeControl: {
      initialTime: Number,
      increment: Number
    },
    allowRevenge: { type: Boolean, default: true }
  },
  status: {
    type: String,
    enum: ['waiting', 'full', 'playing', 'finished'],
    default: 'waiting'
  },
  currentGame: { type: Schema.Types.ObjectId, ref: 'Game' },
  participants: [{
    userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    username: { type: String, required: true },
    isBot: { type: Boolean, default: false },
    joinedAt: { type: Date, default: Date.now },
    ready: { type: Boolean, default: false }
  }],
  spectators: [{
    userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    username: { type: String, required: true },
    joinedAt: { type: Date, default: Date.now }
  }],
  gameHistory: [{
    gameId: { type: Schema.Types.ObjectId, ref: 'Game', required: true },
    winner: {
      type: String,
      enum: ['player1', 'player2', 'draw']
    },
    finishedAt: { type: Date, required: true }
  }]
}, {
  timestamps: true
});

// Indexes
roomSchema.index({ id: 1 }, { unique: true });
roomSchema.index({ gameType: 1, status: 1 });
roomSchema.index({ 'creator.userId': 1 });
roomSchema.index({ status: 1, createdAt: -1 });
roomSchema.index({ 'settings.isPrivate': 1, status: 1 });

export const Room = model<IRoom>('Room', roomSchema);
```

### 6. Bots Collection

```typescript
interface IBot extends Document {
  username: string;
  avatar?: string;
  difficulty: 'easy' | 'medium' | 'hard' | 'expert';
  gameTypes: Array<'chess' | 'checkers' | 'backgammon' | 'tictactoe'>;
  personality: {
    playStyle: 'aggressive' | 'defensive' | 'balanced' | 'unpredictable';
    moveSpeed: {
      min: number; // milliseconds
      max: number; // milliseconds
    };
    errorRate: number; //

### 6. Bots Collection

```typescript
interface IBot extends Document {
  username: string;
  avatar?: string;
  difficulty: 'easy' | 'medium' | 'hard' | 'expert';
  gameTypes: Array<'chess' | 'checkers' | 'backgammon' | 'tictactoe'>;
  personality: {
    playStyle: 'aggressive' | 'defensive' | 'balanced' | 'unpredictable';
    moveSpeed: {
      min: number; // milliseconds
      max: number; // milliseconds
    };
    errorRate: number; // 0-1, probability of making suboptimal moves
    thinkingTime: {
      min: number; // seconds
      max: number; // seconds
    };
  };
  statistics: {
    gamesPlayed: number;
    gamesWon: number;
    gamesLost: number;
    gamesDrawn: number;
    winRate: number;
    averageGameDuration: number;
  };
  settings: {
    isActive: boolean;
    maxConcurrentGames: number;
    preferredStakeRange: {
      min: number;
      max: number;
    };
  };
  aiConfig: {
    engine: string; // AI engine identifier
    depth: number; // Search depth for decision making
    evaluationWeights: Record<string, number>;
    openingBook?: boolean;
    endgameTablebase?: boolean;
  };
  createdAt: Date;
  updatedAt: Date;
}

const botSchema = new Schema<IBot>({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    minlength: 3,
    maxlength: 20
  },
  avatar: String,
  difficulty: {
    type: String,
    enum: ['easy', 'medium', 'hard', 'expert'],
    required: true
  },
  gameTypes: [{
    type: String,
    enum: ['chess', 'checkers', 'backgammon', 'tictactoe']
  }],
  personality: {
    playStyle: {
      type: String,
      enum: ['aggressive', 'defensive', 'balanced', 'unpredictable'],
      default: 'balanced'
    },
    moveSpeed: {
      min: { type: Number, default: 1000 },
      max: { type: Number, default: 5000 }
    },
    errorRate: { type: Number, min: 0, max: 1, default: 0.1 },
    thinkingTime: {
      min: { type: Number, default: 2 },
      max: { type: Number, default: 10 }
    }
  },
  statistics: {
    gamesPlayed: { type: Number, default: 0 },
    gamesWon: { type: Number, default: 0 },
    gamesLost: { type: Number, default: 0 },
    gamesDrawn: { type: Number, default: 0 },
    winRate: { type: Number, default: 0 },
    averageGameDuration: { type: Number, default: 0 }
  },
  settings: {
    isActive: { type: Boolean, default: true },
    maxConcurrentGames: { type: Number, default: 5 },
    preferredStakeRange: {
      min: { type: Number, default: 0 },
      max: { type: Number, default: 1000 }
    }
  },
  aiConfig: {
    engine: { type: String, required: true },
    depth: { type: Number, default: 3 },
    evaluationWeights: { type: Schema.Types.Mixed, default: {} },
    openingBook: { type: Boolean, default: false },
    endgameTablebase: { type: Boolean, default: false }
  }
}, {
  timestamps: true
});

// Indexes
botSchema.index({ username: 1 }, { unique: true });
botSchema.index({ difficulty: 1, 'settings.isActive': 1 });
botSchema.index({ gameTypes: 1, 'settings.isActive': 1 });

export const Bot = model<IBot>('Bot', botSchema);
```

### 7. Notifications Collection

```typescript
interface INotification extends Document {
  userId: Schema.Types.ObjectId;
  type: 'game_invite' | 'game_result' | 'tournament_start' | 'tournament_result' | 'kyc_update' | 'balance_update' | 'system_message' | 'promotion';
  title: string;
  message: string;
  data?: {
    gameId?: string;
    tournamentId?: string;
    transactionId?: string;
    url?: string;
    actionRequired?: boolean;
  };
  priority: 'low' | 'medium' | 'high' | 'urgent';
  isRead: boolean;
  readAt?: Date;
  expiresAt?: Date;
  createdAt: Date;
}

const notificationSchema = new Schema<INotification>({
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  type: {
    type: String,
    enum: ['game_invite', 'game_result', 'tournament_start', 'tournament_result', 'kyc_update', 'balance_update', 'system_message', 'promotion'],
    required: true
  },
  title: {
    type: String,
    required: true,
    maxlength: 100
  },
  message: {
    type: String,
    required: true,
    maxlength: 500
  },
  data: {
    gameId: String,
    tournamentId: String,
    transactionId: String,
    url: String,
    actionRequired: Boolean
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  isRead: {
    type: Boolean,
    default: false
  },
  readAt: Date,
  expiresAt: Date
}, {
  timestamps: true
});

// Indexes
notificationSchema.index({ userId: 1, isRead: 1, createdAt: -1 });
notificationSchema.index({ type: 1, createdAt: -1 });
notificationSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

export const Notification = model<INotification>('Notification', notificationSchema);
```

### 8. Platform Analytics Collection

```typescript
interface IPlatformAnalytics extends Document {
  date: Date;
  metrics: {
    users: {
      total: number;
      active: number;
      new: number;
      verified: number;
    };
    games: {
      total: number;
      byType: {
        chess: number;
        checkers: number;
        backgammon: number;
        tictactoe: number;
      };
      completed: number;
      abandoned: number;
      averageDuration: number;
    };
    tournaments: {
      total: number;
      active: number;
      completed: number;
      participants: number;
    };
    financial: {
      totalRevenue: number;
      commission: number;
      deposits: number;
      withdrawals: number;
      gameStakes: number;
      tournamentFees: number;
    };
    bots: {
      gamesPlayed: number;
      winRate: number;
      activeCount: number;
    };
  };
  hourlyData?: Array<{
    hour: number;
    activeUsers: number;
    gamesStarted: number;
    revenue: number;
  }>;
  createdAt: Date;
}

const platformAnalyticsSchema = new Schema<IPlatformAnalytics>({
  date: {
    type: Date,
    required: true,
    unique: true
  },
  metrics: {
    users: {
      total: { type: Number, default: 0 },
      active: { type: Number, default: 0 },
      new: { type: Number, default: 0 },
      verified: { type: Number, default: 0 }
    },
    games: {
      total: { type: Number, default: 0 },
      byType: {
        chess: { type: Number, default: 0 },
        checkers: { type: Number, default: 0 },
        backgammon: { type: Number, default: 0 },
        tictactoe: { type: Number, default: 0 }
      },
      completed: { type: Number, default: 0 },
      abandoned: { type: Number, default: 0 },
      averageDuration: { type: Number, default: 0 }
    },
    tournaments: {
      total: { type: Number, default: 0 },
      active: { type: Number, default: 0 },
      completed: { type: Number, default: 0 },
      participants: { type: Number, default: 0 }
    },
    financial: {
      totalRevenue: { type: Number, default: 0 },
      commission: { type: Number, default: 0 },
      deposits: { type: Number, default: 0 },
      withdrawals: { type: Number, default: 0 },
      gameStakes: { type: Number, default: 0 },
      tournamentFees: { type: Number, default: 0 }
    },
    bots: {
      gamesPlayed: { type: Number, default: 0 },
      winRate: { type: Number, default: 0 },
      activeCount: { type: Number, default: 0 }
    }
  },
  hourlyData: [{
    hour: { type: Number, min: 0, max: 23 },
    activeUsers: { type: Number, default: 0 },
    gamesStarted: { type: Number, default: 0 },
    revenue: { type: Number, default: 0 }
  }]
}, {
  timestamps: true
});

// Indexes
platformAnalyticsSchema.index({ date: -1 });

export const PlatformAnalytics = model<IPlatformAnalytics>('PlatformAnalytics', platformAnalyticsSchema);
```

## Database Optimization Strategy

### Indexing Strategy

1. **Primary Indexes**: All collections have appropriate primary indexes on frequently queried fields
2. **Compound Indexes**: Multi-field indexes for complex queries
3. **Text Indexes**: For search functionality on usernames and tournament names
4. **TTL Indexes**: Automatic cleanup of expired notifications and temporary data

### Performance Considerations

1. **Connection Pooling**: Configure MongoDB connection pool for optimal performance
2. **Read Preferences**: Use read preferences for analytics queries
3. **Aggregation Pipeline**: Optimize complex queries using aggregation framework
4. **Sharding Strategy**: Plan for horizontal scaling with appropriate shard keys

### Data Integrity

1. **Validation**: Comprehensive schema validation at database level
2. **Referential Integrity**: Proper use of ObjectId references
3. **Transactions**: Use MongoDB transactions for critical operations
4. **Backup Strategy**: Regular automated backups with point-in-time recovery

### Security

1. **Authentication**: MongoDB authentication enabled
2. **Authorization**: Role-based access control
3. **Encryption**: Encryption at rest and in transit
4. **Audit Logging**: Comprehensive audit trail for sensitive operations

This database schema provides a solid foundation for the gaming platform with proper relationships, indexing, and scalability considerations.