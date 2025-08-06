# Testing and Quality Assurance Strategy

## Overview

This document outlines a comprehensive testing and quality assurance strategy for the gaming platform, covering all three applications (Backend API, Client Frontend, CRM Frontend) and ensuring high-quality, reliable, and maintainable code across the entire system.

## 1. Testing Philosophy and Principles

### Core Testing Principles
```typescript
interface TestingPrinciples {
  pyramid_approach: {
    unit_tests: '70% - Fast, isolated, comprehensive coverage';
    integration_tests: '20% - Component interaction validation';
    e2e_tests: '10% - Critical user journey validation';
  };
  
  quality_gates: {
    code_coverage: 'Minimum 80% overall, 90% for critical paths';
    performance: 'All tests must complete within defined SLA';
    reliability: 'Tests must be deterministic and stable';
    maintainability: 'Tests should be easy to understand and modify';
  };
  
  shift_left: 'Testing integrated early in development cycle';
  continuous_testing: 'Automated testing in CI/CD pipeline';
  risk_based: 'Focus testing efforts on high-risk areas';
}
```

### Testing Strategy by Application Layer
```typescript
interface LayeredTestingStrategy {
  backend_api: {
    unit_tests: 'Service logic, utilities, game engines';
    integration_tests: 'Database operations, external APIs';
    contract_tests: 'API endpoint contracts';
    performance_tests: 'Load, stress, and scalability';
    security_tests: 'Authentication, authorization, input validation';
  };
  
  client_frontend: {
    unit_tests: 'Components, hooks, utilities';
    integration_tests: 'Component interactions, state management';
    visual_tests: 'UI consistency and regression';
    accessibility_tests: 'WCAG compliance';
    performance_tests: 'Bundle size, rendering performance';
  };
  
  crm_frontend: {
    unit_tests: 'Admin components, data processing';
    integration_tests: 'Dashboard functionality, reporting';
    visual_tests: 'Admin interface consistency';
    usability_tests: 'Admin workflow efficiency';
  };
  
  system_level: {
    e2e_tests: 'Complete user journeys';
    api_tests: 'Cross-service communication';
    performance_tests: 'System-wide performance';
    security_tests: 'End-to-end security validation';
  };
}
```

## 2. Backend API Testing Strategy

### Unit Testing Framework
```typescript
// Jest Configuration for Backend
interface BackendJestConfig {
  preset: 'ts-jest';
  testEnvironment: 'node';
  roots: ['<rootDir>/src', '<rootDir>/tests'];
  testMatch: ['**/__tests__/**/*.test.ts', '**/?(*.)+(spec|test).ts'];
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts',
    '!src/config/**',
  ];
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  };
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'];
}

// Example Service Unit Test
describe('GameService', () => {
  let gameService: GameService;
  let mockGameRepository: jest.Mocked<GameRepository>;
  let mockSocketService: jest.Mocked<SocketService>;
  
  beforeEach(() => {
    mockGameRepository = createMockGameRepository();
    mockSocketService = createMockSocketService();
    gameService = new GameService(mockGameRepository, mockSocketService);
  });
  
  describe('createGame', () => {
    it('should create a new game with valid parameters', async () => {
      // Arrange
      const gameData: CreateGameRequest = {
        gameType: 'chess',
        timeControl: { type: 'blitz', minutes: 5 },
        isRated: true,
      };
      const expectedGame: Game = {
        id: 'game-123',
        ...gameData,
        status: 'waiting',
        createdAt: new Date(),
      };
      
      mockGameRepository.create.mockResolvedValue(expectedGame);
      
      // Act
      const result = await gameService.createGame('user-123', gameData);
      
      // Assert
      expect(result).toEqual(expectedGame);
      expect(mockGameRepository.create).toHaveBeenCalledWith({
        ...gameData,
        creatorId: 'user-123',
        status: 'waiting',
      });
      expect(mockSocketService.broadcastToLobby).toHaveBeenCalledWith(
        'gameCreated',
        expectedGame
      );
    });
    
    it('should throw error for invalid game type', async () => {
      // Arrange
      const invalidGameData = {
        gameType: 'invalid' as any,
        timeControl: { type: 'blitz', minutes: 5 },
      };
      
      // Act & Assert
      await expect(
        gameService.createGame('user-123', invalidGameData)
      ).rejects.toThrow('Invalid game type');
    });
  });
  
  describe('makeMove', () => {
    it('should process valid move and update game state', async () => {
      // Arrange
      const gameId = 'game-123';
      const playerId = 'user-123';
      const move: ChessMove = {
        from: { row: 1, col: 4 },
        to: { row: 3, col: 4 },
        piece: 'pawn',
      };
      
      const currentGame: Game = createMockChessGame({
        id: gameId,
        currentPlayer: playerId,
        status: 'active',
      });
      
      mockGameRepository.findById.mockResolvedValue(currentGame);
      mockGameRepository.update.mockResolvedValue({
        ...currentGame,
        moves: [...currentGame.moves, move],
      });
      
      // Act
      const result = await gameService.makeMove(gameId, playerId, move);
      
      // Assert
      expect(result.success).toBe(true);
      expect(mockGameRepository.update).toHaveBeenCalled();
      expect(mockSocketService.emitToGame).toHaveBeenCalledWith(
        gameId,
        'moveMade',
        expect.objectContaining({ move })
      );
    });
  });
});
```

### Integration Testing
```typescript
// Database Integration Tests
describe('GameRepository Integration', () => {
  let repository: GameRepository;
  let testDb: TestDatabase;
  
  beforeAll(async () => {
    testDb = await createTestDatabase();
    repository = new GameRepository(testDb.connection);
  });
  
  afterAll(async () => {
    await testDb.cleanup();
  });
  
  beforeEach(async () => {
    await testDb.clearCollections(['games', 'users']);
  });
  
  describe('create', () => {
    it('should persist game to database', async () => {
      // Arrange
      const gameData = {
        gameType: 'chess',
        creatorId: 'user-123',
        timeControl: { type: 'blitz', minutes: 5 },
        status: 'waiting',
      };
      
      // Act
      const createdGame = await repository.create(gameData);
      
      // Assert
      expect(createdGame.id).toBeDefined();
      expect(createdGame.createdAt).toBeInstanceOf(Date);
      
      const foundGame = await repository.findById(createdGame.id);
      expect(foundGame).toEqual(createdGame);
    });
  });
  
  describe('findActiveGamesByUser', () => {
    it('should return only active games for user', async () => {
      // Arrange
      const userId = 'user-123';
      await repository.create({
        gameType: 'chess',
        creatorId: userId,
        status: 'active',
      });
      await repository.create({
        gameType: 'chess',
        creatorId: userId,
        status: 'completed',
      });
      await repository.create({
        gameType: 'chess',
        creatorId: 'other-user',
        status: 'active',
      });
      
      // Act
      const activeGames = await repository.findActiveGamesByUser(userId);
      
      // Assert
      expect(activeGames).toHaveLength(1);
      expect(activeGames[0].status).toBe('active');
      expect(activeGames[0].creatorId).toBe(userId);
    });
  });
});

// API Integration Tests
describe('Game API Integration', () => {
  let app: Express;
  let testDb: TestDatabase;
  let authToken: string;
  
  beforeAll(async () => {
    testDb = await createTestDatabase();
    app = createTestApp(testDb.connection);
    authToken = await createTestUser(app);
  });
  
  afterAll(async () => {
    await testDb.cleanup();
  });
  
  describe('POST /api/v1/games', () => {
    it('should create new game', async () => {
      // Arrange
      const gameData = {
        gameType: 'chess',
        timeControl: { type: 'blitz', minutes: 5 },
        isRated: true,
      };
      
      // Act
      const response = await request(app)
        .post('/api/v1/games')
        .set('Authorization', `Bearer ${authToken}`)
        .send(gameData)
        .expect(201);
      
      // Assert
      expect(response.body.success).toBe(true);
      expect(response.body.data).toMatchObject({
        gameType: 'chess',
        status: 'waiting',
        timeControl: gameData.timeControl,
      });
      expect(response.body.data.id).toBeDefined();
    });
    
    it('should return 401 for unauthenticated request', async () => {
      // Act & Assert
      await request(app)
        .post('/api/v1/games')
        .send({ gameType: 'chess' })
        .expect(401);
    });
  });
});
```

