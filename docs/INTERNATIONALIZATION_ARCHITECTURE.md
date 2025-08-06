# Internationalization Architecture

## Overview

This document outlines the comprehensive internationalization (i18n) architecture for the gaming platform, supporting English and Russian languages with dynamic language switching, localized content management, and cultural adaptation across all applications.

## 1. Internationalization System Overview

```mermaid
graph TB
    subgraph "Client Applications"
        CLIENT_EN[Client App (EN)]
        CLIENT_RU[Client App (RU)]
        CRM_EN[CRM App (EN)]
        CRM_RU[CRM App (RU)]
    end
    
    subgraph "i18n Management Layer"
        LANGUAGE_DETECTOR[Language Detection]
        LOCALE_MANAGER[Locale Manager]
        TRANSLATION_ENGINE[Translation Engine]
        CONTENT_MANAGER[Content Manager]
    end
    
    subgraph "Translation Resources"
        TRANSLATION_API[Translation API]
        RESOURCE_LOADER[Resource Loader]
        CACHE_MANAGER[Translation Cache]
        FALLBACK_HANDLER[Fallback Handler]
    end
    
    subgraph "Content Storage"
        TRANSLATION_DB[(Translation Database)]
        STATIC_RESOURCES[(Static Resources)]
        DYNAMIC_CONTENT[(Dynamic Content)]
        MEDIA_ASSETS[(Localized Media)]
    end
    
    subgraph "Backend Services"
        EMAIL_TEMPLATES[Email Templates]
        NOTIFICATION_SERVICE[Notification Service]
        GAME_CONTENT[Game Content]
        LEGAL_CONTENT[Legal Content]
    end
    
    CLIENT_EN --> LANGUAGE_DETECTOR
    CLIENT_RU --> LANGUAGE_DETECTOR
    CRM_EN --> LANGUAGE_DETECTOR
    CRM_RU --> LANGUAGE_DETECTOR
    
    LANGUAGE_DETECTOR --> LOCALE_MANAGER
    LOCALE_MANAGER --> TRANSLATION_ENGINE
    TRANSLATION_ENGINE --> CONTENT_MANAGER
    
    TRANSLATION_ENGINE --> TRANSLATION_API
    TRANSLATION_API --> RESOURCE_LOADER
    RESOURCE_LOADER --> CACHE_MANAGER
    CACHE_MANAGER --> FALLBACK_HANDLER
    
    RESOURCE_LOADER --> TRANSLATION_DB
    RESOURCE_LOADER --> STATIC_RESOURCES
    CONTENT_MANAGER --> DYNAMIC_CONTENT
    CONTENT_MANAGER --> MEDIA_ASSETS
    
    TRANSLATION_ENGINE --> EMAIL_TEMPLATES
    TRANSLATION_ENGINE --> NOTIFICATION_SERVICE
    TRANSLATION_ENGINE --> GAME_CONTENT
    TRANSLATION_ENGINE --> LEGAL_CONTENT
```

## 2. Core Internationalization Architecture

### Language Configuration
```typescript
interface LanguageConfig {
  code: string;           // ISO 639-1 language code
  name: string;           // Native language name
  englishName: string;    // English language name
  direction: 'ltr' | 'rtl'; // Text direction
  locale: string;         // Full locale (language-country)
  currency: string;       // Default currency
  dateFormat: string;     // Date format pattern
  timeFormat: string;     // Time format pattern
  numberFormat: NumberFormatConfig;
  enabled: boolean;       // Whether language is active
  fallback?: string;      // Fallback language code
}

interface NumberFormatConfig {
  decimal: string;        // Decimal separator
  thousands: string;      // Thousands separator
  currency: {
    symbol: string;
    position: 'before' | 'after';
    space: boolean;
  };
}

const SUPPORTED_LANGUAGES: LanguageConfig[] = [
  {
    code: 'en',
    name: 'English',
    englishName: 'English',
    direction: 'ltr',
    locale: 'en-US',
    currency: 'USD',
    dateFormat: 'MM/DD/YYYY',
    timeFormat: 'HH:mm',
    numberFormat: {
      decimal: '.',
      thousands: ',',
      currency: {
        symbol: '$',
        position: 'before',
        space: false
      }
    },
    enabled: true
  },
  {
    code: 'ru',
    name: 'Русский',
    englishName: 'Russian',
    direction: 'ltr',
    locale: 'ru-RU',
    currency: 'RUB',
    dateFormat: 'DD.MM.YYYY',
    timeFormat: 'HH:mm',
    numberFormat: {
      decimal: ',',
      thousands: ' ',
      currency: {
        symbol: '₽',
        position: 'after',
        space: true
      }
    },
    enabled: true,
    fallback: 'en'
  }
];
```

