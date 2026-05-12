# MedConnect — Engineering Issue Report & Solutions

**Project:** MedConnect (Full-Stack Healthcare Platform)  
**Stack:** Django · React · TypeScript · PostgreSQL · Docker · Nginx  
**Report type:** Comprehensive code review — issues, root causes, and feasible fixes  
**Author:** Engineering review (Claude)  
**Scope:** Backend, frontend, database, auth, DevOps

---

## How to Read This Document

Each issue is structured as:
- **What it is** — plain description
- **Where it lives** — exact file and line
- **Why it matters** — real-world impact
- **Root cause** — what actually went wrong
- **Solution** — working code you can apply

Issues are ordered by severity: 🔴 Critical → ⚠️ High → 📋 Medium → 💡 Low

---

# Part 1 — Security Issues

---

## ISSUE-01 🔴 CSRF Protection Disabled on All API Endpoints

### What it is
Every API endpoint in the backend is decorated with `@csrf_exempt`, which completely disables Django's Cross-Site Request Forgery protection.

### Where it lives
```
BACKEND/medconnect_app/api_views.py — every function
BACKEND/MEDCONNECT/settings.py — line: CSRF_COOKIE_HTTPONLY = True
```

### Why it matters
CSRF is the attack where a malicious website makes your logged-in user perform actions they didn't intend. With `@csrf_exempt` on every endpoint, an attacker can build a page that silently:
- Deletes a patient's medical records
- Sends contact requests on a researcher's behalf
- Updates appointment data
- Posts community content as the user

This is a critical vulnerability for any application handling medical data.

### Root cause
`CSRF_COOKIE_HTTPONLY = True` was set in `settings.py`, which prevents JavaScript from reading the CSRF cookie. When the React frontend tried to send the token, it couldn't find it, causing 403 errors. The workaround was to disable CSRF entirely — which fixes the 403 errors but removes the protection completely.

### Solution

**Step 1 — Fix the root cause in `settings.py`:**
```python
# settings.py
CSRF_COOKIE_HTTPONLY = False   # JS must be able to read this cookie
SESSION_COOKIE_HTTPONLY = True  # Session cookie stays JS-inaccessible (correct)
```

**Step 2 — Create a centralized API utility in the frontend:**
```typescript
// src/lib/api.ts
import { API_BASE } from './config';

function getCsrfToken(): string {
  return (
    document.cookie
      .split('; ')
      .find(row => row.startsWith('csrftoken='))
      ?.split('=')[1] ?? ''
  );
}

export async function apiRequest(
  path: string,
  options: RequestInit = {}
): Promise<Response> {
  const method = (options.method ?? 'GET').toUpperCase();
  const isWrite = ['POST', 'PUT', 'PATCH', 'DELETE'].includes(method);

  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      ...(isWrite ? { 'X-CSRFToken': getCsrfToken() } : {}),
      ...options.headers,
    },
  });

  return response;
}
```

**Step 3 — Remove `@csrf_exempt` from all state-changing views:**
```python
# api_views.py — BEFORE (wrong)
@csrf_exempt
@require_auth
@require_http_methods(["POST"])
def api_create_appointment(request):
    ...

# AFTER (correct) — remove @csrf_exempt entirely
@require_auth
@require_http_methods(["POST"])
def api_create_appointment(request):
    ...
```

Keep `@csrf_exempt` ONLY on endpoints that don't require a session (login/register request-code endpoints), since the user has no cookie yet at that point.

**Step 4 — Replace all `fetch()` calls in the frontend with `apiRequest()`:**
```typescript
// Before
const response = await fetch(`${API_BASE}/api/appointments/create/`, {
  method: 'POST',
  credentials: 'include',
  body: JSON.stringify(data)
});

// After
const response = await apiRequest('/api/appointments/create/', {
  method: 'POST',
  body: JSON.stringify(data)
});
```

---

## ISSUE-02 🔴 Internal Exception Messages Leaked to the Client

### What it is
Every `except` block in `api_views.py` returns the raw Python exception as the error message to the browser.

### Where it lives
```
BACKEND/medconnect_app/api_views.py — every catch-all handler
```

### Why it matters
Raw exceptions expose:
- Database table and column names (`relation "medconnect_app_profile" does not exist`)
- File system paths (`/home/claude/BACKEND/medconnect_app/views.py line 47`)
- Internal logic and variable names
- Installed Python packages and their versions

This is classified as information disclosure — a real attack vector that makes it easier to plan more targeted attacks.

### Root cause
Convenience during development. The raw exception message is the fastest way to debug, but was never replaced before moving toward production.

### Solution

```python
# api_views.py — add at the top
import logging
logger = logging.getLogger(__name__)

# Every catch-all handler — BEFORE (wrong)
except Exception as e:
    return JsonResponse({
        'success': False,
        'message': str(e)   # leaks internal details
    }, status=500)

# AFTER (correct)
except Exception as e:
    logger.exception("Unhandled error in api_create_appointment")
    return JsonResponse({
        'success': False,
        'message': 'An unexpected error occurred. Please try again.'
    }, status=500)
```