### Performance Testing
```typescript
// Load Testing with Artillery
interface LoadTestConfig {
  config: {
    target: 'http://localhost:3000';
    phases: [
      { duration: 60, arrivalRate: 10 }, // Warm up
      { duration: 300, arrivalRate: 50 }, // Sustained load
      { duration: 120, arrivalRate: 100 }, // Peak load
    ];
  };
  scenarios: [
    {
      name: 'Game Creation and Moves';
      weight: 70;
      flow: [
        { post: { url: '/api/v1/auth/login', json: { username: 'testuser', password: 'password' } } },
        { post: { url: '/api/v1/games', json: { gameType: 'chess', timeControl: { type: 'blitz', minutes: 5 } } } },
        { post: { url: '/api/v1/games/{{ gameId }}/moves', json: { move: { from: 'e2', to: 'e4' } } } },
      ];
    },
    {
      name: 'Tournament Participation';
      weight: 30;
      flow: [
        { get: { url: '/api/v1/tournaments' } },
        { post: { url: '/api/v1/tournaments/{{ tournamentId }}/join' } },
      ];
    }
  ];
}

// Performance Test Implementation
describe('Performance Tests', () => {
  describe('Game Engine Performance', () => {
    it('should process chess moves within 50ms', async () => {
      // Arrange
      const chessEngine = new ChessEngine();
      const gameState = createStandardChessPosition();
      const moves = generateRandomMoves(1000);
      
      // Act
      const startTime = performance.now();
      for (const move of moves) {
        chessEngine.isValidMove(gameState, move);
      }
      const endTime = performance.now();
      
      // Assert
      const averageTime = (endTime - startTime) / moves.length;
      expect(averageTime).toBeLessThan(50); // 50ms per move
    });
  });
  
  describe('Database Performance', () => {
    it('should handle concurrent game creation', async () => {
      // Arrange
      const concurrentRequests = 100;
      const gameData = {
        gameType: 'chess',
        timeControl: { type: 'blitz', minutes: 5 },
      };
      
      // Act
      const startTime = performance.now();
      const promises = Array(concurrentRequests).fill(null).map(() =>
        gameService.createGame('user-123', gameData)
      );
      
      const results = await Promise.all(promises);
      const endTime = performance.now();
      
      // Assert
      expect(results).toHaveLength(concurrentRequests);
      expect(endTime - startTime).toBeLessThan(5000); // 5 seconds for 100 games
      results.forEach(game => {
        expect(game.id).toBeDefined();
      });
    });
  });
});
```

## 3. Frontend Testing Strategy

### React Component Testing
```typescript
// Jest + React Testing Library Configuration
interface FrontendJestConfig {
  testEnvironment: 'jsdom';
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.ts'];
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1';
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy';
  };
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.tsx',
    '!src/reportWebVitals.ts',
  ];
  coverageThreshold: {
    global: {
      branches: 75,
      functions: 75,
      lines: 75,
      statements: 75,
    },
  };
}

// Component Unit Tests
describe('GameBoard Component', () => {
  const defaultProps: GameBoardProps = {
    gameType: 'chess',
    gameState: createMockChessGame(),
    isPlayerTurn: true,
    onMove: jest.fn(),
    onResign: jest.fn(),
    onOfferDraw: jest.fn(),
  };
  
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  it('should render chess board with pieces', () => {
    // Act
    render(<GameBoard {...defaultProps} />);
    
    // Assert
    expect(screen.getByTestId('chess-board')).toBeInTheDocument();
    expect(screen.getAllByTestId(/chess-piece/)).toHaveLength(32);
  });
  
  it('should highlight valid moves when piece is selected', async () => {
    // Arrange
    const user = userEvent.setup();
    render(<GameBoard {...defaultProps} />);
    
    // Act
    const pawn = screen.getByTestId('chess-piece-e2');
    await user.click(pawn);
    
    // Assert
    expect(screen.getByTestId('square-e3')).toHaveClass('valid-move');
    expect(screen.getByTestId('square-e4')).toHaveClass('valid-move');
  });
  
  it('should call onMove when valid move is made', async () => {
    // Arrange
    const user = userEvent.setup();
    const mockOnMove = jest.fn();
    render(<GameBoard {...defaultProps} onMove={mockOnMove} />);
    
    // Act
    await user.click(screen.getByTestId('chess-piece-e2'));
    await user.click(screen.getByTestId('square-e4'));
    
    // Assert
    expect(mockOnMove).toHaveBeenCalledWith({
      from: { row: 1, col: 4 },
      to: { row: 3, col: 4 },
      piece: 'pawn',
    });
  });
  
  it('should disable interactions when not player turn', () => {
    // Arrange
    render(<GameBoard {...defaultProps} isPlayerTurn={false} />);
    
    // Act & Assert
    const pieces = screen.getAllByTestId(/chess-piece/);
    pieces.forEach(piece => {
      expect(piece).toHaveClass('disabled');
    });
  });
  
  it('should show resign and draw offer buttons', () => {
    // Act
    render(<GameBoard {...defaultProps} />);
    
    // Assert
    expect(screen.getByRole('button', { name: /resign/i })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /offer draw/i })).toBeInTheDocument();
  });
});

// Custom Hook Testing
describe('useGameLogic Hook', () => {
  const mockGame: Game = createMockChessGame();
  
  beforeEach(() => {
    // Mock RTK Query hooks
    (gameApi.useGetGameQuery as jest.Mock).mockReturnValue({
      data: mockGame,
      isLoading: false,
      error: null,
    });
    
    (gameApi.useMakeMoveQuery as jest.Mock).mockReturnValue([
      jest.fn().mockResolvedValue({ unwrap: () => Promise.resolve() }),
    ]);
  });
  
  it('should return game data and loading state', () => {
    // Act
    const { result } = renderHook(() => useGameLogic('game-123'));
    
    // Assert
    expect(result.current.game).toEqual(mockGame);
    expect(result.current.isLoading).toBe(false);
  });
  
  it('should determine if it is player turn', () => {
    // Arrange
    const mockUser = { id: 'user-123' };
    (useSelector as jest.Mock).mockReturnValue(mockUser);
    
    // Act
    const { result } = renderHook(() => useGameLogic('game-123'));
    
    // Assert
    expect(result.current.isPlayerTurn).toBe(true);
  });
  
  it('should handle move submission', async () => {
    // Arrange
    const mockMakeMove = jest.fn().mockResolvedValue({ unwrap: () => Promise.resolve() });
    (gameApi.useMakeMoveQuery as jest.Mock).mockReturnValue([mockMakeMove]);
    
    const { result } = renderHook(() => useGameLogic('game-123'));
    
    // Act
    const move = { from: { row: 1, col: 4 }, to: { row: 3, col: 4 } };
    await act(async () => {
      await result.current.handleMove(move);
    });
    
    // Assert
    expect(mockMakeMove).toHaveBeenCalledWith({
      gameId: 'game-123',
      move,
    });
  });
});
```