### Translation Resource Structure
```typescript
interface TranslationResource {
  namespace: string;      // Resource namespace (e.g., 'common', 'auth', 'games')
  language: string;       // Language code
  version: string;        // Resource version for cache busting
  translations: Record<string, any>; // Nested translation object
  metadata: {
    lastUpdated: Date;
    translator?: string;
    reviewStatus: 'pending' | 'approved' | 'rejected';
    completeness: number; // Percentage of translated keys
  };
}

// Example translation structure
const TRANSLATION_STRUCTURE = {
  common: {
    buttons: {
      save: 'Save',
      cancel: 'Cancel',
      submit: 'Submit',
      back: 'Back',
      next: 'Next',
      finish: 'Finish'
    },
    messages: {
      success: 'Operation completed successfully',
      error: 'An error occurred',
      loading: 'Loading...',
      noData: 'No data available'
    },
    navigation: {
      home: 'Home',
      games: 'Games',
      tournaments: 'Tournaments',
      profile: 'Profile',
      settings: 'Settings'
    }
  },
  auth: {
    login: {
      title: 'Sign In',
      username: 'Username',
      password: 'Password',
      rememberMe: 'Remember me',
      forgotPassword: 'Forgot password?',
      signIn: 'Sign In',
      noAccount: "Don't have an account?",
      signUp: 'Sign Up'
    },
    register: {
      title: 'Create Account',
      username: 'Username',
      email: 'Email',
      password: 'Password',
      confirmPassword: 'Confirm Password',
      agreeTerms: 'I agree to the Terms of Service',
      createAccount: 'Create Account',
      haveAccount: 'Already have an account?'
    }
  },
  games: {
    chess: {
      name: 'Chess',
      pieces: {
        king: 'King',
        queen: 'Queen',
        rook: 'Rook',
        bishop: 'Bishop',
        knight: 'Knight',
        pawn: 'Pawn'
      },
      moves: {
        check: 'Check',
        checkmate: 'Checkmate',
        stalemate: 'Stalemate',
        castling: 'Castling',
        enPassant: 'En passant'
      }
    },
    checkers: {
      name: 'Checkers',
      pieces: {
        man: 'Man',
        king: 'King'
      },
      moves: {
        jump: 'Jump',
        multipleJump: 'Multiple jump',
        promotion: 'Promotion'
      }
    }
  }
};
```

## 3. Frontend Internationalization

### React i18n Implementation
```typescript
// i18n configuration
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import Backend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

class I18nManager {
  private static instance: I18nManager;
  private currentLanguage: string = 'en';
  private loadedNamespaces: Set<string> = new Set();

  static getInstance(): I18nManager {
    if (!I18nManager.instance) {
      I18nManager.instance = new I18nManager();
    }
    return I18nManager.instance;
  }

  async initialize(): Promise<void> {
    await i18n
      .use(Backend)
      .use(LanguageDetector)
      .use(initReactI18next)
      .init({
        lng: this.detectLanguage(),
        fallbackLng: 'en',
        debug: process.env.NODE_ENV === 'development',
        
        interpolation: {
          escapeValue: false, // React already escapes
          format: (value, format, lng) => {
            return this.formatValue(value, format, lng);
          }
        },

        backend: {
          loadPath: '/api/i18n/{{lng}}/{{ns}}',
          addPath: '/api/i18n/add/{{lng}}/{{ns}}',
          allowMultiLoading: false,
          crossDomain: false,
          withCredentials: false,
          requestOptions: {
            cache: 'default'
          }
        },

        detection: {
          order: ['localStorage', 'navigator', 'htmlTag'],
          lookupLocalStorage: 'i18nextLng',
          caches: ['localStorage'],
          excludeCacheFor: ['cimode']
        },

        react: {
          useSuspense: false,
          bindI18n: 'languageChanged loaded',
          bindI18nStore: 'added removed',
          transEmptyNodeValue: '',
          transSupportBasicHtmlNodes: true,
          transKeepBasicHtmlNodesFor: ['br', 'strong', 'i', 'em']
        }
      });

    // Load initial namespaces
    await this.loadNamespaces(['common', 'auth']);
  }

  private detectLanguage(): string {
    // Check user preference
    const userLang = localStorage.getItem('userLanguage');
    if (userLang && this.isLanguageSupported(userLang)) {
      return userLang;
    }

    // Check browser language
    const browserLang = navigator.language.split('-')[0];
    if (this.isLanguageSupported(browserLang)) {
      return browserLang;
    }

    // Default to English
    return 'en';
  }

  private isLanguageSupported(lang: string): boolean {
    return SUPPORTED_LANGUAGES.some(l => l.code === lang && l.enabled);
  }

  async changeLanguage(language: string): Promise<void> {
    if (!this.isLanguageSupported(language)) {
      throw new Error(`Language ${language} is not supported`);
    }

    this.currentLanguage = language;
    await i18n.changeLanguage(language);
    
    // Save preference
    localStorage.setItem('userLanguage', language);
    
    // Update document language
    document.documentElement.lang = language;
    
    // Update document direction
    const config = SUPPORTED_LANGUAGES.find(l => l.code === language);
    if (config) {
      document.documentElement.dir = config.direction;
    }

    // Emit language change event
    window.dispatchEvent(new CustomEvent('languageChanged', { 
      detail: { language, config } 
    }));
  }

  async loadNamespaces(namespaces: string[]): Promise<void> {
    const newNamespaces = namespaces.filter(ns => !this.loadedNamespaces.has(ns));
    
    if (newNamespaces.length > 0) {
      await i18n.loadNamespaces(newNamespaces);
      newNamespaces.forEach(ns => this.loadedNamespaces.add(ns));
    }
  }

  private formatValue(value: any, format: string, language: string): string {
    const config = SUPPORTED_LANGUAGES.find(l => l.code === language);
    if (!config) return value;

    switch (format) {
      case 'currency':
        return this.formatCurrency(value, config);
      case 'date':
        return this.formatDate(value, config);
      case 'time':
        return this.formatTime(value, config);
      case 'number':
        return this.formatNumber(value, config);
      default:
        return value;
    }
  }

  private formatCurrency(amount: number, config: LanguageConfig): string {
    const formatted = this.formatNumber(amount, config);
    const { symbol, position, space } = config.numberFormat.currency;
    const separator = space ? ' ' : '';
    
    return position === 'before' 
      ? `${symbol}${separator}${formatted}`
      : `${formatted}${separator}${symbol}`;
  }

  private formatNumber(value: number, config: LanguageConfig): string {
    const { decimal, thousands } = config.numberFormat;
    const parts = value.toFixed(2).split('.');
    parts[0] = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, thousands);
    return parts.join(decimal);
  }

  private formatDate(date: Date | string, config: LanguageConfig): string {
    const dateObj = typeof date === 'string' ? new Date(date) : date;
    return new Intl.DateTimeFormat(config.locale).format(dateObj);
  }

  private formatTime(time: Date | string, config: LanguageConfig): string {
    const timeObj = typeof time === 'string' ? new Date(time) : time;
    return new Intl.DateTimeFormat(config.locale, {
      hour: '2-digit',
      minute: '2-digit'
    }).format(timeObj);
  }
}
```

