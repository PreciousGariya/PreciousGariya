
---
## Project: Talentaimer – College Management System (LMS)

**Tech Stack:** Laravel (Multitenancy via `sactle/laravel`), Vue 3 with Inertia.js (Student Web), Filament Admin Panel, React Native (Student App), Tailwind CSS, Vite

**UI Library for Vuejs**: https://www.shadcn-vue.com/ https://primevue.org/ 

**Role:** Fullstack Developer – Setup, Frontend Architecture, Multitenancy Support, Routing & Theming

---

### 1. Multitenancy & Routing Setup

**Challenge:**
The system is multi-tenant, where each college/tenant has its own subdomain (`tenant_name.domain.com`). The challenge was to ensure seamless routing across tenants in both frontend (Vue 3 + Inertia) and backend (Laravel) while maintaining dynamic URL generation and route linking.

**Solution Implemented:**

1. **Custom Global Helper – `tenantRoute`**

   * Created a Vue helper to dynamically generate tenant-specific URLs based on the current tenant ID, Inertia props, or query parameters.
   * Integrated with **Ziggy JS** to replicate Laravel’s route helper on the frontend.

```javascript
import { route } from 'ziggy-js';
import { Ziggy } from '../ziggy';

function getTenantDomain() {
  const tenantId =
    (window.__inertia_page__?.props.tenantId) ||
    new URLSearchParams(window.location.search).get('tenant') ||
    'rims';
  const centralDomain = (Ziggy.url || window.location.hostname).replace(/^https?:\/\//, '');
  const protocol = window.location.protocol;
  return `${protocol}//${tenantId}.${centralDomain}`;
}

export default function tenantRoute(name, params, absolute = true, config = Ziggy) {
  const tenantDomain = getTenantDomain();
  const ziggyConfig = { ...config, url: tenantDomain };
  return route(name, params, absolute, ziggyConfig);
}
```

**Impact:**

* Ensured all route generation on the frontend respects tenant subdomains.
* Eliminated hardcoded URLs and reduced routing errors across tenants.
* Maintains compatibility with Inertia.js navigation and Laravel route names.

---

### 2. Frontend Initialization & Inertia Integration

**Setup Highlights:**

* Vue 3 app initialized via `createInertiaApp` with Ziggy and custom `tenantRoute` global helper.
* PrimeVue used for UI components with **Aura theme**, supporting dynamic light/dark modes.
* Global plugins added: `ToastService`, `ConfirmationService`, `ImgFallback`.
* Smooth Inertia page resolution with `laravel-vite-plugin` helpers for dynamic imports.

```javascript
app.config.globalProperties.route = tenantRoute; // Overriding global route helper
```

**Impact:**

* Frontend fully supports dynamic tenant routing.
* Optimized component imports with Vite SSR for performance.
* Centralized theme management ensures consistent UX across tenants.

---

### 3. Vite & Project Configuration for Multitenancy

**Key Features:**

* `server.host = true` to support LAN access and subdomain testing.
* CORS enabled for API requests across tenant subdomains.
* TailwindCSS integrated with Vue + Vite.
* Path alias `@` mapped to `resources/js` for clean imports.

```javascript
resolve: {
  alias: { '@': path.resolve(__dirname, 'resources/js') },
},
```

**Impact:**

* Simplifies module imports and ensures maintainable frontend structure.
* Multitenancy supported directly in local development without extra proxy setup.

___________________________________________

### 4. Custom Notification Channel – Expo Push Notifications

**Requirement:**
The system needs to notify students in the React Native app whenever key events occur:

* Fee created/paid
* Schedule updated
* Other LMS events

Laravel provides a robust notification system with built-in channels (mail, database, broadcast). The challenge was to **send notifications to mobile devices via Expo Push** while keeping everything queued, scalable, and maintainable.

---

#### **1. Created a Custom Notification Channel**

* Implemented `ExpoPushNotificationChannel` that integrates with Expo’s push API.
* Handles sending, logging, and exception catching.
* Supports `ShouldQueue` interface for asynchronous processing.

```php
namespace App\Broadcasting;