Add logging configuration to `settings.py` so errors are written to a file or stdout:
```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
    'loggers': {
        'medconnect_app': {
            'handlers': ['console'],
            'level': 'DEBUG' if DEBUG else 'ERROR',
            'propagate': False,
        },
    },
}
```

---

## ISSUE-03 🔴 Plaintext Password Logged to stdout

### What it is
The login view logs the user's plaintext password to the console.

### Where it lives
```
BACKEND/medconnect_app/views.py — lines ~42-43
```

### Why it matters
Docker logs are often forwarded to log aggregation systems (Datadog, CloudWatch, Loki). Anyone with access to those logs — including future team members, DevOps engineers, or an attacker who gains server access — can read every password ever typed into the login form.

### Root cause
Debug print statements left in from development.

### Solution
Delete these lines entirely:
```python
# DELETE these lines from views.py
print(f"DEBUG: Login attempt - Username/Email: '{username_or_email}', Password: '{password}'")
print(f"DEBUG: Username authentication result: {user}")
print(f"DEBUG: Email authentication result: {user}")
# ... and all other print() statements
```

If you need login debugging, use the logger at DEBUG level (which is disabled in production):
```python
logger.debug("Login attempt for: %s", username_or_email)
# Never log the password, not even at DEBUG level
```

---

## ISSUE-04 ⚠️ Debug Endpoint Exposed in Production URL Configuration

### What it is
A `/api/debug/users/` endpoint is registered in the URL configuration with no access restriction.

### Where it lives
```
BACKEND/medconnect_app/urls.py — line:
path('api/debug/users/', api_views.api_debug_users, name='api_debug_users'),
```

### Why it matters
If this endpoint returns user data (email addresses, profile information), it's a data breach waiting to happen. It's accessible to anyone who knows the URL — no authentication required.

### Solution
Gate it behind `settings.DEBUG` so it never exists in production:
```python
# medconnect_app/urls.py
from django.conf import settings

urlpatterns = [
    # ... all regular patterns
]

if settings.DEBUG:
    urlpatterns += [
        path('api/debug/users/', api_views.api_debug_users, name='api_debug_users'),
    ]
```

---

# Part 2 — Authentication Issues

---

## ISSUE-05 🔴 localStorage + Session Cookie Race Condition

### What it is
`AuthContext` stores the user in `localStorage` for instant access on page load, but the actual authentication uses session cookies. The session verification happens asynchronously, but `loading` is set to `false` before the verification completes.

### Where it lives
```
MedConnect-main/src/contexts/AuthContext.tsx — useEffect (lines ~45-65)
```

### Why it matters
Timeline of the bug:

```
1. Page loads
2. localStorage has user → setUser(userData) ✓
3. setLoading(false) ← loading is DONE
4. ProtectedRoute: isAuthenticated=true → renders Dashboard ✓ (looks correct)
5. verifySession() runs (still async, in background)
6. Session cookie expired → should redirect to login
7. But Dashboard is already rendered...
8. All 11 DataContext API calls return 401
9. Page appears logged in but nothing loads
10. User is confused, has to manually refresh or clear storage
```

### Root cause
`setLoading(false)` was placed outside the async verification function, so it completes before the network call finishes.

### Solution
Restructure the `useEffect` to keep `loading: true` until verification is complete:

```typescript
// AuthContext.tsx — replace the existing useEffect
useEffect(() => {
  const initAuth = async () => {
    const stored = localStorage.getItem('medconnect_user');

    if (!stored) {
      setLoading(false);
      return;
    }

    try {
      // Verify the session is still valid BEFORE setting loading to false
      const response = await fetch(`${API_BASE}/api/user/`, {
        credentials: 'include',
      });

      if (response.ok) {
        // Session valid — use stored user (or update from server response)
        setUser(JSON.parse(stored));
      } else if (response.status === 401) {
        // Session expired — clear stale data
        localStorage.removeItem('medconnect_user');
        setUser(null);
      }
      // 5xx or network error: keep the user logged in optimistically
    } catch {
      // Network failure — keep user logged in, retry will happen naturally
      setUser(JSON.parse(stored));
    } finally {
      setLoading(false); // Only now is the auth state settled
    }
  };

  initAuth();
}, []); // Empty array — only runs on mount
```

---

## ISSUE-06 🔴 `updateProfile` in AuthContext is a Non-Functional Mock

### What it is
`AuthContext.updateProfile()` simulates an API call with a `setTimeout` but never actually calls the backend.

### Where it lives
```
MedConnect-main/src/contexts/AuthContext.tsx — updateProfile function (~lines 210-222)
```