### Translation Hooks and Components
```typescript
// Custom hooks for translations
import { useTranslation } from 'react-i18next';
import { useCallback, useEffect, useState } from 'react';

export const useGameTranslation = (gameType: string) => {
  const { t, i18n } = useTranslation(['games', 'common']);
  
  const gameT = useCallback((key: string, options?: any) => {
    return t(`games:${gameType}.${key}`, options);
  }, [t, gameType]);

  return { t: gameT, language: i18n.language };
};

export const useFormattedTranslation = () => {
  const { t, i18n } = useTranslation();
  
  const formatT = useCallback((key: string, values: Record<string, any> = {}) => {
    return t(key, {
      ...values,
      formatParams: {
        ...values,
        lng: i18n.language
      }
    });
  }, [t, i18n.language]);

  return { t: formatT, language: i18n.language };
};

// Language switcher component
export const LanguageSwitcher: React.FC = () => {
  const { i18n } = useTranslation();
  const [currentLanguage, setCurrentLanguage] = useState(i18n.language);

  const handleLanguageChange = async (language: string) => {
    try {
      await I18nManager.getInstance().changeLanguage(language);
      setCurrentLanguage(language);
    } catch (error) {
      console.error('Failed to change language:', error);
    }
  };

  return (
    <div className="language-switcher">
      {SUPPORTED_LANGUAGES.filter(lang => lang.enabled).map(lang => (
        <button
          key={lang.code}
          className={`language-option ${currentLanguage === lang.code ? 'active' : ''}`}
          onClick={() => handleLanguageChange(lang.code)}
        >
          <span className="language-name">{lang.name}</span>
          <span className="language-code">{lang.code.toUpperCase()}</span>
        </button>
      ))}
    </div>
  );
};

// Localized content component
export const LocalizedContent: React.FC<{
  contentKey: string;
  namespace?: string;
  values?: Record<string, any>;
  components?: Record<string, React.ReactElement>;
}> = ({ contentKey, namespace = 'common', values = {}, components = {} }) => {
  const { t } = useTranslation(namespace);
  
  return (
    <Trans
      i18nKey={contentKey}
      values={values}
      components={components}
      ns={namespace}
    />
  );
};
```

## 4. Backend Internationalization

### Server-side Translation Service
```typescript
class ServerTranslationService {
  private translations: Map<string, Map<string, any>> = new Map();
  private fallbackLanguage = 'en';

  constructor(
    private translationRepository: TranslationRepository,
    private cacheService: CacheService
  ) {}

  async loadTranslations(language: string, namespace: string): Promise<void> {
    const cacheKey = `translations:${language}:${namespace}`;
    
    // Try cache first
    let translations = await this.cacheService.get(cacheKey);
    
    if (!translations) {
      // Load from database
      const resource = await this.translationRepository.findByLanguageAndNamespace(language, namespace);
      
      if (resource) {
        translations = resource.translations;
        // Cache for 1 hour
        await this.cacheService.set(cacheKey, translations, 3600);
      } else if (language !== this.fallbackLanguage) {
        // Load fallback language
        await this.loadTranslations(this.fallbackLanguage, namespace);
        return;
      }
    }

    if (translations) {
      const key = `${language}:${namespace}`;
      this.translations.set(key, translations);
    }
  }

  translate(key: string, language: string, namespace: string = 'common', values: Record<string, any> = {}): string {
    const translationKey = `${language}:${namespace}`;
    let translations = this.translations.get(translationKey);

    // Try fallback if translation not found
    if (!translations && language !== this.fallbackLanguage) {
      const fallbackKey = `${this.fallbackLanguage}:${namespace}`;
      translations = this.translations.get(fallbackKey);
    }

    if (!translations) {
      return key; // Return key if no translations found
    }

    // Navigate nested object
    const translation = this.getNestedValue(translations, key);
    
    if (typeof translation !== 'string') {
      return key;
    }

    // Interpolate values
    return this.interpolate(translation, values, language);
  }

  private getNestedValue(obj: any, path: string): any {
    return path.split('.').reduce((current, key) => {
      return current && current[key] !== undefined ? current[key] : undefined;
    }, obj);
  }

  private interpolate(template: string, values: Record<string, any>, language: string): string {
    return template.replace(/\{\{(\w+)(?::(\w+))?\}\}/g, (match, key, format) => {
      const value = values[key];
      
      if (value === undefined) {
        return match;
      }

      if (format) {
        return this.formatValue(value, format, language);
      }

      return String(value);
    });
  }

  private formatValue(value: any, format: string, language: string): string {
    const config = SUPPORTED_LANGUAGES.find(l => l.code === language);
    if (!config) return String(value);

    switch (format) {
      case 'currency':
        return this.formatCurrency(Number(value), config);
      case 'date':
        return this.formatDate(new Date(value), config);
      case 'number':
        return this.formatNumber(Number(value), config);
      default:
        return String(value);
    }
  }

  // Email template translation
  async translateEmailTemplate(templateKey: string, language: string, values: Record<string, any> = {}): Promise<EmailTemplate> {
    await this.loadTranslations(language, 'email');
    
    const subject = this.translate(`${templateKey}.subject`, language, 'email', values);
    const body = this.translate(`${templateKey}.body`, language, 'email', values);
    const htmlBody = this.translate(`${templateKey}.htmlBody`, language, 'email', values);

    return {
      subject,
      body,
      htmlBody: htmlBody || this.convertToHtml(body)
    };
  }

  // Notification translation
  async translateNotification(notificationKey: string, language: string, values: Record<string, any> = {}): Promise<NotificationContent> {
    await this.loadTranslations(language, 'notifications');
    
    const title = this.translate(`${notificationKey}.title`, language, 'notifications', values);
    const message = this.translate(`${notificationKey}.message`, language, 'notifications', values);

    return { title, message };
  }
}
```