use Illuminate\Notifications\Notification;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class ExpoPushNotificationChannel
{
    public function send($notifiable, Notification $notification)
    {
        $to = $notifiable->routeNotificationFor('expoPush', $notification);
        if (! $to) {
            Log::warning('ExpoPushNotification: No token found.');
            return;
        }

        $message = $notification->toExpoPushNotification($notifiable);
        $message['to'] = $to;

        try {
            $response = Http::post('https://exp.host/--/api/v2/push/send', $message);
            if (! $response->successful()) {
                Log::error('Expo push failed', [
                    'status' => $response->status(),
                    'body' => $response->body(),
                ]);
            }
            logger('ExpoPushNotification: '.json_encode($response->body()));
        } catch (\Exception $e) {
            Log::error('ExpoPushNotification exception: '.$e->getMessage());
        }
    }
}
```

**Impact:**

* Mobile push notifications fully integrated with Laravel’s notification system.
* Handles offline/queued delivery and ensures logging for failed attempts.
* Centralized channel allows reuse across multiple notification types.

---

#### **2. Example Usage: Schedule Update Notification**

* Created `ScheduleUpdatedNotification` implementing `ShouldQueue`.
* Supports multiple channels: `mail`, `database`, `broadcast`, and `expoPush`.
* Automatically generates HTML table for email, array for database, broadcast payload, and Expo push payload.

```
// Registering a custom channel for Expo push notifications
        $this->app->make(ChannelManager::class)->extend('expoPush', function ($app) {
            return $app->make(ExpoPushNotificationChannel::class);
        });
```

```php
public function via(object $notifiable): array
{
    return ['mail', 'database', 'broadcast', 'expoPush'];
}

public function toExpoPushNotification($notifiable)
{
    return [
        'title' => $this->subject,
        'body' => 'Affected Subject(s): '.$this->schedules
            ->pluck('subject.name')
            ->unique()
            ->implode(', '),
        'data' => [
            'schedules' => $this->schedules->map(fn($s) => [
                'schedule_id' => $s->id,
                'date' => $s->date,
                'start_time' => $s->start_time,
                'end_time' => $s->end_time,
                'subject' => $s->subject->name ?? 'N/A',
                'batch' => $s->batch->name ?? 'N/A',
                'location' => $s->location_id,
            ]),
        ],
        'sound' => 'default',
    ];
}
```

**Impact:**

* Students receive real-time updates in the mobile app.
* Multi-channel support ensures notifications are delivered even if one channel fails.
* Code is modular, testable, and extensible for future notification types.

____________________________________________

### 5. Centralized Constants & Predefined Roles

**Challenge:**
In large systems, static reference data (like blood groups, states, quotas, sections, hostel options, and user roles) is often stored in the database or scattered across the codebase. This can lead to:

* Typos in values
* Inconsistent references across forms and validations
* Extra DB calls for static data

**Solution Implemented:**

1. **Created a Constants Class (`App\Constants\General`)**

* Holds all static data like blood groups, religions, quotas, Indian states, department types, sections, and hostel options.
* Provides helper methods to return structured arrays for forms (`getDepartmentTypes()`, `getSections()`, `getHostelOptions()`).

```php
namespace App\Constants;

class General
{
    public const BLOOD_GROUPS = ['A+', 'A-', 'B+', 'B-', 'AB+', 'AB-', 'O+', 'O-'];
    public const RELIGIONS = ['Hindu', 'Muslim', 'Christian', 'Sikh', 'Buddhist', 'Jain', 'Parsi', 'Jewish', 'Other'];
    public const QUOTAS = ['None', 'Management', 'NRI', 'Other'];
    public const INDIAN_STATES = [...]; // full list omitted for brevity
    public const DEPARTMENT_TYPES = ['Clinical', 'Non-Clinical'];
    public const SECTIONS = ['A','B','C','D','E','F','G','H'];

    public static function getDepartmentTypes(): array {
        return array_combine(self::DEPARTMENT_TYPES, self::DEPARTMENT_TYPES);
    }

    public static function getSections(): array {
        return array_combine(self::SECTIONS, self::SECTIONS);
    }

    public static function getHostelOptions($labeled = true): array { ... }
}
```

**Impact:**

* Eliminates typos and inconsistencies.
* Reduces database queries for static reference data.
* Provides reusable structured data for forms, validations, and APIs.

---

2. **Predefined User Roles (`App\Constants\UserRole`)**

* Created a dedicated class for role names and seeded the database to avoid runtime errors.

```php
namespace App\Constants;

class UserRole
{
    public const SUPERADMIN = 'superadmin';
    public const ADMIN = 'admin';
    public const FACULTY = 'faculty';
    public const STUDENT = 'student';
    public const ACCOUNTANT = 'accounts';
    public const AUTHOR = 'author';
}
```

**Impact:**

* Centralized role definitions prevent typos across authentication, authorization, and seeding scripts.
* Improves maintainability and readability of code using roles in conditions, policies, and access control.

___________________________________________

### 6. Tenant Preferences Middleware

**Requirement:**
Each tenant (college) can have **custom preferences** such as:

* Colors / themes
* Logo
* Favicon

These preferences should be applied dynamically across the frontend for the student web app, admin panel, and API responses. Additionally, loading preferences should be **efficient**, avoiding repeated database queries.

---

#### **Solution Implemented:**

* Created `SetTenantPreferences` middleware that runs on every request.
* Checks for a **JSON file cache** for the current tenant based on the hostname.
* If the cache file exists, preferences are loaded from the file.
* If not, preferences are fetched from the database (`SystemPreference`) and cached as a JSON file for subsequent requests.

```php
namespace App\Http\Middleware;