### Integration Testing for Frontend
```typescript
// Redux Store Integration Tests
describe('Game State Management', () => {
  let store: ReturnType<typeof setupStore>;
  
  beforeEach(() => {
    store = setupStore();
  });
  
  it('should handle game creation flow', async () => {
    // Arrange
    const gameData = {
      gameType: 'chess',
      timeControl: { type: 'blitz', minutes: 5 },
    };
    
    // Mock API response
    const mockGame = { id: 'game-123', ...gameData, status: 'waiting' };
    server.use(
      rest.post('/api/v1/games', (req, res, ctx) => {
        return res(ctx.json({ success: true, data: mockGame }));
      })
    );
    
    // Act
    const result = await store.dispatch(
      gameApi.endpoints.createGame.initiate(gameData)
    );
    
    // Assert
    expect(result.data).toEqual(mockGame);
    
    // Verify state update
    const state = store.getState();
    expect(gameApi.endpoints.createGame.select(gameData)(state)).toMatchObject({
      data: mockGame,
    });
  });
  
  it('should handle real-time game updates', () => {
    // Arrange
    const initialGame = createMockChessGame();
    store.dispatch(
      gameApi.util.upsertQueryData('getGame', 'game-123', initialGame)
    );
    
    // Act - Simulate socket update
    const updatedGame = {
      ...initialGame,
      moves: [...initialGame.moves, { from: 'e2', to: 'e4' }],
    };
    
    store.dispatch(
      gameApi.util.updateQueryData('getGame', 'game-123', (draft) => {
        Object.assign(draft, updatedGame);
      })
    );
    
    // Assert
    const state = store.getState();
    const gameData = gameApi.endpoints.getGame.select('game-123')(state);
    expect(gameData.data?.moves).toHaveLength(initialGame.moves.length + 1);
  });
});

// Component Integration Tests
describe('Game Page Integration', () => {
  it('should render complete game interface', async () => {
    // Arrange
    const mockGame = createMockChessGame();
    server.use(
      rest.get('/api/v1/games/game-123', (req, res, ctx) => {
        return res(ctx.json({ success: true, data: mockGame }));
      })
    );
    
    // Act
    render(
      <Provider store={store}>
        <MemoryRouter initialEntries={['/games/game-123']}>
          <GamePage />
        </MemoryRouter>
      </Provider>
    );
    
    // Assert
    await waitFor(() => {
      expect(screen.getByTestId('game-board')).toBeInTheDocument();
      expect(screen.getByTestId('game-info')).toBeInTheDocument();
      expect(screen.getByTestId('move-history')).toBeInTheDocument();
    });
  });
  
  it('should handle game moves end-to-end', async () => {
    // Arrange
    const user = userEvent.setup();
    const mockGame = createMockChessGame();
    
    server.use(
      rest.get('/api/v1/games/game-123', (req, res, ctx) => {
        return res(ctx.json({ success: true, data: mockGame }));
      }),
      rest.post('/api/v1/games/game-123/moves', (req, res, ctx) => {
        return res(ctx.json({ success: true, data: { valid: true } }));
      })
    );
    
    render(
      <Provider store={store}>
        <MemoryRouter initialEntries={['/games/game-123']}>
          <GamePage />
        </MemoryRouter>
      </Provider>
    );
    
    await waitFor(() => {
      expect(screen.getByTestId('game-board')).toBeInTheDocument();
    });
    
    // Act
    await user.click(screen.getByTestId('chess-piece-e2'));
    await user.click(screen.getByTestId('square-e4'));
    
    // Assert
    await waitFor(() => {
      expect(screen.getByText(/move made/i)).toBeInTheDocument();
    });
  });
});
```

### Visual Regression Testing
```typescript
// Chromatic Configuration
interface ChromaticConfig {
  projectToken: string;
  buildScriptName: 'build-storybook';
  exitZeroOnChanges: true;
  exitOnceUploaded: true;
  skip: 'dependabot/**';
}

// Storybook Stories for Visual Testing
export default {
  title: 'Game/GameBoard',
  component: GameBoard,
  parameters: {
    layout: 'fullscreen',
    chromatic: {
      viewports: [320, 768, 1200],
      delay: 1000, // Wait for animations
    },
  },
} as ComponentMeta<typeof GameBoard>;

export const ChessInitialPosition: ComponentStory<typeof GameBoard> = () => (
  <GameBoard
    gameType="chess"
    gameState={createInitialChessPosition()}
    isPlayerTurn={true}
    onMove={() => {}}
    onResign={() => {}}
    onOfferDraw={() => {}}
  />
);

export const ChessMidGame: ComponentStory<typeof GameBoard> = () => (
  <GameBoard
    gameType="chess"
    gameState={createMidGameChessPosition()}
    isPlayerTurn={false}
    onMove={() => {}}
    onResign={() => {}}
    onOfferDraw={() => {}}
  />
);

export const CheckersGame: ComponentStory<typeof GameBoard> = () => (
  <GameBoard
    gameType="checkers"
    gameState={createCheckersPosition()}
    isPlayerTurn={true}
    onMove={() => {}}
    onResign={() => {}}
    onOfferDraw={() => {}}
  />
);

// Visual Test Automation
describe('Visual Regression Tests', () => {
  it('should match game board snapshots', async () => {
    // Arrange
    const gameStates = [
      createInitialChessPosition(),
      createMidGameChessPosition(),
      createEndGameChessPosition(),
    ];
    
    // Act & Assert
    for (const gameState of gameStates) {
      render(
        <GameBoard
          gameType="chess"
          gameState={gameState}
          isPlayerTurn={true}
          onMove={() => {}}
          onResign={() => {}}
          onOfferDraw={() => {}}
        />
      );
      
      await waitFor(() => {
        expect(screen.getByTestId('game-board')).toBeInTheDocument();
      });
      
      expect(screen.getByTestId('game-board')).toMatchSnapshot();
    }
  });
});
```

## 4. End-to-End Testing Strategy

### Playwright E2E Tests
```typescript
// Playwright Configuration
interface PlaywrightConfig {
  testDir: './e2e';
  timeout: 30000;
  expect: { timeout: 5000 };
  fullyParallel: true;
  forbidOnly: true;
  retries: 2;
  workers: 4;
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }],
  ];
  use: {
    baseURL: 'http://localhost:3000';
    trace: 'on-first-retry';
    screenshot: 'only-on-failure';
    video: 'retain-on-failure';
  };
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile', use: { ...devices['iPhone 12'] } },
  ];
}

// Critical User Journey Tests
describe('Game Playing Journey', () => {
  test('complete chess game flow', async ({ page }) => {
    // Arrange
    await page.goto('/');
    
    // Login
    await page.click('[data-testid="login-button"]');
    await page.fill('[data-testid="username-input"]', 'testuser1');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="submit-login"]');
    
    // Wait for dashboard
    await expect(page.locator('[data-testid="user-dashboard"]')).toBeVisible();
    
    // Create new game
    await page.click('[data-testid="create-game-button"]');
    await page.selectOption('[data-testid="game-type-select"]', 'chess');
    await page.selectOption('[data-testid="time-control-select"]', 'blitz');
    await page.click('[data-testid="create-game-submit"]');
    
    // Wait for game lobby
    await expect(page.locator('[data-testid="game-lobby"]')).toBeVisible();
    
    // Simulate second player joining (using API)
    const gameId = await page.getAttribute('[data-testid="game-id"]', 'data-game-id');
    await joinGameAsSecondPlayer(gameId);
    
    // Wait for game to start
    await expect(page.locator('[data-testid="game-board"]')).toBeVisible();
    
    // Make first move
    await page.click('[data-testid="chess-piece-e2"]');
    await page.click('[data-testid="square-e4"]');
    
    // Verify move was made
    await expect(page.locator('[data-testid="move-history"]')).toContainText('1. e4');
    
    // Wait for opponent move (simulated)
    await simulateOpponentMove(gameId, { from: 'e7', to: 'e5' });
    
    // Verify opponent move appears
    await expect(page.locator('[data-testid="move-history"]')).toContainText('1... e5');
    
    // Continue game for several moves
    const moves = [
      { from: 'g1', to: 'f3' }, // Nf3
      { from: 'b8', to: 'c6' }, // Nc6
      { from: 'f1', to: 'c4' }, // Bc4
    ];
    
    for (const move of moves) {
      if (move.from.includes('1') || move.from.includes('2')) {
        // Player move
        await page.click(`[data-testid="chess-piece-${move.from}"]`);
        await page.click(`[data-testid="square-${move.to}"]`);
      } else {
        // Opponent move (simulated)
        await simulateOpponentMove(gameId, move);
      }
    }
    
    // Test game controls
    await page.click('[data-testid="offer-draw-button"]');
    await expect(page.locator('[data-testid="draw-offer-sent"]')).toBeVisible();
    
    // Simulate draw decline
    await simulateDrawResponse(gameId, false);
    await expect(page.locator('[data-testid="draw-offer-declined"]')).toBeVisible();
    
    // Test resignation
    await page.click('[data-testid="resign-button"]');
    await page.click('[data-testid="confirm-resign"]');
    
    // Verify game ended
    await expect(page.locator('[data-testid="game-result"]')).toContainText('You resigned');
    await expect(page.locator('[data-testid="return-to-lobby"]')).toBeVisible();
  });
  
  test('tournament participation flow', async ({ page }) => {
    // Login
    await loginUser(page, 'testuser1', 'password123');
    
    // Navigate to tournaments
    await page.click('[data-testid="tournaments-nav"]');
    await expect(page.locator('[data-testid="tournaments-page"]')).toBeVisible();
    
    // Join tournament
    await page.click('[data-testid="tournament-card"]:first-child [data-testid="join-tournament"]');
    await expect(page.locator('[data-testid="tournament-joined"]')).toBeVisible();
    
    // Wait for tournament to start
    await simulateTournamentStart(tournamentId);
    
    // Verify tournament bracket
    await expect(page.locator('[data-testid="tournament-bracket"]')).toBeVisible();
    
    // Play tournament games
    await playTournamentRound(page, gameId);
    
    // Verify advancement or elimination
    await expect(page.locator('[data-testid="tournament-result"]')).toBeVisible();
  });
  
  test('payment and subscription flow', async ({ page }) => {
    // Login
    await loginUser(page, 'testuser1', 'password123');
    
    // Navigate to subscription
    await page.click('[data-testid="upgrade-account"]');
    await expect(page.locator('[data-testid="subscription-plans"]')).toBeVisible();
    
    // Select premium plan
    await page.click('[data-testid="premium-plan-select"]');
    await page.click('[data-testid="proceed-to-payment"]');
    
    // Fill payment form (test mode)
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="card-expiry"]', '12/25');
    await page.fill('[data-testid="card-cvc"]', '123');
    await page.click('[data-testid="submit-payment"]');
    
    // Verify successful upgrade
    await expect(page.locator('[data-testid="payment-success"]')).toBeVisible();
    await expect(page.locator('[data-testid="premium-badge"]')).toBeVisible();
  });
});

// Cross-browser Compatibility Tests
describe('Cross-browser Compatibility', () => {
  ['chromium', 'firefox', 'webkit'].forEach(browserName => {
    test(`game functionality works in ${browserName}`, async ({ page }) => {
      // Test core game functionality across browsers
      await loginUser(page, 'testuser1', 'password123');
      await createAndPlayQuickGame(page);
      await verifyGameFunctionality(page);
    });
  });
});

// Mobile Responsiveness Tests
describe('Mobile Responsiveness', () => {
  test('mobile game interface', async ({ page }) => {
    // Set mobile viewport
    await page.setViewportSize({ width: 375, height: 667 });
    
    await loginUser(page, 'testuser1', 'password123');
    
    // Verify mobile navigation
    await page.click('[data-testid="mobile-menu-toggle"]');
    await expect(page.locator('[data-testid="mobile-menu"]')).toBeVisible();
    
    // Test mobile game creation
    await page.click('[data-testid="mobile-create-game"]');
    await createMobileGame(page);
    
    // Verify mobile game board
    await expect(page.locator('[data-testid="mobile-game-board"]')).toBeVisible();
    await testMobileGameInteraction(page);
  });
});
```