## 5. Dynamic Content Management

### Content Management System
```typescript
class LocalizedContentManager {
  constructor(
    private contentRepository: ContentRepository,
    private translationService: ServerTranslationService,
    private cacheService: CacheService
  ) {}

  async getLocalizedContent(contentId: string, language: string): Promise<LocalizedContent> {
    const cacheKey = `content:${contentId}:${language}`;
    
    // Try cache first
    let content = await this.cacheService.get(cacheKey);
    
    if (!content) {
      // Load from database
      content = await this.contentRepository.findByIdAndLanguage(contentId, language);
      
      if (!content && language !== 'en') {
        // Fallback to English
        content = await this.contentRepository.findByIdAndLanguage(contentId, 'en');
      }

      if (content) {
        // Cache for 30 minutes
        await this.cacheService.set(cacheKey, content, 1800);
      }
    }

    return content || this.getDefaultContent(contentId);
  }

  async updateLocalizedContent(contentId: string, language: string, content: Partial<LocalizedContent>): Promise<void> {
    await this.contentRepository.updateContent(contentId, language, content);
    
    // Invalidate cache
    const cacheKey = `content:${contentId}:${language}`;
    await this.cacheService.delete(cacheKey);
  }

  async getGameContent(gameType: string, language: string): Promise<GameContent> {
    const content = await this.getLocalizedContent(`game:${gameType}`, language);
    
    return {
      name: content.title,
      description: content.description,
      rules: content.rules,
      instructions: content.instructions,
      tips: content.tips || []
    };
  }

  async getLegalContent(documentType: string, language: string): Promise<LegalDocument> {
    const content = await this.getLocalizedContent(`legal:${documentType}`, language);
    
    return {
      title: content.title,
      content: content.body,
      lastUpdated: content.updatedAt,
      version: content.version
    };
  }

  private getDefaultContent(contentId: string): LocalizedContent {
    return {
      id: contentId,
      language: 'en',
      title: 'Content not available',
      description: 'This content is not available in the requested language.',
      body: '',
      createdAt: new Date(),
      updatedAt: new Date()
    };
  }
}
```

## 6. Localized Email and Notifications

### Email Template System
```typescript
class LocalizedEmailService {
  constructor(
    private translationService: ServerTranslationService,
    private emailService: EmailService
  ) {}

  async sendLocalizedEmail(
    to: string,
    templateKey: string,
    language: string,
    values: Record<string, any> = {}
  ): Promise<void> {
    const template = await this.translationService.translateEmailTemplate(templateKey, language, values);
    
    await this.emailService.send({
      to,
      subject: template.subject,
      html: template.htmlBody,
      text: template.body
    });
  }

  async sendWelcomeEmail(userEmail: string, username: string, language: string): Promise<void> {
    await this.sendLocalizedEmail(userEmail, 'welcome', language, {
      username,
      loginUrl: `${process.env.CLIENT_URL}/login`,
      supportEmail: process.env.SUPPORT_EMAIL
    });
  }

  async sendPasswordResetEmail(userEmail: string, resetToken: string, language: string): Promise<void> {
    await this.sendLocalizedEmail(userEmail, 'passwordReset', language, {
      resetUrl: `${process.env.CLIENT_URL}/reset-password?token=${resetToken}`,
      expiryTime: '24 hours'
    });
  }

  async sendGameInviteEmail(userEmail: string, inviterName: string, gameType: string, language: string): Promise<void> {
    const gameContent = await this.getGameContent(gameType, language);
    
    await this.sendLocalizedEmail(userEmail, 'gameInvite', language, {
      inviterName,
      gameName: gameContent.name,
      gameUrl: `${process.env.CLIENT_URL}/games/${gameType}`
    });
  }

  async sendTournamentNotificationEmail(
    userEmail: string,
    tournamentName: string,
    eventType: string,
    language: string
  ): Promise<void> {
    await this.sendLocalizedEmail(userEmail, `tournament.${eventType}`, language, {
      tournamentName,
      tournamentUrl: `${process.env.CLIENT_URL}/tournaments`
    });
  }
}
```

### Push Notification Service
```typescript
class LocalizedNotificationService {
  constructor(
    private translationService: ServerTranslationService,
    private pushService: PushNotificationService,
    private socketService: SocketService
  ) {}

  async sendLocalizedNotification(
    userId: string,
    notificationKey: string,
    values: Record<string, any> = {}
  ): Promise<void> {
    const user = await this.getUserWithLanguage(userId);
    const language = user.preferences.language || 'en';
    
    const content = await this.translationService.translateNotification(notificationKey, language, values);
    
    // Send push notification
    await this.pushService.send(userId, {
      title: content.title,
      body: content.message,
      data: values
    });

    // Send real-time notification
    await this.socketService.sendToUser(userId, 'notification', {
      type: notificationKey,
      title: content.title,
      message: content.message,
      timestamp: new Date(),
      data: values
    });
  }

  async notifyGameResult(userId: string, gameResult: GameResult, language: string): Promise<void> {
    const notificationKey = gameResult.winner === userId ? 'game.won' : 'game.lost';
    
    await this.sendLocalizedNotification(userId, notificationKey, {
      gameType: gameResult.gameType,
      opponent: gameResult.opponentName,
      stake: gameResult.stake
    });
  }

  async notifyTournamentUpdate(userId: string, tournamentUpdate: TournamentUpdate, language: string): Promise<void> {
    await this.sendLocalizedNotification(userId, `tournament.${tournamentUpdate.type}`, {
      tournamentName: tournamentUpdate.tournamentName,
      round: tournamentUpdate.round,
      position: tournamentUpdate.position
    });
  }
}
```

