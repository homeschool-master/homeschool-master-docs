# Homeschool Management App - Mobile App Architecture Document

**Version:** 1.0  
**Date:** November 30, 2025  
**Status:** Initial Design  
**Platform:** React Native (iOS & Android)  
**Approach:** Option A - Separate Codebases with Shared Patterns

---

## Table of Contents
1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Project Structure](#project-structure)
4. [Core Architecture Patterns](#core-architecture-patterns)
5. [State Management](#state-management)
6. [Navigation Architecture](#navigation-architecture)
7. [API Service Layer](#api-service-layer)
8. [Authentication Flow](#authentication-flow)
9. [Data Models & TypeScript](#data-models--typescript)
10. [Component Architecture](#component-architecture)
11. [Offline Support Strategy](#offline-support-strategy)
12. [Security Implementation](#security-implementation)
13. [Error Handling](#error-handling)
14. [Testing Strategy](#testing-strategy)
15. [Performance Optimization](#performance-optimization)
16. [Future Web Migration](#future-web-migration)
17. [Development Workflow](#development-workflow)

---

## Overview

### Purpose
This document defines the frontend architecture for the Homeschool Management mobile application built with React Native. It establishes patterns, conventions, and best practices that will enable efficient development, maintainability, and future code sharing with a web version.

### Goals
1. **Build a performant React Native app** for iOS and Android
2. **Establish clean, reusable patterns** that translate to web
3. **Separate concerns** between UI, business logic, and data
4. **Enable offline-first functionality** for homeschool families on the go
5. **Create a maintainable codebase** for solo development and future team scaling

### Design Principles
- **Separation of Concerns** - UI components don't contain business logic
- **Single Responsibility** - Each module does one thing well
- **DRY (Don't Repeat Yourself)** - Extract reusable logic into hooks and utilities
- **Type Safety** - TypeScript everywhere for fewer runtime errors
- **Offline First** - App should work without constant internet
- **Platform Agnostic Logic** - Business logic should be portable to web

---

## Technology Stack

### Core Framework
| Technology | Version | Purpose |
|------------|---------|---------|
| React Native | 0.73+ | Cross-platform mobile framework |
| TypeScript | 5.0+ | Type safety and developer experience |
| Expo | SDK 50+ | Development tooling and native APIs |

### State Management
| Technology | Purpose |
|------------|---------|
| TanStack Query (React Query) | Server state, caching, synchronization |
| Zustand | Client state (UI state, user preferences) |

### Navigation
| Technology | Purpose |
|------------|---------|
| React Navigation 6 | Screen navigation, deep linking |
| @react-navigation/native-stack | Native stack navigator |
| @react-navigation/bottom-tabs | Bottom tab navigator |

### Data & Networking
| Technology | Purpose |
|------------|---------|
| Axios | HTTP client with interceptors |
| AsyncStorage | Persistent key-value storage |
| expo-secure-store | Secure token storage |

### UI & Styling
| Technology | Purpose |
|------------|---------|
| NativeWind (Tailwind) | Utility-first styling |
| React Native Reanimated | Smooth animations |
| React Native Gesture Handler | Native gestures |

### Forms & Validation
| Technology | Purpose |
|------------|---------|
| React Hook Form | Form state management |
| Zod | Schema validation |

### Development & Testing
| Technology | Purpose |
|------------|---------|
| Jest | Unit testing |
| React Native Testing Library | Component testing |
| Detox | End-to-end testing |
| ESLint + Prettier | Code quality and formatting |

---

## Project Structure

### Directory Layout

```
homeschool-mobile/
├── app/                          # Expo Router screens (if using Expo Router)
├── src/
│   ├── api/                      # API service layer
│   │   ├── client.ts             # Axios instance configuration
│   │   ├── endpoints/            # API endpoint functions
│   │   │   ├── auth.ts
│   │   │   ├── students.ts
│   │   │   ├── assignments.ts
│   │   │   ├── calendar.ts
│   │   │   ├── tasks.ts
│   │   │   ├── reportCards.ts
│   │   │   ├── expenses.ts
│   │   │   ├── subjects.ts
│   │   │   └── lessonPlans.ts
│   │   └── index.ts              # API exports
│   │
│   ├── components/               # Reusable UI components
│   │   ├── common/               # Generic components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── LoadingSpinner.tsx
│   │   │   ├── ErrorMessage.tsx
│   │   │   ├── EmptyState.tsx
│   │   │   └── Avatar.tsx
│   │   │
│   │   ├── forms/                # Form-specific components
│   │   │   ├── FormInput.tsx
│   │   │   ├── FormSelect.tsx
│   │   │   ├── FormDatePicker.tsx
│   │   │   └── FormSwitch.tsx
│   │   │
│   │   ├── layout/               # Layout components
│   │   │   ├── ScreenContainer.tsx
│   │   │   ├── Header.tsx
│   │   │   ├── TabBar.tsx
│   │   │   └── SafeAreaWrapper.tsx
│   │   │
│   │   └── features/             # Feature-specific components
│   │       ├── students/
│   │       │   ├── StudentCard.tsx
│   │       │   ├── StudentList.tsx
│   │       │   └── StudentForm.tsx
│   │       ├── calendar/
│   │       │   ├── CalendarView.tsx
│   │       │   ├── EventCard.tsx
│   │       │   └── EventForm.tsx
│   │       ├── assignments/
│   │       │   ├── AssignmentCard.tsx
│   │       │   ├── AssignmentList.tsx
│   │       │   └── AssignmentForm.tsx
│   │       ├── tasks/
│   │       │   ├── TaskCard.tsx
│   │       │   ├── TaskList.tsx
│   │       │   └── TaskForm.tsx
│   │       ├── reportCards/
│   │       │   ├── ReportCardView.tsx
│   │       │   ├── ReportCardEntry.tsx
│   │       │   └── ReportCardForm.tsx
│   │       ├── expenses/
│   │       │   ├── ExpenseCard.tsx
│   │       │   ├── ExpenseList.tsx
│   │       │   ├── ExpenseForm.tsx
│   │       │   └── ExpenseSummary.tsx
│   │       └── lessonPlans/
│   │           ├── LessonPlanCard.tsx
│   │           ├── LessonPlanList.tsx
│   │           └── LessonPlanForm.tsx
│   │
│   ├── hooks/                    # Custom React hooks
│   │   ├── api/                  # API-related hooks (React Query)
│   │   │   ├── useAuth.ts
│   │   │   ├── useStudents.ts
│   │   │   ├── useAssignments.ts
│   │   │   ├── useCalendar.ts
│   │   │   ├── useTasks.ts
│   │   │   ├── useReportCards.ts
│   │   │   ├── useExpenses.ts
│   │   │   ├── useSubjects.ts
│   │   │   └── useLessonPlans.ts
│   │   │
│   │   └── common/               # General-purpose hooks
│   │       ├── useDebounce.ts
│   │       ├── useKeyboard.ts
│   │       ├── useRefreshControl.ts
│   │       ├── useNetworkStatus.ts
│   │       └── useBiometrics.ts
│   │
│   ├── screens/                  # Screen components
│   │   ├── auth/
│   │   │   ├── LoginScreen.tsx
│   │   │   ├── RegisterScreen.tsx
│   │   │   ├── ForgotPasswordScreen.tsx
│   │   │   └── ResetPasswordScreen.tsx
│   │   │
│   │   ├── dashboard/
│   │   │   └── DashboardScreen.tsx
│   │   │
│   │   ├── students/
│   │   │   ├── StudentsListScreen.tsx
│   │   │   ├── StudentDetailScreen.tsx
│   │   │   └── StudentFormScreen.tsx
│   │   │
│   │   ├── calendar/
│   │   │   ├── CalendarScreen.tsx
│   │   │   ├── EventDetailScreen.tsx
│   │   │   └── EventFormScreen.tsx
│   │   │
│   │   ├── assignments/
│   │   │   ├── AssignmentsListScreen.tsx
│   │   │   ├── AssignmentDetailScreen.tsx
│   │   │   └── AssignmentFormScreen.tsx
│   │   │
│   │   ├── tasks/
│   │   │   ├── TasksListScreen.tsx
│   │   │   └── TaskFormScreen.tsx
│   │   │
│   │   ├── reportCards/
│   │   │   ├── ReportCardsListScreen.tsx
│   │   │   ├── ReportCardDetailScreen.tsx
│   │   │   └── ReportCardFormScreen.tsx
│   │   │
│   │   ├── expenses/
│   │   │   ├── ExpensesListScreen.tsx
│   │   │   ├── ExpenseDetailScreen.tsx
│   │   │   ├── ExpenseFormScreen.tsx
│   │   │   └── ExpenseReportScreen.tsx
│   │   │
│   │   ├── lessonPlans/
│   │   │   ├── LessonPlansListScreen.tsx
│   │   │   ├── LessonPlanDetailScreen.tsx
│   │   │   ├── LessonPlanFormScreen.tsx
│   │   │   └── PublicLessonPlansScreen.tsx
│   │   │
│   │   └── settings/
│   │       ├── SettingsScreen.tsx
│   │       ├── ProfileScreen.tsx
│   │       ├── SubjectsScreen.tsx
│   │       └── CategoriesScreen.tsx
│   │
│   ├── navigation/               # Navigation configuration
│   │   ├── RootNavigator.tsx
│   │   ├── AuthNavigator.tsx
│   │   ├── MainNavigator.tsx
│   │   ├── TabNavigator.tsx
│   │   └── linking.ts            # Deep linking config
│   │
│   ├── store/                    # Zustand stores (client state)
│   │   ├── authStore.ts
│   │   ├── uiStore.ts
│   │   └── offlineStore.ts
│   │
│   ├── types/                    # TypeScript definitions
│   │   ├── api.ts                # API response types
│   │   ├── models.ts             # Data model interfaces
│   │   ├── navigation.ts         # Navigation param types
│   │   └── forms.ts              # Form types
│   │
│   ├── utils/                    # Utility functions
│   │   ├── formatting.ts         # Date, currency, text formatting
│   │   ├── validation.ts         # Zod schemas
│   │   ├── storage.ts            # AsyncStorage helpers
│   │   ├── secureStorage.ts      # Secure token storage
│   │   ├── constants.ts          # App constants
│   │   └── helpers.ts            # General helpers
│   │
│   ├── theme/                    # Design system
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   └── index.ts
│   │
│   └── config/                   # App configuration
│       ├── env.ts                # Environment variables
│       └── queryClient.ts        # React Query configuration
│
├── assets/                       # Static assets
│   ├── images/
│   ├── icons/
│   └── fonts/
│
├── __tests__/                    # Test files
│   ├── components/
│   ├── hooks/
│   ├── screens/
│   └── utils/
│
├── app.json                      # Expo configuration
├── babel.config.js
├── tsconfig.json
├── package.json
├── tailwind.config.js            # NativeWind config
└── README.md
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Components | PascalCase | `StudentCard.tsx` |
| Hooks | camelCase with "use" prefix | `useStudents.ts` |
| Utilities | camelCase | `formatting.ts` |
| Types/Interfaces | PascalCase | `Student`, `ApiResponse` |
| Constants | SCREAMING_SNAKE_CASE | `API_BASE_URL` |
| Files | Match export name | `StudentCard.tsx` exports `StudentCard` |

---

## Core Architecture Patterns

### Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│              (Screens, Components, Navigation)               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     HOOKS LAYER                              │
│          (Custom Hooks, React Query Hooks)                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                             │
│              (API Client, Storage Services)                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     DATA LAYER                               │
│           (API Endpoints, Local Storage)                     │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Action
    │
    ▼
Screen (captures intent)
    │
    ▼
Hook (manages state & logic)
    │
    ├──────────────────┐
    ▼                  ▼
React Query        Zustand
(server state)     (client state)
    │
    ▼
API Service
    │
    ▼
Backend API
```

---

## State Management

### Two Types of State

#### 1. Server State (React Query / TanStack Query)
Data that comes from the backend API:
- Students, assignments, calendar events, etc.
- Cached automatically
- Background refetching
- Optimistic updates

#### 2. Client State (Zustand)
Data that exists only on the client:
- Authentication status
- UI state (modals, selected tabs)
- User preferences
- Offline queue

### React Query Setup

**Configuration (`src/config/queryClient.ts`):**

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Data stays fresh for 5 minutes
      staleTime: 5 * 60 * 1000,
      
      // Cache data for 30 minutes
      gcTime: 30 * 60 * 1000,
      
      // Retry failed requests 3 times
      retry: 3,
      
      // Refetch when app comes to foreground
      refetchOnWindowFocus: true,
      
      // Refetch when network reconnects
      refetchOnReconnect: true,
    },
    mutations: {
      // Retry mutations once
      retry: 1,
    },
  },
});
```

**Query Keys Convention:**

```typescript
// src/hooks/api/queryKeys.ts

export const queryKeys = {
  // Students
  students: {
    all: ['students'] as const,
    list: (filters?: StudentFilters) => ['students', 'list', filters] as const,
    detail: (id: string) => ['students', 'detail', id] as const,
  },
  
  // Assignments
  assignments: {
    all: ['assignments'] as const,
    list: (filters?: AssignmentFilters) => ['assignments', 'list', filters] as const,
    detail: (id: string) => ['assignments', 'detail', id] as const,
    byStudent: (studentId: string) => ['assignments', 'student', studentId] as const,
  },
  
  // Calendar Events
  calendar: {
    all: ['calendar'] as const,
    events: (dateRange: DateRange) => ['calendar', 'events', dateRange] as const,
    detail: (id: string) => ['calendar', 'detail', id] as const,
  },
  
  // Tasks
  tasks: {
    all: ['tasks'] as const,
    list: (filters?: TaskFilters) => ['tasks', 'list', filters] as const,
    detail: (id: string) => ['tasks', 'detail', id] as const,
  },
  
  // Report Cards
  reportCards: {
    all: ['reportCards'] as const,
    list: (filters?: ReportCardFilters) => ['reportCards', 'list', filters] as const,
    detail: (id: string) => ['reportCards', 'detail', id] as const,
    byStudent: (studentId: string) => ['reportCards', 'student', studentId] as const,
  },
  
  // Expenses
  expenses: {
    all: ['expenses'] as const,
    list: (filters?: ExpenseFilters) => ['expenses', 'list', filters] as const,
    detail: (id: string) => ['expenses', 'detail', id] as const,
    summary: (filters: ExpenseSummaryFilters) => ['expenses', 'summary', filters] as const,
    categories: ['expenses', 'categories'] as const,
  },
  
  // Subjects
  subjects: {
    all: ['subjects'] as const,
    list: () => ['subjects', 'list'] as const,
    detail: (id: string) => ['subjects', 'detail', id] as const,
  },
  
  // Lesson Plans
  lessonPlans: {
    all: ['lessonPlans'] as const,
    list: (filters?: LessonPlanFilters) => ['lessonPlans', 'list', filters] as const,
    detail: (id: string) => ['lessonPlans', 'detail', id] as const,
    public: (filters?: PublicLessonPlanFilters) => ['lessonPlans', 'public', filters] as const,
  },
  
  // Teacher
  teacher: {
    profile: ['teacher', 'profile'] as const,
  },
};
```

### Zustand Store Setup

**Auth Store (`src/store/authStore.ts`):**

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Teacher } from '@/types/models';

interface AuthState {
  // State
  isAuthenticated: boolean;
  isLoading: boolean;
  teacher: Teacher | null;
  
  // Actions
  setTeacher: (teacher: Teacher) => void;
  setAuthenticated: (value: boolean) => void;
  setLoading: (value: boolean) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      // Initial state
      isAuthenticated: false,
      isLoading: true,
      teacher: null,
      
      // Actions
      setTeacher: (teacher) => set({ teacher, isAuthenticated: true }),
      setAuthenticated: (value) => set({ isAuthenticated: value }),
      setLoading: (value) => set({ isLoading: value }),
      logout: () => set({ teacher: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({ 
        teacher: state.teacher,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);
```

**UI Store (`src/store/uiStore.ts`):**

```typescript
import { create } from 'zustand';

interface UIState {
  // Selected filters
  selectedStudentId: string | null;
  selectedDateRange: { start: Date; end: Date } | null;
  
  // Modal states
  isAddStudentModalOpen: boolean;
  isAddEventModalOpen: boolean;
  
  // Actions
  setSelectedStudentId: (id: string | null) => void;
  setSelectedDateRange: (range: { start: Date; end: Date } | null) => void;
  openAddStudentModal: () => void;
  closeAddStudentModal: () => void;
  openAddEventModal: () => void;
  closeAddEventModal: () => void;
}

export const useUIStore = create<UIState>((set) => ({
  // Initial state
  selectedStudentId: null,
  selectedDateRange: null,
  isAddStudentModalOpen: false,
  isAddEventModalOpen: false,
  
  // Actions
  setSelectedStudentId: (id) => set({ selectedStudentId: id }),
  setSelectedDateRange: (range) => set({ selectedDateRange: range }),
  openAddStudentModal: () => set({ isAddStudentModalOpen: true }),
  closeAddStudentModal: () => set({ isAddStudentModalOpen: false }),
  openAddEventModal: () => set({ isAddEventModalOpen: true }),
  closeAddEventModal: () => set({ isAddEventModalOpen: false }),
}));
```

---

## Navigation Architecture

### Navigation Structure

```
RootNavigator
├── AuthNavigator (when not authenticated)
│   ├── LoginScreen
│   ├── RegisterScreen
│   ├── ForgotPasswordScreen
│   └── ResetPasswordScreen
│
└── MainNavigator (when authenticated)
    └── TabNavigator
        ├── DashboardStack
        │   └── DashboardScreen
        │
        ├── StudentsStack
        │   ├── StudentsListScreen
        │   ├── StudentDetailScreen
        │   └── StudentFormScreen
        │
        ├── CalendarStack
        │   ├── CalendarScreen
        │   ├── EventDetailScreen
        │   └── EventFormScreen
        │
        ├── TasksStack
        │   ├── TasksListScreen
        │   └── TaskFormScreen
        │
        └── MoreStack
            ├── MoreMenuScreen
            ├── AssignmentsListScreen
            ├── AssignmentDetailScreen
            ├── AssignmentFormScreen
            ├── ReportCardsListScreen
            ├── ReportCardDetailScreen
            ├── ReportCardFormScreen
            ├── ExpensesListScreen
            ├── ExpenseDetailScreen
            ├── ExpenseFormScreen
            ├── ExpenseReportScreen
            ├── LessonPlansListScreen
            ├── LessonPlanDetailScreen
            ├── LessonPlanFormScreen
            ├── PublicLessonPlansScreen
            ├── SettingsScreen
            ├── ProfileScreen
            ├── SubjectsScreen
            └── CategoriesScreen
```

### Navigation Type Definitions

**`src/types/navigation.ts`:**

```typescript
import { NavigatorScreenParams } from '@react-navigation/native';
import { NativeStackScreenProps } from '@react-navigation/native-stack';
import { BottomTabScreenProps } from '@react-navigation/bottom-tabs';

// Auth Stack
export type AuthStackParamList = {
  Login: undefined;
  Register: undefined;
  ForgotPassword: undefined;
  ResetPassword: { token: string };
};

// Dashboard Stack
export type DashboardStackParamList = {
  Dashboard: undefined;
};

// Students Stack
export type StudentsStackParamList = {
  StudentsList: undefined;
  StudentDetail: { studentId: string };
  StudentForm: { studentId?: string }; // undefined = create, defined = edit
};

// Calendar Stack
export type CalendarStackParamList = {
  Calendar: { date?: string };
  EventDetail: { eventId: string };
  EventForm: { eventId?: string; date?: string };
};

// Tasks Stack
export type TasksStackParamList = {
  TasksList: undefined;
  TaskForm: { taskId?: string };
};

// More Stack (contains less frequent features)
export type MoreStackParamList = {
  MoreMenu: undefined;
  AssignmentsList: { studentId?: string };
  AssignmentDetail: { assignmentId: string };
  AssignmentForm: { assignmentId?: string; studentId?: string };
  ReportCardsList: { studentId?: string };
  ReportCardDetail: { reportCardId: string };
  ReportCardForm: { reportCardId?: string; studentId?: string };
  ExpensesList: undefined;
  ExpenseDetail: { expenseId: string };
  ExpenseForm: { expenseId?: string };
  ExpenseReport: undefined;
  LessonPlansList: undefined;
  LessonPlanDetail: { lessonPlanId: string };
  LessonPlanForm: { lessonPlanId?: string };
  PublicLessonPlans: undefined;
  Settings: undefined;
  Profile: undefined;
  Subjects: undefined;
  Categories: undefined;
};

// Tab Navigator
export type TabParamList = {
  DashboardTab: NavigatorScreenParams<DashboardStackParamList>;
  StudentsTab: NavigatorScreenParams<StudentsStackParamList>;
  CalendarTab: NavigatorScreenParams<CalendarStackParamList>;
  TasksTab: NavigatorScreenParams<TasksStackParamList>;
  MoreTab: NavigatorScreenParams<MoreStackParamList>;
};

// Root Navigator
export type RootStackParamList = {
  Auth: NavigatorScreenParams<AuthStackParamList>;
  Main: NavigatorScreenParams<TabParamList>;
};

// Helper types for screen props
export type AuthScreenProps<T extends keyof AuthStackParamList> = 
  NativeStackScreenProps<AuthStackParamList, T>;

export type StudentsScreenProps<T extends keyof StudentsStackParamList> = 
  NativeStackScreenProps<StudentsStackParamList, T>;

export type CalendarScreenProps<T extends keyof CalendarStackParamList> = 
  NativeStackScreenProps<CalendarStackParamList, T>;
```

### Navigation Implementation

**Root Navigator (`src/navigation/RootNavigator.tsx`):**

```typescript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useAuthStore } from '@/store/authStore';
import { AuthNavigator } from './AuthNavigator';
import { MainNavigator } from './MainNavigator';
import { linking } from './linking';
import { LoadingScreen } from '@/screens/LoadingScreen';

const Stack = createNativeStackNavigator<RootStackParamList>();

export function RootNavigator() {
  const { isAuthenticated, isLoading } = useAuthStore();
  
  if (isLoading) {
    return <LoadingScreen />;
  }
  
  return (
    <NavigationContainer linking={linking}>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="Main" component={MainNavigator} />
        ) : (
          <Stack.Screen name="Auth" component={AuthNavigator} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### Deep Linking Configuration

**`src/navigation/linking.ts`:**

```typescript
import { LinkingOptions } from '@react-navigation/native';
import { RootStackParamList } from '@/types/navigation';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['homeschoolapp://', 'https://app.homeschoolapp.com'],
  
  config: {
    screens: {
      Auth: {
        screens: {
          Login: 'login',
          Register: 'register',
          ForgotPassword: 'forgot-password',
          ResetPassword: 'reset-password/:token',
        },
      },
      Main: {
        screens: {
          DashboardTab: {
            screens: {
              Dashboard: 'dashboard',
            },
          },
          StudentsTab: {
            screens: {
              StudentsList: 'students',
              StudentDetail: 'students/:studentId',
              StudentForm: 'students/edit/:studentId?',
            },
          },
          CalendarTab: {
            screens: {
              Calendar: 'calendar/:date?',
              EventDetail: 'events/:eventId',
              EventForm: 'events/edit/:eventId?',
            },
          },
        },
      },
    },
  },
};
```

---

## API Service Layer

### HTTP Client Setup

**`src/api/client.ts`:**

```typescript
import axios, { AxiosInstance, AxiosError, InternalAxiosRequestConfig } from 'axios';
import * as SecureStore from 'expo-secure-store';
import { API_BASE_URL } from '@/config/env';
import { useAuthStore } from '@/store/authStore';

// Token keys
const ACCESS_TOKEN_KEY = 'access_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

// Create axios instance
const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// Request interceptor - add auth token
apiClient.interceptors.request.use(
  async (config: InternalAxiosRequestConfig) => {
    const token = await SecureStore.getItemAsync(ACCESS_TOKEN_KEY);
    
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor - handle token refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config;
    
    // If 401 and not a retry, attempt token refresh
    if (error.response?.status === 401 && originalRequest && !originalRequest._retry) {
      originalRequest._retry = true;
      
      try {
        const refreshToken = await SecureStore.getItemAsync(REFRESH_TOKEN_KEY);
        
        if (!refreshToken) {
          throw new Error('No refresh token');
        }
        
        // Call refresh endpoint
        const response = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refresh_token: refreshToken,
        });
        
        const { access_token } = response.data.data;
        
        // Store new access token
        await SecureStore.setItemAsync(ACCESS_TOKEN_KEY, access_token);
        
        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${access_token}`;
        return apiClient(originalRequest);
        
      } catch (refreshError) {
        // Refresh failed, logout user
        await SecureStore.deleteItemAsync(ACCESS_TOKEN_KEY);
        await SecureStore.deleteItemAsync(REFRESH_TOKEN_KEY);
        useAuthStore.getState().logout();
        return Promise.reject(refreshError);
      }
    }
    
    return Promise.reject(error);
  }
);

// Token management helpers
export const tokenService = {
  async setTokens(accessToken: string, refreshToken: string) {
    await SecureStore.setItemAsync(ACCESS_TOKEN_KEY, accessToken);
    await SecureStore.setItemAsync(REFRESH_TOKEN_KEY, refreshToken);
  },
  
  async clearTokens() {
    await SecureStore.deleteItemAsync(ACCESS_TOKEN_KEY);
    await SecureStore.deleteItemAsync(REFRESH_TOKEN_KEY);
  },
  
  async getAccessToken() {
    return SecureStore.getItemAsync(ACCESS_TOKEN_KEY);
  },
  
  async hasTokens() {
    const token = await SecureStore.getItemAsync(ACCESS_TOKEN_KEY);
    return !!token;
  },
};

export default apiClient;
```

### API Endpoint Functions

**`src/api/endpoints/students.ts`:**

```typescript
import apiClient from '../client';
import { 
  Student, 
  CreateStudentDto, 
  UpdateStudentDto,
  ApiResponse,
  PaginatedResponse,
} from '@/types';

const STUDENTS_BASE = '/students';

export const studentsApi = {
  // Get all students
  getAll: async (filters?: StudentFilters): Promise<PaginatedResponse<Student>> => {
    const response = await apiClient.get(STUDENTS_BASE, { params: filters });
    return response.data;
  },
  
  // Get single student
  getById: async (id: string): Promise<ApiResponse<Student>> => {
    const response = await apiClient.get(`${STUDENTS_BASE}/${id}`);
    return response.data;
  },
  
  // Create student
  create: async (data: CreateStudentDto): Promise<ApiResponse<Student>> => {
    const response = await apiClient.post(STUDENTS_BASE, data);
    return response.data;
  },
  
  // Update student
  update: async (id: string, data: UpdateStudentDto): Promise<ApiResponse<Student>> => {
    const response = await apiClient.put(`${STUDENTS_BASE}/${id}`, data);
    return response.data;
  },
  
  // Delete student
  delete: async (id: string, permanent = false): Promise<void> => {
    await apiClient.delete(`${STUDENTS_BASE}/${id}`, { 
      params: { permanent } 
    });
  },
  
  // Upload profile image
  uploadProfileImage: async (id: string, imageUri: string): Promise<ApiResponse<{ profile_image_url: string }>> => {
    const formData = new FormData();
    formData.append('file', {
      uri: imageUri,
      type: 'image/jpeg',
      name: 'profile.jpg',
    } as any);
    
    const response = await apiClient.post(
      `${STUDENTS_BASE}/${id}/profile-image`,
      formData,
      { headers: { 'Content-Type': 'multipart/form-data' } }
    );
    return response.data;
  },
};
```

### API Index Export

**`src/api/index.ts`:**

```typescript
export { default as apiClient } from './client';
export { tokenService } from './client';

// Export all endpoint modules
export * from './endpoints/auth';
export * from './endpoints/students';
export * from './endpoints/assignments';
export * from './endpoints/calendar';
export * from './endpoints/tasks';
export * from './endpoints/reportCards';
export * from './endpoints/expenses';
export * from './endpoints/subjects';
export * from './endpoints/lessonPlans';
```

---

## Authentication Flow

### Complete Auth Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        APP STARTUP                               │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Check for stored     │
              │  access token         │
              └───────────┬───────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
            ▼                           ▼
     ┌─────────────┐           ┌─────────────────┐
     │ Token Found │           │ No Token Found  │
     └──────┬──────┘           └────────┬────────┘
            │                           │
            ▼                           ▼
     ┌─────────────────┐       ┌─────────────────┐
     │ Validate token  │       │ Show Login      │
     │ (call /me)      │       │ Screen          │
     └────────┬────────┘       └─────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌────────┐         ┌────────────┐
│ Valid  │         │ Invalid/   │
│        │         │ Expired    │
└───┬────┘         └─────┬──────┘
    │                    │
    ▼                    ▼
┌──────────────┐   ┌────────────────┐
│ Show Main    │   │ Try Refresh    │
│ Navigator    │   │ Token          │
└──────────────┘   └───────┬────────┘
                           │
                 ┌─────────┴─────────┐
                 │                   │
                 ▼                   ▼
          ┌───────────┐       ┌───────────────┐
          │ Refresh   │       │ Refresh       │
          │ Success   │       │ Failed        │
          └─────┬─────┘       └───────┬───────┘
                │                     │
                ▼                     ▼
         ┌──────────────┐     ┌─────────────────┐
         │ Show Main    │     │ Clear tokens,   │
         │ Navigator    │     │ Show Login      │
         └──────────────┘     └─────────────────┘
```

### Auth Hook Implementation

**`src/hooks/api/useAuth.ts`:**

```typescript
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { useAuthStore } from '@/store/authStore';
import { authApi, tokenService } from '@/api';
import { queryKeys } from './queryKeys';
import { LoginDto, RegisterDto, Teacher } from '@/types';

export function useAuth() {
  const queryClient = useQueryClient();
  const { setTeacher, setAuthenticated, setLoading, logout: logoutStore } = useAuthStore();
  
  // Check if user is authenticated on app start
  const { isLoading: isCheckingAuth } = useQuery({
    queryKey: queryKeys.teacher.profile,
    queryFn: async () => {
      const hasTokens = await tokenService.hasTokens();
      
      if (!hasTokens) {
        setLoading(false);
        return null;
      }
      
      try {
        const response = await authApi.getProfile();
        setTeacher(response.data);
        setLoading(false);
        return response.data;
      } catch (error) {
        await tokenService.clearTokens();
        setLoading(false);
        return null;
      }
    },
    staleTime: Infinity,
    retry: false,
  });
  
  // Login mutation
  const loginMutation = useMutation({
    mutationFn: (credentials: LoginDto) => authApi.login(credentials),
    onSuccess: async (response) => {
      const { teacher, tokens } = response.data;
      
      // Store tokens securely
      await tokenService.setTokens(tokens.access_token, tokens.refresh_token);
      
      // Update auth state
      setTeacher(teacher);
      
      // Cache teacher data
      queryClient.setQueryData(queryKeys.teacher.profile, teacher);
    },
  });
  
  // Register mutation
  const registerMutation = useMutation({
    mutationFn: (data: RegisterDto) => authApi.register(data),
    onSuccess: async (response) => {
      const { teacher, tokens } = response.data;
      
      await tokenService.setTokens(tokens.access_token, tokens.refresh_token);
      setTeacher(teacher);
      queryClient.setQueryData(queryKeys.teacher.profile, teacher);
    },
  });
  
  // Logout mutation
  const logoutMutation = useMutation({
    mutationFn: async () => {
      const refreshToken = await tokenService.getRefreshToken();
      if (refreshToken) {
        await authApi.logout(refreshToken);
      }
    },
    onSettled: async () => {
      // Always clear local state, even if API call fails
      await tokenService.clearTokens();
      logoutStore();
      queryClient.clear();
    },
  });
  
  return {
    isCheckingAuth,
    login: loginMutation.mutateAsync,
    isLoggingIn: loginMutation.isPending,
    loginError: loginMutation.error,
    register: registerMutation.mutateAsync,
    isRegistering: registerMutation.isPending,
    registerError: registerMutation.error,
    logout: logoutMutation.mutate,
    isLoggingOut: logoutMutation.isPending,
  };
}
```

---

## Data Models & TypeScript

### Core Model Interfaces

**`src/types/models.ts`:**

```typescript
// ============================================
// BASE TYPES
// ============================================

export interface BaseEntity {
  id: string;
  created_at: string;
  updated_at: string;
}

// ============================================
// TEACHER
// ============================================

export interface Teacher extends BaseEntity {
  first_name: string;
  last_name: string;
  nickname: string | null;
  email: string;
  phone: string | null;
  newsletter_subscribed: boolean;
  profile_image_url: string | null;
}

export interface CreateTeacherDto {
  first_name: string;
  last_name: string;
  email: string;
  password: string;
  phone?: string;
  newsletter_subscribed?: boolean;
}

export interface UpdateTeacherDto {
  first_name?: string;
  last_name?: string;
  nickname?: string;
  phone?: string;
  newsletter_subscribed?: boolean;
}

// ============================================
// STUDENT
// ============================================

export interface Student extends BaseEntity {
  teacher_id: string;
  first_name: string;
  last_name: string;
  nickname: string | null;
  date_of_birth: string | null;
  grade_level: string | null;
  profile_image_url: string | null;
  notes: string | null;
  is_active: boolean;
}

export interface CreateStudentDto {
  first_name: string;
  last_name: string;
  nickname?: string;
  date_of_birth?: string;
  grade_level?: string;
  notes?: string;
}

export interface UpdateStudentDto extends Partial<CreateStudentDto> {}

// ============================================
// CALENDAR EVENT
// ============================================

export interface CalendarEvent extends BaseEntity {
  teacher_id: string;
  event_type_id: string | null;
  event_type_name: string | null;
  title: string;
  description: string | null;
  location: string | null;
  start_time: string;
  end_time: string | null;
  all_day: boolean;
  is_recurring: boolean;
  recurrence_rule: string | null;
  recurrence_end_date: string | null;
  parent_event_id: string | null;
  color_code: string | null;
  reminder_minutes: number | null;
  attendees: EventAttendee[];
}

export interface EventAttendee {
  student_id: string;
  student_name: string;
  attendance_status: AttendanceStatus;
}

export type AttendanceStatus = 'pending' | 'attended' | 'absent' | 'excused';

// ============================================
// ASSIGNMENT
// ============================================

export interface Assignment extends BaseEntity {
  student_id: string;
  student_name: string;
  teacher_id: string;
  subject_id: string | null;
  subject_name: string | null;
  calendar_event_id: string | null;
  title: string;
  description: string | null;
  due_date: string | null;
  assigned_date: string;
  completion_status: CompletionStatus;
  completed_date: string | null;
  grade: string | null;
  points_earned: number | null;
  points_possible: number | null;
  notes: string | null;
  attachments: Attachment[];
}

export type CompletionStatus = 'not_started' | 'in_progress' | 'completed' | 'overdue';

export interface Attachment {
  name: string;
  url: string;
  type: string;
  size: number;
}

// ============================================
// TASK
// ============================================

export interface Task extends BaseEntity {
  teacher_id: string;
  student_id: string | null;
  student_name: string | null;
  title: string;
  description: string | null;
  due_date: string | null;
  priority: TaskPriority;
  status: TaskStatus;
  completed_date: string | null;
  category: string | null;
}

export type TaskPriority = 'low' | 'medium' | 'high';
export type TaskStatus = 'pending' | 'in_progress' | 'completed' | 'cancelled';

// ============================================
// REPORT CARD
// ============================================

export interface ReportCard extends BaseEntity {
  student_id: string;
  student_name: string;
  teacher_id: string;
  title: string;
  period_type: PeriodType;
  start_date: string;
  end_date: string;
  grading_system: GradingSystem;
  status: ReportCardStatus;
  overall_comments: string | null;
  published_at: string | null;
  entries: ReportCardEntry[];
}

export type PeriodType = 'weekly' | 'monthly' | 'quarterly' | 'semester' | 'annual' | 'custom';
export type GradingSystem = 'letter' | 'percentage' | 'standards';
export type ReportCardStatus = 'draft' | 'finalized' | 'published';

export interface ReportCardEntry extends BaseEntity {
  report_card_id: string;
  subject_id: string | null;
  subject_name: string;
  letter_grade: string | null;
  percentage_grade: number | null;
  standards_rating: string | null;
  comments: string | null;
}

// ============================================
// EXPENSE
// ============================================

export interface Expense extends BaseEntity {
  teacher_id: string;
  student_id: string | null;
  student_name: string | null;
  subject_id: string | null;
  subject_name: string | null;
  category_id: string | null;
  category_name: string | null;
  expense_date: string;
  amount: number;
  currency: string;
  vendor: string | null;
  description: string;
  receipt_url: string | null;
  payment_method: string | null;
  is_tax_deductible: boolean;
  notes: string | null;
  tags: string[];
}

// ============================================
// SUBJECT
// ============================================

export interface Subject extends BaseEntity {
  teacher_id: string;
  name: string;
  description: string | null;
  color_code: string | null;
  icon: string | null;
}

// ============================================
// LESSON PLAN
// ============================================

export interface LessonPlan extends BaseEntity {
  teacher_id: string;
  teacher_name: string;
  subject_id: string | null;
  subject_name: string | null;
  title: string;
  description: string | null;
  content: string | null;
  grade_level: string | null;
  duration_minutes: number | null;
  objectives: string | null;
  materials_needed: string | null;
  activities: LessonActivity[] | null;
  assessment_methods: string | null;
  notes: string | null;
  attachments: Attachment[];
  is_template: boolean;
  visibility: LessonPlanVisibility;
  view_count: number;
}

export type LessonPlanVisibility = 'private' | 'public';

export interface LessonActivity {
  name: string;
  duration: number;
  description: string;
}
```

### API Response Types

**`src/types/api.ts`:**

```typescript
// Standard API response wrapper
export interface ApiResponse<T> {
  success: boolean;
  data: T;
}

// Paginated response
export interface PaginatedResponse<T> {
  success: boolean;
  data: T[];
  meta: PaginationMeta;
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  total_pages: number;
  has_next: boolean;
  has_prev: boolean;
}

// Error response
export interface ApiErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, string[]>;
  };
}

// Auth specific responses
export interface LoginResponse {
  teacher: Teacher;
  tokens: TokenPair;
}

export interface TokenPair {
  access_token: string;
  refresh_token: string;
  expires_in: number;
  token_type: string;
}
```

---

## Component Architecture

### Component Categories

#### 1. Common Components (Generic, Reusable)
Pure UI components with no business logic.

**Example: Button Component (`src/components/common/Button.tsx`):**

```typescript
import React from 'react';
import { 
  TouchableOpacity, 
  Text, 
  ActivityIndicator,
  TouchableOpacityProps,
} from 'react-native';
import { styled } from 'nativewind';

type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps extends TouchableOpacityProps {
  title: string;
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

const StyledTouchable = styled(TouchableOpacity);
const StyledText = styled(Text);

export function Button({
  title,
  variant = 'primary',
  size = 'md',
  isLoading = false,
  leftIcon,
  rightIcon,
  disabled,
  ...props
}: ButtonProps) {
  const isDisabled = disabled || isLoading;
  
  const containerStyles = {
    primary: 'bg-blue-600 active:bg-blue-700',
    secondary: 'bg-gray-600 active:bg-gray-700',
    outline: 'bg-transparent border-2 border-blue-600 active:bg-blue-50',
    ghost: 'bg-transparent active:bg-gray-100',
    danger: 'bg-red-600 active:bg-red-700',
  };
  
  const textStyles = {
    primary: 'text-white',
    secondary: 'text-white',
    outline: 'text-blue-600',
    ghost: 'text-blue-600',
    danger: 'text-white',
  };
  
  const sizeStyles = {
    sm: 'py-2 px-4',
    md: 'py-3 px-6',
    lg: 'py-4 px-8',
  };
  
  return (
    <StyledTouchable
      className={`
        flex-row items-center justify-center rounded-lg
        ${containerStyles[variant]}
        ${sizeStyles[size]}
        ${isDisabled ? 'opacity-50' : ''}
      `}
      disabled={isDisabled}
      {...props}
    >
      {isLoading ? (
        <ActivityIndicator 
          color={variant === 'outline' || variant === 'ghost' ? '#2563EB' : '#FFFFFF'} 
        />
      ) : (
        <>
          {leftIcon && <>{leftIcon}</>}
          <StyledText 
            className={`
              font-semibold
              ${textStyles[variant]}
            `}
          >
            {title}
          </StyledText>
          {rightIcon && <>{rightIcon}</>}
        </>
      )}
    </StyledTouchable>
  );
}
```

#### 2. Feature Components (Domain-Specific)

**Example: StudentCard (`src/components/features/students/StudentCard.tsx`):**

```typescript
import React from 'react';
import { View, Text, TouchableOpacity } from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { styled } from 'nativewind';
import { Student } from '@/types/models';
import { Avatar } from '@/components/common/Avatar';

const StyledView = styled(View);
const StyledText = styled(Text);
const StyledTouchable = styled(TouchableOpacity);

interface StudentCardProps {
  student: Student;
  onPress?: () => void;
}

export function StudentCard({ student, onPress }: StudentCardProps) {
  const navigation = useNavigation();
  
  const handlePress = () => {
    if (onPress) {
      onPress();
    } else {
      navigation.navigate('StudentDetail', { studentId: student.id });
    }
  };
  
  const displayName = student.nickname || `${student.first_name} ${student.last_name}`;
  
  return (
    <StyledTouchable
      className="bg-white rounded-xl p-4 mb-3 shadow-sm border border-gray-100 flex-row items-center"
      onPress={handlePress}
      activeOpacity={0.7}
    >
      <Avatar
        uri={student.profile_image_url}
        name={`${student.first_name} ${student.last_name}`}
        size={56}
      />
      
      <StyledView className="flex-1 ml-4">
        <StyledText className="text-lg font-semibold text-gray-900">
          {displayName}
        </StyledText>
        
        {student.grade_level && (
          <StyledView className="bg-blue-100 rounded-full px-2 py-0.5 mt-1 self-start">
            <StyledText className="text-xs font-medium text-blue-800">
              {student.grade_level}
            </StyledText>
          </StyledView>
        )}
      </StyledView>
      
      <StyledText className="text-gray-400 text-xl">›</StyledText>
    </StyledTouchable>
  );
}
```

#### 3. Screen Components (Page-Level)

**Example: StudentsListScreen (`src/screens/students/StudentsListScreen.tsx`):**

```typescript
import React, { useState } from 'react';
import { View, FlatList, RefreshControl } from 'react-native';
import { styled } from 'nativewind';
import { useStudents } from '@/hooks/api/useStudents';
import { ScreenContainer } from '@/components/layout/ScreenContainer';
import { Header } from '@/components/layout/Header';
import { StudentCard } from '@/components/features/students/StudentCard';
import { EmptyState } from '@/components/common/EmptyState';
import { LoadingSpinner } from '@/components/common/LoadingSpinner';
import { ErrorMessage } from '@/components/common/ErrorMessage';
import { FloatingActionButton } from '@/components/common/FloatingActionButton';
import { SearchBar } from '@/components/common/SearchBar';

const StyledView = styled(View);

export function StudentsListScreen({ navigation }) {
  const [searchQuery, setSearchQuery] = useState('');
  
  const { 
    students, 
    isLoading, 
    isError, 
    error, 
    refetch, 
    isRefetching 
  } = useStudents();
  
  // Filter students by search query
  const filteredStudents = students?.filter(student => {
    const fullName = `${student.first_name} ${student.last_name}`.toLowerCase();
    return fullName.includes(searchQuery.toLowerCase());
  });
  
  const handleAddStudent = () => {
    navigation.navigate('StudentForm', {});
  };
  
  if (isLoading) {
    return <LoadingSpinner />;
  }
  
  if (isError) {
    return (
      <ErrorMessage 
        message={error?.message || 'Failed to load students'} 
        onRetry={refetch}
      />
    );
  }
  
  return (
    <ScreenContainer>
      <Header title="Students" />
      
      <StyledView className="px-4 py-2">
        <SearchBar
          value={searchQuery}
          onChangeText={setSearchQuery}
          placeholder="Search students..."
        />
      </StyledView>
      
      <FlatList
        data={filteredStudents}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <StudentCard student={item} />}
        contentContainerStyle={{ padding: 16 }}
        refreshControl={
          <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
        }
        ListEmptyComponent={
          <EmptyState
            title="No students yet"
            message="Add your first student to get started"
            actionLabel="Add Student"
            onAction={handleAddStudent}
          />
        }
      />
      
      <FloatingActionButton onPress={handleAddStudent} />
    </ScreenContainer>
  );
}
```

---

## Offline Support Strategy

### Offline-First Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        USER ACTION                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Check Network       │
              │   Status              │
              └───────────┬───────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
            ▼                           ▼
     ┌─────────────┐           ┌─────────────────┐
     │   Online    │           │    Offline      │
     └──────┬──────┘           └────────┬────────┘
            │                           │
            ▼                           ▼
     ┌─────────────────┐       ┌─────────────────┐
     │ Execute API     │       │ Queue Action    │
     │ Call            │       │ for Later       │
     └────────┬────────┘       └────────┬────────┘
              │                         │
              ▼                         ▼
     ┌─────────────────┐       ┌─────────────────┐
     │ Update React    │       │ Apply Optimistic│
     │ Query Cache     │       │ Update to Cache │
     └─────────────────┘       └─────────────────┘
                                        │
                                        ▼
                               ┌─────────────────┐
                               │ When Online:    │
                               │ Process Queue   │
                               └─────────────────┘
```

### Offline Queue Store

**`src/store/offlineStore.ts`:**

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface QueuedAction {
  id: string;
  type: 'create' | 'update' | 'delete';
  entity: string;
  payload: any;
  timestamp: number;
  retryCount: number;
}

interface OfflineState {
  queue: QueuedAction[];
  isOnline: boolean;
  isSyncing: boolean;
  lastSyncedAt: number | null;
  
  // Actions
  addToQueue: (action: Omit<QueuedAction, 'id' | 'timestamp' | 'retryCount'>) => void;
  removeFromQueue: (id: string) => void;
  clearQueue: () => void;
  setOnlineStatus: (isOnline: boolean) => void;
  setSyncing: (isSyncing: boolean) => void;
  setLastSyncedAt: (timestamp: number) => void;
}

export const useOfflineStore = create<OfflineState>()(
  persist(
    (set) => ({
      queue: [],
      isOnline: true,
      isSyncing: false,
      lastSyncedAt: null,
      
      addToQueue: (action) => {
        const queuedAction: QueuedAction = {
          ...action,
          id: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
          timestamp: Date.now(),
          retryCount: 0,
        };
        
        set((state) => ({
          queue: [...state.queue, queuedAction],
        }));
      },
      
      removeFromQueue: (id) => {
        set((state) => ({
          queue: state.queue.filter((item) => item.id !== id),
        }));
      },
      
      clearQueue: () => set({ queue: [] }),
      
      setOnlineStatus: (isOnline) => set({ isOnline }),
      
      setSyncing: (isSyncing) => set({ isSyncing }),
      
      setLastSyncedAt: (timestamp) => set({ lastSyncedAt: timestamp }),
    }),
    {
      name: 'offline-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

---

## Security Implementation

### Secure Storage Strategy

| Data Type | Storage Method | Purpose |
|-----------|----------------|---------|
| Access Token | expo-secure-store | Short-lived auth |
| Refresh Token | expo-secure-store | Token renewal |
| User Preferences | AsyncStorage | App settings |
| Cached API Data | React Query (in-memory) | Performance |
| Offline Queue | AsyncStorage | Pending syncs |

### Biometric Authentication Hook

**`src/hooks/common/useBiometrics.ts`:**

```typescript
import { useState, useEffect } from 'react';
import * as LocalAuthentication from 'expo-local-authentication';

export function useBiometrics() {
  const [isAvailable, setIsAvailable] = useState(false);
  const [biometricType, setBiometricType] = useState<'fingerprint' | 'facial' | null>(null);
  
  useEffect(() => {
    checkBiometricAvailability();
  }, []);
  
  const checkBiometricAvailability = async () => {
    const compatible = await LocalAuthentication.hasHardwareAsync();
    const enrolled = await LocalAuthentication.isEnrolledAsync();
    const types = await LocalAuthentication.supportedAuthenticationTypesAsync();
    
    setIsAvailable(compatible && enrolled);
    
    if (types.includes(LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION)) {
      setBiometricType('facial');
    } else if (types.includes(LocalAuthentication.AuthenticationType.FINGERPRINT)) {
      setBiometricType('fingerprint');
    }
  };
  
  const authenticate = async (promptMessage = 'Authenticate to continue'): Promise<boolean> => {
    if (!isAvailable) return false;
    
    const result = await LocalAuthentication.authenticateAsync({
      promptMessage,
      fallbackLabel: 'Use Passcode',
      cancelLabel: 'Cancel',
      disableDeviceFallback: false,
    });
    
    return result.success;
  };
  
  return {
    isAvailable,
    biometricType,
    authenticate,
  };
}
```

---

## Error Handling

### Global Error Boundary

**`src/components/ErrorBoundary.tsx`:**

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, TouchableOpacity } from 'react-native';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
  }
  
  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError) {
      return (
        <View className="flex-1 items-center justify-center p-6 bg-white">
          <Text className="text-xl font-bold text-gray-900 mb-2">
            Oops! Something went wrong
          </Text>
          <Text className="text-gray-600 text-center mb-6">
            We're sorry for the inconvenience. Please try again.
          </Text>
          <TouchableOpacity
            className="bg-blue-600 rounded-lg px-6 py-3"
            onPress={this.handleReset}
          >
            <Text className="text-white font-semibold">Try Again</Text>
          </TouchableOpacity>
        </View>
      );
    }
    
    return this.props.children;
  }
}
```

---

## Testing Strategy

### Testing Pyramid

```
        ┌─────────────┐
        │     E2E     │  ← Few (critical user flows)
        │   (Detox)   │
        ├─────────────┤
        │ Integration │  ← Some (hooks, navigation)
        │   Tests     │
        ├─────────────┤
        │    Unit     │  ← Many (utils, components)
        │   Tests     │
        └─────────────┘
```

### Example Unit Test

**`__tests__/utils/formatting.test.ts`:**

```typescript
import { formatCurrency, formatDate } from '@/utils/formatting';

describe('formatting utilities', () => {
  describe('formatCurrency', () => {
    it('formats USD correctly', () => {
      expect(formatCurrency(1234.56)).toBe('$1,234.56');
    });
    
    it('handles zero', () => {
      expect(formatCurrency(0)).toBe('$0.00');
    });
  });
  
  describe('formatDate', () => {
    it('formats date correctly', () => {
      expect(formatDate('2025-11-14')).toBe('Nov 14, 2025');
    });
  });
});
```

---

## Performance Optimization

### Key Optimizations

1. **List Virtualization** - Use `FlatList` with proper configuration
2. **Image Optimization** - Use `expo-image` for caching
3. **Memoization** - Use `React.memo`, `useMemo`, `useCallback`
4. **Bundle Size** - Tree-shaking, lazy loading

### Example: Optimized FlatList

```typescript
import React, { useCallback } from 'react';
import { FlatList, ListRenderItem } from 'react-native';
import { Student } from '@/types/models';
import { StudentCard } from './StudentCard';

const ITEM_HEIGHT = 80;

export const StudentList = React.memo(function StudentList({
  students,
  onRefresh,
  isRefreshing,
}: {
  students: Student[];
  onRefresh: () => void;
  isRefreshing: boolean;
}) {
  const renderItem: ListRenderItem<Student> = useCallback(
    ({ item }) => <StudentCard student={item} />,
    []
  );
  
  const keyExtractor = useCallback((item: Student) => item.id, []);
  
  const getItemLayout = useCallback(
    (_: any, index: number) => ({
      length: ITEM_HEIGHT,
      offset: ITEM_HEIGHT * index,
      index,
    }),
    []
  );
  
  return (
    <FlatList
      data={students}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      getItemLayout={getItemLayout}
      refreshing={isRefreshing}
      onRefresh={onRefresh}
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={5}
      initialNumToRender={10}
    />
  );
});
```

---

## Future Web Migration

### Code Sharing Strategy

When building the React web version, the following can be shared:

#### Directly Shareable (70-80%)

| Category | Files | Reusability |
|----------|-------|-------------|
| TypeScript Types | `src/types/*` | 100% |
| API Layer | `src/api/*` | 100% |
| React Query Hooks | `src/hooks/api/*` | 100% |
| Zustand Stores | `src/store/*` | 95% |
| Utilities | `src/utils/*` | 90% |
| Validation Schemas | `src/utils/validation.ts` | 100% |

#### Needs Rewriting

| Category | Mobile | Web |
|----------|--------|-----|
| Components | React Native | React DOM |
| Navigation | React Navigation | React Router |
| Storage | SecureStore | Cookies/localStorage |
| Styling | NativeWind | Tailwind CSS |

### Recommended Web Project Structure

```
homeschool-web/
├── src/
│   ├── api/                    # COPY from mobile
│   ├── hooks/api/              # COPY from mobile
│   ├── store/                  # COPY (adapt storage)
│   ├── types/                  # COPY from mobile
│   ├── utils/                  # COPY from mobile
│   ├── components/             # REWRITE for web
│   ├── pages/                  # REWRITE (web routing)
│   └── routes/                 # React Router config
├── public/
├── package.json
└── tailwind.config.js
```

---

## Development Workflow

### Environment Setup

**Required Tools:**
- Node.js 18+ (LTS)
- Expo CLI (`npm install -g expo-cli`)
- Xcode (for iOS, macOS only)
- Android Studio (for Android)
- VS Code with ESLint, Prettier, Tailwind IntelliSense

### Getting Started

```bash
# Clone and install
git clone https://github.com/yourusername/homeschool-mobile.git
cd homeschool-mobile
npm install

# Start development
npx expo start

# Run on simulators
npx expo run:ios
npx expo run:android
```

### Git Workflow

```
main (production)
  └── develop (staging)
        ├── feature/add-student-screen
        ├── fix/login-bug
        └── chore/update-dependencies
```

### Key Dependencies

```json
{
  "dependencies": {
    "expo": "~50.0.0",
    "react": "18.2.0",
    "react-native": "0.73.0",
    "@react-navigation/native": "^6.1.0",
    "@tanstack/react-query": "^5.0.0",
    "zustand": "^4.4.0",
    "axios": "^1.6.0",
    "nativewind": "^2.0.0",
    "react-hook-form": "^7.48.0",
    "zod": "^3.22.0"
  }
}
```

---

## Document Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-30 | Initial mobile app architecture document |

---

## Next Steps

1. **Review and approve** architecture document
2. **Set up development environment** (Expo, simulators)
3. **Initialize project** with recommended structure
4. **Implement core infrastructure** (API client, auth, navigation)
5. **Build authentication flow** (register, login, logout)
6. **Create common components** (Button, Input, Card, etc.)
7. **Implement first feature** (Students CRUD)
8. **Add remaining features** incrementally
9. **Implement offline support**
10. **Testing and optimization**
11. **Prepare for app store submission**