use App\Models\SystemPreference;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\File;
use Symfony\Component\HttpFoundation\Response;

class SetTenantPreferences
{
    public function handle(Request $request, Closure $next): Response
    {
        $host = $request->getHost(); 
        $path = storage_path("preferences/{$host}.json"); 

        if (File::exists($path) && ($preferences = json_decode(File::get($path), true))) {
            // Preferences loaded from JSON file
        } else {
            $preferences = SystemPreference::first();
            if ($preferences) {
                $preferencesArray = $preferences->toArray();
                File::ensureDirectoryExists(storage_path('preferences'));
                File::put($path, json_encode($preferencesArray, JSON_PRETTY_PRINT));
            } else {
                $preferencesArray = []; 
            }
        }

        return $next($request);
    }
}
```

**Impact:**

* **Dynamic tenant branding:** Applies colors, logos, and favicons automatically per tenant.
* **Performance optimization:** Reduces database queries by caching preferences in JSON files.
* **Scalable:** Easy to extend with more preferences without changing core logic.
* **Centralized management:** Middleware ensures preferences are loaded before request reaches controllers or views.
_________________________
---

### 7. Refactored Model Boot Logic Using Observers

**Challenge:**
Originally, the `boot()` method in multiple models contained logic for events like creating, updating, and deleting records. This led to:

* Messy and hard-to-maintain code
* Tight coupling between models and event logic
* Difficulty in testing and extending features

---

#### **Solution Implemented:**

* Replaced inline `boot()` logic in models with **dedicated Observer classes**.
* Each observer handles model-specific event logic, keeping models clean and focused only on data structure.
* Registered all observers in a centralized **Service Provider** (e.g., `AppServiceProvider` or dedicated `ObserverServiceProvider`).

```php
public function boot(): void
{
    User::observe(UserObserver::class);

    // Student and Faculty Profiles
    FacultyProfile::observe(FacultyProfileObserver::class);
    StudentProfile::observe(StudentProfileObserver::class);

    // Academic entities
    Course::observe(CourseObserver::class);
    Batch::observe(BatchObserver::class);

    // Leaves and Fees
    FacultyLeave::observe(FacultyLeaveObserver::class);
    StudentPayment::observe(StudentPaymentObserver::class);
    StudentFee::observe(StudentFeeObserver::class);

    // Faculty Topics
    FacultyTopics::observe(FacultyTopicsObserver::class);
}
```

**Impact:**

* **Cleaner code:** Models no longer contain event logic.
* **Better maintainability:** Observers are separate, reusable classes.
* **Decoupled design:** Event logic can evolve independently of model definitions.
* **Improved testability:** Observers can be tested separately from models.

___________________________________________
---

### 8. Payment Gateway Service Refactoring & Unified Wrapper

**Challenge:**
Initially, the system supported only **Razorpay**, and the payment logic was tightly coupled with controllers. When another gateway (Cashfree) needed to be integrated, this caused:

* Code duplication
* Hard-to-maintain controller logic
* No standard interface for multiple gateways

---

#### **Step 1: Dedicated Service Classes**

* Each payment gateway has its own **service class** (`RazorpayService`, `CashfreeService`).
* Controllers **depend on service classes via Dependency Injection** (`RazorpayController::__construct(RazorpayService $razorpay)`).
* Service classes encapsulate gateway-specific API logic:

  * Creating orders
  * Fetching payments
  * Handling settlements
  * Optional logging and metadata

**Example:** Razorpay Service

```php
class RazorpayService
{
    protected $api;
    protected $gateway;

    public function __construct($gatewayId = null)
    {
        $gateway = PaymentGateway::where('is_active', 1)->first();
        $this->gateway = $gateway;
        if ($gateway) {
            $this->api = new Api($gateway->api_key, $gateway->api_secret);
        }
    }

    public function createOrder(array $data) { return $this->api->order->create($data); }
    public function getPayment($paymentId) { return $this->api->payment->fetch($paymentId); }
    // ...other gateway-specific methods
}
```

**Controller Example (Dependency Injection):**

```php
class RazorpayController extends Controller
{
    protected $razorpay;

    public function __construct(RazorpayService $razorpay)
    {
        $this->razorpay = $razorpay;
    }