### Why it matters
Any component calling `AuthContext.updateProfile()` believes the save was successful. The user sees no error. But the backend was never called, so on next login the profile reverts to the old data.

### Root cause
This was likely a placeholder written during initial development and never completed. The real implementation already exists in `DataContext.updateProfile()`.

### Solution
Two-part fix:

**1. Remove the fake implementation from `AuthContext`:**
```typescript
// AuthContext.tsx — replace the fake updateProfile
const updateProfile = async (data: Partial<User>): Promise<void> => {
  if (!user) return;
  // Only sync the local display state (name in header, etc.)
  // Never call the API from here — that's DataContext's job
  const updated = { ...user, ...data };
  setUser(updated);
  localStorage.setItem('medconnect_user', JSON.stringify(updated));
};
```

**2. Ensure all profile save operations call `DataContext.updateProfile()`:**
```typescript
// ProfilePage.tsx — correct pattern
const { updateProfile: saveToBackend } = useData();    // real API call
const { updateProfile: syncDisplayState } = useAuth(); // local state only

const handleSave = async () => {
  const ok = await saveToBackend(profileData);   // hits /api/profile/update/
  if (ok) {
    await syncDisplayState({ firstName: profileData.first_name }); // updates header
  }
};
```

---

## ISSUE-07 ⚠️ Artificial Delays Masking Session Timing Problems

### What it is
Two `setTimeout` delays were added to work around session cookie timing issues.

### Where it lives
```
AuthContext.tsx — login(): await new Promise(resolve => setTimeout(resolve, 200))
DataContext.tsx — loadData(): await new Promise(resolve => setTimeout(resolve, 300))
```