## 7. Cultural Adaptation

### Cultural Adaptation Service
```typescript
class CulturalAdaptationService {
  private culturalSettings: Map<string, CulturalSettings> = new Map();

  constructor() {
    this.initializeCulturalSettings();
  }

  private initializeCulturalSettings(): void {
    this.culturalSettings.set('en', {
      dateFormat: 'MM/DD/YYYY',
      timeFormat: '12h',
      weekStart: 'sunday',
      currency: 'USD',
      numberFormat: {
        decimal: '.',
        thousands: ','
      },
      colors: {
        primary: '#007bff',
        success: '#28a745',
        warning: '#ffc107',
        danger: '#dc3545'
      },
      gamePreferences: {
        chessNotation: 'algebraic',
        checkersVariant: 'american'
      }
    });

    this.culturalSettings.set('ru', {
      dateFormat: 'DD.MM.YYYY',
      timeFormat: '24h',
      weekStart: 'monday',
      currency: 'RUB',
      numberFormat: {
        decimal: ',',
        thousands: ' '
      },
      colors: {
        primary: '#0056b3',
        success: '#28a745',
        warning: '#fd7e14',
        danger: '#dc3545'
      },
      gamePreferences: {
        chessNotation: 'algebraic',
        checkersVariant: 'russian'
      }
    });
  }

  getCulturalSettings(language: string): CulturalSettings {
    return this.culturalSettings.get(language) || this.culturalSettings.get('en')!;
  }

  adaptGameInterface(gameType: string, language: string): GameInterfaceConfig {
    const cultural = this.getCulturalSettings(language);
    
    const baseConfig: GameInterfaceConfig = {
      boardOrientation: 'white',
      pieceStyle: 'classic',
      coordinateNotation: true,
      moveHighlighting: true,
      soundEffects: true
    };

    // Cultural adaptations
    switch (gameType) {
      case 'chess':
        return {
          ...baseConfig,
          notation: cultural.gamePreferences.chessNotation,
          pieceNames: this.getLocalizedPieceNames('chess', language)
        };
      
      case 'checkers':
        return {
          ...baseConfig,
          variant: cultural.gamePreferences.checkersVariant,
          pieceNames: this.getLocalizedPieceNames('checkers', language)
        };
      
      default:
        return baseConfig;
    }
  }

  private getLocalizedPieceNames(gameType: string, language: string): Record<string, string> {
    // This would typically come from translation resources
    const pieceNames = {
      chess: {
        en: {
          king: 'King', queen: 'Queen', rook: 'Rook',
          bishop: 'Bishop', knight: 'Knight', pawn: 'Pawn'
        },
        ru: {
          king: 'Король', queen: 'Ферзь', rook: 'Ладья',
          bishop: 'Слон', knight: 'Конь', pawn: 'Пешка'
        }
      },
      checkers: {
        en: { man: 'Man', king: 'King' },
        ru: { man: 'Шашка', king: 'Дамка' }
      }
    };

    return pieceNames[gameType]?.[language] || pieceNames[gameType]?.['en'] || {};
  }

  formatCurrency(amount: number, language: string): string {
    const cultural = this.getCulturalSettings(language);
    const config = SUPPORTED_LANGUAGES.find(l => l.code === language);
    
    if (!config) return amount.toString();

    const formatted = new Intl.NumberFormat(config.locale, {
      style: 'currency',
      currency: cultural.currency
    }).format(amount);

    return formatted;
  }

  formatDateTime(date:

  formatDateTime(date: Date, language: string, type: 'date' | 'time' | 'datetime' = 'datetime'): string {
    const cultural = this.getCulturalSettings(language);
    const config = SUPPORTED_LANGUAGES.find(l => l.code === language);
    
    if (!config) return date.toString();

    const options: Intl.DateTimeFormatOptions = {};
    
    switch (type) {
      case 'date':
        options.year = 'numeric';
        options.month = '2-digit';
        options.day = '2-digit';
        break;
      case 'time':
        options.hour = '2-digit';
        options.minute = '2-digit';
        options.hour12 = cultural.timeFormat === '12h';
        break;
      case 'datetime':
        options.year = 'numeric';
        options.month = '2-digit';
        options.day = '2-digit';
        options.hour = '2-digit';
        options.minute = '2-digit';
        options.hour12 = cultural.timeFormat === '12h';
        break;
    }

    return new Intl.DateTimeFormat(config.locale, options).format(date);
  }
}
```

## 8. Translation Management System