    public function makePayment(Request $request)
    {
        $order = $this->razorpay->createOrder([
            'amount' => $request->amount,
            'currency' => 'INR',
            'notes' => ['user_id' => auth()->id()]
        ]);
        return response()->json($order);
    }
}
```

**Impact:**

* Controllers remain **thin and focused**.
* Easy to **swap or add new gateways** without modifying controllers.
* Supports unit testing of services independently from controllers.

---

Here’s your **refined Step 2 write-up** with the GitHub + Packagist links properly included and polished for client review:

---

#### Step 2: Unified Payment Wrapper (**Laravel Payhub**)

I developed a package **Laravel Payhub**, a unified payment integration package that provides a **single, consistent API** for multiple payment gateways (currently **Razorpay** and **Cashfree**).

Instead of writing gateway-specific logic, developers can now create orders, process payments, and handle webhooks using a **standardized interface**.

---

#### 🔑 Key Features

* **Consistent API** – All gateways return responses in the **same format**, eliminating the need for duplicate parsing logic.
* **Extensible** – New gateways can be added easily by implementing a dedicated service class.
* **Custom Behavior** – Supports metadata, event dispatching, and webhook handling per order.
* **Open Source** – Published as a reusable package on GitHub & Packagist, solving a common Laravel payment integration challenge.

---

#### 📦 Installation

* **GitHub:** [https://github.com/PreciousGariya/laravel-payhub](https://github.com/PreciousGariya/laravel-payhub)
* **Packagist:** [https://packagist.org/packages/gokulsingh/laravel-payhub](https://packagist.org/packages/gokulsingh/laravel-payhub)

```bash
composer require gokulsingh/laravel-payhub
```

---

#### 💻 Usage Example

```php
// Create Razorpay order
$order = Payment::createOrder([
    'amount' => 500, 
    'currency' => 'INR',
    'metadata' => ['user_id' => auth()->id()]
]);

// Create Cashfree order
$orderCF = Payment::gateway('cashfree')->createOrder([
    'amount' => 1500,
    'currency' => 'INR',
    'customer_id' => "3297842",
    'email' => 'user@example.com'
]);

// Verify Razorpay payment
$payment = Payment::gateway('razorpay')->charge([
    'payment_id' => 'pay_XXXXXXXX'
]);
```

---

#### 🚀 Impact / Highlights

* **No Gateway-Specific Logic** – Controllers/services don’t need to worry about Razorpay vs. Cashfree responses.
* **Simplified Development** – Post-payment workflows (e.g., updating fees, notifications) can be written **once and reused**.
* **Future-Proof** – New gateways can be plugged in with minimal effort.
* **Community Contribution** – Available as an **open source package** for Laravel developers.

---
### 9. Modularized User Relations with Traits

The `User` model originally contained **dozens of student and faculty relationships**, which made the class heavy, hard to read, and difficult to maintain.

To solve this, I refactored the model by extracting role-specific relations into **dedicated traits**:

* **`FacultyRelations`** → faculty profile, education, experiences, assigned subjects, teaching schedules.
* **`StudentRelations`** → student profile, attendance, batches, fees, documents, courses.

The final `User` model looks clean and role-agnostic:

```php
class User extends Authenticatable implements FilamentUser, HasName, JWTSubject
{
    use FacultyRelations,
        StudentRelations,
        HasFactory,
        HasRoles,
        Notifiable,
        SoftDeletes,
        LogsHistory;
}
```

**Impact / Highlights**
* **Readability** – The `User` model only contains shared/auth logic.
* **Maintainability** – Student and faculty logic evolve independently in traits.
* **Reusability** – Traits can be reused or extended across other roles.
* **Scalability** – Supports new roles (e.g., staff, parents) without bloating the model.

---

### 10: Automatic History Logging (**LogsHistory Trait**)

I created a **reusable trait** `LogsHistory` to automatically record **data changes** for any Eloquent model. It logs **create, update, and delete actions** along with the user and IP who performed the action.

---

#### 🔹 Trait Implementation

```php
namespace App\Traits;

use App\Models\HistoryLog;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Request;

trait LogsHistory
{
    public static function bootLogsHistory()
    {
        static::created(function ($model) {
            if ($model->id) {
                $model->refresh(); // Ensures all attributes, including ID, are loaded
                $model->logHistory('created', [], $model->attributesToArray());
            }
        });

        static::updated(function ($model) {
            $model->logHistory('updated', $model->getOriginal(), $model->getChanges());
        });

        static::deleted(function ($model) {
            $model->logHistory('deleted', $model->getOriginal());
        });
    }

    protected function logHistory(string $action, array $oldData = [], array $newData = [])
    {
        if (!$this->id) return;

        HistoryLog::create([
            'table_name' => $this->getTable(),
            'record_id' => $this->id,
            'action' => $action,
            'old_data' => !empty($oldData) ? $oldData : null,
            'new_data' => !empty($newData) ? $newData : null,
            'user_id' => Auth::id(),
            'ip_address' => Request::ip(),
        ]);
    }
}
```

---

#### 🔹 Usage Example

Attach the trait to any Eloquent model, for example `StudentPayment`:

```php
namespace App\Models;