## 5. Security Testing Strategy

### Authentication and Authorization Testing
```typescript
// Security Test Suite
describe('Security Tests', () => {
  describe('Authentication Security', () => {
    test('should prevent brute force attacks', async ({ page }) => {
      // Attempt multiple failed logins
      for (let i = 0; i < 6; i++) {
        await page.goto('/login');
        await page.fill('[data-testid="username-input"]', 'testuser');
        await page.fill('[data-testid="password-input"]', 'wrongpassword');
        await page.click('[data-testid="submit-login"]');
      }
      
      // Verify account lockout
      await expect(page.locator('[data-testid="account-locked"]')).toBeVisible();
    });
    
    test('should enforce strong password requirements', async ({ page }) => {
      await page.goto('/register');
      
      const weakPasswords = ['123', 'password', 'abc123'];
      
      for (const password of weakPasswords) {
        await page.fill('[data-testid="password-input"]', password);
        await page.blur('[data-testid="password-input"]');
        
        await expect(page.locator('[data-testid="password-error"]')).toBeVisible();
      }
    });
    
    test('should validate JWT token expiration', async ({ page, request }) => {
      // Login and get token
      await loginUser(page, 'testuser1', 'password123');
      
      // Extract token from localStorage
      const token = await page.evaluate(() => localStorage.getItem('authToken'));
      
      // Simulate token expiration by manipulating time
      await page.addInitScript(() => {
        const originalDate = Date;
        Date.now = () => originalDate.now() + 24 * 60 * 60 * 1000; // +24 hours
      });
      
      // Attempt API call with expired token
      const response = await request.get('/api/v1/user/profile', {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      expect(response.status()).toBe(401);
    });
  });
  
  describe('Input Validation Security', () => {
    test('should prevent XSS attacks', async ({ page }) => {
      await loginUser(page, 'testuser1', 'password123');
      
      // Attempt XSS in username field
      const xssPayload = '<script>alert("XSS")</script>';
      
      await page.goto('/profile/edit');
      await page.fill('[data-testid="display-name-input"]', xssPayload);
      await page.click('[data-testid="save-profile"]');
      
      // Verify XSS is prevented
      await page.goto('/profile');
      const displayName = await page.textContent('[data-testid="display-name"]');
      expect(displayName).not.toContain('<script>');
    });
    
    test('should prevent SQL injection', async ({ request }) => {
      const sqlInjectionPayloads = [
        "'; DROP TABLE users; --",
        "' OR '1'='1",
        "' UNION SELECT * FROM users --"
      ];
      
      for (const payload of sqlInjectionPayloads) {
        const response = await request.post('/api/v1/auth/login', {
          data: {
            username: payload,
            password: 'password'
          }
        });
        
        // Should return validation error, not SQL error
        expect(response.status()).toBe(400);
        const body = await response.json();
        expect(body.error.type).toBe('VALIDATION_ERROR');
      }
    });
  });
  
  describe('Authorization Security', () => {
    test('should prevent unauthorized game access', async ({ page, request }) => {
      // Create game as user1
      await loginUser(page, 'testuser1', 'password123');
      const gameId = await createPrivateGame(page);
      
      // Attempt access as user2
      await loginUser(page, 'testuser2', 'password123');
      await page.goto(`/games/${gameId}`);
      
      // Should be redirected or show access denied
      await expect(page.locator('[data-testid="access-denied"]')).toBeVisible();
    });
    
    test('should prevent admin endpoint access by regular users', async ({ request }) => {
      // Login as regular user
      const userToken = await getAuthToken('testuser1', 'password123');
      
      // Attempt admin endpoint access
      const response = await request.get('/api/v1/admin/users', {
        headers: { Authorization: `Bearer ${userToken}` }
      });
      
      expect(response.status()).toBe(403);
    });
  });
});

// Penetration Testing Automation
describe('Penetration Testing', () => {
  test('should resist common attack vectors', async ({ page, request }) => {
    const attackVectors = [
      // Directory traversal
      { path: '/api/v1/files/../../etc/passwd', expectedStatus: 400 },
      
      // Command injection
      { 
        path: '/api/v1/search', 
        data: { query: '; cat /etc/passwd' },
        expectedStatus: 400 
      },
      
      // LDAP injection
      { 
        path: '/api/v1/auth/login',
        data: { username: 'admin)(|(password=*))', password: 'any' },
        expectedStatus: 400
      }
    ];
    
    for (const vector of attackVectors) {
      const response = vector.data 
        ? await request.post(vector.path, { data: vector.data })
        : await request.get(vector.path);
      
      expect(response.status()).toBe(vector.expectedStatus);
    }
  });
});
```

## 6. Performance Testing Strategy

### Load Testing Implementation
```typescript
// K6 Load Testing Scripts
interface LoadTestScenarios {
  game_creation: {
    executor: 'ramping-vus';
    startVUs: 0;
    stages: [
      { duration: '2m', target: 100 },
      { duration: '5m', target: 100 },
      { duration: '2m', target: 200 },
      { duration: '5m', target: 200 },
      { duration: '2m', target: 0 }
    ];
  };
  
  concurrent_games: {
    executor: 'constant-vus';
    vus: 500;
    duration: '10m';
  };
  
  tournament_load: {
    executor: 'ramping-arrival-rate';
    startRate: 0;
    timeUnit: '1s';
    preAllocatedVUs: 50;
    maxVUs: 200;
    stages: [
      { duration: '5m', target: 20 },
      { duration: '10m', target: 20 },
      { duration: '5m', target: 0 }
    ];
  };
}

// Performance Test Implementation
export default function() {
  const baseUrl = 'http://localhost:3000';
  
  // Login
  const loginResponse = http.post(`${baseUrl}/api/v1/auth/login`, {
    username: 'testuser',
    password: 'password123'
  });
  
  check(loginResponse, {
    'login successful': (r) => r.status === 200,
    'login response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  const authToken = loginResponse.json('data.token');
  const headers = { Authorization: `Bearer ${authToken}` };
  
  // Create game
  const gameResponse = http.post(`${baseUrl}/api/v1/games`, {
    gameType: 'chess',
    timeControl: { type: 'blitz', minutes: 5 }
  }, { headers });
  
  check(gameResponse, {
    'game creation successful': (r) => r.status === 201,
    'game creation response time < 300ms': (r) => r.timings.duration < 300,
  });
  
  const gameId = gameResponse.json('data.id');
  
  // Make moves
  for (let i = 0; i < 10; i++) {
    const moveResponse = http.post(`${baseUrl}/api/v1/games/${gameId}/moves`, {
      move: generateRandomMove()
    }, { headers });
    
    check(moveResponse, {
      'move successful': (r) => r.status === 200,
      'move response time < 100ms': (r) => r.timings.duration < 100,
    });
    
    sleep(1);
  }
}

// Performance Thresholds
export const thresholds = {
  http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
  http_req_failed: ['rate<0.01'],   // Error rate under 1%
  checks: ['rate>0.95'],            // 95% of checks pass
};
```