### Why it matters
- Every login is 500ms slower with no benefit
- Race conditions aren't fixed by sleeping — they're fixed by making the code wait for the actual condition
- These will fail on slow connections (500ms isn't enough) and waste time on fast ones

### Root cause
The session wasn't verified before rendering protected content (ISSUE-05). The delays were a workaround instead of a fix.

### Solution
Fix ISSUE-05 (keep `loading:true` until verification completes). Once that's done, both `setTimeout` calls can be deleted entirely. No other change needed.

---

# Part 3 — Data Layer Issues

---

## ISSUE-08 🔴 DataContext `useEffect` Fires 11 Fetches on Every User Object Change

### What it is
`DataContext` fires 11 simultaneous API fetches whenever the `user` object changes. The `user` object is recreated on every `AuthContext` render, so this happens far more often than just "on login".

### Where it lives
```
MedConnect-main/src/contexts/DataContext.tsx — useEffect, line ~1100
```

### Why it matters
On login, `AuthContext` calls `setLoading(true)` then `setLoading(false)`. Both trigger a render. Each render produces a new `user` object reference. Each new reference triggers the `useEffect`. Result: potentially 20+ simultaneous network requests on login instead of 11.

### Root cause
`[user]` in the dependency array compares object references, not values. A new `setUser({...})` call always produces a new reference even if the user data is identical.

### Solution
Depend on the user ID (a primitive) instead of the user object:

```typescript
// DataContext.tsx — change the dependency
useEffect(() => {
  if (!user?.id) return;

  const loadData = async () => {
    // Remove the setTimeout and the pre-flight profile check
    // fetchProfile() below already does that check
    fetchStudies();
    fetchUserStudies();
    fetchCommunities();
    fetchUserCommunities();
    fetchProfile();
    fetchMedicalRecords();
    fetchVitalSigns();
    fetchMedications();
    fetchImmunizations();
    fetchAllergies();
    fetchContactRequests();
  };

  loadData();
}, [user?.id]); // Only re-runs when the actual user ID changes
```

---

## ISSUE-09 ⚠️ All DataContext Functions Silently Return `false` on Failure

### What it is
Every function in `DataContext` catches errors internally and returns `false`. The calling component receives `false` but has no message to show the user.

### Where it lives
```
MedConnect-main/src/contexts/DataContext.tsx — every function
```

### Why it matters
When `applyToStudy()` returns `false`, the user's submit button stops spinning and nothing else happens. They don't know if:
- The network failed
- They already applied
- The study is closed
- The server had an error

This makes the app feel broken even when the error is recoverable and actionable.

### Root cause
Defensive coding that prevents crashes but also prevents useful feedback.

### Solution
Let functions throw errors with the server's message. Components that call them are already wrapped in try/catch — they just need something to catch:

```typescript
// DataContext.tsx — BEFORE (wrong)
const applyToStudy = async (studyId: string): Promise<boolean> => {
  try {
    const response = await fetch(...);
    if (response.ok) {
      const data = await response.json();
      if (data.success) return true;
    }
    return false;
  } catch (error) {
    console.error('Failed:', error);
    return false;
  }
};

// AFTER (correct)
const applyToStudy = async (studyId: string): Promise<void> => {
  const response = await fetch(`${API_BASE}/api/studies/${studyId}/apply/`, {
    method: 'POST',
    credentials: 'include',
  });

  const data = await response.json().catch(() => ({}));

  if (!response.ok || !data.success) {
    throw new Error(data.message || 'Failed to apply to study');
  }

  await fetchUserStudies();
};

// Component — now has something to show
try {
  await applyToStudy(studyId);
  setSuccess('Application submitted successfully!');
} catch (err: any) {
  setError(err.message); // "You have already applied to this study"
}
```

---

## ISSUE-10 ⚠️ `API_BASE` Defined Identically in Three Files

### What it is
The backend URL derivation logic is copy-pasted in `AuthContext.tsx`, `DataContext.tsx`, and `ProfilePage.tsx`.

### Where it lives
```
MedConnect-main/src/contexts/AuthContext.tsx — line ~31
MedConnect-main/src/contexts/DataContext.tsx — line ~121
MedConnect-main/src/pages/ProfilePage.tsx — lines ~15-18
```

### Why it matters
If you ever change the dev port, the logic, or add environment handling, you have to change it in three places and risk them going out of sync.

### Root cause
No shared configuration module was created when the project started.

### Solution
Create one file, import everywhere:

```typescript
// src/lib/config.ts
export const API_BASE = (
  import.meta.env.VITE_API_BASE_URL ||
  (import.meta.env.DEV
    ? `${window.location.protocol}//${window.location.hostname}:8000`
    : '')
).replace(/\/+$/, '');
```

```typescript
// In AuthContext.tsx, DataContext.tsx, ProfilePage.tsx — replace the inline expression:
import { API_BASE } from '../lib/config';
```

This is also the natural home for a shared `apiRequest()` wrapper (see ISSUE-01), reducing all `fetch()` boilerplate to one definition.

---

## ISSUE-11 🔴 `ContactRequest` Interface Defined Twice with Different Fields

### What it is
`types/data.ts` exports `ContactRequest` twice. TypeScript uses the last definition, but components reference fields from the first.

### Where it lives
```
MedConnect-main/src/types/data.ts — ~line 140 (first definition) and ~line 175 (second definition)
```

### Why it matters
`NotificationCenter.tsx` accesses `request.researcherName` and `request.patientName`. These exist in the first definition but not the second (which uses `fromUserName`, `fromUserId`). TypeScript resolves to the second definition, so those fields are `undefined` at runtime — the notification center shows blank names.

### Root cause
Two separate drafts of the interface were written at different times and both kept.

### Solution
Delete the second definition. Keep and complete the first:

```typescript
// types/data.ts — keep only this one
export interface ContactRequest {
  id: string;
  type: 'sent' | 'received';
  researcherId?: string;
  researcherName?: string;
  patientId?: string;
  patientName?: string;
  message: string;
  status: 'pending' | 'accepted' | 'declined';
  createdAt: string;
  updatedAt?: string;
}
```

---

## ISSUE-12 ⚠️ Two Separate `User` Interfaces Across Type Files

### What it is
`types/auth.ts` and `types/data.ts` both export a `User` interface with different and incompatible fields.

### Where it lives
```
MedConnect-main/src/types/auth.ts — User interface
MedConnect-main/src/types/data.ts — User interface (different shape)
```

### Why it matters
`auth.ts/User` has `firstName`, `lastName`, `profileComplete`, `privacySettings`, `patientProfile`, `researcherProfile`. `data.ts/User` has `username`, `isAuthenticated` and is missing most of those fields. Components importing from the wrong file get type errors or silent `undefined` values.

### Solution
Delete `User` from `types/data.ts`. Use `auth.ts/User` everywhere. Update any import that referenced `data.ts/User`:

```typescript
// Before (wrong)
import { User } from '../types/data';

// After (correct)
import type { User } from '../types/auth';
```

---

# Part 4 — Database Issues

---

## ISSUE-13 ⚠️ `PatientProfile.allergies` Duplicates the `Allergy` Model

### What it is
`PatientProfile` has a free-text `allergies` field added in migration 0008. The same migration also creates a structured `Allergy` model with `allergen`, `reaction`, `severity`, and `onset_date`. The same data exists in two places.

### Where it lives
```
BACKEND/medconnect_app/models.py — PatientProfile.allergies (TextField)
BACKEND/medconnect_app/models.py — Allergy model
BACKEND/medconnect_app/migrations/0008_*.py — both added in same migration
```

### Why it matters
A patient's allergy entered as free text in their profile and a structured allergy record are separate data stores. They will diverge: the profile might say "Penicillin" while the Allergy table has no record, or vice versa. Any system that needs to check allergies (future drug interaction checks, researcher filtering) will get inconsistent data.

### Solution
Remove the free-text fields from `PatientProfile` and rely exclusively on the structured EHR models:

```python
# New migration
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('medconnect_app', '0013_passwordresetemailotp'),
    ]

    operations = [
        migrations.RemoveField(model_name='patientprofile', name='allergies'),
        migrations.RemoveField(model_name='patientprofile', name='medical_conditions'),
        migrations.RemoveField(model_name='patientprofile', name='family_history'),
    ]
