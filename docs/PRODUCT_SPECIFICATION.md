# Znakomstva — Product Specification

**Version:** 1.0  
**Date:** 2026-09-25  
**Status:** Approved — Ready for Mockup Phase

---

## Table of Contents

1. [Product Vision](#1-product-vision)
2. [Technology Stack](#2-technology-stack)
3. [Architectural Decisions](#3-architectural-decisions)
4. [Authentication System](#4-authentication-system)
5. [Feature Set](#5-feature-set)
6. [User Flows](#6-user-flows)
7. [Screen List](#7-screen-list)
8. [Navigation Architecture (Expo Router)](#8-navigation-architecture-expo-router)
9. [Data Model](#9-data-model)
10. [Matching Logic](#10-matching-logic)
11. [State Management](#11-state-management)
12. [Service Layer (Backend-Agnostic)](#12-service-layer-backend-agnostic)
13. [Design System](#13-design-system)
14. [Mockup Workflow](#14-mockup-workflow)
15. [Figma Workflow](#15-figma-workflow)
16. [Testing Strategy](#16-testing-strategy)
17. [Security and Privacy](#17-security-and-privacy)
18. [Platform Differences](#18-platform-differences)
19. [MVP Roadmap](#19-mvp-roadmap)
20. [Project Structure](#20-project-structure)
21. [Claude Code Workflow](#21-claude-code-workflow)
22. [Publication Checklist](#22-publication-checklist)

---

## 1. Product Vision

**Znakomstva** — мобильное приложение для знакомств для русскоязычной аудитории.

**Концепция:** минималистичный, быстрый MVP с акцентом на качество совпадений. Swipe-механика как основа Discover. Реальный чат открывается только после взаимного лайка (Match).

**Целевая аудитория:** русскоязычные пользователи 18+, ищущие реальные знакомства.

**Приоритет MVP:** рабочий полный цикл `регистрация → профиль → свайп → матч → чат`, без платных функций и сложных алгоритмов.

**Визуальный стиль:** Modern Dark Romance — тёмный фон, тёплые акценты, премиальное ощущение.

---

## 2. Technology Stack

| Слой | Технология |
|---|---|
| Mobile Framework | React Native (через Expo SDK) |
| Language | TypeScript (strict mode) |
| Build / Dev | Expo (managed workflow) |
| Navigation | **Expo Router v4** (file-based routing) |
| State Management | **Zustand** |
| Backend | **Firebase** (Auth + Firestore + Storage) |
| Service Layer | Backend-agnostic interfaces (реализация Firebase) |
| Platforms | Android + iOS (одна кодовая база) + Web preview |
| Auth methods | Email/Password + Google Sign-In + Sign in with Apple |

---

## 3. Architectural Decisions

Все решения зафиксированы и не подлежат изменению без обсуждения.

### 3.1 Firebase как MVP Backend

- **Firebase Auth** — управление аутентификацией (email/password, Google, Apple)
- **Firestore** — NoSQL база данных (профили, матчи, сообщения)
- **Firebase Storage** — хранение фотографий пользователей
- Бесплатный tier (Spark) достаточен для MVP
- При необходимости может быть заменён (Supabase, кастомный API) через service layer

### 3.2 Backend-Agnostic AuthService

AuthService — это интерфейс. Firebase — одна из реализаций под этим интерфейсом.  
UI-экраны никогда не импортируют Firebase напрямую.  
Все вызовы идут через `services/auth/AuthService`.

```
UI Screen
  → useAuth() hook
    → AuthStore (Zustand)
      → AuthService interface
        → FirebaseAuthService (implementation)
          → Firebase SDK
```

### 3.3 Gender + Dating Preference — два отдельных поля

```
Profile.gender:     enum [MALE, FEMALE, NON_BINARY]
Profile.lookingFor: enum [MEN, WOMEN, EVERYONE]
```

- Это **два независимых поля**, не связанных логикой автовыбора
- Архитектура расширяема: в V1 можно добавить `[TRANSGENDER_MAN, TRANSGENDER_WOMAN, GENDERQUEER, ...]` без изменения основной модели
- Matching использует **только** `lookingFor` — никакого автоматического вывода из `gender`

### 3.4 Expo Router v4 как Navigation System

- File-based routing (как Next.js)
- Автоматические deep links из структуры файлов
- Navigation logic изолирована в `app/` — не смешивается с бизнес-логикой
- Отдельные route groups: `(auth)`, `(onboarding)`, `(main)`
- Auth gate в Root Layout `app/_layout.tsx`

---

## 4. Authentication System

### 4.1 Auth States

Приложение всегда находится в одном из следующих auth-состояний:

| State | Описание | Маршрут |
|---|---|---|
| `loading` | Проверка сохранённой сессии при запуске | SplashScreen |
| `unauthenticated` | Нет активной сессии | `(auth)/welcome` |
| `authenticated_incomplete` | Сессия есть, профиль не заполнен | `(onboarding)/name` |
| `authenticated_unverified` | Сессия есть, email не подтверждён | `(auth)/verify-email` |
| `authenticated` | Полная сессия, профиль заполнен | `(main)/discover` |

Переходы между состояниями управляются в `app/_layout.tsx` через `useAuthStore`.

### 4.2 Регистрация (Email + Password)

**Flow:**
```
RegisterScreen
  → Ввод: имя, email, пароль, подтверждение пароля
  → Валидация на клиенте
  → AuthService.registerWithEmail(name, email, password)
    → Firebase: createUserWithEmailAndPassword
    → Firebase: sendEmailVerification
    → Создание пустого Profile документа в Firestore
  → Переход: VerifyEmailScreen
```

**Поля формы:**
- `name`: string, min 2, max 50, обязательное
- `email`: valid email format, обязательное
- `password`: min 8 символов, хотя бы 1 цифра или спецсимвол, обязательное
- `confirmPassword`: должен совпадать с `password`

**Обработка ошибок:**
| Firebase error code | Сообщение пользователю |
|---|---|
| `auth/email-already-in-use` | "Этот email уже зарегистрирован. Войдите или восстановите пароль." |
| `auth/invalid-email` | "Неверный формат email." |
| `auth/weak-password` | "Пароль слишком простой. Минимум 8 символов." |
| Network error | "Нет соединения. Проверьте интернет." |

### 4.3 Вход (Email + Password)

**Flow:**
```
LoginScreen
  → Ввод: email, пароль
  → AuthService.loginWithEmail(email, password)
    → Firebase: signInWithEmailAndPassword
  → Проверка email verified:
      [не верифицирован] → VerifyEmailScreen
      [верифицирован, профиль не заполнен] → Onboarding
      [верифицирован, профиль заполнен] → Discover
```

**Обработка ошибок:**
| Firebase error code | Сообщение пользователю |
|---|---|
| `auth/user-not-found` | "Аккаунт не найден. Проверьте email или зарегистрируйтесь." |
| `auth/wrong-password` | "Неверный пароль." |
| `auth/too-many-requests` | "Слишком много попыток. Попробуйте позже или восстановите пароль." |
| `auth/user-disabled` | "Аккаунт заблокирован. Обратитесь в поддержку." |

### 4.4 Email Verification

**Flow:**
```
VerifyEmailScreen
  → Показывает: "Письмо отправлено на {email}"
  → Кнопка "Открыть почту" → системный intent
  → Кнопка "Отправить повторно" (активна через 60 сек)
  → Кнопка "Проверить" → Firebase: currentUser.reload() → check emailVerified
  → Автоматическая проверка каждые 5 сек пока экран открыт
  → После подтверждения → Onboarding
```

**Edge cases:**
- Письмо не пришло — повторная отправка (throttle 60 сек)
- Пользователь закрыл приложение и вернулся — проверка state при открытии
- Email устарел — возможность изменить email до верификации (V1)

### 4.5 Восстановление пароля

**Flow:**
```
ForgotPasswordScreen
  → Ввод email
  → AuthService.sendPasswordReset(email)
    → Firebase: sendPasswordResetEmail
  → Успех: показать "Письмо отправлено на {email}"
  → Кнопка "Вернуться к входу" → LoginScreen
```

**Обработка ошибок:**
| Ситуация | Поведение |
|---|---|
| `auth/user-not-found` | Показать успех (не раскрывать, зарегистрирован ли email — безопасность) |
| Network error | "Нет соединения." |

### 4.6 Смена пароля

**Flow:**
```
ChangePasswordScreen (в Settings)
  → Ввод: текущий пароль, новый пароль, подтверждение
  → AuthService.changePassword(currentPassword, newPassword)
    → Firebase: reauthenticateWithCredential → updatePassword
  → Успех → уведомление → назад в Settings
```

**Требования:**
- Reauthentication обязательна (Firebase требует recent login для смены пароля)
- Если сессия устарела — предложить войти заново

### 4.7 Google Sign-In

**Flow:**
```
WelcomeScreen / LoginScreen → Кнопка "Войти с Google"
  → AuthService.signInWithGoogle()
    → Expo AuthSession / @react-native-google-signin/google-signin
    → Firebase: signInWithCredential(GoogleAuthProvider.credential(idToken))
  → Если новый пользователь → создать Profile → Onboarding
  → Если существующий → auth state check → Discover или Onboarding
```

**Linking к существующему email-аккаунту:**
```
Пользователь входит через Google, но email уже существует как email/password аккаунт:
  → Firebase выдаёт auth/account-exists-with-different-credential
  → Предложить: "Этот email уже зарегистрирован через пароль.
                 Войдите через email/пароль и привяжите Google в настройках."
  → В AccountSettingsScreen → "Привязанные аккаунты" → Link Google
    → AuthService.linkWithGoogle()
      → Firebase: linkWithCredential
```

### 4.8 Sign in with Apple

**Flow:**
```
WelcomeScreen → Кнопка "Войти через Apple" (только iOS)
  → expo-apple-authentication: AppleAuthentication.signInAsync
  → Firebase: signInWithCredential(OAuthProvider.credential('apple.com'))
  → Если новый пользователь → создать Profile → Onboarding
  → Если существующий → auth state check
```

**Особенности:**
- Apple Sign-In обязателен для App Store если есть другие OAuth методы
- Apple скрывает реальный email (relay email) — хранить Apple User ID, не полагаться на email
- Linking к email-аккаунту — аналогично Google (см. 4.7)
- Показывается только на iOS

### 4.9 Linking OAuth к существующему аккаунту

В AccountSettingsScreen → раздел "Привязанные аккаунты":

| Метод | Состояние | Действие |
|---|---|---|
| Email/Password | всегда привязан | — |
| Google | привязан / не привязан | Привязать / Отвязать |
| Apple | привязан / не привязан (iOS only) | Привязать / Отвязать |

**Ограничение:** нельзя отвязать метод, если он единственный (пользователь потеряет доступ).

### 4.10 Logout

**Flow:**
```
Settings → AccountSettingsScreen → "Выйти из аккаунта"
  → Confirm Dialog: "Вы уверены, что хотите выйти?"
  → AuthService.signOut()
    → Firebase: signOut()
    → Очистить AuthStore (user = null)
    → Очистить все Zustand stores
    → Очистить Secure Store (токены)
  → Redirect → (auth)/welcome
```

### 4.11 Удаление аккаунта

**Flow:**
```
AccountSettingsScreen → "Удалить аккаунт"
  → ConfirmDialog 1: "Ваш профиль, фотографии и все совпадения будут удалены. 
                       Это действие необратимо."
  → [Если email/password] ConfirmDialog 2: повторный ввод пароля (reauthentication)
  → [Если OAuth] ConfirmDialog 2: повторная аутентификация через Google/Apple
  → AuthService.deleteAccount()
    → Firebase: reauthenticateWithCredential
    → Firestore: soft delete (isDeleted: true, deletedAt: now)
    → Запустить cloud function для hard delete через 30 дней
    → Firebase Storage: пометить фото на удаление
    → Firebase Auth: deleteUser()
    → Очистить все локальные данные
  → Redirect → (auth)/welcome
```

**Состояния:**
- `isDeleted: true` — профиль скрыт, аккаунт недоступен немедленно
- Данные хранятся 30 дней (юридическая защита)
- После 30 дней: hard delete всех данных из Firestore + Storage

### 4.12 Восстановление доступа (Edge Cases)

| Сценарий | Решение |
|---|---|
| Нет доступа к email для верификации | Смена email до верификации (V1) |
| Утеряны Google/Apple credentials | Войти через email/пароль (если был привязан) |
| Аккаунт временно заблокирован (много попыток) | Firebase auto-unblock через 1 час, информировать пользователя |
| Сессия истекла | Firebase auto-refresh через refresh token (30 дней), при истечении → LoginScreen |

---

## 5. Feature Set

### MVP (обязательно для первой рабочей версии)

| # | Функция | Назначение | Данные | Связи |
|---|---|---|---|---|
| 1 | Splash / Launch | Загрузка + auth check | — | → auth state routing |
| 2 | Onboarding slides | Знакомство с приложением | — | → Registration |
| 3 | Email/Password регистрация | Основной способ входа | email, password, name | AuthService |
| 4 | Email/Password вход | Вернуть существующего | email, password | AuthService |
| 5 | Google Sign-In | Быстрый вход Android | OAuth token | AuthService |
| 6 | Apple Sign-In | Обязательно iOS | OAuth token | AuthService |
| 7 | Email verification | Подтверждение почты | — | AuthService |
| 8 | Password reset | Восстановление доступа | email | AuthService |
| 9 | Profile setup flow | Создание анкеты | name, age, gender, photos, bio, city | Profile, Photo |
| 10 | Photo upload (1–6) | Главный элемент анкеты | Photo[] | MediaService |
| 11 | Preferences setup | Параметры поиска | ageRange, distance, lookingFor | Preference |
| 12 | Discover (swipe) | Основной product loop | Profile[] | DiscoveryService |
| 13 | Like | Выразить интерес | Like | MatchService |
| 14 | Pass | Пропустить анкету | Pass | DiscoveryService |
| 15 | Match | Взаимный лайк | Match | MatchService |
| 16 | Match notification | Сигнал о совпадении | Match | — |
| 17 | Matches list | Все совпадения | Match[] | → Chat |
| 18 | Conversations list | Все диалоги | Conversation[] | → Chat |
| 19 | Chat (1-on-1) | Общение после матча | Message[] | ChatService |
| 20 | My Profile view | Просмотр своего профиля | Profile | ProfileService |
| 21 | Edit Profile | Обновление анкеты | Profile | ProfileService |
| 22 | Search Preferences | Изменение фильтров | Preference | ProfileService |
| 23 | Block user | Безопасность | Block | SafetyService |
| 24 | Report user | Безопасность | Report | SafetyService |
| 25 | Logout | Завершение сессии | — | AuthService |
| 26 | Delete account | GDPR, пользовательское право | — | AuthService |
| 27 | Change password | Управление аккаунтом | — | AuthService |
| 28 | Link OAuth account | Привязка Google/Apple | — | AuthService |

### V1 (после MVP)

| Функция | Обоснование |
|---|---|
| Super Like | Усиленный сигнал интереса |
| Boost (временное повышение) | Первая монетизация |
| Просмотр кто лайкнул (Premium) | Монетизация |
| Push-уведомления | Retention |
| Undo last swipe | QoL |
| Typing indicator | UX чата |
| Read receipts | UX чата |
| Фильтр по интересам / тегам | Качество matching |
| Phone number auth | Альтернативный вход |
| Photo verification | Безопасность |

### Future

| Функция |
|---|
| Video профиль |
| Голосовые / видеозвонки |
| AI icebreaker сообщения |
| Групповые события / Activities |
| Web-версия |
| Subscription / IAP |
| Верификация личности |
| Расширенные гендерные идентичности |

---

## 6. User Flows

### 6.1 Основной путь — новый пользователь

```
Splash (auth check)
  → [нет сессии] Welcome
    → [Email] Register → VerifyEmail → Profile Setup Flow → Discover
    → [Google] Google OAuth → Profile Setup Flow → Discover
    → [Apple] Apple OAuth → Profile Setup Flow → Discover
```

### 6.2 Основной путь — возвращающийся пользователь

```
Splash (auth check)
  → [сессия есть, профиль полный, email верифицирован] → Discover
  → [сессия есть, email не верифицирован] → VerifyEmail
  → [сессия есть, профиль не заполнен] → Onboarding (с шага, где остановились)
  → [сессия протухла] → Login
```

### 6.3 Profile Setup Flow (только первый раз)

```
SetupName → SetupAge → SetupGender → SetupLookingFor
  → SetupPhotos (min 1 фото обязательно) → SetupBio (можно skip)
  → SetupLocation → SetupPreferences → Discover
```

**Прогресс сохраняется.** Если пользователь закрыл приложение на шаге 4 — при следующем запуске вернётся на шаг 4.

### 6.4 Discover loop

```
Discover
  ├─→ Swipe Right / Tap Like button
  │     ├─→ [нет матча] → следующая карточка
  │     └─→ [матч!] → MatchModal (overlay)
  │               ├─→ "Написать" → Chat
  │               └─→ "Продолжить свайпать" → Discover
  │
  ├─→ Swipe Left / Tap Pass button → следующая карточка
  ├─→ Tap на фото/карточку → ProfileDetail
  │     └─→ Like / Pass / Report → Discover
  ├─→ Tap Filter → FilterScreen → Apply → Discover (перезагрузить стек)
  └─→ Колода пуста → DiscoverEmpty ("Вернитесь позже" / расширить поиск)
```

### 6.5 Match → Chat flow

```
Match Modal
  → "Написать сообщение" → Chat Screen
Matches Tab
  → Tap на аватар матча → Chat Screen
Messages Tab
  → Tap на диалог → Chat Screen
```

### 6.6 Альтернативные / edge case flows

| Сценарий | Начало | Действие | Результат |
|---|---|---|---|
| Нет фото при создании профиля | SetupPhotos | Попытка перейти без фото | Ошибка "Добавьте минимум 1 фото" — обязательно |
| Нет подходящих анкет | Discover | Стек пуст | DiscoverEmpty → "Расширить поиск" |
| Получен Match | Discover | Автоматически | MatchModal поверх Discover |
| Unmatch | Chat / Matches | Меню → "Убрать из совпадений" | Confirm → Match удалён, чат скрыт |
| Block пользователя | Chat / ProfileDetail | Меню → "Заблокировать" | Confirm → Block → анкета исчезает везде |
| Report пользователя | Chat / ProfileDetail | Меню → "Пожаловаться" | ReportScreen → Submit → уведомление |
| Logout | Settings | AccountSettings → "Выйти" | Confirm → очистка → Welcome |
| Удаление аккаунта | Settings | AccountSettings → "Удалить" | Двойное подтверждение → soft delete → Welcome |
| Нет интернета | Любой | — | NoInternetScreen / offline banner |
| Истекшая сессия | Любой | Попытка API call | Auto-refresh token или → Login |

---

## 7. Screen List

### A. Launch / Onboarding Slides

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| A1 | SplashScreen | `app/_layout` (implicit) | Auth check, загрузка | ✅ |
| A2 | OnboardingScreen | `(auth)/onboarding` | 3 слайда с концепцией | ✅ |

### B. Authentication

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| B1 | WelcomeScreen | `(auth)/welcome` | Выбор способа входа | ✅ |
| B2 | RegisterScreen | `(auth)/register` | Email/Password регистрация | ✅ |
| B3 | LoginScreen | `(auth)/login` | Вход по email/паролю | ✅ |
| B4 | ForgotPasswordScreen | `(auth)/forgot-password` | Восстановление пароля | ✅ |
| B5 | VerifyEmailScreen | `(auth)/verify-email` | Подтверждение email | ✅ |

### C. Profile Setup (одноразовый onboarding flow)

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| C1 | SetupNameScreen | `(onboarding)/name` | Имя пользователя | ✅ |
| C2 | SetupAgeScreen | `(onboarding)/age` | Дата рождения | ✅ |
| C3 | SetupGenderScreen | `(onboarding)/gender` | Гендерная идентичность | ✅ |
| C4 | SetupLookingForScreen | `(onboarding)/looking-for` | Dating Preference | ✅ |
| C5 | SetupPhotosScreen | `(onboarding)/photos` | Загрузка фото (min 1) | ✅ |
| C6 | SetupBioScreen | `(onboarding)/bio` | Описание (опционально) | ✅ |
| C7 | SetupLocationScreen | `(onboarding)/location` | Город / геолокация | ✅ |
| C8 | SetupPreferencesScreen | `(onboarding)/preferences` | Параметры поиска | ✅ |

### D. Discover

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| D1 | DiscoverScreen | `(main)/discover/index` | Основной swipe | ✅ |
| D2 | ProfileDetailScreen | `(main)/discover/[userId]` | Полная анкета | ✅ |
| D3 | DiscoverEmptyScreen | Компонент внутри D1 | Нет анкет | ✅ |
| D4 | FilterScreen | `(main)/discover/filter` | Фильтры поиска | ✅ |

### E. Matches

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| E1 | MatchModal | `app/modals/match` | Праздник совпадения | ✅ |
| E2 | MatchesScreen | `(main)/matches/index` | Список совпадений | ✅ |

### F. Messages

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| F1 | ConversationsScreen | `(main)/messages/index` | Список диалогов | ✅ |

### G. Chat

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| G1 | ChatScreen | `(main)/messages/[matchId]` | 1-on-1 чат | ✅ |
| G2 | ChatMenuModal | `app/modals/chat-menu` | Опции чата (Report, Block, Unmatch) | ✅ |

### H. Profile

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| H1 | MyProfileScreen | `(main)/profile/index` | Просмотр своего профиля | ✅ |
| H2 | EditProfileScreen | `(main)/profile/edit` | Редактирование анкеты | ✅ |
| H3 | EditPhotosScreen | `(main)/profile/photos` | Управление фото | ✅ |

### I. Settings

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| I1 | SettingsScreen | `(main)/profile/settings/index` | Главное меню настроек | ✅ |
| I2 | SearchPreferencesScreen | `(main)/profile/settings/preferences` | Параметры поиска | ✅ |
| I3 | PrivacySettingsScreen | `(main)/profile/settings/privacy` | Приватность | ✅ |
| I4 | AccountSettingsScreen | `(main)/profile/settings/account` | Аккаунт, logout, delete | ✅ |
| I5 | ChangePasswordScreen | `(main)/profile/settings/change-password` | Смена пароля | ✅ |
| I6 | BlockedUsersScreen | `(main)/profile/settings/blocked-users` | Список заблокированных | ✅ |

### J. Safety

| # | Экран | Route | Назначение | MVP |
|---|---|---|---|---|
| J1 | ReportModal | `app/modals/report` | Форма жалобы | ✅ |

### K. System / Error States

| # | Экран | Назначение | MVP |
|---|---|---|---|
| K1 | ErrorScreen | Неожиданная ошибка / boundary | ✅ |
| K2 | NoInternetScreen | Нет соединения | ✅ |

**Итого: 38 экранов (MVP: 38)**

---

## 8. Navigation Architecture (Expo Router)

### 8.1 Структура файлов `app/`

```
app/
├── _layout.tsx                        ← Root Layout: auth gate, global providers
│                                         Логика: читает authStore → redirect
│
├── (auth)/                            ← Auth Route Group
│   ├── _layout.tsx                    ← Stack navigator для auth flow
│   ├── onboarding.tsx                 ← A2: Onboarding slides
│   ├── welcome.tsx                    ← B1: Выбор входа
│   ├── register.tsx                   ← B2: Регистрация
│   ├── login.tsx                      ← B3: Вход
│   ├── forgot-password.tsx            ← B4: Восстановление пароля
│   └── verify-email.tsx               ← B5: Подтверждение email
│
├── (onboarding)/                      ← Onboarding Route Group (первичная настройка)
│   ├── _layout.tsx                    ← Stack navigator с progress bar
│   ├── name.tsx                       ← C1
│   ├── age.tsx                        ← C2
│   ├── gender.tsx                     ← C3
│   ├── looking-for.tsx                ← C4
│   ├── photos.tsx                     ← C5
│   ├── bio.tsx                        ← C6
│   ├── location.tsx                   ← C7
│   └── preferences.tsx                ← C8
│
├── (main)/                            ← Main Route Group (авторизован + профиль готов)
│   ├── _layout.tsx                    ← Bottom Tabs definition
│   │                                     Tabs: Discover | Matches | Messages | Profile
│   │
│   ├── discover/
│   │   ├── _layout.tsx                ← Stack navigator
│   │   ├── index.tsx                  ← D1: DiscoverScreen (tab root)
│   │   ├── [userId].tsx               ← D2: ProfileDetailScreen
│   │   └── filter.tsx                 ← D4: FilterScreen
│   │
│   ├── matches/
│   │   ├── _layout.tsx                ← Stack navigator
│   │   └── index.tsx                  ← E2: MatchesScreen (tab root)
│   │
│   ├── messages/
│   │   ├── _layout.tsx                ← Stack navigator
│   │   ├── index.tsx                  ← F1: ConversationsScreen (tab root)
│   │   └── [matchId].tsx              ← G1: ChatScreen
│   │
│   └── profile/
│       ├── _layout.tsx                ← Stack navigator
│       ├── index.tsx                  ← H1: MyProfileScreen (tab root)
│       ├── edit.tsx                   ← H2: EditProfileScreen
│       ├── photos.tsx                 ← H3: EditPhotosScreen
│       └── settings/
│           ├── index.tsx              ← I1: SettingsScreen
│           ├── preferences.tsx        ← I2: SearchPreferences
│           ├── privacy.tsx            ← I3: PrivacySettings
│           ├── account.tsx            ← I4: AccountSettings
│           ├── change-password.tsx    ← I5: ChangePassword
│           └── blocked-users.tsx      ← I6: BlockedUsers
│
└── modals/                            ← Modal screens (поверх всего)
    ├── _layout.tsx                    ← Modal stack (presentation: modal)
    ├── match.tsx                      ← E1: MatchModal
    ├── chat-menu.tsx                  ← G2: ChatMenuModal
    └── report.tsx                     ← J1: ReportModal
```

### 8.2 Auth Gate Logic (app/_layout.tsx)

```
При запуске:
  1. Показать Splash
  2. Подписаться на Firebase.onAuthStateChanged
  3. Получить auth state → определить redirect:

  null (нет пользователя)         → redirect (auth)/welcome
  User (не верифицирован)         → redirect (auth)/verify-email
  User (верифицирован, нет profile) → redirect (onboarding)/name
  User (верифицирован, profile OK) → redirect (main)/discover

  При смене auth state → автоматический redirect
```

### 8.3 Navigation Rules

- **Нельзя** напрямую перейти в `(main)` без авторизации — защита в Root Layout
- **Нельзя** перейти в `(onboarding)` если профиль уже заполнен
- **ChatScreen** доступен из Matches и Messages — одинаковый компонент, разные стеки
- **MatchModal** — глобальный, показывается поверх любого экрана (из DiscoverStore: `newMatchId`)
- **Back navigation** в Onboarding — можно вернуться назад для редактирования
- **Back navigation** в Auth — всегда можно вернуться к Welcome

### 8.4 Deep Links

| URL | Экран |
|---|---|
| `znakomstva://match/:matchId` | MatchModal → Chat |
| `znakomstva://chat/:matchId` | ChatScreen |
| `znakomstva://profile/:userId` | ProfileDetailScreen |
| `znakomstva://verify-email` | VerifyEmailScreen (из письма) |
| `znakomstva://reset-password` | Обрабатывается Firebase (Custom URL scheme) |

Deep links заложены в архитектуру автоматически через Expo Router (файловая структура = URL структура).

### 8.5 Tab Bar

| Tab | Icon | Route | Badge |
|---|---|---|---|
| Discover | Flame | `(main)/discover` | — |
| Matches | Heart | `(main)/matches` | New matches count |
| Messages | Chat bubble | `(main)/messages` | Unread messages count |
| Profile | Person | `(main)/profile` | — |

---

## 9. Data Model

Все сущности хранятся в **Firestore**. Логический data model (не SQL schema).

### User

```
Collection: users/{userId}

id:             string (= Firebase Auth UID)
email:          string (unique)
createdAt:      Timestamp
lastActiveAt:   Timestamp
isVerified:     boolean (email verified)
isDeleted:      boolean (default: false)
deletedAt:      Timestamp | null
authProviders:  string[]  (["password", "google.com", "apple.com"])
```

### Profile

```
Collection: profiles/{userId}

userId:             string (= User.id, 1:1)
name:               string (max 50)
birthDate:          Timestamp
age:                number (computed, cached)
gender:             "MALE" | "FEMALE" | "NON_BINARY"
lookingFor:         "MEN" | "WOMEN" | "EVERYONE"
bio:                string | null (max 500)
cityName:           string | null
latitude:           number | null  (округляется до ~1km точности)
longitude:          number | null
isVisible:          boolean (default: true)
isProfileComplete:  boolean (all required fields filled)
onboardingStep:     string | null  (текущий шаг если не завершён)
updatedAt:          Timestamp
```

**Расширяемость gender:**
- В V1 можно расширить: `"TRANSGENDER_MAN" | "TRANSGENDER_WOMAN" | "NON_BINARY" | "GENDERQUEER" | "OTHER"`
- Поле `lookingFor` остаётся `"MEN" | "WOMEN" | "EVERYONE"` — matching логика не ломается

### Photo

```
Subcollection: profiles/{userId}/photos/{photoId}

id:         string
userId:     string
url:        string (Firebase Storage URL)
order:      number (0 = главное фото)
isMain:     boolean
createdAt:  Timestamp
```

Максимум 6 фотографий. Минимум 1 обязательно.

### Preference

```
Document: preferences/{userId}  (или поле в Profile)

userId:         string
ageMin:         number (default: 18)
ageMax:         number (default: 45)
maxDistance:    number (km, default: 50)
lookingFor:     "MEN" | "WOMEN" | "EVERYONE"  (дублирует Profile.lookingFor, синхронизируется)
```

### Like

```
Collection: likes/{likeId}

id:           string
fromUserId:   string
toUserId:     string
createdAt:    Timestamp
isSuperLike:  boolean (default: false)
```

Составной индекс: `(fromUserId, toUserId)` UNIQUE — предотвращает двойные лайки.

### Pass

```
Collection: passes/{passId}

id:           string
fromUserId:   string
toUserId:     string
createdAt:    Timestamp
expiresAt:    Timestamp | null (для будущего undo, V1)
```

### Match

```
Collection: matches/{matchId}

id:             string
user1Id:        string (меньший UID по алфавиту — для уникальности пары)
user2Id:        string (больший UID)
createdAt:      Timestamp
isUnmatched:    boolean (default: false)
unmatchedAt:    Timestamp | null
unmatchedBy:    string | null (userId)
```

Правило уникальности: одна пара — один документ. Это достигается через сортировку userId при создании.

### Conversation

```
Collection: conversations/{conversationId}

id:              string (= matchId для простоты)
matchId:         string (FK → Match)
participants:    string[]  ([user1Id, user2Id])
lastMessage:     string | null
lastMessageAt:   Timestamp | null
unreadCount:     map { userId: number }
```

### Message

```
Subcollection: conversations/{conversationId}/messages/{messageId}

id:               string
conversationId:   string
senderId:         string
text:             string (max 2000)
createdAt:        Timestamp
readAt:           Timestamp | null (null = непрочитано)
isDeleted:        boolean (default: false)
```

### Block

```
Collection: blocks/{blockId}

id:          string
blockerId:   string
blockedId:   string
createdAt:   Timestamp
```

Двусторонний эффект: `blockedId` тоже не видит `blockerId` в Discover.

### Report

```
Collection: reports/{reportId}

id:           string
reporterId:   string
reportedId:   string
reason:       "SPAM" | "FAKE" | "INAPPROPRIATE" | "HARASSMENT" | "OTHER"
comment:      string | null (max 500)
createdAt:    Timestamp
status:       "PENDING" | "REVIEWED" | "RESOLVED"
```

### Notification (V1)

```
Collection: notifications/{notificationId}

id:         string
userId:     string
type:       "MATCH" | "MESSAGE" | "LIKE" | "SYSTEM"
payload:    map
isRead:     boolean
createdAt:  Timestamp
```

---

## 10. Matching Logic

### 10.1 Правила показа анкет в Discover

Анкета пользователя B показывается пользователю A если:

```
1. B.isVisible = true
2. B.isDeleted = false
3. B.isProfileComplete = true
4. A не лайкнул B (нет записи в likes где fromUserId=A, toUserId=B)
5. A не пропустил B (нет записи в passes где fromUserId=A, toUserId=B)
6. A не заблокировал B И B не заблокировал A
7. B.age >= A.preferences.ageMin AND B.age <= A.preferences.ageMax
8. distance(A, B) <= A.preferences.maxDistance
9. Соответствие lookingFor:
     A.lookingFor == "MEN"    → показывать B с gender IN [MALE]
     A.lookingFor == "WOMEN"  → показывать B с gender IN [FEMALE]
     A.lookingFor == "EVERYONE" → показывать всех
   И взаимно:
     B.lookingFor должен включать A.gender (взаимная совместимость)
```

### 10.2 Взаимная совместимость lookingFor

| A.lookingFor | B.gender | B.lookingFor | Показывать? |
|---|---|---|---|
| MEN | MALE | WOMEN | ❌ (B ищет женщин, а не мужчин) |
| MEN | MALE | MEN | ✅ |
| MEN | MALE | EVERYONE | ✅ |
| WOMEN | FEMALE | MEN | ✅ |
| WOMEN | FEMALE | WOMEN | ✅ (если A тоже female и ищет women) |
| EVERYONE | любой | (включает A.gender) | ✅ |

**Правило:** NON_BINARY пользователи с `lookingFor = EVERYONE` совместимы со всеми, кто тоже ищет EVERYONE. Детальная логика для NON_BINARY уточняется в V1 при расширении опций.

### 10.3 Создание Match

```
Пользователь A лайкает B:
  1. Создать запись likes: {fromUserId: A, toUserId: B}
  2. Проверить: существует ли запись likes где fromUserId=B, toUserId=A?
     [да] → создать Match:
              matchId = sort([A.id, B.id]).join('_')
              Создать Conversation с тем же id
              Уведомить обоих пользователей
              Вернуть { isMatch: true, matchId }
     [нет] → Вернуть { isMatch: false }
```

### 10.4 Unmatch

```
Пользователь A делает unmatch с B:
  1. matches/{matchId}: isUnmatched = true, unmatchedAt = now, unmatchedBy = A.id
  2. Conversation скрывается для обоих (не удаляется, хранится 30 дней)
  3. Лайки остаются в likes — B снова не появится в Discover у A
```

---

## 11. State Management

Все глобальные stores реализованы через **Zustand**.

### AuthStore

```typescript
{
  user: FirebaseUser | null
  authState: 'loading' | 'unauthenticated' | 'authenticated_unverified'
           | 'authenticated_incomplete' | 'authenticated'
  error: string | null
  isLoading: boolean
  // actions
  initialize: () => void
  signOut: () => Promise<void>
  clearError: () => void
}
```

### ProfileStore

```typescript
{
  profile: Profile | null
  photos: Photo[]
  isSaving: boolean
  // actions
  loadProfile: () => Promise<void>
  updateProfile: (data: Partial<Profile>) => Promise<void>
  addPhoto: (file: File) => Promise<void>
  deletePhoto: (photoId: string) => Promise<void>
  reorderPhotos: (photoIds: string[]) => Promise<void>
}
```

### DiscoveryStore

```typescript
{
  cardStack: Profile[]
  isLoading: boolean
  hasMore: boolean
  filters: Preference
  // actions
  loadProfiles: () => Promise<void>
  like: (userId: string) => Promise<{ isMatch: boolean; matchId?: string }>
  pass: (userId: string) => Promise<void>
  applyFilters: (filters: Preference) => void
}
```

### MatchStore

```typescript
{
  matches: Match[]
  newMatchId: string | null     ← триггер для MatchModal
  isLoading: boolean
  // actions
  loadMatches: () => Promise<void>
  clearNewMatch: () => void
  unmatch: (matchId: string) => Promise<void>
}
```

### ChatStore

```typescript
{
  conversations: Conversation[]
  messages: Record<string, Message[]>
  unreadCounts: Record<string, number>
  activeConversationId: string | null
  isSending: boolean
  // actions
  loadConversations: () => Promise<void>
  loadMessages: (conversationId: string) => Promise<void>
  sendMessage: (conversationId: string, text: string) => Promise<void>
  markAsRead: (conversationId: string) => Promise<void>
  subscribeToConversation: (conversationId: string) => () => void
}
```

### SettingsStore

```typescript
{
  preferences: Preference
  privacy: PrivacySettings
  // actions
  updatePreferences: (data: Partial<Preference>) => Promise<void>
  updatePrivacy: (data: Partial<PrivacySettings>) => Promise<void>
}
```

### UI State

Локальный `useState` / `useReducer` в компонентах — не в глобальном store.  
Примеры: swipe direction, loading per-button, form state, modal open/closed.

---

## 12. Service Layer (Backend-Agnostic)

### 12.1 Принцип

UI никогда не импортирует Firebase напрямую.  
Все обращения через интерфейсы в `src/services/interfaces/`.  
Реализация в `src/services/firebase/`.  
Переключение: `src/services/index.ts` экспортирует нужную реализацию.

### AuthService

```
registerWithEmail(name, email, password) → User
loginWithEmail(email, password) → User
signInWithGoogle() → User
signInWithApple() → User
signOut() → void
sendPasswordReset(email) → void
verifyEmail() → void (resend)
changePassword(currentPassword, newPassword) → void
deleteAccount() → void
linkWithGoogle() → void
linkWithApple() → void
unlinkProvider(providerId) → void
getLinkedProviders() → string[]
onAuthStateChanged(callback) → unsubscribe
getCurrentUser() → User | null
```

### ProfileService

```
getMyProfile() → Profile
updateProfile(data) → Profile
getPreferences() → Preference
updatePreferences(data) → Preference
getPublicProfile(userId) → Profile
markProfileComplete() → void
saveOnboardingStep(step) → void
```

### MediaService

```
uploadPhoto(uri, userId) → Photo
deletePhoto(photoId) → void
reorderPhotos(userId, photoIds[]) → Photo[]
getPhotoUrl(photoId) → string
```

### DiscoveryService

```
getProfiles(userId, filters, limit) → Profile[]
passProfile(fromUserId, toUserId) → void
undoLastPass(userId) → void  (V1)
```

### MatchService

```
likeProfile(fromUserId, toUserId, isSuperLike?) → { isMatch, matchId? }
getMatches(userId) → Match[]
unmatch(matchId, userId) → void
```

### ChatService

```
getConversations(userId) → Conversation[]
getMessages(conversationId, limit, cursor?) → Message[]
sendMessage(conversationId, senderId, text) → Message
markAsRead(conversationId, userId) → void
subscribeToMessages(conversationId, callback) → unsubscribe
subscribeToConversations(userId, callback) → unsubscribe
```

### SafetyService

```
blockUser(blockerId, blockedId) → void
unblockUser(blockerId, blockedId) → void
getBlockedUsers(userId) → User[]
reportUser(reporterId, reportedId, reason, comment?) → void
```

### NotificationService (V1)

```
requestPermissions() → boolean
getNotifications(userId) → Notification[]
markAllRead(userId) → void
subscribeToNotifications(userId, callback) → unsubscribe
```

---

## 13. Design System

### 13.1 Цвета — Dark Theme (MVP, единственная тема)

```
Background:       #0F0F1A
Surface:          #1A1A2E
Surface-2:        #252540
Surface-3:        #2E2E50

Primary:          #E8395D
Primary-Dark:     #C02E4C
Primary-Light:    #FF6B8A

Secondary:        #7B61FF
Gold:             #FFD93D

Text:             #FFFFFF
Text-Secondary:   #B0B0CC
Text-Muted:       #9090A8
Text-Disabled:    #404060

Border:           #2A2A45
Border-Light:     #363660

Success:          #4CAF50
Warning:          #FF9800
Error:            #F44336
Online:           #4CAF50

Overlay-Dark:     rgba(0, 0, 0, 0.7)
Overlay-Light:    rgba(0, 0, 0, 0.4)
```

**Light Theme** — запланирован для V1. Архитектура ThemeProvider готовится сразу.

### 13.2 Типографика

Шрифт: **Inter** (Google Fonts, доступен через `expo-font`)

```
Display:    32px  / 800  / letter-spacing: -0.5
H1:         28px  / 700
H2:         24px  / 700
H3:         20px  / 600
H4:         17px  / 600
Body-LG:    17px  / 400  / line-height: 1.5
Body:       15px  / 400  / line-height: 1.5
Body-SM:    13px  / 400
Caption:    11px  / 400
```

### 13.3 Spacing (8-point grid)

```
XS:    4px
SM:    8px
MD:    12px
LG:    16px
XL:    20px
2XL:   24px
3XL:   32px
4XL:   48px
```

### 13.4 Border Radius

```
XS:    4px   ← chips, badges
SM:    8px   ← inputs, small elements
MD:    12px  ← cards
LG:    16px  ← bottom sheets
XL:    24px  ← большие карточки
Full:  9999px ← аватары, pill-кнопки
```

### 13.5 Компоненты

**Button (Primary)**
- Background: `#E8395D`, height: 52px, radius: 26px, font: 16px/600, color: white
- Pressed: background `#C02E4C`, scale 0.97
- Loading: spinner вместо label
- Disabled: opacity 0.4

**Button (Secondary)**
- Border: 2px `#E8395D`, background: transparent, same dims
- Text: `#E8395D`

**Button (Ghost)**
- No border, no bg, text `#B0B0CC`, font 14px/500

**Input**
- Height: 52px, background: `#1A1A2E`, border: 1px `#2A2A45`
- Focus border: 1px `#E8395D`
- Placeholder: `#7070A0`, text: white
- Radius: 8px, padding: 16px horizontal

**Profile Card (Discover)**
- Aspect ratio: 3:4, radius: 16px
- Photo: full cover, gradient overlay bottom 40%: `transparent → rgba(0,0,0,0.85)`
- Shadow: `0 8px 32px rgba(0,0,0,0.4)`

**Avatar**
- XS: 32px, SM: 48px, MD: 64px, LG: 96px, XL: 128px
- All: radius full, border 2px `#2A2A45`

**Bottom Tab Bar**
- Height: 64px + safe area bottom
- Background: `#1A1A2E`, border-top: 1px `#2A2A45`
- Icon: 24px, Active: `#E8395D`, Inactive: `#7070A0`
- Label: 11px/500

**Bottom Sheet / Modal**
- Background: `#1A1A2E`, radius top: 24px
- Handle: 4×36px `#2A2A45`, radius 2px
- Overlay: `rgba(0,0,0,0.7)`

**Badge (Unread)**
- Background: `#E8395D`, text: white
- Min size: 18×18px, radius full, font: 11px/600

**Skeleton**
- Animated: `#1A1A2E → #252540 → #1A1A2E`, duration: 1.5s

### 13.6 Touch Targets

- Минимум: 44×44pt (Apple HIG) / 48×48dp (Material)
- Tab bar items: 56×56pt minimum

### 13.7 Анимации

```
Swipe card:        физическая пружина (React Native Reanimated 2), velocity-based
Match Modal:       fade-in + scale 0.7→1.0 (300ms) + confetti/pulse
Screen transition: slide horizontal (stack), fade (modal)
Like/Pass buttons: scale 0.9→1.0 при tap (100ms)
Tab switch:        instant (без анимации)
```

### 13.8 Accessibility

- Contrast ratio: ≥ 4.5:1 (WCAG AA) для всего текста
- `accessibilityLabel` на все интерактивные элементы
- `accessibilityRole` для кнопок, ссылок, tab items
- `accessibilityHint` для неочевидных действий (swipe)
- Dynamic Type поддержка (iOS) — V1
- Reduced Motion respect — V1

---

## 14. Mockup Workflow

### 14.1 Порядок прототипирования

| # | Экран | Цель прототипа | Состояния |
|---|---|---|---|
| 1 | WelcomeScreen | Первое впечатление, CTA | Default; loading after tap |
| 2 | RegisterScreen | Форма регистрации | Empty; validation errors; loading |
| 3 | LoginScreen | Форма входа | Default; error state |
| 4 | OnboardingScreen | 3 слайда | Slide 1/2/3 |
| 5 | SetupPhotosScreen | Критичный шаг | 0 фото; 1 фото; 6 фото |
| 6 | DiscoverScreen | Главный экран | Карточка с фото; пустая колода; loading |
| 7 | ProfileDetailScreen | Полная анкета | С bio; без bio; все фото |
| 8 | MatchModal | Праздник совпадения | Default с двумя аватарами |
| 9 | MatchesScreen | Список совпадений | Несколько; пустой |
| 10 | ConversationsScreen | Список чатов | С unread; без; пустой |
| 11 | ChatScreen | Диалог | Начало разговора; длинный |
| 12 | MyProfileScreen | Свой профиль | Полный; без фото |
| 13 | SettingsScreen | Настройки | Default list |

### 14.2 Workflow

```
Specification (текущий документ)
  → /mockup-app (HTML Artifact, команда Claude)
    → Review в чате (выглядит ли правильно?)
      → [Нет] → Корректировка mockup
      → [Да] → Approval: "Берём этот дизайн"
        → React Native implementation
          → Expo Go / web preview
            → Device test (Android)
```

**Правило:** ни один экран не пишется в React Native без утверждённого mockup.

---

## 15. Figma Workflow

| Инструмент | Когда |
|---|---|
| Local Mockup Skill (`/mockup-app`) | Быстрая идея, черновой layout, проверка flow, итерации |
| Figma MCP (`/figma-generate-design`) | Финальный дизайн, компоненты дизайн-системы, Store screenshots |

**MVP workflow:** только Local Mockup. Figma — после MVP release, для публикации.

```
Идея экрана
  → /mockup-app (HTML Artifact)
    → Итерации
      → Approval
        → [MVP] React Native implementation
        → [Pre-release] Figma MCP → Store screenshots
```

---

## 16. Testing Strategy

### Unit / Component Tests
- Библиотека: `@testing-library/react-native` + Jest
- Что: рендер компонентов, видимость элементов, tap-обработчики
- Service layer: unit-тесты с mock реализацией

### Navigation Tests
- Переходы, back stack, params
- `@react-navigation/testing` / Expo Router testing utilities

### Form Validation
- Обязательные поля, форматы, лимиты символов
- Edge: пустые значения, спецсимволы, очень длинные строки

### Auth Testing
- Google / Apple: mock providers
- Email: регистрация, вход, восстановление
- Истёкшая сессия, неверный токен

### Matching Tests
- Like → Match (взаимный)
- Like → No Match (односторонний)
- Pass → не появляется в следующей выборке
- lookingFor фильтрация

### Chat Tests
- Отправка, получение, read state
- Unicode / emoji
- Длинное сообщение (2000 символов)
- Offline → pending → retry

### Offline / Network
- Discover без сети → offline state
- Отправка без сети → pending + retry
- Загрузка фото без сети → error

### Android Testing (основная платформа MVP)
1. Expo Go → QR-код (быстрый тест)
2. Back button behavior (`BackHandler`)
3. Keyboard behavior (`KeyboardAvoidingView`)
4. Permissions: Camera, Gallery, Location
5. Status bar, navigation bar

### iOS Testing (V1)
1. Expo Go → QR-код
2. TestFlight перед release
3. Safe Area (notch, Dynamic Island, home indicator)
4. Apple Sign-In flow
5. Permissions dialog wording (NSUsageDescription)

### Web Preview (ограниченно)
- `npx expo start --web` — только для layout/UI check
- НЕ для нативных фич: Camera, Location, Push, Auth

---

## 17. Security and Privacy

### Блокировки и жалобы
- Block немедленно скрывает обоих пользователей друг у друга (Discover + чат)
- Report сохраняется + manual review в MVP
- 3 жалобы на одного пользователя → временная блокировка (V1)

### Приватность профиля
- `isVisible: false` → не показывается в Discover
- Геолокация: только до ~1km точности (округление координат)
- "Последний раз онлайн": только "недавно" / "давно", не точное время
- Фото: только авторизованные пользователи (Firebase Storage Rules)

### Удаление аккаунта
- Soft delete: `isDeleted: true` + 30 дней хранения
- Hard delete: Firestore docs + Storage files после 30 дней
- Экспорт данных: V1 (GDPR compliance)

### Auth Security
- Tokens: хранить в **Expo SecureStore** (не AsyncStorage)
- JWT: access token 15 мин, refresh token 30 дней (Firebase default)
- Logout: revoke всех токенов

### API Security (при реальном backend)
- Rate limiting: Discover, Like, Message
- Anti-scraping: CAPTCHA после порогового количества запросов
- Firebase Security Rules: пользователь видит только свои данные

### App Store требования
- Privacy Policy: обязательна перед публикацией
- Terms of Service: обязательны
- Возрастное ограничение: 17+ (iOS) / Teen (Android)
- Соответствие Google Play User Generated Content policy
- COPPA: запрет входа детям до 18 (проверка возраста при регистрации)
- GDPR: право на удаление + экспорт (пользователи ЕС)

---

## 18. Platform Differences

| Функция | Web | Android | iOS |
|---|---|---|---|
| Google Sign-In | Частично | ✅ | ✅ |
| Apple Sign-In | ❌ | ❌ | ✅ (обязательно для App Store) |
| Camera | ❌ | ✅ | ✅ |
| Photo Library | Через `<input>` | ✅ | ✅ |
| Geolocation | Browser (ограничено) | ✅ | ✅ |
| Push Notifications | ❌ | ✅ FCM | ✅ APNs |
| Swipe Gestures | Mouse drag | ✅ Touch | ✅ Touch |
| Keyboard Avoidance | CSS | Нативный | Нативный |
| Safe Area | ❌ | ✅ | ✅ (notch, Dynamic Island, home indicator) |
| Haptic Feedback | ❌ | ✅ | ✅ |
| Back Button | Browser back | Android hardware | Gesture / swipe |

**Web используется только для:** layout проверки, цветовая схема, быстрые UI итерации.  
**Web НЕ используется для:** финального тестирования, нативных фич, Auth OAuth.

---

## 19. MVP Roadmap

### Phase 0 — Project Foundation
- Создание Expo проекта с TypeScript
- ESLint + Prettier конфигурация
- Expo Router настройка
- Базовые константы (colors, typography, spacing)
- `.env.example`
- **Критерий:** проект запускается на Android + web

### Phase 1 — Design System
- `constants/colors.ts`, `constants/typography.ts`, `constants/spacing.ts`
- Компоненты: `Button`, `Input`, `Avatar`, `Card`, `Badge`, `Skeleton`, `EmptyState`
- ThemeProvider (dark theme)
- **Критерий:** компоненты отображаются корректно на Android

### Phase 2 — Auth
- Mockup: Welcome, Register, Login, ForgotPassword, VerifyEmail
- WelcomeScreen, RegisterScreen, LoginScreen
- ForgotPasswordScreen, VerifyEmailScreen
- Firebase Auth интеграция (email/password)
- AuthStore, AuthService (Firebase impl)
- Expo SecureStore
- **Критерий:** полный email/password flow работает на Android

### Phase 3 — OAuth
- Google Sign-In (Android)
- Apple Sign-In (iOS — позже)
- OAuth linking (AccountSettings)
- **Критерий:** Google Sign-In работает на Android

### Phase 4 — Profile Setup
- Mockup: все Setup* экраны
- Setup flow (8 экранов)
- Image picker + Firebase Storage upload
- Geolocation / город вручную
- ProfileStore, ProfileService
- **Критерий:** полный onboarding flow создаёт профиль в Firestore

### Phase 5 — Discover
- Mockup: Discover, ProfileDetail, Filter, Empty
- DiscoverScreen с card stack
- Swipe gesture (Reanimated)
- ProfileDetailScreen, FilterScreen
- DiscoveryStore, DiscoveryService
- Mock профили для тестирования
- **Критерий:** свайп работает с анимацией на Android

### Phase 6 — Like / Pass / Match
- Like / Pass логика
- Matching алгоритм
- MatchModal (overlay + анимация)
- MatchStore
- **Критерий:** два пользователя создают Match

### Phase 7 — Matches
- Mockup: MatchesScreen
- MatchesScreen
- Переход Matches → Chat
- **Критерий:** список совпадений отображается

### Phase 8 — Conversations
- Mockup: ConversationsScreen
- ConversationsScreen с unread badge
- **Критерий:** список диалогов с превью

### Phase 9 — Chat
- Mockup: ChatScreen
- ChatScreen с bubble messages
- Firestore real-time (onSnapshot)
- ChatStore, ChatService
- **Критерий:** два пользователя переписываются в реальном времени

### Phase 10 — Profile & Edit
- Mockup: MyProfile, EditProfile, EditPhotos
- MyProfileScreen
- EditProfileScreen, EditPhotosScreen
- **Критерий:** профиль редактируется и сохраняется

### Phase 11 — Settings
- Mockup: Settings
- SettingsScreen и все sub-экраны
- ChangePasswordScreen
- Logout, Delete account
- **Критерий:** все settings работают, logout очищает state

### Phase 12 — Safety
- BlockedUsersScreen
- ReportModal
- Block / Report / Unmatch логика
- **Критерий:** block и report работают корректно

### Phase 13 — Testing & Polish
- Тестирование на реальном Android
- Error boundaries, offline state
- Производительность (FlatList, memo)
- Loading / error states везде
- **Критерий:** нет крашей при 30 мин использования

### Phase 14 — Release Preparation
- App icon, splash screen
- Privacy Policy, Terms of Service
- Store assets: screenshots, описания
- Signing (Android keystore)
- TestFlight (iOS)
- **Критерий:** пройден store checklist

---

## 20. Project Structure

```
znakomstva/
│
├── app/                               ← Expo Router (навигация)
│   ├── _layout.tsx                    ← Root: auth gate, providers
│   ├── (auth)/
│   │   ├── _layout.tsx
│   │   ├── onboarding.tsx
│   │   ├── welcome.tsx
│   │   ├── register.tsx
│   │   ├── login.tsx
│   │   ├── forgot-password.tsx
│   │   └── verify-email.tsx
│   ├── (onboarding)/
│   │   ├── _layout.tsx
│   │   ├── name.tsx
│   │   ├── age.tsx
│   │   ├── gender.tsx
│   │   ├── looking-for.tsx
│   │   ├── photos.tsx
│   │   ├── bio.tsx
│   │   ├── location.tsx
│   │   └── preferences.tsx
│   ├── (main)/
│   │   ├── _layout.tsx                ← Bottom Tabs
│   │   ├── discover/
│   │   │   ├── _layout.tsx
│   │   │   ├── index.tsx
│   │   │   ├── [userId].tsx
│   │   │   └── filter.tsx
│   │   ├── matches/
│   │   │   ├── _layout.tsx
│   │   │   └── index.tsx
│   │   ├── messages/
│   │   │   ├── _layout.tsx
│   │   │   ├── index.tsx
│   │   │   └── [matchId].tsx
│   │   └── profile/
│   │       ├── _layout.tsx
│   │       ├── index.tsx
│   │       ├── edit.tsx
│   │       ├── photos.tsx
│   │       └── settings/
│   │           ├── index.tsx
│   │           ├── preferences.tsx
│   │           ├── privacy.tsx
│   │           ├── account.tsx
│   │           ├── change-password.tsx
│   │           └── blocked-users.tsx
│   └── modals/
│       ├── _layout.tsx
│       ├── match.tsx
│       ├── chat-menu.tsx
│       └── report.tsx
│
├── src/
│   ├── components/
│   │   ├── ui/                        ← Button, Input, Avatar, Badge, Card...
│   │   ├── forms/                     ← ProfileForm, PreferencesForm...
│   │   └── common/                    ← EmptyState, ErrorBoundary, Skeleton...
│   │
│   ├── services/
│   │   ├── interfaces/                ← TypeScript interfaces (AuthService, etc.)
│   │   ├── firebase/                  ← Firebase реализации
│   │   └── mock/                      ← Mock реализации для разработки
│   │
│   ├── store/
│   │   ├── authStore.ts
│   │   ├── profileStore.ts
│   │   ├── discoveryStore.ts
│   │   ├── matchStore.ts
│   │   ├── chatStore.ts
│   │   └── settingsStore.ts
│   │
│   ├── types/
│   │   ├── models.ts                  ← User, Profile, Match, Message...
│   │   └── api.ts                     ← Response types
│   │
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useDiscovery.ts
│   │   ├── useChat.ts
│   │   └── useLocation.ts
│   │
│   ├── utils/
│   │   ├── date.ts
│   │   ├── distance.ts
│   │   └── validation.ts
│   │
│   └── constants/
│       ├── colors.ts
│       ├── typography.ts
│       ├── spacing.ts
│       └── config.ts
│
├── assets/
│   ├── images/                        ← app icon, splash, illustrations
│   └── animations/                    ← Lottie JSON
│
├── docs/
│   └── PRODUCT_SPECIFICATION.md       ← этот файл
│
├── .claude/
│   └── skills/
│       └── mockup-app/
│           └── SKILL.md
│
├── app.json                           ← Expo конфиг
├── tsconfig.json
├── package.json
├── .env.example
├── .gitignore
└── README.md
```

---

## 21. Claude Code Workflow

### Главное правило

**Одна функция — один цикл. Без забегания вперёд.**

### Цикл каждой задачи

```
1. Изучить существующий код в затрагиваемой области
2. Сформулировать план: что меняется, какие файлы, почему
3. [Если >50 строк или >2 файла] Показать план, дождаться "ок"
4. Реализовать только то, что в плане
5. Запустить: tsc, ESLint, тесты
6. Исправить ошибки (в рамках плана)
7. Показать результат
8. Только потом — следующая задача
```

### Mockup-first правило

```
Перед каждым новым экраном:
  1. /mockup-app → создать HTML Artifact
  2. Показать пользователю
  3. Дождаться "ок, берём этот дизайн"
  4. Только тогда писать React Native код
```

### Что Claude НЕ делает без явного запроса

- Не добавляет зависимости без обоснования
- Не рефакторит код вне текущей задачи
- Не "улучшает" попутно
- Не создаёт файлы за пределами согласованной структуры
- Не изменяет дизайн-систему при реализации функции
- Не пишет несколько экранов за один раз
- Не добавляет функции, которых нет в этой спецификации

---

## 22. Publication Checklist

### Android — Google Play

```
□ applicationId:    com.znakomstva.app
□ versionCode:      1
□ versionName:      1.0.0
□ App icon:         1024×1024 PNG, без прозрачности
□ Feature graphic:  1024×500 PNG
□ Screenshots:      мин 2, phone + 7" tablet
□ Short desc:       ≤80 символов
□ Full desc:        ≤4000 символов
□ Category:         Social / Dating
□ Content rating:   Teen (IARC questionnaire)
□ Privacy Policy:   URL обязателен
□ Permissions:      Camera, Storage, Location — с обоснованием
□ Data safety form: что собираем, для чего
□ Signing:          Upload key (keystore) + бэкап
□ Build format:     AAB (не APK)
□ Stages:           Internal → Closed → Open → Production
```

### iOS — Apple App Store

```
□ Bundle ID:        com.znakomstva.app
□ Version:          1.0.0 (Build 1)
□ App icon:         1024×1024 PNG, без скруглений
□ Screenshots:      6.7" + 6.1" (обязательно)
□ App Name:         ≤30 символов
□ Subtitle:         ≤30 символов
□ Keywords:         ≤100 символов
□ Privacy Policy:   URL обязателен
□ Apple Sign-In:    обязателен (есть другие OAuth)
□ Age Rating:       17+
□ Privacy Manifest: Camera, Photo Library, Location
□ TestFlight:       перед release
□ Review notes:     тестовый аккаунт для Apple team
□ Certificates:     Distribution cert + Provisioning Profile
```

---

*Документ создан: 2026-09-25. Следующий этап: Mockup Phase.*