### Frontend Performance Testing
```typescript
// Lighthouse Performance Testing
describe('Frontend Performance', () => {
  test('should meet Lighthouse performance criteria', async ({ page }) => {
    // Navigate to main pages
    const pages = ['/', '/games', '/tournaments', '/profile'];
    
    for (const pagePath of pages) {
      await page.goto(pagePath);
      
      // Run Lighthouse audit
      const lighthouse = await playAudit({
        page,
        thresholds: {
          performance: 90,
          accessibility: 95,
          'best-practices': 90,
          seo: 80,
        },
      });
      
      expect(lighthouse.lhr.categories.performance.score * 100).toBeGreaterThan(90);
    }
  });
  
  test('should have optimal bundle sizes', async () => {
    const bundleAnalysis = await analyzeBundles();
    
    expect(bundleAnalysis.mainBundle.size).toBeLessThan(500 * 1024); // 500KB
    expect(bundleAnalysis.vendorBundle.size).toBeLessThan(1024 * 1024); // 1MB
    expect(bundleAnalysis.totalSize).toBeLessThan(2 * 1024 * 1024); // 2MB
  });
  
  test('should render game board within performance budget', async ({ page }) => {
    await page.goto('/games/new');
    
    // Measure rendering performance
    const metrics = await page.evaluate(() => {
      return new Promise((resolve) => {
        const observer = new PerformanceObserver((list) => {
          const entries = list.getEntries();
          const paintEntries = entries.filter(entry => 
            entry.name === 'first-contentful-paint' || 
            entry.name === 'largest-contentful-paint'
          );
          
          if (paintEntries.length >= 2) {
            resolve({
              fcp: paintEntries.find(e => e.name === 'first-contentful-paint')?.startTime,
              lcp: paintEntries.find(e => e.name === 'largest-contentful-paint')?.startTime,
            });
            observer.disconnect();
          }
        });
        
        observer.observe({ entryTypes: ['paint', 'largest-contentful-paint'] });
      });
    });
    
    expect(metrics.fcp).toBeLessThan(1500); // FCP < 1.5s
    expect(metrics.lcp).toBeLessThan(2500); // LCP < 2.5s
  });
});
```

## 7. Accessibility Testing Strategy

### WCAG Compliance Testing
```typescript
// Accessibility Testing with axe-core
describe('Accessibility Tests', () => {
  test('should meet WCAG 2.1 AA standards', async ({ page }) => {
    const pages = [
      '/',
      '/login',
      '/register',
      '/games',
      '/tournaments',
      '/profile'
    ];
    
    for (const pagePath of pages) {
      await page.goto(pagePath);
      
      // Run axe accessibility audit
      const accessibilityScanResults = await new AxeBuilder({ page })
        .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
        .analyze();
      
      expect(accessibilityScanResults.violations).toEqual([]);
    }
  });
  
  test('should support keyboard navigation', async ({ page }) => {
    await page.goto('/games/new');
    
    // Test tab navigation
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toHaveAttribute('data-testid', 'game-type-select');
    
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toHaveAttribute('data-testid', 'time-control-select');
    
    // Test game board keyboard navigation
    await page.goto('/games/chess-demo');
    
    // Focus on game board
    await page.keyboard.press('Tab');
    await expect(page.locator(':focus')).toHaveAttribute('data-testid', 'game-board');
    
    // Test arrow key navigation
    await page.keyboard.press('ArrowRight');
    await page.keyboard.press('ArrowDown');
    await page.keyboard.press('Enter'); // Select piece
    
    await page.keyboard.press('ArrowRight');
    await page.keyboard.press('ArrowRight');
    await page.keyboard.press('Enter'); // Make move
    
    // Verify move was made
    await expect(page.locator('[data-testid="move-history"]')).toContainText('e4');
  });
  
  test('should provide proper ARIA labels', async ({ page }) => {
    await page.goto('/games/chess-demo');
    
    // Check game board ARIA labels
    const gameBoard = page.locator('[data-testid="game-board"]');
    await expect(gameBoard).toHaveAttribute('role', 'grid');
    await expect(gameBoard).toHaveAttribute('aria-label', 'Chess board');
    
    // Check piece ARIA labels
    const whitePawn = page.locator('[data-testid="chess-piece-e2"]');
    await expect(whitePawn).toHaveAttribute('role', 'gridcell');
    await expect(whitePawn).toHaveAttribute('aria-label', 'White pawn on e2');
    
    // Check move history ARIA labels
    const moveHistory = page.locator('[data-testid="move-history"]');
    await expect(moveHistory).toHaveAttribute('role', 'log');
    await expect(moveHistory).toHaveAttribute('aria-label', 'Game move history');
  });
  
  test('should support screen readers', async ({ page }) => {
    // Test with screen reader simulation
    await page.goto('/games/chess-demo');
    
    // Verify live regions for game updates
    const gameStatus = page.locator('[data-testid="game-status"]');
    await expect(gameStatus).toHaveAttribute('aria-live', 'polite');
    
    const moveAnnouncement = page.locator('[data-testid="move-announcement"]');
    await expect(moveAnnouncement).toHaveAttribute('aria-live', 'assertive');
    
    // Test move announcement
    await page.click('[data-testid="chess-piece-e2"]');
    await page.click('[data-testid="square-e4"]');
    
    await expect(moveAnnouncement).toContainText('White pawn moves from e2 to e4');
  });
});

// Color Contrast Testing
describe('Color Contrast Tests', () => {
  test('should meet contrast ratio requirements', async ({ page }) => {
    await page.goto('/');
    
    // Test color contrast for various elements
    const contrastTests = [
      { selector: '[data-testid="primary-button"]', minRatio: 4.5 },
      { selector: '[data-testid="secondary-button"]', minRatio: 3.0 },
      { selector: '[data-testid="body-text"]', minRatio: 4.5 },
      { selector: '[data-testid="heading-text"]', minRatio: 4.5 },
    ];
    
    for (const test of contrastTests) {
      const element = page.locator(test.selector);
      const contrastRatio = await getContrastRatio(element);
      expect(contrastRatio).toBeGreaterThan(test.minRatio);
    }
  });
});
```

## 8. API Testing Strategy

### Contract Testing with Pact
```typescript
// Consumer Contract Tests (Frontend)
describe('Game API Contract Tests', () => {
  const provider = new Pact({
    consumer: 'gaming-platform-frontend',
    provider: 'gaming-platform-backend',
    port: 1234,
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'INFO',
  });
  
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());
  
  test('should create game successfully', async () => {
    // Arrange
    const gameRequest = {
      gameType: 'chess',
      timeControl: { type: 'blitz', minutes: 5 },
      isRated: true,
    };
    
    const expectedResponse = {
      success: true,
      data: {
        id: like('game-123'),
        gameType: 'chess',
        status: 'waiting',
        timeControl: gameRequest.timeControl,
        isRated: true,
        createdAt: iso8601DateTime(),
      },
    };
    
    await provider.addInteraction({
      state: 'user is authenticated',
      uponReceiving: 'a request to create a chess game',
      withRequest: {
        method: 'POST',
        path: '/api/v1/games',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': like('Bearer token'),
        },
        body: gameRequest,
      },
      willRespondWith: {
        status: 201,
        headers: { 'Content-Type': 'application/json' },
        body: expectedResponse,
      },
    });
    
    // Act
    const response = await gameApi.createGame(gameRequest);
    
    // Assert
    expect(response.success).toBe(true);
    expect(response.data.gameType).toBe('chess');
  });
  
  test('should make move successfully', async () => {
    // Arrange
    const moveRequest = {
      move: {
        from: { row: 1, col: 4 },
        to: { row: 3, col: 4 },
        piece: 'pawn',
      },
    };
    
    await provider.addInteraction({
      state: 'game exists and it is player turn',
      uponReceiving: 'a request to make a move',
      withRequest: {
        method: 'POST',
        path: like('/api/v1/games/game-123/moves'),
        headers: {
          'Content-Type': 'application/json',
          'Authorization': like('Bearer token'),
        },
        body: moveRequest,
      },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          success: true,
          data: {
            valid: true,
            gameState: like({}),
            nextPlayer: like('user-456'),
          },
        },
      },
    });
    
    // Act
    const response = await gameApi.makeMove('game-123', moveRequest.move);
    
    // Assert
    expect(response.success).toBe(true);
    expect(response.data.valid).toBe(true);
  });
});

// Provider Contract Tests (Backend)
describe('Game API Provider Tests', () => {
  const opts = {
    provider: 'gaming-platform-backend',
    providerBaseUrl: 'http://localhost:3000',
    pactUrls: [
      path.resolve(process.cwd(), 'pacts', 'gaming-platform-frontend-gaming-platform-backend.json')
    ],
    stateHandlers: {
      'user is authenticated': async () => {
        // Setup authenticated user state
        await createTestUser('user-123');
        return 'User authenticated';
      },
      'game exists and it is player turn': async () => {
        // Setup game state
        await createTestGame('game-123', 'user-123');
        return 'Game ready for move';
      },
    },
    requestFilter: (req, res, next) => {
      // Add authentication token
      req.headers.authorization = 'Bearer valid-test-token';
      next();
    },
  };
  
  test('should satisfy all consumer contracts', async () => {
    const output = await new Verifier(opts).verifyProvider();
    expect(output).toContain('Pact verification successful');
  });
});
```