### Translation Management API
```typescript
class TranslationManagementService {
  constructor(
    private translationRepository: TranslationRepository,
    private cacheService: CacheService,
    private auditService: AuditService
  ) {}

  async createTranslationResource(data: CreateTranslationResourceRequest): Promise<TranslationResource> {
    // Validate language support
    if (!this.isLanguageSupported(data.language)) {
      throw new ValidationError(`Language ${data.language} is not supported`);
    }

    // Check if resource already exists
    const existing = await this.translationRepository.findByLanguageAndNamespace(
      data.language,
      data.namespace
    );

    if (existing) {
      throw new ConflictError('Translation resource already exists');
    }

    // Create new resource
    const resource: TranslationResource = {
      namespace: data.namespace,
      language: data.language,
      version: '1.0.0',
      translations: data.translations,
      metadata: {
        lastUpdated: new Date(),
        translator: data.translatorId,
        reviewStatus: 'pending',
        completeness: this.calculateCompleteness(data.translations, data.namespace)
      }
    };

    const created = await this.translationRepository.create(resource);
    
    // Invalidate cache
    await this.invalidateTranslationCache(data.language, data.namespace);
    
    // Log audit trail
    await this.auditService.logTranslationChange('create', created, data.translatorId);

    return created;
  }

  async updateTranslations(
    language: string,
    namespace: string,
    updates: Record<string, any>,
    translatorId: string
  ): Promise<TranslationResource> {
    const resource = await this.translationRepository.findByLanguageAndNamespace(language, namespace);
    
    if (!resource) {
      throw new NotFoundError('Translation resource not found');
    }

    // Merge updates
    const updatedTranslations = this.mergeTranslations(resource.translations, updates);
    
    // Update resource
    resource.translations = updatedTranslations;
    resource.version = this.incrementVersion(resource.version);
    resource.metadata.lastUpdated = new Date();
    resource.metadata.translator = translatorId;
    resource.metadata.reviewStatus = 'pending';
    resource.metadata.completeness = this.calculateCompleteness(updatedTranslations, namespace);

    const updated = await this.translationRepository.update(resource);
    
    // Invalidate cache
    await this.invalidateTranslationCache(language, namespace);
    
    // Log audit trail
    await this.auditService.logTranslationChange('update', updated, translatorId);

    return updated;
  }

  async reviewTranslations(
    language: string,
    namespace: string,
    reviewerId: string,
    status: 'approved' | 'rejected',
    comments?: string
  ): Promise<void> {
    const resource = await this.translationRepository.findByLanguageAndNamespace(language, namespace);
    
    if (!resource) {
      throw new NotFoundError('Translation resource not found');
    }

    resource.metadata.reviewStatus = status;
    resource.metadata.lastUpdated = new Date();

    await this.translationRepository.update(resource);
    
    // Log review
    await this.auditService.logTranslationReview(resource, reviewerId, status, comments);
    
    // Notify translator
    await this.notifyTranslationReview(resource.metadata.translator!, status, comments);
  }

  async getTranslationProgress(): Promise<TranslationProgress[]> {
    const resources = await this.translationRepository.findAll();
    const progressMap = new Map<string, TranslationProgress>();

    for (const resource of resources) {
      const key = resource.namespace;
      
      if (!progressMap.has(key)) {
        progressMap.set(key, {
          namespace: key,
          languages: [],
          overallCompleteness: 0
        });
      }

      const progress = progressMap.get(key)!;
      progress.languages.push({
        language: resource.language,
        completeness: resource.metadata.completeness,
        status: resource.metadata.reviewStatus,
        lastUpdated: resource.metadata.lastUpdated
      });
    }

    // Calculate overall completeness
    for (const progress of progressMap.values()) {
      const totalCompleteness = progress.languages.reduce((sum, lang) => sum + lang.completeness, 0);
      progress.overallCompleteness = totalCompleteness / progress.languages.length;
    }

    return Array.from(progressMap.values());
  }

  private calculateCompleteness(translations: Record<string, any>, namespace: string): number {
    // Get reference translation (English) to compare against
    const referenceTranslations = this.getReferenceTranslations(namespace);
    
    if (!referenceTranslations) {
      return 100; // If no reference, consider complete
    }

    const totalKeys = this.countTranslationKeys(referenceTranslations);
    const translatedKeys = this.countTranslationKeys(translations);
    
    return Math.min(100, (translatedKeys / totalKeys) * 100);
  }

  private countTranslationKeys(obj: Record<string, any>): number {
    let count = 0;
    
    for (const value of Object.values(obj)) {
      if (typeof value === 'string') {
        count++;
      } else if (typeof value === 'object' && value !== null) {
        count += this.countTranslationKeys(value);
      }
    }
    
    return count;
  }

  private mergeTranslations(existing: Record<string, any>, updates: Record<string, any>): Record<string, any> {
    const result = { ...existing };
    
    for (const [key, value] of Object.entries(updates)) {
      if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
        result[key] = this.mergeTranslations(result[key] || {}, value);
      } else {
        result[key] = value;
      }
    }
    
    return result;
  }

  private async invalidateTranslationCache(language: string, namespace: string): Promise<void> {
    const cacheKey = `translations:${language}:${namespace}`;
    await this.cacheService.delete(cacheKey);
  }
}
```

## 9. Localization Testing and Quality Assurance