```

Before running this migration, write a one-time data migration that parses the free-text fields and creates structured `Allergy` records for any existing data.

---

## ISSUE-14 ⚠️ 13 Migrations Including 4 Full Model Rebuilds

### What it is
Migrations 0001–0004 represent three complete schema rewrites: the app went from `Patient/Doctor` → `Researcher/ResearchStudy` → abandoned → `Profile/PatientProfile/ResearcherProfile`. All that history is still in the migration chain.

### Where it lives
```
BACKEND/medconnect_app/migrations/0001_initial.py through 0004_*.py
```

### Why it matters
- Every `docker compose up` + `migrate` on a fresh environment applies all 13 migrations including the model creates-and-deletes
- CI/CD pipelines run all migrations from scratch on test databases — this adds unnecessary time
- New developers reading the migration history get a misleading picture of the schema evolution

### Solution
Squash migrations 0001–0004 into a clean initial:

```bash
python manage.py squashmigrations medconnect_app 0001 0004
```

This generates a single file. Rename it `0001_initial.py`, delete the original four, and update the `replaces` list in the squashed file. Test with a fresh database to confirm `migrate` completes cleanly.

---

## ISSUE-15 📋 Missing Database Indexes on OTP Models

### What it is
`LoginEmailOTP`, `SignupEmailOTP`, and `PasswordResetEmailOTP` are queried by `(user, is_used, expires_at)` on every login and OTP verification, but there are no composite indexes on these fields.

### Where it lives
```
BACKEND/medconnect_app/migrations/0011_loginemailotp.py
BACKEND/medconnect_app/migrations/0012_signupemailotp.py
BACKEND/medconnect_app/migrations/0013_passwordresetemailotp.py
```

### Why it matters
At low user counts this is invisible. As users grow, every login triggers a sequential table scan on the OTP table. Since OTPs are created and expired constantly, this table grows quickly.

### Solution
Add composite indexes via a new migration:

```python
# models.py — add Meta classes
class LoginEmailOTP(models.Model):
    # ... existing fields ...
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['user', 'is_used', 'expires_at']),
        ]

class SignupEmailOTP(models.Model):
    # ... existing fields ...
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['email', 'is_used', 'expires_at']),
        ]
```

Then generate and apply the migration:
```bash
python manage.py makemigrations --name add_otp_indexes
python manage.py migrate
```

---

## ISSUE-16 📋 `signals.py` — `save_user_profile` Can Crash on User Creation

### What it is
Two signals fire on `User.post_save`. On a new user creation, their order is not guaranteed. If `save_user_profile` runs before `create_user_profile`, `instance.profile` raises `RelatedObjectDoesNotExist`.

### Where it lives
```
BACKEND/medconnect_app/signals.py
```

### Why it matters
Under specific Django versions or when signal order changes, new user registration crashes with an unhandled exception, leaving the user partially created.

### Solution
Remove `save_user_profile` entirely. The profile is created in `create_user_profile` and explicitly saved in registration views — the second signal adds nothing:

```python
# signals.py — keep only this one
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from .models import Profile

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance, role='patient')

# DELETE save_user_profile entirely
```

---

# Part 5 — Frontend Component Issues

---

## ISSUE-17 🔴 `MedConnectPlatform.tsx` is a Dead Duplicate with a Breaking Bug

### What it is
`pages/MedConnectPlatform.tsx` is a copy of `LandingPage.tsx` with a different color scheme, hardcoded SVG icons (instead of `lucide-react` imports), and a wrapping `<div id="root">` that nests React's root inside itself.

### Where it lives
```
MedConnect-main/src/pages/MedConnectPlatform.tsx
```

### Why it matters
If this component is ever rendered, it creates `<div id="root"><div id="root">...</div></div>` — a nested root. This breaks React's hydration model. The `<button>` elements have no handlers. Navigation doesn't work. The color theme is blue instead of the app's emerald.

### Solution
Delete the file:
```bash
rm MedConnect-main/src/pages/MedConnectPlatform.tsx
```

Check `App.tsx` — if it's registered as a route, replace it with `<LandingPage />`. The `.html` files in `components/pages/` are also static exports of this same component and can be deleted.

---

## ISSUE-18 🔴 Post Ownership Checked by Display Name, Not User ID

### What it is
`CommunityPost.tsx` determines whether to show edit/delete controls by comparing the post author's display name against the logged-in user's name.

### Where it lives
```
MedConnect-main/src/components/community/CommunityPost.tsx — line ~90
const isOwnPost = user && post.authorName === (user.fullName || user.username);
```

### Why it matters
Two users with the same full name would see edit and delete controls on each other's posts. This is not a theoretical edge case in a healthcare community — "John Smith" and "Maria Garcia" are common names. A patient could delete another patient's post.

### Root cause
The post data from the backend includes `author_name` (a display string) but not `author_id`. The frontend used what was available.

### Solution

**Backend** — add `author_id` to the post serializer response:
```python
# api_views.py — in the posts list response
post_data = {
    'id': post.id,
    'author_id': post.author.id,   # Add this field
    'author_name': post.author.user.get_full_name() or post.author.user.username,
    # ...
}
```

**Frontend** — compare IDs:
```typescript
// CommunityPost.tsx
// First, update the post type to include authorId
interface Post {
  id: string;
  authorId: string;   // Add this
  authorName: string;
  // ...
}