use App\Traits\LogsHistory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class StudentPayment extends Model
{
    use LogsHistory, SoftDeletes;

    protected $guarded = [];

    protected $casts = [
        'metadata' => 'array',
        'response' => 'array',
        'gateway_sync_data' => 'array',
        'paid_at' => 'datetime',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
        'deleted_at' => 'datetime',
    ];
}
```

---

#### 🔹 Key Benefits

* **Automatic Logging** – Records all creates, updates, and deletes.
* **User & IP Tracking** – Tracks who made the change and from which IP.
* **Reusable** – Attach the trait to any model without modifying the model’s core logic.
* **Data Comparison** – Logs both old and new values for updates.

---

### 11. Custom File Upload & Preview Components

#### Challenge

In modern apps, file management is critical—students uploading assignments, faculties sharing resources, or admins handling documents.

But existing Vue/NPM solutions have **major gaps**:

* Only support **basic image previews** (poor PDF/video support).
* Require **separate libraries** for uploading and previewing files.
* **Rigid UI** that doesn’t play well with Tailwind or dark mode.
* No **confirm-before-delete** or toast notifications.
* Weak **validation** for file type/size.
* Not easily integrated into real-world API flows (upload, download, delete).

This forces developers to **reinvent the wheel** with boilerplate drag-drop, modals, and validation code across projects.

---

### Solution

To solve this, I built **custom Vue 3 + Tailwind components**:

* **FileUploader.vue** → for uploading new files with drag & drop, previews, size/type validation, and modal support.
* **FilePreviewer.vue** → for managing already uploaded files, with support for image, video, PDF, download, and delete (with confirm + toast).

Both components are designed to be:

* **Consistent** → same UX for all file types.
* **Customizable** → Tailwind-ready, works in light/dark themes.
* **Flexible** → can be plugged into any backend (Laravel, Node.js, etc.).
* **Production-ready** → handles real-world needs like delete confirmations and safe previewing.

---

### 1. **FileUploader.vue**

For uploading new files.

**Features**

* Drag & Drop OR click to select
* Multi-file or single-file mode
* File type + size validation
* Previews: Images & PDFs inline, modal for full view
* Emits `update:modelValue` with selected files

**Usage**

```vue
<FileUploader
  v-model="localFiles"
  :allowMultiple="true"
  label="Upload Supporting Documents"
  accept="image/*,application/pdf"
  :maxFileSizeInMB="5"
/>
```

---

### 2. **FilePreviewer.vue**

For displaying and managing already uploaded files.

**Features**

* Displays images, videos, PDFs, and fallback for others
* Grid or list view
* Modal for enlarged previews
* Download button with file name
* Delete button with confirm dialog + toast
* Emits `delete` event with file URL

**Usage**

```vue
<FilePreviewer
  :files="uploadedFiles"
  label="Submitted Assignments"
  displayType="grid"
  :enableDelete="true"
  @delete="handleDelete"
/>
```

---

#### Combined Workflow Example

```vue
<script setup>
import { ref } from 'vue'
import FileUploader from '@/components/FileUploader.vue'
import FilePreviewer from '@/components/FilePreviewer.vue'

const localFiles = ref([])
const uploadedFiles = ref([
  "https://example.com/files/report1.pdf",
  "https://example.com/files/photo.png"
])

function handleDelete(url) {
  console.log("Delete requested for:", url)
  // API call → delete from server
  uploadedFiles.value = uploadedFiles.value.filter(f => f !== url)
}
</script>

<template>
  <FileUploader v-model="localFiles" label="Upload New Files" />
  <FilePreviewer :files="uploadedFiles" label="Existing Files" @delete="handleDelete" />