### API Integration Testing
```typescript
// Comprehensive API Test Suite
describe('API Integration Tests', () => {
  let testServer: TestServer;
  let authToken: string;
  
  beforeAll(async () => {
    testServer = await createTestServer();
    authToken = await authenticateTestUser(testServer);
  });
  
  afterAll(async () => {
    await testServer.cleanup();
  });
  
  describe('Game Management API', () => {
    test('complete game lifecycle', async () => {
      // Create game
      const createResponse = await testServer.request
        .post('/api/v1/games')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          gameType: 'chess',
          timeControl: { type: 'blitz', minutes: 5 },
          isRated: true,
        })
        .expect(201);
      
      const gameId = createResponse.body.data.id;
      expect(gameId).toBeDefined();
      
      // Join game as second player
      const secondPlayerToken = await authenticateSecondTestUser(testServer);
      await testServer.request
        .post(`/api/v1/games/${gameId}/join`)
        .set('Authorization', `Bearer ${secondPlayerToken}`)
        .expect(200);
      
      // Verify game started
      const gameResponse = await testServer.request
        .get(`/api/v1/games/${gameId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(gameResponse.body.data.status).toBe('active');
      
      // Make moves
      const moves = [
        { from: { row: 1, col: 4 }, to: { row: 3, col: 4 } }, // e2-e4
        { from: { row: 6, col: 4 }, to: { row: 4, col: 4 } }, // e7-e5
      ];
      
      for (let i = 0; i < moves.length; i++) {
        const token = i % 2 === 0 ? authToken : secondPlayerToken;
        
        await testServer.request
          .post(`/api/v1/games/${gameId}/moves`)
          .set('Authorization', `Bearer ${token}`)
          .send({ move: moves[i] })
          .expect(200);
      }
      
      // Verify move history
      const finalGameResponse = await testServer.request
        .get(`/api/v1/games/${gameId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(finalGameResponse.body.data.moves).toHaveLength(2);
    });
    
    test('should handle invalid moves', async () => {
      // Create and start game
      const gameId = await createAndStartTestGame(testServer, authToken);
      
      // Attempt invalid move
      const invalidMove = { from: { row: 0, col: 0 }, to: { row: 7, col: 7 } };
      
      const response = await testServer.request
        .post(`/api/v1/games/${gameId}/moves`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ move: invalidMove })
        .expect(400);
      
      expect(response.body.success).toBe(false);
      expect(response.body.error.type).toBe('INVALID_MOVE');
    });
  });
  
  describe('Tournament API', () => {
    test('tournament creation and management', async () => {
      // Create tournament
      const tournamentResponse = await testServer.request
        .post('/api/v1/tournaments')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          name: 'Test Tournament',
          gameType: 'chess',
          format: 'single-elimination',
          maxParticipants: 8,
          entryFee: 10,
          startTime: new Date(Date.now() + 3600000).toISOString(),
        })
        .expect(201);
      
      const tournamentId = tournamentResponse.body.data.id;
      
      // Join tournament
      await testServer.request
        .post(`/api/v1/tournaments/${tournamentId}/join`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      // Verify participation
      const participantsResponse = await testServer.request
        .get(`/api/v1/tournaments/${tournamentId}/participants`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(participantsResponse.body.data).toHaveLength(1);
    });
  });
  
  describe('User Management API', () => {
    test('user profile management', async () => {
      // Get current profile
      const profileResponse = await testServer.request
        .get('/api/v1/user/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      const originalProfile = profileResponse.body.data;
      
      // Update profile
      const updateData = {
        displayName: 'Updated Name',
        bio: 'Updated bio',
        preferences: {
          theme: 'dark',
          language: 'en',
        },
      };
      
      await testServer.request
        .patch('/api/v1/user/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .send(updateData)
        .expect(200);
      
      // Verify update
      const updatedProfileResponse = await testServer.request
        .get('/api/v1/user/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(updatedProfileResponse.body.data.displayName).toBe(updateData.displayName);
      expect(updatedProfileResponse.body.data.bio).toBe(updateData.bio);
    });
  });
});
```

## 9. Continuous Integration Testing

### CI/CD Pipeline Testing Configuration
```yaml
# GitHub Actions Workflow
name: Comprehensive Testing Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:unit -- --coverage --watchAll=false
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    services:
      mongodb:
        image: mongo:6.0
        ports:
          - 27017:27017
      redis:
        image: redis:7.0
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18.x
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        MONGODB_URI: mongodb://localhost:27017/test
        REDIS_URI: redis://localhost:6379

  

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18.x
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build applications
      run: |
        npm run build:backend
        npm run build:frontend
        npm run build:crm
    
    - name: Start test environment
      run: |
        docker-compose -f docker-compose.test.yml up -d
        npm run wait-for-services
    
    - name: Install Playwright
      run: npx playwright install --with-deps
    
    - name: Run E2E tests
      run: npm run test:e2e
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: playwright-report
        path: playwright-report/
        retention-days: 30

  security-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Run OWASP ZAP security scan
      uses: zaproxy/action-full-scan@v0.4.0
      with:
        target: 'http://localhost:3000'
        rules_file_name: '.zap/rules.tsv'
        cmd_options: '-a'
    
    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

  performance-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup K6
      run: |
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
        echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
        sudo apt-get update
        sudo apt-get install k6
    
    - name: Run performance tests
      run: k6 run tests/performance/load-test.js
    
    - name: Upload performance results
      uses: actions/upload-artifact@v3
      with:
        name: performance-results
        path: performance-results/

  accessibility-tests:
    runs-on: ubuntu-latest
    needs: e2e-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18.x
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run accessibility tests
      run: npm run test:a11y
    
    - name: Upload accessibility report
      uses: actions/upload-artifact@v3
      with:
        name: accessibility-report
        path: accessibility-report/

  visual-regression:
    runs-on: ubuntu-latest
    needs: e2e-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Chromatic visual tests
      uses: chromaui/action@v1
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
        buildScriptName: build-storybook
```

## 10. Test Data Management

### Test Data Strategy
```typescript
// Test Data Factory
class TestDataFactory {
  static createUser(overrides: Partial<User> = {}): User {
    return {
      id: faker.datatype.uuid(),
      username: faker.internet.userName(),
      email: faker.internet.email(),
      displayName: faker.name.fullName(),
      rating: {
        chess: faker.datatype.number({ min: 800, max: 2800 }),
        checkers: faker.datatype.number({ min: 800, max: 2800 }),
        backgammon: faker.datatype.number({ min: 800, max: 2800 }),
        tictactoe: faker.datatype.number({ min: 800, max: 2800 }),
      },
      statistics: {
        gamesPlayed: faker.datatype.number({ min: 0, max: 1000 }),
        gamesWon: faker.datatype.number({ min: 0, max: 500 }),
        gamesLost: faker.datatype.number({ min: 0, max: 500 }),
        gamesDrawn: faker.datatype.number({ min: 0, max: 100 }),
      },
      preferences: {
        theme: faker.helpers.arrayElement(['light', 'dark']),
        language: faker.helpers.arrayElement(['en', 'ru']),
        notifications: {
          email: faker.datatype.boolean(),
          push: faker.datatype.boolean(),
          gameInvites: faker.datatype.boolean(),
        },
      },
      subscription: {
        type: faker.helpers.arrayElement(['free', 'premium', 'pro']),
        expiresAt: faker.date.future(),
      },
      createdAt: faker.date.past(),
      updatedAt: faker.date.recent(),
      ...overrides,
    };
  }
  
  static createGame(overrides: Partial<Game> = {}): Game {
    const gameType = faker.helpers.arrayElement(['chess', 'checkers', 'backgammon', 'tictactoe']);
    
    return {
      id: faker.datatype.uuid(),
      gameType,
      status: faker.helpers.arrayElement(['waiting', 'active', 'completed', 'abandoned']),
      players: {
        player1: {
          userId: faker.datatype.uuid(),
          color: 'white',
          rating: faker.datatype.number({ min: 800, max: 2800 }),
        },
        player2: {
          userId: faker.datatype.uuid(),
          color: 'black',
          rating: faker.datatype.number({ min: 800, max: 2800 }),
        },
      },
      timeControl: {
        type: faker.helpers.arrayElement(['bullet', 'blitz', 'rapid', 'classical']),
        minutes: faker.helpers.arrayElement([1, 3, 5, 10, 15, 30]),
        increment: faker.datatype.number({ min: 0, max: 30 }),
      },
      moves: this.generateGameMoves(gameType),
      result: faker.helpers.arrayElement([null, 'white', 'black', 'draw']),
      isRated: faker.datatype.boolean(),
      createdAt: faker.date.past(),
      startedAt: faker.date.recent(),
      endedAt: faker.date.recent(),
      ...overrides,
    };
  }
  
  static createTournament(overrides: Partial<Tournament> = {}): Tournament {
    return {
      id: faker.datatype.uuid(),
      name: faker.company.name() + ' Tournament',
      description: faker.lorem.paragraph(),
      gameType: faker.helpers.arrayElement(['chess', 'checkers', 'backgammon', 'tictactoe']),
      format: faker.helpers.arrayElement(['single-elimination', 'double-elimination', 'round-robin', 'swiss']),
      status: faker.helpers.arrayElement(['upcoming', 'registration', 'active', 'completed', 'cancelled']),
      maxParticipants: faker.helpers.arrayElement([8, 16, 32, 64, 128]),
      currentParticipants: faker.datatype.number({ min: 0, max: 64 }),
      entryFee: faker.datatype.number({ min: 0, max: 100 }),
      prizePool: faker.datatype.number({ min: 0, max: 10000 }),
      timeControl: {
        type: faker.helpers.arrayElement(['blitz', 'rapid']),
        minutes: faker.helpers.arrayElement([5, 10, 15]),
        increment: faker.datatype.number({ min: 0, max: 10 }),
      },
      startTime: faker.date.future(),
      endTime: faker.date.future(),
      createdBy: faker.datatype.uuid(),
      createdAt: faker.date.past(),
      ...overrides,
    };
  }
  
  private static generateGameMoves(gameType: string): Move[] {
    const moveCount = faker.datatype.number({ min: 0, max: 50 });
    const moves: Move[] = [];
    
    for (let i = 0; i < moveCount; i++) {
      switch (gameType) {
        case 'chess':
          moves.push(this.generateChessMove());
          break;
        case 'checkers':
          moves.push(this.generateCheckersMove());
          break;
        case 'backgammon':
          moves.push(this.generateBackgammonMove());
          break;
        case 'tictactoe':
          moves.push(this.generateTicTacToeMove());
          break;
      }
    }
    
    return moves;
  }
  
  private static generateChessMove(): ChessMove {
    return {
      from: {
        row: faker.datatype.number({ min: 0, max: 7 }),
        col: faker.datatype.number({ min: 0, max: 7 }),
      },
      to: {
        row: faker.datatype.number({ min: 0, max: 7 }),
        col: faker.datatype.number({ min: 0, max: 7 }),
      },
      piece: faker.helpers.arrayElement(['pawn', 'rook', 'knight', 'bishop', 'queen', 'king']),
      captured: faker.helpers.maybe(() => 
        faker.helpers.arrayElement(['pawn', 'rook', 'knight', 'bishop', 'queen'])
      ),
      promotion: faker.helpers.maybe(() => 
        faker.helpers.arrayElement(['queen', 'rook', 'bishop', 'knight'])
      ),
      castling: faker.helpers.maybe(() => 
        faker.helpers.arrayElement(['kingside', 'queenside'])
      ),
      enPassant: faker.datatype.boolean(),
      check: faker.datatype.boolean(),
      checkmate: faker.datatype.boolean(),
    };
  }
}

// Test Database Seeding
class TestDatabaseSeeder {
  constructor(private db: Database) {}
  
  async seedUsers(count: number = 100): Promise<User[]> {
    const users = Array.from({ length: count }, () => TestDataFactory.createUser());
    await this.db.collection('users').insertMany(users);
    return users;
  }
  
  async seedGames(count: number = 500, users?: User[]): Promise<Game[]> {
    const testUsers = users || await this.seedUsers(50);
    const games = Array.from({ length: count }, () => {
      const player1 = faker.helpers.arrayElement(testUsers);
      const player2 = faker.helpers.arrayElement(testUsers.filter(u => u.id !== player1.id));
      
      return TestDataFactory.createGame({
        players: {
          player1: { userId: player1.id, color: 'white', rating: player1.rating.chess },
          player2: { userId: player2.id, color: 'black', rating: player2.rating.chess },
        },
      });
    });
    
    await this.db.collection('games').insertMany(games);
    return games;
  }
  
  async seedTournaments(count: number = 20): Promise<Tournament[]> {
    const tournaments = Array.from({ length: count }, () => TestDataFactory.createTournament());
    await this.db.collection('tournaments').insertMany(tournaments);
    return tournaments;
  }
  
  async seedCompleteDataset(): Promise<TestDataset> {
    const users = await this.seedUsers(200);
    const games = await this.seedGames(1000, users);
    const tournaments = await this.seedTournaments(50);
    
    return { users, games, tournaments };
  }
  
  async cleanup(): Promise<void> {
    await Promise.all([
      this.db.collection('users').deleteMany({}),
      this.db.collection('games').deleteMany({}),
      this.db.collection('tournaments').deleteMany({}),
    ]);
  }
}

interface TestDataset {
  users: User[];
  games: Game[];
  tournaments: Tournament[];
}
```

## 11. Quality Metrics and Reporting

### Quality Gates Configuration
```typescript
// Quality Gates Definition
interface QualityGates {
  code_coverage: {
    minimum_threshold: 80;
    critical_path_threshold: 90;
    new_code_threshold: 85;
  };
  
  code_quality: {
    sonarqube_quality_gate: 'passed';
    technical_debt_ratio: '<5%';
    duplicated_lines_density: '<3%';
    maintainability_rating: 'A';
    reliability_rating: 'A';
    security_rating: 'A';
  };
  
  performance: {
    api_response_time_p95: '<200ms';
    frontend_lighthouse_performance: '>90';
    bundle_size_limit: '<2MB';
    time_to_interactive: '<3s';
  };
  
  security: {
    vulnerability_scan: 'passed';
    dependency_check: 'no_high_vulnerabilities';
    penetration_test: 'passed';
  };
  
  accessibility: {
    wcag_compliance: 'AA';
    axe_violations: 0;
    color_contrast_ratio: '>4.5';
  };
}

// Quality Metrics Collection
class QualityMetricsCollector {
  async collectCodeCoverage(): Promise<CoverageMetrics> {
    const coverage = await this.runCoverageAnalysis();
    
    return {
      overall: coverage.overall,
      byComponent: coverage.byComponent,
      criticalPaths: coverage.criticalPaths,
      newCode: coverage.newCode,
      trends: await this.getCoverageTrends(),
    };
  }
  
  async collectPerformanceMetrics(): Promise<PerformanceMetrics> {
    const [apiMetrics, frontendMetrics] = await Promise.all([
      this.collectAPIPerformanceMetrics(),
      this.collectFrontendPerformanceMetrics(),
    ]);
    
    return {
      api: apiMetrics,
      frontend: frontendMetrics,
      trends: await this.getPerformanceTrends(),
    };
  }
  
  async collectSecurityMetrics(): Promise<SecurityMetrics> {
    const [vulnerabilities, dependencies, penetrationResults] = await Promise.all([
      this.scanVulnerabilities(),
      this.checkDependencies(),
      this.runPenetrationTests(),
    ]);
    
    return {
      vulnerabilities,
      dependencies,
      penetrationResults,
      trends: await this.getSecurityTrends(),
    };
  }
  
  async generateQualityReport(): Promise<QualityReport> {
    const [coverage, performance, security, accessibility] = await Promise.all([
      this.collectCodeCoverage(),
      this.collectPerformanceMetrics(),
      this.collectSecurityMetrics(),
      this.collectAccessibilityMetrics(),
    ]);
    
    const qualityScore = this.calculateQualityScore({
      coverage,
      performance,
      security,
      accessibility,
    });
    
    return {
      timestamp: new Date().toISOString(),
      qualityScore,
      coverage,
      performance,
      security,
      accessibility,
      recommendations: this.generateRecommendations({
        coverage,
        performance,
        security,
        accessibility,
      }),
    };
  }
  
  private calculateQualityScore(metrics: QualityMetrics): number {
    const weights = {
      coverage: 0.25,
      performance: 0.25,
      security: 0.30,
      accessibility: 0.20,
    };
    
    const scores = {
      coverage: this.scoreCoverage(metrics.coverage),
      performance: this.scorePerformance(metrics.performance),
      security: this.scoreSecurity(metrics.security),
      accessibility: this.scoreAccessibility(metrics.accessibility),
    };
    
    return Object.entries(weights).reduce(
      (total, [key, weight]) => total + scores[key] * weight,
      0
    );
  }
}

// Automated Quality Reporting
class QualityReporter {
  async generateDailyReport(): Promise<void> {
    const metrics = await new QualityMetricsCollector().generateQualityReport();
    
    // Generate HTML report
    const htmlReport = await this.generateHTMLReport(metrics);
    await this.saveReport('daily', htmlReport);
    
    // Send notifications if quality gates fail
    const failedGates = this.checkQualityGates(metrics);
    if (failedGates.length > 0) {
      await this.sendQualityAlert(failedGates);
    }
    
    // Update quality dashboard
    await this.updateQualityDashboard(metrics);
  }
  
  async generateWeeklyTrendReport(): Promise<void> {
    const weeklyMetrics = await this.collectWeeklyMetrics();
    const trendAnalysis = this.analyzeTrends(weeklyMetrics);
    
    const report = await this.generateTrendReport(trendAnalysis);
    await this.sendWeeklyReport(report);
  }
  
  private checkQualityGates(metrics: QualityReport): string[] {
    const failedGates: string[] = [];
    
    if (metrics.coverage.overall < 80) {
      failedGates.push('Code coverage below 80%');
    }
    
    if (metrics.performance.frontend.lighthouse < 90) {
      failedGates.push('Frontend performance below 90');
    }
    
    if (metrics.security.vulnerabilities.high > 0) {
      failedGates.push('High severity vulnerabilities found');
    }
    
    if (metrics.accessibility.violations > 0) {
      failedGates.push('Accessibility violations found');
    }
    
    return failedGates;
  }
}
```

## 12. Test Environment Management

### Environment Configuration
```typescript
// Test Environment Manager
class TestEnvironmentManager {
  private environments: Map<string, TestEnvironment> = new Map();
  
  async createEnvironment(name: string, config: EnvironmentConfig): Promise<TestEnvironment> {
    const environment = new TestEnvironment(name, config);
    await environment.setup();
    
    this.environments.set(name, environment);
    return environment;
  }
  
  async destroyEnvironment(name: string): Promise<void> {
    const environment = this.environments.get(name);
    if (environment) {
      await environment.teardown();
      this.environments.delete(name);
    }
  }
  
  async resetEnvironment(name: string): Promise<void> {
    const environment = this.environments.get(name);
    if (environment) {
      await environment.reset();
    }
  }
  
  getEnvironment(name: string): TestEnvironment | undefined {
    return this.environments.get(name);
  }
}

class TestEnvironment {
  private containers: Map<string, Container> = new Map();
  private databases: Map<string, TestDatabase> = new Map();
  
  constructor(
    private name: string,
    private config: EnvironmentConfig
  ) {}
  
  async setup(): Promise<void> {
    // Start infrastructure containers
    await this.startInfrastructure();
    
    // Setup databases
    await this.setupDatabases();
    
    // Deploy applications
    await this.deployApplications();
    
    // Wait for health checks
    await this.waitForHealthy();
    
    // Seed test data
    await this.seedTestData();
  }
  
  async teardown(): Promise<void> {
    // Stop applications
    await this.stopApplications();
    
    // Cleanup databases
    await this.cleanupDatabases();
    
    // Stop infrastructure
    await this.stopInfrastructure();
  }
  
  async reset(): Promise<void> {
    // Reset databases
    await this.resetDatabases();
    
    // Reseed test data
    await this.seedTestData();
    
    // Clear caches
    await this.clearCaches();
  }
  
  private async startInfrastructure(): Promise<void> {
    const services = ['mongodb', 'redis', 'elasticsearch'];
    
    for (const service of services) {
      const container = await this.startContainer(service, this.config.services[service]);
      this.containers.set(service, container);
    }
  }
  
  private async setupDatabases(): Promise<void> {
    // Setup MongoDB
    const mongodb = new TestDatabase('mongodb', this.config.databases.mongodb);
    await mongodb.setup();
    this.databases.set('mongodb', mongodb);
    
    // Setup Redis
    const redis = new TestDatabase('redis', this.config.databases.redis);
    await redis.setup();
    this.databases.set('redis', redis);
  }
  
  private async seedTestData(): Promise<void> {
    const seeder = new TestDatabaseSeeder(this.databases.get('mongodb')!);
    await seeder.seedCompleteDataset();
  }
}

// Docker Compose Test Configuration
interface DockerComposeTestConfig {
  version: '3.8';
  services: {
    mongodb: {
      image: 'mongo:6.0';
      ports: ['27017:27017'];
      environment: {
        MONGO_INITDB_ROOT_USERNAME: 'test';
        MONGO_INITDB_ROOT_PASSWORD: 'test';
      };
      volumes: ['./test-data/mongodb:/docker-entrypoint-initdb.d'];
    };
    
    redis: {
      image: 'redis:7.0';
      ports: ['6379:6379'];
      command: 'redis-server --requirepass test';
    };
    
    backend: {
      build: './backend';
      ports: ['3000:3000'];
      environment: {
        NODE_ENV: 'test';
        MONGODB_URI: 'mongodb://test:test@mongodb:27017/test';
        REDIS_URI: 'redis://:test@redis:6379';
      };
      depends_on: ['mongodb', 'redis'];
    };
    
    frontend: {
      build: './frontend';
      ports: ['3001:80'];
      environment: {
        REACT_APP_API_URL: 'http://backend:3000';
      };
      depends_on: ['backend'];
    };
    
    crm: {
      build: './crm';
      ports: ['3002:80'];
      environment: {
        REACT_APP_API_URL: 'http://backend:3000';
      };
      depends_on: ['backend'];
    };
  };
  
  networks: {
    test_network: {
      driver: 'bridge';
    };
  };
}
```

## Summary

This comprehensive testing and quality assurance strategy provides:

### Testing Coverage
- **Unit Testing**: 70% of testing effort with Jest and React Testing Library
- **Integration Testing**: 20% focusing on component interactions and API contracts
- **End-to-End Testing**: 10% covering critical user journeys with Playwright

### Quality Assurance Framework
- **Code Quality**: ESLint, Prettier, SonarQube integration
- **Performance Testing**: Load testing with K6, frontend performance with Lighthouse
- **Security Testing**: OWASP ZAP, Snyk, penetration testing automation
- **Accessibility Testing**: WCAG 2.1 AA compliance with axe-core

### Continuous Testing Pipeline
- **CI/CD Integration**: Automated testing in GitHub Actions
- **Quality Gates**: Enforced thresholds for coverage, performance, security
- **Test Environment Management**: Automated setup and teardown
- **Reporting**: Comprehensive quality metrics and trend analysis

### Key Testing Principles
1. **Test Pyramid Approach**: Focus on fast, reliable unit tests
2. **Shift-Left Testing**: Early integration in development cycle
3. **Risk-Based Testing**: Prioritize high-risk areas
4. **Continuous Feedback**: Real-time quality metrics and alerts

This strategy ensures the gaming platform maintains high quality, security, and performance standards throughout the development lifecycle while providing comprehensive coverage across all applications and user scenarios.