### Localization Testing Framework
```typescript
class LocalizationTestingFramework {
  private testSuites: Map<string, LocalizationTestSuite> = new Map();

  constructor(
    private translationService: ServerTranslationService,
    private contentManager: LocalizedContentManager
  ) {
    this.initializeTestSuites();
  }

  private initializeTestSuites(): void {
    // Translation completeness tests
    this.testSuites.set('completeness', {
      name: 'Translation Completeness',
      tests: [
        this.testTranslationCompleteness.bind(this),
        this.testMissingKeys.bind(this),
        this.testEmptyTranslations.bind(this)
      ]
    });

    // Format validation tests
    this.testSuites.set('format', {
      name: 'Format Validation',
      tests: [
        this.testPlaceholderConsistency.bind(this),
        this.testHtmlTagConsistency.bind(this),
        this.testNumberFormatting.bind(this),
        this.testDateFormatting.bind(this)
      ]
    });

    // Cultural adaptation tests
    this.testSuites.set('cultural', {
      name: 'Cultural Adaptation',
      tests: [
        this.testCurrencyFormatting.bind(this),
        this.testDateTimeFormatting.bind(this),
        this.testTextDirection.bind(this),
        this.testColorSchemes.bind(this)
      ]
    });

    // Content quality tests
    this.testSuites.set('quality', {
      name: 'Content Quality',
      tests: [
        this.testTextLength.bind(this),
        this.testSpecialCharacters.bind(this),
        this.testContextualAccuracy.bind(this),
        this.testTerminologyConsistency.bind(this)
      ]
    });
  }

  async runAllTests(language: string): Promise<LocalizationTestReport> {
    const report: LocalizationTestReport = {
      language,
      timestamp: new Date(),
      overallScore: 0,
      suiteResults: [],
      issues: [],
      recommendations: []
    };

    let totalScore = 0;
    let totalTests = 0;

    for (const [suiteId, suite] of this.testSuites) {
      const suiteResult = await this.runTestSuite(suite, language);
      report.suiteResults.push(suiteResult);
      
      totalScore += suiteResult.score * suiteResult.tests.length;
      totalTests += suiteResult.tests.length;
      
      // Collect issues
      report.issues.push(...suiteResult.issues);
    }

    report.overallScore = totalTests > 0 ? totalScore / totalTests : 0;
    report.recommendations = this.generateRecommendations(report);

    return report;
  }

  private async runTestSuite(suite: LocalizationTestSuite, language: string): Promise<TestSuiteResult> {
    const result: TestSuiteResult = {
      suiteName: suite.name,
      score: 0,
      tests: [],
      issues: []
    };

    for (const test of suite.tests) {
      try {
        const testResult = await test(language);
        result.tests.push(testResult);
        
        if (!testResult.passed) {
          result.issues.push(...testResult.issues);
        }
      } catch (error) {
        result.tests.push({
          name: test.name,
          passed: false,
          score: 0,
          issues: [{
            type: 'error',
            severity: 'high',
            message: `Test failed with error: ${error.message}`,
            location: 'test_framework'
          }]
        });
      }
    }

    // Calculate suite score
    const totalScore = result.tests.reduce((sum, test) => sum + test.score, 0);
    result.score = result.tests.length > 0 ? totalScore / result.tests.length : 0;

    return result;
  }

  private async testTranslationCompleteness(language: string): Promise<TestResult> {
    const issues: LocalizationIssue[] = [];
    const namespaces = ['common', 'auth', 'games', 'tournaments', 'profile'];
    
    let totalKeys = 0;
    let translatedKeys = 0;

    for (const namespace of namespaces) {
      const referenceKeys = await this.getTranslationKeys('en', namespace);
      const targetKeys = await this.getTranslationKeys(language, namespace);
      
      totalKeys += referenceKeys.length;
      translatedKeys += targetKeys.length;
      
      // Find missing keys
      const missingKeys = referenceKeys.filter(key => !targetKeys.includes(key));
      
      for (const key of missingKeys) {
        issues.push({
          type: 'missing_translation',
          severity: 'medium',
          message: `Missing translation for key: ${key}`,
          location: `${namespace}:${key}`
        });
      }
    }

    const completeness = totalKeys > 0 ? (translatedKeys / totalKeys) * 100 : 100;
    
    return {
      name: 'Translation Completeness',
      passed: completeness >= 95,
      score: completeness,
      issues
    };
  }

  private async testPlaceholderConsistency(language: string): Promise<TestResult> {
    const issues: LocalizationIssue[] = [];
    const namespaces = ['common', 'auth', 'games'];
    
    for (const namespace of namespaces) {
      const referenceTranslations = await this.getTranslations('en', namespace);
      const targetTranslations = await this.getTranslations(language, namespace);
      
      await this.checkPlaceholderConsistency(
        referenceTranslations,
        targetTranslations,
        namespace,
        issues
      );
    }

    return {
      name: 'Placeholder Consistency',
      passed: issues.length === 0,
      score: issues.length === 0 ? 100 : Math.max(0, 100 - issues.length * 10),
      issues
    };
  }

  private async checkPlaceholderConsistency(
    reference: Record<string, any>,
    target: Record<string, any>,
    namespace: string,
    issues: LocalizationIssue[],
    keyPath: string = ''
  ): Promise<void> {
    for (const [key, value] of Object.entries(reference)) {
      const currentPath = keyPath ? `${keyPath}.${key}` : key;
      
      if (typeof value === 'string') {
        const referencePlaceholders = this.extractPlaceholders(value);
        const targetValue = target[key];
        
        if (typeof targetValue === 'string') {
          const targetPlaceholders = this.extractPlaceholders(targetValue);
          
          // Check for missing placeholders
          for (const placeholder of referencePlaceholders) {
            if (!targetPlaceholders.includes(placeholder)) {
              issues.push({
                type: 'missing_placeholder',
                severity: 'high',
                message: `Missing placeholder ${placeholder} in translation`,
                location: `${namespace}:${currentPath}`
              });
            }
          }
          
          // Check for extra placeholders
          for (const placeholder of targetPlaceholders) {
            if (!referencePlaceholders.includes(placeholder)) {
              issues.push({
                type: 'extra_placeholder',
                severity: 'medium',
                message: `Extra placeholder ${placeholder} in translation`,
                location: `${namespace}:${currentPath}`
              });
            }
          }
        }
      } else if (typeof value === 'object' && value !== null) {
        if (target[key] && typeof target[key] === 'object') {
          await this.checkPlaceholderConsistency(
            value,
            target[key],
            namespace,
            issues,
            currentPath
          );
        }
      }
    }
  }

  private extractPlaceholders(text: string): string[] {
    const placeholderRegex = /\{\{(\w+)(?::(\w+))?\}\}/g;
    const placeholders: string[] = [];
    let match;
    
    while ((match = placeholderRegex.exec(text)) !== null) {
      placeholders.push(match[0]);
    }
    
    return placeholders;
  }

  private generateRecommendations(report: LocalizationTestReport): string[] {
    const recommendations: string[] = [];
    
    if (report.overallScore < 80) {
      recommendations.push('Overall localization quality is below acceptable threshold. Consider comprehensive review.');
    }
    
    const missingTranslations = report.issues.filter(issue => issue.type === 'missing_translation');
    if (missingTranslations.length > 0) {
      recommendations.push(`Complete ${missingTranslations.length} missing translations to improve coverage.`);
    }
    
    const placeholderIssues = report.issues.filter(issue => 
      issue.type === 'missing_placeholder' || issue.type === 'extra_placeholder'
    );
    if (placeholderIssues.length > 0) {
      recommendations.push('Review placeholder consistency to prevent runtime errors.');
    }
    
    const highSeverityIssues = report.issues.filter(issue => issue.severity === 'high');
    if (highSeverityIssues.length > 0) {
      recommendations.push('Address high-severity issues immediately to prevent user experience problems.');
    }
    
    return recommendations;
  }
}
```