</template>
```

###  Impacts

* **Unified Workflow** → Same UX for uploading and managing files across the app.
* **Reduced Boilerplate** → No need to repeatedly write drag-drop, previews, and validation logic.
* **Improved UX** → Users can preview, confirm, and manage files without leaving the page.
* **Stronger Validations** → Prevents oversized/unsupported files at the client-side itself.
* **Safer File Management** → Confirm dialogs + toasts avoid accidental deletions.
* **Better Compatibility** → Works in both light and dark mode with Tailwind classes.
* **Extensible** → Easily adapts to any backend (Laravel, Node.js, etc.) without refactor.
* **Future-proof** → Supports multiple file formats (image, PDF, video, docs) instead of just images.

---

# **Coco Hospitals – Multi-Tenant SaaS with Multi-Guard Authentication & POC**

### **Challenge**

Coco Hospitals aimed to build a **SaaS hospital management platform** where multiple hospitals could operate independently with:

* **Isolated databases per hospital**
* **Role-based access** (superadmin, hospital admin, doctor, patient, lab admin)
* **Module-based development** (nwidart modules)

**Stack:**

* **Frontend:** Next.js (subdomain resolver for tenants)
* **Backend:** Laravel (multi-tenant DB + multi-guard auth) without any package or library.
* **UI Widgets:** Vue CDN-based
* **Large Data Handling:** Yajra Datatables

**POC:** Before full implementation, a **Next.js POC** was created to verify subdomain-based multitenancy and tenant routing:

* Detect tenant via subdomain (`tenant1.cocohospitals.com`)
* Rewrite URL to tenant-specific path (`/tenant/tenant1/...`)
* Validate tenant exists in a JSON list

**POC Repository:** [tenant-educators](https://github.com/PreciousGariya/tenant-educators/)

---

## **POC – Next.js Multi-Tenant Middleware**

`middleware.ts`

```ts
import { NextResponse } from "next/server";
import { tenants } from "./data/tenants";

const BASE_DOMAIN = process.env.NEXT_PUBLIC_BASE_DOMAIN || "example.com";

export function middleware(request) {
    const url = request.nextUrl;
    const host = request.headers.get("host")?.toLowerCase();
    const subdomain = host?.split(".")[0];

    console.log("Middleware Debug:", { host, subdomain, pathname: url.pathname });

    if (!subdomain || subdomain === "www" || host === BASE_DOMAIN || url.pathname.endsWith("/not-found")) {
        return NextResponse.next();
    }

    const isValid = tenants.some(t => t.subdomain === subdomain);

    if (!isValid) {
        return NextResponse.redirect(new URL(`https://${BASE_DOMAIN}/not-found`));
    }

    const tenantUrl = `/tenant/${subdomain}${url.pathname}${url.search}${url.hash}`;
    return NextResponse.rewrite(new URL(tenantUrl, request.url));
}