// Then fix the ownership check
const isOwnPost = !!user && post.authorId === user.id;
```

---

## ISSUE-19 ⚠️ `alert()` and `window.confirm()` Used for User Feedback

### What it is
Browser-native `alert()` dialogs are used for success/error feedback, and `window.confirm()` is used for delete confirmations across multiple components.

### Where it lives
```
MedConnect-main/src/components/notifications/NotificationCenter.tsx
MedConnect-main/src/components/notifications/ContactRequestModal.tsx
MedConnect-main/src/pages/ProfilePage.tsx
MedConnect-main/src/components/appointments/AppointmentModal.tsx
MedConnect-main/src/components/community/CommunityPost.tsx
```

### Why it matters
- Cannot be styled — they look like system dialogs, not part of your UI
- Block the JavaScript thread — the entire app freezes while they're open
- Cannot be auto-dismissed
- Look unprofessional and break trust for a healthcare platform

### Solution
Create one reusable inline feedback component:

```typescript
// src/components/ui/Alert.tsx
import React from 'react';
import { CheckCircle, AlertCircle, X } from 'lucide-react';

interface AlertProps {
  type: 'success' | 'error' | 'info';
  message: string;
  onDismiss?: () => void;
}

export const Alert: React.FC<AlertProps> = ({ type, message, onDismiss }) => {
  const config = {
    success: {
      bg: 'bg-green-50 border-green-200',
      text: 'text-green-800',
      Icon: CheckCircle,
    },
    error: {
      bg: 'bg-red-50 border-red-200',
      text: 'text-red-800',
      Icon: AlertCircle,
    },
    info: {
      bg: 'bg-blue-50 border-blue-200',
      text: 'text-blue-800',
      Icon: AlertCircle,
    },
  }[type];

  return (
    <div className={`flex items-start gap-3 p-3 rounded-md border ${config.bg}`}>
      <config.Icon className={`h-5 w-5 mt-0.5 flex-shrink-0 ${config.text}`} />
      <span className={`text-sm flex-1 ${config.text}`}>{message}</span>
      {onDismiss && (
        <button onClick={onDismiss} className={`${config.text} opacity-60 hover:opacity-100`}>
          <X className="h-4 w-4" />
        </button>
      )}
    </div>
  );
};
```

For delete confirmations, use inline confirm state instead of `window.confirm()`:

```typescript
// Instead of window.confirm(...)
const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);

// In JSX:
{showDeleteConfirm ? (
  <div className="flex items-center gap-2 p-2 bg-red-50 rounded-md">
    <span className="text-sm text-red-800">Delete this post?</span>
    <button
      onClick={() => { handleDelete(); setShowDeleteConfirm(false); }}
      className="text-sm font-medium text-red-600 hover:text-red-800"
    >
      Yes, delete
    </button>
    <button
      onClick={() => setShowDeleteConfirm(false)}
      className="text-sm text-gray-600 hover:text-gray-800"
    >
      Cancel
    </button>
  </div>
) : (
  <button onClick={() => setShowDeleteConfirm(true)} className="...">
    Delete
  </button>
)}
```

---

## ISSUE-20 ⚠️ `window.location.reload()` After Profile Picture Upload

### What it is
After uploading a profile picture, `ProfilePage.tsx` calls `window.location.reload()` to show the new image.

### Where it lives
```
MedConnect-main/src/pages/ProfilePage.tsx — line ~141
window.location.reload();
```

### Why it matters
A full page reload destroys all React state, re-runs all auth verification, re-fires all 11 DataContext fetches, and adds ~1–2 seconds of blank screen. It's unnecessary and makes the app feel slow.

### Solution
Expose a `refreshProfile` function from `DataContext` and call that instead:

```typescript
// DataContext.tsx — add this function
const refreshProfile = async () => {
  await fetchProfile();
};

// Add to context value:
// refreshProfile,

// ProfilePage.tsx — replace window.location.reload()
const { refreshProfile } = useData();