## 10. Performance Optimization for i18n

### Translation Caching Strategy
```typescript
class TranslationCacheManager {
  private memoryCache: Map<string, any> = new Map();
  private cacheStats: CacheStats = {
    hits: 0,
    misses: 0,
    evictions: 0
  };

  constructor(
    private redisCache: RedisCache,
    private maxMemoryEntries: number = 1000
  ) {}

  async getTranslations(language: string, namespace: string): Promise<Record<string, any> | null> {
    const cacheKey = `${language}:${namespace}`;
    
    // Try memory cache first
    if (this.memoryCache.has(cacheKey)) {
      this.cacheStats.hits++;
      return this.memoryCache.get(cacheKey);
    }
    
    // Try Redis cache
    const cached = await this.redisCache.get(`translations:${cacheKey}`);
    if (cached) {
      this.cacheStats.hits++;
      
      // Store in memory cache
      this.setMemoryCache(cacheKey, cached);
      return cached;
    }
    
    this.cacheStats.misses++;
    return null;
  }

  async setTranslations(language: string, namespace: string, translations: Record<string, any>): Promise<void> {
    const cacheKey = `${language}:${namespace}`;
    
    // Store in Redis with 1 hour TTL
    await this.redisCache.setex(`translations:${cacheKey}`, 3600, translations);
    
    // Store in memory cache
    this.setMemoryCache(cacheKey, translations);
  }

  private setMemoryCache(key: string, value: any): void {
    // Implement LRU eviction
    if (this.memoryCache.size >= this.maxMemoryEntries) {
      const firstKey = this.memoryCache.keys().next().value;
      this.memoryCache.delete(firstKey);
      this.cacheStats.evictions++;
    }
    
    this.memoryCache.set(key, value);
  }

  async invalidateCache(language?: string, namespace?: string): Promise<void> {
    if (language && namespace) {
      // Invalidate specific translation
      const cacheKey = `${language}:${namespace}`;
      this.memoryCache.delete(cacheKey);
      await this.redisCache.del(`translations:${cacheKey}`);
    } else if (language) {
      // Invalidate all translations for language
      const pattern = `translations:${language}:*`;
      await this.redisCache.delPattern(pattern);
      
      // Clear from memory cache
      for (const key of this.memoryCache.keys()) {
        if (key.startsWith(`${language}:`)) {
          this.memoryCache.delete(key);
        }
      }
    } else {
      // Invalidate all translations
      await this.redisCache.delPattern('translations:*');
      this.memoryCache.clear();
    }
  }

  getCacheStats(): CacheStats {
    return {
      ...this.cacheStats,
      hitRate: this.cacheStats.hits / (this.cacheStats.hits + this.cacheStats.misses) * 100,
      memoryEntries: this.memoryCache.size
    };
  }
}
```

### Bundle Optimization
```typescript
class TranslationBundleOptimizer {
  async optimizeClientBundle(language: string, namespaces: string[]): Promise<OptimizedBundle> {
    const bundle: OptimizedBundle = {
      language,
      version: this.generateBundleVersion(),
      namespaces: {},
      metadata: {
        size: 0,
        compressionRatio: 0,
        generatedAt: new Date()
      }
    };

    for (const namespace of namespaces) {
      const translations = await this.getTranslations(language, namespace);
      
      if (translations) {
        // Remove unused translations for client
        const optimized = this.removeServerOnlyTranslations(translations);
        
        // Compress nested structures
        const compressed = this.compressTranslations(optimized);
        
        bundle.namespaces[namespace] = compressed;
      }
    }

    // Calculate bundle size
    const bundleString = JSON.stringify(bundle.namespaces);
    bundle.metadata.size = new Blob([bundleString]).size;
    
    // Apply compression
    const compressed = await this.compressBundle(bundleString);
    bundle.metadata.compressionRatio = compressed.length / bundleString.length;

    return bundle;
  }

  private removeServerOnlyTranslations(translations: Record<string, any>): Record<string, any> {
    const clientTranslations = { ...translations };
    
    // Remove email templates (server-only)
    delete clientTranslations.email;
    
    // Remove admin-only translations
    delete clientTranslations.admin;
    
    // Remove internal error messages
    if (clientTranslations.errors) {
      delete clientTranslations.errors.internal;
    }
    
    return clientTranslations;
  }

  private compressTranslations(translations: Record<string, any>): Record<string, any> {
    // Implement translation compression strategies
    // 1. Remove redundant whitespace
    // 2. Compress common phrases
    // 3. Use shorter keys for frequently used translations
    
    return this.recursiveCompress(translations);
  }

  private recursiveCompress(obj: Record<string, any>): Record<string, any> {
    const compressed: Record<string, any> = {};
    
    for (const [key, value] of Object.entries(obj)) {
      if (typeof value === 'string') {
        // Trim whitespace and compress
        compressed[key] = value.trim().replace(/\s+/g, ' ');
      } else if (typeof value === 'object' && value !== null) {
        compressed[key] = this.recursiveCompress(value);
      } else {
        compressed[key] = value;
      }
    }
    
    return compressed;
  }

  private async compressBundle(bundleString: string): Promise<string> {
    // Use gzip compression for bundle
    const compressed = await this.gzipCompress(bundleString);
    return compressed;
  }
}
```

This comprehensive internationalization architecture provides robust support for English and Russian languages with dynamic switching, cultural adaptation, and performance optimization across all platform components.