export const config = {
    matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```

**POC Tenant Data Example (`data/tenants.ts`):**

```ts
export const tenants = [
  {
    id: "1",
    name: "Emma Wilson",
    subdomain: "emma",
    domain: "https://emma.com",
    theme: "dark",
    logo: "https://images.unsplash.com/photo-1583468982228-19f19164aee2",
    about: "Emma is a leading digital marketing expert...",
  },
  {
    id: "2",
    name: "Michael Chen",
    subdomain: "michael",
    domain: "https://michael.com",
    theme: "light",
    logo: "https://plus.unsplash.com/premium_photo-1681681082145-248d75ebee8a",
    about: "Michael is a top sales strategist...",
  },
];
```

**POC Demo:**

* Rewrite subdomain `emma.cocohospitals.com` → `/tenant/emma`
* Rewrite subdomain `michael.cocohospitals.com` → `/tenant/michael`
* Invalid subdomain → redirect to `/not-found`

**Impact:**

* Verified subdomain detection, validation, and tenant routing
* Served as blueprint for full SaaS implementation
* Allowed frontend team to start building tenant-specific dashboards

---

## **Full Coco Hospitals Implementation**

The POC seamlessly integrated with **Laravel backend**:

### **1. Multi-Tenant Middleware (Laravel)**

`SwitchHospitalDatabase.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\DB;
use App\Models\Hospital;
use Illuminate\Support\Facades\Cache;

class SwitchHospitalDatabase
{

    public function handle($request, Closure $next)
    {
        // Get hospital slug from header or route
        $hospitalSlug = $request->header('X-TENANT-Short-Name') ?? $request->route('hospital');

        if (! $hospitalSlug) {
            abort(404, 'Hospital identifier not provided');
        }

        // Try caching hospital lookup to avoid hitting DB every time (optional)
        $hospital = Cache::remember("hospital:{$hospitalSlug}", now()->addMinutes(5), function () use ($hospitalSlug) {
            return Hospital::where('short_name', $hospitalSlug)->first();
        });

        if (! $hospital) {
            abort(404, 'Hospital not found');
        }

        // Attach hospital model and slug to request
        $request->merge([
            'hospital' => $hospital,
            'hospital_short_name' => $hospitalSlug
        ]);

        // Switch DB connection
        $this->switchDatabaseConnection($hospital);

        return $next($request);
    }

    private function switchDatabaseConnection(Hospital $hospital): void
    {
        $dynamicConfig = [
            'driver'    => 'mysql',
            'host'      => $hospital->db_host,
            'database'  => $hospital->db_name,
            'username'  => $hospital->db_username,
            'password'  => $hospital->db_password,
            'charset'   => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
        ];

        // Only switch if it's actually a different database
        if (
            config('database.default') !== 'dynamic' ||
            config('database.connections.dynamic.database') !== $dynamicConfig['database']
        ) {
            logger("🔁 Switching to database: " . $hospital->db_name);

            config(['database.connections.dynamic' => $dynamicConfig]);

            DB::purge('dynamic');
            DB::reconnect('dynamic');

            config(['database.default' => 'dynamic']);
        }
    }
}


```

* Dynamically switches DB per hospital
* Cached hospital lookups
* Attaches hospital info to request

### **2. Multi-Tenant Migrations**

`DbMigrationAll.php`

```php
<?php

namespace App\Console\Commands;

use App\Models\Hospital;
use App\Models\User;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Artisan;

class DbMigrationAll extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'db:migrate {database?} {--seed}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Run migrations and optionally seed data for the specified database';

    /**
     * Execute the console command.
     */
    public function handle()
    {
        $database = $this->argument('database'); // This is the optional argument for the database
        $shouldSeed = $this->option('seed'); // This is the optional flag for seeding

        // If a database is provided, filter hospitals by db_name; otherwise, use all hospitals
        if ($database) {
            $hospitals = Hospital::where('db_name', $database)->get();
        } else {
            $hospitals = Hospital::all(); // Get all hospitals if no database is specified
        }

        foreach ($hospitals as $hospital) {
            $db = $hospital->db_name;
            $this->connect($db, $shouldSeed);
        }
    }

    /**
     * Connect to the specified database and run migrations and optionally seed data.
     *
     * @param string $database
     * @param bool $shouldSeed
     */
    public function connect($database, $shouldSeed)
    {
        // Dynamically set the database connection based on the provided database name
        config(['database.connections.dynamic.database' => $database]);

        DB::purge('dynamic');

        try {
            DB::reconnect('dynamic');

            $user = User::find(1);

            // Run Migrations
            Artisan::call('migrate', [
                '--database' => 'dynamic',
                '--path' => 'database/migrations/hospital', // Adjust the path if necessary
            ]);

            $this->info("Migrations for database '{$database}' have been executed successfully.");

            // Optionally run seeders if the --seed flag is passed
            if ($shouldSeed) {
                Artisan::call('db:seed', [
                    '--database' => 'dynamic',
                    '--class' => 'DatabaseSeeder', // You can specify a specific seeder class if needed
                ]);

                $this->info("Seeders for database '{$database}' have been executed successfully.");
            }
            if(User::get()->count() == 0){
                User::create($user);
            }
        } catch (\Exception $e) {
            $this->error("Error running migrations or seeders for database '{$database}': " . $e->getMessage());
        }
    }
}

```

* Run migrations per hospital database
* Optionally seed data

### **3. Multi-Tenant Queue Workers**

`DBQueueCommand.php`
```php
<?php

namespace App\Console\Commands;

use App\Models\Hospital;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Log;

class DBQueueCommand extends Command
{


    protected $signature = 'dbqueue:run
    {--queue=default : The queue to process}
    {--tries=3 : Number of times to attempt a job before failing}
    {--sleep=3 : Number of seconds to sleep when no jobs are available}
    {--timeout=60 : Maximum seconds a job can run before timing out}
';

    protected $description = 'Run tasks in parallel using PCNTL';

    public function handle()
    {
        $databases = Hospital::pluck('db_name');

        if ($databases->isEmpty()) {
            $this->error('No hospitals found in the database.');
            return;
        }

        $databases = $databases->toArray(); // Convert to array only after checking
        $processes = [];

        $queue = $this->option('queue');
        $tries = $this->option('tries');
        $sleep = $this->option('sleep');
        $timeout = $this->option('timeout');
        $verbose = $this->getOutput()->isVerbose(); // Check verbosity


        Log::info("Starting MultiThreadCommand with parameters", [
            'queue' => $queue,
            'tries' => $tries,
            'sleep' => $sleep,
            'timeout' => $timeout,
            'verbose' => $verbose
        ]);

        foreach ($databases as $database) {
            $pid = pcntl_fork();

            if ($pid == -1) {
                $this->error("Failed to fork process for $database");
                Log::error("Failed to fork process for database: $database");
            } elseif ($pid) {
                // Parent process - Store PID
                $processes[] = $pid;
            } else {
                // Child process
                $this->info("Starting queue worker for database: $database");
                Log::info("Starting queue worker for database: $database");

                $command = "DB_DATABASE=$database php artisan queue:work --queue=$queue --tries=$tries --sleep=$sleep --timeout=$timeout";

                if ($verbose) {
                    $command .= " --verbose";
                }
                $this->info("Executing command: $command");
                Log::info("Executing command: $command");

                $this->info(exec($command, $output, $retval));

                Log::info("Queue worker output", ['database' => $database, 'output' => $output, 'return_code' => $retval]);

                exit(0); // Ensure child process exits
            }
        }

        // Wait for all child processes
        foreach ($processes as $pid) {
            pcntl_waitpid($pid, $status);
        }

        $this->info("All queue workers started successfully!");
        Log::info("All queue workers started successfully.");
    }
}