// In the upload handler, after success:
await refreshProfile(); // Updates profile state, re-renders avatar — no page reload
```

---

## ISSUE-21 📋 Re-fetching All Posts After Every Like and Comment

### What it is
`CommunityPage.tsx` calls `fetchCommunityPosts()` (a full network request for all posts) after every like, unlike, comment, post update, and post delete.

### Where it lives
```
MedConnect-main/src/pages/CommunityPage.tsx — handleLikePost, handleUnlikePost, handleAddComment, etc.
```

### Why it matters
In an active community with 50 posts, liking a post triggers a fetch of all 50 posts, their comments, and their attachments. With multiple users active simultaneously, this multiplies quickly.

### Solution
Update local state optimistically instead of re-fetching:

```typescript
// DataContext.tsx — optimistic like
const likePost = async (postId: string): Promise<void> => {
  // Update local state immediately (optimistic)
  setCommunityPosts(prev =>
    prev.map(p =>
      p.id === postId
        ? { ...p, is_liked: true, likes: [...p.likes, 'me'] }
        : p
    )
  );

  try {
    const response = await apiRequest(`/api/posts/${postId}/like/`, { method: 'POST' });
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.message);
  } catch (err) {
    // Revert on failure
    setCommunityPosts(prev =>
      prev.map(p =>
        p.id === postId
          ? { ...p, is_liked: false, likes: p.likes.slice(0, -1) }
          : p
      )
    );
    throw err;
  }
};
```

For new comments, append to the existing list rather than re-fetching everything.

---

## ISSUE-22 💡 Inconsistent Color Theme (Blue vs Emerald)

### What it is
The app uses two different primary color schemes. Core pages (Landing, Login, Header, Dashboard) use `emerald-600`. Component-level elements (modals, community posts, create post button) use `blue-600`.

### Where it lives
```
Components using blue-600:
- AppointmentModal.tsx
- CommunityPost.tsx
- CreatePost.tsx
- ContactRequestModal.tsx
- NotificationCenter.tsx

Components using emerald-600:
- LandingPage.tsx
- LoginPage.tsx
- Header.tsx
- Dashboard.tsx
- ProfilePage.tsx
```

### Why it matters
This looks like two different designers worked on different parts of the app. For a healthcare platform asking users to trust it with medical data, visual consistency is part of building that trust.

### Solution
Since the brand is clearly emerald (logo, landing page, header all use it), do a search-and-replace across the `blue-*` components:

```bash
# In the MedConnect-main/src directory:
# Replace bg-blue-600 → bg-emerald-600
# Replace hover:bg-blue-700 → hover:bg-emerald-700
# Replace text-blue-600 → text-emerald-600
# Replace ring-blue-500 → ring-emerald-500
# Replace border-blue-600 → border-emerald-600
# Replace focus:ring-blue-500 → focus:ring-emerald-500
```

Do this carefully — `blue` appears in non-Tailwind contexts too (SVG stroke colors, external links).

---

# Part 6 — DevOps and Infrastructure Issues

---

## ISSUE-23 ⚠️ No Vite Dev Proxy — CORS and Cookie Friction in Development

### What it is
In development, the React dev server runs on `localhost:5173` and the Django backend on `localhost:8000`. These are different origins, causing CORS preflight requests on every API call. Session cookies also behave differently across origins.

### Where it lives
```
MedConnect-main/vite.config.ts
```

### Why it matters
- CORS adds an extra preflight request to every API call in development (doubles the network requests)
- Session cookie `SameSite=Lax` can cause cookies to not be sent on cross-origin requests in some browsers
- The dev environment doesn't match the production environment (where Nginx proxies everything on the same origin)

### Solution
Add a Vite dev proxy so the frontend and backend appear to be on the same origin:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        credentials: true,
      },
      '/media': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
  preview: {
    port: 5173,
  },
});
```

Also add path aliases to `tsconfig.app.json`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## ISSUE-24 📋 No Media File Serving Strategy for Production

### What it is
In development, Django serves uploaded media files directly (profile pictures, medical record attachments, community post attachments). In production, this doesn't work — Django with Gunicorn is not designed to serve static or media files efficiently.

### Where it lives
```
BACKEND/MEDCONNECT/urls.py — static() only enabled when DEBUG=True
BACKEND/MEDCONNECT/settings.py — MEDIA_ROOT, MEDIA_URL
```

### Why it matters
Without a media serving strategy, uploaded files (profile pictures, medical records) will be inaccessible in production.

### Solution — Option A (simpler): Nginx serves media directly
```nginx
# MedConnect-main/nginx/default.conf — add to server block
location /media/ {
    alias /app/media/;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

In `docker-compose.prod.yml`, mount the same volume to both the backend and Nginx containers:
```yaml
services:
  backend:
    volumes:
      - media_files:/app/media

  nginx:
    volumes:
      - media_files:/app/media:ro  # read-only for Nginx

volumes:
  media_files:
```

---

## ISSUE-25 📋 No Test Coverage

### What it is
`tests.py` is empty. There is no automated test for any feature.

### Where it lives
```
BACKEND/medconnect_app/tests.py
```

### Why it matters
For a healthcare platform, regressions are particularly dangerous. A bug that lets patient A view patient B's medical records, or lets an unauthenticated user access any data, could be introduced silently and go undetected.

### Solution
Start with the highest-risk tests — access control and role enforcement:

```python
# tests.py
from django.test import TestCase, Client
from django.contrib.auth.models import User
from .models import Profile, PatientProfile, ResearcherProfile, MedicalRecord

class RoleEnforcementTests(TestCase):
    def setUp(self):
        self.client = Client()

        # Create patient
        self.patient_user = User.objects.create_user(
            username='patient1', email='patient@test.com', password='TestPass123!'
        )
        self.patient_profile = self.patient_user.profile
        self.patient_profile.role = 'patient'
        self.patient_profile.save()

        # Create researcher
        self.researcher_user = User.objects.create_user(
            username='researcher1', email='researcher@test.com', password='TestPass123!'
        )
        self.researcher_profile = self.researcher_user.profile
        self.researcher_profile.role = 'researcher'
        self.researcher_profile.save()

    def test_unauthenticated_cannot_access_medical_records(self):
        response = self.client.get('/api/medical-records/')
        self.assertEqual(response.status_code, 401)

    def test_patient_cannot_create_study(self):
        self.client.force_login(self.patient_user)
        response = self.client.post('/api/studies/create/', {
            'title': 'Test Study',
            'description': 'Test',
        }, content_type='application/json')
        self.assertIn(response.status_code, [403, 401])

    def test_patient_cannot_see_other_patients_medical_records(self):
        # Create second patient with a record
        other_patient = User.objects.create_user(
            username='patient2', email='patient2@test.com', password='TestPass123!'
        )
        record = MedicalRecord.objects.create(
            patient=other_patient.profile,
            record_type='lab_result',
            title='Private Record',
            description='Should not be visible',
            date='2024-01-01',
            provider='Test Hospital'
        )

        # Log in as first patient
        self.client.force_login(self.patient_user)
        response = self.client.get('/api/medical-records/')
        data = response.json()

        # Should return empty list, not the other patient's record
        record_ids = [r['id'] for r in data.get('records', [])]
        self.assertNotIn(str(record.id), record_ids)
```

---

# Summary — Issues by Priority

## Fix immediately (before any real users)

| ID | Issue | File |
|---|---|---|
| ISSUE-01 | CSRF disabled on all endpoints | `api_views.py`, `settings.py`, frontend |
| ISSUE-02 | Internal exceptions leaked to client | `api_views.py` |
| ISSUE-03 | Plaintext password logged | `views.py` |
| ISSUE-04 | Debug endpoint in production | `urls.py` |
| ISSUE-05 | Session/localStorage race condition | `AuthContext.tsx` |
| ISSUE-06 | `updateProfile` is a mock | `AuthContext.tsx` |
| ISSUE-11 | Duplicate `ContactRequest` interface | `types/data.ts` |
| ISSUE-17 | `MedConnectPlatform.tsx` dead duplicate | `pages/MedConnectPlatform.tsx` |
| ISSUE-18 | Post ownership by name string | `CommunityPost.tsx` |
| ISSUE-16 | Signal crash on user creation | `signals.py` |

## Fix before launch (before public deployment)

| ID | Issue | File |
|---|---|---|
| ISSUE-07 | Artificial login delays | `AuthContext.tsx`, `DataContext.tsx` |
| ISSUE-08 | 11 fetches on every user render | `DataContext.tsx` |
| ISSUE-09 | Errors silently swallowed | `DataContext.tsx` |
| ISSUE-10 | `API_BASE` in three files | All context files |
| ISSUE-12 | Two `User` interfaces | `types/data.ts` |
| ISSUE-13 | Duplicate allergies field | `models.py`, migration needed |
| ISSUE-19 | `alert()` for user feedback | Multiple components |
| ISSUE-20 | `window.location.reload()` | `ProfilePage.tsx` |
| ISSUE-23 | No Vite dev proxy | `vite.config.ts` |
| ISSUE-24 | No media serving in production | `nginx/default.conf`, `docker-compose.prod.yml` |

## Fix when you have time

| ID | Issue | File |
|---|---|---|
| ISSUE-14 | 13 migrations with 4 rebuilds | Migrations folder |
| ISSUE-15 | Missing OTP indexes | `models.py`, new migration |
| ISSUE-21 | Re-fetch on every like/comment | `CommunityPage.tsx`, `DataContext.tsx` |
| ISSUE-22 | Blue vs emerald color inconsistency | Multiple component files |
| ISSUE-25 | No test coverage | `tests.py` |

---

*End of MedConnect Engineering Issue Report*