```

* Parallel queue workers per tenant using PCNTL fork
* Isolated DB context for background jobs

### **4. Multi-Guard Authentication**

`config/auth.php`

* Guards: `web`, `hospital`, `superadmin`, `doctor`, `patient`, `labadmin`
* API Guards: `api`, `api_doctor`, `api_patient`
* Providers map each guard to the proper Eloquent model

### **5. Module-Based Development**

* Nwidart modules: `SuperAdmin`, `Hospital`, `Doctor`, `Patient`, `Lab`
* Separation of concerns, easier parallel development

### **6. Datatables Integration**
```php
public function index(Request $request)
    {
        try {
            if ($request->ajax()) {
                $with = ['patient', 'doctor', 'appointment'];

                // Initialize query
                $query = Feedback::with($with);

                // Apply status filter if provided
                // Apply status filter if provided
                if ($request->filled('status')) {
                    $query->where('status', $request->get('status'));
                }

                // Apply feedback_type filter if provided
                if ($request->filled('feedback_type')) {
                    $query->where('feedback_type', $request->get('feedback_type'));
                }
                if ($request->filled('source')) {
                    $query->where('source', $request->get('source'));
                }


                // Apply search filter if provided
                if ($request->filled('search')) {
                    $searchTerm = $request->get('search');
                    $query->where(function ($q) use ($searchTerm) {
                        $q->where('feedback', 'like', "%$searchTerm%")
                        ->orWhere('admin_response', 'like', "%$searchTerm%")
                            ->orWhereHas('patient', function ($q) use ($searchTerm) {
                                $q->where('name', 'like', "%$searchTerm%");
                            })
                            ->orWhereHas('doctor', function ($q) use ($searchTerm) {
                                $q->where('DoctName', 'like', "%$searchTerm%");
                            })
                            ->orWhereHas('appointment', function ($q) use ($searchTerm) {
                                $q->where('appointment_reference_number', 'like', "%$searchTerm%");
                            });
                    });
                }
                $feedback = $query->get();

                return DataTables::of($feedback)
                    ->addIndexColumn()
                    ->make(true);
            }
        } catch (\Exception $ex) {
            logger($ex->getMessage());
            return response()->json([
                'status' => 'error',
                'message' => $ex->getMessage()
            ], 500);
        }

        return view('admin.feedback.index');
    }
```

* Yajra Datatables server-side filtering for large datasets
* Each hospital sees only its own data

---

### **Impact of POC & Final Implementation**

* **Validated subdomain routing** before full build
* **Safe multi-tenancy:** each hospital has isolated DB
* **Role-specific access:** guards for superadmins, hospitals, doctors, patients, lab admins
* **Parallel background processing:** queues per hospital
* **Modular, maintainable codebase** with Nwidart
* **Performance at scale:** handles large datasets efficiently
* **Audit & compliance ready** for healthcare regulations

---

#### **Demo / References**

* 🌐 **Main Portal:** [cocohospitals.com](https://cocohospitals.com/)
* 🏥 **Tenant Portal (RIMS):** [rims.cocohospitals.com](https://rims.cocohospitals.com/)
* 💻 **POC Source Code:** [tenant-educators](https://github.com/PreciousGariya/tenant-educators/)


---


## Open Source Contribution – Signature App 

I developed and published an **open-source Nuxt.js application** that allows users to **draw, preview, and download their digital signatures** directly in the browser.

####  About

By leveraging the capabilities of **Nuxt.js 3** and **Vue Canvas Drawing**, this solution delivers a **seamless and intuitive signature experience**, empowering users to effortlessly create and download their signatures while ensuring precision and convenience.

#### Live Demo

[nuxtjs-signature-app-lovat.vercel.app](https://nuxtjs-signature-app-lovat.vercel.app/)

#### GitHub Repository

[https://github.com/PreciousGariya/nuxtjsSignatureApp](https://github.com/PreciousGariya/nuxtjsSignatureApp)

#### Impacts / Highlights

* **Open Source Support** – Available for anyone to use, customize, and extend.
* **Cross-browser Compatibility** – Works across major browsers without plugins.
* **Lightweight & Fast** – Built with Nuxt.js 3, optimized for performance.
* **Practical Use Case** – Helpful for digital forms, e-signatures, and online agreements.

---
